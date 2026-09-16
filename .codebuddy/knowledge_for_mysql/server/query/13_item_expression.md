# 13 Item 表达式体系：MySQL 一切可求值之物的抽象

> `Item` 是 MySQL 对"**表达式**"的统一抽象——字面量、列引用、函数、聚合、子查询、条件、`?` 参数，SQL 里一切能求值之物都是 `Item`。它是解析器、优化器、执行器三层共用的数据结构，也是理解"SQL 到底是怎么被求值的"必经之路。
>
> 本篇是体系篇，把此前分散在 resolver、优化器各篇的 Item 相关内容串成一棵树。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、Item 类继承树](#一item-类继承树)
- [二、求值模型：val_* 协议](#二求值模型val_-协议)
- [三、fix_fields：两阶段绑定](#三fix_fields两阶段绑定)
- [四、Item 与 Field](#四item-与-field)
- [五、特殊 Item 子类](#五特殊-item-子类)
- [六、聚合函数状态机：Item_sum 与 Aggregator](#六聚合函数状态机item_sum-与-aggregator)
- [七、参数](#七参数)
- [参考](#参考)

---

## 设计思想与理论基础

### 为什么用"继承 + 虚函数"而不是"枚举 + switch"？

`Item` 有 **122 种 `Functype`**、近 30 种 `Type`。如果用"枚举 + switch"实现，所有求值语义都会挤进几个巨型 switch，加一个函数要改 N 处。

MySQL 的选择是**多态**：

- **求值语义多态**——每个函数的求值就在自己的 `val_xxx()` 里，加函数只需加一个类
- **元数据也是多态的**——`resolve_type()`、`used_tables()`、`get_filtering_effect()`、`get_monotonicity_info()`、`propagate_type()`、`walk()` 全是虚函数。优化器想知道"这个表达式依赖哪些表""选择率多少""是否单调"，都不需要 `switch(functype())`，直接虚调用

**代价**（要如实说）：

- 每个 Item 对象都有虚表指针，且 Item 全部从 MEM_ROOT 分配（`operator new` 被重载为 `mem_root->Alloc`），**没有逐个析构**——整个查询的 Item 在查询结束时一次性释放
- 求值路径全是虚调用，无法内联
- 这是典型的"**用 CPU 与内存换可扩展性**"的取舍，而 MySQL 的表达式求值确实不快（对比 PG 的表达式求值器）

### 火山模型下的求值

执行器是火山模型：每次 `Read()` 取一行。行取到后，WHERE/HAVING/SELECT list 里的 Item 树被**就地求值**：

```
RowIterator::Read() 取到一行
  → 条件求值：Item_cond_and::val_int()（短路）
      → Item_func_eq::val_int()
          → Item_field::val_int() → Field::val_int()（从 record buffer 读）
          → Item_int::val_int()
  → SELECT list 求值：Item_func_xxx::val_str() ...
```

即：**Item 树是"被动"的——它不主动求值，而是被上层按需拉动**。

---

### Item 既是 AST 又是执行节点：零转换 vs 状态分散

MySQL **没有独立的 AST 层**——parser 直接构造 Item 树，`fix_fields()` 在这棵树上**原地**完成名字解析、类型推导、常量折叠，执行期继续用同一棵树求值。

**收益（零转换）**：不需要"AST → 逻辑计划 → 物理计划"的多层 IR 转换，省掉了中间结构的构造与遍历开销；改写（IN→EXISTS、等值传播）直接改树即可。

**代价（状态分散）**：同一个 Item 对象同时承载四层互不相同的状态——

| 层 | 字段举例 |
|---|---|
| 语法信息 | 原始名字 `m_orig_*`、位置 |
| 语义信息 | `data_type` / `collation` / `max_length` / `unsigned_flag` |
| 优化期状态 | `fixed`、`used_tables_cache`、`item_equal` 指针 |
| **执行期状态** | `null_value`、`Item_sum` 的累加器、`Item_cache` 的缓存值 |

后果是：Item 的"阶段"没有类型系统保护，全靠 `fixed` 标志和调用约定维持。跨多次执行（PS）时必须显式 `cleanup()`，而 `cleanup()` 又**不清 `fixed`**——这些隐含契约是 Item 类体系最容易出错的地方。

### 树改写三件套：walk / transform / compile

Item 树的遍历与改写不用访问者类，而是三个函数式接口：

```cpp
  virtual bool walk(Item_processor processor, enum_walk walk, uchar *arg);
  virtual Item *transform(Item_transformer transformer, uchar *arg);
  virtual Item *compile(Item_analyzer analyzer, uchar **arg_p,
                        Item_transformer transformer, uchar *arg_t);
```

- **`walk`**——只读遍历（`enum_walk::PREFIX` / `POSTFIX`），用于收集信息
- **`transform`**——自底向上改写，返回新节点替换自己（如常量传播）
- **`compile`**——"分析 + 改写"二合一：`analyzer` 先判断是否值得深入，`transformer` 再改写，**用于避免无谓的整树遍历**

实战例子：`Item_cond::fix_fields()` 用 `walk(&Item::is_non_const_over_literals, POSTFIX, nullptr)` 排除"含子查询/存储函数"的伪常量，防止误折叠。

### 三套类型标签，别搞混

容易混淆的三套：

| 接口 | 粒度 | 作用 |
|---|---|---|
| **`type()`** | 28 个 `Type` 枚举值 | **运行时类别标签**——`FIELD_ITEM` / `FUNC_ITEM` / `SUM_FUNC_ITEM` / `SUBSELECT_ITEM` / `PARAM_ITEM` ... 用于"这是个什么东西"的判断与 `down_cast` 断言 |
| **`result_type()`** | 5 个 `Item_result` | 决定**调哪个 `val_xxx`**（见第二章） |
| **`data_type()`** | `enum_field_types` | 决定**精确 SQL 类型、精度、字符集** |

另有 `functype()`（122 个值）用于函数内部的语义判断。源码明确提醒：**运行时判断用 `functype()`，不要用 `func_name()` 做字符串比较**：

```
    This method should not be used for runtime type identification, use enum
    {Sum}Functype and Item_func::functype()/Item_sum::sum_func() instead.
```

### 为什么 MySQL 没有表达式 JIT

PostgreSQL 有 LLVM JIT 把表达式编译成机器码，MySQL 8.0 **没有**。从 Item 的设计能直接看出原因：

1. **没有可编译的中间表示**——前面"状态分散"意味着 Item 把语法、类型、执行期状态混在一起，没有一层干净的、可 lowering 的 IR
2. **求值入口是 5 个虚函数**而非单一 `eval()`；且 `val_str()` 的 buffer 协议依赖"返回内部缓冲还是外部缓冲"这种**运行期约定**，难以静态编译
3. **生命周期跨执行且可变**——PS 下每次执行前要 `bind_fields()` 重绑 `Field*`；`Item_cache` 的惰性求值、`Item_sum` 的累加器状态都是动态插入的，编译产物容易失效
4. **定位取舍**——MySQL 的传统优势场景是 OLTP 短查询（主键点查、简单 join），这类负载中表达式求值占比不高，JIT 收益有限，而实现 JIT 的复杂度（维护 IR、失效处理、跨版本 LLVM 兼容）很大

对比：PG 的 `ExprEvalStep` 是一套扁平的、类型专有的"指令数组"，天生适合编译；MySQL 的 Item 树是面向对象的高层结构，**适合改写而不适合编译**。这是两种设计取向的分野，而非能力高下。

---

## 一、Item 类继承树

### 1.1 基类：`Item : public Parse_tree_node`

`Item` 基类的官方注释是理解它的最好开篇：

```
/**
  Base class that is used to represent any kind of expression in a
  relational query. The class provides subclasses for simple components, like
  literal (constant) values, column references and variable references,
  as well as more complex expressions like comparison predicates,
  arithmetic and string functions, row objects, function references and
  subqueries.
  ...
  For Item objects with longer lifespan than one execution, we must take
  special precautions when referencing objects with shorter lifespan.
  For example, TABLE and Field objects against most tables are valid only for
  one execution. For such objects, Item classes should rather reference
  Table_ref and Item_field objects instead of TABLE and Field, because
  these classes support dynamic rebinding of objects before each execution.
  See Item::bind_fields() which binds new objects per execution and
  Item::cleanup() that deletes references to such objects.
*/
```

三个要点：

1. **Item 是 parse tree node 的一种**（继承 `Parse_tree_node`）——这直接对应第三章的"两阶段"
2. **Item 是表达式，不是列**——列引用是 `Item_field`，但 `Item` 本身是"可求值之物"
3. **生命周期可能跨多次执行**（prepared statement）——所以 Item 不能直接持有 `TABLE*`/`Field*`，要持有 `Table_ref*` 并在每次执行前 `bind_fields()`

**内存分配：只用 MEM_ROOT**

```cpp
static void *operator new(size_t size) noexcept {
  return (*THR_MALLOC)->Alloc(size);
}
```

**5 个 `val_xxx()` 是核心协议**（详见第二章）：

```cpp
/* valXXX methods must return NULL or 0 or 0.0 if null_value is set. */
virtual double val_real() = 0;
virtual longlong val_int() = 0;
virtual String *val_str(String *str) = 0;
virtual my_decimal *val_decimal(my_decimal *decimal_buffer) = 0;
virtual bool val_bool();          // 非纯虚，基类有默认实现
```

**核心虚接口**：

```cpp
virtual enum Type type() const = 0;                    // 28 个值的粗粒度类型标签
virtual Item_result result_type() const { return REAL_RESULT; }
virtual table_map used_tables() const { return (table_map)0L; }   // 默认"不依赖表"=常量
virtual bool fix_fields(THD *, Item **);
virtual bool is_null() { return false; }
virtual bool walk(Item_processor, enum_walk, uchar *);  // 访问者模式（全 Item 树遍历都靠它）
virtual Item *transform(Item_transformer, uchar *arg);
virtual float get_filtering_effect(...);                // 选择率估算
```

**关键数据成员**（这些字段就是"类型元数据"）：

```cpp
DTCollation collation;      // 结果的字符集/排序规则
uint32 max_length;          // 结果的最大字节长度
Item_result cmp_context;    // 比较上下文
bool fixed;                 ///< True if item has been resolved  ★
uint8 decimals;
bool null_value;            ///< True if item is null（动态，执行期）  ★
bool unsigned_flag;
uint8 m_data_type;          ///< Data type（静态，prepare 期算出）  ★
bool m_nullable;            // 是否可能为 NULL（静态）
uint8 m_accum_properties;   // 含子查询/存储程序/聚合/窗口/ROLLUP/GROUPING？
```

### 1.2 完整继承树

```
Parse_tree_node
└── Item                                     ★ 抽象：表达式
    ├── Item_basic_constant                  ★ 字面量/基本常量
    │   ├── Item_num → Item_int / Item_uint / Item_decimal / Item_float
    │   ├── Item_string / Item_varbin / Item_null
    │   └── Item_cache                       ★ 求值缓存（见 2.5）
    │       └── Item_cache_int/real/decimal/str/datetime/json
    │
    ├── Item_ident                           ★ 标识符基类（有名字、有解析上下文）
    │   ├── Item_field                       ★ 真正的表列（见第四章）
    │   └── Item_ref                         ★ 间接引用 Item **m_ref_item
    │       ├── Item_view_ref                view / derived 的列
    │       ├── Item_outer_ref               外层引用（correlated）
    │       └── Item_aggregate_ref           HAVING / ORDER BY 引用聚合结果
    │
    ├── Item_result_field                    ★ 有"落地字段" + 纯虚 resolve_type()
    │   ├── Item_func                        ★ 函数/算子（122 种 Functype）
    │   │   ├── Item_real_func / Item_int_func / Item_str_func
    │   │   ├── Item_bool_func               ★ 布尔函数
    │   │   │   ├── Item_func_true / Item_func_false     ★ 常量折叠的产物
    │   │   │   ├── Item_bool_func2 → Item_func_eq / Item_func_ge ...
    │   │   │   ├── Item_cond → Item_cond_and / Item_cond_or        ★
    │   │   │   ├── Item_equal                ★ 等值传播（MULT_EQUAL_FUNC）
    │   │   │   └── Item_func_in / Item_func_between / Item_func_like ...
    │   │   ├── Item_func_numhybrid → Item_num_op（+ - * /）
    │   │   ├── Item_typecast_*
    │   │   ├── Item_func_case / Item_func_if / Item_func_coalesce
    │   │   └── Item_sum                     ★ 聚合函数（也是一种 Item_func！）
    │   │       ├── Item_sum_num → Item_sum_sum / Item_sum_avg / Item_sum_variance
    │   │       ├── Item_sum_int → Item_sum_count / Item_sum_bit
    │   │       ├── Item_sum_hybrid → Item_sum_min / Item_sum_max
    │   │       └── 窗口函数：Item_sum_row_number / rank / dense_rank / lead_lag / ntile ...
    │   │
    │   └── Item_subselect                   ★ 子查询（一定是 Item 树的叶子）
    │       ├── Item_singlerow_subselect     标量子查询
    │       └── Item_exists_subselect        EXISTS / IN / ALL-ANY
    │           ├── Item_in_subselect        IN (SELECT ...)
    │           └── Item_allany_subselect    = ANY / <> ALL
    │
    ├── Item_row                             (a,b,c) 行表达式，ROW_RESULT
    ├── Item_param                           ★ 预编译参数 ?（见 12 篇）
    ├── Item_sp_variable                     存储过程变量
    └── Item_type_holder                     UNION 等场景的类型合并
```

**几个反直觉的事实**：

- **`Item_sum : public Item_func`**——聚合函数**是一种函数**，它有 `args[]`
- `Item_cond` 虽然是函数，但**不用 `args[]` 而用 `List<Item> list`**（源码有断言："Item_cond never uses args. Inspect list instead"）
- `Item_subselect` **不从 `Item_func` 继承**——它的"参数"是整棵子查询而非 Item 列表。源码注释明确了它的叶子性质：**"item_subselect 在 item tree 中一定是 leaf node"**
- `Item_row` **拒绝被求值**（`val_real()` 直接 `illegal_method_call("val_real")`）

### 1.3 三个重要中间类

**① `Item_ident`——把"有名字、需解析的东西"抽象出来**

```cpp
class Item_ident : public Item {
 protected:
  const char *m_orig_db_name, *m_orig_table_name, *m_orig_field_name;
  bool m_alias_of_expr;         // 名字是否来自 SELECT 表达式的别名
 public:
  Name_resolution_context *context;  // ★ 名字解析作用域
  Query_block *depended_from;        // ★ 非 NULL ⇒ 外层引用（correlated）
};
```

它自己不实现 `val_xxx`（仍纯虚），只提供"三层名字 + 解析上下文 + 相关标记"。

**② `Item_result_field`——有"落地字段"的 Item**

```cpp
class Item_result_field : public Item {
 protected:
  Field *result_field{nullptr}; /* Save result here */
 public:
  table_map used_tables() const override { return 1; }   // 默认"非常量"

  /**
    Resolve type-related information for this item, such as result field type,
    maximum size, precision, signedness, character set and collation.
    Also check compatibility of argument types and return error when applicable.
    Also adjust nullability when applicable.
  */
  virtual bool resolve_type(THD *thd) = 0;      // ★ 类型解析契约
  virtual const char *func_name() const = 0;
};
```

两个角色：**物化落点**（结果写到临时表时落到 `result_field`）+ **类型解析契约**（`resolve_type()` 强制每个函数/聚合/子查询在 fix_fields 阶段声明自己的结果类型）。

> **③ `Item_arena`——8.0 已彻底移除**（全库 0 次匹配）。5.x 里它负责 PS 的 Item 内存 arena，8.0 由 `THD::mem_root` + `Prepared_stmt_arena_holder` + `Item::cleanup()` 取代。

### 1.4 ⚠️ 8.0.39 的命名纠错（容易写成过时知识）

| 老名字 / 常见误解 | 8.0.39 实际情况 |
|---|---|
| `Item_func::fix_length_and_dec()` | **已重命名为 `Item_result_field::resolve_type(THD*)`**（纯虚） |
| `Item::field_type()` | **已重命名为 `Item::data_type()`** |
| `Item_arena` | **8.0 已移除** |
| `Item_equal_fields` | **不存在**——等值传播只有 `Item_equal` + `COND_EQUAL` |
| `Item_direct_ref` | **不存在**——ref 家族只有 `Item_ref` / `Item_view_ref` / `Item_outer_ref` / `Item_aggregate_ref` |
| `type_handler` | **MySQL 8.0 没有**（那是 MariaDB 的框架）。8.0 用 `enum_field_types` + `set_data_type_xxx()` |

---

## 二、求值模型：val_* 协议

### 2.1 为什么是 5 个 `val_xxx()`？

**根本原因：MySQL 没有统一的"值对象"（没有 `Datum`/`Value` 类）。**

每个 Item 内部用自己的原生 C 类型存值（`longlong` / `double` / `String` / `my_decimal`）。如果统一通过一个泛型返回值取，就必然要**装箱**——每行每表达式都装箱，OLTP 场景下不可接受。

所以 MySQL 选择"**调用方按表达式的静态类型，直接调对应的 `val_xxx`**"——零拷贝、零装箱。

**调用方怎么知道调哪个？→ 看 `result_type()`。** 标准范式就是 `Item::val_bool()`：

```cpp
bool Item::val_bool() {
  switch (result_type()) {
    case INT_RESULT:     return val_int() != 0;
    case DECIMAL_RESULT: {
      my_decimal decimal_value;
      my_decimal *val = val_decimal(&decimal_value);
      if (val) return !my_decimal_is_zero(val);
      return false;
    }
    case REAL_RESULT:
    case STRING_RESULT:  return val_real() != 0.0;
    case ROW_RESULT:
    default:             assert(0); return false;
  }
}
```

**`Item_result` 枚举**（定义在 UDF 头文件里，因为它同时是 UDF ABI 的一部分）：

```cpp
enum Item_result {
  INVALID_RESULT = -1,
  STRING_RESULT = 0, REAL_RESULT, INT_RESULT, ROW_RESULT, DECIMAL_RESULT
};
```

**`val_str` 的 buffer 协议**（理解 String 复用的关键）：

```
    Return string representation of this item object.
      str   an allocated buffer this or any nested Item object can use to
            store return value of this method.
    NOTE
      Buffer passed via argument should only be used if the item itself
      doesn't have an own String buffer. In case when the item maintains
      it's own string buffer, it's preferable to return it instead to
      minimize number of mallocs/memcpys.
```

即：调用方传入备用缓冲，被调方**优先返回自己的内部缓冲**避免拷贝；返回 `nullptr` 表示 NULL 或出错。

### 2.2 两层类型：`result_type()`（粗）vs `data_type()`（细）

| | `result_type()` | `data_type()` |
|---|---|---|
| 类型 | `Item_result`（5 值） | `enum_field_types`（几十值） |
| 作用 | 决定**调哪个 `val_xxx`** | 决定**精确 SQL 类型、精度、字符集** |
| 存储 | 子类 override（或 `hybrid_type`） | `uint8 m_data_type` 成员 |

二者可互转（`result_to_type()` / `type_to_result()`）。

**关键**：绝大多数 Item 在**构造时**就通过 `set_data_type_xxx()` 定死了类型，此时 `resolve_type()` 几乎无事可做。只有 `+ - * /` 这类"结果类型依赖参数"的函数，才需要在 `resolve_type()` 里真正做推导。

### 2.3 `set_data_type_xxx()`：元数据一次算全

```cpp
  inline void set_data_type_longlong() {
    set_data_type(MYSQL_TYPE_LONGLONG);
    collation.set_numeric();
    fix_char_length(21);          // 20 位数字 + 符号
  }
  inline void set_data_type_decimal(uint8 precision, uint8 scale) {
    set_data_type(MYSQL_TYPE_NEWDECIMAL);
    collation.set_numeric();
    decimals = scale;
    fix_char_length(my_decimal_precision_to_length_no_truncation(precision, scale, unsigned_flag));
  }
  inline void set_data_type_string(uint32 max_l) {
    max_length = max_l * collation.collation->mbmaxlen;
    decimals = DECIMAL_NOT_SPECIFIED;
    if (max_length <= Field::MAX_VARCHAR_WIDTH)      set_data_type(MYSQL_TYPE_VARCHAR);
    else if (max_length <= Field::MAX_MEDIUM_BLOB_WIDTH) set_data_type(MYSQL_TYPE_MEDIUM_BLOB);
    else set_data_type(MYSQL_TYPE_LONG_BLOB);
  }
```

**每次 set 都同时设置 `data_type` + `collation` + `decimals` + `max_length`**——这就是"类型元数据"的完整内容。

### 2.4 类型推导：数值运算的提升规则

`Item_num_op::set_numeric_type()` 很短，但它是 MySQL 数值类型聚合的核心：

```cpp
void Item_num_op::set_numeric_type(void) {
  Item_result r0 = args[0]->numeric_context_result_type();
  Item_result r1 = args[1]->numeric_context_result_type();

  if (r0 == REAL_RESULT || r1 == REAL_RESULT) {
    set_data_type(MYSQL_TYPE_DOUBLE);
    hybrid_type = REAL_RESULT;
    aggregate_float_properties(args, arg_count);
    max_length = float_length(decimals);
  } else if (r0 == DECIMAL_RESULT || r1 == DECIMAL_RESULT) {
    set_data_type(MYSQL_TYPE_NEWDECIMAL);
    hybrid_type = DECIMAL_RESULT;
    result_precision();
  } else {
    set_data_type(MYSQL_TYPE_LONGLONG);
    decimals = 0;
    hybrid_type = INT_RESULT;
    result_precision();
  }
}
```

**规则**（可直接记）：

```
REAL    参与           ⇒ DOUBLE
DECIMAL 参与且无 REAL  ⇒ NEWDECIMAL
否则                   ⇒ LONGLONG
```

注意用的是 `numeric_context_result_type()` 而非 `result_type()`——这是为了让 `TIME`/`DATETIME` 在数值上下文里表现为整数。

**精度推导**（`+`/`-` 的 DECIMAL 例子）：

```cpp
void Item_func_additive_op::result_precision() {
  decimals = max(args[0]->decimals, args[1]->decimals);
  int arg1_int = args[0]->decimal_precision() - args[0]->decimals;
  int arg2_int = args[1]->decimal_precision() - args[1]->decimals;
  int precision = max(arg1_int, arg2_int) + 1 + decimals;   // +1 是进位保护位
  ...
}
```

**多操作数合并**（UNION / CASE / COALESCE）用 `Item::aggregate_type()`：逐对 `Field::field_type_merge()`，`decimals` 取 max，**符号混用时整数类型逐级升档**：

```
TINY → SHORT → INT24 → LONG → LONGLONG → DECIMAL
```

### 2.5 NULL 处理：`null_value` 与 `is_null()`

**为什么不用"返回特殊值"？** 因为 C 原生类型里没有"能表示 NULL 的 longlong/double"——任何取值都可能是合法值。所以 MySQL 用**副作用标志位**（输出参数式）：

```cpp
  bool null_value;  ///< True if item is null（public 成员，直接读）
 private:
  bool m_nullable;  // 静态可空性（prepare 期算）
```

协议（源码注释原文）：*"`valXXX` methods must return NULL or 0 or 0.0 if `null_value` is set."*

典型实现：

```cpp
longlong Item_field::val_int() {
  assert(fixed == 1);
  if ((null_value = field->is_null())) return 0;
  return field->val_int();
}
```

**`m_nullable`（静态）vs `null_value`（动态）的区分很重要**，而且源码特意警告 `m_nullable` 会失效：

```
    It is worth noting that this information is correct only until
    equality propagation has been run by the optimization phase.
    Indeed, consider:
     select * from t1, t2,t3 where t1.pk=t2.a and t1.pk+1...
    the '+' is not nullable as t1.pk is not nullable;
    but if the optimizer chooses plan is t2-t3-t1, then, due to equality
    propagation it will replace t1.pk in '+' with t2.a (as t2 is before t1
    in plan), making the '+' capable of returning NULL when t2.a is NULL.
```

**`is_null()` 是"短路判空"优化**——用于 `IS NULL` / `COUNT(x)`，避免完整求值：

```
  /**
    The method allows to determine nullness of a complex expression
    without fully evaluating it, instead of calling val*() then
    checking null_value. ...
    Any item which can be NULL must implement this method.
  */
```

### 2.6 求值缓存 `Item_cache_*`

**延迟求值（lazy）设计**：

```cpp
class Item_cache : public Item_basic_constant {
 protected:
  Item *example{nullptr};         // 被缓存的表达式
  /*
    true <=> cache holds value of the last stored item (i.e actual value).
    store() stores item to be cached and sets this flag to false.
    On the first call of val_xxx function if this flag is set to false the
    cache_value() will be called to actually cache value of saved item.
  */
  bool value_cached{false};
 public:
  Item_cache() { fixed = true; set_nullable(true); null_value = true; }
  virtual bool cache_value() = 0;
};
```

三个设计点：

1. **cache 对外表现为一个"常量"**（继承 `Item_basic_constant`）
2. **延迟求值**——`store()` 只记下表达式，`val_xxx` 首次调用时才真正求值
3. **构造时即 `fixed = true`**——cache 不走 `fix_fields()`

使用场景：标量子查询的结果（`Item_singlerow_subselect` 用 `Item_cache` 存值）、HAVING 引用聚合结果、比较算子常量侧的缓存（`Arg_comparator`）。

---

## 三、fix_fields：两阶段绑定

### 3.1 阶段 1：contextualize / itemize（parse 后）

MySQL 8.0 的 parser 产物是 `Parse_tree_node` 树。Item 把 `contextualize()` **私有化隐藏**，改用 `itemize()`：

```cpp
 private:
  /* Hide the contextualize*() functions: call/override the itemize()
     in Item class tree instead. */
  bool contextualize(Parse_context *) override { assert(0); return true; }
 public:
  virtual bool itemize(Parse_context *pc, Item **res);
```

**为什么要有 `itemize()`？** 因为 Item 的构造函数分两类：`Item(POS)` 是"**上下文无关**"的构造函数（parser 里可以随便 new），所有需要 `thd->lex` 的收尾工作挪到 `itemize()`。这是 8.0 为让 parser 与 THD 解耦做的重构。

### 3.2 阶段 2：fix_fields（resolve 期）

触发点在 `Query_block::prepare()`，且注意有个 **`resolve_place` 状态机**——它决定了 `SUM()` 在 WHERE 里非法、在 HAVING 里合法。

调用形态几乎全库统一：

```cpp
    if ((!item->fixed && item->fix_fields(thd, item_pos)) ||
        (item = *item_pos)->check_cols(1)) {
      return true;
    }
```

**`Item **` 二级参数是关键**：`fix_fields` 有权**替换整棵子树**（如 `Item_field` → `Item_ref`、`Item_in_subselect` → `Item_in_optimizer`），调用方必须用返回的 `*ref`。

### 3.3 `fix_fields()` 到底做哪七件事

以 `Item_func::fix_fields` 为模板（标准后序遍历：递归参数 → 传播属性 → 类型推导 → 置位）：

```cpp
bool Item_func::fix_fields(THD *thd, Item **) {
  assert(fixed == 0 || basic_const_item());
  used_tables_cache = get_initial_pseudo_tables();
  not_null_tables_cache = 0;
  ...
  if (arg_count) {
    for (arg = args, arg_end = args + arg_count; arg != arg_end; arg++) {
      if (fix_func_arg(thd, arg)) return true;      // ① 递归 fix 子节点
    }
  }
  if (resolve_type(thd) || thd->is_error()) return true;   // ③ 类型推导
  fixed = true;
  return false;
}

bool Item_func::fix_func_arg(THD *thd, Item **arg) {
  if ((!(*arg)->fixed && (*arg)->fix_fields(thd, arg))) return true;
  ...
  set_nullable(is_nullable() || item->is_nullable());    // 可空性向上聚合
  used_tables_cache |= item->used_tables();              // ★ used_tables 向上传播
  if (null_on_null) not_null_tables_cache |= item->not_null_tables();
  add_accum_properties(item);                            // ★ 属性（subquery/agg/wf）向上传播
  return false;
}
```

完整清单：

1. **递归 fix 子节点**
2. **名字解析**——`Item_field` 调 `find_field_in_tables()` 找 `Field`；`Item_ref` 解析别名/外层引用
3. **类型推导**——`resolve_type(THD*)`（原 `fix_length_and_dec`）
4. **属性向上传播**——`used_tables_cache |=`、`not_null_tables_cache |=`、`set_nullable()`、`add_accum_properties()`
5. **常量折叠与语义改写**——`Item_cond::fix_fields` 的 `remove_const_conds`；子查询的 `substitution`
6. **权限检查**——`Item_field::fix_fields` 里 `find_field_in_tables(..., thd->want_privilege, ...)`。**列权限检查就在这里**
7. **置 `fixed = true`**

### 3.4 `resolve_type()`（原 `fix_length_and_dec()`）

契约（源码注释）：*"Resolve type-related information for this item, such as result field type, maximum size, precision, signedness, character set and collation. Also check compatibility of argument types..."*

典型实现（数值类）：

```cpp
bool Item_func_numhybrid::resolve_type(THD *thd) {
  // 处理 PS 参数 ? 的类型未知情况 —— 推迟/传播
  if (arg_count == 1) {
    if (args[0]->data_type() == MYSQL_TYPE_INVALID) return false;
  } else {
    if (args[0]->data_type() == MYSQL_TYPE_INVALID &&
        args[1]->data_type() == MYSQL_TYPE_INVALID)
      return false;                    // 全是 ? 参数 → 推迟到 propagate_type()
    ...
  }
  if (resolve_type_inner(thd)) return true;
  return reject_geometry_args(arg_count, args, this);   // 参数兼容性检查
}

bool Item_func_numhybrid::resolve_type_inner(THD *) {
  fix_num_length_and_dec();     // 算 decimals / max_length
  set_numeric_type();           // 算 data_type + result_type
  return false;
}
```

**返回类型固定的函数**（绝大多数）类型在构造函数里就定死了，`resolve_type` 只处理 PS 参数：

```cpp
class Item_typecast_real final : public Item_func {
 public:
  Item_typecast_real(const POS &pos, Item *a, bool as_double) : Item_func(pos, a) {
    if (as_double) set_data_type_double();
    else           set_data_type_float();
  }
  bool resolve_type(THD *thd) override {
    if (param_type_is_default(thd, 0, 1, MYSQL_TYPE_DOUBLE)) return true;
    return false;
  }
};
```

### 3.5 `fixed` 标志为什么必要

`fixed == true` 表示"**这个 Item 已完成 resolve，它的 `data_type`/`collation`/`max_length`/`used_tables` 全部有效，且 `val_xxx` 可以被安全调用**"。

**所有 `val_xxx` 实现的第一行都是 `assert(fixed == 1)`**——这是全库最一致的断言。

重复 fix_fields 会怎样？

1. **`used_tables_cache` 会重复叠加**（它是 `|=` 累积的）⇒ 错误的 `const_item()` 判断
2. **重复做常量折叠 / 子树替换**⇒ 直接破坏 Item 树
3. **PS 二次执行**：prepared statement 第二次执行时 `fix_fields` 会被再次调用（表可能变了），所以大量代码有 `select->first_execution` 守卫

有个特殊后门：`quick_fix_field()`（`inline void quick_fix_field() { fixed = true; }`），用于"确定不需要完整 fix_fields 流程"的场景。

### 3.6 `const_item()` 与常量折叠：`WHERE 1=0` 的完整流程

**判定就是一行**：

```cpp
  bool const_item() const { return (used_tables() == 0); }
  bool const_for_execution() const { return !(used_tables() & ~INNER_TABLE_BIT); }
```

即：**"常量性"完全是通过 `used_tables()` 的位传播算出来的**，不是靠什么 `is_constant` 标志。

- `Item::used_tables()` 基类返回 0 ⇒ 普通 Item 默认就是常量
- `Item_field` 返回 `table_ref->map()`（const table 返回 0）
- 三个伪表位让"非常量"不被误判为常量（见 4.2）

**常量折叠在 `Item_cond::fix_fields` 里**：

```cpp
    if (ref != nullptr && select->first_execution && item->const_item() &&
        !item->walk(&Item::is_non_const_over_literals, ...) && ...) {
      if (remove_const_conds(thd, item, &new_item)) return true;
      if (new_item != nullptr) { remove_condition = true; continue; }
      ...
      li.remove();        // ★ 只删掉这个常量项
      continue;
    }
```

```cpp
bool Item_cond::remove_const_conds(THD *thd, Item *item, Item **new_item) {
  const bool and_condition = functype() == Item_func::COND_AND_FUNC;
  bool cond_value = true;
  bool err = eval_const_cond(thd, item, &cond_value);      // ★ 真的求值了！
  ...
  if (cond_value) {
    if (!and_condition || (argument_list()->elements == 1))
      *new_item = new Item_func_true();                    // OR 里出现 TRUE → 整体 TRUE
  } else {
    if (and_condition || (argument_list()->elements == 1))
      *new_item = new Item_func_false();                   // AND 里出现 FALSE → 整体 FALSE
  }
```

**`WHERE 1=0` 的完整流程**：

```
WHERE 1=0
  → parser:      Item_func_eq(Item_int(1), Item_int(0))
  → fix_fields:  递归 fix 两个 Item_int → resolve_type
                 两个参数 used_tables()==0 ⇒ Item_func_eq::used_tables_cache == 0
                 ⇒ const_item() == true
  → remove_const_conds 触发 → eval_const_cond() → val_bool() → val_int() → 0
  → cond_value == false, and_condition == true ⇒ new_item = new Item_func_false()
  → remove_condition = true ⇒ 清空 list，push_front(Item_func_false)
  → 最终 WHERE 变成 Item_func_false，优化器后续直接判 Impossible WHERE
```

---

## 四、Item 与 Field

### 4.1 `Item_field`：Item 世界与 Field 世界的唯一桥接点

绑定动作就是 `set_field()`：

```cpp
void Item_field::set_field(Field *field_par) {
  table_ref = field_par->table->pos_in_table_list;
  field = result_field = field_par;

  set_nullable(field->is_nullable() || field->is_tmp_nullable() ||
               field->table->is_nullable());
  collation.set(field_par->charset(), field_par->derivation(), field_par->repertoire());
  set_data_type(field_par->type());           // ★ data_type 从 Field 拷过来
  decimals = field->decimals();
  unsigned_flag = field_par->is_flag_set(UNSIGNED_FLAG);
  max_length = char_to_byte_length_safe(field_par->char_length(),
                                        collation.collation->mbmaxlen);
  ...
  fixed = true;                               // ★★ set_field 直接置 fixed
}
```

`result_type()` 与 `val_xxx()` 都是**纯转发**给 Field：

```cpp
  Item_result result_type() const override { return field->result_type(); }
  longlong val_int() {
    assert(fixed == 1);
    if ((null_value = field->is_null())) return 0;
    return field->val_int();
  }
```

完整链路：

```
SQL 文本 "t.a"
  → parser:    Item_field(table="t", field="a")   ← 只有名字，field == nullptr
  → fix_fields: find_field_in_tables() → set_field(Field *f)
                  data_type = f->type()
                  collation = (f->charset(), f->derivation(), f->repertoire())
                  decimals  = f->decimals()
                  max_length= f->char_length() * mbmaxlen
                  fixed = true
  → 执行期:    val_int() / val_str() → field->val_xxx()（Field 从 record buffer 读）
```

### 4.2 `used_tables()` 与三个伪表位

这三个位**放在真实表位（0..MAX_TABLES-1）之后**，不占用表位，但能让 `used_tables() != 0`，从而 `const_item() == false`：

| 伪位 | 含义 | 谁设置 | 效果 |
|---|---|---|---|
| `INNER_TABLE_BIT` | 含参数 / 访问表的子查询 / 访问表的函数 | `Item_param`、`Item_subselect`、存储程序变量 | `const_item()`=false，但 **`const_for_execution()`=true**（锁表后求值一次即可） |
| `OUTER_REF_TABLE_BIT` | 含外层引用 | `depended_from != nullptr` 的 `Item_field`/`Item_ref` | 子查询内恒定但对外层每行变 ⇒ **不能提前求值、不能下推** |
| `RAND_TABLE_BIT` | 非确定性（RAND/UUID/NOW） | `get_initial_pseudo_tables()` | **永远不能缓存、必须尽量晚求值** |

```cpp
constexpr const table_map INNER_TABLE_BIT{1ULL << (MAX_TABLES + 0)};
constexpr const table_map OUTER_REF_TABLE_BIT{1ULL << (MAX_TABLES + 1)};
constexpr const table_map RAND_TABLE_BIT{1ULL << (MAX_TABLES + 2)};
```

`Item_func` 用 `get_initial_pseudo_tables()` 声明自己自带的伪位（`NOW()`、`RAND()` 返回 `RAND_TABLE_BIT`）。

**伪位的灵活运用**（`Item_insert_value` 用它"防折叠"）：

```cpp
  /*
   We use RAND_TABLE_BIT to prevent Item_insert_value from
   being treated as a constant and precalculated before execution
  */
```

### 4.3 `Item_ref` 家族：二级指针

**为什么用 `Item **m_ref_item` 二级指针？** 因为被引用者的位置（`Query_block::base_ref_items[i]` 或 SELECT list 的槽位）本身可能被改写。二级指针让 `Item_ref` **自动跟随**，无需手工同步。

| 子类 | 角色 |
|---|---|
| `Item_view_ref` | view / derived table 的列；**会主动下推 `fix_fields()`** |
| `Item_outer_ref` | 相关外层引用，构造时 `fixed = false`（外层还没解析） |
| `Item_aggregate_ref` | **HAVING / ORDER BY 引用聚合结果**；`depended_from` 标为跨层引用 |

`Item_ref` 还有引用计数 `m_ref_count`，用于"无用表达式消除"——只有引用计数归零才能真正删除一个 Item 子树。

---

## 五、特殊 Item 子类

| 子类 | 要点 |
|---|---|
| **`Item_param`** | 预编译参数。**唯一 `actual_data_type() != data_type()` 的 Item**（前者=实际传值类型，后者=期望类型）；实现 `propagate_type()` 接收上下文类型；`used_tables()` 返回 `INNER_TABLE_BIT`。详见 [`12_prepared_statement.md`](12_prepared_statement.md) |
| **`Item_subselect`** | 一定是叶子节点。`fix_fields()` 里会**递归 `subquery->prepare()`**；`substitution` 机制允许整个子查询被替换成别的 Item（如 `Item_in_optimizer`） |
| **`Item_sum`** | 聚合函数（也是一种 `Item_func`）。**Aggregator 策略模式**把 DISTINCT 从聚合算法里解耦：`Aggregator_simple` 直通，`Aggregator_distinct` 用临时表树 / `Unique` 去重。窗口函数**不用 Aggregator、不支持 DISTINCT、且 `val_*()` 会先累加再返回** |
| **`Item_equal`** | 等值传播的核心。`Item_equal(f1,f2,...fk)` 表示多重等值，用于：①提供额外的表间访问路径（join transitive closure）②推导新的 sargable 谓词 ③执行计划优化时替换列引用。`Item_cond_and` 持有 `COND_EQUAL` |
| **`Item_cond_and/or`** | 条件树（用 `List<Item>` 而非 `args[]`）。`get_filtering_effect()` 用独立性假设连乘：AND 是 `P(A)*P(B)`，OR 是 `P(A)+P(B)-P(A)*P(B)`。默认常量 `COND_FILTER_EQUALITY=0.1`、`INEQUALITY=0.3333`、`BETWEEN=0.1111` |
| **`Item_in_subselect`** | 继承关系是 `Item_in_subselect : Item_exists_subselect : Item_subselect`——**IN 是一种 EXISTS**（因为 IN→EXISTS 是最常见的改写方向） |

### `Item_equal` 与等价类：等值传播的完整机制

源码里的类注释本身就是一篇小文档，值得整段读：

```
/*
  All equality predicates of the form field1=field2 contained in a
  conjunction are substituted for a sequence of items of this class.
  An item of this class Item_equal(f1,f2,...fk) represents a
  multiple equality f1=f2=...=fk.
  ...
  Objects of the class Item_equal are used for the following:
  1. An object Item_equal(t1.f1,...,tk.fk) allows us to consider any
     pair of tables ti and tj as joined by an equi-condition.
     Thus it provide us with additional access paths from table to table.
  2. An object Item_equal(t1.f1,...,tk.fk) is applied to deduce new
     SARGable predicates:
       f1=...=fk AND P(fi) => f1=...=fk AND P(fi) AND P(fj).
  3. An object Item_equal(t1.f1,...,tk.fk) is used to optimize the
     selected execution plan for the query: if table ti is accessed
     before the table tj then in any predicate P in the where condition
     the occurrence of tj.fj is substituted for ti.fi.
*/
```

三个用途，逐个展开：

**① join transitive closure（连接传递闭包）**
`WHERE t1.a = t2.a AND t2.a = t3.a` 会合成一个等价类 `{t1.a, t2.a, t3.a}`。之后**任意两表之间都被视为有等值连接条件**——即使原查询没写 `t1.a = t3.a`，ref 访问也能从 t1 直接定位 t3。多表 join 的可达路径因此变多。

**② sargable transitive closure（sargable 传递闭包）**
`f1=...=fk AND P(fi) ⇒ f1=...=fk AND P(fi) AND P(fj)`。比如 `t1.a = t2.a AND t2.a > 10` 推导出 `t1.a > 10`——**让原本只能全表扫的 `t1.a` 也能走 range**。这是"看起来没用索引，其实索引被等值传播救活了"的根因。

**③ 计划期列替换**
如果执行计划里 t1 在 t2 之前，就把谓词里的 `t2.a` 替换成 `t1.a`——**让谓词在"它引用的行已读到"的时刻才被求值**（t2 的行还没读到时引用 `t2.a` 是无意义的）。这也是为什么交换 join 顺序后"WHERE 条件用的列"看起来会变。

**结构**：

```cpp
class Item_equal final : public Item_bool_func {
  /// List of equal field items.
  List<Item_field> fields;
  /// Optional constant item equal to all the field items.
  Item *m_const_arg{nullptr};
  ...
  enum Functype functype() const override { return MULT_EQUAL_FUNC; }
  const char *func_name() const override { return "multiple equal"; }
  optimize_type select_optimize(const THD *) override { return OPTIMIZE_EQUAL; }
};

class COND_EQUAL {
 public:
  uint max_members;                 // 当前层及所有低层的最大成员数
  COND_EQUAL *upper_levels;         // ★ 上层的多重等值
  List<Item_equal> current_level;   // 当前 AND 层的多重等值列表
};
```

**跨层传播**：`COND_EQUAL::upper_levels` 把当前 AND 层的等价类链到上层——嵌套 `((a=b) AND (b=c))` 会逐层合并成 `{a,b,c}`。`Item_cond_and` 持有 `cond_equal` 成员；`Item_field` 上有**反向指针** `item_equal`（由等值类构建阶段设置），用于 O(1) 找到自己所属的等价类。

**它通常不被求值**（源码注释）：

```
  Item equal objects are employed only at the optimize phase. Usually they are
  not supposed to be evaluated. Yet in some cases we call the method val_int()
  for them. ...
```

——**`Item_equal` 是优化期的元数据，不是运行期谓词**。`val_int()` 只在少数场景（没有 access path 兜底时）被调。

### unsigned 陷阱：类型聚合里的符号规则

符号在类型聚合里的规则**不对称**（`Item_func_additive_op::result_precision()`）：

```cpp
  /* Integer operations keep unsigned_flag if one of arguments is unsigned */
  if (result_type() == INT_RESULT)
    unsigned_flag = args[0]->unsigned_flag | args[1]->unsigned_flag;
  else
    unsigned_flag = args[0]->unsigned_flag & args[1]->unsigned_flag;
```

- **INT**：`|`——**任一**参数 unsigned，结果就 unsigned
- **DECIMAL**：`&`——必须**双方**都 unsigned 才 unsigned

**陷阱**：两个 `INT UNSIGNED` 相减，结果仍是 `INT UNSIGNED`——**负数会回绕**：

```sql
SELECT CAST(1 AS UNSIGNED) - CAST(2 AS UNSIGNED);
-- 18446744073709551615   （不是 -1）
```

这是 SQL 的 signedness 传播规则，但对用户极其反直觉。要靠 `sql_mode` 救：

```cpp
bool Item_func_minus::resolve_type(THD *thd) {
  if (Item_num_op::resolve_type(thd)) return true;
  /**
    The following function is here to allow the user to force
    subtraction of UNSIGNED BIGINT/DECIMAL to return negative values.
  */
  if (unsigned_flag && (thd->variables.sql_mode & MODE_NO_UNSIGNED_SUBTRACTION))
    unsigned_flag = false;
  return false;
}
```

**另一个陷阱在 UNION / CASE 的类型合并**（`Item::aggregate_type()`）：有符号/无符号**混用**时不是简单取 signed，而是整数类型**逐级升档**保住范围：

```
TINY → SHORT → INT24 → LONG → LONGLONG → DECIMAL
```

升到 LONGLONG 仍有混用就升到 DECIMAL——用"多一位表示符号"换取不溢出。这就是为什么 `UNION` 一个 `INT` 和一个 `INT UNSIGNED` 出来的是 `BIGINT`。



---

## 六、聚合函数状态机：`Item_sum` 与 `Aggregator`

聚合函数是 Item 体系里**唯一有执行期状态**的一类——普通 Item 是无状态的纯函数，而聚合要跨行累积。它的状态机由三件套驱动：

```cpp
  virtual void clear() = 0;   // 开始新的一组
  virtual bool add() = 0;     // 累加一行
  virtual void endup() = 0;   // 全部加完，算出最终结果
```

### ★ `endup()` 是"泵回"的

`endup()` **不由聚合循环显式调用，而是在 `val_xxx()` 首次被调用时才触发**——源码里大量出现这个形态：

```cpp
  if (aggr) aggr->endup();      // 出现在各 Item_sum::val_xxx() 里
```

配合 `Aggregator_distinct::endup_done` 标志避免重复计算：

```
    flag to prevent consecutive runs of endup(). Normally in endup there are
    expensive calculations (like walking the distinct tree for example)
    which we must do only once if there are no data changes.
    We can re-use the data for the second and subsequent val_xxx() calls.
```

**即：`val_xxx()` → 首次触发 `endup()`（泵一下）→ 之后靠 `endup_done` 复用。** 这是聚合"惰性收尾"的实现，也是为什么 `endup()` 里那些昂贵计算（遍历去重树）只会做一次。

### Aggregator：把 DISTINCT 从聚合算法里解耦

`Item_sum` 只实现"无 DISTINCT"的语义，DISTINCT 由 **`Aggregator`** 策略类在外面套一层：

| Aggregator | 行为 |
|---|---|
| `Aggregator_simple` | 直通——`setup`/`clear`/`add` 直接转发给 `item_sum` |
| `Aggregator_distinct` | 在 `add()` 前查重；`endup()` 时走树/临时表算出最终结果 |

`Aggregator_distinct` 的物理载体按场景分两种（源码注释）：

```
    - For COUNT(DISTINCT) and no blob fields this points to a real temporary
      table. It's used as a hash table.
    - For AVG/SUM(DISTINCT) or COUNT(DISTINCT) with blob fields only the
      in-memory data structure of a temporary table is constructed.
```

即：能用真实临时表当 hash 表就 hash 去重，否则退化成内存树结构。

### ★ COUNT(DISTINCT) 的 O(1) 特例

`COUNT(DISTINCT)` 通常要建去重结构，但有一条**完全不需要任何数据结构的快捷路径**——当参数是"常量且非 NULL"时：

```cpp
bool Aggregator_distinct::add() {
  if (const_distinct == CONST_NULL) return false;

  if (item_sum->sum_func() == Item_sum::COUNT_FUNC ||
      item_sum->sum_func() == Item_sum::COUNT_DISTINCT_FUNC) {
    if (const_distinct == CONST_NOT_NULL) {
      assert(item_sum->fixed == 1);
      Item_sum_count *sum = (Item_sum_count *)item_sum;
      sum->count = 1;                    // ★ O(1)：直接定成 1，不建任何结构
      return false;
    }
    ...
```

对比普通 `Item_sum_count::add()`——它**本身就是 O(1)**：

```cpp
bool Item_sum_count::add() {
  assert(!m_is_window_function);
  if (aggr->arg_is_null(false)) return current_thd->is_error();
  count++;                               // ★ 就是个计数器
  return current_thd->is_error();
}
```

**这是理解聚合代价的关键对比**：

| 形式 | 空间 | 每行代价 |
|---|---|---|
| `COUNT(*)` / `COUNT(非空列)` | **O(1)**（一个计数器） | `count++` |
| `COUNT(DISTINCT x)` | **O(n)**（hash 表或树） | 一次查重 + 可能插入 |

即：**`COUNT(*)` 从不存储任何行；而 `COUNT(DISTINCT x)` 必须记住每个见过的 `x`，代价从 O(1) 跃迁到 O(n) 空间 + 每行一次查找。** 这是 SQL 里最容易被忽视的代价跃迁之一。

### 窗口函数不用 Aggregator

`Item_sum` 用作窗口函数时有三个差异（源码头部注释）：

- **不使用任何 `Aggregator`**
- **不支持 DISTINCT**
- **`val_*()` 不只是返回当前值——它先把参数累加进函数状态**

即：普通聚合是"`add()` 累加 / `val_xxx()` 取值"分离，而窗口函数把**累加和求值合并在 `val_xxx()` 里**——因为窗口函数的输出是逐行的，每行的 frame 不同。

---

## 七、参数

与 Item / 表达式直接相关的参数不多，主要有：

| 参数 | 默认 | 说明 |
|---|---|---|
| `div_precision_increment` | **4** | `/` 运算结果的小数位增量 |
| `max_allowed_packet` | 64MB | 间接约束表达式结果的字符串长度 |

表达式求值的**类型规则**主要由 SQL 标准与 `sql_mode` 决定（如 `MODE_NO_UNSIGNED_SUBTRACTION` 会让无符号减法返回负数，见 `Item_func_minus::resolve_type`）。

## 参考

- 源码：`class Item`（`sql/item.h` 的类注释是本篇最好的开篇引文）、`Item_func` / `Item_field` / `Item_ref` / `Item_sum` / `Item_subselect` / `Item_param`
- 源码：`Item_func::set_numeric_type()`、`Item::aggregate_type()`、`Item_cond::remove_const_conds()`、`Item_field::set_field()`
- MySQL 8.0 Reference Manual → *Type Conversion in Expression Evaluation*
- MySQL Internals Manual → *Item class*（社区维护，版本偏旧，注意与 8.0 的命名差异，见 1.4 纠错表）
- 相关篇：[`06_resolver_prepare.md`](06_resolver_prepare.md)（fix_fields 的调用现场）、[`07_optimize/logical/05_logical_predicate.md`](07_optimize/logical/05_logical_predicate.md)（等值传播）、[`12_prepared_statement.md`](12_prepared_statement.md)（`Item_param` 与 PS）
