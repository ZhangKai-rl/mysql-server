# 21 Collation 与类型如何影响索引可用性

> 基于 MySQL 8.0.39。本篇讲优化器视角下的**两件容易被混为一谈的事**：字符集/collation 聚合（`DTCollation` / `agg_item_charsets`）与比较类型聚合（`agg_cmp_type` / `Field::field_type_merge`），以及它们各自对"这个条件还能不能走索引"的影响。
>
> **边界**：字符集本身（编码、转换规则、`mbmaxlen`）见 [`../../infra/encoding.md`](../../infra/encoding.md)；Item 类型体系见 [`../13_item_expression.md`](../13_item_expression.md)；代价与统计见 [`01_cost_model.md`](01_cost_model.md)；range 优化器的区间构造见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)。本篇只讲"**为什么我的条件建不出 range / 用不上索引**"这条因果链。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、DTCollation 与 Derivation 强度体系](#一dtcollation-与-derivation-强度体系)
  - [二、agg_item_charsets：聚合 + 就地永久替换](#二agg_item_charsets聚合--就地永久替换)
  - [三、类型聚合：索引失效的真正元凶](#三类型聚合索引失效的真正元凶)
  - [四、NO PAD vs PAD SPACE：看不见的语义差异](#四no-pad-vs-pad-space看不见的语义差异)
  - [五、索引长度限制与前缀索引](#五索引长度限制与前缀索引)
  - [六、排查清单](#六排查清单)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 两条独立的链，别混为一谈

很多人把"隐式类型转换导致索引失效"笼统归因于"字符集"。实际上源码里有**两条完全独立**的聚合链：

| 链 | 发生位置 | 聚合什么 | 会不会导致索引失效 |
|---|---|---|---|
| **字符集聚合** | `agg_item_charsets` → `DTCollation::aggregate` + `agg_item_set_converter` | 比较用哪个 collation | **通常不会**（原因见第二章的 `only_consts`） |
| **类型聚合** | `agg_cmp_type` → `item_cmp_type`；`agg_field_type` → `Field::field_type_merge` | 比较在哪个类型域进行 | **会**（这才是元凶） |

★ 本篇最重要的结论：**让索引失效的通常不是字符集，而是类型域的提升**。`WHERE varchar_col = 123` 走不了索引，是因为类型聚合把比较提升到了 `REAL_RESULT`，而不是因为字符集不同。

### 一个反直觉的开场

```sql
-- ① 字符集不一致，但索引仍然可用
SET NAMES latin1;
SELECT * FROM t1 WHERE utf8mb4_col = 'abc';
-- 常量被转换到字段的字符集，索引可用

-- ② 类型不一致，索引失效
SELECT * FROM t1 WHERE varchar_col = 123;
-- 比较在 REAL 域进行，索引失效

-- ③ 显式 COLLATE 反而可能不影响索引（取决于是否触发转换器）
SELECT * FROM t1 WHERE varchar_col COLLATE utf8mb4_bin = 'abc';
```

三条里最容易被误判的是第一条：字符集不同**不会**自动导致索引失效，因为比较场景只给**常量**加转换器，字段本身不会被包上 `CONVERT`。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | 引入 coercibility（derivation）体系，7 档 |
| 8.0.0 | **默认字符集改 utf8mb4，默认 collation 改 `utf8mb4_0900_ai_ci`**（UCA 9.0.0，且是 **NO PAD**） |
| 8.0.x | `utf8mb3` 与 `utf8mb4` 分离，`utf8` 成为 `utf8mb3` 的别名（已废弃） |
| 8.0.x | `PAD_ATTRIBUTE` 进入 `information_schema.collations`，NO PAD 成为新默认的行为 |

★ 8.0 的这次默认 collation 变更是**破坏性的**：从 `latin1_swedish_ci`（PAD SPACE）到 `utf8mb4_0900_ai_ci`（NO PAD），会改变 CHAR 列比较、去重、去尾空格的行为。

---

## 理论基础

### Coercibility（可强制性 / derivation）

SQL 标准里有 coercibility 概念：表达式的 collation 不是平权的，而是有**强弱**之分。标准只定义了三档（explicit / implicit / none），MySQL 扩展成了七档（源码注释明说 *"MySQL supports more coercibility types than the SQL standard"*）。

之所以需要强弱：两个不同 collation 的字符串比较时，必须选一个"胜出"的 collation 来做比较。选谁？按强弱规则。

### repertoire（字符曲目）

比 collation 更高一层的概念：某个字符串的**字符集范围**。

| repertoire | 含义 |
|---|---|
| `MY_REPERTOIRE_ASCII` | 该值只可能包含 ASCII 字符（如数字常量、纯 ASCII 字面量） |
| `MY_REPERTOIRE_EXTENDED` | 可能包含非 ASCII |
| `MY_REPERTOIRE_UNICODE30` | Unicode 3.0 |

repertoire 让优化器能跳过不必要的转换：一个纯 ASCII 的数字常量与目标 ASCII 兼容的字符集比较时**不需要**转换（源码注释给了 `datetime_field = '2010-01-01'` 的例子）。

### 为什么类型提升会毁掉索引

索引是按**字段原生类型的排序规则**组织的有序结构。要利用它做区间扫描，比较必须在**同一个排序序**下进行：

- 字符串按 collation 排序
- 数字按数值大小排序

这两者**不同**：`'10' < '9'`（字符串）但 `10 > 9`（数字）。

所以一旦类型聚合把 `varchar_col = 123` 提升为 REAL 域比较，字段必须被"转成 double 再比较"，而索引的字符串排序序对 double 比较毫无意义——range 优化器无法构造区间。

### 他库对比

| 库 | 隐式转换与索引 |
|---|---|
| **MySQL** | 隐式转换宽松（字符串↔数字），转换发生在字段侧时索引失效 |
| **PostgreSQL** | **严格类型**：`varchar_col = 123` 直接报类型错误，不存在"隐式转换导致索引失效" |
| **Oracle** | 隐式转换，但 Oracle 优化器能在某些情况下把转换移到常量侧 |
| **SQL Server** | 有 `CONVERT_IMPLICIT`，同样会导致索引 seek 变 scan，但会明确出现在执行计划里 |

MySQL 的特色问题在于：**它既允许隐式转换，又不把转换显式显示在 EXPLAIN 里**，导致问题难以察觉。

---

## 核心实现

### 一、DTCollation 与 Derivation 强度体系

> `sql/item.cc` 的 `DTCollation::aggregate()`。

#### 1.1 七档 derivation（源码注释原文）

```
DERIVATION_EXPLICIT  - an explicitly written COLLATE clause
DERIVATION_NONE      - a mix of two different collations
DERIVATION_IMPLICIT  - a column
DERIVATION_SYSCONST  - a system function
DERIVATION_COERCIBLE - a string constant
DERIVATION_NUMERIC   - a numeric constant coerced to a character string
DERIVATION_IGNORABLE - a NULL value.

These are ordered by strength from highest (DERIVATION_EXPLICIT) to
lowest (DERIVATION_IGNORABLE), and a low enum value means higher strength.
```

★ 注意"低枚举值 = 高强度"，读代码时容易搞反。强度序列：

```
EXPLICIT (COLLATE 子句)  >  IMPLICIT (列)  >  SYSCONST  >  COERCIBLE (字符串常量)  >  NUMERIC  >  IGNORABLE (NULL)
```

`DERIVATION_NONE` 是特殊的"混合态"：它表示"两边 collation 不同且强度相同，需要第三方来裁决"。

#### 1.2 aggregate 的核心规则

```cpp
bool DTCollation::aggregate(DTCollation &dt, uint flags) {
  // 两个 EXPLICIT 且 collation 不同 → 直接错误
  if (collation != dt.collation && derivation == DERIVATION_EXPLICIT &&
      dt.derivation == DERIVATION_EXPLICIT) {
    return true;
  }
  if (!my_charset_same(collation, dt.collation)) {
    // 字符集不同：按 flag 决定能否转换
    //   MY_COLL_ALLOW_SUPERSET_CONV  - 允许转到超集
    //   MY_COLL_ALLOW_COERCIBLE_CONV - 允许转换 coercible 值
    ...
  }
  ...
}
```

三条规则（源码注释列出）：

1. **collation 相同** → 选它，derivation 取更强的
2. **collation 不同**：
   - 两个 EXPLICIT → **报错**（`CONCAT(a COLLATE x, b COLLATE y)` 是不合法的）
   - 否则**强度小的一方胜出**（列强于常量，COLLATE 强于列）
3. **强度相同但 collation 不同** → `DERIVATION_NONE`，等第三个参数来裁决（注释给了 `CONCAT` 三参数的例子）

#### 1.3 聚合失败的两个错误

| 错误 | 触发 |
|---|---|
| `ER_CANT_AGGREGATE_2COLLATIONS` | 两个参数无法聚合（`Illegal mix of collations`） |
| `ER_CANT_AGGREGATE_NCOLLATIONS` | 三个及以上参数无法聚合 |

### 二、agg_item_charsets：聚合 + 就地永久替换

> `sql/item.cc`。

#### 2.1 两步

```cpp
bool agg_item_charsets(DTCollation &coll, const char *fname, Item **args,
                       uint nargs, uint flags, int item_sep, bool only_consts) {
  if (agg_item_collations(coll, fname, args, nargs, flags, item_sep))
    return true;                                  // ① 决定用哪个 collation
  return agg_item_set_converter(coll, fname, args, nargs, flags, item_sep,
                                only_consts);     // ② 给需要转换的参数套转换器
}
```

#### 2.2 ★ `only_consts`：比较场景只转换常量

`agg_item_set_converter` 的循环开头：

```cpp
for (i = 0, arg = args; i < nargs; i++, arg += item_sep) {
  size_t dummy_offset;
  // If told so (from comparison code), only add converter for const values.
  if (only_consts && !(*arg)->const_item()) continue;
  if (!String::needs_conversion(1, (*arg)->collation.collation,
                                coll.collation, &dummy_offset))
    continue;
  ...
```

★ **这就是"字符集不同通常不会让索引失效"的源码答案**：比较调用链（`agg_arg_charsets_for_comparison`）会传 `only_consts=true`，于是**只有常量会被套上转换器，字段永远不会被包**。

推论：`WHERE utf8mb4_col = latin1常量` 的处理是"把常量转换到字段的字符集"，字段仍是裸 `Item_field`，range 优化器照常工作。

#### 2.3 转换器的三种跳过情形

| 条件 | 说明 |
|---|---|
| `only_consts && !const_item()` | 非比较场景下的字段，或不转换非常量 |
| `!String::needs_conversion(...)` | 不需要转换（字符集兼容） |
| `derivation == DERIVATION_NUMERIC && repertoire == ASCII && 两边都 ASCII-based` | 源码注释的例子：`SELECT * FROM t1 WHERE datetime_field = '2010-01-01'` **不会**被改写成 `CONVERT(datetime_field USING cs) = '2010-01-01'` |

第三条是专门为 `datetime/numeric` 这类"本质是 ASCII 的值"开的绿灯，注释还留了 TODO：希望扩展到所有 ASCII repertoire 的值。

#### 2.4 就地永久替换

```cpp
Item *conv = (*arg)->safe_charset_converter(thd, coll.collation);
if (conv == nullptr && (*arg)->collation.repertoire == MY_REPERTOIRE_ASCII)
  conv = new Item_func_conv_charset(thd, *arg, coll.collation, true);
...
// Update the Item pointer in-place
if (thd->lex->is_exec_started())
  thd->change_item_tree(arg, conv);
else
  *arg = conv;

(*arg)->disable_constant_propagation(nullptr);
```

三点：

- 转换器优先用 `safe_charset_converter`（能安全转换则用），否则退化为 `Item_func_conv_charset`（可能报错）
- **就地替换 Item 指针**是永久性改写（不是临时视图），并且执行期用 `thd->change_item_tree()` 保证 PS 复用时能正确回滚
- 转换后**禁用常量传播**（`disable_constant_propagation`）——因为转换后的常量不再适合做常量传播

★ 这就是为什么 `EXPLAIN` 里有时看不到 `CONVERT`：它发生在 prepare 期，已经融进了 Item 树。

### 三、类型聚合：索引失效的真正元凶

> `sql/item_cmpfunc.cc`。

#### 3.1 比较类型的聚合

```cpp
static Item_result agg_cmp_type(Item **items, uint nitems) {
  Item_result type = items[0]->result_type();
  for (uint i = 1; i < nitems; i++) {
    type = item_cmp_type(type, items[i]->result_type());
  }
  return type;
}
```

`item_cmp_type(STRING_RESULT, INT_RESULT)` 的结果是 **`REAL_RESULT`**（MySQL 用 double 作为字符串与数字的公共比较域）。

#### 3.2 为什么这毁掉索引

`arg_comparator` 会根据聚合出的比较类型选择取值函数：

| 比较类型 | 字段侧取值 | 索引可用性 |
|---|---|---|
| `STRING_RESULT` | `val_str()`，按 collation 比较 | ✅ 可用（与索引排序序一致） |
| `INT_RESULT` | `val_int()` | ✅ 可用 |
| `REAL_RESULT`（字段是字符串时） | `val_real()`，**字段被转成 double** | ❌ **不可用** |

关键在最后一行：当字段是字符串而比较域是 REAL 时，字段值必须先转 double 才能比较。索引是按字符串序组织的，无法用于 double 域的区间扫描。

#### 3.3 字段类型的聚合：`Field::field_type_merge`

另外一个用途：多参数函数（如 `CASE`、`COALESCE`、UNION 的列）的结果字段类型。

```cpp
enum_field_types agg_field_type(Item **items, uint nitems) {
  assert(nitems > 0 && items[0]->result_type() != ROW_RESULT);
  enum_field_types res = items[0]->data_type();
  for (uint i = 1; i < nitems; i++)
    res = Field::field_type_merge(res, items[i]->data_type());
  return real_type_to_type(res);
}
```

★ 这解释了 UNION / 派生表的一个常见坑：**结果列的类型是聚合出来的**，可能与任一 operand 都不同（例如 `VARCHAR` 与 `INT` 聚合出 `VARCHAR`，但宽度取最大）。若聚合后的类型与外层比较的类型不一致，又是一次隐式转换。

#### 3.4 一个可操作的结论

| 写法 | 索引 |
|---|---|
| `WHERE varchar_col = '123'` | ✅ 字符串域比较，可用 |
| `WHERE varchar_col = 123` | ❌ 提升到 REAL 域 |
| `WHERE int_col = '123'` | ✅ 常量被转成 int，字段不变 |
| `WHERE utf8_col = latin1_string` | ✅ 常量被转换，字段不变 |
| `WHERE varchar_col COLLATE utf8mb4_bin = 'abc'` | 取决于是否触发转换器（见 2.2） |

**规律**：只要转换发生在**常量侧**，索引就还在；转换一旦落到**字段侧**，索引就没了。

### 四、NO PAD vs PAD SPACE：看不见的语义差异

#### 4.1 两者的定义

| | 尾随空格 |
|---|---|
| **PAD SPACE** | 比较时忽略尾随空格（`'abc' = 'abc '` 为真） |
| **NO PAD** | 尾随空格是有效字符（`'abc' != 'abc '`） |

8.0 的默认 `utf8mb4_0900_ai_ci` 是 **NO PAD**，而老的 `utf8mb4_general_ci` / `latin1_swedish_ci` 是 **PAD SPACE**。

#### 4.2 对 CHAR 列的影响

`Field_string`（CHAR）在去尾空格时的处理（`sql/field.cc`）：

```cpp
if (field_charset->pad_attribute == NO_PAD && ...) {
  /* Our CHAR default behavior is to strip spaces. For PAD SPACE collations,
     this doesn't matter, for but NO PAD, we need to do it ourselves here. */
}
```

即：**CHAR 列默认会去掉尾随空格存储**。在 PAD SPACE 下这不影响比较（反正比较时忽略），但在 **NO PAD 下这会导致"表里的值"与"索引里的值"不一致**。

#### 4.3 ★ 优化器因此必须保守

`sql/sql_optimizer.cc` 里判断"等值谓词是否被索引完全覆盖"（用于删冗余谓词）时：

```cpp
if (field->type() == MYSQL_TYPE_STRING &&
    field->charset()->pad_attribute == NO_PAD) {
  /*
    For "NO PAD" collations on CHAR columns, this function must return
    false, because removal of trailing space in CHAR columns makes the
    table value and the index value compare differently. As the column
    strips trailing spaces, it can return false candidates. Further
    comparison of the actual table values is required.
   */
  return false;
}
```

★ 这段注释点出了一个 subtle 的正确性问题：**NO PAD + CHAR 时，索引查找可能返回"假候选行"**（索引里有尾随空格的版本，表里却存着去掉空格的版本），所以优化器不能假设"索引查找结果 = 最终结果"，必须保留进一步的表值比较。

同类保守判定还有（同一函数的注释）：

- **BINARY/VARBINARY**：等值查找时常量不会被填充到字段全长，ref access 可能返回多余行（源码留了 `@todo`）
- **FLOAT**：`Field_float::store()` 把 double 转 float 后比较，与"字段转 double 后比较"的结果可能不同

这三条都是"**索引的排序/比较语义与表值的比较语义不完全一致**"的例子，优化器必须在这类情形下放弃某些优化。

#### 4.4 对 filesort 与去重的影响

`sql/filesort.cc` 里为 PAD SPACE 与 NO PAD 选择了不同的 `strnxfrm` 标志：

```cpp
(field_charset->pad_attribute == NO_PAD) ? 0 : MY_STRXFRM_PAD_TO_MAXLEN;
```

也就是说排序时的归一化方式不同。这会影响到"索引顺序能否直接满足 ORDER BY"的判定。

### 五、索引长度限制与前缀索引

| 限制 | 值 | 影响 |
|---|---|---|
| InnoDB 单列索引最大字节 | 767（老格式）/ **3072**（DYNAMIC/COMPRESSED） | `utf8mb4` 每字符 4 字节，所以 `VARCHAR(768)+` 建全列索引可能超限，必须前缀索引 |
| 前缀索引 | `col(N)` | **只索引前 N 个字符**，会导致"索引查找返回候选行，还需回表精确比较" |

★ 前缀索引对优化器的影响与第四章的 NO PAD 情形同构：**索引不再能完全决定匹配结果**。这会影响：

- `ref` 访问的行数估算（前缀选择性 ≠ 全列选择性）
- 覆盖索引的判定（前缀索引不能覆盖 `SELECT col`）
- "索引是否足够"的保守判定

代价模型层面，`rec_per_key` 反映的是前缀的选择性，因此估算通常偏低（真实匹配更少）。

### 六、排查清单

"我的条件为什么没走索引"按此顺序排查：

```
1. 比较两边类型是否一致？
   → 字段字符串、常量数字 ⇒ agg_cmp_type 提升到 REAL ⇒ 索引失效
   → 修法：给常量加引号（转成字符串域）

2. 字段是否被表达式包裹？
   → 函数、算术、显式 COLLATE（触发转换器时）⇒ 裸 Item_field 消失
   → 修法：改写 SQL 或建函数索引（见 13_functional_mv_index.md）

3. 是否前缀索引 / NO PAD + CHAR？
   → 索引可能返回假候选，优化器保守
   → 影响的是"索引是否够用"，而非"能否用索引"

4. 是否外连接 / OR / 隐式分组等结构限制？
   → 见 logical/ 各篇
```

---

## ★ 本机制里的工程实现技法

### 1. 用"强度枚举"表达优先级，且低值=强

Derivation 枚举故意让"更强的 = 更小的数值"，这样比较大小就能直接判强弱（`derivation < dt.derivation` 即"我更强"）。这是 C 代码里常见的技巧，但读起来反直觉（源码专门加了注释说明）。

### 2. `only_consts`：用参数区分"比较"与"取值"两种语义

同一个 `agg_item_charsets` 要服务两种场景：

- **比较**（`WHERE a = b`）：只转换常量，保护字段（保护索引可用性）
- **取值/拼接**（`CONCAT`、`UNION` 的列）：两边都可以转换

用一个 bool 参数区分，而不是写两个函数。代价是调用方必须清楚自己在哪种场景。

### 3. 就地永久替换 + `change_item_tree`

`agg_item_set_converter` 直接改 Item 指针，执行已开始时用 `thd->change_item_tree()`（保证 PS 复用能回滚），否则直接赋值。这是 MySQL 里"**prepare 期改写 Item 树**"的标准手法，与 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) 里的各类 transform 同源。

### 4. repertoire 作为"免转换"的证明

`MY_REPERTOIRE_ASCII` 让优化器能证明"这个值是纯 ASCII，转不转换都一样"，从而跳过转换器。这是一个**用类型系统携带的元数据换取运行时开销**的经典做法。

### 5. 保守判定写成显式 early return

NO PAD / BINARY / FLOAT 三条"索引不可全信"的情形，都被写成 `return false` 的早退分支，且每条都配了详细注释说明原因。这与"优化器假设索引总是可信"的默认形成对照——**默认乐观，特例显式悲观**。

---

## 可观测性

### 怎么确认比较类型被提升了

```sql
-- EXPLAIN 的 key 字段为空 + 全表扫，是类型提升的典型表现
EXPLAIN SELECT * FROM t WHERE varchar_col = 123;

-- 用 warnings 看是否有隐式转换提示
EXPLAIN FORMAT=JSON SELECT * FROM t WHERE varchar_col = 123;
```

`EXPLAIN FORMAT=JSON` 的 `attached_condition` 里若出现 `cast`/`convert`，说明发生了转换。

### 怎么确认字符集转换发生了

- 聚合失败会报 `ER_CANT_AGGREGATE_2COLLATIONS`（`Illegal mix of collations`）——这是唯一"显式暴露"的情形
- 成功的转换**不出现在 EXPLAIN**，因为它在 prepare 期就融进了 Item 树

### 排查 NO PAD

```sql
SELECT COLLATION_NAME, PAD_ATTRIBUTE
FROM information_schema.collations
WHERE COLLATION_NAME IN ('utf8mb4_0900_ai_ci', 'utf8mb4_general_ci', 'latin1_swedish_ci');
```

| collation | PAD_ATTRIBUTE |
|---|---|
| `utf8mb4_0900_ai_ci` | NO PAD |
| `utf8mb4_general_ci` | PAD SPACE |
| `latin1_swedish_ci` | PAD SPACE |

### 相关变量

| 变量 | 作用 |
|---|---|
| `character_set_connection` / `collation_connection` | 字符串常量的 derivation（COERCIBLE） |
| `character_set_results` | 仅影响返回结果 |
| 无 | 隐式转换**没有开关**，无法关闭（这与 PG 的严格类型形成对照） |

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 让 `varchar_col = 123` 也能用索引 | 需要在类型聚合时判断"能否把转换移到常量侧"，MySQL 目前只在少数情形做 |
| 扩展 repertoire 的免转换范围 | `agg_item_set_converter` 里的 TODO（希望覆盖所有 ASCII repertoire 值） |
| 新增一种 collation 行为 | 引擎层 `CHARSET_INFO` + DD 的 `collations` 表 + `pad_attribute` |
| 让优化器知道"前缀索引的选择性" | 代价模型层面（当前 `rec_per_key` 不区分前缀与全列） |

### 已知缺陷

- **隐式转换不可关闭**：MySQL 没有 PG 那样的严格类型模式，也没有开关。
- **转换不体现在 EXPLAIN**：prepare 期就地替换，DBA 无法从执行计划直接看到。
- **NO PAD + CHAR 的假候选行问题**：优化器只能保守处理，没有更精确的机制。
- **前缀索引的选择性估算偏粗**：影响 `ref` 访问的行数估算。
- `agg_item_set_converter` 里有 `@todo - check why the constructors may return error`，说明这块历史上处理得不够干净。

### 社区边界澄清

- **没有严格类型模式**：PG 的 `varchar = 123` 报错，MySQL 静默转换。
- **没有 collation 层面的"索引不可用"提示**：只能靠 DBA 推断。
- **NO PAD 是 8.0 默认 collation 的新行为**，从 5.7 升级到 8.0 时 CHAR 列行为可能变化——这是升级兼容性检查的重点。

---

## 参考

**标准 / 论文**

- SQL:2016 Feature F690 "Collation support" 与 coercibility 定义（MySQL 扩展为 7 档，源码注释明说）
- Unicode Collation Algorithm (UCA) 9.0.0 —— `utf8mb4_0900_ai_ci` 的算法依据

**官方文档**

- MySQL 8.0 Reference Manual, "Character Sets, Collations, Unicode" / "Collation Coercibility in Expressions"
- MySQL 8.0 Reference Manual, "Column Indexes"（前缀索引与索引长度限制）
- MySQL 8.0 Reference Manual, "Type Conversion in Expression Evaluation"

**相关文档**

- [`../../infra/encoding.md`](../../infra/encoding.md) —— 字符集与编码基础
- [`../13_item_expression.md`](../13_item_expression.md) —— Item 类型体系
- [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) —— range 区间的构造
- [`13_functional_mv_index.md`](13_functional_mv_index.md) —— 函数索引（解决"字段被包裹"的手段）
- [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) —— prepare 期的 Item 树改写
