# Field 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `Field` 的**类继承树与类型系统**（子类的功能域划分）、**`store` ↔ `val_*` 双向值协议**（与 `Item` 的对偶）、**在行缓冲里的存储布局**（`ptr` / 长度头 / NULL 位 / `pack_length`）、**字符集与比较排序**（`cmp` / `get_key_image` / `make_sort_key`）、**与 `Item`、`TABLE`、handler、DD 的接缝**，以及**默认值、instant ADD/DROP 与虚拟列**在 Field 层的投影。
>
> **边界**：本篇讲"**列**"这个对象的实现（`Field` 那一族）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [一、类继承树与类型系统](#一类继承树与类型系统)
  - [二、值协议：store ↔ val_* 的对偶](#二值协议store--val_-的对偶)
  - [三、存储布局：ptr、长度头、NULL 位与 pack_length](#三存储布局ptr长度头null-位与-pack_length)
  - [四、字符集、比较与排序键](#四字符集比较与排序键)
  - [五、与 Item / TABLE / handler / DD 的接缝](#五与-item--table--handler--dd-的接缝)
  - [六、特殊子类：JSON / ENUM-SET / BIT / 时态类型](#六特殊子类json--enum-set--bit--时态类型)
  - [七、默认值、instant 列与虚拟列](#七默认值instant-列与虚拟列)
  - [八、★ `make_field`：把列定义变成 `Field` 对象](#八-make_field把列定义变成-field-对象)
  - [九、★ `fix_fields`：列名 → `Field*` 的完整解析链路](#九-fix_fields列名--field-的完整解析链路)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

> **源码坐标**（正文中以 `[F1]`…`[F12]` 引用；每个文件名在全文只在此出现一次）
> `[F1]` sql/field.h — Field 类族、`type_conversion_status`、`ptr`/`field_length`、变长与 BLOB 布局
> `[F2]` sql/field.cc — 类型合并表、`enum_field_types → Item_result` 映射表、告警与错误落地点
> `[F3]` sql/item.h — `Item_field`、`Item_ident_for_show` 的求值接口
> `[F4]` sql/item.cc — `Item_field::val_*()` 的实现体
> `[F5]` sql/table.h — `TABLE_SHARE` / `TABLE` 与行缓冲
> `[F6]` sql/handler.h — `enum_alter_inplace_result` 与 INSTANT 算法
> `[F7]` include/field_types.h — `enum_field_types`
> `[F8]` include/mysql/udf_registration_types.h — `enum Item_result`
> `[F9]` storage/innobase/include/data0type.h — `dtype_t`（引擎侧类型表示）
> `[F10]` sql/dd/types/column.h — `dd::Column::enum_hidden_type`
> `[F11]` sql/field_conv.cc — `set_field_to_null()` 等转换路径
> `[F12]` sql/sql_table.cc — ALTER 算法调度

### 是什么

`Field` 是 server 层对**"一列"**的抽象。它同时持有三样东西：

- **类型与元数据**：类型码（`type()` / `real_type()` / `binlog_type()`）、长度（`field_length` / `pack_length()` / `char_length()` / `decimals()`）、字符集（`charset()`）、可空性（`is_nullable()` / `m_null_ptr` + `null_bit`）、默认值与生成表达式（`auto_flags` / `set_default()` / `m_default_val_expr`）。
- **行内布局**：该列在 `TABLE::record[0]` 中的起始位置（`ptr`，注释即 "Holds the position to the field in record"）与占多少字节。
- **双向转换接口**：`store*(...)` 把 SQL 值写进 `ptr` 后的字节；`val_int()` / `val_real()` / `val_str()` / `val_decimal()` / `get_date()` 从 `ptr` 后的字节解出 SQL 值；另有 `pack()` / `unpack()` / `cmp()` / `make_sort_key()` 供索引与排序使用。

继承树在 `[F1]` 顶部的注释里给出；8.0.39 里共 **38 个子类**（其中 28 个是可实例化的叶子类型）——详见「一、类继承树与类型系统」。

### 用途

`Field` 处在"表定义"与"行字节"之间，是**唯一的翻译层**：

```
   dd::Column / Create_field                       （[F10]、[F1]）
   持久化元数据：类型码、字符集、可空性、默认值/生成表达式、隐藏属性
        │  开表时物化（make_field）
        ▼
   TABLE_SHARE                                     （[F5]）
    s->fields / s->field[]（Field **）/ s->reclength / s->stored_rec_length
    s->default_values（与 record[0] 平行的"默认值行"）
        │  为每列 new 一个 Field 子类，并把 ptr 指向 record[0] 内的偏移
        ▼
   Field  ── ptr ──┐                               （[F1]）
        │          │
        │          ▼
        │   TABLE::record[0]:  ┌ null bits ─┬ col0 bytes ─┬ col1 bytes ─┬ … ┐
        │                      └───────────┴─────────────┴─────────────┴───┘
        ▼
   Item_field / 表达式树、协议输出、索引与排序
```

`Item` 与 `Field` 是一对对偶：前者是"可求值的表达式"，后者是"可解码的存储槽"。二者由 `Item_field` 缝合（`[F3]`）：

```
   表达式侧 Item                          存储侧 Field
   val_int()  = 求值（可能触发计算/子查询）  val_int() = 从 ptr 解码整数
   val_str()  = 求值                       val_str() = 从 ptr 解码字符串
        \                                      /
         \            Item_field                /
          ── result_type()  { return field->result_type(); }
             val_int()      { if ((null_value = field->is_null())) return 0;
                              return field->val_int(); }              （[F4]）
```

> **反直觉切入点**：`Item` 与 `Field` 都声明了 `val_int()` / `val_str()` / `val_real()` / `val_decimal()`，但 `Item::val_*()` 是**求值**（可能递归下探整个表达式树、触发子查询/函数/类型转换），`Field::val_*()` 是**解码**（只读 `ptr` 指向的若干字节，不产生副作用——除了 `val_str(buf1, buf2)` 的双缓冲技巧，注释明确说明这是为了减少一次 malloc）。同名不是巧合，是**刻意设计**：正因为签名一致，`Item_field` 才能把每个 `val_*()` 写成"先判 NULL、再转发"的一行代码。代价是读代码时极易混淆——判断依据只有一个：**看对象是不是 `Field`**。

### 版本演进

| 版本 | 变化 | 源码依据 |
|---|---|---|
| 5.0 系（true VARCHAR） | VARCHAR 按字节计长、长度前缀化；`Field::MAX_VARCHAR_WIDTH = 65535`（字节），`Field_varstring::pack_length() = field_length + length_bytes`（前缀 1 或 2 字节） | `[F1]`；`[F5]` 中 `mysql_version` 的注释 "0 if .frm is created before 5.0" 佐证 5.0 是行格式分水岭 |
| 5.6 系（时态小数秒） | 新增 `MYSQL_TYPE_TIME2 / DATETIME2 / TIMESTAMP2`，注释标明 "Internal to MySQL. Not used in protocol"；`real_type_to_type()` 负责把它们映射回 `type()`；`binlog_type()` 注释解释为何带 fsp 的时态类型必须落 real_type | `[F7]`、`[F1]` |
| 5.7 系（JSON、生成列） | `MYSQL_TYPE_JSON`；`Field::gcol_info` + `is_gcol()` / `is_virtual_gcol()` / `stored_in_db` | `[F7]`、`[F1]` |
| 8.0（`auto_flags`） | 用 `enum_auto_flags` 位图（NEXT_NUMBER / DEFAULT_NOW / ON_UPDATE_NOW / GENERATED_FROM_EXPRESSION）取代旧的 `utype` / `unireg_check`，注释明确写 "in pre-8.0 versions of server" | `[F1]` |
| 8.0（INSTANT ALTER） | `HA_ALTER_INPLACE_INSTANT`；`ALGORITHM=INSTANT`；注释 "For instant algorithm this is the step during which SE changes are done"，prepare / inplace 两步对 instant 都是 no-op | `[F6]`、`[F12]` |
| 8.0（默认值表达式 + 隐藏列） | `Field::m_default_val_expr`；`dd::Column::enum_hidden_type`（HT_VISIBLE / HT_HIDDEN_SE / HT_HIDDEN_SQL，后者注释点名 "implementing functional indexes"） | `[F1]`、`[F10]` |
| 8.0（多值索引） | `MYSQL_TYPE_TYPED_ARRAY`（"Used for replication only"）与 `Field_typed_array` | `[F7]`、`[F1]` |

> 说明：上表中"5.0 / 5.6 / 5.7 系"是依据常量与注释的语义推断，源码内**未找到显式版本字符串**；8.0 各项也只能确认到 8.0 大版本。

---

## 理论基础

### 设计思想与权衡

#### 为什么"列"要同时承担类型、布局、转换三件事

**对比引擎侧。** `[F9]` 里 `dtype_t` 只有三个字段：`prtype`（32 位，"precise type; MySQL data type, charset code, flags to indicate nullability, signedness…"）、`mtype`、`len`；配套工具是 `dtype_form_prtype()` / `dtype_get_prtype()` / `dtype_get_mblen()`。也就是说引擎把"类型 + 字符集 collation 码 + NOT NULL + 无符号 + 是否虚拟列"**压进一个 32 位整数**。它足以回答"这列多少字节、怎么比较字节序"，但答不出：

- `TIME(3)` 的小数位是多少（`decimals()`）；
- `ENUM` 的取值表 `TYPELIB` 在哪；
- 生成列的表达式是什么（`gcol_info`）；
- 截断该报 note、warning 还是 error（`type_conversion_status`）。

**为什么不拆开。** 如果语义与布局分离成两个对象，热路径上每次 `val_str()` / `cmp()` / `make_sort_key()` 都要先做一次"类型描述表"的间接查找。MySQL 选择把三者合成一个对象，让"该列怎么放、怎么读、怎么解释"在同一个 vtable 里——代价是类爆炸与对象变重。

**三者的耦合点是 `ptr`。**

```
        ┌────────── 类型 ──────────┐   ┌──────── 布局 ────────┐   ┌──── 转换 ────┐
        │ type() / real_type()     │   │ pack_length()        │   │ store*()    │
        │ result_type() / charset()│   │ pack_length_in_rec() │   │ val_*()     │
        │ decimals() / is_unsigned │   │ data_length()        │   │ pack/unpack │
        └────────────┬─────────────┘   │ make_sort_key()      │   │ cmp()       │
                     │                 └──────────┬───────────┘   └──────┬──────┘
                     └──────────────► ptr ◄───────┴──────────────────────┘
                 "ptr 之后多少字节有效" + "如何解释这些字节" + "如何双向翻译"
```

注释里的细节也印证了这种耦合：`Field::eq()` 用 `ptr == field->ptr && m_null_ptr == field->m_null_ptr && null_bit == field->null_bit && type() == type()` 判断两个 Field 是否同一列——**"是不是同一列"这个语义问题，是靠指针（布局）+ 类型码共同回答的**。

**代价已被承认**：`[F1]` 中 `class Field` 开头那段注释坦白，本想禁止拷贝但"due to a typo, this was never enforced"，现在只能保留默认拷贝构造并劝阻新代码依赖它。

#### 为什么 `store` 返回 `type_conversion_status` 而不是 bool

**理由一：严重度是有序的，可以直接比较。** `[F1]` 的注释原话：*"The values should be listed in order of increasing seriousness so that if two type_conversion_status variables are compared, the bigger one is most serious."* 于是调用方可以写 `if (status > TYPE_WARN_TRUNCATED) …`，bool 做不到。九个取值与语义：

| 值 | 语义（按注释） |
|---|---|
| `TYPE_OK` | 存储/转换正常 |
| `TYPE_NOTE_TIME_TRUNCATED` | 时态间转换丢失信息，如 DATETIME→DATE 丢掉时间部分 |
| `TYPE_NOTE_TRUNCATED` | 存进去了，被裁掉的部分无关紧要，只发 note（如 5 位小数存入 3 位列、尾部空格被裁） |
| `TYPE_WARN_OUT_OF_RANGE` | 超出类型 min/max，改存边界值 |
| `TYPE_WARN_TRUNCATED` | 存进去了，但被裁掉的部分有意义，要发 warning |
| `TYPE_WARN_INVALID_STRING` | 字符串非法；注释特别指出"出现在等值谓词里时，range optimizer 返回一个 impossible where" |
| `TYPE_ERR_NULL_CONSTRAINT_VIOLATION` | 往 NOT NULL 列存 NULL |
| `TYPE_ERR_BAD_VALUE` | 类型不兼容（如 "foo" 转 DATE） |
| `TYPE_ERR_OOM` | 内存不足 |

**理由二："存进去了"和"存得对不对"是正交的。** `TYPE_WARN_TRUNCATED` 意味着**确实写入了截断后的值**（注释："The string 'fo' is stored in this case"），`TYPE_ERR_*` 才意味着没写成。bool 无法表达"写了一半 + 还带一个 warning"。

**理由三：status 是纯数据，策略在调用方。** 把"严重度 → note/warning/error"的裁决权留在 Field 内部会让 strict mode 无处安放：

- `[F2]` 的 `Field::set_warning()` 先判 `thd->check_for_truncated_fields`，关着就直接返回 `level >= SL_WARNING` 而**不发**任何告警；且只对 INSERT/REPLACE 聚合告警（"We aggregate warnings from only INSERT and REPLACE statements"）。
- `[F2]` 的 `Field::check_constraints(mysql_errno)`：`CHECK_FIELD_WARN` 发完 warning 后 `[[fallthrough]]` 到 `CHECK_FIELD_IGNORE` 并**返回 `TYPE_OK`**；只有 `CHECK_FIELD_ERROR_FOR_NULL` 才 `my_error(ER_BAD_NULL_ERROR)` 并返回 `TYPE_ERR_NULL_CONSTRAINT_VIOLATION`。同一个 "NOT NULL 列收到 NULL" 的事实，三种策略给出三种 status。
- 非 strict 模式下，`Field_temporal::store(longlong,…)` 会把本该是 `TYPE_ERR_BAD_VALUE` 的零日期降级为 `TYPE_NOTE_TIME_TRUNCATED`。
- `[F4]` 的 `Item::save_in_field_no_warnings()` 靠临时清掉 `MODE_STRICT_*` + 置 `CHECK_FIELD_IGNORE` 来整体降级；`Item::save_in_field()` 返回前还要补一句 `if (ret == TYPE_OK && thd->is_error()) return TYPE_ERR_BAD_VALUE;`，因为错误可能从求值路径抛出来而 status 仍是 OK。

**理由四：status 还要被优化器消费。** `TYPE_WARN_INVALID_STRING` 的注释直接描述了对 range optimizer 的影响。返回 bool 会丢掉这条信息通道。

#### 为什么类型码有两套：`enum_field_types` 与 `Item_result`

- **`enum_field_types`（`[F7]`，`Field::type()`）**：物理/协议层面的类型码，用于 DDL、binlog 元数据、客户端协议。它**区分同语义不同字节宽的变体**：`MYSQL_TYPE_TIME` vs `MYSQL_TYPE_TIME2`、`VARCHAR` vs `VAR_STRING`、`TINY/MEDIUM/LONG_BLOB` vs `BLOB`、`MYSQL_TYPE_DATE` vs `MYSQL_TYPE_NEWDATE`。
- **`Item_result`（`[F8]`，`Field::result_type()`）**：表达式求值的**结果大类**，只有 6 个值 `INVALID / STRING / REAL / INT / ROW / DECIMAL`。它不区分 BLOB 宽度、不区分 DATE 与 TIME，回答的是"这个值在算术与比较中当什么用"。

**为什么不能合并——三点证据：**

**(1) 是多对一映射，且不是从物理码能猜出来的。** `[F2]` 的 `field_types_result_type[]` 就是这张表：所有 BLOB 变体、`DATE/TIME/DATETIME/NEWDATE`、`VARCHAR`、`JSON`、`ENUM/SET` → `STRING_RESULT`；`TINY/SHORT/LONG/LONGLONG/INT24/**YEAR**/BIT` → `INT_RESULT`；`FLOAT/DOUBLE` → `REAL_RESULT`；`DECIMAL/NEWDECIMAL` → `DECIMAL_RESULT`。注意 **`YEAR` 是 `INT_RESULT` 而 `DATE` 是 `STRING_RESULT`**——这正说明"物理类型码"推不出"求值大类"，必须查表。

**(2) 结果大类随上下文变化，Field 为此留了钩子**（`[F1]`）：

- `result_type()` —— 纯虚，每个子类定；
- `numeric_context_result_type()` —— 注释举例：`SELECT time_column + 1`、`SELECT SUM(time_column)` 中，`TIME/DATETIME/TIMESTAMP` 当 `INT`，而 `TIME(1)/DATETIME(1)` 当"1 位小数的 DECIMAL"；
- `cmp_type()`、`cast_to_int_type()` —— 比较与显式转整数时的大类。

**(3) 还有第三套视角 `binlog_type()`。** `[F1]` 的注释解释：CHAR/VARCHAR/SET/ENUM 一律压成 `MYSQL_TYPE_STRING` 并把细节塞进 metadata；而带小数秒的时态类型"cannot store field->type()"，因为 `MYSQL_TYPE_TIME / MYSQL_TYPE_DATETIME` 的 binlog 记录**没有 metadata 位**，所以必须返回 `real_type()`（即 `*2` 系列）+ 把精度写进 metadata。

#### 简化假设的失效场景与退化阈值

**假设 A："`ptr` 指向行缓冲中该列连续的 `pack_length()` 个字节，这些字节就是值。"**

- **失效 A1 · BLOB 外存。** `[F1]` 中 `Field_blob::pack_length() = packlength + portable_sizeof_char_ptr`（后者宏定义为 8，注释说明这是"in-memory data record 里指向数据片段的指针所占字节数"）。行缓冲里**根本没有值**，只有 `packlength` 字节的长度 + 一个指针；行内真实字节数是 `pack_length_no_ptr()`。真值要 `get_blob_data()` 从 `ptr + packlength` 处 memcpy 出一个 `uchar*` 再跳出去，可能落在外部存储页。因此任何"按 `pack_length()` memcpy 整行"的代码都会把指针值当数据拷走——这正是 `TABLE::copy_blobs` 与 `Field_blob::value / old_value / m_blob_backup` 存在的原因。**退化阈值：只要 `packlength`（1~4）字节的长度字段存在，行缓冲就不是自包含的。**
- **失效 A2 · 索引虚拟生成列上的 BLOB。** `handle_old_value()` 定义为 `(is_flag_set(BLOB_FLAG) || is_array()) && is_virtual_gcol()`，注释说明必须同时保留 new / old / deleted 三份值，因为 InnoDB 不认得旧值。此时 Field 从"一个值的槽"退化为"三个值的容器"。
- **失效 A3 · NDB 的 row_offset。** `get_blob_data(ptrdiff_t row_offset)` 的注释写 "row_offset is only used by NDB"——即"`ptr` 就是当前行的这一列"这个假设对 NDB 不成立，需要额外偏移。
- **失效 A4 · 临时可空。** `m_is_tmp_nullable` / `m_is_tmp_null` 让 NOT NULL 列在触发器场景可短暂为 NULL（注释："This flag is used for trigger handling"）。此时 `ptr` 里的字节不可信，必须先 `is_null()`。

**假设 B："`field_length` 是列宽，`pack_length() == field_length`。"**

- 对字符类型，`field_length` 是**字节数**：`[F1]` 的注释明确"单位是 byte，如 `varchar(12)` 在 utf8mb4 下 `field_length = 12 * mbmaxlen`"。字符数要靠 `char_length() = field_length / charset()->mbmaxlen`，而 `Field_str` 专门缓存了 `char_length_cache`，注释说"dividing by charset()->mbmaxlen can be surprisingly costly compared to the rest of e.g. make_sort_key()"——**一次除法都要缓存，就是这条路径热度的量化证据。**
- 对 VARCHAR，`pack_length() = field_length + length_bytes`（1 或 2），`row_pack_length()` 又等于 `field_length`，`data_length()` 才是实际字节数——**三个长度语义互不相同**，用错一个就是越界或截断。
- 对 BLOB 更极端：`max_data_length() = (1 << (packlength*8)) - 1`，完全由 `packlength` 决定，与 `field_length` 无关。

**假设 C："每一行都有表中全部列的字节，行缓冲长度就是 `reclength`。"**

- **失效 C1 · 虚拟生成列。** `[F5]` 里 `stored_rec_length` 的注释是 "no generated-only generated fields"，与 `reclength` 并存；`Field::stored_in_db == false` 时，`ptr` 仍指向 record[0] 的某个位置，但内容是**待计算**的——必须先求值生成表达式再 `val_*()`。
- **失效 C2 · instant ADD COLUMN。** server 侧可见的只有 `[F6]` 的 `HA_ALTER_INPLACE_INSTANT` 与 `[F12]` 里对 `ALGORITHM=INSTANT` 的调度。**未找到**：在 server 层源码中没有找到 `Field` 针对 instant 列的专门分支或标志位——只能确认 server 未为此改变 `Field` 的接口，补齐发生在 `[F1]` 之外。
- **失效 C3 · 隐藏列。** `[F10]` 的 `enum_hidden_type`（`HT_HIDDEN_SE` = 对 server 完全不可见；`HT_HIDDEN_SQL` = 对 server 可见、对用户隐藏，注释点名用于 functional indexes）对应 `Field::m_hidden` / `is_hidden()`。函数索引的隐藏列对 Field 是普通列，对用户不可见，连报错都要改写——`[F4]` 的 `Item::save_in_field()` 专门插入 `Functional_index_error_handler` 把列名换成索引名。
- **失效 C4 · 默认值不在行里，在另一块缓冲区。** `Field::m_default_val_expr`、`TABLE::gen_def_fields_ptr`、`TABLE_SHARE::gen_def_field_count`，以及 `TABLE::default_values_offset() = s->default_values - record[0]`——默认值是一块与 `record[0]` **平行**的缓冲区，`store_record / restore_record` 的 memcpy 用的是 `s->reclength`，两者不能混用。

---

## 核心实现

### 主链路

把后面七章串起来——**一个 Field 从"被造出来"到"参与一次 INSERT 与一次 SELECT"的完整顺序**：

```
【建表 / 开表期：只做一次】
  dd::Column (DD)
     │ open_table_def() → fill_share_from_dd()：先算 reclength/null_bytes/default_values
     │ fill_columns_from_dd()：逐列 make_field() → set_field_index() → set_hidden()
     ▼
  TABLE_SHARE::field[]（模板，ptr 指向 default_values）
     │ open_table_from_share()：clone() → init() → move_field_offset()
     ▼
  TABLE::field[]（实例，ptr 指向本 TABLE 的 record[0]）
     │                                    ┌── 首次 build_template()：填 mysql_row_templ_t
     │                                    │   （只取 offset/pack_length/null_bit/type 四项）
     ▼                                    ▼
  Field（几何已定）                  handler / InnoDB 模板

【一次 INSERT】
  restore_record(table, s->default_values)        ← 整行灌默认值
     │
  对每个显式给出的列：
     Item::save_in_field(Field*)                  ← 按 result_type() → Field::type() 分派
       ├─ set_notnull()                           ← ★ 必须在 store* 之前（清 NULL 位）
       └─ Field::store*()                         ← 值协议：编码 + 校验 + 返回 status
     （未给出的列：set_default() 或 COPY_INFO::set_function_defaults() 求 DEFAULT (expr)）
     │
  update_generated_write_fields()                 ← 写方向：算生成列
     │
  handler::write_row(uchar *buf)                  ← ★ 参数是 record[0] 裸字节，不是 Field[]
     └─ 引擎按 mysql_row_templ_t 逐项 memcpy：buf + mysql_col_offset

【一次 SELECT（读一行）】
  handler::ha_rnd_next(buf) / ha_index_read_map(buf, ...)
     ├─ rnd_next() / index_read_map()             ← 引擎按模板把行填进 buf
     └─ update_generated_read_fields(buf, table)   ← ★ 返回之后立即补算虚拟列
     │
  表达式取值：
     Item_field::val_*() → field->is_null() → field->val_*()   ← 从 ptr 解码
     （VARCHAR/BLOB 走 buf2 零拷贝；INT/DECIMAL 走 buf1 转换）
     │
  MySQL 行（record[0]）→ 协议发送 / filesort / 临时表
```

**三个贯穿点**：

1. **几何先定、语义后用**：`record[0]` 的偏移在开表期一次性算好（`pack_length_in_rec()` 线性推进），之后所有读写都是"基址 + 偏移"的指针操作，**没有运行期的偏移计算**。
2. **`store` 与 `val_*` 是唯一的语义闸门**：Item 与 Field 之间的全部数据交换收在 4 个 `store*` + 4 个 `val_*` 上；引擎与 Field 之间**完全没有语义交换**（只有 offset / 长度 / NULL 位）。
3. **默认值 / instant / 虚拟列是三种"行里没有真数据"的特例**（见第七章），都靠"**同一套 offset 模型 + 不同来源的字节**"解决，`Field` 的定位代码一行都不用改。

### 一、类继承树与类型系统

> 源码坐标（每个文件名在本文只出现这一次）：类定义与绝大多数内联实现在 `sql/field.h`；非内联实现与 `make_field()` 工厂在 `sql/field.cc`；`enum_field_types` 在 `include/field_types.h`；`Create_field` 在 `sql/create_field.h`；DD 列 → Field 的映射在 `sql/dd_table_share.cc`；打开表时把 share 的字段克隆进 TABLE 在 `sql/table.cc`；`MY_INT32_NUM_DECIMAL_DIGITS` 在 `include/my_inttypes.h`。

#### 1.1 完整继承树

```
Field                                  ← 抽象根：ptr / field_length / null_bit / charset / type() / store() / val_*()
│
├── Create_field_wrapper   (final)     ← CREATE/ALTER 期的"伪 Field"，体内只抱一个 Create_field*，
│                                        让解析器阶段能复用 Field 的接口，但禁止向下转型（is_wrapper_field()）
│
├── Field_num                          ← 数值族：const bool unsigned_flag; const uint8 dec; bool zerofill;
│   │                                    charset() 恒为 my_charset_numeric，derivation 恒 NUMERIC
│   ├── Field_real                     ← result_type()=REAL_RESULT
│   │   ├── Field_decimal      (final) ← MYSQL_TYPE_DECIMAL（5.0 遗留，仅供升级兼容，make_field 仍造得出）
│   │   ├── Field_float        (final) ← MYSQL_TYPE_FLOAT
│   │   └── Field_double       (final) ← MYSQL_TYPE_DOUBLE
│   ├── Field_new_decimal              ← MYSQL_TYPE_NEWDECIMAL，result_type()=DECIMAL_RESULT
│   ├── Field_tiny                     ← MYSQL_TYPE_TINY
│   │   └── Field_year        (final)  ← MYSQL_TYPE_YEAR（复用 1 字节存储，构造时 zerofill=unsigned=true）
│   ├── Field_short          (final)   ← MYSQL_TYPE_SHORT
│   ├── Field_medium         (final)   ← MYSQL_TYPE_INT24
│   ├── Field_long                     ← MYSQL_TYPE_LONG
│   └── Field_longlong                 ← MYSQL_TYPE_LONGLONG
│
├── Field_str                          ← 字符串族：const CHARSET_INFO *field_charset; field_derivation;
│   │                                    char_length_cache（char_length() 的缓存，避免每次除 mbmaxlen）
│   ├── Field_longstr                  ← "值不在 ptr 直接可比"的长串基类：校验/拷贝错误上报、max_data_length()
│   │   ├── Field_string               ← MYSQL_TYPE_STRING（同时吃下 MYSQL_TYPE_VAR_STRING：CHAR/BINARY）
│   │   ├── Field_varstring            ← MYSQL_TYPE_VARCHAR（VARCHAR/VARBINARY，1~2 字节长度头）
│   │   └── Field_blob                 ← MYSQL_TYPE_BLOB 家族（四种 BLOB 共用一个类，靠 packlength 区分）
│   │       ├── Field_geom    (final)  ← MYSQL_TYPE_GEOMETRY（额外带 geometry_type + srid）
│   │       └── Field_json             ← MYSQL_TYPE_JSON（BLOB 的二进制布局 + JSON 语义）
│   │           └── Field_typed_array (final) ← MYSQL_TYPE_TYPED_ARRAY（8.0.17+ 多值索引 / CAST(... AS ... ARRAY)）
│   ├── Field_enum                     ← MYSQL_TYPE_ENUM（TYPELIB *typelib；type() 却返回 MYSQL_TYPE_STRING！）
│   │   └── Field_set        (final)   ← MYSQL_TYPE_SET（位图存储，type() 同样返回 MYSQL_TYPE_STRING）
│   └── Field_null           (final)   ← MYSQL_TYPE_NULL（SELECT NULL 这类纯 NULL 列）
│
├── Field_temporal                     ← 时间族：result_type()=STRING_RESULT，numeric_context 才是 INT/DECIMAL
│   ├── Field_temporal_with_date       ← 含日期：提供日期相关的 cmp / get_date / store_packed
│   │   ├── Field_temporal_with_date_and_time
│   │   │   ├── Field_temporal_with_date_and_timef   ← 抽象：带小数秒，do_save_field_metadata 写 dec
│   │   │   │   ├── Field_timestampf    ← real MYSQL_TYPE_TIMESTAMP2（type() 返回 TIMESTAMP）
│   │   │   │   └── Field_datetimef     ← real MYSQL_TYPE_DATETIME2 （type() 返回 DATETIME）
│   │   │   ├── Field_timestamp         ← 旧 TIMESTAMP（注释明说 "We will be removed eventually"）
│   │   │   └── Field_datetime          ← 旧 DATETIME
│   │   └── Field_newdate               ← MYSQL_TYPE_DATE（real_type()=MYSQL_TYPE_NEWDATE，3 字节打包）
│   └── Field_time_common              ← 只有时间：TIME/TIME(n) 的公共编码
│       ├── Field_time       (final)   ← MYSQL_TYPE_TIME
│       └── Field_timef      (final)   ← real MYSQL_TYPE_TIME2
│
└── Field_bit                          ← MYSQL_TYPE_BIT（result_type()=INT_RESULT）
    └── Field_bit_as_char    (final)   ← 同一个 BIT 列按 CHAR 语义读出（配合 treat_bit_as_char）
```

几个从树形上就能读出来的设计事实：

- **中间基类不是"分类装饰"，而是实打实承接成员**。`Field_num` 持有 `unsigned_flag/dec/zerofill`，`Field_str` 持有 `field_charset/field_derivation/char_length_cache`，`Field_blob` 持有 `packlength + value/old_value/m_blob_backup`。子类不重复定义这些。
- **一个叶子类常常服务多个 SQL 类型**：四种 BLOB（TINY/MEDIUM/LONG/普通）都是 `Field_blob`，靠 `packlength`(1/2/3/4) 区分；`CHAR` 和 `VARCHAR`(FRM 时代的 `VAR_STRING`) 都落进 `Field_string`；`DATE` 落进 `Field_newdate`。
- **`type()` 与"真实物理类型"经常不同**：`Field_enum`/`Field_set` 的 `type()` 返回 `MYSQL_TYPE_STRING`，`Field_timestampf` 返回 `MYSQL_TYPE_TIMESTAMP`——真正区分靠 `real_type()`。
- **`Field_json` 继承 `Field_blob` 而不是另起一支**：JSON 列在 server 层的行内表示就是一个带长度头的 BLOB（二进制 JSONB），`Field_json` 只是在它之上叠加"值必须是合法 JSON"的校验与部分更新能力。

#### 1.2 "68 个子类"这个说法：考证与纠正

统计方式（可直接复现）：在整棵树里找 `^class .* : public Field.*` 形式的类定义。

```text
生产代码（sql/ 下）命中 39 条 —— 全部集中在同一个头文件里，没有任何一个类外流到别的目录：
  Field（根，不算子类）
  Create_field_wrapper, Field_num, Field_str, Field_longstr, Field_real, Field_decimal,
  Field_new_decimal, Field_tiny, Field_short, Field_medium, Field_long, Field_longlong,
  Field_float, Field_double, Field_null, Field_temporal, Field_temporal_with_date,
  Field_temporal_with_date_and_time, Field_temporal_with_date_and_timef, Field_timestamp,
  Field_timestampf, Field_year, Field_newdate, Field_time_common, Field_time, Field_timef,
  Field_datetime, Field_datetimef, Field_string, Field_varstring, Field_blob, Field_geom,
  Field_json, Field_typed_array, Field_enum, Field_set, Field_bit, Field_bit_as_char
⇒ 子类 = 38 个
拆开看：
  · 28 个是"能被 new 出来的具体类型"（就是 make_field() 那个大 switch 里出现的全部类）
  · 10 个不是：Field_num / Field_real / Field_str / Field_longstr / Field_temporal /
    Field_temporal_with_date / Field_temporal_with_date_and_time /
    Field_temporal_with_date_and_timef / Field_time_common 这 9 个是纯中间抽象基类，
    外加 Create_field_wrapper 这个"伪 Field"
```

**结论：8.0.39 里 `Field` 的子类是 38 个，不是 68 个。** 而且这个数字无法用"模板实例化/宏生成"来解释：

1. **整条继承链上一个模板都没有**。`Field_temporal_with_date_and_timef` 看起来"名字被折行"，但它就是个普通类，不是 `template<int N>`。
2. **没有任何宏批量生成 Field 子类**（没有 `DEFINE_FIELD_CLASS(...)` 之类的东西）。
3. 把单测里的 mock 也算上（19 个：`Mock_field_long`/`Mock_field_string`/`Mock_field_varstring`/`Mock_field_timestamp(f)`/`Mock_field_datetime(f)`/`Mock_field_date`/`Mock_field_new_decimal`/`Base_mock_field_{longlong,varstring,blob,json}`/`Mock_dd_field_*`/`Mock_field` 等），也只有 38 + 19 = **57**，仍不到 68。

所以"68"应视为**早期文档里的一个未核实数字**（很可能是把 `enum_field_types` 的枚举项、`Item` 子类、或某一版 5.x 的统计混了进来）。正确说法是"**38 个子类，其中 28 个是可实例化的叶子类型**"。

#### 1.3 `Field` 基类的核心成员：行缓冲中的角色

```cpp
// ---- 行的"我在哪"：ptr 是整章最关键的一个指针 ----
 protected:
  /// Holds the position to the field in record
  uchar *ptr;              // ★ 指向本列在 TABLE::record[0]（或其偏移副本）中的首字节。
                           //   所有 store()/val_*()/cmp() 都围绕它做 memcpy / 字节序转换。
                           //   注意它不是"值指针"而是"缓冲指针"：值就躺在这段裸字节里，
                           //   定长列（int/date/decimal）就地编码，变长列（varchar/blob）这里只放
                           //   长度头 +（blob 才有）指向真实数据的 char*。

 private:
  uchar *m_null_ptr;       // ★ 行缓冲里"存放本列 NULL 标志位的那个字节"的地址。
                           //   NOT NULL 列该成员为 NULL —— 这就是"能不能为 NULL"的物理判据。
                           //   注意它指向的是整行开头那片 null_bits 区域中的某一字节，不是 ptr 附近。

  bool m_is_tmp_nullable;  // NOT NULL 列被临时允许为 NULL（触发器里 BEFORE 触发前短暂置 NULL）
  bool m_is_tmp_null;      // 当前 record[0] 里该列是否处于这种"临时 NULL"
  enum_check_fields m_check_for_truncated_fields_saved;  // 置临时 NULL 那一刻的截断检查模式快照

 protected:
  static uchar dummy_null_buffer;  // ★ 给"语法上 nullable、但本对象无处放 NULL 位"的 Field 兜底
                                   //   （典型：create_tmp_field() 造出来的对象、单测里的裸 Field_*）

 public:
  uchar *get_null_ptr() { return m_null_ptr; }
  TABLE *table;                    // ★ 反向指针：store() 里要查 table->s->db_low_byte_first（字节序）、
                                   //   table->in_use（报错/warning 的 THD）等
  const char *orig_db_name{nullptr};    // 仅临时表非 NULL：该列"来自哪个库"
  const char *orig_table_name{nullptr}; // 同上，用于给客户端发元数据（Send_field / 协议头）
  const char **table_name, *field_name; // table_name 指向 table->alias，field_name 是列名
  LEX_CSTRING comment;             // 列的 COMMENT，只在 CREATE/ALTER 与 SHOW 时参与
```

剖析：`ptr` 与 `m_null_ptr` 是"**一个列的全部物理坐标**"——前者告诉引擎"值写在哪"，后者告诉引擎"NULL 位在哪"。这也解释了为什么 `Field` 对象本身可以被廉价地复制/移动：真正的数据永远在 `TABLE::record[]` 里，`Field` 只是一组"偏移量 + 语义"。

```cpp
  Key_map key_start;          /* Keys that starts with this field */
  Key_map part_of_key;        ///< Keys that includes this field except of prefix keys.
  Key_map part_of_prefixkey;  ///< Prefix keys
  Key_map part_of_sortkey;    /* ^ but only keys usable for sorting */
  Key_map part_of_key_not_extended;  // 含本列、但没被引擎偷偷补上主键列的那些索引
                                     // （InnoDB 二级索引会隐式追加 PK 列，这里把它排掉）
```

剖析：五个 `Key_map` 是**优化器专用的位图缓存**，在建 `TABLE_SHARE` 时算好，后续"这列能否走索引/能否用索引排序/做覆盖索引"全靠它们做 O(1) 判断，避免每次遍历 `KEY` 数组。

```cpp
  // Max width for a VARCHAR column, in number of bytes
  static constexpr size_t MAX_VARCHAR_WIDTH{65535};
  // Maximum sizes of the four BLOB types, in number of bytes
  static constexpr size_t MAX_TINY_BLOB_WIDTH{255};
  static constexpr size_t MAX_SHORT_BLOB_WIDTH{65535};
  static constexpr size_t MAX_MEDIUM_BLOB_WIDTH{16777215};
  static constexpr size_t MAX_LONG_BLOB_WIDTH{4294967295};

  // Length of field. Never write to this member directly; instead, use set_field_length().
  // 当前字符集下最大字节数，单位为 byte。如 (c varchar(12))，Field::field_length = 12 * mbmaxlen
  uint32 field_length;                                   // ★ 注意单位是"字节"不是"字符"！
  virtual void set_field_length(uint32 length) { field_length = length; }
                                   // 走虚函数的理由：Field_str 需要在改长度时同步刷新 char_length_cache
 private:
  uint32 flags{0};                 // NOT_NULL_FLAG / BINARY_FLAG / UNSIGNED_FLAG / ZEROFILL_FLAG ...
  uint16 m_field_index;            // 本列在 TABLE::field[] 中的下标
 public:
  uchar null_bit;                  // ★ 不是"第几位"而是"已移位后的掩码"：1<<n
                                   //   判空时是 (*m_null_ptr & null_bit)
  uchar auto_flags;                // AUTO_INCREMENT / DEFAULT NOW / ON UPDATE NOW / 表达式默认值
  bool is_created_from_null_item;  // 该列是从 NULL 常量推出来的（临时表），类型只是"猜测"
  bool m_indexed;                  // 该列被任何索引包含（哪怕只是前缀）
  LEX_CSTRING m_engine_attribute = EMPTY_CSTR;
  LEX_CSTRING m_secondary_engine_attribute = EMPTY_CSTR;

 public:
  Value_generator *gcol_info{nullptr};   // ★ 非空即生成列（GC），表达式与依赖都挂在这里
  bool stored_in_db;                     // ★ VIRTUAL(false) / STORED(true) 的唯一判别位
  bool is_gcol() const { return gcol_info; }
  bool is_virtual_gcol() const { return gcol_info && !stored_in_db; }
  Value_generator *m_default_val_expr{nullptr};  // DEFAULT (expr) 的求值器（8.0.13+）
```

```cpp
// ---- 构造函数：一眼看清哪些成员是"从外部注入的坐标" ----
Field::Field(uchar *ptr_arg, uint32 length_arg, uchar *null_ptr_arg,
             uchar null_bit_arg, uchar auto_flags_arg,
             const char *field_name_arg)
    : ptr(ptr_arg),                                  // ← 行的位置
      m_hidden(dd::Column::enum_hidden_type::HT_VISIBLE),
      m_null_ptr(null_ptr_arg),                      // ← NULL 位的位置
      m_is_tmp_nullable(false), m_is_tmp_null(false),
      m_check_for_truncated_fields_saved(CHECK_FIELD_IGNORE),
      table(nullptr), table_name(nullptr),
      field_name(field_name_arg),
      field_length(length_arg),                      // ← 字节长度
      null_bit(null_bit_arg),
      auto_flags(auto_flags_arg),
      is_created_from_null_item(false),
      m_indexed(false), m_warnings_pushed(0),
      gcol_info(nullptr), stored_in_db(true),
      m_default_val_expr(nullptr) {}

void Field::init(TABLE *table_arg) {                 // 克隆/建表后回填宿主表
  table = table_arg;
  table_name = &table_arg->alias;
}

// ---- NULL 判定 ----
  bool is_null(ptrdiff_t row_offset = 0) const {
    /* 可空列：看 m_null_ptr[row_offset] 的那个位；
       不可空列：看 TABLE::has_null_row()——外连接的 NULL 补行、或"有聚合无 GROUP BY 且无命中行"
       的隐式分组，这两种情况整行被视作全 NULL。测试顺序不能换。 */
  }

// ---- 移动：为什么 Field 可以被"整体搬移" ----
  void move_field(uchar *ptr_arg, uchar *null_ptr_arg, uchar null_bit_arg) {
    ptr = ptr_arg; m_null_ptr = null_ptr_arg; null_bit = null_bit_arg;
  }
  virtual void move_field_offset(ptrdiff_t ptr_diff) {
    ptr += ptr_diff;                                  // 值指针整体平移
    if (is_nullable()) m_null_ptr += ptr_diff;        // NULL 位也一起平移
  }
```

剖析：`move_field_offset()` 是"**一份 `TABLE_SHARE` 的字段描述，服务 N 个 `TABLE` 实例**"的关键。每个 `TABLE` 有自己的 `record[0]`，字段对象被克隆后只需把两个指针各加同一个差值，就能指向自己那套缓冲。

#### 1.4 `type()` / `enum_field_types` / `sql_type()`

```cpp
enum enum_field_types {          // 协议 + server 层共用的类型码，注意它是"稀疏"的
  MYSQL_TYPE_DECIMAL, MYSQL_TYPE_TINY, MYSQL_TYPE_SHORT, MYSQL_TYPE_LONG,
  MYSQL_TYPE_FLOAT, MYSQL_TYPE_DOUBLE, MYSQL_TYPE_NULL, MYSQL_TYPE_TIMESTAMP,
  MYSQL_TYPE_LONGLONG, MYSQL_TYPE_INT24, MYSQL_TYPE_DATE, MYSQL_TYPE_TIME,
  MYSQL_TYPE_DATETIME, MYSQL_TYPE_YEAR,
  MYSQL_TYPE_NEWDATE,   /**< Internal to MySQL. Not used in protocol */
  MYSQL_TYPE_VARCHAR, MYSQL_TYPE_BIT,
  MYSQL_TYPE_TIMESTAMP2,
  MYSQL_TYPE_DATETIME2, /**< Internal to MySQL. Not used in protocol */
  MYSQL_TYPE_TIME2,     /**< Internal to MySQL. Not used in protocol */
  MYSQL_TYPE_TYPED_ARRAY, /**< Used for replication only */
  MYSQL_TYPE_INVALID = 243, MYSQL_TYPE_BOOL = 244,   /* Currently just a placeholder */
  MYSQL_TYPE_JSON = 245, MYSQL_TYPE_NEWDECIMAL = 246, MYSQL_TYPE_ENUM = 247,
  MYSQL_TYPE_SET = 248, MYSQL_TYPE_TINY_BLOB = 249, MYSQL_TYPE_MEDIUM_BLOB = 250,
  MYSQL_TYPE_LONG_BLOB = 251, MYSQL_TYPE_BLOB = 252, MYSQL_TYPE_VAR_STRING = 253,
  MYSQL_TYPE_STRING = 254, MYSQL_TYPE_GEOMETRY = 255
};
```

前 0~24 是**协议可见**的老类型，243 起是**只在 server 内部/复制里流转**的新类型；中间 25~242 是一大片空洞——这直接导致后面要有一张"稀疏 → 稠密"的下标映射表（`field_type2index`）。

```cpp
  virtual enum_field_types type() const = 0;                    // ★ 逻辑类型
  virtual enum_field_types real_type() const { return type(); } // ★ 物理类型，行内真实编码
  virtual enum_field_types binlog_type() const {
    /* Binlog 默认存 field->type()：CHAR/VARCHAR/SET/ENUM 统一记 MYSQL_TYPE_STRING，细节另存 metadata。
       但带小数秒的 TIME(n)/DATETIME(n)/TIMESTAMP(n) 不能这么干——因为 MYSQL_TYPE_TIME/DATETIME
       这两种 binlog 记录格式不带 metadata，精度会丢。所以这几类必须存 real_type()（TIME2/
       DATETIME2/TIMESTAMP2），精度写进 metadata。*/
    return type();
  }
```

各叶子类的 `type()` / `real_type()` 对照（源码核实）：

| 类 | `type()` | `real_type()` | 说明 |
|---|---|---|---|
| `Field_tiny` / `short` / `medium` / `long` / `longlong` | `TINY/SHORT/INT24/LONG/LONGLONG` | 同 | 整数族一一对应 |
| `Field_float` / `double` / `decimal` | `FLOAT/DOUBLE/DECIMAL` | 同 | |
| `Field_new_decimal` | `NEWDECIMAL` | 同 | 注释点明 "NEWDECIMAL is a type, not only a real_type" |
| `Field_year` | `YEAR` | 同 | |
| `Field_string` | `STRING` | `STRING` | CHAR/BINARY |
| `Field_varstring` | `VARCHAR` | `VARCHAR` | |
| `Field_blob` | `BLOB` | — | ★四种 BLOB 全是 `BLOB`，靠 `packlength` 区分 |
| `Field_geom` / `Field_json` | `GEOMETRY` / `JSON` | 同 | |
| `Field_enum` | **`STRING`** | `ENUM` | ★ 逻辑类型被"降级"成 STRING |
| `Field_set` | **`STRING`** | `SET` | 同上 |
| `Field_time` / `Field_timef` | `TIME` | `TIME` / `TIME2` | |
| `Field_datetime` / `datetimef` | `DATETIME` | `DATETIME` / `DATETIME2` | |
| `Field_timestamp` / `timestampf` | `TIMESTAMP` | `TIMESTAMP` / `TIMESTAMP2` | |
| `Field_newdate` | `DATE` | `NEWDATE` | |
| `Field_bit` | `BIT` | 同 | |

```cpp
inline enum_field_types real_type_to_type(enum_field_types real_type) {
  switch (real_type) {
    case MYSQL_TYPE_TIME2:       return MYSQL_TYPE_TIME;
    case MYSQL_TYPE_DATETIME2:   return MYSQL_TYPE_DATETIME;
    case MYSQL_TYPE_TIMESTAMP2:  return MYSQL_TYPE_TIMESTAMP;
    case MYSQL_TYPE_NEWDATE:     return MYSQL_TYPE_DATE;
    default:                     return real_type;
  }
}
```

**`sql_type()`**（给 `SHOW CREATE TABLE` / `DESCRIBE` / 错误信息用）：

```cpp
  /* Caller beware: sql_type can change str.Ptr, so check ptr() to see if it changed ... */
  virtual void sql_type(String &str) const = 0;      // 纯虚，每个具体类自己拼
```

整数族共用一个小工具，它解释了 `int(11)` 里那个"11"从哪来：

```cpp
static void integer_sql_type(const Field_num *field, const char *type_name, String *res) {
  res->length(0);
  res->append(type_name);                                  // "int" / "tinyint" / "bigint" ...
  if (field->zerofill) res->append_parenthesized(field->field_length);  // ★ 只有 ZEROFILL 才显示宽度
  append_zerofill_and_unsigned(field, res);                // 补 " unsigned" / " zerofill"
}
void Field_long::sql_type(String &res) const { integer_sql_type(this, "int", &res); }
```

字符串族处理"字符 vs 字节"换算与 binary/char 分流：

```cpp
void Field_varstring::sql_type(String &res) const {
  const CHARSET_INFO *cs = res.charset();
  size_t length = cs->cset->snprintf(cs, res.ptr(), res.alloced_length(), "%s(%d)",
                              (has_charset() ? "varchar" : "varbinary"),   // ★ 非 binary → varchar
                              (int)field_length / charset()->mbmaxlen);    // ★ 字节 → 字符
  res.length(length);
}
```

`Field_blob` 靠 `packlength` 反推 `tiny/med/long`——"四种 BLOB 一个类"最直观的证据：

```cpp
void Field_blob::sql_type(String &res) const {
  const char *str; uint length;
  switch (packlength) {           // 长度头占几字节 ⇒ 哪种 BLOB
    default:     str = "tiny";   length = 4; break;   // 1 字节 → TINY
    case 2:      str = "";       length = 0; break;   // 2 字节 → 普通
    case 3:      str = "medium"; length = 6; break;
    case 4:      str = "long";   length = 4; break;
  }
  res.set_ascii(str, length);
  if (charset() == &my_charset_bin) res.append(STRING_WITH_LEN("blob"));  // binary → blob
  else                              res.append(STRING_WITH_LEN("text"));  // 有字符集 → text
}
```

#### 1.5 `result_type()` 与 `Item_result`

```cpp
  virtual Item_result result_type() const = 0;               // 纯虚
  /* 在数值上下文里当什么算（SELECT time_col + 1; SELECT SUM(time_col);）：
     - TIME/DATETIME/TIMESTAMP 当 INT
     - TIME(1)/DATETIME(1)/TIMESTAMP(1) 当 DECIMAL（1 位小数） */
  virtual Item_result numeric_context_result_type() const { return result_type(); }
  virtual Item_result cmp_type() const        { return result_type(); }
  virtual Item_result cast_to_int_type() const{ return result_type(); }
```

映射规则（两层）：

```cpp
// 第一层：基类给出族默认值，子类只在必要时改写
class Field_num : public Field {
  Item_result result_type() const override { return REAL_RESULT; }   // ★ 数值族默认是 REAL
};
// 每个具体整数类压回 INT_RESULT（TINY/SHORT/MEDIUM/LONG/LONGLONG/YEAR/BIT 全部如此）
enum Item_result result_type() const final { return INT_RESULT; }

class Field_str : public Field {
  Item_result result_type() const override { return STRING_RESULT; }
  Item_result numeric_context_result_type() const final { return REAL_RESULT; }  // ★ 字符串参与算术当 REAL
};
enum Item_result result_type() const final { return DECIMAL_RESULT; }  // Field_new_decimal
enum Item_result result_type() const final { return STRING_RESULT; }   // Field_temporal（时间族！）
  Item_result numeric_context_result_type() const final {
    return dec ? DECIMAL_RESULT : INT_RESULT;    // 有小数秒 → DECIMAL，否则 INT
  }
enum Item_result result_type() const final { return INT_RESULT; }      // Field_bit
```

```cpp
// 第二层：只有一个 enum_field_types 时（Create_field_wrapper、数组元素类型）查静态表
static Item_result field_types_result_type[FIELDTYPE_NUM] = {
    DECIMAL_RESULT, INT_RESULT,        // DECIMAL, TINY
    INT_RESULT,     INT_RESULT,        // SHORT, LONG
    REAL_RESULT,    REAL_RESULT,       // FLOAT, DOUBLE
    STRING_RESULT,  STRING_RESULT,     // NULL, TIMESTAMP
    INT_RESULT,     INT_RESULT,        // LONGLONG, INT24
    STRING_RESULT,  STRING_RESULT,     // DATE, TIME
    STRING_RESULT,  INT_RESULT,        // DATETIME, ★ YEAR（注意：YEAR 是 INT 而 DATE 是 STRING）
    STRING_RESULT,  STRING_RESULT,     // NEWDATE, VARCHAR
    INT_RESULT,     INVALID_RESULT,    // BIT, INVALID
    // Unused entries: <17>-<242>     ← 枚举稀疏，表却是稠密的
    INT_RESULT,     STRING_RESULT,     // BOOL, JSON
    DECIMAL_RESULT, STRING_RESULT,     // NEWDECIMAL, ENUM
    STRING_RESULT,  STRING_RESULT,     // SET, TINY_BLOB
    ...
};

// 稀疏枚举 → 稠密下标
inline int field_type2index(enum_field_types field_type) {
  field_type = real_type_to_type(field_type);
  return (field_type < FIELDTYPE_TEAR_FROM
              ? field_type
              : ((int)FIELDTYPE_TEAR_FROM) + (field_type - FIELDTYPE_TEAR_TO) - 1);
}
```

★ **时间族是唯一的"双重人格"**：`result_type()` 是 `STRING_RESULT`（`SELECT ts_col` 返回 `'2024-01-01 00:00:00'` 这种串），但 `numeric_context_result_type()` 是 `INT_RESULT`/`DECIMAL_RESULT`（`SELECT ts_col + 1` 走数值路径）。所以**任何"这列是什么结果类型"的判断都必须先问清上下文**——这正是三个函数并存的原因。

#### 1.6 `clone()`：为什么需要克隆，克隆了什么

```cpp
  /**
     Makes a shallow copy of the Field object.
     @note This member function must be overridden in all concrete subclasses.
     Several of the Field subclasses are concrete even though they are not leaf
     classes, so the compiler will not always catch this.
   */
  virtual Field *clone(MEM_ROOT *mem_root) const = 0;   // 纯虚：每个具体类都必须给出协变版本
```

★ 注释里藏着一个坑：**`Field_bit`、`Field_new_decimal`、`Field_string` 这些"既是具体类、又有子类"的中间类，编译器不会因为子类忘了改写而报错**，所以每个类都得手写一遍：

```cpp
  Field_long *clone(MEM_ROOT *mem_root) const final {
    assert(type() == MYSQL_TYPE_LONG);       // 断言：clone 出来的必须同类型（防复制粘贴出错）
    return new (mem_root) Field_long(*this); // MEM_ROOT 上 placement new
  }
  Field_varstring *clone(MEM_ROOT *mem_root) const final {
    assert(type() == MYSQL_TYPE_VARCHAR);
    assert(real_type() == MYSQL_TYPE_VARCHAR);
    return new (mem_root) Field_varstring(*this);
  }
```

基类里那句决定性的注释——**拷贝构造函数就是默认的，没有深拷贝**：

```cpp
class Field {
  /* 原本想让 Field 不可拷贝，但当年写错了一个 typo，没能强制；如今大量代码在拷贝 Field，
     所以默认拷贝构造必须保留（赋值运算符被禁掉了），但最好不要写依赖它的新代码。 */
  Field(const Field &) = default;
  void operator=(Field &) = delete;
```

**为什么需要它**——`open_table_from_share()` 为每个 `TABLE` 实例造一套自己的字段对象：

```cpp
  /* We will create fields by cloning TABLE_SHARE's fields; then we will need
     to make all new fields' pointers point into the new TABLE's record[0], by
     applying an offset to them. */
  move_offset = outparam->record[0] - share->default_values + ...;

  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);   // ★ 一个 TABLE 一套 Field 对象
    *field_ptr = new_field;
    new_field->init(outparam);                        // 回填 table / table_name
    new_field->move_field_offset(move_offset);        // ★ ptr 与 m_null_ptr 平移到本 TABLE 的 record[0]
    if (!has_default_values) new_field->reset();
```

**克隆了什么 / 没克隆什么**：

- **克隆了**：`Field` 对象自身（含 `ptr`、`m_null_ptr`、`null_bit`、`field_length`、`charset` 指针、flags、`gcol_info` 指针、五个 Key_map…），并且 `new (mem_root) Field_long(*this)` 把**虚函数表指针修正成正确的派生类**——`clone()` 返回 `Field*`，但 `store()`/`val_str()` 必须落到正确的子类实现。
- **没克隆**：`record[]` 里的实际数据、`TYPELIB`（ENUM/SET 元素表）、`Value_generator` 指向的表达式树——只拷指针、共享。
- **必须事后修正**：`ptr` / `m_null_ptr` 仍指向 `share->default_values`，所以要 `init()` + `move_field_offset()`。

#### 1.7 `max_display_length()`：各类型"最多能显示多长"

```cpp
  /* maximum possible display length */
  virtual uint32 max_display_length() const = 0;   // 纯虚
```

规则一览（**单位仍是字节**）：

| 类 | 返回值 | 含义 |
|---|---|---|
| `Field_str`（string/varstring/enum/set 继承） | `field_length` | 声明的最大字节数（`varchar(12)` utf8mb4 ⇒ 48） |
| `Field_real`、`Field_new_decimal` | `field_length` | |
| `Field_tiny` | `4` | −128 |
| `Field_short` | `6` | −32768 |
| `Field_medium` | `8` | −8388608 |
| `Field_long` | `MY_INT32_NUM_DECIMAL_DIGITS`（= **11U**） | 10 位数字 + 1 位符号 |
| `Field_longlong` | `20` | |
| `Field_temporal` 各时态类 | `field_length` | 如 DATETIME 的 19、DATE 的 10 |
| `Field_null` | `4` | 字面量 `"NULL"` |
| `Field_bit` | `field_length` | |
| `Create_field_wrapper` | `m_field->max_display_width_in_codepoints()` | 按"字符"计 |

唯一写成函数体的（按长度头字节数分档）：

```cpp
uint32 Field_blob::max_display_length() const {
  switch (packlength) {
    case 1:  return 255 * field_charset->mbmaxlen;        // TINYBLOB/TINYTEXT
    case 2:  return 65535 * field_charset->mbmaxlen;      // BLOB/TEXT
    case 3:  return 16777215 * field_charset->mbmaxlen;   // MEDIUMBLOB/MEDIUMTEXT
    case 4:  return (uint32)4294967295U;                  // LONGBLOB/LONGTEXT（已到 uint32 上限）
    default: assert(0); return 0;
  }
}
```

`max_display_length()` 服务于**协议元数据**（`Send_field::length`）与优化器的上界估算，它**不是**"这一行的实际长度"（实际是 `data_length()`）。口径是"最坏情况下转文本要多少字节"，因此带符号位（11 而非 10）、按字节而非字符。

#### 1.8 类型创建的入口：`Create_field` / DD 列定义 → `Field`

三条入口最终汇到**同一个大 switch**——`make_field(MEM_ROOT*, TABLE_SHARE*, ...)`：

```cpp
Field *make_field(MEM_ROOT *mem_root, TABLE_SHARE *share, uchar *ptr,
                  size_t field_length, uchar *null_pos, uchar null_bit,
                  enum_field_types field_type, const CHARSET_INFO *field_charset, ...) {
  if (field_type == MYSQL_TYPE_BIT && !treat_bit_as_char) {
    bit_ptr = null_pos; bit_offset = null_bit;      // ★ BIT 特例：BIT 会"偷用" NULL 位那片字节
    if (is_nullable) { bit_ptr += (null_bit == 7); bit_offset = (bit_offset + 1) & 7; }
  }

  if (!is_nullable) { null_pos = nullptr; null_bit = 0; }   // 不可空：抹掉 NULL 坐标
  else              { null_bit = ((uchar)1) << null_bit; }  // ★ 把"第几位"变成"掩码"

  if (is_temporal_real_type(field_type)) field_charset = &my_charset_numeric;
      // ★ 时间列在行内是打包的二进制，字符集一律强制 numeric

  switch (field_type) {
    case MYSQL_TYPE_VAR_STRING:
    case MYSQL_TYPE_STRING:
      return new (mem_root) Field_string(ptr, field_length, null_pos, null_bit,
                                         auto_flags, field_name, field_charset);
    case MYSQL_TYPE_VARCHAR:
      return new (mem_root) Field_varstring(
          ptr, field_length, HA_VARCHAR_PACKLENGTH(field_length), null_pos,   // ★ 长度头 1 还是 2
          null_bit, auto_flags, field_name, share, field_charset);
    case MYSQL_TYPE_BLOB: case MYSQL_TYPE_MEDIUM_BLOB:
    case MYSQL_TYPE_TINY_BLOB: case MYSQL_TYPE_LONG_BLOB: {
      /* Field_blob 的构造参数不是真正的 pack_length，而是"表示长度的字节数" */
      uint pack_length = calc_pack_length(field_type, field_length) - portable_sizeof_char_ptr;
      return new (mem_root) Field_blob(ptr, null_pos, null_bit, auto_flags, field_name,
                                       share, pack_length, field_charset);
    }
    case MYSQL_TYPE_JSON: { /* 同上：JSON 也是 blob 布局 */ ... }
    case MYSQL_TYPE_ENUM:
      assert(interval);                              // ENUM 必须有 TYPELIB（元素表）
      return new (mem_root) Field_enum(..., get_enum_pack_length(interval->count), interval, field_charset);
    case MYSQL_TYPE_SET:  /* 对称，用 get_set_pack_length(interval->count) */
    case MYSQL_TYPE_NEWDECIMAL: return new (mem_root) Field_new_decimal(...);
    case MYSQL_TYPE_LONG:       return new (mem_root) Field_long(...);
    case MYSQL_TYPE_TIMESTAMP2: // 精度由 field_length 反推：> MAX_DATETIME_WIDTH 才说明带小数秒
      return new (mem_root) Field_timestampf(..., field_length > MAX_DATETIME_WIDTH
                                               ? field_length - 1 - MAX_DATETIME_WIDTH : 0);
    case MYSQL_TYPE_BIT:      return treat_bit_as_char
                                ? new (mem_root) Field_bit_as_char(...)
                                : new (mem_root) Field_bit(ptr, field_length, null_pos, null_bit,
                                                           bit_ptr, bit_offset, auto_flags, field_name);
    // … 其余类型同理
  }
}
```

**DD 路径**：`open_table_def()` 先把 `thd->mem_root` 临时切成 `share->mem_root`（注释直说 "Needed for make_field()"），然后逐列把 `dd::Column` 翻译成工厂参数：

```cpp
static Field *make_field(const dd::Column &col_obj, const CHARSET_INFO *charset,
                         TABLE_SHARE *share, uchar *ptr, uchar *null_pos, size_t null_bit) {
  auto field_type   = dd_get_old_field_type(col_obj.type());   // DD 类型码 → enum_field_types
  auto field_length = col_obj.char_length();                    // DD 存的是"字符长度"
  if (field_type == MYSQL_TYPE_ENUM || field_type == MYSQL_TYPE_SET) {
    // 现场在 share->mem_root 上重建 TYPELIB：元素个数、名字数组（末尾 nullptr）、长度数组
    interval = (TYPELIB *)share->mem_root.Alloc(sizeof(TYPELIB));
    interval->type_names[interval_parts] = nullptr; ...
  }
  if (field_type == MYSQL_TYPE_DECIMAL || field_type == MYSQL_TYPE_NEWDECIMAL)
    decimals = col_obj.numeric_scale();                        // DECIMAL 必填 scale
  return make_field(*THR_MALLOC, share, ptr, field_length, null_pos, null_bit, ...);
}
```

**三条路径的收束关系**：

```text
① CREATE / ALTER / 临时表：  解析器 → Create_field → make_field(const Create_field&, ...) ─┐
② 打开已存在的表：           dd::Column → make_field(const dd::Column&, ...)             ─┼→ make_field(MEM_ROOT*, ...)
                                                                                          │   （唯一的大 switch）
                                                                                          └→ new (mem_root) Field_xxx(...)
③ 后续每个 TABLE 实例：      TABLE_SHARE::field[] → Field::clone() → init() + move_field_offset()
```

要点：`Create_field` 是**"声明期"**的列描述（来自 SQL 文本），`Field` 是**"运行期"**的列对象（带 `ptr`/`null_ptr` 行内坐标）。二者不是继承关系，只有 `Create_field_wrapper` 为复用 `Field` 接口才把 `Create_field*` 包进继承树。

### 二、值协议：`store` ↔ `val_*` 的对偶

> 引用标记沿用本章章首的源码坐标表（`[F1]`…`[F4]`），此处不再重复文件名。

#### 2.1 协议总览：一条记录的两个方向

`Field` 是**记录缓冲区（`TABLE::record[0]` 中 `ptr` 指向的那段裸内存）与 SQL 语义值之间的唯一闸门**。它对外只暴露两组方法：

- **写方向 `store*`**：把 `Item` 求出的语义值（字符串 / double / longlong / `my_decimal` / `MYSQL_TIME`）**编码**进 `ptr`；
- **读方向 `val_*`**：把 `ptr` 里的物理字节**解码**成 `Item` 需要的语义值。

这两组方法构成对偶：`store` 的返回值是 `type_conversion_status`（写入是否失真），而 `val_*` 没有错误通道——因为**所有合法性裁决都发生在写入时**。这是 MySQL "写入即校验" 设计的核心：一行数据一旦进入记录缓冲区，就保证是合法的，`val_*` 只需要无条件解码。

#### 2.2 `store` 的全部重载与返回类型

##### 纯虚函数组（[F1]）

```cpp
  /* Store functions returns 1 on overflow and -1 on fatal error */
  virtual type_conversion_status store(const char *to, size_t length,
                                       const CHARSET_INFO *cs) = 0;
  virtual type_conversion_status store(double nr) = 0;
  virtual type_conversion_status store(longlong nr, bool unsigned_val) = 0;
  /**
    Store a temporal value in packed longlong format into a field.
    The packed value is compatible with TIME_to_longlong_time_packed(),
    TIME_to_longlong_date_packed() or TIME_to_longlong_datetime_packed().
    ...
  */
  virtual type_conversion_status store_packed(longlong nr) {
    return store(nr, false);
  }
  virtual type_conversion_status store_decimal(const my_decimal *d) = 0;
  /**
    Store MYSQL_TIME value with the given amount of decimal digits into a field.
    Note, the "dec" parameter represents number of digits of the Item that
    previously created the MYSQL_TIME value. It's needed when we store the value
    into a CHAR/VARCHAR/TEXT field to display the proper amount of fractional
    digits. For other field types the "dec" value does not matter and is ignored.
  */
  virtual type_conversion_status store_time(MYSQL_TIME *ltime, uint8 dec_arg);
  type_conversion_status store_time(MYSQL_TIME *ltime) {
    return store_time(ltime, 0);
  }
  type_conversion_status store(const char *to, size_t length,
                               const CHARSET_INFO *cs,
                               enum_check_fields check_level);
```

就近剖析：

- 四个 `= 0` 是**协议骨架**：任何 `Field` 子类必须能接受"字符串、双精度、整数、定点小数"四种输入形态，因为 `Item::save_in_field_inner()` 的通用分派表只会用这四种形态喂进来。
- `store_packed(longlong)` 不是纯虚：默认退化为 `store(nr, false)`，只有时态类型（`Field_datetimef`、`Field_time`、`Field_newdate`、`Field_timestampf`）才覆写它直接用 packed 编码落盘，避免"packed → MYSQL_TIME → packed"的往返。
- `store_time()` 的 `dec_arg` 只对**字符串类字段**有意义——因为默认实现是把 `MYSQL_TIME` 格式化成文本再走 `store(const char*, ...)`。
- 最后一个四参数 `store()` 是**带检查级别**的入口（见 2.2.3）。

##### 返回类型 `type_conversion_status`（[F1]）

```cpp
/**
  Status when storing a value in a field or converting from one
  datatype to another. The values should be listed in order of
  increasing seriousness so that if two type_conversion_status
  variables are compared, the bigger one is most serious.
*/
enum type_conversion_status {
  /// Storage/conversion went fine.
  TYPE_OK = 0,
  /// A minor problem when converting between temporal values, e.g.
  /// if datetime is converted to date the time information is lost.
  TYPE_NOTE_TIME_TRUNCATED,
  /// Value was stored, but something was cut. What was cut is
  /// considered insignificant enough to only issue a note. Example:
  /// trying to store a number with 5 decimal places into a field that
  /// can only store 3 decimals. ... Another example: storing the string
  /// "foo " into a VARCHAR(3). The string "foo" is stored in this case.
  TYPE_NOTE_TRUNCATED,
  /// Value outside min/max limit of datatype. The min/max value is
  /// stored by Field::store() instead (if applicable)
  TYPE_WARN_OUT_OF_RANGE,
  /// Value was stored, but something was cut. What was cut is
  /// considered significant enough to issue a warning. Example: storing
  /// the string "foo" into a VARCHAR(2). The string "fo" is stored. ...
  TYPE_WARN_TRUNCATED,
  /// Value has invalid string data. When present in a predicate with
  /// equality operator, range optimizer returns an impossible where.
  TYPE_WARN_INVALID_STRING,
  /// Trying to store NULL in a NOT NULL field.
  TYPE_ERR_NULL_CONSTRAINT_VIOLATION,
  /// Store/convert incompatible values, like converting "foo" to a date.
  TYPE_ERR_BAD_VALUE,
  /// Out of memory
  TYPE_ERR_OOM
};
```

就近剖析：

- **这个枚举是有序的**：注释明确要求"按严重性递增排列"，于是代码里可以直接写 `if (tmp_error > error) error = tmp_error;`（见 `Field_temporal::store`）来取更严重者。这是 MySQL 里少见的"靠枚举顺序做代数运算"的技巧。
- 语义分三档：`TYPE_*_NOTE_*` 只出 Note；`TYPE_WARN_*` 出 Warning 但**值已写进去**；`TYPE_ERR_*` 表示值没写进去或语义上不可能。
- `TYPE_WARN_OUT_OF_RANGE` 的语义是"**夹断后写入**"：写的是 min/max，不是原值。
- `TYPE_WARN_INVALID_STRING` 除报 warning 外还有特殊用途：range optimizer 见到它就把等值谓词判定为 `impossible where`。

两个把底层错误码翻译成该枚举的适配器（[F1]）：

```cpp
inline type_conversion_status decimal_err_to_type_conv_status(int dec_error) {
  if (dec_error & E_DEC_OOM) return TYPE_ERR_OOM;
  if (dec_error & (E_DEC_DIV_ZERO | E_DEC_BAD_NUM)) return TYPE_ERR_BAD_VALUE;
  if (dec_error & E_DEC_TRUNCATED) return TYPE_NOTE_TRUNCATED;
  if (dec_error & E_DEC_OVERFLOW) return TYPE_WARN_OUT_OF_RANGE;
  if (dec_error == E_DEC_OK) return TYPE_OK;
  assert(false);
  return TYPE_ERR_BAD_VALUE;
}

inline type_conversion_status time_warning_to_type_conversion_status(const int warn) {
  if (warn & MYSQL_TIME_NOTE_TRUNCATED) return TYPE_NOTE_TIME_TRUNCATED;
  if (warn & MYSQL_TIME_WARN_OUT_OF_RANGE) return TYPE_WARN_OUT_OF_RANGE;
  if (warn & MYSQL_TIME_WARN_TRUNCATED) return TYPE_NOTE_TRUNCATED;
  if (warn & (MYSQL_TIME_WARN_ZERO_DATE | MYSQL_TIME_WARN_ZERO_IN_DATE))
    return TYPE_ERR_BAD_VALUE;
  if (warn & MYSQL_TIME_WARN_INVALID_TIMESTAMP)
    // date was fine but pointed to daylight saving time switch gap
    return TYPE_OK;
  assert(!warn);
  return TYPE_OK;
}
```

就近剖析：`MYSQL_TIME_WARN_INVALID_TIMESTAMP`（落在 DST 跳变空洞里的时刻）被翻译成 `TYPE_OK`——值是"合法但不存在"的，已被规范化为别的时刻。

##### 带检查级别的入口（[F2]）

```cpp
type_conversion_status Field::store(const char *to, size_t length,
                                    const CHARSET_INFO *cs,
                                    enum_check_fields check_level) {
  THD *thd = current_thd;
  enum_check_fields old_check_level = thd->check_for_truncated_fields;
  thd->check_for_truncated_fields = check_level;
  const type_conversion_status res = store(to, length, cs);
  thd->check_for_truncated_fields = old_check_level;
  return res;
}
```

就近剖析：这是一个 **RAII 式的作用域切换**。截断告警策略不是参数层层传递，而是挂在 `THD` 上的全局状态 `check_for_truncated_fields`；这个包装函数临时改写它、调虚函数、再恢复。

#### 2.3 具体子类的 `store` 实现

##### `Field_long::store(longlong, bool)`（[F2]）

```cpp
type_conversion_status Field_long::store(longlong nr, bool unsigned_val) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  type_conversion_status error = TYPE_OK;
  int32 res;

  if (is_unsigned()) {
    if (nr < 0 && !unsigned_val) {
      res = 0;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else if ((ulonglong)nr >= (1LL << 32)) {
      res = (int32)(uint32)~0L;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else
      res = (int32)(uint32)nr;
  } else {
    if (nr < 0 && unsigned_val) {
      nr = ((longlong)INT_MAX32) + 1;  // Generate overflow
      error = TYPE_WARN_OUT_OF_RANGE;
    }
    if (nr < (longlong)INT_MIN32) {
      res = (int32)INT_MIN32;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else if (nr > (longlong)INT_MAX32) {
      res = (int32)INT_MAX32;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else
      res = (int32)nr;
  }
  if (error)
    set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);

  if (table->s->db_low_byte_first)
    int4store(ptr, res);
  else
    longstore(ptr, res);
  return error;
}
```

就近剖析：

- ★ **INT 类型从不产生 `TYPE_WARN_TRUNCATED`，只产生 `TYPE_WARN_OUT_OF_RANGE`**——整数没有"切掉一部分"的概念，只有"夹断到边界值"。
- 双重符号标记是精髓：`nr` 的机器位 + `unsigned_val` 的解释标记 + 列自身的 `is_unsigned()`，三者组合决定分支。当 `nr < 0 && unsigned_val` 时，源码故意把 `nr` 改写成 `INT_MAX32 + 1` 来"制造溢出"，让下面的范围判断自然命中上界分支。
- 落盘用 `int4store` / `longstore` 按 `db_low_byte_first` 选字节序。
- `ASSERT_COLUMN_MARKED_FOR_WRITE` 在 debug 下断言该列在 `table->write_set` 位图中。

##### `Field_new_decimal::store_decimal`（[F2]）

```cpp
void Field_new_decimal::set_value_on_overflow(my_decimal *decimal_value,
                                              bool sign) const {
  max_my_decimal(decimal_value, precision, decimals());
  if (sign) {
    if (is_unsigned())
      my_decimal_set_zero(decimal_value);
    else
      decimal_value->sign(true);
  }
}
```

就近剖析：溢出时先造出该精度下的最大值再按符号取负；对 `UNSIGNED` 列，负数溢出被夹到 **0**（而不是最小负数）。

```cpp
type_conversion_status Field_new_decimal::store_value(const my_decimal *decimal_value) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  type_conversion_status error = TYPE_OK;

  /* check that we do not try to write negative value in unsigned field */
  if (is_unsigned() && decimal_value->sign()) {
    set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);
    error = TYPE_WARN_OUT_OF_RANGE;
    decimal_value = &decimal_zero;
  }

  int err = my_decimal2binary(E_DEC_FATAL_ERROR & ~E_DEC_OVERFLOW,
                              decimal_value, ptr, precision, dec);
  if (warn_if_overflow(err)) {
    my_decimal buff;
    set_value_on_overflow(&buff, decimal_value->sign());
    my_decimal2binary(E_DEC_FATAL_ERROR, &buff, ptr, precision, dec);
  }
  return (err != E_DEC_OK) ? decimal_err_to_type_conv_status(err) : error;
}
```

就近剖析：

- `E_DEC_FATAL_ERROR & ~E_DEC_OVERFLOW` 是**"允许溢出，但把溢出作为返回码告诉我"**——`store_value` 不因溢出致命，而是拿到 `E_DEC_OVERFLOW` 后自己造边界值重存。
- `E_DEC_TRUNCATED`（标度不够，四舍五入）与 `E_DEC_OVERFLOW`（整数位不够）走**不同路径**：前者是 Note 级的 `WARN_DATA_TRUNCATED`，后者才触发边界值替换。
- 返回值优先采信底层 `err`，只有 `err == E_DEC_OK` 时才返回自己算出的 `error`。

##### `Field_varstring::store`（[F2]）

```cpp
type_conversion_status Field_varstring::store(const char *from, size_t length,
                                              const CHARSET_INFO *cs) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  size_t copy_length;
  const char *well_formed_error_pos;
  const char *cannot_convert_error_pos;
  const char *from_end_pos;

  copy_length = field_well_formed_copy_nchars(
      field_charset, (char *)ptr + length_bytes, field_length, cs, from, length,
      field_length / field_charset->mbmaxlen,   // ★ 按"字符数"限制，不是字节数
      &well_formed_error_pos, &cannot_convert_error_pos, &from_end_pos);

  if (length_bytes == 1)
    *ptr = (uchar)copy_length;
  else
    int2store(ptr, static_cast<uint16>(copy_length));

  return check_string_copy_error(well_formed_error_pos, cannot_convert_error_pos,
                                 from_end_pos, from + length, true, cs);
}
```

就近剖析：**先写数据再写长度头**（即使中途出错，长度头也不指向未初始化内存）；限制参数是 `field_length / mbmaxlen`——**按字符数**，`VARCHAR(10) utf8mb4` 允许 10 字符（最多 40 字节），截断在多字节边界对齐，不会切出半个字符。

截断判定（[F2]）：

```cpp
type_conversion_status Field_longstr::report_if_important_data(
    const char *pstr, const char *end, bool count_spaces) {
  if (pstr < end) {                                    // String is truncated
    if (test_if_important_data(field_charset, pstr, end)) {
      if (current_thd->check_for_truncated_fields) {
        if (!current_thd->lex->is_ignore() && current_thd->is_strict_mode())
          set_warning(Sql_condition::SL_WARNING, ER_DATA_TOO_LONG, 1);
        else
          set_warning(Sql_condition::SL_WARNING, WARN_DATA_TRUNCATED, 1);
      }
      return TYPE_WARN_TRUNCATED;
    } else if (count_spaces) {
      // If we lost only spaces then produce a NOTE, not a WARNING
      if (current_thd->check_for_truncated_fields)
        set_warning(Sql_condition::SL_NOTE, WARN_DATA_TRUNCATED, 1);
      return TYPE_NOTE_TRUNCATED;
    }
  }
  return TYPE_OK;
}
```

就近剖析：★ `TYPE_NOTE_TRUNCATED` 最典型的产地——"只丢了尾部空格"不算真丢数据，只出 Note。注意 `count_spaces` 参数：`Field_varstring::store` 传 `true`（VARCHAR 保留尾部空格语义），`Field_string::store` 传 `false`（CHAR 本来补空格，丢了无所谓）。strict 下报 `ER_DATA_TOO_LONG`(1406)，否则 `WARN_DATA_TRUNCATED`(1265)。

##### `Field_blob::store`（[F2]）

```cpp
type_conversion_status Field_blob::store(const char *from, size_t length,
                                         const CHARSET_INFO *cs) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  if (table->blob_storage)   // GROUP_CONCAT with ORDER BY | DISTINCT
    return store_to_mem(from, length, cs,
                        current_thd->variables.group_concat_max_len,
                        table->blob_storage);
  return store_internal(from, length, cs);
}
```

`store_to_mem` 是 `TYPE_ERR_OOM` 的典型产地：分配失败时先 `reset()` 清成空值再返回 OOM。物理布局由 `store_ptr_and_length` 决定（[F1]）：

```cpp
  void store_ptr_and_length(const char *from, uint32 length) {
    store_length(length);
    memmove(ptr + packlength, &from, sizeof(char *));
  }
```

就近剖析：BLOB 在记录缓冲区里**只占 `packlength` + 指针大小**（通常 4+8=12 字节）；前 `packlength` 字节是长度，后面是 `char*` 指向真正数据。这也是为什么 BLOB 不能 `memcpy`、`fields_are_memcpyable()` 对它返回 false。

##### `Field_string::store`（[F2]）

```cpp
type_conversion_status Field_string::store(const char *from, size_t length,
                                           const CHARSET_INFO *cs) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  size_t copy_length;
  const char *well_formed_error_pos;
  const char *cannot_convert_error_pos;
  const char *from_end_pos;

  copy_length = field_well_formed_copy_nchars(
      field_charset, (char *)ptr, field_length, cs, from, length,
      field_length / field_charset->mbmaxlen, &well_formed_error_pos,
      &cannot_convert_error_pos, &from_end_pos);

  /* Append spaces if the string was shorter than the field. */
  if (copy_length < field_length)
    field_charset->cset->fill(field_charset, (char *)ptr + copy_length,
                              field_length - copy_length,
                              field_charset->pad_char);

  return check_string_copy_error(well_formed_error_pos, cannot_convert_error_pos,
                                 from_end_pos, from + length, false, cs);
}
```

就近剖析：CHAR 是**定长**，不足部分用 `field_charset->pad_char` 填满——这就是 `CHAR` 与 `VARCHAR` 在 `store` 层的唯一本质差异：**VARCHAR 写长度头，CHAR 补 pad_char**。填充用 `cset->fill` 而非逐字节赋值，因为某些字符集（如 UCS2）的 pad_char 是多字节序列。最后传 `count_spaces = false`（CHAR 的尾部空格比较时被忽略，不算丢数据）。

##### `Field_temporal::store`（[F2]）

```cpp
type_conversion_status Field_temporal::store(longlong nr, bool unsigned_val) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  int warnings = 0;
  MYSQL_TIME ltime;
  type_conversion_status error =
      convert_number_to_TIME(nr, unsigned_val, 0, &ltime, &warnings);
  if (error == TYPE_OK || error == TYPE_NOTE_TRUNCATED)
    error = store_internal(&ltime, &warnings);
  else {
    assert(warnings != 0);  // Must be set by convert_number_to_TIME
    if (warnings & (MYSQL_TIME_WARN_ZERO_DATE | MYSQL_TIME_WARN_ZERO_IN_DATE) &&
        !current_thd->is_strict_mode())
      error = TYPE_NOTE_TIME_TRUNCATED;
  }
  if (warnings && set_warnings(ErrConvString(nr, unsigned_val), warnings))
    return TYPE_ERR_BAD_VALUE;
  return error;
}
```

就近剖析：数字 → 时态的路径是 **数字 → `MYSQL_TIME` → 物理存储**；`convert_number_to_TIME` 同时负责"零日期是否允许"的策略判断，结果放进 `warnings` 位向量。非 strict 模式下零日期降级为 `TYPE_NOTE_TIME_TRUNCATED`。

字符串 → 时态（最复杂的一个）：

```cpp
type_conversion_status Field_temporal::store(const char *str, size_t len,
                                             const CHARSET_INFO *cs) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  type_conversion_status error = TYPE_OK;
  MYSQL_TIME ltime;
  MYSQL_TIME_STATUS status;
  if (convert_str_to_TIME(str, len, cs, &ltime, &status)) {
    /*
      When convert_str_to_TIME() returns error, ltime has been set to
      0 so there's nothing to store in the field.
    */
    reset();
    if (status.warnings & (MYSQL_TIME_WARN_ZERO_DATE | MYSQL_TIME_WARN_ZERO_IN_DATE) &&
        !current_thd->is_strict_mode())
      error = TYPE_NOTE_TIME_TRUNCATED;
    else
      error = TYPE_ERR_BAD_VALUE;
  } else {
    ...
    error = time_warning_to_type_conversion_status(status.warnings);
    const type_conversion_status tmp_error =
        store_internal_adjust_frac(&ltime, &status.warnings);
    // Return the most serious error of the two, see type_conversion_status
    if (tmp_error > error) error = tmp_error;
  }
  if (status.warnings && set_warnings(ErrConvString(str, len, cs), status.warnings))
    return TYPE_ERR_BAD_VALUE;
  return error;
}
```

就近剖析：

- 解析失败时 **先把字段 `reset()` 成零值再返回错误**——保证记录缓冲区始终合法，绝不留下垃圾。
- `if (tmp_error > error) error = tmp_error;` 就是前面说的"枚举有序性"的实际用法。

两个具体的 `store_internal` 展示物理编码的多样性（[F2]）：

```cpp
type_conversion_status Field_time::store_internal(const MYSQL_TIME *ltime, int *) {
  long tmp = ((ltime->month ? 0 : ltime->day * 24L) + ltime->hour) * 10000L +
             (ltime->minute * 100 + ltime->second);
  if (ltime->neg) tmp = -tmp;
  int3store(ptr, tmp);
  return TYPE_OK;
}

type_conversion_status Field_datetimef::store_internal(const MYSQL_TIME *ltime, int *) {
  /*
    If time zone displacement information is present in "ltime"
    - adjust the value to UTC based on the time zone
    - convert to the local time zone
  */
  MYSQL_TIME temp_t = *ltime;
  if (convert_time_zone_displacement(current_thd->time_zone(), &temp_t))
    return TYPE_ERR_BAD_VALUE;
  store_packed(TIME_to_longlong_datetime_packed(temp_t));
  return TYPE_OK;
}
```

就近剖析：老 `TIME` 用 3 字节 `HHMMSS` 十进制打包；`DATETIME(f)` 用 5 字节 packed 位打包。**同一个 `store_internal` 契约，底下是完全不同的位布局**——这就是 `Field` 多态的价值。

#### 2.4 `val_*`：四个解码方向

##### 接口与 ★ 双缓冲技巧（[F1]）

```cpp
  virtual double val_real() const = 0;
  virtual longlong val_int() const = 0;
  virtual my_decimal *val_decimal(my_decimal *) const = 0;
  String *val_str(String *str) const { return val_str(str, str); }
  /*
     val_str(buf1, buf2) gets two buffers and should use them as follows:
     if it needs a temp buffer to convert result to string - use buf1
       example Field_tiny::val_str()
     if the value exists as a string already - use buf2
       example Field_string::val_str()
     consequently, buf2 may be created as 'String buf;' - no memory
     will be allocated for it. buf1 will be allocated to hold a
     value if it's too small. ...
     This trickery is used to decrease a number of malloc calls.
  */
  virtual String *val_str(String *, String *) const = 0;
```

就近剖析（★ 本节要点）：

- **为什么两个参数**：调用方通常有一个长期持有、已分配缓冲的 `String`（buf1，如 `Item::str_value`），还可能是栈上的空 `String`（buf2，零分配）。实现二选一：
  - 值**已经是字符串**（`Field_varstring`/`Field_blob`/`Field_string`）→ 用 **buf2**，直接把 `ptr/length` 指到记录缓冲区，**零拷贝、零分配**；
  - 值**需要转换**（`Field_long`/`Field_new_decimal`）→ 用 **buf1**，`alloc()` 后格式化。
- 单参数版 `val_str(str)` 就是 `val_str(str, str)`。
- 这条约定是**纯性能约定，不是语义约定**：实现必须遵守，否则调用方栈上的 buf2 会被误写导致悬垂指针。

##### `Field_long::val_int()`（[F2]）

```cpp
longlong Field_long::val_int() const {
  ASSERT_COLUMN_MARKED_FOR_READ;
  int32 j;
  if (table->s->db_low_byte_first)
    j = sint4korr(ptr);
  else
    j = longget(ptr);
  return is_unsigned() ? (longlong)(uint32)j : (longlong)j;
}
```

就近剖析：解码是 `store` 的严格逆运算。**`is_unsigned()` 决定的是"如何解释这 4 个字节"**，而不是"是否检查符号"；`val_int()` 永不返回错误，因为它读的是已被 `store()` 校验过的值。

##### `Field_varstring::val_str()`（[F2]）—— buf2 路径的教科书

```cpp
String *Field_varstring::val_str(String *, String *val_ptr) const {
  ASSERT_COLUMN_MARKED_FOR_READ;
  val_ptr->set(pointer_cast<const char *>(data_ptr()), data_length(),
               field_charset);
  return val_ptr;
}
```

就近剖析：第一个参数被完全忽略（未命名），只用 **buf2**，`set()` 只是把 `ptr/length/charset` 三个字段指过去，**没有任何 memcpy**。VARCHAR 的 `val_str` 在热路径上是零分配的。

##### `Field_new_decimal::val_decimal()`（[F2]）

```cpp
my_decimal *Field_new_decimal::val_decimal(my_decimal *decimal_value) const {
  ASSERT_COLUMN_MARKED_FOR_READ;
  binary2my_decimal(E_DEC_FATAL_ERROR, ptr, decimal_value, precision, dec,
                    m_keep_precision);
  return decimal_value;
}
```

就近剖析：调用方提供 `my_decimal` 存储（通常栈变量），返回同一指针——**无分配**。三个 `val_*` 都建立在其上（`val_real` → `my_decimal2double`、`val_int` → `my_decimal2int`、`val_str` → `my_decimal2string` 且 `zerofill` 时用 `fixed_precision` 补零），说明 **`val_decimal` 是 DECIMAL 列的"规范解码"**。

#### 2.5 与 `Item` 的对偶

##### `Item_field::val_*`：纯粹的转发层（[F4]）

```cpp
double Item_field::val_real() {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return 0.0;
  return field->val_real();
}

longlong Item_field::val_int() {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return 0;
  return field->val_int();
}

String *Item_field::val_str(String *str) {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return nullptr;
  str->set_charset(str_value.charset());
  return field->val_str(str, &str_value);     // ★ 双缓冲技巧的标准调用方
}
```

就近剖析：`Item_field` 是 `Field` 的**代理**，`val_*` 只做"NULL 判定 + 转发"。`val_str` 里 `str`（调用方给的）当 buf1、`&str_value`（Item 自己的成员）当 buf2——对 VARCHAR/BLOB 列结果直接指向记录缓冲区，对 INT/DECIMAL 列写进调用方的 `str`。

##### `Item::save_in_field()`：反方向的分派表（[F4]）

入口（[F4]）：

```cpp
type_conversion_status Item::save_in_field(Field *field, bool no_conversions) {
  Functional_index_error_handler functional_index_error_handler(field, current_thd);
  const type_conversion_status ret = save_in_field_inner(field, no_conversions);
  /*
    If an error was raised during evaluation of the item,
    save_in_field_inner() might not notice and return TYPE_OK. Make
    sure that we return not OK if there was an error.
  */
  if (ret == TYPE_OK && current_thd->is_error()) return TYPE_ERR_BAD_VALUE;
  return ret;
}
```

就近剖析：最后一个 `if` 是**兜底**——有些 `Field::store` 出错时只 `my_error()` 却忘了返回非 OK，这里统一补上。

`save_in_field_inner()` 的最末三支是**按 `Item` 自己的 `result_type()` 决定调用哪个 `val_*`**：

```cpp
  if (result_type() == REAL_RESULT) {
    double nr = val_real();
    if (null_value) return set_field_to_null_with_conversions(field, no_conversions);
    field->set_notnull();
    return field->store(nr);
  }
  if (result_type() == DECIMAL_RESULT) {
    my_decimal decimal_value;
    my_decimal *value = val_decimal(&decimal_value);
    if (null_value) return set_field_to_null_with_conversions(field, no_conversions);
    field->set_notnull();
    return field->store_decimal(value);
  }
  longlong nr = val_int();
  if (null_value) return set_field_to_null_with_conversions(field, no_conversions);
  field->set_notnull();
  return field->store(nr, unsigned_flag);
```

就近剖析：**`Item::result_type()` 决定拉取方式，拉取结果喂给对应的 `Field::store` 重载**——这就是 2.1 所说"对偶"的具体形态。`STRING_RESULT` 分支内部还有一层按 `field->type()` 的细分（JSON / 时态 / DECIMAL / 整数 / 浮点 / 其他字符串），目的是**避免"先转字符串再让 Field 解析"的往返**。

每个分支都遵循同一个三步模板：`val_*()` → 判 `null_value` → `field->set_notnull()` + `field->store*()`。★ **`set_notnull()` 必须在 `store()` 之前**——因为 `store()` 只写数据区不写 NULL 位图。

#### 2.6 `type_conversion_status` 的消费：三种检查策略与 strict mode

##### 策略枚举与告警出口（[F1] / [F2]）

```cpp
enum enum_check_fields : int {
  CHECK_FIELD_IGNORE = 0,
  CHECK_FIELD_WARN,
  CHECK_FIELD_ERROR_FOR_NULL
};
```

`Field::set_warning()` 是唯一的告警出口，开头就是第一道闸（[F2]）：

```cpp
  if (!thd->check_for_truncated_fields)
    return level >= Sql_condition::SL_WARNING;
```

就近剖析：**`CHECK_FIELD_IGNORE`（0）时完全静默，但返回值仍有信息量**：Warning 及以上返回 1，Note 返回 0——调用方用这个返回值决定是否升级为 `TYPE_ERR_BAD_VALUE`。第二道闸是"只对 INSERT/REPLACE 做告警聚合"（用 `m_warnings_pushed` 位掩码保证 `ER_BAD_NULL_ERROR` 等每列只报一次）。

##### NOT NULL 裁决（[F2]）

```cpp
type_conversion_status Field::check_constraints(int mysql_errno) {
  assert(!is_tmp_nullable());
  if (is_nullable()) return TYPE_OK;      // 可空列不检查
  if (!m_is_tmp_null) return TYPE_OK;     // 没被临时置 NULL
  if (this == table->next_number_field) return TYPE_OK;   // AUTO_INCREMENT 豁免

  switch (m_check_for_truncated_fields_saved) {
    case CHECK_FIELD_WARN:
      set_warning(Sql_condition::SL_WARNING, mysql_errno, 1);
      [[fallthrough]];
    case CHECK_FIELD_IGNORE:
      return TYPE_OK;                     // ★ 静默放过（值被改成隐式默认值）
    case CHECK_FIELD_ERROR_FOR_NULL:
      my_error(ER_BAD_NULL_ERROR, MYF(0), field_name);
      return TYPE_ERR_NULL_CONSTRAINT_VIOLATION;
  }
  ...
}
```

就近剖析：★ 三种策略的语义在这里最清楚——**`CHECK_FIELD_WARN` 发完 warning 仍返回 `TYPE_OK`（值被悄悄改成隐式默认值，语句继续）**，`CHECK_FIELD_ERROR_FOR_NULL` 才报错 + 返回违规。用 `m_check_for_truncated_fields_saved`（而非当前 `thd` 状态）是因为触发器执行期间该值可能已被改写。

##### 三种策略在语句层的设定（[F5]）

```cpp
    thd->check_for_truncated_fields =
        ((insert_many_values.size() == 1 && !lex->is_ignore())
             ? CHECK_FIELD_ERROR_FOR_NULL
             : CHECK_FIELD_WARN);
```

就近剖析：**单行 INSERT 且不带 IGNORE → `CHECK_FIELD_ERROR_FOR_NULL`**（NOT NULL 违规直接报错 1048）；**多行 INSERT / INSERT...SELECT / 带 IGNORE → `CHECK_FIELD_WARN`**（降级为 warning + 写隐式默认值）。有意思的是 `Query_result_insert::send_data()` 的顺序：`store_values()` 期间用 `CHECK_FIELD_WARN`（赋值阶段宽容），写完立刻改成 `CHECK_FIELD_ERROR_FOR_NULL` 再跑触发器与约束（裁决阶段严格）。

##### strict mode 的影响点

strict mode **不改变 `type_conversion_status` 的取值**，只改变两件事：① **告警码的选择**（`is_strict_mode()` → `ER_DATA_TOO_LONG`(1406)，否则 `WARN_DATA_TRUNCATED`(1265)）；② **是否放弃存储**（`Field_new_decimal::store(const char*)` 在 strict 下 `E_DEC_BAD_NUM` 直接返回错误而不写值；非 strict 下仍写 0）。与 `lex->is_ignore()` 联用——源码里到处是 `!thd->lex->is_ignore() && thd->is_strict_mode()` 这样的合取。

##### `Item::save_in_field_no_warnings()`（[F4]）

```cpp
  thd->variables.sql_mode &=
      ~(MODE_NO_ZERO_IN_DATE | MODE_NO_ZERO_DATE | MODE_STRICT_ALL_TABLES |
        MODE_STRICT_TRANS_TABLES);
  thd->check_for_truncated_fields = CHECK_FIELD_IGNORE;
  const type_conversion_status res = save_in_field(field, no_conversions);
```

就近剖析：这是**三重屏蔽**（关告警、剥 sql_mode、放开 write_set 位图），但**返回值仍然保留**——调用方（优化器常量折叠、range 优化器探测边界）据此判断"这个常量能不能放进这个列"，从而决定能否做分区裁剪或把谓词判为 `impossible where`。这是"值与告警解耦"的典范：`type_conversion_status` 是**程序可读的语义信号**，`set_warning()` 才是**面向用户的呈现**。

#### 2.7 不变式清单

1. **`store()` 之后记录缓冲区一定合法**：失败路径一律先 `reset()` 或写入边界值（夹断到 min/max、pad_char 补空格），绝不留下半写状态。
2. **`store()` 从不抛异常，只返回 `type_conversion_status`**；`val_*()` 没有错误通道，因为它是 `store()` 的逆运算且输入已合法。
3. **枚举有序**：`>` 比较被 `Field_temporal::store` 等处直接使用。
4. **`set_notnull()` 必须早于 `store*`**。
5. **告警是 `THD` 级状态驱动的**：`check_for_truncated_fields` 三态 + `sql_mode` + `lex->is_ignore()` 共同决定同一个 status 最终表现为 error、warning 还是静默。

#### 2.8 ★ `send_to_protocol()`：读方向的第二个出口

`val_*()` 是给 `Item` / 表达式用的（返回 `String` / 整数 / decimal 等**表示**）。但把列值**发给客户端**还有一条**独立出口**——`Field::send_to_protocol(Protocol*)`（`[F2]`）。

默认实现就是"字符串化再存"：

```cpp
// [F2] sql/field.cc
bool Field::send_to_protocol(Protocol *protocol) const {
  if (is_null()) return protocol->store_null();
  char buff[MAX_FIELD_WIDTH];
  String tmp(buff, sizeof(buff), charset());
  String *res = val_str(&tmp);
  return res ? protocol->store(res) : protocol->store_null();
}
```

各类型**重写**它以走类型专用的协议 API，避免"字符串化再解析"：

```cpp
bool Field_long::send_to_protocol(Protocol *protocol) const {
  ASSERT_COLUMN_MARKED_FOR_READ;
  if (is_null()) return protocol->store_null();
  return protocol->store_long(Field_long::val_int(),
                              zerofill ? field_length : 0);   // ★ zerofill 时把宽度传给协议
}
bool Field_new_decimal::send_to_protocol(Protocol *protocol) const {
  if (is_null()) return protocol->store_null();
  my_decimal dec_value;
  return protocol->store_decimal(val_decimal(&dec_value),
                                 zerofill ? precision : 0, dec);
}
bool Field_temporal_with_date::send_to_protocol(Protocol *protocol) const {
  if (is_null()) return protocol->store_null();
  MYSQL_TIME ltime;
  if (get_date_internal(&ltime)) {
    /* Only MYSQL_TYPE_TIMESTAMP can return an error in get_date_internal() */
    assert(type() == MYSQL_TYPE_TIMESTAMP);
    set_zero_time(&ltime, MYSQL_TIMESTAMP_DATETIME);          // ★ 只有 TIMESTAMP 会走到这
  }
  return protocol->store_datetime(ltime, dec);
}
```

★ 两个易错点：① **`pack_length()` 与协议完全无关**——它只服务三个消费者：行缓冲布局、filesort / 临时表、复制 row image（见 3.9）；② `zerofill` 时把 `field_length` / `precision` 当作"显示宽度"传给协议——这就是 `INT(11) ZEROFILL` 显示补零的来源。

---

#### 2.9 `Field_num::get_int`：数值族 `store` 的钳制助手

`store(const char*)` 的整数解析路径上，除 LONGLONG 外**全部**走这个统一助手——它负责"字符串 → 整数 + 上下界钳制 + 告警"：

```cpp
// [F2] sql/field.cc
/*
  Convert a string to an integer, then check bounds.
  ...
  @return TYPE_OK, TYPE_WARN_OUT_OF_RANGE, TYPE_ERR_BAD_VALUE or TYPE_WARN_TRUNCATED
*/
type_conversion_status Field_num::get_int(const CHARSET_INFO *cs,
                                          const char *from, size_t len,
                                          longlong *rnd, ulonglong unsigned_max,
                                          longlong signed_min, longlong signed_max) {
  const char *end;
  int error;
  *rnd = (longlong)cs->cset->strntoull10rnd(cs, from, len, is_unsigned(), &end, &error);
  if (is_unsigned()) {
    if ((((ulonglong)*rnd > unsigned_max) && (*rnd = (longlong)unsigned_max)) ||
        error == MY_ERRNO_ERANGE)
      goto out_of_range;                    // ★ 越界时"钳到边界值"再告警
  } else {
    if (*rnd < signed_min)  { *rnd = signed_min;  goto out_of_range; }
    else if (*rnd > signed_max) { *rnd = signed_max; goto out_of_range; }
  }
  if (current_thd->check_for_truncated_fields != 0)
    return check_int(cs, from, len, end, error);   // 有没有尾随垃圾字符
  return TYPE_OK;

out_of_range:
  set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);
  return TYPE_WARN_OUT_OF_RANGE;
}
```

调用点（每类传自己的上下界）：

```cpp
// [F2] sql/field.cc
  Field_tiny::store(...)   → get_int(cs, from, len, &rnd, 255, -128, 127);
  Field_short::store(...)  → get_int(cs, from, len, &rnd, UINT_MAX16, INT_MIN16, INT_MAX16);
  Field_medium::store(...) → get_int(cs, from, len, &rnd, UINT_MAX24, INT_MIN24, INT_MAX24);
  Field_long::store(...)   → get_int(cs, from, len, &rnd, UINT_MAX32, INT_MIN32, INT_MAX32);
```

★ 三个要点：
1. **钳制 = 截断到边界值 + 告警**，不是报错；严格模式的升级**不在 `store` 里**（见 2.6 的 4 参重载——`store` 一律只返回 `TYPE_WARN_OUT_OF_RANGE`）。
2. ★ **`Field_longlong::store` 不走 `get_int`**——它直接 `strntoull10rnd`、**不做钳制**（整数上界即 `ulonglong` 自身，无需钳）。这是历史分层，也是"LONGLONG 与其他整型行为略有差异"的根源。
3. `check_int()` 只在 `check_for_truncated_fields != 0` 时调用——即"是否检查尾随垃圾字符"由 THD 上的开关决定（同样是 2.6 的三种检查策略）。

#### 2.10 ★ `val_*_temporal`：时态类型的 packed 值读取接口

时态类型除了 `val_int()` / `val_str()` 之外，还有**一组绕过 `MYSQL_TIME` 解析的快路径**——直接返回打包后的 longlong：

```cpp
// [F1] sql/field.h
  virtual longlong val_int() const = 0;
  /**
    Returns TIME value in packed longlong format.
    This method should not be called for non-temporal types.
    Temporal field types override the default method.
  */
  virtual longlong val_time_temporal() const { assert(0); return 0; }
  /**
    Returns DATE/DATETIME value in packed longlong format.
    This method should not be called for non-temporal types.
    Temporal field types override the default method.
  */
  virtual longlong val_date_temporal() const { assert(0); return 0; }

  virtual longlong val_time_temporal_at_utc() const { return val_time_temporal(); }
  virtual longlong val_date_temporal_at_utc() const { return val_date_temporal(); }
```

★ 四个要点：

1. **返回的是 `TIME_to_longlong_*_packed()` 的打包值**，不是 `MYSQL_TIME` 结构、也不是 `YYYYMMDDHHMMSS` 整数——用于"取出来还要存进另一个时态列、或做时态比较"的场景，省掉 `MYSQL_TIME` 的中间转换。例如 `Field_datetimef::val_date_temporal()` = `my_datetime_packed_from_binary(ptr, dec)`。
2. ★ **基类是 `assert(0)` 兜底**——非时态类型调用它（debug 下）会立即断言失败，注释明说"should not be called for non-temporal types"。
3. `_at_utc` 变体默认直接转发给非 `_at_utc` 版本（只有需要时区归一的子类才覆写它）。
4. ★ **`Item_field::val_*_temporal()` 只是转发**，且**不判 NULL**（与 `val_int()` 不同——后者先写 `null_value`）：

```cpp
// [F4] sql/item.cc
longlong Item_field::val_date_temporal() {
  assert(fixed == 1);
  return field->val_date_temporal();
}
longlong Item_field::val_time_temporal() {
  assert(fixed == 1);
  return field->val_time_temporal();
}
```

★ **引擎侧（`storage/innobase`）对这一组接口 0 使用**——这是纯 server 侧接口。InnoDB 取行走的是 `row_sel_field_store_in_mysql_format` 的字节路径（见第五章与 `row_search.md`），不经过 `val_*`。

另有一个按 `type()` 自动分派的便利函数 `val_temporal_by_field_type()`——"Returns 'native' packed longlong representation of a TIME or DATE/DATETIME field depending on field type"，内部按 `type()` 决定调上面哪一个。

---

### 三、存储布局：ptr、长度头、NULL 位与 pack_length

> 补充源码坐标（章首表之外的）：`[F13]` sql/table.cc、`[F14]` sql/dd_table_share.cc、`[F15]` sql/default_values.cc、`[F16]` include/my_base.h、`[F17]` sql/sql_tmp_table.cc；另 InnoDB 侧的 `get_field_offset()` 在 `storage/innobase/handler/ha_innodb.cc`。

#### 3.1 `ptr` 与行缓冲

**`ptr` 是什么**：`ptr` 是 `Field` 中最重要的成员——**指向"本列在行缓冲区中起始字节"的裸指针**。行缓冲区就是 `TABLE::record[0]`（`record[1]` 是第二份缓冲，用于 UPDATE 前像 / range 比较）。

```cpp
// [F5] sql/table.h
  Field **field{nullptr}; /* Pointer to fields */
  // note: 存储到 se 的 row data（mysql format 的行数据）
  uchar *record[2]{nullptr, nullptr};
  uchar *null_flags{nullptr};  ///< Pointer to the null flags of record[0]

// [F1] sql/field.h
 protected:
  /// Holds the position to the field in record
 uchar *ptr;
 private:
  /**
     Byte where the @c NULL bit is stored inside a record. If this Field is a
     @c NOT @c NULL field, this member is @c NULL.
  */
  uchar *m_null_ptr;
```

`ptr` 是 protected，外部通过 `field_ptr()` / `set_field_ptr()` 访问。

**`offset()`：从 `ptr` 反推列内偏移**

```cpp
  uint offset(uchar *record) const { return (uint)(ptr - record); }
```

一次指针减法，把"绝对地址"翻译成"行内相对偏移"——这是 server 层与存储引擎交换布局信息的唯一通用手段。

★ **`get_field_offset()` 只在 InnoDB 侧存在**——sql 层没有。它是 `ha_innodb.cc` 里的 `static inline` 小工具，本质是对 `offset()` 的包装：

```cpp
/* The following accessor functions should really be inside MySQL code! */
static inline uint get_field_offset(const TABLE *table, const Field *field) {
  return field->offset(table->record[0]);
}
```

上面那句注释是 InnoDB 开发者的吐槽：这个访问器本该在 server 层。它的调用点就是 InnoDB 构造 `mysql_col_offset` 模板的地方。

★ **`set_record_ptr` 不存在**（全仓库检索无此符号）。真正干这件事的是三个函数：

```cpp
  void move_field(uchar *ptr_arg, uchar *null_ptr_arg, uchar null_bit_arg) {
    ptr = ptr_arg; m_null_ptr = null_ptr_arg; null_bit = null_bit_arg;
  }
  virtual void move_field_offset(ptrdiff_t ptr_diff) {
    ptr += ptr_diff;
    if (is_nullable()) m_null_ptr += ptr_diff;
  }
```

- `move_field(...)`：**绝对重定位**（三个成员一起赋新值）——用于"把字段放到一块全新缓冲区"（临时表建表、key field 构造）。
- `move_field_offset(diff)`：**相对平移**（`+=` 差值，掩码不变）——用于"整块缓冲搬家但布局不变"。

`Field_bit` 额外 override 来搬它的"零散位"指针：

```cpp
  void move_field_offset(ptrdiff_t ptr_diff) final {
    Field::move_field_offset(ptr_diff);
    if (bit_ptr != nullptr) bit_ptr += ptr_diff;
  }
```

**`ptr` 何时被赋值（普通表：share 布局 → 开表时 clone + 平移）**

`TABLE_SHARE` 那套 `Field` 的 `ptr` 指向 `share->default_values`（"默认值行"，布局计算的基准缓冲）。开表时 `[F13]`：

```cpp
  outparam->field = field_ptr;
  record = (uchar *)outparam->record[0] - 1; /* Fieldstart = 1 */
  outparam->null_flags = (uchar *)record + 1;/* record[0] 开头 */
  move_offset = outparam->record[0] - share->default_values + ...;

  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);
    *field_ptr = new_field;
    new_field->init(outparam);
    new_field->move_field_offset(move_offset);   // ★ 平移到本 TABLE 的 record[0]
    if (!has_default_values) new_field->reset();
```

关键点：

1. `null_flags == outparam->record[0]`，而 `record[0]-1` 那一个字节是保留的 **delete-link 字节**——这正是 `rec_buff_length = ALIGN_SIZE(reclength + 1 + extra)` 里 `+1` 的来源。
2. `move_offset = record[0] - default_values`：一次减法拿到平移量。

`record[]` 的分配（[F13]）：

```cpp
  record = root->ArrayAlloc<uchar>(share->rec_buff_length * records + share->null_bytes);
  outparam->record[0] = record;
  if (records > 1) outparam->record[1] = record + share->rec_buff_length;
  else             outparam->record[1] = outparam->record[0];  // Safety
```

**临时表：走 `relocate_field()`**（`[F17]`）——建好字段后用 `move_field()` 做绝对重定位：

```cpp
inline void relocate_field(Field *field, uchar *pos, uchar *null_flags,
                           uint *null_count) {
  if (!field->is_flag_set(NOT_NULL_FLAG)) {
    field->move_field(pos, null_flags + *null_count / 8,
                      (uint8)1 << (*null_count & 7));   // ★ NULL 位布局公式的"可执行版"
    (*null_count)++;
  } else
    field->move_field(pos, nullptr, 0);
  if (field->type() == MYSQL_TYPE_BIT) {
    /* We have to reserve place for extra bits among null bits */
    ((Field_bit *)field)->set_bit_ptr(null_flags + *null_count / 8, *null_count & 7);
    (*null_count) += (field->field_length & 7);
  }
  field->reset();
```

调用形如 `relocate_field(field_r, pos, null_flags, &null_count); pos += field_r->pack_length();`——即"行内按 `pack_length()` 线性排布"的直接证据。

#### 3.2 `pack_length()` vs `pack_length_in_rec()`

```cpp
  /*
    pack_length() returns size (in bytes) used to store field data in memory
    (i.e. it returns the maximum size of the field in a row of the table,
    which is located in RAM).
  */
  virtual uint32 pack_length() const { return (uint32)field_length; }

  /*
    pack_length_in_rec() returns size (in bytes) used to store field data on
    storage (i.e. it returns the maximal size of the field in a row of the
    table, which is located on disk).
  */
  virtual uint32 pack_length_in_rec() const { return pack_length(); }
```

- `pack_length()` = **内存（MySQL row buffer）里占多少字节**——所有行内布局计算都用它。
- `pack_length_in_rec()` = **磁盘行格式里这一列占的槽位大小**。绝大多数类型两者相等。

★ **`Field_bit` 是唯一 override 出差异的类**：

```cpp
  uint32 pack_length() const final { return (uint32)(field_length + 7) / 8; }
  uint32 pack_length_in_rec() const final { return bytes_in_rec; }
```

原因：`BIT(n)` 里 `n % 8 != 0` 的"零散高位"被塞进行首的 NULL 位区（preamble），`pack_length()` 要把这 1 个 preamble 字节算进来（向上取整），而 `bytes_in_rec` 不含零散位。这也是 `find_record_length()` 里 `leftover_bits` 的来源。类上方注释还说明：因此 `Field_bit::cmp_binary()` 只能用基类版本（它按 `pack_length()` 取长度，已含 preamble 字节）。

常见定长类型的 `pack_length()`：`TINYINT`=1、`SMALLINT`=2、`MEDIUMINT`=3、`INT`=4、`BIGINT`=8、`FLOAT`=4、`DOUBLE`=8；`CHAR(n)` = `n*mbmaxlen`（不含长度头）；`VARCHAR(n)` = `n*mbmaxlen + length_bytes`；`*BLOB`/`JSON`/`GEOMETRY` = `packlength + 8`。

#### 3.3 ★ VARCHAR 的长度头

**宏定义：只看总字节数，与 `mbmaxlen` 无关**（`[F16]`）：

```cpp
#define HA_VARCHAR_PACKLENGTH(field_length) ((field_length) < 256 ? 1 : 2)
```

★ **这是最需要纠正的一个误解**：判断依据是**该列 `field_length`（总字节数）是否 < 256**，不是字符数、也不是"字符数×mbmaxlen 后再比"。所以：

- `VARCHAR(63) utf8mb4` → `field_length = 252` < 256 → **1 字节头**
- `VARCHAR(64) utf8mb4` → `field_length = 256` ≥ 256 → **2 字节头**
- `VARCHAR(255) latin1` → `field_length = 255` → **1 字节头**

**"utf8mb4 下 VARCHAR(64) 变 2 字节头"是正常且正确的行为，不是 bug。**

**`length_bytes` 成员**（`[F1]`）：

```cpp
class Field_varstring : public Field_longstr {
  uint32 pack_length() const final { return (uint32)field_length + length_bytes; }
  uint32 key_length() const final { return (uint32)field_length; }
  const uchar *data_ptr() const final { return ptr + length_bytes; }
  uint32 get_length_bytes() const override { return length_bytes; }
 private:
  /* Store number of bytes used to store length (1 or 2) */
  uint32 length_bytes;
};
```

- `pack_length() = field_length + length_bytes`（长度头 + 数据区）；`data_ptr() = ptr + length_bytes`；`key_length()` 与 `row_pack_length()` 都**不含**长度头。
- 基类 `Field::get_length_bytes()` 是个 `assert(0)` 桩——"长度头"这个概念只对 varchar 类有意义。
- 正式建表路径：`length_bytes = HA_VARCHAR_PACKLENGTH(field_length)`（见 `make_field()`）。

**长度头的写入与读取**：

```cpp
// 写（Field_varstring::store）：先写数据到 ptr+length_bytes，再写长度头
  if (length_bytes == 1) *ptr = (uchar)copy_length;
  else                   int2store(ptr, static_cast<uint16>(copy_length));
// 读（data_length）
uint32 Field_varstring::data_length(ptrdiff_t row_offset) const {
  return length_bytes == 1 ? (uint32)*(ptr + row_offset) : uint2korr(ptr + row_offset);
}
```

1 字节头直接赋值（因此该列单条最多 255 字节）；2 字节头用 `int2store()`（**本机字节序**，与 `db_low_byte_first` 体系一致）。

★ **例外：键里恒 2 字节**——`new_key_field()` 强制改 `length_bytes = 2`：

```cpp
Field *Field_varstring::new_key_field(...) const {
  Field_varstring *res = ...;
  if (res) {
    /* Keys length prefixes are always packed with 2 bytes */
    res->length_bytes = 2;
  }
  return res;
}
```

于是出现反直觉组合：**前缀索引列可能 `field_length == 1` 而 `length_bytes == 2`**。所以取长度头**一律用 `get_length_bytes()`**，不要用 `HA_VARCHAR_PACKLENGTH(field_length)` 去推。

#### 3.4 ★ NULL 位布局

**两个成员**（`[F1]`）：`m_null_ptr`（指向"行缓冲中存放本列 NULL 位的那个字节"，NOT NULL 列是 `nullptr`）+ `null_bit`（掩码 `1 << n`）。

```cpp
  bool is_nullable() const { return m_null_ptr != nullptr; }
```

★ **8.0.39 的 `Field` 上没有 `maybe_null()`**——只有 `is_nullable()`。`maybe_null()` 只存在于 `SEL_ARG`（`sql/range_optimizer/tree.h`）与 `Item` 侧。写 `field->maybe_null()` 是**编译不过**的旧习惯。

**`is_null()`**（`[F1]`）：

```cpp
  bool is_null(ptrdiff_t row_offset = 0) const {
    /*
      if the field is NULLable, it returns NULLity based on
      m_null_ptr[row_offset] value. Otherwise it returns NULL flag depending
      on TABLE::has_null_row() value. ... Do not change the order of testing.
    */
    if (is_nullable()) return (m_null_ptr[row_offset] & null_bit);
    if (is_tmp_nullable()) return m_is_tmp_null;
    return table->has_null_row();
  }
```

就近剖析：不可空列也"可能返回 NULL"——外连接的 NULL 补行、或"有聚合无 GROUP BY 且无命中行"的隐式分组，这两种情况 `TABLE::has_null_row()` 为真、整行视作全 NULL。**测试顺序不能换**（注释明确要求）。

`set_null()` / `set_notnull()` 就是 `|=` / `&= ~`（`[F2]`），两者都有 `assert(m_null_ptr != &dummy_null_buffer)`——`dummy_null_buffer` 是给"语义上可空但物理上无处存 NULL"的字段（如 `create_tmp_field()` 造的对象）兜底的假 NULL 位。

**位号分配：第 n 个可空列 → `null_flags + n/8`，掩码 `1 << (n%8)`**

`[F17]` 的 `relocate_field()` 已给出"可执行版"（见 3.1）。DD 路径完全同构（`[F14]`）：

```cpp
  null_flags = null_pos = share->default_values;
  rec_pos = share->default_values + share->null_bytes;
  uint null_bit_pos = (share->db_create_options & HA_OPTION_PACK_RECORD) ? 0 : 1;
  for (const dd::Column *col_obj : tab_obj->columns()) {
    if (!col_obj->is_virtual()) {
      fill_column_from_dd(thd, share, col_obj, null_pos, null_bit_pos, rec_pos, field_nr);
      if ((null_bit_pos += column_preamble_bits(col_obj)) > 7) {
        null_pos++;                 // 进位到下一个字节
        null_bit_pos -= 8;
      }
      rec_pos += share->field[field_nr]->pack_length_in_rec();   // ★ 另一条游标
    }
    field_nr++;
  }
```

注意两条游标独立前进：`null_pos` 按"位计数溢出进位"，`rec_pos` 用 **`pack_length_in_rec()`**。虚拟生成列走**第二遍**，保证 NULL 位顺序符合预期。收尾用两个断言锁死：`assert(share->null_bytes == (null_pos - null_flags + (null_bit_pos + 7) / 8));`

每列占多少 preamble 位（[F14]）——可空占 1 位，BIT 的零散位额外占：

```cpp
static uint column_preamble_bits(const dd::Column *col_obj) {
  uint result = 0;
  if (col_obj->is_nullable()) result++;
  if (col_obj->type() == dd::enum_column_types::BIT) {
    bool treat_bit_as_char = false;
    (void)col_obj->options().get("treat_bit_as_char", &treat_bit_as_char);
    if (!treat_bit_as_char) result += col_obj->char_length() & 7;
  }
  return result;
}
```

★ **bit0 是不是"行未删除"标记，取决于 `HA_OPTION_PACK_RECORD`**：

```cpp
  uint null_bit_pos = (share->db_create_options & HA_OPTION_PACK_RECORD) ? 0 : 1;
```

- **设置了**（InnoDB COMPACT / DYNAMIC / COMPRESSED）：`null_bit_pos` 从 **0** 起 → "前 **8** 个可空列落在 `null_flags` 的第 0 字节"。
- **没设置**（REDUNDANT 等非 packed 格式）：从 **1** 起，**bit0 被"row not deleted"标记占用** → "前 **7** 个可空列落在第 0 字节"。

`[F15]` 对同一条件做对称处理（不 packed 时把 bit0 置 1，并把尾部未用位也置 1）：

```cpp
static void set_pack_record_and_unused_preamble_bits(bool pack_record,
                                                     ulong preamble_bits,
                                                     uchar *default_values) {
  // Set first bit if the HA_OPTION_PACK_RECORD is not set.
  if (!pack_record) *default_values |= 1;
  // We need to set the unused bits to 1. ...
  if (preamble_bits & 7)
    *(default_values + preamble_bits / 8) |= ~(((uchar)1 << (preamble_bits & 7)) - 1);
}
```

**`null_bytes` 公式**（`[F15]`，8.0 里算行长的唯一权威实现）：

```cpp
  share->null_bytes = (share->null_fields + leftover_bits + 7) / 8;
  share->last_null_bit_pos = (share->null_fields + leftover_bits) & 7;
  share->reclength += share->null_bytes;      // ★ NULL 位图算在 reclength 里、位于行首
```

- `null_fields`：可空列数（每列 1 位）。
- `leftover_bits`：`pack_record ? 0 : 1` + 所有非 `treat_bit_as_char` 的 BIT 列的 `char_length & 7` 之和。
- `+7) / 8`：向上取整到字节。

`null_offset()` 与 `offset()` 同理（指针减法）；InnoDB 的 `templ->mysql_null_byte_offset = field->null_offset()` 用的就是它。

#### 3.5 `field_length` 是字节数

```cpp
  // Length of field. Never write to this member directly; instead, use set_field_length().
  // 当前字符集下最大字节数，单位为 byte。如 (c varchar(12))，Field::field_length = 12 * mbmaxlen
  uint32 field_length;
  virtual uint32 char_length() const { return field_length / charset()->mbmaxlen; }
```

★ `varchar(12)` 在 utf8mb4（`mbmaxlen=4`）下 **`field_length = 48`**，不是 12。直接写 `field_length` 被禁止（必须走 `set_field_length()`，因为子类挂钩子）。

**`char_length_cache` 存在的理由**（`[F1]` 注释原文）：

```cpp
  // An always-updated cache of the result of char_length(), because
  // dividing by charset()->mbmaxlen can be surprisingly costly compared
  // to the rest of e.g. make_sort_key().
  uint32 char_length_cache;
```

即：**一次整数除法（`/ mbmaxlen`）相对于 `make_sort_key()` 的主体竟然贵得不成比例**——这是"热路径上避免除法"的典型优化，也侧面量化了该路径的热度。

#### 3.6 ★ BLOB 布局

```cpp
class Field_blob : public Field_longstr {
 protected:
  /// The number of bytes used to represent the length of the blob.
  uint packlength;
  /// The 'value'-object is a cache fronting the storage engine.
  String value;      // ★ 真正的数据在这里，不在行缓冲
};
```

行缓冲里的布局：

```
 ptr                       ptr+packlength
   |                            |
   v                            v
 +---------------------------+---------------------------+
 |  长度 (packlength 字节)    |  指针 (8 字节)            |
 +---------------------------+---------------------------+
        pack_length() = packlength + portable_sizeof_char_ptr
```

```cpp
  uint32 pack_length() const final { return (uint32)(packlength + portable_sizeof_char_ptr); }
  uint32 pack_length_no_ptr() const { return (uint32)(packlength); }
  uint32 max_data_length() const final { return (uint32)(((ulonglong)1 << (packlength * 8)) - 1); }
```

`portable_sizeof_char_ptr` **恒为 8**（不是 `sizeof(char*)`）——为了让行格式在不同平台/指针宽度下可移植。

| 类型 | `packlength` | `max_data_length()` | `pack_length()` |
|---|---|---|---|
| `TINYBLOB` | 1 | 255 | 9 |
| `BLOB` | 2 | 65535 | 10 |
| `MEDIUMBLOB` | 3 | 16777215 | 11 |
| `LONGBLOB` | 4 | 4294967295 | 12 |

`make_field()` 里的构造很绕——先按类型 `calc_pack_length`，再**减掉** `portable_sizeof_char_ptr` 才是 `packlength`。

**长度/指针的读写**：

```cpp
  void store_ptr_and_length(const char *from, uint32 length) {
    store_length(length);
    memmove(ptr + packlength, &from, sizeof(char *));
  }
  static uchar *get_blob_data(const uchar *position) {
    uchar *data;
    memcpy(&data, position, sizeof(data));   // ★ 用 memcpy 而非裸强转（规避对齐/别名，顺带处理大小端）
    return data;
  }
```

**推论**：BLOB 的 `pack_length()`（9~12 字节）只是"行内句柄"大小，`reclength` 里只算它——真实数据大小不计入行长（这是 BLOB 表 `reclength` 看起来很小的原因）。因为行缓冲里存的是指针，`memcpy(record[1], record[0], rec_buff_length)` **不会**深拷 BLOB 数据——所以 `Field_blob` 才需要 `old_value` / `m_blob_backup` 那一整套备份机制。

#### 3.7 ★ 整行布局 ASCII 图

算例（`utf8mb4`、两列均可空、InnoDB 默认 DYNAMIC 即 `HA_OPTION_PACK_RECORD`）：

```sql
CREATE TABLE t (a INT, b VARCHAR(10)) CHARACTER SET utf8mb4;
```

| 量 | 计算 | 值 |
|---|---|---|
| `null_fields` | a、b 均可空 | 2 |
| `leftover_bits` | packed → 0；无 BIT 列 | 0 |
| **`null_bytes`** | `(2 + 0 + 7) / 8` | **1** |
| `pack_length(a)` | INT | **4** |
| `field_length(b)` | `10 * mbmaxlen(4)` | **40** |
| `length_bytes(b)` | `HA_VARCHAR_PACKLENGTH(40)` → 40 < 256 | **1** |
| `pack_length(b)` | `40 + 1` | **41** |
| **`reclength`** | `1 + 4 + 41` | **46** |
| **`rec_buff_length`** | `ALIGN_SIZE(46 + 1 + 0)` = `ALIGN_SIZE(47)` | **48** |

（`ALIGN_SIZE(A) = MY_ALIGN(A, sizeof(double))`，即按 8 字节对齐。）

```
                         record[0] (= TABLE::null_flags)
                         |
                         v
 相对 record[0] 的偏移:  0        1 .. 4        5           6 ................ 45   46
                       +-------+------------+-----------+-------------------------+------+
                       | NULL  |  a: INT    | b: 长度头 |  b: 数据区 (40 字节)     | pad  |
                       | 位图  |  (4 字节)  | (1 字节)  |  (VARCHAR(10) utf8mb4)   | (1B) |
                       +-------+------------+-----------+-------------------------+------+
                       ^       ^            ^           ^
                       |       |            |           +-- Field_b::data_ptr() = ptr+1
          null_flags --+       |            |
                               |            +-- Field_b::ptr，offset() = 5
                 Field_a::ptr -+

 相对 record[0]-1 的偏移:
                       -1       0          1..4        5          6..........45    46
                       +-------+---------+-----------+----------+----------------+------+
                       |delete |  NULL   |  a (4B)   | b.len(1B)|  b.data (40B)  | pad  |
                       | link  |  位图   |           |          |                |      |
                       +-------+---------+-----------+----------+----------------+------+
                       ^
         缓冲区基址 base = record[0] - 1，共 rec_buff_length = 48 字节（覆盖 -1 .. 46）
```

具体值（`a = 42`，`b = 'hello'`）：

```
 偏移(相对record[0]): 0x00  0x01 0x02 0x03 0x04  0x05  0x06 0x07 0x08 0x09 0x0A ... 0x2D
                     +----+----+----+----+----+-----+----+----+----+----+----+     +----+
 内容:                |0x00| 2A | 00 | 00 | 00 | 05 | 68 | 65 | 6C | 6C | 6F | ... | 00  |
                     +----+----+----+----+----+-----+----+----+----+----+----+     +----+
                       ^    \______ a = 42 ______/   ^    \______ 'hello' (5B) ____/
                    NULL位图(全0=两列都非NULL)      b 的长度头 = 5

 **NULL 位图内部（第 0 字节，null_bytes = 1，packed 格式）**：
   bit:     7    6    5    4    3    2    1    0
          +----+----+----+----+----+----+----+----+
          | 1  | 1  | 1  | 1  | 1  | 1  | b  | a  |
          +----+----+----+----+----+----+----+----+
             \__________ 未使用 __________/  ^    ^
                          b -> null_bit = 1 << 1 = 0x02
                                             a -> null_bit = 1 << 0 = 0x01
```

**若改为非 packed 格式（无 `HA_OPTION_PACK_RECORD`）**：

```
   bit:     7    6    5    4    3    2    1    0
          +----+----+----+----+----+----+----+----+
          | 1  | 1  | 1  | 1  | 1  | b  | a  | D  |
          +----+----+----+----+----+----+----+----+
             \__________ 未使用 __________/  ^    ^
                          b -> null_bit = 1 << 2 = 0x04
                                             a -> null_bit = 1 << 1 = 0x02
                                             bit0 = "row not deleted" 标记（恒置 1）
```

**各 `Field` 指针的落地值（`open_table_from_share()` 之后）**：

```
  Field_a : ptr        = record[0] + 1     offset(record[0]) = 1
            m_null_ptr = record[0] + 0     null_offset()     = 0
            null_bit   = 0x01              pack_length()     = 4
  Field_b : ptr        = record[0] + 5     offset(record[0]) = 5
            m_null_ptr = record[0] + 0     null_bit   = 0x02
            field_length = 40   length_bytes = 1
            pack_length() = 41  data_ptr() = record[0] + 6
```

`record[1] = record[0] + rec_buff_length`，布局完全相同，所以 `offset(record[1]) == offset(record[0])`；访问另一份缓冲时用带 `row_offset` 参数的接口（`is_null(off)` / `data_length(off)`）或先 `move_field_offset()`。

#### 3.8 易错点清单

1. **`ptr` 是绝对地址**，`offset(record)` 才是行内偏移；跨缓冲用 `row_offset` 参数或 `move_field_offset()`，不要混用。
2. **`set_record_ptr()` 不存在**；真正的是 `move_field()`（绝对）、`move_field_offset()`（相对）、`set_field_ptr()`（只改 ptr）。
3. **`get_field_offset()` 只在 InnoDB 侧**；sql 层用 `Field::offset()`。
4. **`pack_length()`（内存）≠ `pack_length_in_rec()`（磁盘槽位）**；只有 `Field_bit` 有差异。
5. **`HA_VARCHAR_PACKLENGTH(field_length)` 只看总字节数 < 256，与 mbmaxlen 无关**。
6. **键里的 VARCHAR 恒 2 字节长度头**；取长度头一律用 `get_length_bytes()`。
7. **NULL 位：`null_flags + n/8`，掩码 `1 << (n%8)`**；n 的起点由 `HA_OPTION_PACK_RECORD` 决定（packed 从 0 起 = 前 8 列在第 0 字节；否则 bit0 是"row not deleted"，从 1 起 = 前 7 列）。
8. **`null_bytes = (null_fields + leftover_bits + 7) / 8`**，计入 `reclength` 且位于行首。
9. **`Field` 上只有 `is_nullable()`，没有 `maybe_null()`**。
10. **`field_length` 是字节数**，`char_length() = field_length / mbmaxlen`；`char_length_cache` 是为热路径省掉这次除法。
11. **BLOB 行内只有 `packlength`(1~4) + 8 字节指针**，真实数据在 `String value`。
12. **`rec_buff_length = ALIGN_SIZE(reclength + 1 + extra)`**，`+1` 是 delete-link 字节；`record[1] = record[0] + rec_buff_length`。

#### 3.9 ★ `pack()` / `unpack()`：第三套字节表示

到这里已经见过两套字节表示：SQL 层的**值**（`store()` 的输入）与 `record[0]` 的**内存记录格式**。还有第三套——`pack()`。★ 理解它的关键在于**它接收的输入就是 `record[0]` 里已有的字节**，输出一个"自描述、无空洞"的紧凑流：

| | `store()` | `pack()` |
|---|---|---|
| 输入 | SQL 层的**值**（文本 / 整数 / double / `MYSQL_TIME` / decimal） | **`record[0]` 里的字节**（`pack(to, from)` 可指定源，默认 `from = ptr`） |
| 输出 | 内存记录格式：小端、定长、CHAR 补尾空格、BLOB 存"长度+指针"、BIT 零头位进 NULL 字节区 | 自描述**紧凑流**：解引用 BLOB 实数据、去尾空格 + 长度前缀、位域合并、必要时字节序归一 |
| 校验 | 有 `type_conversion_status`（WARN / ERR、截断、字符集合法性） | **无**——不做任何 SQL 语义检查 |
| 消费者 | `record[0]` → 引擎写行 | filesort addon、binlog row image（复制） |

```cpp
// 默认实现就是 memcpy——"record 里怎么存，流里就怎么写"
size_t length = std::min<size_t>(pack_length(), max_length);
memcpy(to, from, length);
return to + length;      // ★ 返回「推进后的目标指针」——流式追加语义
```

**谁重写、谁不重写，判据很清楚**：

| 类别 | 例子 | 需要重写 `pack` 吗 |
|---|---|---|
| record 里已是"磁盘二进制" | `Field_newdate`（`my_date_to_binary`）、`Field_timef`/`Field_datetimef`（`my_*_packed_to_binary`）、`Field_new_decimal`（`decimal2bin` 已在 store 时做）、`Field_json`/`Field_geom`（blob 包装） | **不需要**——memcpy 即正确 |
| record 格式 ≠ 流格式 | `Field_string`（去尾空格 + 长度前缀）、`Field_varstring`（长度前缀）、`Field_blob`（解引用真数据）、`Field_bit`（零头位并进首字节）、`Field_enum`（按 packlength 分支）、老式 `Field_timestamp`/`Field_datetime`（**小端 → 字节序归一**） | 需要 |

⇒ **"为什么有的类型要重写 `pack`"的答案就是"它的 record 格式不是磁盘格式"**——5.6+ 的新式时间类型被改成"落盘格式就是可 memcmp 的字节"，顺带连 `pack` 也不用重写了（这正是第六章"新旧两代"的另一面）。

**★ `unpack` 的 `param_data` 是个兼容性钩子**：高位是**主库的 `real_type`**、低位是**主库的老 pack length**（供 `Table_map_log_event::save_field_metadata()` 用）。只有 `param_data == 0 || length == param_data || from_type != real_type()` 时才整体 memcpy，否则按较小者拷贝——这是"主库字段比从库短"的复制兼容设计，也说明 **`pack` / `unpack` 不是对称的逆运算**。

（`pack` 在 filesort addon 里的用法详见 [`query/runtime/01_filesort.md`](query/runtime/01_filesort.md)，这里不重复。）

#### 3.10 ★ `pack()` / `unpack()`：接口、语义与默认实现

> **先澄清归属（这是最常被问错的一点）**：`pack()` / `unpack()` **不是网络层的**。三条硬证据：① 它是 `Field` 自己的**虚成员函数**（`[F1] sql/field.h`），与 `store()` / `val_*()` / `send_to_protocol()` 并列；② 客户端协议栈（`sql/protocol.cc`、`protocol_classic.cc`、`Protocol_text`/`Protocol_binary`）中**没有任何 `pack()` 调用**——发给客户端的出口是 `send_to_protocol()`（见 2.8）；③ 真实消费者只有两处（全仓 grep 确认）：**复制的 binlog row image**（`sql/rpl_record.cc`）与 **filesort 的 packed addons**（`sql/filesort.cc`）。

```cpp
// [F1] sql/field.h
  virtual uchar *pack(uchar *to, const uchar *from, size_t max_length) const;
  uchar *pack(uchar *to) const { return pack(to, ptr, UINT_MAX); }   // 便利重载
  virtual const uchar *unpack(uchar *to, const uchar *from, uint param_data);
  const uchar *unpack(const uchar *from) { return unpack(ptr, from, 0U); }
```

源码注释里那句最关键的话：*"...it is used among others in the replication log, so you cannot change it without incurring a format break."*

**★ 三个语义要点**：

1. **输入是"record 里已有的字节"，不是 SQL 值**——`from` 是 `const uchar*`，通常是 `field->field_ptr()`（即 `record[0] + offset`）；因为是显式参数，所以也能指向 `record[1]` 或任意缓冲——`pack_row()` 就是用 `field->field_ptr() + rec_offset` 处理 before-image 的。它**不做类型转换、不产生 warning**。
2. **返回值是"推进后的游标"**——流式追加语义。整个 row image / addon 缓冲**没有任何 per-column 偏移表**，靠的正是 pack 格式自身的自描述性（长度前缀）。
3. **`max_length` 是输出缓冲的硬上界**——默认取 `min(pack_length(), max_length)`。⚠️ 注释明确警告：被截断后的输出**未必能被 `unpack()` 正确解回**。

```cpp
// [F2] sql/field.cc —— 默认实现就是"record 里怎么存，流里就怎么写"
uchar *Field::pack(uchar *to, const uchar *from, size_t max_length) const {
  size_t length = std::min<size_t>(pack_length(), max_length);
  memcpy(to, from, length);
  return to + length;
}

const uchar *Field::unpack(uchar *to, const uchar *from, uint param_data) {
  uint length = pack_length();
  int from_type = 0;
  if (param_data > 255) {                       // ★ 高位字节是 real_type
    from_type   = (param_data & 0xff00) >> 8U;  //   低位字节是原 pack length
    param_data  = param_data & 0x00ff;
  }
  if ((param_data == 0) || (length == param_data) || (from_type != real_type())) {
    memcpy(to, from, length);                   // ① 同构 → 全拷
    return from + length;
  }
  uint len = (param_data && (param_data < length)) ? param_data : length;
  memcpy(to, from, param_data > length ? length : len);   // ② 窄→宽：只拷低地址端
  return from + len;
}
```

还有一个只在复制路径用的变体 `pack_with_metadata_bytes()`：它的 `max_length` 含义是"**输入**的最大字段长度（必须恰好是 `max_data_length()`）"，且**不检查 `to` 的边界**——因为 binlog 里 blob 的长度前缀字节数必须与 `Table_map` 声明一致，即使数据被截断也要把前缀写全。

#### 3.11 ★ 各类型的 `pack()` 重写

**整数族与时间类型的公共助手**（`pack_int16/24/32/64`）：★ **packed 流的整数一律小端**——`pack` 方向是 `from = table->s->db_low_byte_first`（record 自己的字节序）→ `to = true`（固定小端）；`unpack` 相反。

```cpp
// [F2] sql/field.cc
static inline void handle_int16(uchar *to, const uchar *from, size_t max_length,
                                bool low_byte_first_from, bool low_byte_first_to) {
  int16 val; uchar buf[sizeof(val)];
  if (low_byte_first_from) val = sint2korr(from); else val = shortget(from);
  if (low_byte_first_to)   int2store(buf, val);  else shortstore(buf, val);
  if (max_length >= sizeof(buf)) memcpy(to, buf, sizeof(buf));
  else                           memcpy(to, buf, max_length);
}
uchar *Field::pack_int16(uchar *to, const uchar *from, size_t max_length) const {
  handle_int16(to, from, max_length, table->s->db_low_byte_first, true);
  return to + sizeof(int16);
}
```

同一套 `handle_intN` 撑起全部六个助手（★ **`pack` 方向 `to` 恒为小端、`unpack` 方向 `from` 恒为小端**，中间再用本地 `db_low_byte_first` 与 record 对接）：

```cpp
// [F2] sql/field.cc —— int24
uchar *Field::pack_int24(uchar *to, const uchar *from, size_t max_length) const {
  handle_int24(to, from, max_length, table->s->db_low_byte_first, true);
  return to + 3;
}
const uchar *Field::unpack_int24(uchar *to, const uchar *from) const {
  handle_int24(to, from, UINT_MAX, true, table->s->db_low_byte_first);
  return from + 3;
}
// [F2] —— int32 / int64 结构完全相同
uchar *Field::pack_int32(uchar *to, const uchar *from, size_t max_length) const {
  handle_int32(to, from, max_length, table->s->db_low_byte_first, true);
  return to + sizeof(int32);
}
const uchar *Field::unpack_int32(uchar *to, const uchar *from) const {
  handle_int32(to, from, UINT_MAX, true, table->s->db_low_byte_first);
  return from + sizeof(int32);
}
uchar *Field::pack_int64(uchar *to, const uchar *from, size_t max_length) const {
  handle_int64(to, from, max_length, table->s->db_low_byte_first, true);
  return to + sizeof(int64);
}
const uchar *Field::unpack_int64(uchar *to, const uchar *from) const {
  handle_int64(to, from, UINT_MAX, true, table->s->db_low_byte_first);
  return from + sizeof(int64);
}
// unpack_int16 同理（略）
```

| 助手 | 字节数 | `pack` 返回 | 使用者 |
|---|---|---|---|
| `pack_int16` / `unpack_int16` | 2 | `to + 2` | `Field_short`、`Field_enum`(packlength=2) |
| `pack_int24` / `unpack_int24` | 3 | `to + 3` | `Field_medium`、`Field_enum`(packlength=3) |
| `pack_int32` / `unpack_int32` | 4 | `to + 4` | `Field_long`、老式 `Field_timestamp`、`Field_enum`(4) |
| `pack_int64` / `unpack_int64` | 8 | `to + 8` | `Field_longlong`、老式 `Field_datetime`、`Field_enum`(8) |

★ 注意两点：① `handle_int24` 用 `sint3korr`/`int3store`（3 字节没有原生类型，靠手工拼）；② 整数类型的 `pack` **返回值恒等于 `to + sizeof(intN)`**，即使被 `max_length` 截断——**截断只影响内容、不影响游标推进**，这正是注释里"截断后 contents are not unpackable"的一个来源。

**★ `Field_string::pack`（去尾空格 + 长度前缀）**——这是"为什么不用 `record[0]`"的最直接理由：

```cpp
// [F2] sql/field.cc
uchar *Field_string::pack(uchar *to, const uchar *from, size_t max_length) const {
  uint length = my_charpos(field_charset, from, from + field_length,
                           field_length / field_charset->mbmaxlen);
  uint length_bytes = (field_length > 255) ? 2 : 1;      // ★ 由【声明长度】决定，非实际数据
  if (field_charset->mbmaxlen == 1) {                    // 去尾空格
    while (length && from[length - 1] == field_charset->pad_char) length--;
  } else
    length = field_charset->cset->lengthsp(field_charset, (const char *)from, length);

  if (max_length < length_bytes) length = 0;
  else if (length > max_length - length_bytes) length = max_length - length_bytes;

  /* Length always stored little-endian */
  if (max_length >= 1) {
    *to++ = length & 0xFF;
    if (length_bytes == 2 && max_length >= 2) *to++ = (length >> 8) & 0xFF;
  }
  memcpy(to, from, length);
  return to + length;
}
```

```
record[0] CHAR(10) utf8:  'a' 'b' ' ' ' ' ' ' ' ' ' ' ' '   ← 定长 10 字节，尾部 pad
        ↓ Field_string::pack()：去尾空格 + 1 字节小端长度前缀
packed:  [0x02] 'a' 'b'                                     ← 3 字节
```

**★ `Field_blob::pack`（解引用真实数据）**——与 `store()` 只存指针形成鲜明对照：

```cpp
// [F2] sql/field.cc
uchar *Field_blob::pack(uchar *to, const uchar *from, size_t max_length) const {
  uint32 length = get_length(from);                 // record 里的长度
  uchar len_buf[4];
  store_blob_length(len_buf, packlength, length);   // 1/2/3/4 字节前缀
  if (packlength >= max_length) { memcpy(to, len_buf, max_length); return to + max_length; }
  memcpy(to, len_buf, packlength);
  size_t store_length = min<size_t>(length, max_length - packlength);
  if (store_length > 0)
    memcpy(to + packlength, get_blob_data(from + packlength), store_length);  // ★ 解引用指针
  return to + packlength + store_length;
}
```

```
record[0] BLOB:  ┌────────┬────────────────┐
                 │ len=3  │ 0x7f8a… (char*)│   ← ptr 处只有「长度 + 裸指针」
                 └────────┴────────┬───────┘
                                   ↓ 指向堆外 'x' 'y' 'z'
packed:          [0x03] 'x' 'y' 'z'                 ← 扁平、自描述、可 memcpy
```

**★ `Field_bit::pack`（零头位并进首字节）**——那段推导注释是理解 `from` 参数为何必须存在的钥匙：

```cpp
// [F2] sql/field.cc
uchar *Field_bit::pack(uchar *to, const uchar *from, size_t max_length) const {
  if (max_length == 0) return to + 1;
  size_t length;
  if (bit_len > 0) {
    /*
      ptr - bit_ptr = from - from_bitp
      ⇒   from_bitp = bit_ptr + from - ptr      ★ 把"ptr 与 bit_ptr 的固定偏移"搬到目标 record
    */
    uchar bits = get_rec_bits(bit_ptr + (from - ptr), bit_ofs, bit_len);
    *to++ = bits;
  }
  length = min<size_t>(bytes_in_rec, max_length - (bit_len > 0));
  memcpy(to, from, length);
  return to + length;
}
```

即：**把散落在 NULL 字节区里的零头位"并进" packed 流的首字节**，让 BIT 值在流里变成连续整体。`unpack` 里对称地用 `set_rec_bits(*from, bit_ptr + (to - ptr), ...)`（把 `from - ptr` 换成 `to - ptr`，因为 `to` 才是目标 record）。

**`Field_enum::pack`（按 packlength 分支）**：ENUM 在 record 里就是 1/2/3/4/8 字节整数下标，`pack` 用同一套 `pack_int*` 归一到小端。**`Field_set` 不重写**（继承 `Field_enum`，且 `Field_enum::pack` 是 `final`）。

**老式 `Field_timestamp::pack` / `Field_datetime::pack`**：★ 必须重写，因为它们的 record 布局**依赖引擎字节序**——

```cpp
// [F2] sql/field.cc —— 若直接 memcpy 进 binlog，主备引擎字节序不同就会解错
void Field_timestamp::store_timestamp_internal(const my_timeval *tm) {
  if (table && table->s->db_low_byte_first) int4store(ptr, tm->m_tv_sec);
  else longstore(ptr, (uint32)tm->m_tv_sec);
}
```

**★ 新式时间类型（`Field_newdate` / `Field_timef` / `Field_datetimef` / `Field_timestampf`）与 `Field_new_decimal` 为什么不重写**（grep 确认 `[F1]` 里无它们的 `pack` 声明）：因为它们的 record 格式**本身就是磁盘/线上格式**——`my_datetime_packed_to_binary()` 用 `mi_int5store`（MyISAM 系**大端** store），不依赖 `db_low_byte_first`；定长、无 pad、无指针 ⇒ 默认 memcpy 就是正确答案。

⇒ **主线结论**：`pack()` 重写的**唯一理由**是 record 布局与目标布局存在 gap，gap 只有三类——**尾部填充/长度信息外置**（string/blob/bit）、**字节序依赖引擎**（整数族/real/老式时间）、**前缀宽度依赖声明长度**（string>255、blob 1/2/3/4）。

#### 3.12 ★ `unpack` 的 `param_data` 与"为什么不是对称逆运算"

**`param_data` 的来龙去脉**（这是复制兼容的核心钩子）：

```
主库：Table_map_log_event::save_field_metadata()          [sql/log_event.cc]
        └─ field->save_field_metadata() → do_save_field_metadata()   [F2] 各子类
             → 写 1~2 字节/列（如 Field_string 写 real_type 与 field_length）
备库：table_def::table_def(...) → read_field_metadata()   [sql/rpl_utility.cc]
        └─ 把字节流译码成 uint param_data（★ 注释点明小端平台上字节序被"反转"）
       unpack_row() → unpack_field() → field->unpack(..., metadata)  [sql/rpl_record.cc]
```

各类型的编码（节选）：

| 类型 | `do_save_field_metadata` 写入 | `param_data` 含义 |
|---|---|---|
| `Field_string` | `(real_type ^ ((len & 0x300) >> 4))`, `len & 0xFF` | 高位 real_type + 低位声明长度 |
| `Field_varstring` | `int2store(field_length)` | 主库声明长度（≤65535） |
| `Field_blob` | `pack_length_no_ptr()` | blob 前缀字节数（1/2/3/4） |
| `Field_enum`/`Field_set` | `real_type`, `pack_length()` | real_type + 字节数 |
| `Field_new_decimal` | `precision`, `decimals` | 精度 + 标度 |
| `Field_bit` | `field_length % 8`, `field_length / 8` | bit_len + bytes_in_rec |

**★ 为什么 `pack` / `unpack` 不是对称的逆运算**（四条证据）：

1. **`pack()` 不知道 `param_data`，`unpack()` 必须知道**——`pack` 用**本地**的 `field_length`/`length_bytes`/`packlength` 决定前缀宽度；`unpack` 必须用**对端（主库）**的声明宽度才能读前缀。主备宽度不同时二者天然不同。
2. **`unpack()` 可以写比输入更多的字节**——`Field_string::unpack` 读完 `length` 字节后还要**补 pad 到 `field_length`**（`cset->fill(..., pad_char)`）；`Field_varstring::unpack` 在 1→2 字节场景补 `to[1] = 0`。
3. **方向的处理内容不对称**——`pack` = 去 pad / 抽指针 / 抽零头位；`unpack` = 补 pad / 写指针 / 零头位回写 NULL 字节区。
4. ★ **`Field_blob::unpack` 干脆忽略 `to` 参数、直接走 `store()`**——因为 blob 的 record 里要填的是"长度 + 指针"，`memcpy` 根本做不到，必须走完整的 `store()`（含字符集校验、`bitmap_set_bit(table->write_set, ...)`）。

**两个消费者的真实调用链**：

```cpp
// ① 复制：sql/rpl_record.cc::pack_row()（节选）
  ptrdiff_t const rec_offset = record - table->record[0];   // 支持 record[1]（before image）
  uchar *pack_ptr = row_data;
  pack_ptr += (image_column_count + 7) / 8;                 // 先跳 NULL 位图
  for (auto field : fields) {
    if (field->is_null(rec_offset)) null_bits.set(true);
    else { null_bits.set(false);
           pack_field(&pack_ptr, field, rec_offset, ...); }  // → pack_with_metadata_bytes()
  }
  return static_cast<size_t>(pack_ptr - row_data);           // ★ 游标差 = row image 长度

// ② filesort：sql/filesort.cc::make_sortkey()（节选）
  if (addon_fields->using_packed_addons()) {
    for (const Sort_addon_field &addonf : *addon_fields) {
      Field *field = addonf.field;
      if (addonf.null_bit && field->is_null()) nulls[addonf.null_offset] |= addonf.null_bit;
      else {
        to = field->pack(to, field->field_ptr(), to_end - to);   // ★ 三实参：目标游标/源/剩余容量
        if (to >= to_end) return UINT_MAX;
      }
    }
    Addon_fields::store_addon_length(p_len, to - p_len);      // 写 2 字节实际长度
  }
```

★ **一个极有说服力的证据**：`unpack_row()` 里有断言 `tabledef->calc_field_size(col_i, ...) == (uint32)(pack_ptr - old_pack_ptr)`——`calc_field_size()` 是**独立于 Field 的、纯按 metadata 计算长度**的第二套实现，它算出的长度必须与 `Field::unpack()` 实际推进的字节数**逐字节一致**。这只有在 packed 流"能靠 metadata + 流内前缀自解析"时才成立。

**为什么这两处需要"自描述紧凑流"而不是直接用 `record[0]`**：`record[0]` 是**全列定长、带尾部 pad、BLOB 只有指针、BIT 零头位在 NULL 区、字节序随引擎**的内部布局；而 row image 是**稀疏列集**（`columns_in_image`）、addon 只含输出列、且两者都需要"不依赖表定义就能切开流"与"体积最小化"（`CHAR(200)` 存 `'ab'`：200 → 3 字节）。

**四方对照**（收尾）：

| 维度 | `store()` | `val_*()` | **`pack()` / `unpack()`** | `send_to_protocol()` |
|---|---|---|---|---|
| 方向 | SQL 值 → `record[0]` | `record[0]` → SQL 值 | record 字节 ⇄ 紧凑流 | `record[0]` → 客户端 |
| 输入 | 值（char*/ll/double/decimal/TIME） | 无（`ptr` 隐式） | **record 字节**（`from` 可任意） | 无（`ptr` 隐式） |
| 校验 | **有**（范围/舍入/字符集/警告） | 有 | **无**（纯字节搬运） | 有 |
| 字节序 | 随 `db_low_byte_first`（老式） | 同上 | **归一到小端**（整数族） | 协议自定 |
| NULL | 不涉及 | 需先 `is_null()` | **NULL 列不写流**（调用方 null_bits） | `store_null()` |
| 消费者 | SQL 写、引擎接口 | 表达式/比较 | **binlog row image、filesort addon** | 结果集 |

（`pack` 在 filesort addon 里的完整版式见 [`query/runtime/01_filesort.md`](query/runtime/01_filesort.md)；复制侧见相关复制文档。）

---

### 四、字符集、比较与排序键

> 补充源码坐标：`[F18]` sql/key.cc、`[F19]` include/m_ctype.h、`[F20]` strings/ctype-simple.cc、`[F21]` strings/ctype-bin.cc、`[F22]` sql/rpl_write_set_handler.cc、`[F23]` sql/filesort.cc、`[F24]` sql/partitioning/partition_handler.cc、`[F25]` sql/sql_update.cc、`[F26]` sql/opt_sum.cc 与 sql/range_optimizer/range_analysis.cc。

#### 4.1 Field 持有的字符集信息

**基类 `Field` 的默认：一切都是 `binary`**（`[F1]`）：

```cpp
  virtual bool binary() const { return true; }
  virtual enum ha_base_keytype key_type() const { return HA_KEYTYPE_BINARY; }
  virtual uint32 key_length() const { return pack_length(); }
  virtual const CHARSET_INFO *charset() const { return &my_charset_bin; }
  const CHARSET_INFO *charset_for_protocol() const {
    return binary() ? &my_charset_bin : charset();
  }
  virtual const CHARSET_INFO *sort_charset() const { return charset(); }
  virtual bool has_charset() const { return false; }
```

就近剖析：基类 `binary()` 无条件 `true` **不是占位，而是有含义的默认值**——非字符串类型（整数、浮点、时态、BIT）都被视为"没有字符语义的字节串"，`key_type()` 默认 `HA_KEYTYPE_BINARY` 是同一立场的延伸。`charset_for_protocol()` 是"对外协议视图"：只要 `binary()` 为真就降级成 `binary`——这也是 `SHOW FULL COLUMNS` 里 `BINARY(16)`/`TIMESTAMP` 不显示真实 collation 的原因。`has_charset()` 问的是"`SHOW CREATE TABLE` 要不要补 `CHARACTER SET` 子句"，**不是**"有没有排序规则"。

**`Field_str`：真正持有 `field_charset` / `field_derivation`**（`[F1]`）：

```cpp
class Field_str : public Field {
 protected:
  const CHARSET_INFO *field_charset;
  enum Derivation field_derivation;
 public:
  const CHARSET_INFO *charset() const override { return field_charset; }
  enum Derivation derivation() const final { return field_derivation; }
  bool binary() const override { return field_charset == &my_charset_bin; }
  // An always-updated cache of the result of char_length(), because
  // dividing by charset()->mbmaxlen can be surprisingly costly compared
  // to the rest of e.g. make_sort_key().
  uint32 char_length_cache;
};
```

★ **同名陷阱 1**：`Field::binary()`（常量 `true`）与 `Field_str::binary()`（判定 `field_charset == &my_charset_bin`）是**继承覆写关系**，但语义不同——前者表达"本类族不参与字符语义"，后者表达"虽属字符串族、但字符集恰是 binary 因而退化为纯字节"。若只看名字会把"默认 true"误读成"所有字段都无字符语义"。**真正的判据永远是 `charset() == &my_charset_bin`**。

**`binary()` 退化到底丢了什么**：`Field_str::binary()` 为真 ⇒ collation 处理函数整体换成 `[F21]` 的那一套——`my_strnncollsp_binary` 直接落 `memcmp`，`my_lengthsp_binary` **原样返回 length（不剥尾空格）**：

```cpp
static int my_strnncoll_8bit_bin(const CHARSET_INFO *cs, const uchar *s,
                                 size_t slen, const uchar *t, size_t tlen,
                                 bool t_is_prefix) {
  size_t len = std::min(slen, tlen);
  int cmp = memcmp(s, t, len);
  return cmp ? cmp : (int)((t_is_prefix ? len : slen) - tlen);
}
static size_t my_lengthsp_binary(const CHARSET_INFO *cs, const char *ptr,
                                 size_t length) { return length; }
```

于是：① **大小写语义消失**（`'a' != 'A'`，而在 `utf8mb4_0900_ai_ci` 下相等）；② **重音/展开语义消失**（`'é'` 与分解形式、`'ß'` 与 `'ss'` 不再等价）；③ **尾空格语义改变**（`'a '` 与 `'a'` 在 `BINARY/VARBINARY/BLOB` 下不相等，而 PAD SPACE 字符序下相等）。

**三条支线**（`[F1]`）：

```cpp
// Field_string / Field_varstring / Field_blob：非 binary 即 true
  bool has_charset() const final { return charset() == &my_charset_bin ? false : true; }
// Field_num 与 Field_temporal 共用 numeric 字符集
  const CHARSET_INFO *charset() const final { return &my_charset_numeric; }
  enum Derivation derivation() const final { return DERIVATION_NUMERIC; }
// Field_json 显式 false（注释：JSON 列无关联字符集）
  bool has_charset() const final { return false; }
```

★ 注意**时态类是 `STRING_RESULT` 的**——这正是 `match_collation_to_optimize_range()` 那段注释要区分的坑：range 优化器必须靠它把 `DATE/DATETIME` 从"真字符串"里剔除，否则会做无意义的 collation 兼容性检查。

```
                         Field
                    (binary()=true 常量, charset()=&my_charset_bin)
        ┌───────────────┼────────────────┬──────────────┐
    Field_num       Field_temporal     Field_bit     Field_str
  charset=numeric   charset=numeric   (binary 语义)   field_charset / field_derivation
  derivation=NUM    derivation=NUM                   binary() = (cs == &my_charset_bin)
                                          ┌──────────────┼──────────────┐
                                     Field_longstr    Field_enum     ...
                                  ┌────────┼─────────┐
                     Field_string  Field_varstring  Field_blob ──> Field_json (has_charset=false)
```

#### 4.2 `cmp()` 的语义

**一族六个入口（★ 同名陷阱 2）**（`[F1]`）：

```cpp
  virtual int cmp(const uchar *, const uchar *) const = 0;                  // 逻辑比较
  virtual int cmp_max(const uchar *a, const uchar *b, uint max_len) const { return cmp(a, b); }
  virtual int cmp_binary(const uchar *a, const uchar *b,
                         uint32 max_length = ~0L) const { return memcmp(a, b, pack_length()); }
  virtual int cmp_offset(ptrdiff_t row_offset) const { return cmp(ptr, ptr + row_offset); }
  virtual int cmp_binary_offset(ptrdiff_t row_offset) const {
    return cmp_binary(ptr, ptr + row_offset);
  }
  virtual int key_cmp(const uchar *a, const uchar *b) const { return cmp(a, b); }
```

- `cmp(a,b)`（纯虚）= **逻辑比较**，按类型语义；返回值规范化成 `-1/0/1`。
- `cmp_binary(a,b)` = **物理比较**，基类就是 `memcmp(pack_length())`，**返回字节差值**（不是 -1/0/1），完全忽略 collation。
- `*_offset(row_offset)` = `record[0]` vs `record[1]` 的偏移版（相距 `rec_buff_length`），用于"这行到底改没改"。
- `key_cmp` = 与**索引键格式**比较。

**整数**：`Field_tiny::cmp` 解出字节后按 `is_unsigned()` 决定用 `(uchar)` 还是 `(signed char)` 解释，返回 -1/0/1。**时态**：`Field_timestamp::cmp` 先按 `db_low_byte_first` 解包，再把结果**当无符号**比（`(uint32)a`），让早于 1970 的负时间戳落在正确位置。

★ **字符串：`Field_string::cmp` 走 collation，不走字节**（`[F2]`）：

```cpp
int Field_string::cmp(const uchar *a_ptr, const uchar *b_ptr) const {
  size_t a_len, b_len;
  if (field_charset->mbmaxlen != 1) {   // 多字节：换算到字符边界，避免切出半个字
    uint char_len = field_length / field_charset->mbmaxlen;
    a_len = my_charpos(field_charset, a_ptr, a_ptr + field_length, char_len);
    b_len = my_charpos(field_charset, b_ptr, b_ptr + field_length, char_len);
  } else
    a_len = b_len = field_length;

  /* NO PAD 字符序要自己剥尾空格（PAD SPACE 的语义由 strnncollsp 内部完成） */
  if (field_charset->pad_attribute == NO_PAD &&
      !(current_thd->variables.sql_mode & MODE_PAD_CHAR_TO_FULL_LENGTH)) {
    a_len = field_charset->cset->lengthsp(field_charset, (const char *)a_ptr, a_len);
    b_len = field_charset->cset->lengthsp(field_charset, (const char *)b_ptr, b_len);
  }
  return field_charset->coll->strnncollsp(field_charset, a_ptr, a_len, b_ptr, b_len);
}
```

就近剖析（★ 本节最重要的事实）：**CHAR 的比较不是 `memcmp`，而是 `cs->coll->strnncollsp(...)`——按排序权重（collation）比较**。同一份字节在 `utf8mb4_0900_ai_ci` 与 `utf8mb4_0900_as_cs` 下可给出不同 `cmp` 结果，而索引顺序、range 边界、去重全部建立在这个结果上。

`Field_varstring::cmp` 委托 `cmp_max`（`[F2]`）：先读长度头（`length_bytes` 1 或 2）、跳过头，再对数据区 `strnncollsp`。`Field_blob::cmp_max` 用 `get_blob_data()` 解指针 + `get_length()` 解长度，最终仍落 `strnncollsp`——所以 `TEXT` 同样"按 collation，不按字节"。

**"按排序权重"的源码铁证**（`[F20]`）：

```cpp
int my_strnncollsp_simple(const CHARSET_INFO *cs, const uchar *a,
                          size_t a_length, const uchar *b, size_t b_length) {
  const uchar *map = cs->sort_order, *end;      // ★ 256 项权重表
  ...
  while (a < end) {
    if (map[*a++] != map[*b++]) return ((int)map[a[-1]] - (int)map[b[-1]]);
  }
  ...
```

就近剖析：`const uchar *map = cs->sort_order;` —— 每字节先过权重表再比。`latin1_swedish_ci` 里 `'a'` 与 `'A'` 的 `sort_order` 值相同 → `cmp` 得 0；`my_charset_bin` 的 handler 换成 `my_strnncollsp_binary`，`sort_order` 退化为恒等映射 + `memcmp`，"字符语义"就此消失。**尾空格也在权重域里处理**（与 `map[' ']` 比，而非 `0x20`）——所以"尾空格语义"同样是 collation 的一部分。

#### 4.3 `cmp_binary()`：物理比较

基类一次 `memcmp(pack_length())`，**返回值是字节差值**、**完全忽略 collation**。定长类型（INT/FLOAT/DATE/CHAR/BINARY/ENUM）都吃默认实现——`Field_string` 就没有覆写 `cmp_binary`，所以 CHAR 的 `cmp_binary` 是把整段 `field_length` 字节（含尾空格填充）一起 memcmp 的。

三个真正覆写它的变长类型：`Field_varstring`（先解长度头、等长才 memcmp，**长度不等时直接 `return 1`**——是"相等性判定 + 等长字节序"，不是全序）；`Field_blob`（解 `packlength` 长度与指针）；`Field_json`（解析 binary JSON 后按 JSON 值语义比较）。

★ **`Field_bit::cmp_binary()` 为什么只能用基类版本**（`[F1]` 类前注释原文）：

```cpp
/*
  Note:
    To use Field_bit::cmp_binary() you need to copy the bits stored in
    the beginning of the record (the NULL bytes) to each memory you
    want to compare (where the arguments point).
    This is the reason:
    - Field_bit::cmp_binary() is only implemented in the base class (Field::cmp_binary()).
    - Field::cmp_binary() currently uses pack_length() to calculate how long the data is.
    - pack_length() includes size of the bits stored in the NULL bytes of the record.
*/
```

就近剖析（★ 同名陷阱 3）：`Field_bit` **根本没有自己的 `cmp_binary`**。原因链：`BIT(n)` 的"零头位"寄存在记录的 NULL 位区（`bit_ptr`）→ `pack_length()` 把这段也算进长度 → 基类 `cmp_binary` 按 `pack_length()` memcmp，会**越过 `ptr` 一直比到 NULL 区**。所以只有调用方把记录开头的 NULL 字节也拷过去它才有意义；否则 `Field_bit` 自己用 `cmp`/`key_cmp` 手工拼零头位。

**`cmp_binary` 的真实用途**（`[F25]`）：判断 UPDATE 是否真的改动了行——

```cpp
        if (field->cmp_binary_offset(table->s->rec_buff_length)) return true;
```

就近剖析：`record[0]`（旧值）与 `record[1]`（新值）相距 `rec_buff_length`；这里要的正是**忽略 collation 的物理相等**——否则把 `'a'` 改成 `'A'`（ci 下逻辑相等）会误判为"未修改"，binlog 就丢了。这是 `cmp` 与 `cmp_binary` 分工最清晰的例子。★ 注意：**filesort 并不调用 `Field::cmp_binary`**（8.0 的排序键是 Item 自造的）——"filesort 用 binary 比较"是老版本说法。

#### 4.4 ★ `get_key_image()` vs `make_sort_key()`

二者名字都像"造一个可以 memcmp 的键"，但**目的、格式、调用者完全不同**：

```
   ┌─────────────── 行内格式 record[0] ───────────────┐
   └───────┬──────────────────────────────────┬───────┘
           │ get_key_image()                  │ make_sort_key()
           ▼                                  ▼
   ┌──────────────────────┐        ┌────────────────────────────┐
   │ 索引键 (key format)   │        │ 排序键 (sort key)           │
   │ CHAR: 补 pad_char     │        │ strnxfrm 权重               │
   │ VARCHAR/BLOB: 2B 长度头│        │ 整数: 大端 + 符号位翻转      │
   │ 不做 collation 变换   │        │ 定长、可 memcmp             │
   │ 交引擎按 HA_KEYTYPE 解释│        │ 只给复制 write set 用(8.0)  │
   └──────────────────────┘        └────────────────────────────┘
     调用者: key_copy() / opt_sum  │ 调用者: rpl_write_set_handler.cc
            / range_analysis      │        (filesort 已不用!)
```

**`get_key_image()` 的契约**（`[F1]`）：`length` 单位是字节，但对变长字符集它等于"字符数 × mbmaxlen"（NOTES 原文）。基类实现 = `get_image()` = 一次 `memcpy`——**整数/浮点/时态/ENUM 全部吃这个默认版本**：原地拷贝，不翻字节序、不翻符号位。这与 `make_sort_key` 里整数要翻转符号位形成鲜明对比——因为索引键交给引擎后由引擎自己的 key 比较器（按 `HA_KEYTYPE_*`）解释，不需要 MySQL 层预加工。`itMBR` 只对几何列有意义（取 MBR 作键）。

**主调用者 `key_copy()`**（`[F18]`）：

```cpp
    if (key_part->null_bit) {                       // 可空段：先写 1 字节 NULL 标记
      bool key_is_null = from_record[key_part->null_offset] & key_part->null_bit;
      *to_key++ = (key_is_null ? 1 : 0);
      key_length--;
    }
    if (key_part->key_part_flag & HA_BLOB_PART ||
        key_part->key_part_flag & HA_VAR_LENGTH_PART) {
      key_length -= HA_KEY_BLOB_LENGTH;             // 变长段：预留 2 字节长度头
      length = min<uint>(key_length, key_part->length);
      key_part->field->get_key_image(to_key, length, Field::itRAW);
      to_key += HA_KEY_BLOB_LENGTH;
    } else {
      length = min<uint>(key_length, key_part->length);
      size_t bytes = field->get_key_image(to_key, length, Field::itRAW);
      if (bytes < length)                           // ★ 定长段：key_copy 自己补空格
        cs->cset->fill(cs, (char *)to_key + bytes, length - bytes, ' ');
    }
```

就近剖析：定长分支由 `key_copy` **自己补空格**（与 `get_key_image` 注释里"变长类型内部填 0"不矛盾——后者是为了 `opt_range` 的 memcmp 去重）。

**三个字符串类型的 `get_key_image`**（`[F2]`）：

```cpp
// CHAR：补 pad_char，无长度头
size_t Field_string::get_key_image(uchar *buff, size_t length, imagetype) const {
  size_t bytes = my_charpos(field_charset, (char *)ptr, (char *)ptr + field_length,
                            length / field_charset->mbmaxlen);
  memcpy(buff, ptr, bytes);
  if (bytes < length)
    field_charset->cset->fill(field_charset, (char *)buff + bytes,
                              length - bytes, field_charset->pad_char);
  return bytes;                                     // ★ 返回"有效字节数"，不含填充
}

// VARCHAR：★ 键里恒 2 字节小端长度头 + 尾部清零
size_t Field_varstring::get_key_image(uchar *buff, size_t length, imagetype) const {
  uint f_length = is_null() ? 0 : data_length();
  uint local_char_length = length / field_charset->mbmaxlen;
  uchar *pos = ptr + length_bytes;
  local_char_length = my_charpos(field_charset, pos, pos + f_length, local_char_length);
  f_length = std::min(f_length, local_char_length);
  /* Key is always stored with 2 bytes */
  int2store(buff, f_length);
  memcpy(buff + HA_KEY_BLOB_LENGTH, pos, f_length);
  if (f_length < length) {
    /* Must clear this as we do a memcmp in opt_range.cc to detect identical keys */
    memset(buff + HA_KEY_BLOB_LENGTH + f_length, 0, (length - f_length));
  }
  return HA_KEY_BLOB_LENGTH + f_length;
}
```

就近剖析：① 行内 1/2 字节头、**键里永远 2 字节小端**；② 尾部 `memset 0` 而非空格，注释直接给出理由——`opt_range.cc` 用 memcmp 判定"两键是否相同"，未清零的垃圾字节会误判；③ `is_null()` 时长度写 0（不去读引擎可能留下的随机字节）。

`Field_blob::get_key_image` 的 `itRAW` 分支与 VARCHAR 同构；`itMBR` 分支是空间索引专用——把几何对象换成 4 个 `double` 的 MBR 并**强制转小端**（`float8store`），因为 R-tree 键值必须跨平台一致。

**★ `make_sort_key()` 的 8.0.39 现状**（`[F1]` 注释原文）：

```cpp
  /**
    Writes a copy of the current value in the record buffer, suitable for
    sorting using byte-by-byte comparison. Integers are always in big-endian
    regardless of hardware architecture. ...
    @note This is now only used by replication; filesort makes its own
     sort keys based off of Items, not Fields.
  */
  virtual size_t make_sort_key(uchar *buff, size_t length) const = 0;
```

就近剖析（★ 现状确认）：这条 `@note` 在 8.0.39 的**两个重载上都存在**。即：`make_sort_key` 的**唯一生产调用者是复制**（`[F22]`），filesort 早已改为"基于 Item 自造排序键"。★ 另有一处**陈旧注释**会误导人——`sql/sort_param.h` 里仍写着 "Fields are fixed-size, specially encoded with `Field::make_sort_key()`..."，那是历史遗留，以 `field.h` 的 `@note` 为准。

**filesort 现在怎么造键**（`[F23]` 证据）：

```cpp
    case STRING_RESULT: {
      const CHARSET_INFO *cs = item->collation.collation;    // ★ 取自 Item 而非 Field
      String *res = item->val_str(tmp_buffer);
      ...
      actual_length = cs->coll->strnxfrm(
          cs, to, dst_length.value(), item->max_char_length(),
          pointer_cast<const uchar *>(from), src_length, MY_STRXFRM_PAD_TO_MAXLEN);
      ...
```

就近剖析：filesort 的键来源是 **`Item`**（`item->val_str()` / `int_sort_key()` / `collation`），`strnxfrm` 在这里被直接调用。这带来两个 `Field::make_sort_key` 做不到的能力：① 排序表达式可以是任意 Item（函数、`ORDER BY a+b`），不只是列；② 支持 varlen 模式，键不必定长。

**`make_sort_key` 各类型的实现**（`[F2]`）：

```cpp
// CHAR / VARCHAR / BLOB：都走 strnxfrm（变换成权重串）
size_t Field_string::make_sort_key(uchar *to, size_t length, size_t trunc_pos) const {
  size_t input_length = std::min<size_t>(
      field_length,
      field_charset->cset->charpos(field_charset, (const char *)ptr,
                                   (const char *)ptr + field_length, trunc_pos));
  if (field_charset->pad_attribute == NO_PAD &&
      !(current_thd->variables.sql_mode & MODE_PAD_CHAR_TO_FULL_LENGTH)) {
    input_length = field_charset->cset->lengthsp(field_charset, (const char *)ptr,
                                                 input_length);
  }
  assert(char_length_cache == char_length());
  size_t tmp [[maybe_unused]] = field_charset->coll->strnxfrm(
      field_charset, to, length, char_length_cache, ptr, input_length,
      MY_STRXFRM_PAD_TO_MAXLEN);
  assert(tmp == length);                            // ★ 输出必须精确等于 length
  return length;
}

// 整数：★ 大端 + 有符号翻转符号位（get_key_image 一步都不做）
size_t Field_tiny::make_sort_key(uchar *to, size_t length) const {
  assert(length == 1);
  if (is_unsigned())
    *to = *ptr;
  else
    to[0] = (char)(ptr[0] ^ (uchar)128); /* Revers signbit */
  return 1;
}
```

就近剖析：排序键要求"字节序即数值序"，所以必须①统一转大端；②有符号数翻转最高位（把 `-128..127` 单调映射到 `0..255`）。这两步 `get_key_image` **一步都不做**——这是两者最本质的分歧。

**调用者为什么必须是 `make_sort_key`**（`[F22]`）：组复制用它生成 PKE（Primary Key Equivalent）——不同节点上的同一逻辑值（如 `'A'` 与 `'a'`）必须产生**完全相同**的字节，否则认证会误判冲突；这只有 collation 归一化的 `strnxfrm` 能做到（缓冲区大小由 `cs->coll->strnxfrmlen()` 给出）。

**差异总表**：

| 维度 | `get_key_image()` | `make_sort_key()` |
|---|---|---|
| 目的 | 构造**索引键**（row → key format） | 构造**排序键**（value → 可 memcmp 的权重串） |
| 主调用者 | `key_copy()`、`opt_sum.cc`、`range_analysis.cc` | **仅复制** `rpl_write_set_handler.cc`（filesort 已不用） |
| 长度头 | VARCHAR/BLOB **恒 2 字节小端**；CHAR 无 | **无**（定长，padding 到 `length`） |
| 尾部填充 | CHAR 填 `pad_char`；VARCHAR/BLOB 填 `0`（为 `opt_range` memcmp 去重） | `strnxfrm` + `MY_STRXFRM_PAD_TO_MAXLEN` |
| 是否按 collation | **否**（原字节，交引擎解释） | **是**（走 `strnxfrm`，即 `sort_order`/UCA 权重） |
| 整数 | 原样 `memcpy` | 转大端 + 翻符号位 |
| 特殊 | `itMBR`（几何取 MBR，4×double 小端） | 无 |
| 反向 | `set_key_image()` / `key_restore()` | **无**（`strnxfrm` 有损不可逆） |

#### 4.5 `sort_order` / `strnxfrm` 的作用

`[F20]` 的注释给出了整章的理论核心：

```
     Comparing two strings using memcmp() after my_strnxfrm_xxx()
     is equal to comparing two original strings with my_strnncollsp_xxx().
```

即 ★ **`memcmp(strnxfrm(a), strnxfrm(b))` ≡ `strnncollsp(a, b)`**——`cmp()`（一次比较）与 `make_sort_key()`（一次变换、之后可反复 memcmp）语义等价，只是代价分摊不同（排序要比较 O(n log n) 次，预变换更划算）。

实现就是 `sort_order` 查表（8bit 简单字符序，`[F20]`）：`const uchar *map = cs->sort_order;` 每字节换成权重，循环手工展开 8 路，最后 `my_strxfrm_pad` 按 `flags` 补齐尾部（`MY_STRXFRM_PAD_TO_MAXLEN` 生效处）。

★ **binary 下的退化**（`[F21]`）：`my_charset_bin` 的 `strnxfrm` 就是一次 `memcpy`（+ 可选补 `pad_char`）。所以**当且仅当 `Field_str::binary()` 为真时，`make_sort_key` 退化成"字节原样拷贝"**，大小写/重音/尾空格语义同时消失——与 4.1.3 完全闭合。对比 `latin1_bin` 等非 binary 的 `_bin` 字符序用的是 `my_strnxfrm_8bit_bin_pad_space`（**会** pad），而 binary 的 `no_pad` 版默认不 pad——这是 `BINARY(N)` 与 `CHAR(N) COLLATE ..._bin` 行为差异的又一处来源。

#### 4.6 `Field::hash()` 与哈希场景

```cpp
void Field::hash(ulong *nr, ulong *nr2) const {
  if (is_null()) {
    *nr ^= (*nr << 1) | 1;                    // NULL 有独立的扰动分支
  } else {
    uint len = pack_length();
    const CHARSET_INFO *cs = sort_charset();
    uint64 tmp1 = *nr, tmp2 = *nr2;
    cs->coll->hash_sort(cs, ptr, len, &tmp1, &tmp2);
    // NOTE: This truncates to 32-bit on Windows, to keep on-disk stability.
    *nr = static_cast<ulong>(tmp1);
    *nr2 = static_cast<ulong>(tmp2);
  }
}
```

就近剖析三点：① NULL 独立扰动（保证与任何非空值不同哈希）；② 用 **`sort_charset()`** 的 `hash_sort`——**按 collation 哈希**，ci 下 `'A'` 与 `'a'` 同哈希，这是分区/去重正确性的前提；③ 显式截断 32-bit 为的是**跨版本跨平台磁盘布局稳定**（注释原文），不能"优化"成 64 位。

`hash_sort` 的实现（`[F20]`）同样走 `sort_order`，并主动 `skip_trailing_space` 剥尾空格（注释：`"Remove end space. We have to do this to be able to compare 'A ' and 'A' as identical"`）。

两个覆写：`Field_varstring::hash`（因为基类用 `pack_length()` 会把长度头算进去，而"长度头相同的不同值应区分、长度头不同的相同值应合并"）；`Field_bit::hash`（同 `cmp_binary` 的理由——改为 `val_int()` 取逻辑值再按 `my_charset_bin` 哈希 8 字节）。

**与 hash 分区的关系**（`[F24]`）：`KEY` 分区的哈希由 `Field::hash()` 逐字段累积（`nr1/nr2` 初值 1/4）；为兼容 5.1 布局保留了 `use_51_hash` 分支——对数值/时态强制 `my_hash_sort_bin`，对 STRING/VARCHAR/BIT 落到 `field->hash()`。★ 注释 `@note Uses the hash function on the character set of the field.` 正是"按 collation 哈希"的直接后果：**分区函数对字符串列是 collation 敏感的**，改列 collation 会改变哈希值。

★ **注意与 hash join 的区别**：8.0 的 hash join（`sql/hash_join_*.cc`）**不使用 `Field::hash()`**，它在 Iterator 层用自己的哈希（基于 Item）。这与 `make_sort_key` 的"filesort 已不用"是同一类历史遗留造成的认知偏差。

#### 4.7 小结：三条编码路径的分工

```
                          Field（持有 field_charset / field_derivation）
                                        │
        ┌───────────────────────────────┼────────────────────────────────┐
   逻辑比较 cmp()                  物理比较 cmp_binary()            编码键
   (按 collation: strnncollsp)      (memcmp, 忽略 collation)              │
   谁用: record 比较 / range        谁用: UPDATE 变更判定 / ndb       ┌───────┴───────┐
                                   ★ BIT 只能用基类版           get_key_image  make_sort_key
                                                              (索引键)        (排序键)
                                                            key_copy()     仅复制 write set
                                                            CHAR 补空格    strnxfrm 权重
                                                            VARCHAR 2B头   整数翻符号位+大端
                                                            itMBR(几何)
```

- **`cmp()` 是语义的**：字符串走 `cs->coll->strnncollsp`（内部是 `sort_order` 权重表），绝不能当 `memcmp`。
- **`cmp_binary()` 是物理的**：基类 `memcmp(pack_length())`；`Field_bit` 因 `pack_length()` 跨 NULL 区而只能用基类版并附加强制前置条件。
- **`get_key_image()` 造索引键**：原字节 + 长度头/补空格，**不做 collation 变换**，由引擎按 `HA_KEYTYPE_*` 解释。
- **`make_sort_key()` 造排序键**：走 `strnxfrm`、整数转大端+翻符号位，**8.0.39 里只剩复制在用**（`field.h` 注释原文为证）。
- **`binary()` 是贯穿四者的总开关**：`charset() == &my_charset_bin` 时 collation 函数整体退化为 `memcmp`/`memcpy`。

### 五、与 Item / TABLE / handler / DD 的接缝

> 补充源码坐标：`[F27]` sql/sql_base.cc、`[F28]` sql/dd/dd_table.cc、`[F29]` storage/innobase/handler/ha_innodb.cc、`[F30]` storage/innobase/include/row0mysql.h、`[F31]` storage/innobase/row/row0sel.cc（其余沿用章首表：`[F1]`/`[F2]`/`[F3]`/`[F4]`/`[F5]`/`[F6]`/`[F10]`/`[F11]`/`[F13]`/`[F14]`/`[F15]`/`[F17]`）。

#### 5.0 总览：Field 是四个世界的"唯一真源"

```
                        ┌──────────────────────────────────────────┐
                        │   dd::Column (sql/dd/types/column.h)     │
                        │   持久化元数据：类型/长度/隐藏/默认值     │
                        └───────────────┬──────────────────────────┘
                                        │ make_field() / fill_column_from_dd()  [F14]
                                        ▼
     ┌────────────────────────────────────────────────────────────────────┐
     │  TABLE_SHARE::field[]  (Field** —— 模板，1 份/表定义)               │
     │  ptr 指向 share->default_values（不是真正的行缓冲）                 │
     └───────────┬────────────────────────────────────────────────────────┘
                 │  open_table_from_share(): share->field[i]->clone(root)
                 │  new_field->init(table); new_field->move_field_offset(move_offset)   [F13]
                 ▼
     ┌────────────────────────────────────────────────────────────────────┐
     │  TABLE::field[]       (Field** —— 实例，1 份/TABLE)                 │
     │  每个 Field::ptr 指向 本 TABLE 的 record[0] 内部偏移                │
     └───────┬──────────────────────────────────┬─────────────────────────┘
             │ 语义方向：val_*() / store*()      │ 布局方向：offset()/pack_length()/null_bit
             ▼                                   ▼
   ┌──────────────────┐            ┌──────────────────────────────────┐
   │  Item_field      │            │  handler (ha_innobase)           │
   │  Item::save_in_  │            │  build_template_field() 造       │
   │  field(Field*)   │            │  mysql_row_templ_t，之后纯 memcpy│
   └──────────────────┘            └──────────────────────────────────┘
```

一句话概括分层：**DD 给"定义"，TABLE_SHARE 给"模板"，TABLE 给"行缓冲绑定"，Item 用"语义"，handler 只用"布局"**。

#### 5.1 与 `Item` 的接缝

**`Item_field` 如何持有 `Field*`**——它同时持有 `field`（源列）与 `result_field`（写入目标列），后者默认是前者：

```cpp
// [F3] sql/item.h
class Item_field : public Item_ident {
 protected:
  void set_field(Field *field);
 public:
  Table_ref *table_ref;
  /// Source field
  Field *field;
 private:
  /// Result field
  Field *result_field{nullptr};
```

★ `result_field` 独立存在是因为一个 `Item_field` 既可能"读"（从 `field` 取 `val_*()`），也可能"写"（物化进临时表的某列）。`mark_field_in_map()` 把**两个都**登记进位图，正因为读写两侧的 Field 不是同一对象。

`set_field()` 是把 `Field*` "吸收"进 Item 的唯一入口，一次性把 Item 需要的全部类型信息从 Field 拷出来——**这是 Item 与 Field 之间最重要的"属性单向快照"**：

```cpp
// [F4] sql/item.cc
void Item_field::set_field(Field *field_par) {
  table_ref = field_par->table->pos_in_table_list;
  assert(field_par->field_index() != NO_FIELD_INDEX);
  field_index = field_par->field_index();

  field = result_field = field_par;   // ★ 非临时表场景二者是同一指针
  set_nullable(field->is_nullable() || field->is_tmp_nullable() ||
               field->table->is_nullable());
  ...
  m_orig_field_name = field_par->field_name;
  collation.set(field_par->charset(), field_par->derivation(),
                field_par->repertoire());
  set_data_type(field_par->type());
  decimals = field->decimals();
  unsigned_flag = field_par->is_flag_set(UNSIGNED_FLAG);
  max_length = char_to_byte_length_safe(field_par->char_length(),
                                        collation.collation->mbmaxlen);
  max_length = adjust_max_effective_column_length(field_par, max_length);
  ...
  fixed = true;
}
```

`result_type()` / `val_*()` 是**纯转发**——先 `field->is_null()` 判空并写 `Item::null_value`，再调 `field->val_xxx()`：

```cpp
// [F3] sql/item.h
  Item_result result_type() const override { return field->result_type(); }
  Item_result numeric_context_result_type() const override {
    return field->numeric_context_result_type();
  }
  bool is_null() override { return field->is_null(); }
```

```cpp
// [F4] sql/item.cc
longlong Item_field::val_int() {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return 0;
  return field->val_int();
}
String *Item_field::val_str(String *str) {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return nullptr;
  str->set_charset(str_value.charset());
  return field->val_str(str, &str_value);     // ★ 双缓冲技巧的标准调用方
}
```

**★ `fix_fields()`：列名 → `Field*`**（8.0.39 有一个同名陷阱）：

它**不走** `find_field_in_table_sef()`。`_sef`（side-effect-free）版本只被分区与直方图代码用来"无副作用探查列是否存在"。真正的链是 `find_field_in_tables()` → `find_field_in_table_ref()` → `find_field_in_table()`：

```cpp
// [F4] sql/item.cc
bool Item_field::fix_fields(THD *thd, Item **reference) {
  assert(!fixed);
  Field *from_field = not_found_field;
  ...
  from_field = find_field_in_tables(
      thd, this, context->first_name_resolution_table,
      context->last_name_resolution_table, reference, ...);
```

名字解析的落点就是 `TABLE::field[]` 里的某一个 `Field*`，线性扫描 + 大小写不敏感比较：

```cpp
// [F27] sql/sql_base.cc
Field *find_field_in_table(TABLE *table, const char *name, bool allow_rowid,
                           uint *field_index_ptr) {
  Field **field_ptr = nullptr, *field;
  if (!(field_ptr = table->field)) return nullptr;
  for (; *field_ptr; ++field_ptr) {
    if (!my_strcasecmp(system_charset_info, (*field_ptr)->field_name, name))
      break;
  }
  if (field_ptr && *field_ptr) {
    *field_index_ptr = field_ptr - table->field;   // ★ 指针差算列号
    field = *field_ptr;
  } else {
    if (!allow_rowid || my_strcasecmp(system_charset_info, name, "_rowid") ||
        table->s->rowid_field_offset == 0)
      return (Field *)nullptr;
    field = table->field[table->s->rowid_field_offset - 1];
  }
  return field;
}
```

找到之后立刻把该列登记进 `read_set`/`write_set`：`fld->table->mark_column_used(fld, thd->mark_used_columns);`

**★ `Item::save_in_field(Field*)` 的完整分派链**——这是 Item → Field 的总入口：

```cpp
// [F4] sql/item.cc
type_conversion_status Item::save_in_field(Field *field, bool no_conversions) {
  // 函数索引隐藏列：装一个把"列名"改写成"索引名"的错误处理器
  Functional_index_error_handler functional_index_error_handler(field, current_thd);
  const type_conversion_status ret = save_in_field_inner(field, no_conversions);
  /* 求值出错但 status 仍是 OK 的兜底 */
  if (ret == TYPE_OK && current_thd->is_error()) return TYPE_ERR_BAD_VALUE;
  return ret;
}
```

基类 `save_in_field_inner()` 即"按 `result_type()` 分派"，而 **STRING_RESULT 分支内部还要再看一次目标 `Field::type()`**（因为字符串是万能中间表示）：

```cpp
// [F4] sql/item.cc（节选骨架）
type_conversion_status Item::save_in_field_inner(Field *field, bool no_conversions) {
  assert(!returns_array());
  if (result_type() == STRING_RESULT) {
    if (data_type() == MYSQL_TYPE_JSON) {
      const enum_field_types field_type = field->type();
      if (field_type == MYSQL_TYPE_JSON) {
        Json_wrapper wr;
        if (val_json(&wr)) return TYPE_ERR_BAD_VALUE;
        if (null_value) return set_field_to_null(field);
        field->set_notnull();
        return down_cast<Field_json *>(field)->store_json(&wr);
      }
      if (is_temporal_type(field_type) && field_type != MYSQL_TYPE_YEAR) {
        MYSQL_TIME t; bool res = true;
        switch (field_type) {
          case MYSQL_TYPE_TIME:     res = get_time(&t);    break;
          case MYSQL_TYPE_DATETIME: case MYSQL_TYPE_TIMESTAMP:
          case MYSQL_TYPE_DATE: case MYSQL_TYPE_NEWDATE:
                                    res = get_date(&t, 0); break;
          ...
        }
        if (res) { null_value = true;
                   return set_field_to_null_with_conversions(field, no_conversions); }
        field->set_notnull();
        return field->store_time(&t);
      }
      if (field_type == MYSQL_TYPE_NEWDECIMAL) { /* val_decimal → store_decimal */ }
      if (field_type == MYSQL_TYPE_INT24 || field_type == MYSQL_TYPE_TINY ||
          field_type == MYSQL_TYPE_SHORT || field_type == MYSQL_TYPE_LONG ||
          field_type == MYSQL_TYPE_LONGLONG) { /* val_int → store(ll, unsigned_flag) */ }
      if (field_type == MYSQL_TYPE_FLOAT || field_type == MYSQL_TYPE_DOUBLE) { /* val_real → store(double) */ }
    }
    /* 通用字符串路径 */
    char buff[MAX_FIELD_WIDTH];              // 小列用栈缓冲（性能技巧）
    str_value.set_quick(buff, sizeof(buff), cs);
    String *result = val_str(&str_value);
    if (null_value) { str_value.set_quick(nullptr, 0, cs);
                      return set_field_to_null_with_conversions(field, no_conversions); }
    field->set_notnull();
    type_conversion_status error = field->store(result->ptr(), result->length(), cs);
    str_value.set_quick(nullptr, 0, cs);
    return error;
  }
  if (result_type() == REAL_RESULT)    { /* val_real    → field->store(double) */ }
  if (result_type() == DECIMAL_RESULT) { /* val_decimal → field->store_decimal */ }
  longlong nr = val_int();                   // INT_RESULT / fallthrough
  if (null_value) return set_field_to_null_with_conversions(field, no_conversions);
  field->set_notnull();
  return field->store(nr, unsigned_flag);
}
```

★ **要点**：① 两级分派（Item 的 result_type → Field 的 type）；② 任何分支都是"判 `null_value` → `set_field_to_null*()`，否则 `set_notnull()` + `store*()`"；③ **落到 Field 的只有 4 类原语**——`store(const char*,len,cs)` / `store(double)` / `store(longlong,bool)` / `store_decimal()`，外加特化的 `store_json()` / `store_time()`。**Item 世界与 Field 世界的全部数据交换就收在这几个 `store*` 上**。

**★ `Item_field::save_in_field` 的快速路径：列到列拷贝**——源和目标是两个 Field 时，最优解是直接搬记录缓冲区的字节：

```cpp
// [F4] sql/item.cc
static inline type_conversion_status field_conv_with_cache(
    Field *to, Field *from, Field **last_to, uint32_t *to_is_memcpyable) {
  if (to != *last_to) {
    *last_to = to;
    if (fields_are_memcpyable(to, from)) *to_is_memcpyable = to->pack_length();
    else                                 *to_is_memcpyable = -1;
  }
  if (*to_is_memcpyable != static_cast<uint32_t>(-1) &&
      *to_is_memcpyable == to->pack_length() && from->type() == to->type() &&
      (from->type() != MYSQL_TYPE_NEWDECIMAL ||
       from->decimals() == to->decimals())) {
    const size_t length = *to_is_memcpyable;
    assert(to->field_ptr() != from->field_ptr() || length == 0);
    memcpy(to->field_ptr(), from->field_ptr(), length);   // ★ 直接 memcpy
    return TYPE_OK;
  } else {
    return field_conv_slow(to, from);
  }
}
```

```cpp
// [F4] sql/item.cc
type_conversion_status Item_field::save_in_field_inner(Field *to, bool no_conversions) {
  if (field->is_null()) { null_value = true;
    return set_field_to_null_with_conversions(to, no_conversions); }
  to->set_notnull();
  null_value = false;
  if (to == field) return TYPE_OK;   // ★ 'SET x = x' 场景
  return field_conv_with_cache(to, field, &last_destination_field,
                               &last_destination_field_memcpyable);
}
```

判断"能否 memcpy"的安全性条件集中在 `fields_are_memcpyable()`（`[F11]`）——★ **凡是"内存布局 ≠ 值语义"的类型一律不许 memcpy**：JSON（二进制 DOM）、GEOMETRY、VARCHAR（长度前缀）、ENUM/SET（下标）、BIT（位域）、BLOB（指针+外存）；连 `sql_mode` 都能否决（严格模式下日期要校验，memcpy 会绕过）。

**反向：由 Item 造 Field（临时表）**——`create_tmp_field()` 按 `Item::Type` 分两条路：

```cpp
// [F17] sql/sql_tmp_table.cc（节选）
  switch (type) {
    case Item::FIELD_ITEM:
    case Item::DEFAULT_VALUE_ITEM:
    case Item::TRIGGER_FIELD_ITEM: {
      Item_field *item_field = down_cast<Item_field *>(item);
      if (item_field->is_nullable() && !(item_field->field->is_nullable() ||
          item_field->field->table->is_nullable())) {
        result = create_tmp_field_from_item(item_field, table);      // ② 按 result_type 造
      } else if (...) {
        ...
      } else {
        result = create_tmp_field_from_field(thd, item_field->field, ...);  // ① 克隆原列
      }
      if (modify_item) item_field->set_result_field(result);         // ★ field 与 result_field 分离的时刻
      *from_field = item_field->field;
      break;
    }
```

① `create_tmp_field_from_field()`：`org_field->new_field(thd->mem_root, table)` + `init(table)`；★ 两个细节——外连接会把 `NOT_NULL` 清掉（补 NULL 行）；**用户不可见列（HT_HIDDEN_USER）进入内部临时表后被"洗白"成 HT_VISIBLE**。
② `create_tmp_field_from_item()`：按 `item->result_type()` 造列（REAL→`Field_float`/`Field_double`、INT→`Field_long`/`Field_longlong`、STRING→`make_string_field` 或 `tmp_table_field_from_field_type`、DECIMAL→`Field_new_decimal::create_from_item`）；★ 为保住 DATE/TIME/GEOMETRY/JSON 这些"result_type 是 STRING 但语义不是字符串"的类型，特地绕道 `tmp_table_field_from_field_type()`。

所有造列路径的共同收尾都是 `if (field) field->init(table);`——**没有 `init()` 的 Field 是"游离"的，其 `ptr` 尚未绑定到任何 `record[0]`**。

#### 5.2 与 `TABLE` / `TABLE_SHARE` 的接缝

**★ 为什么必须克隆**：`TABLE_SHARE::field[]` 是**元数据模板**（一份/表定义），其 Field 的 `ptr` 指向 `share->default_values`；每个 `TABLE` 实例必须有**自己的 Field 数组**，因为各自的 `record[0]` 不同：

```cpp
// [F13] sql/table.cc（open_table_from_share，节选）
  record = root->ArrayAlloc<uchar>(share->rec_buff_length * records + share->null_bytes);
  outparam->record[0] = record;
  outparam->record[1] = (records > 1) ? record + share->rec_buff_length : outparam->record[0];

  outparam->field = field_ptr;
  record = (uchar *)outparam->record[0] - 1;   /* Fieldstart = 1 —— delete-link 字节 */
  outparam->null_flags = (uchar *)record + 1;  /* record[0] 开头 */

  move_offset = outparam->record[0] - share->default_values + (internal_tmp ? 2 * share->rec_buff_length : 0);

  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);   // ★ 虚函数：new (mem_root) Field_xxx(*this)
    *field_ptr = new_field;
    new_field->init(outparam);                         // 只设 table / table_name
    new_field->move_field_offset(move_offset);         // ★ ptr 与 m_null_ptr 一起平移
    if (!has_default_values) new_field->reset();
  }
  (*field_ptr) = nullptr;                              // 末尾哨兵
```

三步的价值：`clone()` 复制全部布局信息 → `init()` 绑定 TABLE → `move_field_offset(diff)` **一次减法完成重绑**（避免逐字段重算偏移）。代价是每个 TABLE 一份 Field；收益是取值/写入都是一次指针解引用、无间接层。

**★ `record[0]` / `Field::ptr` / `null_flags` 的关系**：

```
  TABLE::record[0]  (长度 = rec_buff_length = ALIGN_SIZE(reclength + 1 + extra))
  ├─► null_flags ─── 前置区(preamble)，长度 = null_bytes
  │     bit0: 非 PACK_RECORD 时恒 1（"row not deleted"）
  │     bit1..: 各可空列的 NULL 位（+ BIT 列的余位）
  └─► 列数据区（按 dd::Column 顺序线性排布，各自 pack_length_in_rec()）

  Field::ptr            = record[0] + 本列起始偏移
  Field::offset(record) = ptr - record              ← 引擎层唯一关心的量
  Field::m_null_ptr     = record[0] + null_offset
  Field::null_bit       = 该字节内的位掩码
  share->reclength      = null_bytes + Σ pack_length(所有列)
```

对应源码（全是**指针减法**）：

```cpp
// [F1] sql/field.h
  uint offset(uchar *record) const { return (uint)(ptr - record); }
  uint null_offset(const uchar *record) const { return (uint)(m_null_ptr - record); }
  bool is_nullable() const { return m_null_ptr != nullptr; }
  void set_null_ptr(uchar *p_null_ptr, uint p_null_bit) {
    m_null_ptr = p_null_ptr; null_bit = p_null_bit;
  }
// [F2] sql/field.cc
uint Field::null_offset() const { return null_offset(table->record[0]); }
```

`reclength` / `null_bytes` 由 `find_record_length()`（`[F15]`）预先算好：

```cpp
// [F15] sql/default_values.cc
static bool find_record_length(const dd::Table &table, size_t min_length, TABLE_SHARE *share) {
  share->fields = 0; share->null_fields = 0; share->reclength = 0;
  ulong leftover_bits = pack_record ? 0 : 1;
  for (const dd::Column *col_obj : table.columns()) {
    if (col_obj->is_se_hidden()) continue;          // ★ SE 隐藏列完全不进 server 布局
    if (col_obj->is_nullable()) share->null_fields++;
    if (col_obj->type() == dd::enum_column_types::BIT) { /* 余位计入 leftover_bits */ }
    share->reclength += column_pack_length(*col_obj);
    share->fields++;
  }
  share->null_bytes = (share->null_fields + leftover_bits + 7) / 8;
  share->last_null_bit_pos = (share->null_fields + leftover_bits) & 7;
  share->reclength += share->null_bytes;
  share->reclength = std::max<size_t>(min_length, share->reclength);   // 被引擎下限顶上去
  share->stored_rec_length = share->reclength;
  return false;
}
```

★ `share->default_values` 是"模板行"——既是**列偏移计算的基准**（`move_offset` 的减数），也是**每列默认值的存放处**；二者合一不是巧合。

**`read_set` / `write_set` 位图**：**第 i 位 = `table->field[i]` = `field->field_index()`**。设置者是 `TABLE::mark_column_used()`（`[F13]`）：

```cpp
void TABLE::mark_column_used(Field *field, enum enum_mark_columns mark) {
  switch (mark) {
    case MARK_COLUMNS_READ:
      bitmap_set_bit(read_set, field->field_index());
      bitmap_set_bit(&read_set_internal, field->field_index());
      ...
      if (field->is_virtual_gcol()) mark_gcol_in_maps(field);
      break;
    case MARK_COLUMNS_WRITE:
      bitmap_set_bit(write_set, field->field_index());
      if (field->is_gcol()) mark_gcol_in_maps(field);
      break;
    ...
  }
}
```

`m_field_index` 在 `fill_column_from_dd()` 里由 `reg_field->set_field_index(field_nr)` 写入，**恒有 `table->field[i]->field_index() == i`**；`Field::init()` 并不设置它，所以"克隆 → init → move_field_offset"三步不会破坏列号。

#### 5.3 与 handler 的接缝

**★ handler 只取 Field 的"布局"，不碰"语义"**——引擎侧拿 Field 信息的入口是一个极小的适配器：

```cpp
// [F29] ha_innodb.cc
/* The following accessor functions should really be inside MySQL code! */
static inline uint get_field_offset(const TABLE *table, const Field *field) {
  return field->offset(table->record[0]);
}
```

整个函数体就是一个指针减法。随后 InnoDB 把它固化进 `mysql_row_templ_t`（`[F30]`）——**从 Field 借来的只有 4 个数**：`mysql_col_offset`（偏移）、`mysql_col_len`（长度）、`mysql_null_byte_offset` + `mysql_null_bit_mask`（NULL 位），外加 `mysql_type`。**没有函数指针、没有 `val_*()`、没有 `store*()`**：

```cpp
// [F29] ha_innodb.cc（build_template_field，节选）
  templ->mysql_col_offset = (ulint)get_field_offset(table, field);   // = Field::offset()
  templ->mysql_col_len    = (ulint)field->pack_length();
  if (field->is_nullable()) {
    templ->mysql_null_byte_offset = field->null_offset();
    templ->mysql_null_bit_mask    = (ulint)field->null_bit;
  } else {
    templ->mysql_null_bit_mask = 0;
  }
  templ->mysql_type = (ulint)field->type();
  ...
```

对 `Field` 的调用清单只有：`innobase_is_v_fld()`、`is_nullable()`、`null_offset()`、`null_bit`、`pack_length()`、`type()`、`key_length()`、`get_length_bytes()`，**全是"布局/形状"查询，没有一次 `val_*()` / `store*()`**——这就是"handler 层不依赖 Field 语义"的直接证据。

模板建好之后，回写 MySQL 行就是纯 `memcpy` + 位操作（`[F31]`）：

```cpp
    row_sel_field_store_in_mysql_format(mysql_rec + templ->mysql_col_offset,
                                        templ, rec_index, field_no, data, len, sec_field_no);
  ...
  if (templ->mysql_null_bit_mask) {
    mysql_rec[templ->mysql_null_byte_offset] &= ~(byte)templ->mysql_null_bit_mask;
  }
```

`mysql_rec + templ->mysql_col_offset` 与 `Field::ptr` 完全等价；清 NULL 位与 `Field::set_notnull()` 做的事一模一样，只是引擎自己用位运算重做了一遍——**因为它不能（也不应该）调用 Field 的方法**。

哪些列进模板由 server 的位图决定（`build_template_needs_field`，`[F29]`）：`bitmap_is_set(table->read_set, i) || bitmap_is_set(table->write_set, i)`——**下标 i 直接作位号**，形成完整闭环：`Item_field` 引用某列 → `mark_column_used()` 点亮 `read_set` → 引擎只为这些列建模板 → 只回传这些列的字节。

★ 另一个边界事实：`handler` 里**没有** `field` 成员（全 `handler.h` 里 `Field` 只有前置声明 + 一个虚函数签名 + 注释）。`handler` 持有的是 `TABLE *table`；接口签名是 `write_row(uchar *buf)`——**参数是裸字节缓冲，不是 Field 数组**。

#### 5.4 与 DD（Data Dictionary）的接缝

**`dd::Column` → `Field`**：`make_field(const dd::Column&, ...)`（`[F14]`）是"DD 类型系统 → server 类型系统"的翻译层。注意 `ptr` / `null_pos` / `null_bit` 是**由调用方算好的布局参数**传进来的——`make_field()` 只造对象、不排布；`interval`（ENUM/SET 取值表）、`geom_type`、`treat_bit_as_char`、`srs_id` 等藏在 `dd::Properties options()` 里的属性也在这里还原。

外层 `fill_column_from_dd()` 负责列号、生成列、默认值、隐藏属性：

```cpp
// [F14] sql/dd_table_share.cc（节选）
  reg_field = make_field(*col_obj, charset, share, rec_pos, null_pos, null_bit_pos);
  reg_field->set_field_index(field_nr);          // ★ m_field_index 的唯一写入点
  reg_field->stored_in_db = true;
  if (!col_obj->is_generation_expression_null()) { /* 生成列：只存表达式"文本"，Item 到 TABLE 层才建 */ }
  if (auto_flags & Field::GENERATED_FROM_EXPRESSION) { reg_field->m_default_val_expr = ...; }
  if ((auto_flags & Field::NEXT_NUMBER) != 0) share->found_next_number_field = &share->field[field_nr];
  ...
  reg_field->set_hidden(col_obj->hidden());      // ★ 隐藏属性的唯一入口
  share->field[field_nr] = reg_field;
```

`fill_columns_from_dd()` 是**偏移分配器**：`rec_pos += pack_length_in_rec()`、`null_bit_pos += column_preamble_bits()`（>7 进位）；虚拟生成列**第二遍**扫描（保证排在记录末尾）。三个 `assert` 保证与 `find_record_length()` 的预算一致。

**★ 隐藏列 `dd::Column::enum_hidden_type` → `Field::m_hidden`**：

```cpp
// [F10] sql/dd/types/column.h
  enum class enum_hidden_type {
    HT_VISIBLE = 1,       /// 普通列
    HT_HIDDEN_SE = 2,     /// 对 server 完全不可见（引擎内部列）
    HT_HIDDEN_SQL = 3,    /// server 可见、用户隐藏——用于函数索引
    HT_HIDDEN_USER = 4    /// 用户级 INVISIBLE
  };
```

```cpp
// [F1] sql/field.h
  bool is_hidden() const { return hidden() != dd::Column::enum_hidden_type::HT_VISIBLE && ...; }
  bool is_hidden_by_system() const { return (hidden() == HT_HIDDEN_SE || hidden() == HT_HIDDEN_SQL) && ...; }
  bool is_hidden_by_user() const { return hidden() == HT_HIDDEN_USER; }
  bool is_field_for_functional_index() const {
    return hidden() == HT_HIDDEN_SQL && gcol_info != nullptr;   // ★ 函数索引承载列
  }
```

★ 三点：① **HT_HIDDEN_SE 不产生 Field**（`fill_columns_from_dd()` 里 `is_se_hidden()` 直接 continue，`reclength` 也不含它）；② **HT_HIDDEN_SQL 是函数索引的实现手段**（`CREATE INDEX idx ON t((a+b))` 真正建一个隐藏生成列，索引建在它上面；`is_field_for_functional_index()` 额外要求有 `gcol_info`）；③ **HT_HIDDEN_USER 是用户级 INVISIBLE**，进入内部临时表时被"洗白"为 HT_VISIBLE。

**默认值**：DD 侧有 `default_value()`（内部二进制镜像）、`default_value_utf8()`（人类可读文本）、`default_option()`（表达式文本 / CURRENT_TIMESTAMP）。

```cpp
// [F14] sql/dd_table_share.cc（节选）
  if (col_obj->is_default_value_null()) {
    reg_field->reset(); reg_field->set_null();
  } else if (field_type == MYSQL_TYPE_BIT && !treat_bit_as_char && (col_obj->char_length() & 7)) {
    /* BIT 余位单独写进 preamble，主体写进 rec_pos */
    set_rec_bits(leftover_bits, bitfield->bit_ptr, bitfield->bit_ofs, bitfield->bit_len);
    memcpy(rec_pos, col_obj->default_value().data(), reg_field->pack_length() - 1);
  } else {
    memcpy(rec_pos, col_obj->default_value().data(), reg_field->pack_length());   // ★ 零解析灌入
  }
```

★ **DD 里存的就是"内存行格式的字节镜像"**，所以默认值可以零成本灌进 `share->default_values`（也解释了为什么"模板 Field 读起来像读到了一个有默认值的行"）。反方向（`Field` → `dd::Column`）则把默认值**渲染成字符串**存进 `default_value_utf8()`（给人看：`SHOW CREATE TABLE`、I_S）——正向字节镜像、反向字符串渲染的不对称是刻意的。表达式型默认值走 `default_option()` → `Field::m_default_val_expr` → **到 TABLE 层才解析成 Item**。

运行期终点：`empty_record()` 把 `share->default_values` `memcpy` 进 `record[0]` 并把 NULL 字节全置 1。

#### 5.5 小结：四条接缝的"契约"

| 邻域 | Field 提供的 | 邻域反过来给的 | 关键函数 |
|---|---|---|---|
| **Item** | `val_*()` 语义取值、`store*()` 语义写入、`result_type()`/`charset()`/`decimals()` | `Field*` 目标、`result_field`、由 Item 造出的新 Field | `Item_field::set_field()`、`Item::save_in_field_inner()`、`create_tmp_field()` |
| **TABLE / SHARE** | `ptr`/`m_null_ptr`/`null_bit`/`m_field_index`/`pack_length()` | `record[0]`、`null_flags`、`read_set`/`write_set`、`table` 归属 | `open_table_from_share()`（clone+init+move_field_offset）、`mark_column_used()` |
| **handler** | **仅布局**：`offset()`/`pack_length()`/`null_offset()`/`null_bit`/`type()` | 行缓冲 `uchar *buf`、行长下限 | `get_field_offset()`、`build_template_field()`、`build_template_needs_field()` |
| **DD** | ——（Field 是消费者） | 列定义（类型/长度/可空/隐藏/默认值/生成表达式）、`reclength`/`null_bytes` 预算 | `make_field()`、`fill_column_from_dd()`、`open_table_def()` |

一句话：**DD 决定 Field 长什么样，TABLE 决定 Field 站在哪，Item 决定 Field 里装什么，handler 只关心 Field 在哪儿、多长。**

### 六、特殊子类：JSON / ENUM-SET / BIT / 时态类型

> 补充源码坐标：`[F32]` include/my_time.h、`[F33]` mysys/my_time.cc、`[F34]` include/my_compare.h、`[F35]` include/typelib.h、`[F36]` sql-common/json_binary.h、`[F37]` sql-common/json_dom.cc、`[F38]` sql/json_diff.h、`[F39]` libbinlogevents/src/binary_log_funcs.cpp、`[F40]` sql/sql_const.h、`[F41]` sql/strfunc.h、`[F42]` sql/sql_time.h、`[F43]` sql/item_json_func.cc（其余沿用：`[F1]`/`[F2]`/`[F7]`/`[F5]`/`[F13]`）。
>
> **归属声明**：JSON 的部分更新真正下推到引擎（InnoDB 的 `m_partial_update` / binary-diff 下推 / `mach_write_to_*` 写盘宏）**属 InnoDB 文档**，本节只讲 server 侧 `Field` 接口层。

#### 6.1 `Field_json`：为什么挂在 `Field_blob` 下

```cpp
// [F1] sql/field.h
class Field_json : public Field_blob {
 public:
  Field_json(uchar *ptr_arg, uchar *null_ptr_arg, uint null_bit_arg,
             uchar auto_flags_arg, const char *field_name_arg,
             TABLE_SHARE *share, uint blob_pack_length)
      : Field_blob(ptr_arg, null_ptr_arg, null_bit_arg, auto_flags_arg,
                   field_name_arg, share, blob_pack_length, &my_charset_bin) {}
  enum_field_types type() const override { return MYSQL_TYPE_JSON; }
  /* 返回文本字符集，使字符串函数自动把它当非二进制串处理 */
  const CHARSET_INFO *charset() const override { return &my_charset_utf8mb4_bin; }
  /* Sort should treat the field as binary and not attempt any conversions. */
  const CHARSET_INFO *sort_charset() const final { return field_charset; }
  /* JSON columns don't have an associated charset —— 让 SHOW CREATE TABLE 不加 CHARACTER SET */
  bool has_charset() const final { return false; }
```

★ 三个要点：① **JSON 值长度完全可变**，直接复用 `Field_blob` 的"定长前缀槽 + 外部数据指针"模型，行内只放 `[长度][指针]`；② **传给基类的是 `my_charset_bin`，但 `charset()` 覆写成 `my_charset_utf8mb4_bin`**——这个"二元性"是刻意的：字符串函数/比较走 `charset()`（当文本），排序走 `sort_charset()`（binary，不做任何转换）；③ `has_charset() == false`。

行内槽位：`calc_pack_length(MYSQL_TYPE_JSON, ...)` 恒为 `4 + portable_sizeof_char_ptr`，所以 JSON 的 `packlength` 永远 **4**（与 LONGBLOB/GEOMETRY 一致，可表达最大 4GB）。

**行内布局与 JSONB**（`get_binary()` 是拿到 JSONB 裸字节的入口）：

```cpp
// [F2] sql/field.cc
const char *Field_json::get_binary(ptrdiff_t row_offset) const {
  return pointer_cast<char *>(get_blob_data(ptr + packlength + row_offset));
}
```

```
  record[0]                                      BLOB 存储
  ┌────────── ptr ─────────────────────┐          ┌──────────────────────────┐
  │ 字节 0..3   : packlength=4，长度(小端)│          │ JSONB                    │
  │ 字节 4..11  : 8 字节 char* 指针 ──────┼─────────►│ byte0: Value::enum_type  │
  └─────────────────────────────────────┘          │ ...   计数/偏移表/键值表  │
                                                   └──────────────────────────┘
```

**★ `val_json()` 是唯一的读出口**（注意：8.0.39 里**没有** `get_json()`，只有 `val_json()`）：

```cpp
// [F2] sql/field.cc
bool Field_json::val_json(Json_wrapper *wr) const {
  String tmp;
  String *s = Field_blob::val_str(&tmp, &tmp);      // 借基类公开接口拿到 (ptr,len)
  json_binary::Value v(json_binary::parse_binary(s->ptr(), s->length()));
  if (v.type() == json_binary::Value::ERROR) {
    my_error(ER_INVALID_JSON_BINARY_DATA, MYF(0));  // ★ 读取侧的合法性兜底
    return true;
  }
  *wr = Json_wrapper(v);
  return false;
}
longlong Field_json::val_int() const {
  Json_wrapper wr;
  if (val_json(&wr)) return 0;
  return wr.coerce_int(field_name);
}
```

★ `parse_binary()` 是**惰性**的：`Json_wrapper(v)` 只持有 `Value` 视图，不展开成 DOM——所以 `SELECT j->>'$.a'` 不需要反序列化整个文档。

**★ 合法性检查在哪做**：**在 `store()` / `store_json()` 里，不在 `validate()` 里**（`Field_json` 没有覆写 `validate_stored_val()`）：

```cpp
// [F2] sql/field.cc
type_conversion_status Field_json::store(const char *from, size_t length,
                                         const CHARSET_INFO *cs) {
  reset();                                    // ★ 先清干净，避免错误路径留下垃圾
  const char *s; size_t ss; String v(from, length, cs);
  if (ensure_utf8mb4(v, &value, &s, &ss, true)) return TYPE_ERR_BAD_VALUE;   // 字符集门槛
  std::unique_ptr<Json_dom> dom(Json_dom::parse(s, ss, [&](...) { my_error(ER_INVALID_JSON_TEXT, ...); },
                                                JsonDocumentDefaultDepthHandler));
  if (dom.get() == nullptr) return TYPE_ERR_BAD_VALUE;
  if (json_binary::serialize(current_thd, dom.get(), &value)) return TYPE_ERR_BAD_VALUE;
  return store_binary(value.ptr(), value.length());
}

type_conversion_status Field_json::store_binary(const char *data, size_t length) {
  assert(json_binary::parse_binary(data, length).is_valid());   // ★ 兜底断言
  if (length > UINT_MAX32) { my_error(ER_JSON_VALUE_TOO_BIG, MYF(0)); return TYPE_ERR_BAD_VALUE; }
  return Field_blob::store(data, length, field_charset);
}
```

非 JSON 源（int/double/decimal/time）一律拒绝，错误文案统一走 `unsupported_conversion()`（`ER_INVALID_JSON_TEXT`：*"not a JSON text, may need CAST"*）。

**部分更新（server 侧接口）**：`Field_json` 暴露 `pack_diff()` / `get_diff_vector_and_length()` / `is_before_image_equal_to_after_image()` / `unpack_diff()`，全部围绕 `Json_diff_vector`。决策链：

```cpp
// [F2] sql/field.cc（节选）
longlong Field_json::get_diff_vector_and_length(ulonglong value_options,
                                                const Json_diff_vector **diff_vector_p) const {
  if (is_null()) { ...; return 0; }
  longlong length_of_full_format = get_length();
  if ((value_options & PARTIAL_JSON_UPDATES) == 0) {          // ① 开关未开 → 全量
    length = length_of_full_format;
  } else {
    diff_vector = table->get_logical_diffs(this);             // ② 优化器算出的 logical diff
    if (diff_vector == nullptr) {
      if (is_before_image_equal_to_after_image()) { diff_vector = &EMPTY_JSON_DIFF_VECTOR; ... }
      else length = length_of_full_format;
    } else {
      ...
      else if (length_of_diff_vector < length_of_full_format) length = length_of_diff_vector;  // ③ diff 更短才用
      else { diff_vector = nullptr; length = length_of_full_format; }
    }
  }
  ...
}
```

★ 三个门槛：① `@@binlog_row_value_options` 是否含 `PARTIAL_JSON`；② 优化器有没有算出 logical diff；③ **diff 序列化后是否真的比全量短**（改一个 10 字节字段才划算）。
★ **logical diff vs binary diff**：`add_logical_diff()` 记的是"在 path 上做 replace/insert/remove"这种**语义操作**（`JSON_SET`/`JSON_REPLACE` 求值时直接调）；`is_binary_diff_enabled()` 判断的是"能否在原 JSONB 字节流上原地打补丁"——**后者主要由引擎决定，属 InnoDB 文档**。

**比较与排序（两条路）**：
- **语义比较** `cmp_binary()`：先 `parse_binary()` 成 `Json_wrapper` 再 `compare()`（类型序由 `type_comparison[][]` 决定，数组逐元素比）。
- **排序键** `make_sort_key()`：生成带 `JSON_KEY_*` 前缀的可 memcmp 字节串——`JSON_KEY_NULL=0x00` < `NUMBER_NEG=0x01` < `NUMBER_ZERO=0x02` < `NUMBER_POS=0x03` < `STRING=0x04` < `OBJECT=0x05` < `ARRAY=0x06` < `FALSE=0x07` < `TRUE=0x08` < `DATE` < `TIME` < `DATETIME` < `OPAQUE`。
- ⚠️ **易踩的坑**：`Field_json` **没有覆写 `cmp(const uchar*, const uchar*)`**，继承的是 `Field_blob::cmp()` → `strnncollsp` with `my_charset_bin` = **按 JSONB 字节流做二进制比较**，不等价于 JSON 语义序。真正需要语义序的地方走 `make_sort_key()` / `cmp_binary()`。

#### 6.2 `Field_enum` / `Field_set`：物理是整数、语义是字符串

**★ `type()` 与 `real_type()` 的分裂**（这是 ENUM/SET 最反直觉的地方）：

```cpp
// [F1] sql/field.h
class Field_enum : public Field_str {
 protected:
  uint packlength;
 public:
  TYPELIB *typelib;
  enum_field_types type() const final { return MYSQL_TYPE_STRING; }   // ★ 逻辑：字符串！
  enum_field_types real_type() const override { return MYSQL_TYPE_ENUM; }  // ★ 物理：ENUM
  enum Item_result cmp_type() const final { return INT_RESULT; }
  enum Item_result cast_to_int_type() const final { return INT_RESULT; }
  bool optimize_range(uint, uint) const final { return false; }
  bool zero_pack() const override { return false; }              // ENUM: 0 非法
  /* enum and set are sorted as integers */
  const CHARSET_INFO *sort_charset() const final { return &my_charset_bin; }
};
```

- `type()` 回答"**表达式/协议层面表现得像什么**"→ 字符串（决定 `result_type()`、Item 类型归并、协议发送）。
- `real_type()` 回答"**磁盘/元数据里到底是什么**"→ ENUM/SET（DD、binlog、`pack_length_from_metadata()` 需要区分）。
- ★ **`sort_charset() == &my_charset_bin`**，注释直言 "enum and set are sorted as integers"——**`ORDER BY enum_col` 按元素下标排序，不是按名字字母序**。这是 ENUM 最广为人知的"反直觉"行为，源码就在这两行里。
- `zero_pack()`：ENUM 为 false（下标 0 = 非法/空串，不能被当 0 压缩），SET 为 true（0 = 空集，合法）。

**`TYPELIB` 与 `pack_length()`**：

```cpp
// [F35] include/typelib.h
struct TYPELIB {
  size_t count{0};              /* How many types */
  const char *name{nullptr};
  const char **type_names{nullptr};      /* 元素名数组 */
  unsigned int *type_lengths{nullptr};   /* 元素名长度数组（type_names 不保证 NUL 结尾语义） */
};
```

```cpp
// [F1] sql/field.h
inline uint get_enum_pack_length(int elements) { return elements < 256 ? 1 : 2; }
inline uint get_set_pack_length(int elements) {
  uint len = (elements + 7) / 8;
  return len > 4 ? 8 : len;      // ★ 合法结果只有 {1,2,3,4,8}，不存在 5/6/7
}
```

★ ENUM 存 **1-based 下标**（0 保留给"非法/空串"），最多 65535 个元素；注意判据是 `< 256`（255 个元素仍用 1 字节）。★ SET 存**位图**，`(elements+7)/8` 字节；但小于 4 字节时向上取到 8（因为 SET 最多 64 元素、`val_int()` 走 `ulonglong`）。

**双向映射**：

```cpp
// [F2] sql/field.cc
void Field_enum::store_type(ulonglong value) {        // ★ 唯一的物理写点
  switch (packlength) {
    case 1: ptr[0] = (uchar)value; break;
    case 2: if (table->s->db_low_byte_first) int2store(ptr, (unsigned short)value);
            else shortstore(ptr, (unsigned short)value); break;
    ...
  }
}

// 按名字存
type_conversion_status Field_enum::store(const char *from, size_t length, const CHARSET_INFO *cs) {
  ...
  uint tmp = find_type2(typelib, from, length, field_charset);     // [F41] 查名字
  if (!tmp) {
    if (length < 6) {   // Can't be more than 99999 enums —— 专为 LOAD DATA INFILE 的数字
      tmp = (uint)my_strntoul(cs, from, length, 10, &end, &err);
      if (err || end != from + length || tmp > typelib->count) { tmp = 0; ... ret = TYPE_WARN_TRUNCATED; }
    } else set_warning(Sql_condition::SL_WARNING, WARN_DATA_TRUNCATED, 1);
  }
  store_type((ulonglong)tmp);
  return ret;
}

// 读下标 / 读名字
static longlong enum_val_int(const uchar *ptr, uint packlength, bool low_byte_first) { ... }
longlong Field_enum::val_int() const { return enum_val_int(ptr, packlength, table->s->db_low_byte_first); }
String *Field_enum::val_str(String *, String *val_ptr) const {
  uint tmp = (uint)Field_enum::val_int();
  if (!tmp || tmp > typelib->count) val_ptr->set("", 0, field_charset);
  else val_ptr->set(typelib->type_names[tmp - 1], typelib->type_lengths[tmp - 1], field_charset);
  return val_ptr;
}
```

★ `val_int()` 返回 1-based 下标（0 = 非法/空），`val_str()` 返回元素名——这就是 `SELECT enum_col+0` 与 `SELECT enum_col` 结果"对不上"的根源。

**`Field_set` 的位图语义**：物理值是 `ulonglong` 位图（第 i 位 = 第 i 个元素是否选中），所以 `WHERE set_col = 3` 等价于"选中第 0、1 两个元素"。

```cpp
// [F2] sql/field.cc
type_conversion_status Field_set::store(longlong nr, bool) {
  ulonglong max_nr = (sizeof(ulonglong) * 8 <= typelib->count)
                       ? ULLONG_MAX : (1ULL << typelib->count) - 1;
  if ((ulonglong)nr > max_nr) { nr &= max_nr; ...; error = TYPE_WARN_TRUNCATED; }  // ★ 掩码截断，不报错
  store_type((ulonglong)nr); return error;
}

String *Field_set::val_str(String *val_buffer, String *) const {
  ulonglong tmp = (ulonglong)Field_enum::val_int();
  ...
  while (tmp && bitnr < typelib->count) {          // ★ 从低位往高位扫，用 ',' 拼接
    if (tmp & 1) {
      if (val_buffer->length()) val_buffer->append(&field_separator, 1, &my_charset_latin1);
      String str(typelib->type_names[bitnr], typelib->type_lengths[bitnr], field_charset);
      val_buffer->append(str);
    }
    tmp >>= 1; bitnr++;
  }
  return val_buffer;
}
```

★ 输出顺序 = **定义顺序**（typelib 顺序），不是存储顺序——这正是"在末尾追加 SET/ENUM 成员不会打乱已有数据"的原因，也是 `Field_enum::is_equal()` 允许"末尾追加成员做 inplace ALTER"的依据（只允许 `new_typelib->count >= typelib->count`）。

#### 6.3 `Field_bit`：零头位借住 NULL 字节区

先读 `field.h` 顶部那段**必须读的注释**（它解释了 `cmp_binary()` 为什么要求调用方把 NULL 字节一起拷过来）：

```cpp
// [F1] sql/field.h
/*
  Note:
    To use Field_bit::cmp_binary() you need to copy the bits stored in
    the beginning of the record (the NULL bytes) to each memory you
    want to compare (where the arguments point).
    This is the reason:
    - Field_bit::cmp_binary() is only implemented in the base class (Field::cmp_binary()).
    - Field::cmp_binary() currently uses pack_length() to calculate how long the data is.
    - pack_length() includes size of the bits stored in the NULL bytes of the record.
*/
class Field_bit : public Field {
 public:
  uchar *bit_ptr;  // position in record where 'uneven' bits store
  uchar bit_ofs;   // offset to 'uneven' high bits
  uint bit_len;    // number of 'uneven' high bits
  uint bytes_in_rec;
```

构造与 `bit_ptr` 的计算（`make_field()` 直接复用 NULL 位区里尚未使用的位）：

```cpp
// [F2] sql/field.cc
Field_bit::Field_bit(...)
    : Field(...), bit_ptr(bit_ptr_arg), bit_ofs(bit_ofs_arg),
      bit_len(len_arg & 7), bytes_in_rec(len_arg / 8) {
  if (!null_ptr_arg) null_bit = bit_ofs_arg;   // 让 Field::eq() 能区分两个不同的 BIT 字段
}

// make_field() 里：
  if (field_type == MYSQL_TYPE_BIT && !treat_bit_as_char) {
    bit_ptr = null_pos;
    bit_offset = null_bit;
    if (is_nullable) { bit_ptr += (null_bit == 7); bit_offset = (bit_offset + 1) & 7; }
  }
```

```
  BIT(11) NOT NULL，占用 NULL 位区的 null_bit = 3

  行首 NULL 位区                               数据区
  +---------+---------+                       +------------------+
  |  byte0  |  byte1  |  ...                  | ptr[0]           |
  +---------+---------+                       +------------------+
       ^ bit_ptr
  bit_ofs = 4（null_bit=3 被自己占了，NOT NULL 时直接用 null_bit 起始位）
  bit_len = 11 & 7 = 3
  bytes_in_rec = 11 / 8 = 1

  byte0:  b7  b6  b5 [b4 b3 b2] b1  b0
                       ^^^^^^^^
                       这 3 位存 BIT(11) 的最高 3 位 (bit10..bit8)
  ptr[0]: 存 BIT(11) 的低 8 位

  val_int() 组装：
      bits  = get_rec_bits(bit_ptr, bit_ofs, bit_len)   // 取高 3 位
      bits <<= (bytes_in_rec * 8)                        // 左移 8
      return bits | ptr[0]                               // 拼上低 8 位
```

★ **为什么这么设计**：`n % 8` 的零头位若单独占新字节就浪费 7 位；而 NULL 位区**本来就按位分配、几乎总有剩余**。更重要的是这样 `BIT(n)` 的**全部有效位物理上仍连续**（NULL 区的零头位在前、`ptr[0..]` 的整字节在后），可构造出"大端、可 memcmp"的字节序列。

零头位的读写宏（`[F34]`）——注意 `get_rec_bits` 读 **2 个字节**（零头位可能跨字节）：

```cpp
#define get_rec_bits(bit_ptr, bit_ofs, bit_len)                          \
  (((((uint16)(bit_ptr)[1] << 8) | (uint16)(bit_ptr)[0]) >> (bit_ofs)) & \
   ((1 << (bit_len)) - 1))
#define set_rec_bits(bits, bit_ptr, bit_ofs, bit_len)  { ... }   // 跨字节时写 ptr[1]
#define clr_rec_bits(bit_ptr, bit_ofs, bit_len) set_rec_bits(0, bit_ptr, bit_ofs, bit_len)
```

**★ 三个长度各不相同**：

```cpp
  uint32 pack_length() const final { return (uint32)(field_length + 7) / 8; }  // 逻辑字节数（向上取整）
  uint32 pack_length_in_rec() const final { return bytes_in_rec; }              // = field_length/8（向下取整）
  uint row_pack_length() const final { return bytes_in_rec + ((bit_len > 0) ? 1 : 0); }
```

在 `BIT(8)` 上三者相同（1/1/1），在 `BIT(9)` 上全不同（2/1/2）。

**★ 为什么 `cmp` 只能手工拼零头位**：`ptr` 指向的整字节与 `bit_ptr` 指向的零头位**在内存中不相邻**（一个在数据区、一个在行首 NULL 区），所以不能简单 `memcmp(ptr, ...)`——所有比较入口都要**先比零头位、再 memcmp 整字节**：

```cpp
int Field_bit::key_cmp(const uchar *str, uint length) const {
  if (bit_len) {
    uchar bits = get_rec_bits(bit_ptr, bit_ofs, bit_len);
    int flag = (int)(bits - *str);
    if (flag) return flag;
    str++; length--;
  }
  return memcmp(ptr, str, length);
}
int Field_bit::cmp_offset(ptrdiff_t row_offset) const {
  if (bit_len) { /* 先比 get_rec_bits(bit_ptr) vs get_rec_bits(bit_ptr+row_offset) */ }
  return memcmp(ptr, ptr + row_offset, bytes_in_rec);
}
```

而 `get_key_image()` 把零头位**物化成第一个字节**，产出连续字节序列：

```cpp
size_t Field_bit::get_key_image(uchar *buff, size_t length, imagetype) const {
  if (bit_len) { uchar bits = get_rec_bits(bit_ptr, bit_ofs, bit_len); *buff++ = bits; length--; }
  size_t data_length = min(length, static_cast<size_t>(bytes_in_rec));
  memcpy(buff, ptr, data_length);
  return data_length + 1;
}
```

`pack()` 里那段注释把"如何在另一条记录里定位对应的零头位字节"推导得很清楚（`ptr - bit_ptr = from - from_bitp` ⇒ `from_bitp = bit_ptr + from - ptr`）。`new_key_field()` 则**放弃零头位机制**（`res->bit_ptr = new_ptr; res->bit_ofs = 0; if (bit_len) res->ptr++;`）。另外 `reset()` 必须清两处、`set_default()` 要把零头位从 default row 搬过来、`move_field_offset()` 必须同步移动 `bit_ptr`。

**`Field_bit_as_char`**：核心是三件事——`bit_ptr = nullptr`、`bit_len = 0`、`bytes_in_rec = (len+7)/8`，即**零头位不再借住 NULL 区，而是和数据挤在同一整字节里**（高位补零浪费），`key_type()` 也变成 `HA_KEYTYPE_BINARY`。这是给不支持 NULL 区借位的引擎用的兼容形态（注释明说 non-MyISAM），由 `treat_bit_as_char` 开关控制。

#### 6.4 时态类型族：新旧两代并存

**继承树**：

```
Field_temporal                        核心成员只有 uint8 dec（小数秒位数）
├── Field_temporal_with_date          DATE / DATETIME / TIMESTAMP
│   ├── Field_temporal_with_date_and_time      DATETIME / TIMESTAMP（无 fsp）
│   │   ├── Field_timestamp          旧 TIMESTAMP，4 字节 Unix 秒（"We will be removed eventually"）
│   │   └── Field_datetime           旧 DATETIME，8 字节 YYYYMMDDHHMMSS
│   ├── Field_temporal_with_date_and_timef     DATETIME(N) / TIMESTAMP(N)
│   │   ├── Field_timestampf          MYSQL_TYPE_TIMESTAMP2
│   │   └── Field_datetimef           MYSQL_TYPE_DATETIME2
│   └── Field_newdate                DATE，MYSQL_TYPE_NEWDATE，3 字节位域
└── Field_time_common                TIME / TIME(N)
    ├── Field_time                   旧 TIME，3 字节 HHMMSS（"It will be removed eventually"）
    └── Field_timef                  TIME(N)，MYSQL_TYPE_TIME2
```

**★ 新旧两代并存的原因**：1999 年的第一代把值压成"十进制整数"（DATETIME 8 字节 `YYYYMMDDHHMMSS`、TIMESTAMP 4 字节 Unix 秒、TIME 3 字节 `HHMMSS`），**根本没有小数秒的位置**。5.6 引入 `DATETIME(6)` 时不能改变既有表的数据布局（否则全表重写），于是新增三个"只用于磁盘/DD/binlog，不出现在客户端协议"的类型码：

```cpp
// [F7] include/field_types.h
  MYSQL_TYPE_TIMESTAMP2,
  MYSQL_TYPE_DATETIME2,   /**< Internal to MySQL. Not used in protocol */
  MYSQL_TYPE_TIME2,       /**< Internal to MySQL. Not used in protocol */
```

| 逻辑 `type()` | 物理 `real_type()` / `binlog_type()` | Field 类 |
|---|---|---|
| `MYSQL_TYPE_DATE` | `MYSQL_TYPE_NEWDATE` | `Field_newdate` |
| `MYSQL_TYPE_TIME` | `MYSQL_TYPE_TIME2` | `Field_timef` |
| `MYSQL_TYPE_DATETIME` | `MYSQL_TYPE_DATETIME2` | `Field_datetimef` |
| `MYSQL_TYPE_TIMESTAMP` | `MYSQL_TYPE_TIMESTAMP2` | `Field_timestampf` |

反向归一化靠 `real_type_to_type()`。客户端协议永远只看到 `MYSQL_TYPE_DATE/TIME/DATETIME/TIMESTAMP`——这就是 "Not used in protocol" 的含义。

**★ 物理打包格式（本节核心）**

**(1) `Field_newdate`（DATE）—— 3 字节位域**

```cpp
// [F33] mysys/my_time.cc
void my_date_to_binary(const MYSQL_TIME *ltime, uchar *ptr) {
  long tmp = ltime->day + ltime->month * 32 + ltime->year * 16 * 32;
  int3store(ptr, tmp);
}
// [F2] 读：
bool Field_newdate::get_date_internal(MYSQL_TIME *ltime) const {
  uint32 tmp = uint3korr(ptr);
  ltime->day   = tmp & 31;
  ltime->month = (tmp >> 5) & 15;
  ltime->year  = (tmp >> 9);
  ...
}
```

```
  bit:  23                     9 8      5 4     0
        +----------------------+--------+--------+
        |      year (15b)      |month(4)| day(5) |
        +----------------------+--------+--------+
```

**(2) `Field_time`（旧 TIME）—— 3 字节 HHMMSS 十进制**

```cpp
type_conversion_status Field_time::store_internal(const MYSQL_TIME *ltime, int *) {
  long tmp = ((ltime->month ? 0 : ltime->day * 24L) + ltime->hour) * 10000L +
             (ltime->minute * 100 + ltime->second);
  if (ltime->neg) tmp = -tmp;
  int3store(ptr, tmp);
  return TYPE_OK;
}
size_t Field_time::make_sort_key(uchar *to, size_t) const {
  assert(length == 3);
  to[0] = (uchar)(ptr[2] ^ 128);   // ★ 翻符号位：存储是有符号，memcmp 需要无符号序
  to[1] = ptr[1]; to[2] = ptr[0];
  return 3;
}
```

**(3) `Field_timef`（TIME(N)）—— `3 + (dec+1)/2` 字节 packed**（`[F33]`）

```cpp
longlong TIME_to_longlong_time_packed(const MYSQL_TIME &my_time) {
  long hms = (((my_time.month ? 0 : my_time.day * 24) + my_time.hour) << 12) |
             (my_time.minute << 6) | my_time.second;      // 22 bit
  longlong tmp = my_packed_time_make(hms, my_time.second_part);
  return my_time.neg ? -tmp : tmp;
}
/* On disk we convert from signed to unsigned using TIMEF_OFS,
   so all values become binary comparable. */
#define TIMEF_OFS 0x800000000000LL
#define TIMEF_INT_OFS 0x800000LL
void my_time_packed_to_binary(longlong nr, uchar *ptr, uint dec) {
  switch (dec) {
    case 0: default: mi_int3store(ptr, TIMEF_INT_OFS + my_packed_time_get_int_part(nr)); break;
    case 1: case 2: mi_int3store(...); ptr[3] = frac/10000; break;
    case 4: case 3: mi_int3store(...); mi_int2store(ptr+3, frac/100); break;
    case 5: case 6: mi_int6store(ptr, nr + TIMEF_OFS); break;
  }
}
unsigned int my_time_binary_length(unsigned int dec) { return 3 + (dec + 1) / 2; }   // [F39]
```

**(4) `Field_timestamp`（旧 TIMESTAMP）—— 4 字节 Unix 秒**

```cpp
void Field_timestamp::store_timestamp_internal(const my_timeval *tm) {
  if (table && table->s->db_low_byte_first) int4store(ptr, tm->m_tv_sec);
  else longstore(ptr, (uint32)tm->m_tv_sec);
}
bool Field_timestamp::get_date_internal_at(const Time_zone *tz, MYSQL_TIME *ltime) const {
  my_time_t temp = (table && table->s->db_low_byte_first) ? uint4korr(ptr) : ulongget(ptr);
  if (temp == 0) return true;                 // ★ 0 是哨兵（零时间戳）
  tz->gmt_sec_to_TIME(ltime, temp);           // ★ 读时才按会话时区转换
  return false;
}
```

★ 磁盘上只有一个 UTC 秒数——**"改 `time_zone` 后同一行读出来不同"是读侧转换的结果**。

**(5) `Field_timestampf`（TIMESTAMP(N)）—— `4 + (dec+1)/2` 字节**

```cpp
void my_timestamp_to_binary(const my_timeval *tm, uchar *ptr, uint dec) {
  mi_int4store(ptr, tm->m_tv_sec);
  switch (dec) {
    case 0: default: break;
    case 1: case 2: ptr[4] = tm->m_tv_usec / 10000; break;
    case 3: case 4: mi_int2store(ptr + 4, tm->m_tv_usec / 100); break;
    case 5: case 6: mi_int3store(ptr + 4, tm->m_tv_usec);
  }
}
unsigned int my_timestamp_binary_length(unsigned int dec) { return 4 + (dec + 1) / 2; }
```

**(6) `Field_datetime`（旧 DATETIME）—— 8 字节十进制整数**

```cpp
static inline type_conversion_status datetime_store_internal(TABLE *table, ulonglong tmp, uchar *ptr) {
  if (table && table->s->db_low_byte_first) int8store(ptr, tmp); else longlongstore(ptr, tmp);
  return TYPE_OK;
}
bool Field_datetime::get_date_internal(MYSQL_TIME *ltime) const {
  longlong tmp = datetime_get_internal(table, ptr);
  TIME_set_yymmdd(ltime, (uint)(tmp / 1000000LL));     // YYYYMMDD
  TIME_set_hhmmss(ltime, (uint)(tmp % 1000000LL));     // HHMMSS
  ltime->second_part = 0;
  return false;
}
```

**(7) `Field_datetimef`（DATETIME(N)）—— `5 + (dec+1)/2` 字节 packed**

```cpp
/* On disk we store as unsigned number with DATETIMEF_INT_OFS offset,
   for HA_KETYPE_BINARY compatibility purposes. */
#define DATETIMEF_INT_OFS 0x8000000000LL
void my_datetime_packed_to_binary(longlong nr, uchar *ptr, uint dec) {
  mi_int5store(ptr, my_packed_time_get_int_part(nr) + DATETIMEF_INT_OFS);
  switch (dec) { /* 同 TIMESTAMP(N) 的分档：ptr[5] / mi_int2store(ptr+5) / mi_int3store(ptr+5) */ }
}
unsigned int my_datetime_binary_length(unsigned int dec) { return 5 + (dec + 1) / 2; }
```

★ 因为新代打包时**已加偏置变成"无符号大端"**，所以排序/比较变得极简——直接 `memcpy` / `memcmp`：

```cpp
  size_t make_sort_key(uchar *to, size_t length) const final { memcpy(to, ptr, length); return length; }
  int cmp(const uchar *a_ptr, const uchar *b_ptr) const final { return memcmp(a_ptr, b_ptr, pack_length()); }
  const CHARSET_INFO *sort_charset() const final { return &my_charset_bin; }
```

**★ 精度如何编码进 `field_length`**：`Field_temporal` 构造器用 `field_length = 整数部分宽度 + (dec ? dec + 1 : 0)`（+1 是小数点）。判别式是 `length > MAX_*_WIDTH`：

```cpp
// [F2] calc_pack_length()
    case MYSQL_TYPE_DATETIME2:
      return length > MAX_DATETIME_WIDTH
                 ? my_datetime_binary_length(length - MAX_DATETIME_WIDTH - 1)   // ★ 反解 dec
                 : 5;
```

即 `DATETIME(3)` → `field_length = 19+3+1 = 23 > 19` → `my_datetime_binary_length(3) = 5+2 = 7` 字节。

**★ 零日期处理 `date_flags()`**——sql_mode 与类型语义的交汇点：

```cpp
// [F2] sql/field.cc
my_time_flags_t Field_timestamp::date_flags(const THD *thd) const {
  /* We don't want to store invalid or fuzzy datetime values in TIMESTAMP */
  my_time_flags_t date_flags = TIME_NO_ZERO_IN_DATE;      // ★ TIMESTAMP 无条件禁止"月/日为 0"
  if (thd->variables.sql_mode & MODE_NO_ZERO_DATE) date_flags |= TIME_NO_ZERO_DATE;
  ...
}
my_time_flags_t Field_newdate::date_flags(const THD *thd) const {
  my_time_flags_t date_flags = TIME_FUZZY_DATE;           // ★ DATE/DATETIME 允许宽松格式
  if (thd->variables.sql_mode & MODE_NO_ZERO_DATE) date_flags |= TIME_NO_ZERO_DATE;
  if (thd->variables.sql_mode & MODE_NO_ZERO_IN_DATE) date_flags |= TIME_NO_ZERO_IN_DATE;
  if (thd->variables.sql_mode & MODE_INVALID_DATES) date_flags |= TIME_INVALID_DATES;
  ...
}
```

★ **DATE/DATETIME 加 `TIME_FUZZY_DATE`（允许零日期/宽松格式），TIMESTAMP 不加、且无条件加 `TIME_NO_ZERO_IN_DATE`**——这就是"同一串 `'0000-00-00'` 在 DATE 列和 TIMESTAMP 列上行为不同"的实现根源。

warnings 位 → 告警的翻译由 `Field_temporal::set_warnings()` 完成（`[[nodiscard]]`，调用方不能忽略返回值）：`MYSQL_TIME_WARN_ZERO_DATE` 与 `ZERO_IN_DATE` 归到**同一个** `ER_WARN_DATA_OUT_OF_RANGE`；`MYSQL_TIME_NOTE_TRUNCATED` 降级为 `SL_NOTE`。★ "零日期 + 非严格模式"会降级成 `TYPE_NOTE_TIME_TRUNCATED` 且值被 `reset()` 清零。

**字符串解析全链路**：

```
store(str, len, cs)                                   Field_temporal
  └─ convert_str_to_TIME()                            虚函数
       ├─ str_to_datetime(..., date_flags(), ...)     DATE/DATETIME/TIMESTAMP
       └─ str_to_time(..., date_flags(), ...)         TIME/TIME(N)
  └─ [可选] MYSQL_TIMESTAMP_DATETIME_TZ 时区归位 + 边界检查
  └─ store_internal_adjust_frac()                     按 dec 舍入/截断小数秒
       └─ store_internal()                            虚函数 → 各类型打包
            ├─ my_date_to_binary()                    DATE          3B
            ├─ int3store(HHMMSS)                      TIME          3B
            ├─ my_time_packed_to_binary()             TIME(N)       3+(d+1)/2
            ├─ datetime_store_internal()              DATETIME      8B
            ├─ my_datetime_packed_to_binary()         DATETIME(N)   5+(d+1)/2
            ├─ int4store(tv_sec)                      TIMESTAMP     4B
            └─ my_timestamp_to_binary()               TIMESTAMP(N)  4+(d+1)/2
  └─ set_warnings()                                   warnings 位 → SQL 告警/错误
```

#### 6.5 小结：四组"反直觉"

1. **JSON**：没有自己的存储结构，行内就是 `Field_blob` 的 `[4 字节长度][8 字节指针]`；`val_json()` 是唯一读出口；合法性由 `store()`（`Json_dom::parse` + `serialize`）与 `store_json()` 强制（`store_binary()` 有 `assert(parse_binary(...).is_valid())` 兜底）；比较有两条路（`cmp_binary` 语义 / `make_sort_key` 前缀键），但继承来的 `cmp()` 实际是 **JSONB 字节流二进制比较**。
2. **ENUM/SET**：`type()` 报 `STRING`、`real_type()` 报 `ENUM`/`SET`；**`sort_charset()` 被强制为 binary ⇒ ENUM 按下标排序、不按名字**；SET 是 `ulonglong` 位图，越界位在 `store(longlong)` 里被 `nr &= max_nr` 掩码掉。
3. **BIT**：`n%8` 个零头位**借住行首 NULL 区**，导致 `pack_length()=(n+7)/8`、`pack_length_in_rec()=n/8`、`row_pack_length()=n/8+(bit_len>0)` 三者不同；所有比较都要"先比零头位、再 memcmp"，或先用 `get_key_image()` 物化。
4. **时态**：新旧两代并存（`real_type()` 报 `NEWDATE`/`TIME2`/`DATETIME2`/`TIMESTAMP2`，靠 `real_type_to_type()` 归一）；老代是"十进制整数"、新代是"packed + 偏置"（`TIMEF_INT_OFS`/`DATETIMEF_INT_OFS`），字节数 `3/4/5 + (dec+1)/2`；精度编码进 `field_length`，靠 `length > MAX_*_WIDTH` 判别、`length - MAX_*_WIDTH - 1` 反解。

### 七、默认值、instant 列与虚拟列

> 补充源码坐标：`[F47]` sql/sql_alter.h；其余沿用：`[F1]`/`[F2]`/`[F5]`/`[F6]`(handler.h+handler.cc)/`[F12]`(sql_table.cc)/`[F13]`/`[F14]`/`[F15]`(default_values.cc)/`[F30]`/`[F31]`。
>
> ★ **贯穿全章的分界线**：**server 层 = `Field`/`TABLE`/`TABLE_SHARE`/DD 公开接口；引擎层 = InnoDB 的 `dict_table_t`/`rec_offs`/`row_prebuilt_t`**。后者只点名、不展开（属 InnoDB / record 文档）。

#### 7.0 引子：三种"行里没有真正数据"的场景

| 场景 | 行缓冲里有字节吗 | 值从哪来 | server 层 `Field` 感知吗 |
|---|---|---|---|
| 默认值（INSERT 未给值） | 有 | `TABLE_SHARE::default_values` 或表达式求值 | 感知：`set_default()` |
| instant ADD COLUMN 的旧行 | 物理记录里**没有** | 引擎取行时用列默认值补齐 | **不感知**（下节给证据） |
| 虚拟生成列 | 有占位字节，但引擎不落盘 | 读行后 server 现算 | 感知：`is_virtual_gcol()` + `gcol_info` |

三块缓冲的 offset **完全一致**，这是 `default_values_offset()` 能工作的前提：

```
record[0]          ┌ null_bytes ─┬ c0 ─┬ c1 ─┬ … ─┬ vcol1 ─┬ vcol2 ─┐   ← 当前行
record[1]          ┌ null_bytes ─┬───── 同样的 offset 布局 ─────────┐   ← 旧行/比较行
s->default_values  ┌ null_bytes ─┬ 每列按 DD 的 default_value 填好的“模板行” ┐
   default_values_offset() = (ptrdiff_t)(s->default_values - record[0])   // [F5]
```

#### 7.1 默认值

**`Field::set_default()` / `copy_data()` / `reset()`**——基类把"取默认值"拆成两条路：

```cpp
// [F2] sql/field.cc
void Field::set_default() {
  if (has_insert_default_datetime_value_expression() ||
      has_insert_default_general_value_expression())
    evaluate_insert_default_function();              // 动态默认：现算
  else
    copy_data(table->default_values_offset());       // 常量默认：从模板行搬
}

void Field::copy_data(ptrdiff_t src_record_offset) {
  memcpy(ptr, ptr + src_record_offset, pack_length());   // 数据：整段 memcpy
  if (is_nullable()) {
    /* NULL 位：单独按位搬（不能整字节覆盖，会踩到同字节里别的列） */
    m_null_ptr[0] = (m_null_ptr[0] & ~null_bit) |
                    (m_null_ptr[src_record_offset] & null_bit);
  } else if (is_tmp_nullable())
    m_is_tmp_null = false;
}
```

★ `copy_data()` 是"整行 memcpy"的**逐列版本**：数据按 `pack_length()` 搬，NULL 位按位搬。`src_record_offset` 就是 `default_values_offset()`——因为三块缓冲布局一致，任何 Field 都能用"自己的 ptr + 偏移"在模板行里找到自己。

`reset()` 与之**语义完全不同**：

```cpp
// [F1] 基类：写 0 / 类型零值，不查 DD、不查 default_values
  virtual type_conversion_status reset() {
    memset(ptr, 0, pack_length());
    return TYPE_OK;
  }
// 覆写例（[F1]）：字符串列的"零值"是空格不是 0
  type_conversion_status Field_string::reset() final {
    charset()->cset->fill(charset(), (char *)ptr, field_length,
                          (has_charset() ? ' ' : 0));
    return TYPE_OK;
  }
```

**★ 只有 `Field_bit` 覆写了 `set_default()`**（8.0.39 全树检索 `::set_default()` 只有两个定义）：

```cpp
// [F2] sql/field.cc
void Field_bit::set_default() {
  if (bit_len > 0) {
    ptrdiff_t offset = table->default_values_offset();
    uchar bits = get_rec_bits(bit_ptr + offset, bit_ofs, bit_len);
    set_rec_bits(bits, bit_ptr, bit_ofs, bit_len);   // ★ 补搬 preamble 里的零头位
  }
  Field::set_default();
}
type_conversion_status Field_bit::reset() {
  memset(ptr, 0, bytes_in_rec);
  if (bit_ptr && (bit_len > 0)) clr_rec_bits(bit_ptr, bit_ofs, bit_len);
  return TYPE_OK;
}
```

★ **为什么偏偏是 BIT**：`BIT(n)` 被切成两段（`n/8` 整字节在记录体、`n%8` 零头位在 NULL 前缀），而 `copy_data()` 只 memcpy `pack_length()` 字节（**够不到 preamble**），所以必须单独补一趟。
★ **为什么 BLOB 不需要**：BLOB/JSON/GEOMETRY 不允许常量默认值（只许 `DEFAULT (expr)`），走 `evaluate_insert_default_function()`，压根不 memcpy。
★ **为什么 ENUM/SET 不需要**：DD 里存的 `default_value` **已经是 packed 后的下标字节**，反向 memcpy 即可。

**`m_default_val_expr`（`DEFAULT (expr)`）**：

```cpp
// [F1] sql/field.h
class Value_generator {
 public:
  /**
    Item representing the generation expression. ...
    Contrast this with the Field of a TABLE_SHARE, which has expr_item==NULL
    even if it's a generated column; that makes sense, as an Item tree cannot
    be shared.
  */
  Item *expr_item{nullptr};
  LEX_STRING expr_str{nullptr, 0};    /* TABLE_SHARE 里只有文本 */
  MY_BITMAP base_columns_map;         /* 表达式依赖的基列位图 */
  ...
};
```

★ **关键分工**：`TABLE_SHARE` 里的 `Value_generator` **只有 `expr_str`（文本）**——Item 树不能被多个 TABLE 共享；每个 `TABLE` 打开时才在 `open_table_from_share()` 里 `unpack_value_generator()` 解成 `expr_item`（`[F13]`）。

求值入口（**注意**：8.0.39 没有 `Table::evaluate_generated_default_function` 这个名字）：

```cpp
// [F2] sql/field.cc
void Field::evaluate_insert_default_function() {
  if (has_insert_default_datetime_value_expression())
    Item_func_now_local::store_in(this);                        // DEFAULT CURRENT_TIMESTAMP
  else if (has_insert_default_general_value_expression())
    m_default_val_expr->expr_item->save_in_field(this, false);  // DEFAULT (expr)
}
void Field::evaluate_update_default_function() {
  if (has_update_default_datetime_value_expression())
    Item_func_now_local::store_in(this);                        // ON UPDATE CURRENT_TIMESTAMP
}
```

调用者是 `COPY_INFO::set_function_defaults()`（INSERT/UPDATE 通用）。★ `base_columns_map` 的作用：因为 `DEFAULT (a+1)` 可能依赖别的列，求值时必须先把基列加入 `read_set`（引擎得真正读出来）——这也解释了为什么分区裁剪遇到函数默认值要放弃。

**`TABLE_SHARE::default_values` 的构造与灌入**：

```cpp
// [F5] sql/table.h —— 三个宏（B 是"成员名片段"，靠 ## 式拼接）
#define store_record(A, B)   memcpy((A)->B, (A)->record[0], (size_t)(A)->s->reclength)
#define restore_record(A, B) memcpy((A)->record[0], (A)->B, (size_t)(A)->s->reclength)
#define cmp_record(A, B)     memcmp((A)->record[0], (A)->B, (size_t)(A)->s->reclength)

static inline void empty_record(TABLE *table) {
  restore_record(table, s->default_values);       // 整行灌默认值
  if (table->s->null_bytes > 0)
    memset(table->null_flags, 255, table->s->null_bytes);   // 再把所有 NULL 位置 1
}
```

★ `set_default()` 是**单列粒度**补默认值，`restore_record` 是**整行粒度**（一次 memcpy 全部列）。INSERT 一般是先 `restore_record(table, s->default_values)` 拿整行默认值，再对显式给出的列 `store()` 覆盖。`empty_record()` = 灌默认值 + 全置 NULL。**DD 里存的就是"内存行格式的字节镜像"**，所以 `fill_column_from_dd()` 里 `memcpy(rec_pos, col_obj->default_value().data(), pack_length())` 可以零解析灌入（BIT 的零头位要单独 `set_rec_bits`）。

**隐式默认值**（无 explicit DEFAULT 时）——答案在 **DDL 建表阶段**（`prepare_default_value()`，`[F15]`）固化进 DD：

1. **可空列** → `set_null()`，DD 记 `default_value_null = true`
2. **NOT NULL 列** → `reset()`（类型零值：数值 0、字符串空格、DECIMAL 0、BIT 清 0）
3. ★ **ENUM NOT NULL 无 DEFAULT** → 特判 `store((longlong)1, true)`，即**第一个枚举元素**（下标 1），不是 0
4. `NO_DEFAULT_VALUE_FLAG` 只用于报错（严格模式），**不影响默认字节本身**

**`auto_flags` 与 `CURRENT_TIMESTAMP` 的时区语义**：

```cpp
// [F1] sql/field.h
  enum enum_auto_flags {
    NONE = 0,
    NEXT_NUMBER = 1,               ///<  AUTO_INCREMENT
    DEFAULT_NOW = 2,               ///<  DEFAULT CURRENT_TIMESTAMP
    ON_UPDATE_NOW = 4,             ///<  ON UPDATE CURRENT_TIMESTAMP
    GENERATED_FROM_EXPRESSION = 8  ///<  DEFAULT (expression)
  };
```

★ `auto_flags` 不直接存在 DD 字段里，而是**从 DD 的 `default_option`/`update_option` 文本重算**（`get_auto_flags()`，`[F14]`）——用 `compare(0,17,"CURRENT_TIMESTAMP")` 匹配前缀（兼容 `CURRENT_TIMESTAMP(6)`）。

链路：`set_default()` → `evaluate_insert_default_function()` → `Item_func_now_local::store_in(this)` → `store_timestamp()` → `Field_timestampf::store_timestamp_internal()`。★ 两个关键事实：

- 取的是**语句开始时间** `thd->query_start_timeval_trunc(decimals())`（同一语句多行同值），并按列精度截断——这就是 `CURRENT_TIMESTAMP(n)` 必须与列精度一致的原因。
- ★ **`store_timestamp()` / `get_timestamp()` 不依赖时区、永远"in UTC"**（`[F1]` 注释原文）。`my_timeval` 是 UTC epoch，直接 `my_timestamp_to_binary()` 落盘；**时区只在"用户值 → 存储"方向出现**（`store_internal()` 里 `convert_TIME_to_timestamp(ltime, *thd->time_zone(), &tm, ...)` 把会话时区下的 `MYSQL_TIME` 转成 UTC epoch）。所以 `SET time_zone=…` 对 `DEFAULT CURRENT_TIMESTAMP` 写入的**绝对时刻无影响**（只影响 SELECT 出来看到什么）——这与 DATETIME 的"字面量存"形成对照。

#### 7.2 ★ Instant ADD / DROP COLUMN：`Field` 完全不受影响

**核心矛盾**：DD/表定义多了一列 → `Field` 数组多一项、多一段 offset；但**聚簇索引里已有的旧物理记录没有这段字节**。

**结论（server 层）**：`Field` 侧**没有任何**"物理缺失"标记、没有 instant 分支、没有补默认值的代码。补齐工作 100% 由引擎在"把物理记录解成 MySQL 行"时完成；server 拿到的 `record[0]` 永远是一行完整的行。

**证据链（8.0.39 实证检索）**：

1. `sql/table.h`、`sql/field.h`、`sql/dd/**` 中**不存在**任何 instant 相关成员。对 `sql/**` 做全词检索 `\binstant\b`，命中只有 `sql/handler.h`/`handler.cc`（算法枚举与注释）和 `sql/sql_table.cc`（算法选择分支）——**没有一处是 `Field`/`TABLE`/`TABLE_SHARE` 的数据成员或谓词**。
2. `Field` 没有 `is_instant_added()` / `is_instant_dropped()` 之类谓词；`set_default()` / `copy_data()` / `pack_length()` / `offset()` 全是"按定义走 offset"的纯几何逻辑。
3. `sql/dd` 的公开接口里没有 instant 键（`instant_col|INSTANT_COL|has_instant` 在 `sql/` 下 **0 命中**）——instant 元数据写在 `se_private_data` 里由 InnoDB 用私有键读写（`DD_TABLE_INSTANT_COLS`、`DD_INSTANT_COLUMN_DEFAULT`、`DD_INSTANT_VERSION_ADDED/DROPPED`、`DD_INSTANT_PHYSICAL_POS`，`[F31]`）→ **属 InnoDB/record 文档**。
4. `fill_columns_from_dd()` 只遍历 `dd::Table::columns()`，instant 新增列在 DD 里就是普通列 → server 层甚至无从判断"这列是哪次 ALTER 加的"。

```
DD / table->field[] 视角（server 层）
   c0 | c1 | c2 | c3(INSTANT ADD, DEFAULT 42)
        ↑ 每个 Field 的 ptr = record[0] + 固定 offset，与 c3 何时加进来无关

聚簇索引中一条旧物理行（引擎层）
   c0 | c1 | c2                ← 没有 c3 的字节
        ↑ rec_offs 里 c3 打 REC_OFFS_DEFAULT，用 dict_col_t::instant_default 补齐
          （UNIV_SQL_ADD_COL_DEFAULT / has_instant_cols() —— 属 InnoDB 文档）

server 层 Field c3：从来不知道自己“在旧行里缺失过”
```

**server 侧 INSTANT 入口只有"算法选择"，没有"数据处理"**：

```cpp
// [F47] sql/sql_alter.h
  enum enum_alter_table_algorithm {
    ALTER_TABLE_ALGORITHM_DEFAULT,
    ALTER_TABLE_ALGORITHM_INPLACE,
    ALTER_TABLE_ALGORITHM_INSTANT,   // Instant if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_COPY
  };
// [F12] sql/sql_table.cc —— mysql_alter_table()
    if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
        inplace_supported != HA_ALTER_INPLACE_INSTANT && inplace_supported != HA_ALTER_ERROR) {
      ha_alter_info.report_unsupported_error("ALGORITHM=INSTANT", "ALGORITHM=COPY/INPLACE");
      ...
    }
```

★ server 层对 INSTANT 的全部职责就是"问引擎能不能、不能就报错/降级"；`handler.h` 注释明确说 **prepare 与 main 两个阶段对 INSTANT 是 no-op**，真正的字典改动在 `ha_commit_inplace_alter_table()`（由 InnoDB 完成）。整个 `mysql_alter_table()` 里**没有任何一处按列判断是否 instant**。

**instant DROP**：被 DROP 的列**仍留在 DD 的 columns 列表里**（保留元数据以便旧行仍能解释），但被标记为 **SE 隐藏**，server 建 `TABLE_SHARE` 时直接跳过：

```cpp
// [F14] fill_columns_from_dd()，两处同样的跳过
  for (const dd::Column *col_obj : tab_obj->columns()) {
    if (col_obj->is_se_hidden()) continue;      // ★ HT_HIDDEN_SE → 不生成 Field
```

后果：`table->field[]`、`share->fields`、`field_index()` 全不含该列（SQL 层无法再引用，`SELECT *` 也看不到）；旧行里的残留字节由引擎按 `version_dropped` / 行版本跳过（**属 InnoDB 文档**）。

**`stored_rec_length` vs `reclength`**：

```cpp
// [F5] sql/table.h
  ulong reclength{0};           /* 内存行缓冲长度（含虚拟列占位） */
  ulong stored_rec_length{0};   /* Stored record length (no generated-only generated fields) */
```

`reclength` 用于 `store_record/restore_record/cmp_record` 三个宏的 memcpy；`stored_rec_length` 初值等于 `reclength`，**在虚拟列被排到记录尾部时被修正**（见 7.3）。instant ADD 对二者的影响：新列在 server 层就是普通列 → 打开表时二者直接包含新列（又一次佐证"server 不区分旧行有没有"）。

#### 7.3 虚拟生成列

**★ 三态（符号核对：8.0.39 **不存在** `is_stored_gcol()`，也不存在 `vcol_info`——旧版的 `vcol_info` 已统一更名为 `gcol_info`）**：

```cpp
// [F1] sql/field.h
  Value_generator *gcol_info{nullptr};   /* 非空 = 生成列 */
  bool stored_in_db;                     /* false 只能是虚拟生成列 */
  Value_generator *m_default_val_expr{nullptr};   /* DEFAULT (expr) */
  bool is_gcol() const { return gcol_info; }
  bool is_virtual_gcol() const { return gcol_info && !stored_in_db; }
```

| | `gcol_info` | `stored_in_db` | `m_default_val_expr` |
|---|---|---|---|
| 普通列 | NULL | true | NULL |
| STORED 生成列 | 非 NULL | **true** | NULL |
| VIRTUAL 生成列 | 非 NULL | **false** | NULL |
| 有 `DEFAULT (expr)` | NULL（除非本身也是生成列） | true | 非 NULL |

★ "STORED 生成列"的写法是 `is_gcol() && stored_in_db`（`update_generated_columns()` 里用 `virtual_only && !field->is_virtual_gcol()` 跳过它）。★ `DEFAULT GENERATED is always stored`——默认值表达式的 `Value_generator::stored_in_db` 恒 true，不参与"虚拟/存储"语义。

**★ `Field_blob` 虚拟列特例：old / new / deleted 三份值**

判据：

```cpp
// [F1] sql/field.h
  bool handle_old_value() const {
    return (is_flag_set(BLOB_FLAG) || is_array()) && is_virtual_gcol();
  }
```

三个成员：`old_value`（旧值缓冲）、`m_keep_old_value`（是否需要转移）、`m_blob_backup`（第三份"deleted"的备份，`[F1]` 注释详述了 INSERT 与刚删除的索引项重合时需要三份值）。

```cpp
// [F1] sql/field.h
  void keep_old_value() {
    assert(is_virtual_gcol());
    /* Transfer ownership of the current BLOB value to old_value */
    if (m_keep_old_value) { old_value.takeover(value); m_keep_old_value = false; }
  }
// [F2] sql/field.cc
bool  Field_blob::backup_blob_field()   { value.swap(m_blob_backup); return false; }
void  Field_blob::restore_blob_backup() { value.swap(m_blob_backup); }
```

```
 (1) 读行 → update_generated_read_fields() 算出虚拟 BLOB 值，写进 value        ⇒ "old"
 (2) UPDATE 算新值前 → keep_old_value() 把 value 所有权 takeover 到 old_value，
     record[1] 的 BLOB 指针指向 old_value                                      ⇒ old_value="old"，value 空出
 (3) 算新值写进 value ⇒ value="new"；引擎同时读 record[1](old) 与 record[0](new) 维护虚拟列上的索引
 (4) 若同语句还需"被删除的那份"（INSERT 与刚删除的索引项重合时 InnoDB 要重算）⇒
     backup_blob_field() 把 value 换到 m_blob_backup，腾出空间算 "deleted"
```

★ **为什么需要这套**：普通（stored）BLOB 的旧值由**引擎自己持有缓冲**，`Field_blob::value` 一直是空的、可直接放新值；而虚拟列的旧值只能由 server 现算并存进 `value`，于是必须"先转移再复用"。

**★ 虚拟列的存储布局：第二遍扫描**（`[F14]`）：

```cpp
// 第一遍：只放物理列
  for (const dd::Column *col_obj : tab_obj->columns()) {
    if (col_obj->is_se_hidden()) continue;
    /* Skip virtual generated columns at this point. They reside at the end of
       the record, so we need to do separate pass, to evaluate their offsets
       correctly. */
    if (!col_obj->is_virtual()) {
      fill_column_from_dd(thd, share, col_obj, null_pos, null_bit_pos, rec_pos, field_nr);
      if ((null_bit_pos += column_preamble_bits(col_obj)) > 7) { null_pos++; null_bit_pos -= 8; }
      rec_pos += share->field[field_nr]->pack_length_in_rec();
    } else has_vgc = true;
    field_nr++;
  }
  if (has_vgc) {
    /* ★ 就在第二遍开始前修正 stored_rec_length（此刻 rec_pos 停在最后一个物理列之后） */
    if (share->stored_rec_length > static_cast<ulong>(rec_pos - share->default_values))
      share->stored_rec_length = (rec_pos - share->default_values);
    field_nr = 0;
    for (const dd::Column *col_obj2 : tab_obj->columns()) {   // 第二遍：只放虚拟列
      if (col_obj2->is_se_hidden()) continue;
      if (col_obj2->is_virtual()) {
        fill_column_from_dd(thd, share, col_obj2, null_pos, null_bit_pos, rec_pos, field_nr);
        ...
        rec_pos += share->field[field_nr]->pack_length_in_rec();
      }
      field_nr++;
    }
  }
```

★ 三个推论：① **虚拟列在记录缓冲里永远排在末尾**，而 `share->field[]` 的**下标顺序按 DDL 顺序**——二者不一致；② NULL 位也分两遍推进（注释："so the NULL bits end up in the expected order"）；③ **虚拟列仍然"占位"**（`pack_length_in_rec()`）——BLOB/JSON 虚拟列占的是"长度前缀 + 指针"，server 算完后把指针写这里，引擎不落盘但能顺着指针读值去维护索引。

**★ 求值时机：引擎返回行之后、server 使用行之前**

```cpp
// [F13] sql/table.cc
bool update_generated_read_fields(uchar *buf, TABLE *table, uint active_index) {
  assert(table != nullptr && table->has_gcol());
  if (current_thd->is_error()) return true;
  if (active_index != MAX_KEY && table->key_read) {
    /* 覆盖索引已提供所有列（含生成列）→ 不必重算 */
    return false;
  }
  /* buf 不是 record[0] 时，临时把 Field::ptr 整体平移，算完再移回 */
  if (buf != table->record[0]) repoint_field_to_record(table, table->record[0], buf);
  const bool error = update_generated_columns(table, table->read_set, true, nullptr);
  if (buf != table->record[0]) repoint_field_to_record(table, buf, table->record[0]);
  return error;
}

static bool update_generated_columns(TABLE *table, const MY_BITMAP *columns,
                                     bool virtual_only, MY_BITMAP *updated_columns) {
  for (Field **field_ptr = table->vfield; *field_ptr != nullptr; ++field_ptr) {
    Field *field = *field_ptr;
    if (virtual_only && !field->is_virtual_gcol()) continue;         // 跳过 STORED
    if (!bitmap_is_set(columns, field->field_index())) continue;     // 只算 read_set 里的
    if (field->handle_old_value()) {                                  // BLOB 虚拟列：保留旧值
      const auto blob = down_cast<Field_blob *>(field);
      blob->keep_old_value(); blob->set_keep_old_value(true);
    }
    type_conversion_status status = field->gcol_info->expr_item->save_in_field(field, false);
    if (status != TYPE_OK && thd->is_error()) return true;
    ...
  }
}
```

触发点是每个读接口（`[F6]`，`ha_rnd_next` / `ha_rnd_pos` / `ha_index_read_map` / `ha_index_next` / …同一个套式）：

```cpp
int handler::ha_rnd_next(uchar *buf) {
  m_update_generated_read_fields = table->has_gcol();     // ① 先置标志
  MYSQL_TABLE_IO_WAIT(..., { result = rnd_next(buf); })   // ② 引擎填行
  if (!result && m_update_generated_read_fields) {
    result = update_generated_read_fields(buf, table);    // ③ ★ 返回之后立即补算
    m_update_generated_read_fields = false;
  }
  ...
}
```

★ 要点：① **不是"在 rnd_next 之前"，而是"之后立即"**；② 条件是 `table->has_gcol()`；③ 覆盖索引（`key_read`）短路；④ `buf != record[0]` 时用 `repoint_field_to_record()` 临时平移——又一次印证"**Field 的定位 = 基址 + 固定 offset**"这一贯穿前几章的核心模型。

**虚拟列上的二级索引**：值**会物化在索引里**（server 算 → 引擎持久化）。server 侧用 KEY flag 标记：

```cpp
        if (field->is_virtual_gcol()) keyinfo->flags |= HA_VIRTUAL_GEN_KEY;
// [F5]
  inline bool index_contains_some_virtual_gcol(uint index_no) const {
    return key_info[index_no].flags & HA_VIRTUAL_GEN_KEY;
  }
```

★ 含虚拟列的索引**不作为"可用唯一键"**（`[F12]`：`if (key->flags & HA_VIRTUAL_GEN_KEY) return false;`），也不能当主键（`is_suitable_for_primary_key()` 里 `if (table_field->is_virtual_gcol()) return false;`）。
★ 符号核对：`m_read_virtual_key` 在 8.0.39 中**不在 server 层**（`sql/` 下 0 命中），它是 InnoDB `row_prebuilt_t` 的成员（`[F30]`）→ **属 InnoDB 文档**。

#### 7.4 小结

| 主题 | server 层 `Field` 的行为 | 引擎层补位 |
|---|---|---|
| 默认值 | `set_default()`：`copy_data(default_values_offset())` 或 `evaluate_insert_default_function()`；整行用 `restore_record(table, s->default_values)` | 无 |
| 隐式默认值 | DDL 期由 `prepare_default_value()` 固化进 DD（可空→NULL / NOT NULL→`reset()` / ENUM→下标 1） | 无 |
| `DEFAULT (expr)` | `m_default_val_expr`，开表时 `unpack_value_generator()` 生成 `expr_item`，DML 期 `COPY_INFO::set_function_defaults()` 求值 | 无 |
| instant ADD | **完全无感**：新列在 server 层就是普通列 | InnoDB 用 `has_instant_cols()` / `REC_OFFS_DEFAULT` / `UNIV_SQL_ADD_COL_DEFAULT` 补齐 —— **属 InnoDB 文档** |
| instant DROP | 列被标记 `is_se_hidden()`，`fill_columns_from_dd()` 直接 `continue` ⇒ 从 `table->field[]` 消失 | 旧行残留按 `version_dropped` 跳过 —— **属 InnoDB 文档** |
| 虚拟生成列 | `is_virtual_gcol()`；排在物理列之后；读行后由 `update_generated_read_fields()` 现算；BLOB 虚拟列要保留 old/new/deleted 三份值 | 索引里的虚拟列键值由引擎持久化；`m_read_virtual_key` 是引擎侧开关 —— **属 InnoDB 文档** |

**★ 贯穿三者的统一模型**：`Field` 只认识"**基址 + 偏移 + `pack_length()`**"。默认值是"把模板行的那段字节搬到当前行"；instant 是"那段字节由引擎假装从磁盘读出来"；虚拟列是"那段字节由 server 在引擎返回后自己算出来"。**三种场景下 `Field` 的几何定位代码一行都不用改**——这正是这套 offset 模型的价值所在。

---

### 八、★ `make_field`：把列定义变成 `Field` 对象

> 补充源码坐标：`[F48]` sql/create_field.h、`[F49]` sql/create_field.cc、`[F50]` sql/table.cc（FRM 遗留路径）、`[F51]` sql/dd_sp.cc、`[F52]` sql/rpl_utility.cc、`[F53]` sql/sql_table.cc、`[F54]` sql/default_values.cc / sql/sql_tmp_table.cc / sql/dd/dd_table.cc（调用点）。

#### 8.0 ★ 第一易错点：它是**自由函数重载**，不是 `Field` 的成员

`Field` 类里**没有任何**叫 `make_field` 的成员函数。`make_field` 是定义在 `sql/field.cc`、声明在 `sql/field.h` 的**全局自由函数**，靠 C++ 重载决议区分入参，也不参与虚派发。它是"**把列定义变成 `Field` 对象**"的唯一集中工厂——所有 `Field_xxx` 的 `new` 都收敛在这一个函数里。

还要当心**三个同名陷阱**：

| 同名者 | 位置 | 是什么 |
|---|---|---|
| `static make_field(const dd::Column&, ...)` | `[F14] sql/dd_table_share.cc:807` | 文件内 static，内部**回调**大参数版 |
| `static make_field(const dd::Parameter&, ...)` | `[F51] sql/dd_sp.cc:96` | 给存储过程参数造"临时 Field"，只为拼 `sql_type()` 字符串 |
| `virtual void Item::make_field(Send_field*)` | `sql/item.h:1206` | **与建列毫无关系**——它是给客户端协议填 `Send_field` 元数据的。grep `make_field` 时 90% 的命中都来自它 |

#### 8.1 全部重载

**① 大参数版（唯一的"真身"，其它最终都落到它）**

```cpp
// [F1] sql/field.h
/// This function should only be called from legacy code.
Field *make_field(MEM_ROOT *mem_root_arg, TABLE_SHARE *share, uchar *ptr,
                  size_t field_length, uchar *null_pos, uchar null_bit,
                  enum_field_types field_type,
                  const CHARSET_INFO *field_charset,
                  Field::geometry_type geom_type, uchar auto_flags,
                  TYPELIB *interval, const char *field_name, bool is_nullable,
                  bool is_zerofill, bool is_unsigned, uint decimals,
                  bool treat_bit_as_char, uint pack_length_override,
                  std::optional<gis::srid_t> srid, bool is_array);
```

★ `share` 允许为 `nullptr`（只借 Field 做类型推导时用）；`null_bit` 这里还是**"序号"**（0..7），进入函数体才变掩码。直接调用点只有 4 处：`[F50] table.cc:1344`（FRM 遗留）、`[F14] dd_table_share.cc:884`、`[F51] dd_sp.cc:108`、`[F2] field.cc:10038`（数组元素字段）。

**② `Create_field` 版三兄弟**（CREATE / ALTER / 临时表 / 默认值）——都是薄包装，把 `Create_field` 摊平成大参数版的实参：

```cpp
Field *make_field(const Create_field &create_field, TABLE_SHARE *share,
                  const char *field_name, size_t field_length, uchar *ptr,
                  uchar *null_pos, size_t null_bit);   // 最灵活：名字和长度都能改写
Field *make_field(const Create_field &create_field, TABLE_SHARE *share,
                  uchar *ptr, uchar *null_pos, size_t null_bit);  // ★ 长度取 max_display_width_in_bytes()
Field *make_field(const Create_field &create_field, TABLE_SHARE *share);  // 无缓冲版：只问类型
```

典型调用点：

```cpp
// [F54] sql/sql_tmp_table.cc —— 创建内部临时表：可空列给一个假 null_pos（指向常量 ""）
  *reg_field = cdef->is_nullable
      ? make_field(*cdef, share, nullptr,
                   pointer_cast<uchar *>(const_cast<char *>("")), 1)
      : make_field(*cdef, share);
  (*reg_field)->init(table);                       // ★ 必须 init() 才能用

// [F54] sql/default_values.cc —— 默认值落盘：buf[0] 是 null 位，数据从 buf+1 开始
  Field *regfield = make_field(field, table->s, buf + 1, buf, 0 /* null_bit */);
```

**③ DD 版**（打开表时从数据字典重建 `TABLE_SHARE`）——它承担了所有 **options（非一等公民属性）** 的读取，见 8.5。

#### 8.2 ★ 函数体开头的三件"与字节布局直接相关"的事

```cpp
// [F2] sql/field.cc
  uchar *bit_ptr = nullptr;
  uchar bit_offset = 0;

  /* ① BIT 列：算出"真正的 BIT 数据位"所在的字节指针与位内偏移。
        BIT 的数据位紧跟在 null 位之后（同一个 null 字节区），所以起点就是
        null_pos + null_bit；若该列可空，null 标志本身已占了这一位，必须整体错开一位。 */
  if (field_type == MYSQL_TYPE_BIT && !treat_bit_as_char) {
    bit_ptr = null_pos;
    bit_offset = null_bit;
    if (is_nullable) {
      bit_ptr += (null_bit == 7);            // 位号已到字节末尾 → 进位到下一字节
      bit_offset = (bit_offset + 1) & 7;     // 位号 +1 并对 8 取模
    }
  }

  /* ② null_bit 语义切换：入参是"第几个 null 位(序号)"，Field 内部要的是"掩码" */
  if (!is_nullable) {
    null_pos = nullptr;                      // 不可空列：从此不再碰 null 字节
    null_bit = 0;
  } else {
    null_bit = ((uchar)1) << null_bit;       // 序号 0..7 → 掩码 1/2/4/.../128
  }

  /* ③ 时态类型强制 numeric 字符集：它们本质是数值，不该带 collation 语义 */
  if (is_temporal_real_type(field_type)) field_charset = &my_charset_numeric;
```

★ ③ 很重要：`Field::charset()` 决定 `has_charset()`、排序规则、`val_str()` 的填充方式。时态列若继承表的 `utf8mb4` 会误触发 `Field_str` 路径。同理 BLOB/GEOMETRY 走 `my_charset_bin`（由**调用方**保证）。

#### 8.3 前置分支 `is_array`（多值索引的内部列）

在 `switch (field_type)` **之前**还有一个拦截分支——多值索引的内部列**不走普通 `Field_json`**：

```cpp
  if (is_array) {
    switch (field_type) {
      case MYSQL_TYPE_VARCHAR: case MYSQL_TYPE_NEWDECIMAL:
      case MYSQL_TYPE_LONGLONG: case MYSQL_TYPE_NEWDATE:
        break;                                     // ★ 只有这 4 种元素类型被支持
      case MYSQL_TYPE_TIME2:                       // 元素若是 TIME：从长度反解小数秒位
        decimals = (field_length > MAX_TIME_WIDTH)
                       ? field_length - 1 - MAX_TIME_WIDTH : 0;
        break;
      case MYSQL_TYPE_DATETIME2:
        decimals = (field_length > MAX_DATETIME_WIDTH)
                       ? field_length - 1 - MAX_DATETIME_WIDTH : 0;
        break;
      case MYSQL_TYPE_YEAR:
      default:
        assert(0); return nullptr;
    }
    /* Field_json constructor expects number of bytes used to represent length
       of the underlying blob as parameter and not the real field pack_length.
       JSON is used as array storage. */
    uint pack_length = calc_pack_length(MYSQL_TYPE_JSON, field_length) -
                       portable_sizeof_char_ptr;    // 12 - 8 = 4
    return new (mem_root) Field_typed_array(
        field_type, is_unsigned, field_length, decimals, ptr, null_pos,
        null_bit, auto_flags, field_name, share, pack_length, field_charset);
  }
```

#### 8.4 ★ 核心 `switch (field_type)`：每个 case 与构造实参

**字符族**——★ "建 `Field_string` 还是 `Field_varstring`"**只由 `enum_field_types` 决定，没有长度阈值判断**：

```cpp
    case MYSQL_TYPE_VAR_STRING:                    // ★ FRM 遗留：老 .frm 里 VARCHAR 用这个编号
    case MYSQL_TYPE_STRING:                        // CHAR → 定长，不带长度前缀
      return new (mem_root) Field_string(ptr, field_length, null_pos, null_bit,
                                         auto_flags, field_name, field_charset);
    case MYSQL_TYPE_VARCHAR:                       // 真 VARCHAR：多传"长度前缀字节数"与 share
      return new (mem_root) Field_varstring(
          ptr, field_length, HA_VARCHAR_PACKLENGTH(field_length), null_pos,
          null_bit, auto_flags, field_name, share, field_charset);
```

**BLOB 族 / GEOMETRY / JSON**——★ **四种 BLOB 靠 `field_type` 区分**，而传进去的 `pack_length` 是**"长度字段的字节数"，不是总长**（内存布局 = 长度前缀 + 一个 8 字节 `char*`）：

```cpp
    case MYSQL_TYPE_BLOB: case MYSQL_TYPE_MEDIUM_BLOB:
    case MYSQL_TYPE_TINY_BLOB: case MYSQL_TYPE_LONG_BLOB: {
      /* Field_blob constructor expects number of bytes used to represent length
         of the blob as parameter and not the real field pack_length. */
      uint pack_length =
          calc_pack_length(field_type, field_length) - portable_sizeof_char_ptr;  // 1/2/3/4
      return new (mem_root) Field_blob(ptr, null_pos, null_bit, auto_flags,
                                       field_name, share, pack_length, field_charset);
    }
    case MYSQL_TYPE_GEOMETRY: { /* 同式 → pack_length == 4；额外传 geom_type 与 srid */ }
    case MYSQL_TYPE_JSON: {     /* 同式 → pack_length == 4；不需要 charset（隐含 utf8mb4_bin） */ }
```

★ 注意 BLOB 分支**根本没传 `field_length`**——"最大长度"由 `pack_length`（前缀宽度）隐含推出（1→255, 2→65535, 3→16M, 4→4G）。

**ENUM / SET**——★ `pack_length_override` 的用武之地：

```cpp
    case MYSQL_TYPE_ENUM:
      assert(interval);                            // ENUM/SET 必须有 TYPELIB
      return new (mem_root) Field_enum(
          ptr, field_length, null_pos, null_bit, auto_flags, field_name,
          (pack_length_override ? pack_length_override
                                : get_enum_pack_length(interval->count)),
          interval, field_charset);
    case MYSQL_TYPE_SET:                           // 同理，Field_set
```

```cpp
// [F1] sql/field.h
inline uint get_enum_pack_length(int elements) { return elements < 256 ? 1 : 2; }
inline uint get_set_pack_length(int elements) {
  uint len = (elements + 7) / 8;
  return len > 4 ? 8 : len;                        // ★ 合法结果只有 {1,2,3,4,8}
}
```

`pack_length_override` **非 0 时优先于按成员数算出的宽度**，唯一来源是 RBR：

```cpp
// [F52] sql/rpl_utility.cc
    case MYSQL_TYPE_ENUM:
    case MYSQL_TYPE_SET:
      /* Number of elements in interval on master and slave might differ.
         Use pack length from binary log instead of one calculated from
         number of interval elements on slave. */
      pack_length_override = field_metadata(col_i) & 0x00ff;
```

**数值族**——整数类构造签名是 `(ptr, field_length, null_pos, null_bit, auto_flags, field_name, is_zerofill, is_unsigned)`（**无 `decimals`**）；浮点类是 `(..., decimals, is_zerofill, is_unsigned)`，未指定小数位时 `decimals == DECIMAL_NOT_SPECIFIED`；`MYSQL_TYPE_INT24`（MEDIUMINT）→ `Field_medium`。

**时态族 + YEAR**——★ 新旧两代的分水岭与 fsp 反解：

```cpp
    case MYSQL_TYPE_TIMESTAMP:                     // 旧：定长 4 字节，无小数秒
      return new (mem_root) Field_timestamp(ptr, field_length, null_pos,
                                            null_bit, auto_flags, field_name);
    case MYSQL_TYPE_TIMESTAMP2:                    // 新：fsp 从 field_length 反解
      return new (mem_root) Field_timestampf(
          ptr, null_pos, null_bit, auto_flags, field_name,
          field_length > MAX_DATETIME_WIDTH ? field_length - 1 - MAX_DATETIME_WIDTH : 0);
    case MYSQL_TYPE_YEAR:
      assert(field_length == 4);  // Field_year is only for length 4.
      return new (mem_root) Field_year(ptr, null_pos, null_bit, auto_flags, field_name);
    case MYSQL_TYPE_NEWDATE:                       // ★ switch 里【没有】 MYSQL_TYPE_DATE 这个 case！
      return new (mem_root) Field_newdate(ptr, null_pos, null_bit, auto_flags, field_name);
    case MYSQL_TYPE_TIME:                          // 旧 3 字节
    case MYSQL_TYPE_TIME2:                         // 新：fsp = field_length - 1 - MAX_TIME_WIDTH(10)
    case MYSQL_TYPE_DATETIME:                      // 旧 8 字节
    case MYSQL_TYPE_DATETIME2:                     // 新：fsp = field_length - 1 - MAX_DATETIME_WIDTH(19)
```

| SQL 类型 | 老编号 | 新编号 | Field | fsp 来源 |
|---|---|---|---|---|
| DATE | `MYSQL_TYPE_DATE`（已废弃） | `MYSQL_TYPE_NEWDATE` | `Field_newdate` | 无 |
| TIME | `MYSQL_TYPE_TIME` | `MYSQL_TYPE_TIME2` | `Field_time` / `Field_timef` | `field_length - 1 - MAX_TIME_WIDTH` |
| DATETIME | `MYSQL_TYPE_DATETIME` | `MYSQL_TYPE_DATETIME2` | `Field_datetime` / `Field_datetimef` | `field_length - 1 - MAX_DATETIME_WIDTH` |
| TIMESTAMP | `MYSQL_TYPE_TIMESTAMP` | `MYSQL_TYPE_TIMESTAMP2` | `Field_timestamp` / `Field_timestampf` | 同上 |
| YEAR | `MYSQL_TYPE_YEAR` | （无新版） | `Field_year` | 无，`assert(field_length == 4)` |

反解公式的由来（`[F49] create_field.cc`）：`m_max_display_width_in_codepoints = MAX_DATETIME_WIDTH + (decimals ? 1 + decimals : 0)`——即 `field_length = 19 + (fsp ? 1+fsp : 0)`，反解即 `fsp = field_length - 1 - 19`。

**BIT / NULL / default**：

```cpp
    case MYSQL_TYPE_BIT:
      /* treat_bit_as_char 决定两个完全不同的物理布局 */
      return treat_bit_as_char
                 ? new (mem_root) Field_bit_as_char(ptr, field_length, null_pos,
                                                    null_bit, auto_flags, field_name)
                 : new (mem_root) Field_bit(ptr, field_length, null_pos, null_bit,
                                            bit_ptr, bit_offset, auto_flags, field_name);
    case MYSQL_TYPE_NULL:                          // SELECT NULL 之类的"空列"
      return new (mem_root) Field_null(ptr, field_length, auto_flags, field_name, field_charset);
    case MYSQL_TYPE_INVALID:
    case MYSQL_TYPE_BOOL:
    default:  break;                               // ★ 不构造任何对象
  }
  return nullptr;                                  // 调用方必须判空！
```

#### 8.5 `Create_field` 版 vs DD 版的分工

| 维度 | `Create_field` 版 | DD 版（`[F14]:807`） |
|---|---|---|
| 触发场景 | CREATE / ALTER / 内部临时表 / 默认值落盘 / 存储函数返回值 / I_S 类型串 | `open_table_def()` 打开表，从 `dd::columns` 重建 `TABLE_SHARE->field[]` |
| 输入 | `Create_field`（SQL 解析后的内存结构，已完整） | `dd::Column`（持久化对象，只存固定列 + options） |
| 需要"现场补足" | 无 | **重建 `TYPELIB`**（ENUM/SET 元素）、从 `options()` 解出 `geom_type` / `treat_bit_as_char`、用 `get_auto_flags()` 还原 `auto_flags` |
| 长度语义 | `max_display_width_in_bytes()`（字符列乘 `mbmaxlen`） | `col_obj.char_length()`（DD 里已按字节存） |
| `pack_length_override` | 来自 `Create_field`（RBR 场景） | **恒为 0** |

★ **DD 期必须切换 mem_root**（`[F14]:2294`）：

```cpp
  thd->mem_root = &share->mem_root;  // Needed for make_field()++
```

因为 `make_field()` 内部大量用 `*THR_MALLOC`（即 `thd->mem_root`）做 `new (mem_root) Field_xxx(...)`；不切换则 Field 会随语句结束而失效。

#### 8.6 ★ 易错点清单（11 条）

1. **`make_field` 是自由函数重载，不是 `Field` 成员**；`Item::make_field(Send_field*)` 是同名但无关的方法（协议元数据）。
2. **`TINYINT(1)` 与 BOOL**：`BOOL` 只是 `TINYINT(1)` 的同义词，走 `Field_tiny`；★ 枚举值 `MYSQL_TYPE_BOOL` 在 `make_field` 里**明确没有 case**，与 `MYSQL_TYPE_INVALID` 一起落到 `default` **返回 `nullptr`**。
3. ★ **`MYSQL_TYPE_DATE` 不在 switch 里**——`DATE` 在 `Create_field::init()` / `init_for_tmp_table()` 就被改写成 `MYSQL_TYPE_NEWDATE`（`[F49]:500`、`:606`）。若真有 `MYSQL_TYPE_DATE` 传进来（老 FRM / binlog），**返回 `nullptr`** 而不是 `Field_newdate`。
4. **`zerofill` 不改变 `pack_length`**——整数/定点的 pack 长度只由类型决定。它只做两件事：`set_flag(ZEROFILL_FLAG)` 与 `val_str()` 时左补 `'0'`。⇒ **zerofill 的"宽度"是显示宽度，不是存储宽度**。
5. **长度 0 的列合法**：`CHAR(0)` → `Field_string` 的 `row_pack_length() == 0`；`VARCHAR(0)` 仍占 `HA_VARCHAR_PACKLENGTH(0) == 1` 字节（长度前缀）。
6. ★ **`MYSQL_TYPE_STRING` 与 `VAR_STRING` 的历史差异**：FRM 时代 VARCHAR 用 `VAR_STRING`；DD 时代 VARCHAR 一律 `MYSQL_TYPE_VARCHAR`。本版本 `VAR_STRING` 与 `STRING` **共用同一 case**，都建成 `Field_string`（定长）——读 FRM 路径时最容易踩。
7. **`null_bit` 入参是"序号"不是"掩码"**；不可空列被强制 `null_pos = nullptr, null_bit = 0`。
8. **BIT 的数据位与 null 位"挤在同一字节区"**，可空时整体错开一位。
9. ★ **BLOB / GEOMETRY / JSON 的 `pack_length` 形参名骗人**——它装的是"长度字段字节数"（1/2/3/4），真实内存占用 = 该值 + 8（指针）。
10. **必须 `init()`**：`make_field()` 只 `new` 对象、不绑定 `TABLE`；所有调用点后面都紧跟 `field->init(table)`。
11. **`Create_field` 版传的是"显示宽度"而非"pack 长度"**——真正存储宽度要看 `Create_field::pack_length()` 或建成后的 `Field::pack_length()`。

---

### 九、★ `fix_fields`：列名 → `Field*` 的完整解析链路

> **与第八章 `make_field`、3.10~3.12 的 `pack`/`unpack` 并列为 `Field` 的三个核心函数**：`make_field` 负责"造出 Field 对象"，`fix_fields` 负责"把 SQL 里的列名绑定到某个 Field"，`pack`/`unpack` 负责"把 Field 解释的字节变成可传输流"。
>
> 补充源码坐标：`[F31]` sql/sql_const.h（`enum_mark_columns`）、`[F32]` sql/sql_class.h（`THD::mark_used_columns`）、`[F33]` sql/auth/sql_authorization.cc、`[F34]` sql/sql_resolver.cc、`[F36]` sql/sql_partition.cc、`[F37]` sql/histograms/histogram.cc、`[F35]` ha_innodb.cc（`build_template_needs_field`）。

#### 9.0 全链路图

```
SQL 文本:  SELECT t.a FROM t
    │  词法/语法分析 → PTI_simple_ident → new Item_field(context, "t", "a")
    │     ▲ 此刻只有 3 个字符串 + Name_resolution_context，field == nullptr
    ▼
Item_field::fix_fields(thd, Item **reference)                    [F4]
    ├─ table_ref != nullptr ? → set_field(field); return false;   (cloned field 快路径)
    └─ find_field_in_tables(...)                                  [F27]
         └─ for (每个名字解析序上的 Table_ref)
              find_field_in_table_ref(...)                        [F27]
                ├─ 形态一 field_translation（视图/I_S）→ find_field_in_view()
                │     → *ref = Item_view_ref; return view_ref_found(0x2)
                ├─ 形态二 nested_join（NATURAL/USING）→ find_field_in_natural_join()
                └─ 形态三 基表 → find_field_in_table()            [F27]
                       └─ 线性扫 table->field[] + my_strcasecmp
                          *field_index_ptr = field_ptr - table->field;  ★ 指针差算列号
              ├─ 副作用① check_column_grant_in_table_ref() → 失败返回 WRONG_GRANT(-1)
              └─ 副作用② fld->table->mark_column_used(fld, thd->mark_used_columns)
         ├─ 第二张表也命中 → ER_NON_UNIQ_ERROR（歧义）
         └─ 一张都没命中 → REPORT_ALL_ERRORS ? ER_BAD_FIELD_ERROR : not_found_field(0x1)
    ▼
  set_field(from_field)  ← ★ 名字解析的落点：field = table->field[field_index]
  fixed = true
    ▼
执行期 val_int()/val_str() 直接读 field->ptr（= record[0] + offset）
    ▼
引擎侧 build_template_needs_field()：bitmap_is_set(table->read_set, i) 决定第 i 列进不进模板  [F35]
```

#### 9.1 第 0 层：解析产物里只有字符串

```cpp
// [F3] sql/item.h —— 搜索范围由这两个指针界定
struct Name_resolution_context {
  Name_resolution_context *outer_context;      // 找不到时向上一层子查询继续找
  Table_ref *table_list;                       // 本层 FROM 的完整表引用树
  Table_ref *first_name_resolution_table;      // 真正用于列解析的"叶子表链表"起点
  Table_ref *last_name_resolution_table;       // 终点（开区间）
  ...
  void resolve_in_table_list_only(Table_ref *tables) {   // JOIN..ON 的局部上下文
    table_list = first_name_resolution_table = tables;
    resolve_in_select_list = false;
  }
};
#define NO_FIELD_INDEX ((uint16)(-1))          // field_index 的"未知"哨兵
```

#### 9.2 第 1 层：`Item_field::fix_fields()`——决定"范围 + 策略 + 收尾"

```cpp
// [F4] sql/item.cc
bool Item_field::fix_fields(THD *thd, Item **reference) {
  assert(!fixed);
  Field *from_field = not_found_field;          // 哨兵 0x1 = "没找到"
  bool outer_fixed = false;

  if (table_ref != nullptr) {
    /* 分支 A：cloned field（条件下推到 derived table 时克隆出来的），
       已带 table_ref 和 field，只需重跑 set_field() 补齐派生属性 */
    Table_ref *orig_table_ref = table_ref;
    set_field(field);
    table_ref = orig_table_ref;   // set_field() 会覆盖 table_ref，这里强行改回
    return false;
  }
  assert(field == nullptr);

  /* 分支 B：正常名字解析。★ 报错策略：
     - use_only_table_context（生成列/默认值/检查约束表达式）→ REPORT_ALL_ERRORS，立刻报
     - 否则 IGNORE_EXCEPT_NON_UNIQUE ——"找不到"不报（留机会给 SELECT 列表别名与外层
       相关子查询），但"歧义"必须立刻报（换作用域也救不回来） */
  from_field = find_field_in_tables(
      thd, this, context->first_name_resolution_table,
      context->last_name_resolution_table, reference,
      thd->lex->use_only_table_context ? REPORT_ALL_ERRORS
                                       : IGNORE_EXCEPT_NON_UNIQUE,
      any_privileges ? 0 : thd->want_privilege, true);
  if (thd->is_error()) return true;

  if (from_field == not_found_field || from_field == nullptr) {
    /* ① 先在本层 SELECT 列表里按"别名"找（ORDER BY c / GROUP BY c） */
    if (qb->is_item_list_lookup) { ... set_field(new_field) 或 *reference = Item_ref ... }
    /* ② 再向外层查询逐层找（相关子查询） */
    int ret;
    if ((ret = fix_outer_field(thd, &from_field, reference)) < 0) return true;
    outer_fixed = true;
    if (!ret) return false;
  }
  ...
  /* 分支 C：解析到视图/derived 的列 → *reference 已被替换为 Item_view_ref */
  if (from_field == view_ref_found) { ...; return false; }

  /* ★ 隐藏列（引擎隐藏 / 函数索引的隐藏生成列）对用户报"列不存在" */
  if (from_field->is_hidden_by_system()) {
    my_error(ER_BAD_FIELD_ERROR, MYF(0), from_field->field_name, thd->where);
    return true;
  }

  /* 分支 D：基表列 —— 落点就在 set_field() */
  set_field(from_field);
  ...
  fixed = true;
  /* ★ 隐式聚合（SELECT COUNT(*), col FROM t 无 GROUP BY）：空表时整行置 NULL
        ⇒ 非 NULL 列也可能返回 NULL，要把 Item 和 TABLE 一起标 nullable */
  if (is_null_on_empty_table(thd, this)) {
    set_nullable(true);
    field->table->set_nullable();
  }
  return false;
}
```

★ `fix_fields` 自己**不做字符串比较**，只负责"范围 + 策略 + 收尾"。三种结果形态：`Field*`（基表列）、`view_ref_found`（`*reference` 被替换成 `Item_view_ref`）、`not_found_field`（继续找别名/外层）。

#### 9.3 第 2 层：`find_field_in_tables()`——遍历表链表 + 歧义判定

先记住**三个哨兵返回值**（这一族函数的"协议"）：

```cpp
// [F27] sql/sql_base.cc
Field *not_found_field = (Field *)0x1;   // 没找到（且未报错）
Field *view_ref_found  = (Field *)0x2;   // 找到的是视图/derived 的列，真身在 *ref
#define WRONG_GRANT (Field *)-1          // 找到了但没权限
```

```cpp
  /* prepared statement 快路径：上次已记住表，直接查这一张 */
  if (item->cached_table) {
    ...
    if (table_ref->table && !table_ref->is_view()) {
      found = find_field_in_table(table_ref->table, name, true, &field_index);
      if (found && want_privilege && check_column_grant_in_table_ref(...))
        found = WRONG_GRANT;
      if (found && found != WRONG_GRANT)
        table_ref->table->mark_column_used(found, thd->mark_used_columns);
    }
    ...
  }

  /* ★ 核心：沿 next_name_resolution_table 走"名字解析序"链表（不是 next_local） */
  for (cur_table = first_table; cur_table != last_table;
       cur_table = cur_table->next_name_resolution_table) {
    Field *cur_field = find_field_in_table_ref(...);
    if ((cur_field == nullptr && thd->is_error()) || cur_field == WRONG_GRANT)
      return nullptr;                          // 权限错误：错误已报
    if (cur_field) {
      if (db) return cur_field;                // 完全限定名不可能重复 → 直接返回
      if (found) {                             // ★ 第二张表也命中 → 歧义
        my_error(ER_NON_UNIQ_ERROR, MYF(0),
                 table_name ? item->full_name() : name, thd->where);
        return (Field *)nullptr;
      }
      found = cur_field;
    }
  }
  if (found) return found;
  /* 限定名 + 一张表都没搜 → "表"本身不存在 */
  if (table_name && (cur_table == first_table) && ...)
    my_error(ER_UNKNOWN_TABLE, MYF(0), table_name, thd->where);
  else {
    /* ★ 先查权限：无权限就报 access denied，不用 'bad field' 掩盖（也避免泄露列是否存在） */
    if (!first_table || (want_privilege == 0) ||
        !check_column_grant_in_table_ref(thd, first_table, name, length, want_privilege))
      my_error(ER_BAD_FIELD_ERROR, MYF(0), item->full_name(), thd->where);
    else
      found = not_found_field;                 // 延迟报错，交给 fix_fields 继续找
  }
  return found;
```

#### 9.4 第 3 层：`find_field_in_table_ref()`——三种形态 + 两个副作用

```cpp
// [F27] sql/sql_base.cc（结构）
  /* 限定名过滤：用户写了 t.a 而当前表引用的别名对不上 → 跳过 */
  if ((!table_list->nested_join || table_list->field_translation) &&
      table_name && table_name[0] &&
      (my_strcasecmp(table_alias_charset, table_list->alias, table_name) || ...))
    return nullptr;

  if (table_list->field_translation) {           // 形态一：MERGE 视图 / I_S 表
    if ((fld = find_field_in_view(thd, table_list, name, ref, register_tree_change)))
      *actual_table = table_list;
  } else if (!table_list->nested_join) {         // 形态二：真实存储表
    if ((fld = find_field_in_table(table_list->table, name, allow_rowid, field_index_ptr)))
      *actual_table = table_list;
  } else {                                        // 形态三：NATURAL JOIN / USING
    if (table_name && table_name[0]) {            // 限定名 → 绕过合并列，逐成员表查
      for (Table_ref *table : table_list->nested_join->m_tables) { ... }
    } else                                        // 非限定名 → 查合并结果列 join_columns
      fld = find_field_in_natural_join(thd, table_list, name, ref,
                                       register_tree_change, actual_table);
  }

  if (fld) {
    /* ==== 副作用①：列级权限 ==== */
    if (want_privilege) {
      if (fld != view_ref_found) {
        if (check_column_grant_in_table_ref(thd, *actual_table, name, length, want_privilege))
          return WRONG_GRANT;
      } else {
        /* 视图列：权限要顺着 *ref 这棵 Item 树走一遍（可能引用多张基表） */
        Column_privilege_tracker tracker(thd, want_privilege);
        if ((*ref)->walk(&Item::check_column_privileges, enum_walk::PREFIX, (uchar *)thd))
          return WRONG_GRANT;
      }
    }
    /* ==== 副作用②：登记 read_set / write_set ====
       "Get read_set correct for this field so that the handler knows that
        this field is involved in the query and gets retrieved." */
    if (fld == view_ref_found) {
      Mark_field mf(thd->mark_used_columns);
      (*ref)->walk(&Item::mark_field_in_map, enum_walk::SUBQUERY_POSTFIX, (uchar *)&mf);
    } else
      fld->table->mark_column_used(fld, thd->mark_used_columns);
  }
  return fld;
```

#### 9.5 第 4 层：`find_field_in_table()`——真正的"列名 → `Field*`"

```cpp
// [F27] sql/sql_base.cc
/**
  Find field by name in a base table.
  No privileges are checked, and the column is not marked in read_set/write_set.
*/
Field *find_field_in_table(TABLE *table, const char *name, bool allow_rowid,
                           uint *field_index_ptr) {
  Field **field_ptr = nullptr, *field = nullptr;
  if (!(field_ptr = table->field)) return nullptr;
  /* ★ 线性扫描：table->field 是以 nullptr 结尾的 Field* 数组 */
  for (; *field_ptr; ++field_ptr) {
    // NOTE: 这里本该用 strncollsp()，但 Ñ != N 的差异会破坏已有表定义，
    //       只能在将来改系统排序规则时再动 —— 故仍是 my_strcasecmp(system_charset_info)。
    if (!my_strcasecmp(system_charset_info, (*field_ptr)->field_name, name))
      break;
  }
  if (field_ptr && *field_ptr) {
    *field_index_ptr = field_ptr - table->field;   // ★ 指针差 = 列号
    field = *field_ptr;
  } else {
    /* ★ "_rowid" 是唯一"虚拟列名"，只在 allow_rowid（限定名/单表 FROM）时可用 */
    if (!allow_rowid || my_strcasecmp(system_charset_info, name, "_rowid") ||
        table->s->rowid_field_offset == 0)
      return (Field *)nullptr;
    field = table->field[table->s->rowid_field_offset - 1];
  }
  return field;
}
```

#### 9.6 无副作用版 `find_field_in_table_sef()`

```cpp
// [F27] sql/sql_base.cc
/*
  Find field in table, no side effects, only purpose is to check for field
  in table object and get reference to the field if found.
*/
Field *find_field_in_table_sef(TABLE *table, const char *name) {
  Field **field_ptr = nullptr;
  if (!(field_ptr = table->field)) return nullptr;
  for (; *field_ptr; ++field_ptr) {
    if (!my_strcasecmp(system_charset_info, (*field_ptr)->field_name, name))
      break;
  }
  return field_ptr ? *field_ptr : (Field *)nullptr;
}
```

| | `find_field_in_table()` | `find_field_in_table_sef()` |
|---|---|---|
| 出参 | 写 `*field_index_ptr`（列号缓存） | 无出参 |
| 副作用 | 上层会 `mark_column_used()` 登记 read/write_set | **完全无副作用** |
| 权限 | 调用方随后检查 | 不涉及 |
| `_rowid` | 支持 | 不支持 |
| 返回值 | 只能是 `nullptr` 或真 `Field*`（**不返回哨兵**） | 同上 |

★ 谁用它（都是"只想按名字拿 `Field*`、不想污染查询位图"的场景）：

```cpp
// [F36] sql/sql_partition.cc —— KEY 分区字段列表解析
  Field *field = find_field_in_table_sef(table, field_name);
  if (likely(field != nullptr))
    field->set_flag(GET_FIXED_FIELDS_FLAG);   // ★ 只打标记，不进 read_set
  else { my_error(ER_FIELD_NOT_FOUND_PART_ERROR, MYF(0)); ... }

// [F37] sql/histograms/histogram.cc —— ANALYZE TABLE ... UPDATE HISTOGRAM ON 的列名解析
  Field *field = find_field_in_table_sef(tbl, column_name.c_str());
```

#### 9.7 `set_field()` 之后：`field_index` 为什么必须缓存

```cpp
// [F4] sql/item.cc（关键三行）
void Item_field::set_field(Field *field_par) {
  table_ref = field_par->table->pos_in_table_list;
  assert(field_par->field_index() != NO_FIELD_INDEX);
  field_index = field_par->field_index();            // ★ 缓存列号
  field = result_field = field_par;                  // ★ 落点就是 table->field[i]
  set_nullable(field->is_nullable() || field->is_tmp_nullable() ||
               field->table->is_nullable());         // ★ 外连接补 NULL 行在这里生效
  ...
  fixed = true;
}
```

★ **因为 prepared statement 每次执行都会重开表，`Field*` 会失效**——重开后靠下标复位：

```cpp
// [F4] sql/item.cc —— Item_field::bind_fields()
void Item_field::bind_fields() {
  if (!fixed) return;
  assert(field_index != NO_FIELD_INDEX);
  if (field == nullptr) {
    field = result_field = table_ref->table->field[field_index];   // ★ 按下标复位
    ...
```

#### 9.8 三个"找不到/不能用"的错误路径

| 场景 | 错误 | 位置 | 触发条件 |
|---|---|---|---|
| **未知列** | `ER_BAD_FIELD_ERROR`（1054，*"Unknown column ... in ..."*） | `[F27]` `find_field_in_tables()` 末尾 | 所有表都没命中且确认无权限问题；另有 `[F4]` 的隐藏列分支与 `fix_outer_field()` 兜底 |
| **未知表** | `ER_UNKNOWN_TABLE`（1109） | `[F27]` | `table_name` 非空且一张表都没搜过 |
| **歧义列** | `ER_NON_UNIQ_ERROR`（1052，"Column ... is ambiguous"） | `[F27]` 遍历中第二张表也命中 | ★ 不存在 `ER_AMBIGUOUS_FIELD_TERM` 这个**列歧义**错误码——本树里它只用于 `SELECT ... INTO OUTFILE` 的分隔符混淆 warning |
| **权限不足** | `ER_COLUMNACCESS_DENIED_ERROR`（1143） | 返回哨兵 `WRONG_GRANT` | `check_column_grant_in_table_ref()` 失败；CREATE VIEW 场景在 `[F4]` 另报一次 |

#### 9.9 ★ `mark_column_used()`：这一步决定引擎取哪些列

```cpp
// [F31] sql/sql_const.h
enum enum_mark_columns {
  MARK_COLUMNS_NONE, MARK_COLUMNS_READ, MARK_COLUMNS_WRITE, MARK_COLUMNS_TEMP
};

// [F13] sql/table.cc
void TABLE::mark_column_used(Field *field, enum enum_mark_columns mark) {
  switch (mark) {
    case MARK_COLUMNS_NONE:
      if (get_fields_in_item_tree) field->set_flag(GET_FIXED_FIELDS_FLAG);
      break;
    case MARK_COLUMNS_READ: {
      Key_map part_of_key = field->part_of_key;
      bitmap_set_bit(read_set, field->field_index());          // ★ 引擎可见的"要读"
      bitmap_set_bit(&read_set_internal, field->field_index()); // SQL 层内部记账
      part_of_key.merge(field->part_of_prefixkey);
      covering_keys.intersect(part_of_key);   // ★ 只有含该列的索引才可能被判为覆盖索引
      merge_keys.merge(field->part_of_key);
      if (field->is_virtual_gcol()) mark_gcol_in_maps(field);
      break;
    }
    case MARK_COLUMNS_WRITE:
      bitmap_set_bit(write_set, field->field_index());
      if (field->is_gcol()) mark_gcol_in_maps(field);
      break;
    case MARK_COLUMNS_TEMP:                  // filesort 用：置 read_set 但不动 key 集合
      bitmap_set_bit(read_set, field->field_index());
      if (field->is_virtual_gcol()) mark_gcol_in_maps(field);
      break;
  }
}
```

★ `THD::mark_used_columns` 是"解析期副作用的全局旋钮"（`[F32]`）——`fix_fields` 遍历 item 树时由它决定这棵树里出现的所有列被记进哪张位图。解析生成列/默认值/检查约束表达式时会临时切到 `MARK_COLUMNS_NONE`，避免污染表的 `read_set`。

**★ 闭环**（这是本章与引擎的接口）：`find_field_in_table_ref()` → `mark_column_used()` → `read_set` 置位 → 建模板时 `[F35] build_template_needs_field()` 用 `bitmap_is_set(table->read_set, i)` 判断 → 决定 `mysql_row_templ_t` 里有没有第 i 列。**SQL 层"提到了某列"就是靠这条位图链传给引擎的**（只写 WHERE 没写 SELECT 的列也要登记，否则引擎不会回传该列供过滤）。

#### 9.10 三个特殊场景

**① 视图 / derived table：列被"替换"而不是"绑定"**——视图列是**已 fix 过的 Item 表达式**（`Field_translator{Item *item; const char *name;}`），命中后 `*ref = Item_view_ref` 并返回哨兵 `view_ref_found`；上层语义是"**我没给你 `Field*`，我改写了你这棵 item 树**"，所以 `fix_fields()` 看到 `view_ref_found` 直接 `return false`，不再 `set_field()`。副作用登记也随之改成 `walk(&Item::mark_field_in_map, SUBQUERY_POSTFIX)`——把它触及的**所有基表列**都登记（否则视图列取值会缺数据）。

**② NATURAL JOIN / USING：合并列**——★ **`Field_merge` 在 8.0 已不存在**，现在是 `Natural_join_column`（`[F5]`）：

```cpp
class Natural_join_column {
 public:
  Field_translator *view_field;   /* 来自 merge view */
  Item_field *table_field;        /* 来自基表 */
  Table_ref *table_ref;           /* ★ 列真正所属的表引用 */
  bool is_common;                 /* 是否两操作数共有 */
};
```

非限定名 → 只在合并列里搜（所以 `SELECT a FROM t1 NATURAL JOIN t2` 只返回一份）；限定名（`t1.a`）→ 绕过合并列直接到成员表。命中后 `*actual_table = nj_col->table_ref`（合并列本身没有 `TABLE`）。

**③ 外连接补 NULL 行与 `is_nullable()`**：

```cpp
// [F34] sql/sql_resolver.cc —— 沿 join 树向内表/嵌套传播"可空"
void propagate_nullability(mem_root_deque<Table_ref *> *tables, bool nullable) {
  for (Table_ref *tr : *tables) {
    if (tr->table && !tr->table->is_nullable() && (nullable || tr->outer_join))
      tr->table->set_nullable();
    if (tr->nested_join == nullptr) continue;
    propagate_nullability(&tr->nested_join->m_tables, nullable || tr->outer_join);
  }
}
```

★ `TABLE::nullable` 是**表级**属性，被 `Item_field::set_field()` 吸收（`set_nullable(... || field->table->is_nullable())`）。执行期 `Item_field::is_null()` 直接返回 `field->is_null()`——外连接补的 NULL 行由 `table->set_null_row()` 让所有字段读出来是 NULL。**顺序很重要**：`propagate_nullability()` 必须发生在 `fix_fields()` 之前，否则补 NULL 行的可能性不会写进 `maybe_null`，影响外连接优化的正确性。

---

## ★ 本机制里的工程实现技法

八条贯穿 `Field` 全族的技法，每一条都是"**热路径上的一次取舍**"：

1. **★ 一次分配、整体平移**（`move_field_offset()`）：模板 Field 的 `ptr` 指向 `default_values`，克隆到每个 `TABLE` 时只做一次 `record[0] - default_values` 的减法，把 `ptr` 与 `m_null_ptr` 一起平移——**避免逐字段重算偏移**。
2. **★ `val_str(buf1, buf2)` 双缓冲技巧**：值为字符串时用 buf2（直接指向 `ptr`，零拷贝零分配），需转换时用 buf1（调用方已分配的缓冲）。注释直言目的是 *"to decrease a number of malloc calls"*。代价是这是**纯性能约定**，实现违反会写坏调用方栈上的 buf2。
3. **★ 哨兵值复用**（`UNIV_SQL_NULL = UINT32_UNDEFINED`）：用长度字段的**不可能取值**当 NULL 标记，省掉一个 bool；代价是每个读 `len` 的地方都要记得先查哨兵。
4. **★ 位域打包与"下沉"**：`prtype` 一个 32 位整数塞进"类型 + charset-collation + NOT NULL + 无符号 + 是否虚拟列"；`mbminmaxlen` 是把"字符对齐所需的最小/最大字节"下沉进 `prtype`——因为写方向要用它、比较方向不用（见理论基础）。
5. **★ 缓存一次除法**（`Field_str::char_length_cache`）：`char_length() = field_length / mbmaxlen`，一次除法在 `make_sort_key()` 里竟贵得不成比例，于是改长度时同步刷新缓存。
6. **★ 黑名单式安全判定**（`fields_are_memcpyable()`）：不用白名单，而是**枚举所有不安全情形后返回 true**——JSON/GEOMETRY/VARCHAR/ENUM/SET/BIT/BLOB + 字符集不同 + 长度不同 + 无符号性不同 + 字节序不同 + sql_mode 严格。**凡是"内存布局 ≠ 值语义"的一律不许 memcpy**。
7. **★ 魔数防呆**（`magic_n`/`magic_n2`、fetch cache 首尾的 `ROW_PREBUILT_FETCH_MAGIC_N`）：把"悬垂"与"越界/错位写"两类错误挡在断言上（poka-yoke 思想，同 `THD_SENTRY_MAGIC` / `MEM_BLOCK_MAGIC_N`）。
8. **★ 延迟决策 + 事后修正**（`build_template` 的第 4 个循环）：建模板时同时写好 `clust_rec_field_no` 与 `rec_field_no`，最终是否回表在最后统一决定——**避免在循环内做当时无法做的判断**。

9. **★ "gap 驱动"的重写策略**（`pack()`，见 3.11）：**只在 record 布局与目标布局存在 gap 时才重写**，而 gap 只有三类——① 尾部填充/长度信息外置（`Field_string` 尾空格、`Field_blob` 外挂指针、`Field_bit` 零头位在 NULL 区）；② 字节序依赖引擎（整数族、`Field_real`、老式 `Field_timestamp`/`Field_datetime`）；③ 前缀宽度依赖声明长度（`Field_string > 255` 用 2 字节、`Field_blob` 1/2/3/4）。⇒ 想知道"这个类型为什么重写了 `pack`、那个没有"，只需问一句"**gap 在哪**"——新式时间类型与 `Field_new_decimal` 没有 gap，所以一行都不用写。
10. **★ 两阶段布局 + assert 对账**：`find_record_length()` 先**算尺寸**（`null_bytes` / `reclength` / `last_null_bit_pos`），`fill_columns_from_dd()` 后**摆位**——这是同一算法的**两次独立实现**，靠三个 `assert` 保证它们对 DD 的解读完全一致。任何一边漏掉 `is_se_hidden()` 跳过或漏掉 BIT 余位，assert 立刻失败。
11. **★ 有损编码 + 启发式还原**（`auto_flags`）：DD 用一个 `default_option` 文本字段承载两类完全不同的语义（`CURRENT_TIMESTAMP` 与 `DEFAULT (expr)`），server 侧靠**前 17 字符前缀比较**区分，并用 `assert` 兜底。代价是它依赖上游写入侧的纪律。
12. **★ 工厂集中化**：所有 `new Field_xxx` 都收敛在 `make_field()` 的**一个 switch** 里（第八章）。新增一个列类型只需改这一处（外加 `calc_pack_length()`、`field_types_result_type[]`、引擎侧 `get_innobase_type_from_mysql_type()` 等少数几处，见 Misc 的扩展点清单）。
13. **★ 不对称是刻意的设计**（`pack` / `unpack`）：`pack()` 用**本地**声明宽度决定前缀，`unpack()` 必须用**对端（主库）**宽度才能读前缀——因为要处理主备异构，二者天然不构成对称逆运算（见 3.12）。

> **这些技法的共同代价**：它们几乎都是"**把不变式从代码挪进约定**"——`val_str` 的双缓冲、哨兵值、`param_data`、`make_hash_key` 都靠调用方/实现对约定的自觉遵守。一旦违反，错误通常表现为**内存踩坏而非逻辑错误**；debug 构建下由 `ASSERT_COLUMN_MARKED_FOR_READ/WRITE`、各类 `assert(0)` 兜底（如 `val_*_temporal` 基类、几种 `MYSQL_TYPE_*` 落到 `make_field` 的 default），release 下则直接是未定义行为。

**一个贯穿全族的设计模式**：`Field` 是"**几何 + 语义**"的合体——几何（`ptr` / `pack_length()` / `null_bit`）面向引擎，语义（`val_*` / `store*`）面向 `Item`；两者只在 `build_template` 那一刻相遇，之后各行其道。这是 server 与引擎能长期解耦的关键。

---

## 可观测性

`Field` 本身没有运行期计数器（它只是数据结构），但可以从四处观察它的行为：

1. **★ debug 断言 `ASSERT_COLUMN_MARKED_FOR_READ` / `_WRITE`**：`val_*()` 与 `store*()` 的第一句。debug 构建下任何"读了没标记的列 / 写了没标记的列"都会立即断言失败——这是 `read_set`/`write_set` 与真实访问不一致的头号探测器。
2. **`SHOW CREATE TABLE` / `information_schema.COLUMNS`**：间接观察 `Field` 的元数据视图——`sql_type()`（ZEROFILL 时显示宽度）、`has_charset()`（决定补不补 `CHARACTER SET` 子句）、`is_hidden()`（INVISIBLE 列不出现在 `SELECT *` 但出现在 `SHOW CREATE TABLE`）。
3. **DD 表 `mysql.columns`**：`default_value()`（二进制镜像）与 `default_value_utf8()`（文本）可对照看"默认值到底存成了什么"；`hidden()` 直接暴露隐藏列类型（函数索引承载列是 `HT_HIDDEN_SQL`）。
4. **`mysql.columns` 的 `options()` / `se_private_data`**：`treat_bit_as_char`、`geom_type`、`srs_id`，以及 **instant 相关的私有键**（`DD_INSTANT_*`，由 InnoDB 读写）——想确认"这列是不是 instant 加的"只能看这里，**server 层无从判断**。

★ 若要在 debug 下看到隐藏列（含 SE 隐藏列），可用调试选项 `show_hidden_columns`（`Field::is_hidden()` 里的 `DBUG_EVALUATE_IF`）。

---

## Misc

### 扩展点：加一个新列类型要改哪几处

| # | 位置 | 要做什么 |
|---|---|---|
| 1 | `enum_field_types` | 加类型码（协议可见的新类型用 <25 或 243+；只在内部的加 "Internal to MySQL" 注释） |
| 2 | 新 `Field_xxx` 子类 | 实现全部纯虚：`type()`/`real_type()`/`result_type()`/`pack_length()`/`key_type()`/`sql_type()`/`store()` 四重载/`val_*()` 四个/`cmp()`/`clone()`/`make_sort_key()`/`get_key_image()` |
| 3 | `make_field()` 的 switch | 加 `case MYSQL_TYPE_XXX: return new (mem_root) Field_xxx(...)`（注意 BLOB 系传的是"长度字段字节数"而非总长） |
| 4 | `calc_pack_length()` | 加该类型的字节数计算（含 fsp 时用 `> MAX_*_WIDTH` 判别 + 反解） |
| 5 | `field_types_result_type[]` | 加映射（否则 `result_type()` 查表得到 `INVALID_RESULT`） |
| 6 | `create_tmp_field_from_item()` / `tmp_table_field_from_field_type()` | 加"由 Item 造该类型列"的分支（否则表达式物化会退化成字符串） |
| 7 | 引擎侧 `get_innobase_type_from_mysql_type()` | 加 `MYSQL_TYPE_XXX → mtype` 的映射（**属 InnoDB 文档**） |
| 8 | `real_type_to_type()` | 若新增的是"内部物理类型"，要在这里归一回逻辑类型 |

### 坑与已知缺陷（逐条都有源码依据）

1. ★ **`type()` ≠ `real_type()`**：ENUM/SET 报 `STRING`、时态新代报 `DATETIME2` 等——**判断物理类型必须用 `real_type()`**。
2. ★ **`pack_length()` ≠ `pack_length_in_rec()`**：`Field_bit` 是唯一差异（`(n+7)/8` vs `n/8`），因为零头位在 NULL 区。
3. ★ **`val_int()` 与 `val_str()` 语义不一致**：ENUM 的 `val_int()` 是 1-based 下标、`val_str()` 是名字，`0` 同时表示"非法"和"空串"。
4. ★ **`cmp()` 不是 memcmp**：字符串走 `strnncollsp`（collation 权重）；而 `Field_json` 继承来的 `cmp()` 反而是**字节流二进制比较**（语义比较要走 `cmp_binary()` / `make_sort_key()`）。
5. ★ **`make_sort_key()` 与 `get_key_image()` 绝不能混用**：前者做 collation 变换 + 整数翻符号位（8.0.39 只剩复制在用），后者是原字节 + 长度头（给引擎比较器）。
6. ★ **NULL 位的起点依赖 `HA_OPTION_PACK_RECORD`**：packed（COMPACT/DYNAMIC）从 bit0 起（前 **8** 列在第 0 字节），非 packed 时 bit0 是"row not deleted"（前 **7** 列）。
7. ★ **`field_length` 是字节数不是字符数**：`varchar(12)` utf8mb4 下 = 48；字符数要 `char_length()`。
8. ★ **VARCHAR 长度头只看总字节数 < 256**（`HA_VARCHAR_PACKLENGTH`），与 `mbmaxlen` 无关；**键里恒 2 字节**。
9. ★ **8.0.39 不存在 `set_record_ptr()` 与 `Field::maybe_null()`**——真正在做的是 `move_field()`/`move_field_offset()`；`maybe_null()` 只存在于 `Item`/`SEL_ARG`。
10. ★ **`Field::binary()` 有两层语义**：基类常量 `true`（"本类族无字符语义"）vs `Field_str::binary()`（`charset() == &my_charset_bin`）——判据永远是后者。
11. ★ **VARCHAR 的 `length_bytes` 特例**：前缀键段可能出现 `field_length == 1` 而 `length_bytes == 2`，取长度头必须用 `get_length_bytes()`，不能用宏推。
12. **虚拟列的数组下标顺序与内存 offset 顺序不一致**（下标按 DDL 顺序，`offset` 按"物理列 → 虚拟列"两遍布局）。

---

## 参考

### 知识库内相关文档

| 主题 | 归属 |
|---|---|
| InnoDB 侧的列类型（`dtype_t` / prtype 位域 / mtype）与行内记录布局 | [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md)、[`../innodb/physical/record.md`](../innodb/physical/record.md) |
| 引擎键格式（`key_copy` / `key_restore` / `get_key_image`） | [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md)「键写（server 侧）」 |
| 模板构建（`build_template` 族 / `mysql_row_templ_t` / `row_prebuilt_t`） | [`handler.md`](handler.md)「row_prebuilt_t」大节 |
| 读路径（取行、字段级 switch、end-range、ICP） | [`../innodb/row_search.md`](../innodb/row_search.md) |
| 字符集与 collation（sort_order / strnxfrm / mbminmaxlen） | [`../server/infra/charset.md`](../server/infra/charset.md) |
| instant ADD/DROP、虚拟列的**引擎侧**实现 | InnoDB 文档（`dict0inst`、`rec_offs`、`UNIV_SQL_ADD_COL_DEFAULT`） |
| 时态类型的 SQL 语义与 sql_mode | [`../server/datatype/`](../server/datatype/) 与时间函数相关文档 |

### 外部参考

> ⚠️ 待补：按知识库规范，动笔前应深度检索 **MySQL 8.0 Reference Manual**（Data Types / JSON / Generated Columns / Instant ADD COLUMN 各章）与相关 **WorkLog**（如 WL#6255 Instant ADD COLUMN、WL#8114 Generated Columns、WL#8963 时态类型精度），并回源码核实后补入本节。

- MySQL 8.0 Reference Manual — *Data Types* / *The JSON Data Type* / *CREATE TABLE and Generated Columns* / *ALTER TABLE … ALGORITHM=INSTANT*（**尚未检索核实**）
- InnoDB 数据字典与 instant 列的内部文档（`storage/innobase/dict/`）（**属 InnoDB 文档，本篇未展开**）
