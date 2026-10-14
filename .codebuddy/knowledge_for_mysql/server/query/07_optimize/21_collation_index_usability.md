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

#### 1.1 七档 derivation

> ⚠️ **勘误**：`enum Derivation` 定义在 **`sql/field.h:179`**（不是 `sql/item.h`），且枚举值是**倒着排**的：

```cpp
/// For use @see DTCollation::aggregate()
enum Derivation {
  DERIVATION_IGNORABLE = 6,
  DERIVATION_NUMERIC = 5,
  DERIVATION_COERCIBLE = 4,
  DERIVATION_SYSCONST = 3,
  DERIVATION_IMPLICIT = 2,
  DERIVATION_NONE = 1,
  DERIVATION_EXPLICIT = 0
};
```

七档的语义（源码注释原文，在 `DTCollation::aggregate()` 的 doxygen，`sql/item.cc:2350`）：

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

`DERIVATION_NONE = 1` 夹在 `IMPLICIT(2)` 与 `EXPLICIT(0)` 之间，语义上是"混合态/待定"，不是最强。它表示"两边 collation 不同且强度相同，需要第三方来裁决"。

#### 1.1.1 repertoire：不是枚举，是三个 `#define`

```cpp
// include/m_ctype.h:184
/* Character repertoire flags */
#define MY_REPERTOIRE_ASCII 1     /* Pure ASCII            U+0000..U+007F */
#define MY_REPERTOIRE_EXTENDED 2  /* Extended characters:  U+0080..U+FFFF */
#define MY_REPERTOIRE_UNICODE30 3 /* ASCII | EXTENDED:     U+0000..U+FFFF */
```

```cpp
// sql/field.h:257 —— 第四档其实只是 ASCII 的别名
#define my_charset_numeric my_charset_latin1
#define MY_REPERTOIRE_NUMERIC MY_REPERTOIRE_ASCII
```

> ★ `UNICODE30(3) == ASCII(1) | EXTENDED(2)`——所以 repertoire 是**位域**，聚合时直接 `|=`。`DTCollation::repertoire` 的声明类型是 **`uint`**，不是某个 `enum my_repertoire_t`（该类型不存在）。

#### 1.1.2 `MY_COLL_*`：聚合模式位标志

```cpp
// sql/item.h:166
/*
   Flags for collation aggregation modes:
  MY_COLL_ALLOW_SUPERSET_CONV  - allow conversion to a superset
  MY_COLL_ALLOW_COERCIBLE_CONV - allow conversion of a coercible value
                                 (i.e. constant).
  MY_COLL_ALLOW_CONV           - allow any kind of conversion
                                 (combination of the above two)
  MY_COLL_ALLOW_NUMERIC_CONV   - if all items were numbers, convert to
                                 @@character_set_connection
  MY_COLL_DISALLOW_NONE        - don't allow return DERIVATION_NONE
                                 (e.g. when aggregating for comparison)
  MY_COLL_CMP_CONV             - combination of MY_COLL_ALLOW_CONV
                                 and MY_COLL_DISALLOW_NONE
*/
#define MY_COLL_ALLOW_SUPERSET_CONV 1
#define MY_COLL_ALLOW_COERCIBLE_CONV 2
#define MY_COLL_DISALLOW_NONE 4
#define MY_COLL_ALLOW_NUMERIC_CONV 8

#define MY_COLL_ALLOW_CONV \
  (MY_COLL_ALLOW_SUPERSET_CONV | MY_COLL_ALLOW_COERCIBLE_CONV)
#define MY_COLL_CMP_CONV (MY_COLL_ALLOW_CONV | MY_COLL_DISALLOW_NONE)   // = 7
```

> ★ `MY_COLL_CMP_CONV = 1|2|4 = 7` 是**比较类**函数（`=`、`LIKE`、`IN`、`BETWEEN`）的标准组合：允许转换，但**不允许结果停在 `DERIVATION_NONE`**——因为比较必须当场定下 collation，没有"等第三个参数来救"的机会。

#### 1.2 `DTCollation` 与 `aggregate()` 完整源码

```cpp
// sql/item.h:175 —— 整个类就是一个 POD 式三元组
class DTCollation {
 public:
  const CHARSET_INFO *collation;
  Derivation derivation{DERIVATION_NONE};
  uint repertoire;

  void set_repertoire_from_charset(const CHARSET_INFO *cs) {
    repertoire = cs->state & MY_CS_PUREASCII ? MY_REPERTOIRE_ASCII
                                             : MY_REPERTOIRE_UNICODE30;
  }
  DTCollation() {
    collation = &my_charset_bin;
    derivation = DERIVATION_NONE;
    repertoire = MY_REPERTOIRE_UNICODE30;
  }
  ...
  void set_numeric() {
    collation = &my_charset_numeric;      // == my_charset_latin1
    derivation = DERIVATION_NUMERIC;
    repertoire = MY_REPERTOIRE_NUMERIC;   // == MY_REPERTOIRE_ASCII
  }
  ...
  void set(Derivation derivation_arg) { derivation = derivation_arg; }
  // ★ 只改 derivation、不动 repertoire —— 这正是 aggregate() 里
  //   set(DERIVATION_NONE) 能把"待定"状态保留下来的原因
  bool aggregate(DTCollation &dt, uint flags = 0);
  const char *derivation_name() const { ... }
};
```

**超集判定的 helper**（`sql/item.cc:2322`）：

```cpp
static bool left_is_superset(DTCollation *left, DTCollation *right) {
  /* Allow convert to Unicode */
  if (left->collation->state & MY_CS_UNICODE &&
      (left->derivation < right->derivation ||
       (left->derivation == right->derivation &&
        (!(right->collation->state & MY_CS_UNICODE) ||
         /* The code below makes 4-byte utf8 a superset over 3-byte utf8 */
         (left->collation->state & MY_CS_UNICODE_SUPPLEMENT &&
          !(right->collation->state & MY_CS_UNICODE_SUPPLEMENT) &&
          left->collation->mbmaxlen > right->collation->mbmaxlen &&
          left->collation->mbminlen == right->collation->mbminlen)))))
    return true;
  /* Allow convert from any Unicode to utf32 or utf8mb4 */
  if (test_all_bits(left->collation->state,
                    MY_CS_UNICODE | MY_CS_UNICODE_SUPPLEMENT) &&
      right->collation->state & MY_CS_UNICODE &&
      left->derivation == right->derivation)
    return true;
  /* Allow convert from ASCII */
  if ((right->collation->state & MY_CS_PUREASCII) &&
      (left->derivation < right->derivation ||
       (left->derivation == right->derivation &&
        !(left->collation->state & MY_CS_PUREASCII))))
    return true;
  /* Disallow conversion otherwise */
  return false;
}
```

**`aggregate()` 完整源码**（`sql/item.cc:2403`；★ 只有一个定义，声明处的 `flags = 0` 是默认实参，不是重载）：

```cpp
bool DTCollation::aggregate(DTCollation &dt, uint flags) {
  // With two EXPLICIT derivations, collations must be equal:
  if (collation != dt.collation && derivation == DERIVATION_EXPLICIT &&
      dt.derivation == DERIVATION_EXPLICIT) {
    return true;                    // ★ 指针比较，不是 my_charset_same()
  }
  if (!my_charset_same(collation, dt.collation)) {
    /* 分支 A：字符集不同 */
    /*
       We do allow to use binary strings (like BLOBS)
       together with character strings.
       Binaries have more precedence than a character
       string of the same derivation.
    */
    if (collation == &my_charset_bin) {
      if (derivation <= dt.derivation)
        ;  // Do nothing —— ★ binary 那一侧只要不强于对方就保留
      else {
        set(dt);
      }
    } else if (dt.collation == &my_charset_bin) {
      if (dt.derivation <= derivation) {
        set(dt);
      }
    } else if ((flags & MY_COLL_ALLOW_SUPERSET_CONV) &&
               left_is_superset(this, &dt)) {
      // Do nothing
    } else if ((flags & MY_COLL_ALLOW_SUPERSET_CONV) &&
               left_is_superset(&dt, this)) {
      set(dt);
    } else if ((flags & MY_COLL_ALLOW_COERCIBLE_CONV) &&
               derivation < dt.derivation &&
               dt.derivation >= DERIVATION_SYSCONST) {
      // Do nothing;
    } else if ((flags & MY_COLL_ALLOW_COERCIBLE_CONV) &&
               dt.derivation < derivation &&
               derivation >= DERIVATION_SYSCONST) {
      set(dt);
    } else {
      // Cannot apply conversion
      set(&my_charset_bin, DERIVATION_NONE, (dt.repertoire | repertoire));
      return true;
    }
  } else if (derivation < dt.derivation) {
    // Do nothing                   /* 分支 B：字符集相同，纯比强度，小者胜 */
  } else if (dt.derivation < derivation) {
    set(dt);
  } else {
    /* 分支 C：字符集相同、强度相同、但 collation 不同 —— 真正的混合态 */
    if (collation == dt.collation) {
      // Do nothing
    } else {
      if (derivation == DERIVATION_EXPLICIT) {
        set(nullptr, DERIVATION_NONE, 0);
        return true;
      }

      /*
        If we have two different binary collations for the same character set,
        and none of them is explicit, we don't know which to choose. For
        example: utf8mb4_bin is a binary padding collation, utf8mb4_0900_bin is
        a binary non-padding collation. Cannot determine if the resulting
        collation should be padding or non-padding, unless they are also
        aggregated with a third explicit collation.
      */
      if ((collation->state & MY_CS_BINSORT) &&
          (dt.collation->state & MY_CS_BINSORT)) {
        set(DERIVATION_NONE);
        return false;               // ★ 返回 false：等第三个 EXPLICIT 参数
      }

      // When aggregating a binary and a non-binary collation for the same
      // character set, the binary collation is preferred.
      if (collation->state & MY_CS_BINSORT) return false;   // ★ 提前 return，跳过下面的 repertoire 聚合
      if (dt.collation->state & MY_CS_BINSORT) {
        set(dt);
        return false;
      }
      const CHARSET_INFO *bin =
          get_charset_by_csname(collation->csname, MY_CS_BINSORT, MYF(0));
      set(bin, DERIVATION_NONE);
    }
  }
  repertoire |= dt.repertoire;      // ★ repertoire 聚合只有这一行：位或
  return false;
}
```

三条规则（源码注释列出）：

1. **collation 相同** → 选它，derivation 取更强的
2. **collation 不同**：
   - 两个 EXPLICIT → **报错**（`CONCAT(a COLLATE x, b COLLATE y)` 是不合法的）
   - 否则**强度小的一方胜出**（列强于常量，COLLATE 强于列）
3. **强度相同但 collation 不同** → `DERIVATION_NONE`，等第三个参数来裁决（注释给了 `CONCAT` 三参数的例子）

#### 1.3 聚合失败的三个错误

| 错误 | 触发 | 抛出位置 |
|---|---|---|
| `ER_CANT_AGGREGATE_2COLLATIONS` | 两个参数无法聚合（`Illegal mix of collations`） | `sql/item.cc:2489` |
| `ER_CANT_AGGREGATE_3COLLATIONS` | 三个参数无法聚合 | `sql/item.cc:2497` |
| `ER_CANT_AGGREGATE_NCOLLATIONS` | 四个及以上（只报操作名，不逐个列 collation） | `sql/item.cc:2510` |

```cpp
// sql/item.cc:2486 —— 三个同名重载，按参数个数 2/3/N 分派
static void my_coll_agg_error(DTCollation &c1, DTCollation &c2,
                              const char *fname) {
  my_error(ER_CANT_AGGREGATE_2COLLATIONS, MYF(0), c1.collation->m_coll_name,
           c1.derivation_name(), c2.collation->m_coll_name,
           c2.derivation_name(), fname);
}

static void my_coll_agg_error(DTCollation &c1, DTCollation &c2, DTCollation &c3,
                              const char *fname) {
  my_error(ER_CANT_AGGREGATE_3COLLATIONS, MYF(0), ...);
}

static void my_coll_agg_error(Item **args, uint count, const char *fname,
                              int item_sep) {
  if (count == 2)
    my_coll_agg_error(args[0]->collation, args[item_sep]->collation, fname);
  else if (count == 3)
    my_coll_agg_error(args[0]->collation, args[item_sep]->collation,
                      args[2 * item_sep]->collation, fname);
  else
    my_error(ER_CANT_AGGREGATE_NCOLLATIONS, MYF(0), fname);
}
```

> ★ `item_sep` 是参数步长：非连续参数（如 `args[2]`、`args[5]`）时用它索引。
> ★ `sql/item_cmpfunc.cc:225` 有一份**同名同体的 static 副本**（同 TU 私有），与 `item.cc` 的互不冲突。
> ★ `sql/sql_union.cc:849` 是 UNION 分支里另一处 `ER_CANT_AGGREGATE_NCOLLATIONS`。

### 二、agg_item_charsets：聚合 + 就地永久替换

> `sql/item.cc`。

#### 2.1 两步

```cpp
// sql/item.cc:2660 —— 注释在 :2631
/*
  Collect arguments' character sets together.
  ...
  For functions with more than two arguments:

    collect(A,B,C) ::= collect(collect(A,B),C)

  When a character set conversion is needed, the respective Item pointer
  is updated in-place as a permanent transformation.      // ★★ 永久改写

  If the items are not consecutive (eg. args[2] and args[5]), use the
  item_sep argument, ie.

    agg_item_charsets(coll, fname, &args[2], 2, flags, 3)
*/
bool agg_item_charsets(DTCollation &coll, const char *fname, Item **args,
                       uint nargs, uint flags, int item_sep, bool only_consts) {
  if (agg_item_collations(coll, fname, args, nargs, flags, item_sep))
    return true;                                  // ① 决定用哪个 collation
  return agg_item_set_converter(coll, fname, args, nargs, flags, item_sep,
                                only_consts);     // ② 给需要转换的参数套转换器
}
```

> ⚠️ **勘误**：`agg_item_collations()` 是 **`static`**（`sql/item.cc:2513`，文件内私有）。对外暴露的包装叫 **`agg_item_collations_for_comparison()`**（`:2548`，声明 `sql/item.h:3894`）。

```cpp
// sql/item.cc:2513 —— 折左式聚合，只算 DTCollation，不改写 Item 树
static bool agg_item_collations(DTCollation &c, const char *fname, Item **av,
                                uint count, uint flags, int item_sep) {
  uint i;
  Item **arg;
  bool unknown_cs = false;

  c.set(av[0]->collation);
  for (i = 1, arg = &av[item_sep]; i < count; i++, arg += item_sep) {
    if (c.aggregate((*arg)->collation, flags)) {
      if (c.derivation == DERIVATION_NONE && c.collation == &my_charset_bin) {
        unknown_cs = true;
        continue;      // ★ 字符集未知，不立即报错：后面的 EXPLICIT 参数可能救回来
      }
      my_coll_agg_error(av, count, fname, item_sep);
      return true;
    }
  }

  if (unknown_cs && c.derivation != DERIVATION_EXPLICIT) {
    my_coll_agg_error(av, count, fname, item_sep);
    return true;
  }

  if ((flags & MY_COLL_DISALLOW_NONE) && c.derivation == DERIVATION_NONE) {
    my_coll_agg_error(av, count, fname, item_sep);
    return true;       // ★★ MY_COLL_DISALLOW_NONE 的生效点
  }

  /* If all arguments were numbers, reset to @@collation_connection */
  if (flags & MY_COLL_ALLOW_NUMERIC_CONV && c.derivation == DERIVATION_NUMERIC)
    c.set(Item::default_charset(), DERIVATION_COERCIBLE, MY_REPERTOIRE_NUMERIC);

  return false;
}

// sql/item.cc:2548
bool agg_item_collations_for_comparison(DTCollation &c, const char *fname,
                                        Item **av, uint count, uint flags) {
  return (agg_item_collations(c, fname, av, count,
                              flags | MY_COLL_DISALLOW_NONE, 1));
}
```

**两个包装的 flags 与 `only_consts` 都不同**（`sql/item.h:3898` / `:3908`）：

```cpp
inline bool agg_item_charsets_for_string_result(DTCollation &c,
                                                const char *name, Item **items,
                                                uint nitems, int item_sep = 1) {
  uint flags = MY_COLL_ALLOW_SUPERSET_CONV | MY_COLL_ALLOW_COERCIBLE_CONV |
               MY_COLL_ALLOW_NUMERIC_CONV;      // = 1|2|8 = 11
  return agg_item_charsets(c, name, items, nitems, flags, item_sep, false);
  //                                                              ★ false：都可转换
}
inline bool agg_item_charsets_for_comparison(DTCollation &c, const char *name,
                                             Item **items, uint nitems,
                                             int item_sep = 1) {
  uint flags = MY_COLL_ALLOW_SUPERSET_CONV | MY_COLL_ALLOW_COERCIBLE_CONV |
               MY_COLL_DISALLOW_NONE;           // = 1|2|4 = 7
  return agg_item_charsets(c, name, items, nitems, flags, item_sep, true);
  //                                                              ★ true：只转换常量
}
```

```cpp
// sql/item_func.h:528 —— 实测确认：比较路径确实传 only_consts=true
  /*
    Aggregate arguments for comparison, e.g: a=b, a LIKE b, a RLIKE b
    - don't convert to @@character_set_connection if all arguments are numbers
    - don't allow DERIVATION_NONE
  */
  bool agg_arg_charsets_for_comparison(DTCollation &c, Item **items,
                                       uint nitems, int item_sep = 1) {
    return agg_item_charsets_for_comparison(c, func_name(), items, nitems,
                                            item_sep);
  }
```

#### 2.2 ★ `only_consts`：比较场景只转换常量

**`agg_item_set_converter()` 完整源码**（`sql/item.cc:2554`）：

```cpp
bool agg_item_set_converter(DTCollation &coll, const char *fname, Item **args,
                            uint nargs, uint, int item_sep, bool only_consts) {
  // ★ 第 4 个形参是**匿名 uint**，函数体内从不使用 flags（flags 只在 agg_item_collations 阶段生效）

  Item *safe_args[2] = {nullptr, nullptr};
  /*
    For better error reporting: save the first and the second argument.
    We need this only if the the number of args is 3 or 2:
    - for a longer argument list, "Illegal mix of collations"
      doesn't display each argument's characteristics.
    - if nargs is 1, then this error cannot happen.
  */
  if (nargs >= 2 && nargs <= 3) {
    safe_args[0] = args[0];
    safe_args[1] = args[item_sep];
  }

  THD *thd = current_thd;

  uint i;
  Item **arg;
  for (i = 0, arg = args; i < nargs; i++, arg += item_sep) {
    size_t dummy_offset;
    // If told so (from comparison code), only add converter for const values.
    if (only_consts && !(*arg)->const_item()) continue;      // ★★ ①
    if (!String::needs_conversion(1, (*arg)->collation.collation,
                                  coll.collation, &dummy_offset))
      continue;                                              // ★★ ②

    /*
      No needs to add converter if an "arg" is NUMERIC or DATETIME
      value (which is pure ASCII) and at the same time target DTCollation
      is ASCII-compatible. For example, no needs to rewrite:
        SELECT * FROM t1 WHERE datetime_field = '2010-01-01';
      to
        SELECT * FROM t1 WHERE CONVERT(datetime_field USING cs) = '2010-01-01';

      TODO: avoid conversion of any values with
      repertoire ASCII and 7bit-ASCII-compatible,
      not only numeric/datetime origin.
    */
    if ((*arg)->collation.derivation == DERIVATION_NUMERIC &&
        (*arg)->collation.repertoire == MY_REPERTOIRE_ASCII &&
        my_charset_is_ascii_based((*arg)->collation.collation) &&
        my_charset_is_ascii_based(coll.collation))
      continue;                                              // ★★ ③ 免转换绿灯

    Item *conv = (*arg)->safe_charset_converter(thd, coll.collation);
    // @todo - check why the constructors may return error
    if (thd->is_error()) return true;
    if (conv == nullptr &&
        ((*arg)->collation.repertoire == MY_REPERTOIRE_ASCII))
      conv = new Item_func_conv_charset(thd, *arg, coll.collation, true);

    if (conv == nullptr) {
      if (nargs >= 2 && nargs <= 3) {
        /* restore the original arguments for better error message */
        args[0] = safe_args[0];
        args[item_sep] = safe_args[1];
      }
      my_coll_agg_error(args, nargs, fname, item_sep);
      return true;
    }

    // Update the Item pointer in-place
    if (thd->lex->is_exec_started())
      thd->change_item_tree(arg, conv);
    else
      *arg = conv;

    (*arg)->disable_constant_propagation(nullptr);

    if (conv->fix_fields(thd, arg)) return true;
  }

  return false;
}
```

```cpp
// include/sql_string.h:629 —— needs_conversion 是头文件里的 inline
inline bool String::needs_conversion(size_t arg_length,
                                     const CHARSET_INFO *from_cs,
                                     const CHARSET_INFO *to_cs,
                                     size_t *offset) {
  *offset = 0;
  if (to_cs == nullptr || (to_cs == &my_charset_bin) || from_cs == to_cs ||
      my_charset_same(from_cs, to_cs) ||
      ((from_cs == &my_charset_bin) &&
       (0 == (*offset = (arg_length % to_cs->mbminlen)))))
    return false;
  return true;
}
```

> ★ 由于 `agg_item_set_converter()` 传的 `arg_length = 1`，最后那个"binary 对齐"检查退化为 `1 % mbminlen == 0`，即**只有 `mbminlen == 1` 的单字节字符集才免转换**。

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

**`safe_charset_converter()` 家族**（这是"首选路径"如何工作的关键）：

```cpp
// sql/item.cc:1284 —— 基类：不自己做转换，只看 m_safe
Item *Item::safe_charset_converter(THD *thd, const CHARSET_INFO *tocs) {
  Item_func_conv_charset *conv =
      new Item_func_conv_charset(thd, this, tocs, true);
  return conv && conv->m_safe ? conv : nullptr;
  //                     ★ m_safe=false 表示转换会丢数据 → 返回 nullptr 让调用方报错
}

// sql/item.cc:1363 —— 字符串常量：lossless 时一旦 conv_errors>0 就返回 nullptr
Item *Item_string::charset_converter(THD *thd, const CHARSET_INFO *tocs,
                                     bool lossless) {
  uint conv_errors;
  String tmp, cstr, *ostr = val_str(&tmp);
  cstr.copy(ostr->ptr(), ostr->length(), ostr->charset(), tocs, &conv_errors);
  if (lossless && conv_errors > 0) {
    /*
      Safe conversion is not possible.
      We could not convert a string into the requested character set
      without data loss. The target charset does not cover all the
      characters from the string. Operation cannot be done correctly.
    */
    return nullptr;
  }

  char *ptr = thd->strmake(cstr.ptr(), cstr.length());
  if (ptr == nullptr) return nullptr;
  auto conv =
      new Item_string(ptr, cstr.length(), cstr.charset(), collation.derivation);
  if (conv == nullptr) return nullptr;
  /* Ensure that no one is going to change the result string */
  conv->mark_result_as_const();
  return conv;
  // ★ 成功时折算出**新的字面量 Item_string**，不是包一层 CONVERT
  //   ⇒ 优化器后续仍可把它当常量用（这一点对 range 优化很关键）
}

// sql/item.cc:1290 —— 数值常量：ASCII-based 目标直接返回 this
Item *Item_num::safe_charset_converter(THD *thd, const CHARSET_INFO *tocs) {
  /*
    Item_num returns pure ASCII result,
    so conversion is needed only in case of "tricky" character
    sets like UCS2. If tocs is not "tricky", return the item itself.
  */
  if (my_charset_is_ascii_based(tocs)) return this;
  ...
}

// sql/item.cc:5929 —— 列：只多设一个标志
Item *Item_field::safe_charset_converter(THD *thd, const CHARSET_INFO *tocs) {
  no_constant_propagation = true;
  return Item::safe_charset_converter(thd, tocs);
}

// sql/item.cc:3609 —— NULL：就地改 collation，返回自身
Item *Item_null::safe_charset_converter(THD *, const CHARSET_INFO *tocs) {
  collation.set(tocs);
  return this;
}
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
// sql/item_cmpfunc.cc:134（注释在 :134-146）
/**
  Aggregates result types from the array of items.
  ...
  Aggregation itself is performed by the item_cmp_type() function.
*/
static Item_result agg_cmp_type(Item **items, uint nitems) {
  Item_result type = items[0]->result_type();
  for (uint i = 1; i < nitems; i++) {
    type = item_cmp_type(type, items[i]->result_type());
  }
  return type;
}
```

> ⚠️ **勘误**：`agg_cmp_type` 是 **`static`**（`sql/item_cmpfunc.cc:148`），**没有 `item.h` 声明**，全仓仅 **1 处调用**：`sql/item_cmpfunc.cc:2950` 的
> ```cpp
> cmp_type = agg_cmp_type(args, 3);   // BETWEEN 的三参数聚合
> ```
> 也就是说：**绝大多数比较根本不走 `agg_cmp_type`**，而是在 `Arg_comparator::set_cmp_func()` 里直接调 `item_cmp_type(left->result_type(), right->result_type())`（`sql/item_cmpfunc.cc:1256`）。

**★ `item_cmp_type()` 全文只有 13 行**（`sql/item.cc:9292`）：

```cpp
Item_result item_cmp_type(Item_result a, Item_result b) {
  if (a == b) {
    assert(a != INVALID_RESULT);
    return a;
  } else if (a == ROW_RESULT || b == ROW_RESULT) {
    return ROW_RESULT;
  }
  if ((a == INT_RESULT || a == DECIMAL_RESULT) &&
      (b == INT_RESULT || b == DECIMAL_RESULT)) {
    return DECIMAL_RESULT;
  }
  return REAL_RESULT;      // ★★★ 兜底：任何"非 INT/DECIMAL 组合"的异类比较都落到 REAL
}
```

> ★★ **关键点**：**不存在** "STRING vs INT → REAL" 的显式行——这个结论完全由**最后一行兜底 `return REAL_RESULT;`** 产生。这解释了为什么 MySQL 的字符串↔数字比较统一走 double。

**完整聚合矩阵**（由上面 13 行推导）：

| a \ b | STRING | REAL | INT | DECIMAL | ROW |
|---|---|---|---|---|---|
| **STRING** | STRING | **REAL** | **REAL** | **REAL** | ROW |
| **REAL** | **REAL** | REAL | **REAL** | **REAL** | ROW |
| **INT** | **REAL** | **REAL** | INT | **DECIMAL** | ROW |
| **DECIMAL** | **REAL** | **REAL** | **DECIMAL** | DECIMAL | ROW |
| **ROW** | ROW | ROW | ROW | ROW | ROW |

> ★ 只有 **INT×INT → INT**、**INT×DECIMAL → DECIMAL**、**DECIMAL×DECIMAL → DECIMAL** 三格保住了精确域；其余任何"沾了 STRING 或 REAL"的组合都落到 REAL。

**★ 选值函数的真相**：8.0.39 **不用** `switch (compare_type)` 来选 `val_*`，而是查表（`sql/mysqld.cc:1050`）：

```cpp
arg_cmp_func Arg_comparator::comparator_matrix[5] = {
    &Arg_comparator::compare_string,      // STRING_RESULT  = 0 → val_str()
    &Arg_comparator::compare_real,        // REAL_RESULT    = 1 → val_real()
    &Arg_comparator::compare_int_signed,  // INT_RESULT     = 2 → val_int()
    &Arg_comparator::compare_row,         // ROW_RESULT     = 3
    &Arg_comparator::compare_decimal};    // DECIMAL_RESULT = 4 → val_decimal()
```

```cpp
// sql/item_cmpfunc.cc:764 —— switch 只做"修正"，不做选择
bool Arg_comparator::set_compare_func(Item_result_field *item,
                                      Item_result type) {
  m_compare_type = type;
  owner = item;
  func = comparator_matrix[type];        // ★★ 真正的分派在这一行

  switch (type) {
    ...
    case STRING_RESULT: {
      /*
        We must set cmp_charset here as we may be called from for an automatic
        generated item, like in natural join
      */
      if (cmp_collation.set((*left)->collation, (*right)->collation,
                            MY_COLL_CMP_CONV) ||
          cmp_collation.derivation == DERIVATION_NONE) {
        const char *func_name = owner ? owner->func_name() : "";
        my_coll_agg_error((*left)->collation, (*right)->collation, func_name);
        return true;                     // ★ "illegal mix of collations" 的第二个抛出点
      }
      if (cmp_collation.collation == &my_charset_bin) {
        /*
          We are using BLOB/BINARY/VARBINARY, change to compare byte by byte,
          without removing end space
        */
        if (func == &Arg_comparator::compare_string)
          func = &Arg_comparator::compare_binary_string;
      }
      /*
        If the comparison's and arguments' collations differ, prevent column
        substitution. Otherwise we would get into trouble with comparisons
        like:
        WHERE col = 'j' AND col = BINARY 'j'
        which would be transformed to:
        WHERE col = 'j' AND 'j' = BINARY 'j', then to:
        WHERE col = 'j'. That would be wrong, if col contains 'J'.
      */
      if ((*left)->collation.collation != cmp_collation.collation)
        (*left)->walk(&Item::disable_constant_propagation, enum_walk::POSTFIX,
                      nullptr);
      ...
      break;
    }
    case REAL_RESULT: {
      if ((*left)->decimals < DECIMAL_NOT_SPECIFIED &&
          (*right)->decimals < DECIMAL_NOT_SPECIFIED) {
        precision = 5 / log_10[max((*left)->decimals, (*right)->decimals) + 1];
        if (func == &Arg_comparator::compare_real)
          func = &Arg_comparator::compare_real_fixed;
      }
      break;
    }
    ...
  }
  return false;
}
```

> ★ `Arg_comparator`（**大写 A**，类名在 `sql/item_cmpfunc.h:172`）。
> ★ REAL 兜底还有一个**逃生门**（`item_cmpfunc.cc:1213`）：`DECIMAL 非常量` 与 `STRING 常量` 比较时改按 **DECIMAL** 比，避免丢精度。

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
// sql/item_cmpfunc.cc:180（注释在 :162-179）
enum_field_types agg_field_type(Item **items, uint nitems) {
  assert(nitems > 0 && items[0]->result_type() != ROW_RESULT);
  enum_field_types res = items[0]->data_type();
  for (uint i = 1; i < nitems; i++)
    res = Field::field_type_merge(res, items[i]->data_type());
  return real_type_to_type(res);
}
```

> ★ 与 `agg_cmp_type` 分工不同：这个返回 **`enum_field_types`**（`MYSQL_TYPE_*`），用于 **CASE / UNION / 多参函数的结果列类型**；`agg_cmp_type` 返回 `Item_result`，用于**比较方式**。声明在 `sql/item_func.h:4000`。

**★ `Field::field_type_merge` 全文只有 3 行——纯查表**（`sql/field.cc:1240`）：

```cpp
/**
  Return type of which can carry value of both given types in UNION result.
  ...
*/
enum_field_types Field::field_type_merge(enum_field_types a,
                                         enum_field_types b) {
  assert(a != MYSQL_TYPE_INVALID && b != MYSQL_TYPE_INVALID);
  return field_types_merge_rules[field_type2index(a)][field_type2index(b)];
}
```

> ⚠️ **勘误**：这里**没有任何"宽度取最大"逻辑**——`field_type_merge` 只决定"类型"，不决定长度/decimals。

矩阵本体在 `sql/field.cc:251-1183`（`[行][列]`，行=第一个类型）。三条代表行：

```cpp
static enum_field_types field_types_merge_rules[FIELDTYPE_NUM][FIELDTYPE_NUM] =
    {
        /* MYSQL_TYPE_DECIMAL -> */
        {// MYSQL_TYPE_DECIMAL      MYSQL_TYPE_TINY
         MYSQL_TYPE_NEWDECIMAL, MYSQL_TYPE_NEWDECIMAL,
         // MYSQL_TYPE_SHORT        MYSQL_TYPE_LONG
         MYSQL_TYPE_NEWDECIMAL, MYSQL_TYPE_NEWDECIMAL,
         // MYSQL_TYPE_FLOAT        MYSQL_TYPE_DOUBLE
         MYSQL_TYPE_DOUBLE, MYSQL_TYPE_DOUBLE,       // ★ DECIMAL × FLOAT → DOUBLE
         // MYSQL_TYPE_NULL         MYSQL_TYPE_TIMESTAMP
         MYSQL_TYPE_NEWDECIMAL, MYSQL_TYPE_VARCHAR,  // ★ DECIMAL × TIMESTAMP → VARCHAR
         ...
         // MYSQL_TYPE_STRING       MYSQL_TYPE_GEOMETRY
         MYSQL_TYPE_STRING, MYSQL_TYPE_VARCHAR},
        ...
        /* MYSQL_TYPE_STRING -> */                  // ★★ CHAR 行是"霸道"的
        {// MYSQL_TYPE_DECIMAL      MYSQL_TYPE_TINY
         MYSQL_TYPE_STRING, MYSQL_TYPE_STRING,
         // MYSQL_TYPE_SHORT        MYSQL_TYPE_LONG
         MYSQL_TYPE_STRING, MYSQL_TYPE_STRING,
         // MYSQL_TYPE_FLOAT        MYSQL_TYPE_DOUBLE
         MYSQL_TYPE_STRING, MYSQL_TYPE_STRING,      // ★ CHAR × 数值 → 仍是 STRING
         ...
```

> ★ **`MYSQL_TYPE_STRING`（CHAR）行是"霸道"的**：CHAR 与任何数值合并都还是 `MYSQL_TYPE_STRING`。这是 CHAR 列参与 UNION/CASE 时**不会被提升成数值**的根源——但注意这跟**比较**行为无关，比较仍走 `item_cmp_type` 落到 `REAL_RESULT`。
>
> ★ 总体规律：**数值 × 数值 → 更宽的数值类型；数值 × 字符串 → VARCHAR；字符串 × 大对象 → 更大的 BLOB**。

**"宽度/小数位取最大"的真实位置**（`sql/item.cc:556` 的 `Item::aggregate_type`）：

```cpp
void Item::aggregate_type(Bounds_checked_array<Item *> items) {
  ...
  enum_field_types new_type = real_data_type(items[itemno]);
  uint8 new_dec = items[itemno]->decimals;
  bool new_unsigned = items[itemno]->unsigned_flag;
  bool mixed_signs = false;

  for (itemno = itemno + 1; itemno < count; itemno++) {
    if (items[itemno]->data_type() == MYSQL_TYPE_NULL) continue;
    new_type = Field::field_type_merge(new_type, real_data_type(items[itemno]));
    mixed_signs |= (new_unsigned != items[itemno]->unsigned_flag);
    new_dec = max<uint8>(new_dec, items[itemno]->decimals);  // ★★ 小数位取最大
  }
  if (mixed_signs && is_integer_type(new_type)) {
    // ★ 有符号/无符号混用 ⇒ 类型"升档"，避免溢出
    bool bump_range = false;
    ...
    if (bump_range) {
      switch (new_type) {
        case MYSQL_TYPE_TINY:      new_type = MYSQL_TYPE_SHORT;      break;
        case MYSQL_TYPE_SHORT:     new_type = MYSQL_TYPE_INT24;      break;
        case MYSQL_TYPE_INT24:     new_type = MYSQL_TYPE_LONG;       break;
        case MYSQL_TYPE_LONG:      new_type = MYSQL_TYPE_LONGLONG;   break;
        case MYSQL_TYPE_LONGLONG:  new_type = MYSQL_TYPE_NEWDECIMAL; break;
        default: break;
      }
    }
  }
  ...
}
```

> ★★ 这解释了 UNION / 派生表的一个常见坑：**结果列的类型是聚合出来的**，可能与任一 operand 都不同。而且 `mixed_signs` 分支会**因为一个无符号 operand 就把整个结果升一档**（`LONGLONG` 甚至能升到 `NEWDECIMAL`）。若聚合后的类型与外层比较的类型不一致，又是一次隐式转换。

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

#### 4.2 对 CHAR 列的影响："存满读空"

★ **准确说法**：CHAR 不是"去掉尾随空格存储"，而是**存的时候补满、读的时候去尾**——两条方向相反的规则。

**写入侧补满**（`Field_string::store`，`sql/field.cc:6127`；注意这里有 `\r\n`，实际源码无 CR）：

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
                              field_charset->pad_char);   // ★★ 补 pad_char 到 field_length
  ...
}
```

**读取侧去尾**（`Field_string::val_str`，`sql/field.cc:6278`）：

```cpp
String *Field_string::val_str(String *, String *val_ptr) const {
  ASSERT_COLUMN_MARKED_FOR_READ;
  size_t length;
  if (current_thd->variables.sql_mode & MODE_PAD_CHAR_TO_FULL_LENGTH)
    length = my_charpos(field_charset, ptr, ptr + field_length,
                        field_length / field_charset->mbmaxlen);
  else
    length = field_charset->cset->lengthsp(field_charset, (const char *)ptr,
                                           field_length);
    // ★★ 默认（无 PAD_CHAR_TO_FULL_LENGTH）就用 lengthsp 去掉尾部空格
  val_ptr->set((const char *)ptr, length, field_charset);
  return val_ptr;
}
```

> ★★ 注意：这个"去尾"**跟 `pad_attribute` 无关**——CHAR 的尾空格剥离是**无条件**的。NO PAD 只是让**比较/排序侧也不再忽略尾空格**，于是"表里读到的值"与"索引/排序键里的值"两套语义分叉。

**需要手工去尾的地方**（`Field_string::cmp`，`sql/field.cc:6319`）：

```cpp
int Field_string::cmp(const uchar *a_ptr, const uchar *b_ptr) const {
  ...
  if (field_charset->pad_attribute == NO_PAD &&
      !(current_thd->variables.sql_mode & MODE_PAD_CHAR_TO_FULL_LENGTH)) {
    /*
      Our CHAR default behavior is to strip spaces. For PAD SPACE collations,
      this doesn't matter, for but NO PAD, we need to do it ourselves here.
    */
    // ★ PAD SPACE 时排序规则自身就忽略尾空格；NO PAD 时必须手工 lengthsp
    a_len = field_charset->cset->lengthsp(field_charset, (const char *)a_ptr,
                                          a_len);
    b_len = field_charset->cset->lengthsp(field_charset, (const char *)b_ptr,
                                          b_len);
  }
```

**`Field_string::make_sort_key`**（`sql/field.cc:6349`）——★ CHAR 恒用定长排序键：

```cpp
  if (field_charset->pad_attribute == NO_PAD &&
      !(current_thd->variables.sql_mode & MODE_PAD_CHAR_TO_FULL_LENGTH)) {
    input_length = field_charset->cset->lengthsp(
        field_charset, (const char *)ptr, input_length);
  }

  size_t tmp [[maybe_unused]] = field_charset->coll->strnxfrm(
      field_charset, to, length, char_length_cache, ptr, input_length,
      MY_STRXFRM_PAD_TO_MAXLEN);   // ★★ CHAR 无条件 PAD_TO_MAXLEN（定长）
```

#### 4.3 ★ 优化器因此必须保守

⚠️ **勘误**：这个函数真名是 **`ref_lookup_subsumes_comparison`**（`sql/sql_optimizer.cc:8903`）。**不存在** `equal_propagation_is_guaranteed` / `index_is_complete_for_equal`（全仓 0 命中）。

它的语义是：回答"**ref/eq_ref 访问能否完全吞掉（subsume）这个等值条件、从而把条件从 WHERE 里删掉**"。返回 `*subsumes = false` ⇒ 条件必须保留在表过滤层。

**完整源码**（`sql/sql_optimizer.cc:8903-8962`）：

```cpp
bool ref_lookup_subsumes_comparison(THD *thd, Field *field, Item *right_item,
                                    bool can_evaluate, bool *subsumes) {
  *subsumes = false;
  right_item = right_item->real_item();
  if (right_item->type() == Item::FIELD_ITEM) {
    *subsumes = field->eq_def(down_cast<Item_field *>(right_item)->field);
    return false;
  } else if (right_item->type() == Item::CACHE_ITEM) {
    // remove equalities injected by IN->EXISTS transformation
    *subsumes = down_cast<Item_cache *>(right_item)->eq_def(field);
    return false;
  }
  bool right_is_null = true;
  if (can_evaluate) {
    assert(evaluate_during_optimization(right_item,
                                        thd->lex->current_query_block()));
    right_is_null = right_item->is_nullable() && right_item->is_null();
    if (thd->is_error()) return true;
  }
  if (!right_is_null) {
    /*
      We can remove all fields except:
      1. String data types:
       - For BINARY/VARBINARY fields with equality against a
         string: Ref access can return more rows than match the
         string. The reason seems to be that the string constant
         is not "padded" to the full length of the field when
         setting up ref access. @todo Change how ref access for
         BINARY/VARBINARY fields are done so that only qualifying
         rows are returned from the storage engine.
      2. Float data type: Comparison of float can differ
       - When we search "WHERE field=value" using an index,
         the "value" side is converted from double to float by
         Field_float::store(), then two floats are compared.
       - When we search "WHERE field=value" without indexes,
         the "field" side is converted from float to double by
         Field_float::val_real(), then two doubles are compared.
    */
    if (field->type() == MYSQL_TYPE_STRING &&
        field->charset()->pad_attribute == NO_PAD) {
      /*
        For "NO PAD" collations on CHAR columns, this function must return
        false, because removal of trailing space in CHAR columns makes the
        table value and the index value compare differently. As the column
        strips trailing spaces, it can return false candidates. Further
        comparison of the actual table values is required.
       */
      return false;       // ★★ 唯一的真正 early return（且只针对 CHAR，不含 VARCHAR）
    }
    if (!((field->type() == MYSQL_TYPE_STRING ||        // 1 BINARY/VARBINARY
           field->type() == MYSQL_TYPE_VARCHAR) &&
          field->binary()) &&
        !(field->type() == MYSQL_TYPE_FLOAT && field->decimals() > 0))  // 2 FLOAT
    {
      *subsumes = !right_item->save_in_field_no_warnings(field, true);
      if (thd->is_error()) return true;
    }
  }
  return false;
}
```

> ★ 这段注释点出了一个 subtle 的正确性问题：**NO PAD + CHAR 时，索引查找可能返回"假候选行"**（索引里有尾随空格的版本，表里却存着去掉空格的版本），所以优化器不能假设"索引查找结果 = 最终结果"，必须保留进一步的表值比较。
>
> ★ 注意三条保守判定的**形态不同**：NO PAD+CHAR 是真正的 `return false;`（且只针对 `MYSQL_TYPE_STRING`，**不含 VARCHAR**）；BINARY/VARBINARY 与 FLOAT 则合并在下面那个 `if` 的**否定条件**里，效果是跳过 `*subsumes = ...` 赋值，让它保持 `false`。

同类保守判定还有（同一函数的注释）：

- **BINARY/VARBINARY**：等值查找时常量不会被填充到字段全长，ref access 可能返回多余行（源码留了 `@todo`）
- **FLOAT**：`Field_float::store()` 把 double 转 float 后比较，与"字段转 double 后比较"的结果可能不同

这三条都是"**索引的排序/比较语义与表值的比较语义不完全一致**"的例子，优化器必须在这类情形下放弃某些优化。

#### 4.4 对 filesort 与去重的影响

> ⚠️ **勘误**：`(field_charset->pad_attribute == NO_PAD) ? 0 : MY_STRXFRM_PAD_TO_MAXLEN;` **不在 `sql/filesort.cc`**。它在 **`sql/field.cc:6727`**（`Field_varstring::make_sort_key`）和 **`sql/field.cc:7301`**（`Field_blob::make_sort_key`）：

```cpp
// sql/field.cc:6726
size_t Field_varstring::make_sort_key(uchar *to, size_t length,
                                      size_t trunc_pos) const {
  const int flags =
      (field_charset->pad_attribute == NO_PAD) ? 0 : MY_STRXFRM_PAD_TO_MAXLEN;
  ...
```

```cpp
// sql/field.cc:7300 —— 同一三元式的第二处（BLOB）
size_t Field_blob::make_sort_key(uchar *to, size_t length,
                                 size_t trunc_pos) const {
  static const uchar EMPTY_BLOB[1] = {0};
  const int flags =
      (field_charset->pad_attribute == NO_PAD) ? 0 : MY_STRXFRM_PAD_TO_MAXLEN;
  ...
```

**filesort 侧的机制不同**（`sql/filesort.cc:2120`）：它把 NO PAD 转成"**变长排序键**"：

```cpp
      case STRING_RESULT: {
        const CHARSET_INFO *cs = item->collation.collation;
        sortorder->length = item->max_length;

        if (cs->pad_attribute == NO_PAD) {
          sortorder->is_varlen = true;    // ★★ NO PAD ⇒ 排序键按变长处理，不补齐
        }
        ...
      }
```

然后在真正生成排序键时按 `is_varlen` 分派（`sql/filesort.cc:1285`）：

```cpp
      if (is_varlen) {
        ...
        actual_length = cs->coll->strnxfrm(
            cs, to, max_length, item->max_char_length(),
            pointer_cast<const uchar *>(from), src_length, 0);   // ★ flags=0
        if (actual_length == max_length) {
          /*
            The sort key eithen fit perfectly, or overflowed; we can't
            distinguish between the two, so we have to count it as overflow.
          */
          return UINT_MAX;
        }
      } else {
        actual_length = cs->coll->strnxfrm(
            cs, to, dst_length.value(), item->max_char_length(),
            pointer_cast<const uchar *>(from), src_length,
            MY_STRXFRM_PAD_TO_MAXLEN);   // ★ 定长路径：补齐到 maxlen
        assert(actual_length == dst_length.value());
      }
```

> ★★ **一张对照表收尾**：
>
> | 类型 | 排序键 | 由谁决定 |
> |---|---|---|
> | **CHAR**（`Field_string::make_sort_key`） | **恒定长** `MY_STRXFRM_PAD_TO_MAXLEN` | 无条件（先手工 `lengthsp` 去尾） |
> | **VARCHAR**（`Field_varstring::make_sort_key`） | `pad_attribute == NO_PAD ? 0 : PAD_TO_MAXLEN` | 按 collation |
> | **BLOB**（`Field_blob::make_sort_key`） | 同上 | 按 collation |
> | filesort 的 Item 排序键 | `is_varlen = (pad_attribute == NO_PAD)` | 按 collation |
>
> CHAR 恒用定长排序键这一点，与 §4.3 `ref_lookup_subsumes_comparison` **只针对 `MYSQL_TYPE_STRING`（CHAR）**做保守判定完全吻合——二者是同一个根因的两处表现。

也就是说排序时的归一化方式不同。这会影响到"索引顺序能否直接满足 ORDER BY"的判定。

### 五、索引长度限制与前缀索引

| 限制 | 值 | 定义位置 | 影响 |
|---|---|---|---|
| MySQL 层键长上限 | **`MAX_KEY_LENGTH = 3072`** | `sql/sql_const.h:48` | handler 接口的默认 `max_supported_key_length()` |
| handler 默认列前缀上限 | **255** | `sql/handler.h:5806` | 与 `HA_MAX_KEY_LENGTH 1000`（`include/my_compare.h:53`，MyISAM/HEAP 历史值）**是两套东西** |
| InnoDB REDUNDANT/COMPACT | **767**（= 768 − 1） | `rem0types.h:68` + `ha_innodb.cc:7669` | 老行格式 |
| InnoDB DYNAMIC/COMPRESSED | **3072** | `rem0types.h:73` + `ha_innodb.cc:7672` | 新行格式 |

```cpp
// sql/sql_const.h:48
constexpr const unsigned int MAX_KEY_LENGTH{3072}; /* max possible key */

// sql/handler.h:5800
  virtual uint max_supported_key_length() const { return MAX_KEY_LENGTH; }
  virtual uint max_supported_key_part_length(HA_CREATE_INFO *create_info
                                             [[maybe_unused]]) const {
    return 255;
  }
```

```cpp
// storage/innobase/include/rem0types.h:66
constexpr uint32_t REC_ANTELOPE_MAX_INDEX_COL_LEN = 768;      // −1 = 767

/** Maximum indexed field length for tables that have atomic BLOBs.
This (3072) is the maximum index row length allowed, so we cannot create index
prefix column longer than that. */
constexpr uint32_t REC_VERSION_56_MAX_INDEX_COL_LEN = 3072;
```

```cpp
// storage/innobase/handler/ha_innodb.cc:7663 —— 767 / 3072 在这里才出现
uint ha_innobase::max_supported_key_part_length(
    HA_CREATE_INFO *create_info) const {
  switch (create_info->row_type) {
    case ROW_TYPE_REDUNDANT:
    case ROW_TYPE_COMPACT:
      return (REC_ANTELOPE_MAX_INDEX_COL_LEN - 1);   // ★ 768 − 1 = 767
    default:
      return (REC_VERSION_56_MAX_INDEX_COL_LEN);     // ★ 3072
  }
}
```

> ★ **全仓没有 `= 767` 的字面量常量**，只有 `768` 减 1。另外 `ha_innobase::max_supported_key_length()` 对 16K 页返回 **3500**——但那只是"历史报告值"，注释明说 `MySQL imposes its own limit to this number; MAX_KEY_LENGTH = 3072`，**实际生效的始终是 3072**。

**前缀索引**：`col(N)` **只索引前 N 个字符**，会导致"索引查找返回候选行，还需回表精确比较"。

**★ 前缀索引的判定式**（`field->key_length() != key_part->length`）：

```cpp
// sql/sql_optimizer.cc:8887 —— 优化器判定：前缀索引不能被 ref 完全吞掉
static Item *part_of_refkey(TABLE *table, Index_lookup *ref,
                            const Field *field) {
  uint ref_parts = ref->key_parts;
  if (ref_parts) {
    if (ref->has_guarded_conds()) return nullptr;

    const KEY_PART_INFO *key_part = table->key_info[ref->key].key_part;

    for (uint part = 0; part < ref_parts; part++, key_part++)
      if (field->eq(key_part->field) &&
          !(key_part->key_part_flag & HA_PART_KEY_SEG))   // ★★ 前缀 ⇒ 返回 nullptr
        return ref->items[part];
  }
  return nullptr;
}
```

> ★★ 这就是"**前缀索引导致 ref 可能返回多余行**"的判定：`HA_PART_KEY_SEG` 置位 = 该 key part 只是列的前缀 ⇒ `part_of_refkey` 返回 `nullptr` ⇒ 优化器**不敢**删除该列的等值条件，必须回表二次过滤。

该判定式的其他出现点（证明 `HA_PART_KEY_SEG ≡ key_length() != part->length`）：

```cpp
// sql/table.cc:2821 —— 建"前缀字段"：克隆出一个截断版 Field 供键比较
      if (field->key_length() != key_part->length &&
          !field->is_flag_set(BLOB_FLAG)) {
        /*
          We are using only a prefix of the column as a key:
          Create a new field for the key part that matches the index
        */

// sql/sql_table.cc:11880 —— ALTER 时判定是否需要重建表
      bool old_part_key_seg = (key_part->key_part_flag & HA_PART_KEY_SEG);
      bool new_part_key_seg = (new_field->key_length() != new_part->length);
      if (old_part_key_seg ^ new_part_key_seg) return true;

// sql/range_optimizer/index_range_scan_plan.cc:746 —— PK 是否完全覆盖该索引
    if (param->table->field[fieldnr]->key_length() != kp->length) return false;
```

```cpp
// sql/field.h —— key_length() 的三个典型定义
  virtual uint32 key_length() const { return pack_length(); }      // :1164 基类默认
  uint32 key_length() const final { return (uint32)field_length; } // :3537 Field_varstring
  uint32 key_length() const override { return 0; }                 // :3737 Field_blob（前缀长度由 KEY_PART_INFO::length 单独给）
```

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
