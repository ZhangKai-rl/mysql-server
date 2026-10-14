# 13 Item 表达式体系：MySQL 一切可求值之物的抽象

> `Item` 是 MySQL 对"**表达式**"的统一抽象——字面量、列引用、函数、聚合、子查询、条件、`?` 参数，SQL 里一切能求值之物都是 `Item`。它是解析器、优化器、执行器三层共用的数据结构，也是理解"SQL 到底是怎么被求值的"必经之路。
>
> 本篇是体系篇，把此前分散在 resolver、优化器各篇的 Item 相关内容串成一棵树。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象：一条 SQL 表达式的一生](#先有个整体印象一条-sql-表达式的一生)（★ `SELECT ABS(t.a)+t.b*2 WHERE t.c>5` 的三阶段全链路：解析 / prepare / 执行）
- [一、Item 类继承树](#一item-类继承树)
- [二、求值模型：val_* 协议](#二求值模型val_-协议)
- [三、fix_fields：两阶段绑定](#三fix_fields两阶段绑定)
- [四、Item 与 Field](#四item-与-field)
- [五、特殊 Item 子类](#五特殊-item-子类)
  - [5.x `Item_func_case`](#item_func_case-多分支类型合并ps-参数定形与比较器缓存)（★ `args[]` 扁平编码两种语法 / swap 临时换数组定形 / 两轮类型合并 / `value_added_map` 比较器缓存）
- [六、聚合函数状态机：Item_sum 与 Aggregator](#六聚合函数状态机item_sum-与-aggregator)
  - [6.5 Sumfunctype（25 值）](#65-sumfunctype25-个值)（★ 与 `functype()` 正交；DISTINCT 不是独立类）
  - [6.6 各聚合实现要点](#66-各聚合的实现要点)（★ `COUNT(*)` 伪装成 `COUNT(0)`；★ `SUM(int)` 走 DECIMAL；VAR 的两种递推算法）
  - [6.7 ROLLUP 双实现](#67--rollup-的双实现此前完全没提)（★ switcher + `real_sum_func()` + `SetRollupLevel` 同时切两件事）
  - [6.8 临时表契约与伴生类](#68-临时表契约与伴生类)（★ 为什么 AVG/VAR/STD/BIT 需要伴生 Item）
- [七、★ Arg_comparator：比较的类型推导与函数选择](#七-arg_comparator比较的类型推导与函数选择)
  - [7.3 类型推导 item_cmp_type](#73--类型推导item_cmp_typeitemcc)（四条规则）
  - [7.4 set_cmp_func 的七步决策](#74--set_cmp_func-的七步决策item_cmpfunccc1132-1250)（★ JSON 短路 / 日期短路 + 常量缓存 / TIME / 字符集聚合 / DECIMAL 精度保护）
  - [7.5 comparator_matrix](#75-set_compare_func-与-comparator_matrix)（★ 表定义在 `mysqld.cc:1050`）
  - [7.8 实践含义速查](#78-实践含义速查)
- [八、★ IN 谓词、子查询与缓存](#八-in-谓词子查询与缓存)
  - [8.2 常量列表二分](#82-①-常量列表二分bisection_possible-的四条件)（★ `in_vector` 体系、`fill()` 跳过 NULL、`in_longlong` 符号陷阱）
  - [8.4 NULL 语义速查](#84--null-语义速查)（★ `NOT IN` 遇 NULL 为什么返回空）
  - [8.6 IN (SELECT) 的三个引擎与两阶段决策](#86-③-in-select-三个引擎与两阶段决策)（★ 物化前要**拆掉** in2exists 注入的条件）
  - [8.8 Item_cache 9 个类](#88--item_cache-9-个类)（★ `set_used_tables(1)` 防再缓存的完整原理）
  - [8.9 四个 0 覆盖的特殊类](#89-四个此前-0-覆盖的特殊-item-类)（`Item_splocal` / `Item_trigger_field` / `Item_default_value` / `Item_insert_value`）
- [九、★ 内置函数：从"名字 → 对象"到"对象 → 结果"](#九-内置函数从名字--对象到对象--结果)
  - [9.2 13 个 SQL_FN_* 宏](#92--13-个-sql_fn_-宏不是-12-个还有一个-sql_factory)（完整表，含 ODD/EVEN/INTERNAL 系列）
  - [9.3 Function_factory + Instantiator 模板体系](#93--function_factory--instantiator这套模板解决什么问题)（★ 注册 = 取一个静态单例地址；参数校验是编译期常量）
  - [9.5 UDF 接入](#95--udf-接入)（★ 8 个类 = 2×4；**UDF 无参数个数检查**）
  - [9.6 加一个内置函数要动几处](#96--加一个内置函数要动几处扩展点清单)（★ 只需 3 处，`sql_yacc.yy` 不用动）
- [十、参数](#十参数)
  - [9.8 ★ `Item_func` 实现骨架：函数被创建之后怎么算出结果](#98--item_func-实现骨架函数被创建之后怎么算出结果)
  - [9.9 ★ 函数族谱：各大族代表实现 + `Item_typecast_*` + `Item_cache_*`](#99--函数族谱各大族代表实现--item_typecast_--item_cache_)
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
| **`type()`** | **30** 个 `Type` 枚举值 | **运行时类别标签**——`FIELD_ITEM` / `FUNC_ITEM` / `SUM_FUNC_ITEM` / `SUBSELECT_ITEM` / `PARAM_ITEM` ... 用于"这是个什么东西"的判断与 `down_cast` 断言 |
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

## 先有个整体印象：一条 SQL 表达式的一生

> 本章用一条具体 SQL 把后面十章串起来——**先建立全貌，再深入各章细节**。
> 本章只给骨架与关键代码；每阶段的完整机制见：解析 → [`05_contextualize.md`](05_contextualize.md)，绑定 → 本文第三章，求值 → 本文第二章。

**示例 SQL**：

```sql
SELECT ABS(t.a) + t.b * 2 FROM t WHERE t.c > 5
```

### 目标：★ 一个 SQL 不是"一棵 Item 树"，而是"若干棵树挂在 `Query_block` 的槽位上"

★ **最常见的误解**：以为一个查询有一棵统一的表达式树。**没有**——`Item` 分散挂在 `Query_block`（旧名 `SELECT_LEX`）的**多个彼此独立的槽位**上，槽位之间没有任何 Item 节点相连。真正的"根"是 `Query_block` 这个**结构容器**（它自己不是 `Item`）。

```cpp
// sql/sql_lex.h —— Query_block 的表达式槽位（★ 5 个，互不相干）
  mem_root_deque<Item *> fields;      // SELECT 列表：每项一棵树
  Item *m_where_cond;                 // WHERE：一棵树
  SQL_I_List<ORDER> group_list{};     // GROUP BY：链表，每项一棵树
  Item *m_having_cond;                // HAVING：一棵树
  SQL_I_List<ORDER> order_list{};     // ORDER BY：链表，每项一棵树
```

⇒ 本例的完整形状（★ 注意容器层）：

```
Query_block（"select1"）        ← ★ 容器层，本身不是 Item
  │
  ├─ fields[0] ──────────►  Item_func_plus("+")          ← 树 1（结果列）
  │                          ├── Item_func_abs("abs")
  │                          │   └── Item_field(t.a)
  │                          └── Item_func_mul("*")
  │                              ├── Item_field(t.b)
  │                              └── Item_int(2)
  │
  ├─ m_where_cond ───────►  Item_func_gt(">")            ← 树 2（WHERE）
  │                          ├── Item_field(t.c)
  │                          └── Item_int(5)
  │
  ├─ group_list   ───────►  (空)
  ├─ m_having_cond ──────►  (空)
  └─ order_list   ───────►  (空)
```

★ 三点推论：

1. **遍历一个查询的所有表达式 = 遍历所有槽位**，不存在"从根往下走一遍"——这正是优化器要逐个槽位跑 `walk()` / `compile()` 的原因（见"树改写三件套"）。
2. **`ORDER BY` / `GROUP BY` 是 `SQL_I_List<ORDER>` 链表**：每个 `ORDER` 节点持有一个 `Item*`，所以这两处是"**列表 of 树**"而非单棵树。
3. **树与树之间只能借 `Query_block` / `Query_expression` 互访**——典型是子查询：`Item_subselect` 持有 `Query_expression*`，从而从"外层 WHERE 的树"跳进"内层查询的树"。源码注释（`sql/sql_lex.h` 顶部）画的就是这条跨树链接：

```
QB1: select1
  └─ m_where_cond (Item*)
       └─ Item_in_subselect
            └─ unit: Query_expression* → QE2: unit1.1    ← ★ 跨树链接
                 └─ first_query_block → QB2: select1.1.1
```

⇒ 所以"一个 SQL 有两条独立的 Item 树"是对的，但**图上必须画出 `Query_block` 这层容器**，否则读者会以为漏画了父节点。

★ 树之外还有一层隐含结构：**`Item_func` 用 `Item **args` + `uint arg_count` 持有子节点**——`args[0]`/`args[1]` 就是上图的左右分支。**后面所有阶段的遍历都是对这个数组做循环或递归**，`args[]` 的编码方式是理解 `Item` 的第一课（见 5.x 的 `Item_func_case`）。

### 阶段 ①：解析——PT 树 → Item 树

```
Bison 归约
  └─> PT_select_stmt（→ PT_query_expression → PT_query_specification）
        │  make_cmd() → contextualize() 递归
        ▼
  PT_query_specification::contextualize()
        ├─ 处理 FROM → PT_table_factor_table_ident::contextualize  → 建 Table_ref + TABLE
        └─ 处理 SELECT 列表 → itemize()（第二套递归）
              ├─ PT_select_item_list → 逐项
              └─ PTI_expr（占位符）→ PT_function → itemize()
                    └─ Instantiator<Item_func_plus>::instantiate()
                          ├─ 先递归拿到 args[0]（ABS 子树）、args[1]（MUL 子树）
                          └─ new Item_func_plus(POS(), left, right)
```

★ **造 Item 是"后序遍历"**——必须先把两个子 Item 造出来，才能把指针交给父节点的构造函数。所以 `+` 的构造函数执行时，`ABS(...)` 与 `t.b*2` **已经是完整的 Item 了**，只是还没 `fix_fields`。

★ 这里的 `POS()` 就是 1.1 节讲的 Bison 位置：从 `PT_function` 持有的 `@$` 一路传到 `Item(const POS &)`。

### 阶段 ②：prepare——★ 两轮方向**相反**的递归

这是整条链路最反直觉的地方：**`fix_fields` 自顶向下，`resolve_type` 自底向上**，二者交错在同一趟遍历里。

```cpp
// sql/item_func.cc
bool Item_func::fix_fields(THD *thd, Item **) {
  ...
  Condition_context CCT(thd->lex->current_query_block());
  used_tables_cache = get_initial_pseudo_tables();
  not_null_tables_cache = 0;
  ...
  if (arg_count) {
    for (arg = args, arg_end = args + arg_count; arg != arg_end; arg++) {
      if (fix_func_arg(thd, arg)) return true;      // ★ ① 逐个子节点递归（向下）
    }
  }
  if (resolve_type(thd) || thd->is_error())         // ★ ② 子节点都就绪后，才定自己的类型（向上）
    return true;
  fixed = true;                                     // ★ 最后才置 fixed
  return false;
}
```

递归的"推进器"是 `fix_func_arg`——**递归 + 检查列权限 + 把子节点属性累加到自己身上**：

```cpp
// sql/item_func.cc
bool Item_func::fix_func_arg(THD *thd, Item **arg) {
  if ((!(*arg)->fixed && (*arg)->fix_fields(thd, arg)))   // ★ 递归（子已 fixed 则跳过）
    return true;
  Item *item = *arg;
  if (allowed_arg_cols) {
    if (item->check_cols(allowed_arg_cols)) return true;  // 列访问权限
  } else {
    allowed_arg_cols = item->cols();                      // 从第一个参数取"允许的列集"
  }
  set_nullable(is_nullable() || item->is_nullable());      // ★ 可空性上浮
  used_tables_cache |= item->used_tables();                // ★ 依赖表集合上浮
  if (null_on_null) not_null_tables_cache |= item->not_null_tables();
  add_accum_properties(item);
  return false;
}
```

⇒ 走到叶子时触发列名解析：`Item_field::fix_fields()` → `find_field_in_tables()` → `find_field_in_table()` → **`Item_field::set_field(Field*)`**，把 `data_type` / `collation` / `unsigned_flag` / `max_length` 从 `Field` 一次性快照进 Item（详见本文 4.1）。**这一刻 `t.a` 才知道自己是 `INT`。**

**类型信息自底向上传播**：

```cpp
// sql/item_func.cc —— ABS(a) 的类型推导
bool Item_func_abs::resolve_type(THD *thd) {
  if (Item_func_num1::resolve_type(thd)) return true;
  unsigned_flag = args[0]->unsigned_flag;      // ★ 无符号性从参数继承
  return false;
}

// sql/item_func.cc —— num1 的降级规则
void Item_func_num1::set_numeric_type() {
  switch (hybrid_type = args[0]->result_type()) {   // ★ 问子节点：你是哪一类？
    case INT_RESULT:
      set_data_type(MYSQL_TYPE_LONGLONG);
      unsigned_flag = args[0]->unsigned_flag;
      break;
    case STRING_RESULT:
    case REAL_RESULT:
      set_data_type(MYSQL_TYPE_DOUBLE);
      hybrid_type = REAL_RESULT;                    // ★ 覆盖赋值：STRING → REAL
      max_length = float_length(decimals);
      break;
    case DECIMAL_RESULT:
      set_data_type(MYSQL_TYPE_NEWDECIMAL);
      unsigned_flag = args[0]->unsigned_flag;
      break;
    default:
      assert(0);
  }
}
```

```
Item_field(t.a)   → result_type = INT_RESULT        （已由 set_field 从 Field 拷来）

Item_func_abs     → resolve_type → set_numeric_type
                    看 args[0]（t.a）= INT → hybrid_type = INT_RESULT
                    结果类型 = MYSQL_TYPE_LONGLONG

Item_func_mul     → Item_func_numhybrid::resolve_type
                    看 args[0](t.b)=INT, args[1](2)=INT → hybrid_type = INT
                    ★ mul 还要做"溢出升级"判断（INT*INT 可能溢出 → 升级为 DECIMAL）

Item_func_plus    → 同理，看 abs(=INT)、mul(=INT) → hybrid_type = INT
```

★ **为什么两轮方向相反**：`fix_fields` 要自顶向下**把上下文推下去**（列权限、`Query_block` 等"环境"信息）；`resolve_type` 要自底向上**把类型信息收上来**。二者在同一个 `fix_fields` 函数里交错——每到一层，先递归子节点（向下），子节点返回后就地调 `resolve_type`（向上）。即**"下行建连、上行定型"的一次遍历**。

### 阶段 ③：执行——递归求值

```cpp
// sql/item_func.cc —— Item_func_numhybrid::val_int()：用 prepare 期定好的类型分派
longlong Item_func_numhybrid::val_int() {
  assert(fixed == 1);
  switch (hybrid_type) {
    case DECIMAL_RESULT: {
      my_decimal decimal_value, *val;
      if (!(val = decimal_op(&decimal_value))) return 0;   // null is set
      longlong result;
      my_decimal2int(E_DEC_FATAL_ERROR, val, unsigned_flag, &result);
      return result;
    }
    case INT_RESULT:
      return int_op();                    // ★ 本例走这条
    case REAL_RESULT: { /* real_op() 再转 longlong */ }
  }
}
```

```cpp
// sql/item_func.cc —— ★ 全链路里最复杂的一段：加法的溢出协商
longlong Item_func_plus::int_op() {
  longlong val0 = args[0]->val_int();              // ★ 递归进 ABS
  if (current_thd->is_error()) return error_int();
  longlong val1 = args[1]->val_int();              // ★ 递归进 MUL
  if (current_thd->is_error()) return error_int();
  longlong res = static_cast<unsigned long long>(val0) +
                 static_cast<unsigned long long>(val1);   // ★ 先按无符号环绕加，暂不判溢出
  bool res_unsigned = false;

  if ((null_value = args[0]->null_value || args[1]->null_value)) return 0;  // ★ 空值传染

  /*
    First check whether the result can be represented as a
    (bool unsigned_flag, longlong value) pair, then check if it is compatible
    with this Item's unsigned_flag by calling check_integer_overflow().
  */
  if (args[0]->unsigned_flag) {
    if (args[1]->unsigned_flag || val1 >= 0) {
      if (test_if_sum_overflows_ull((ulonglong)val0, (ulonglong)val1)) goto err;
      res_unsigned = true;
    } else {                                       /* val1 is negative */
      if ((ulonglong)val0 > (ulonglong)LLONG_MAX) res_unsigned = true;
    }
  } else {
    if (args[1]->unsigned_flag) {
      if (val0 >= 0) {
        if (test_if_sum_overflows_ull((ulonglong)val0, (ulonglong)val1)) goto err;
        res_unsigned = true;
      } else {
        if ((ulonglong)val1 > (ulonglong)LLONG_MAX) res_unsigned = true;
      }
    } else {
      if (val0 >= 0 && val1 >= 0)
        res_unsigned = true;
      ...
    }
  }
  ...
}
```

★ **为什么一个"加法"要写这么多**：操作数是 `(值, 有符号性)` 二元组——`1 + 1` 与"无符号 `1 + 1`"是两组语义。这段代码枚举 **4 种符号组合**，每种判断"结果能否用 `(longlong, res_unsigned)` 表示"。**两个操作数都不知道对方的符号性**（prepare 期只把 `unsigned_flag` 逐层上浮用于定型，求值时每个节点仍带自己的标志），只能运行时协商。

ABS 侧（一元，简单得多）：

```cpp
// sql/item_func.cc
longlong Item_func_abs::int_op() {
  longlong value = args[0]->val_int();
  if ((null_value = args[0]->null_value)) return 0;      // ★ 先取值、再判 null（顺序不能反）
  if (unsigned_flag) return value;
  /* -LLONG_MIN = LLONG_MAX + 1 => outside of signed longlong range */
  if (value == LLONG_MIN) return raise_integer_overflow();  // ★ 唯一"取绝对值反而溢出"的值
  return (value >= 0) ? value : -value;
}
```

**叶子：一路走到 `Field`**，完整递归时序（★ 深度优先、先左后右）：

```
plus.val_int
 ├─ abs.val_int ── int_op ── a.val_int ── Field_long::val_int  ⇒ 读 record  ⇒ 假设 = -3
 │                            abs 算得 3
 ├─ mul.val_int ── int_op ─┬─ b.val_int ── Field_long::val_int  ⇒ 读 record  ⇒ 假设 = 4
 │                         └─ Item_int(2).val_int()             ⇒ 常量 2
 │                          mul 算得 8
 └─ plus.int_op: 3 + 8 = 11（做溢出协商）
```

⇒ **纯函数、无缓存**——每个节点每次被求值都重走一遍。`ABS(a)` 在 `WHERE` 与 `SELECT` 里各写一次就会被算两次，这正是优化器要做公共子表达式消除（CSE）的原因；需要跨行缓存时用 `Item_cache_*`（见 2.6）。

### 三个"反直觉"总结

| # | 反直觉之处 | 原因 |
|---|---|---|
| 1 | **`fix_fields` 向下、`resolve_type` 向上**，却写在同一个函数里 | 前者推环境（列权限/上下文）、后者收类型，交错完成一趟遍历 |
| 2 | **求值期没有"编译"，每次 `val_*()` 都重新 `switch`** | `Item` 树是解释执行的；`hybrid_type` 只把"选算法"提前到 prepare 期 |
| 3 | ★ **`+` 的 `int_op()` 有 40 行溢出协商** | 操作数是 `(值, 符号性)` 二元组，只能在运行时协商 |

---

## 一、Item 类继承树

### 1.0 ★ 家族规模：575 个 Item 子类

先看这棵"树"有多大——实测 `sql/item*.h` 里 `class Item_*` 的真实定义（扣除 53 处前向声明与 2 个同名但非子类）：

| 文件 | 真实定义 | 范畴 |
|---|---:|---|
| `item_func.h`（129 KB） | **132** | 数学/数值/位/类型转换/UDF |
| `item.h`（257 KB） | **53** | 基类、常量、字段、ref、cache、param |
| `item_geofunc.h` | **89** | 地理 / GIS |
| `item_strfunc.h` | **87** | 字符串 / 加解密 |
| `item_timefunc.h` | **61** | 时间 |
| `item_cmpfunc.h`（104 KB） | **42** | 比较 / 条件 / IN / LIKE / BETWEEN / IF / CASE |
| `item_sum.h` | **42** | 聚合 + 窗口 |
| `item_json_func.h` | **35** | JSON |
| `item_inetfunc.h` | 10 | INET6 / IP |
| `item_subselect.h` | **6** | 子查询 |
| `item_regexp_func.h` | 6 | REGEXP |
| `item_gtid_func.h` / `item_pfs_func.h` | 4 / 4 | GTID / PFS |
| `item_xmlfunc.h` | 3 | XPath |
| `item_row.h` | 1 | 行表达式 |
| **合计** | **575** | |

★ **`Item_func` 子树独占 515 个（≈90%）**——`item_func.h 132 + strfunc 87 + geofunc 89 + timefunc 61 + cmpfunc 42 + sum 42 + json 35 + 其余`。也就是说 Item 家族的"体积"几乎全在**函数**上。

**不以 `Item_` 命名但属于该家族的成员**（未计入 575）：`Aggregator` / `Aggregator_distinct` / `Aggregator_simple`、`SubqueryWithResult`、`subselect_indexsubquery_engine`、`subselect_hash_sj_engine`。

> ⚠️ **8.0.39 已无 `Item_leaf`**（全库 0 匹配）——旧资料里的这个名字已不成立。

#### 分层继承树（实测）

```
Parse_tree_node
└── Item                                    item.h
    ├── Item_basic_constant  3652   ★ 常量体系
    │   ├── Item_num → Item_int → {Item_int_0, Item_temporal, Item_uint,
    │   │                          Item_int_with_ref → Item_temporal_with_ref}
    │   │            → Item_decimal, Item_float → Item_func_pi
    │   ├── Item_string → Item_static_string_func → Item_partition_func_safe_string
    │   │                                        → {Item_blob, Item_empty_string}
    │   ├── Item_null / Item_hex_string → Item_bin_string / Item_json
    │   └── Item_cache 6679  ★ → 9 个 cache 类（int/bit/real/decimal/str/row/datetime/json）
    ├── Item_ident          3930   ★ 标识符
    │   ├── Item_field 4192 → Item_asterisk / Item_default_value
    │   │                    / Item_insert_value / Item_trigger_field
    │   ├── Item_ref 5733 → Item_view_ref / Item_outer_ref / Item_ref_null_helper
    │   └── Item_ident_for_show
    ├── Item_result_field   5656   ★ 纯虚 resolve_type()
    │   ├── Item_func  item_func.h:102   ★★ 全家 90%
    │   │   ├── Item_real_func / Item_dec_func / Item_int_func
    │   │   ├── Item_func_numhybrid → num1 / num_op(+,−,×,÷,mod) / min_max / coalesce
    │   │   ├── Item_int_func → Item_bool_func  ★（item_cmpfunc.h:295）
    │   │   │     ├ Item_func_bool_const → true/false
    │   │   │     ├ Item_bool_func2 → Item_func_comparison → Item_eq_base
    │   │   │     │        → eq / equal / ge / gt / le / lt / ne
    │   │   │     ├ Item_func_not → Item_func_not_all → Item_func_nop_all
    │   │   │     ├ Item_cond → Item_cond_and / Item_cond_or
    │   │   │     ├ Item_equal（MULT_EQUAL_FUNC）
    │   │   │     ├ Item_func_in（200 行）/ Item_func_like
    │   │   │     └ Item_func_opt_neg → Item_func_between
    │   │   ├── Item_func_bit → or / and / xor
    │   │   ├── Item_udf_func → udf_float / int / decimal / str
    │   │   ├── Item_str_func（strfunc.h:76）→ sysconst / charset_conversion ...
    │   │   ├── Item_temporal_func（timefunc.h:471）→ date / datetime / time / hybrid
    │   │   ├── Item_geometry_func（geofunc.h:179）→ 89 个 GIS 类
    │   │   ├── Item_json_func（json_func.h:151）→ 35 个 JSON 类
    │   │   └── Item_sum  item_sum.h:399   ★ 42 个类
    │   │         ├── Item_sum_num → sum_int → count / sum → avg / variance → std
    │   │         ├── Item_sum_hybrid → min / max；  Item_sum_bit → or/and/xor
    │   │         ├── Item_sum_json → array / object；  Item_udf_sum
    │   │         ├── Item_func_group_concat 2100
    │   │         ├── 窗口：Item_non_framing_wf → row_number/rank/cume_dist/
    │   │         │        percent_rank/ntile/lead_lag/first_last_value/nth_value
    │   │         ├── Item_rollup_sum_switcher 2715 ★（ROLLUP 双实现）
    │   │         └── 伴生：Item_avg_field / Item_variance_field / Item_std_field
    │   └── Item_subselect  item_subselect.h:83   ★ 6 个 Item 类 + 3 个引擎类
    │         ├── Item_singlerow_subselect → Item_maxmin_subselect
    │         └── Item_exists_subselect → Item_in_subselect → Item_allany_subselect
    ├── Item_sp_variable 3684 → Item_splocal / Item_case_expr
    ├── Item_param 4602
    ├── Item_aggregate_type 7069 → Item_type_holder / Item_values_column
    ├── Item_name_const / Item_metadata_copy
    └── Item_row
```

#### `type()` 枚举：**30 个值**（8.0.39 实测，`item.h:918-949`）

```
INVALID_ITEM, FIELD_ITEM, FUNC_ITEM, SUM_FUNC_ITEM, STRING_ITEM, INT_ITEM,
REAL_ITEM, NULL_ITEM, VARBIN_ITEM, METADATA_COPY_ITEM, FIELD_AVG_ITEM,
DEFAULT_VALUE_ITEM, PROC_ITEM, COND_ITEM, REF_ITEM, FIELD_STD_ITEM,
FIELD_VARIANCE_ITEM, INSERT_VALUE_ITEM, SUBSELECT_ITEM, ROW_ITEM, CACHE_ITEM,
TYPE_HOLDER, PARAM_ITEM, TRIGGER_FIELD_ITEM, DECIMAL_ITEM, XPATH_NODESET,
XPATH_NODESET_CMP, VIEW_FIXER_ITEM, FIELD_BIT_ITEM, VALUES_COLUMN_ITEM
```

对照：`FIELD_ITEM`→`Item_field`、`SUM_FUNC_ITEM`→`Item_sum`、`STRING_ITEM`→`Item_string`、`INT_ITEM`→`Item_int`、`REAL_ITEM`→`Item_float`、`NULL_ITEM`→`Item_null`、`REF_ITEM`→`Item_ref`、`SUBSELECT_ITEM`→`Item_subselect`、`CACHE_ITEM`→`Item_cache`、`PARAM_ITEM`→`Item_param`、`TRIGGER_FIELD_ITEM`→`Item_trigger_field`、`DECIMAL_ITEM`→`Item_decimal`、`COND_ITEM`→`Item_cond`、`ROW_ITEM`→`Item_row`、`TYPE_HOLDER`→`Item_type_holder`、`DEFAULT_VALUE_ITEM`→`Item_default_value`、`INSERT_VALUE_ITEM`→`Item_insert_value`、`FIELD_AVG/STD/VARIANCE/BIT_ITEM`→对应的"聚合结果落临时表列后再读出来"的伴生类。

> （另：`Item_func::functype()` 枚举有 **122** 个值，`item_func.h:185-308`——那是"哪个函数"的标签，与 `type()` 的"哪类东西"是两个正交维度。）

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

#### ★ 继承 `Parse_tree_node` 带来的实际产物：`POS` 源码位置

这就是 `Item_func(POS(), ...)` 里那个 `POS` 的来历——**它是 Bison 的位置类型，不是普通的 id**：

```cpp
// sql/parse_location.h:57 —— ★ 注意：POS 就是 YYLTYPE
typedef YYLTYPE POS;

/** Helper class for the YYLTYPE */
struct Symbol_location {
  const char *start;  // token 起始
  const char *end;    // token 之后的第一个字节
  bool is_empty() const { return length() == 0; }
  size_t length() const { return static_cast<size_t>(end - start); }
};
/** Bison "location" class */
struct YYLTYPE {
  Symbol_location cpp;  // 预处理后缓冲中的位置
  Symbol_location raw;  // 原始缓冲中的位置（★ 两套并存：预处理会改写文本）
};
```

⇒ **为什么它是"parse tree node 的一种"会体现在构造函数签名上**：8.0 把"语法归约时直接造 Item"改成了三阶段，位置信息随之被一路带下来：

```
① Bison 归约            每个规则带 @$（YYLTYPE），bison 自动填 start/end
      │
② 造 PT_* 节点          PT_xxx 继承 Parse_tree_node，持有 POS
      │
③ contextualize/itemize  PTI_* 占位符 itemize() 时【把 POS 交给 Item 构造函数】
      │                   explicit Item(const POS &);         ← sql/item.h:1130
      ▼
④ Item 保留该位置        用于报错时精确定位
```

**它的唯一用途是精确报错**——基类提供了三个带位置的工具：

```cpp
void Parse_tree_node_tmpl<Context>::error(Context *pc, const POS &pos) const;
void Parse_tree_node_tmpl<Context>::error(Context *pc, const POS &pos, const char *msg) const;
void Parse_tree_node_tmpl<Context>::errorf(Context *pc, const POS &pos, const char *format, ...) const;
```

⇒ 所以 `SELECT 1 + 'abc'` 报 `ER_TRUNCATED_WRONG_VALUE` 时能指向那个具体表达式在原始 SQL 文本里的位置，而不是笼统说"SQL 有错"。★ **`cpp` / `raw` 两套位置并存**是因为预处理（PS 的 `?` 替换、存储过程展开）会改写文本，`raw` 保留用户真正写的那段。

（PT_* 节点的完整生命周期与 `itemize()` 契约见 [`05_contextualize.md`](05_contextualize.md) 第三章与第九章。）

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
virtual enum Type type() const = 0;                    // 30 个值的粗粒度类型标签（8.0.39 实测）
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

### 2.2 ★ 两层类型：`result_type()`（粗）vs `data_type()`（细）

> ★ **先澄清归属（极易混淆）**：**`data_type()` 是 `Item` 的方法，不是 `Field` 的**——实测 `sql/field.h` 里 `data_type` 与 `set_data_type_*` **全 0 命中**；它们在 `sql/item.h`（`data_type()` 在 1391，另有 **21 个** `set_data_type_*`）。
> `Field` 侧对应的是 **1.4 / 1.5 讲过的三套标签**：`type()`（逻辑）/ `real_type()`（物理）/ `result_type()`（`Item_result`）。本文以下讲的都是 **Item 侧**。

| | `result_type()` | **`data_type()`** |
|---|---|---|
| 类型 | `Item_result`（5 值：INT/REAL/STRING/DECIMAL/ROW） | `enum_field_types`（几十值：`MYSQL_TYPE_*`） |
| 作用 | 决定**调哪个 `val_xxx`**（求值分派） | 决定**精确 SQL 类型、精度、字符集**（元数据） |
| 存储 | 子类 override（或 `hybrid_type` 动态） | **`uint8 m_data_type`** 成员（8.0 为省空间压成 1 字节） |

```cpp
// [F1] sql/item.h
  /// Retrieve the derived data type of the Item.
  inline enum_field_types data_type() const {
    return static_cast<enum_field_types>(m_data_type);   // ★ uint8 → enum 的强转
  }
  /** Retrieve actual data type... Equal to data_type() for all items,
      except parameters. */
  virtual enum_field_types actual_data_type() const { return data_type(); }

  inline void set_data_type(enum_field_types data_type) {
    m_data_type = static_cast<uint8>(data_type);
  }
```

**★ `set_data_type_*()` 系列（21 个）是"打包设置器"**——每个都一次性设好 `data_type` + `collation` + `max_length`（+ `decimals`），避免调用方漏设某一项：

```cpp
// [F1] sql/item.h
  inline void set_data_type_null() {
    set_data_type(MYSQL_TYPE_NULL);
    collation.set(&my_charset_bin, DERIVATION_IGNORABLE);
    max_length = 0;
    set_nullable(true);
  }
  inline void set_data_type_bool() {
    set_data_type(MYSQL_TYPE_LONGLONG);      // ★ BOOL 在 8.0 就是 LONGLONG
    collation.set_numeric();
    max_length = 1;
  }
  inline void set_data_type_longlong() {
    set_data_type(MYSQL_TYPE_LONGLONG);
    collation.set_numeric();
    fix_char_length(21);                     // ★ 20 位数字 + 1 位符号
  }
  inline void set_data_type_decimal(uint8 precision, uint8 scale) {
    set_data_type(MYSQL_TYPE_NEWDECIMAL);
    collation.set_numeric();
    decimals = scale;
    fix_char_length(my_decimal_precision_to_length_no_truncation(
        precision, scale, unsigned_flag));   // ★ 由 precision/scale 反推显示宽度
  }
  inline void set_data_type_string(uint32 max_l) {
    max_length = max_l * collation.collation->mbmaxlen;   // ★ 字符数 → 字节数
    decimals = DECIMAL_NOT_SPECIFIED;
    if (max_length <= Field::MAX_VARCHAR_WIDTH)      set_data_type(MYSQL_TYPE_VARCHAR);
    else if (max_length <= Field::MAX_MEDIUM_BLOB_WIDTH) set_data_type(MYSQL_TYPE_MEDIUM_BLOB);
    ...
  }
```

⇒ 三点：① `set_data_type_string()` 会**按长度自动升级类型**（VARCHAR → MEDIUM_BLOB → BLOB），这正是"长字符串自动变 BLOB"的实现点；② `set_data_type_decimal()` 要求**先设好 `unsigned_flag`**（注释明说）；③ `collation.set_numeric()` 是数值类的标配（时态/数值列在 `Field` 侧被强制为 `my_charset_numeric`，这里是 Item 侧的对应做法）。

**★ `default_data_type()`：给"结果类型随参数变"的函数兜底**

```cpp
// [F1] sql/item.h（注释原文节选）
  /**
    Get the default data (output) type for the specific item.
    Important for some SQL functions that may deliver multiple result types...
    An example of such function is '+', which may return INT, DECIMAL, DOUBLE,
    depending on arguments.
    On the contrary, many other functions have a fixed output type, usually
    set with set_data_type_XXX(), which overrides the value of
    default_data_type(). For example, COS always returns DOUBLE,
  */
  virtual enum_field_types default_data_type() const {
    assert(data_type() == MYSQL_TYPE_INVALID);   // ★ 只在"还没定型"时才问它
    return MYSQL_TYPE_VARCHAR;
  }
```

⇒ 它服务于 **PS 参数定型**：`CAST(? AS DATE)` 或 `? + 5` 里的 `?` 无法从自身推出类型，只能问父节点"你默认输出什么类型"。固定输出类型的函数（`COS`→DOUBLE）用 `set_data_type_double()` 覆盖掉默认值。

**★ `MYSQL_TYPE_INVALID` 是"我还没有类型"的哨兵值**——`Item_param` 首次 prepare 时带着它就置 `fixed = true`（见 3.7.6），等父节点用 `propagate_type()` 反向注入（见 3.7.6 与 9.8.5）。

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

**浮点（REAL）路径的精度推导是另一套**，由 `fix_num_length_and_dec()` 负责（与上面 DECIMAL 的 `result_precision()` 对照看才完整）：

```cpp
void Item_func::fix_num_length_and_dec() {
  uint fl_length = 0;
  decimals = 0;
  for (uint i = 0; i < arg_count; i++) {
    decimals = max(decimals, args[i]->decimals);     // ① 小数位 = 各参数最大值
    fl_length = max(fl_length, args[i]->max_length); // ② 同时记下参数里最长的显示长度
  }
  max_length = float_length(decimals);               // ③ 由小数位反推浮点显示长度

  if (fl_length > max_length) {                      // ★ ④ 装不下 → 退化
    decimals = DECIMAL_NOT_SPECIFIED;
    max_length = float_length(DECIMAL_NOT_SPECIFIED);
  }
}
```

★ 第 ④ 行的**退化路径**是这个函数的重点：先按“各参数小数位的最大值”算出结果的小数位与显示长度，但如果**某个参数本身的长度就超过了这个长度**（`fl_length > max_length`），说明用这个小数位根本装不下——于是放弃精确推导，把 `decimals` 置为 `DECIMAL_NOT_SPECIFIED`（未指定），并按它重算 `max_length`。

⇒ 两条路径的对比值得记：

| | DECIMAL 路径 `result_precision()` | 浮点路径 `fix_num_length_and_dec()` |
|---|---|---|
| 小数位 | 各参数 max | 各参数 max（同） |
| 长度/精度 | `max(整数位) + 1 + decimals`（**+1 是进位保护位**） | `float_length(decimals)` |
| 放不下时 | 按 DECIMAL 规则截断/进位 | ★ 退化为 `DECIMAL_NOT_SPECIFIED` |

即：**DECIMAL 会为进位预留 1 位，浮点则不预留、装不下就直接放弃精度声明**。这也是为什么浮点运算结果的小数位有时显得“没有规律”——它可能是退化后的 `DECIMAL_NOT_SPECIFIED`。

（`Item_func_numhybrid::fix_num_length_and_dec()` 是**空实现**——numhybrid 的结果类型取决于运行期实际类型，在 prepare 期无从推导，故留空。）

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

### 2.7 ★ 求值递归的栈视角：深度上限与检查点

> 前面几节讲 `val_*` **协议**（怎么取值），本节补一个常被忽略的维度：**求值递归吃的是 C 栈**，它决定了表达式能嵌套多深。
> 栈机制的完整剖析（OS 层 `execve`/`mmap`/guard page/`mprotect`、`thread_stack` 变量、`check_stack_overrun` 实现、JSON 容量分析、424 层事故复盘）见 [`../../infra/threading/stack_depth.md`](../../infra/threading/stack_depth.md)——按"一份内容只在一处详写"，**本篇不重复**，只写 Item 这一层的选择。

**★ 数据在堆上，遍历在栈上**

| 维度 | 位置 | 成本 |
|---|---|---|
| **Item 对象** | `MEM_ROOT`（堆，placement new） | 424 个嵌套对象仅几十 KB——很便宜 |
| **`val_*` 递归** | **C 栈** | 424 层 × 数百字节 ≈ 200~250KB——**致命** |

`Item` 用 `operator new(size_t, MEM_ROOT *)` 分配在堆上，对象本身很便宜；但**递归求值它们**要逐层压栈——`val_json()` 调子节点的 `val_json()`，一层层直到叶子。所以"MySQL 用 MEM_ROOT 所以不吃栈"是错的：那是**数据**放在哪的问题，而**调用钻多深**是另一回事。

**★ 检查点分布（8.0.39 实测 `check_stack_overrun` 处数）**

| 文件 | 处数 | 含义 |
|---|---|---|
| `item_func.cc` / `item_cmpfunc.cc` | 3 / 3 | 部分通用函数、比较/逻辑函数有防护 |
| `item_subselect.cc` | 1 | 子查询 |
| `item_xmlfunc.cc`（XPath） | **2** | ★ 有防护 |
| **`item_json_func.cc`**（JSON 全部函数） | **0** | ★ **零防护 → 超限直接 SIGSEGV** |
| **`item.cc`**（`walk` / `copy` / `split_sum_func` 等基类递归） | **0** | ★ **零防护** |

**XML 防了、JSON 没防**——两者是同一类结构（对嵌套结构递归求值），差异更像是**演进中的遗漏**（XML 函数年代久远、补过栈检查；JSON 函数是 5.7 新代码，没继承这个教训），而非设计取舍。

**★ 与 contextualize 的对照（关键）**

另一条递归路径 `contextualize`（parse tree → Item 树）在**基类**里就有检查：

```cpp
// sql/parse_tree_node_base.h
virtual bool contextualize(Context *pc) {
    uchar dummy;
    if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)) return true;
```

因为每个 PT 节点的 contextualize 都经过基类，**这条递归全程有保护**。而执行期的 `val_*` 递归只在部分文件有检查点。

**后果**：同一条深嵌套 SQL，可能在 contextualize 阶段**未触发**阈值（每层消耗较小）而被放行，到了执行期却因每层栈帧更大而**冲过实际栈尾**——且这条路径没有检查点，于是 SIGSEGV。

> **诊断判据**（详见 [`stack_depth.md`](../../infra/threading/stack_depth.md)）：
> 报 `ER_STACK_OVERRUN_NEED_MORE`（"Thread stack overrun"）= 崩在**有检查点**路径，错误信息自带已用/总量诊断；
> **SIGSEGV**（无 MySQL 错误信息）= 崩在**无检查点**路径，如 JSON 求值递归，只能靠 core dump 定位。

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

**★ 补充 1：漏了"第 0 步"**——基类 `Item::fix_fields` 的**首行**是 `assert(is_contextualized())`（见 3.7.1）。它把 3.1 的 contextualize 与 3.2 的 fix_fields 硬绑定：任何 Item 必须先 `itemize()` 拿到 `Name_resolution_context`，才允许 `fix_fields`。⇒ **fix_fields 与 itemize 是严格两阶段，不能合并**。

**★ 补充 2：还漏了三件实质工作**：

- **栈深度守卫**：`if (check_stack_overrun(thd, STACK_MIN_SIZE * 2, buff)) return true;`——所有树节点覆写都有（深嵌套表达式递归有爆栈风险）。
- **`Item_sum` 独有的聚合层级/嵌套校验**：`init_sum_func_check()`（下降时）+ `check_sum_func()`（上升时）成对出现（见 3.7.3）。
- **PS 参数的类型反传 `propagate_type()`**：对 `?` 而言 `resolve_type` 常常"推迟"，真正定型靠父节点 fix 完子节点后**反向**调用（见 3.7.6）。

**★ 补充 3：`cleanup()` 不复位 `fixed`**——注释原文 *"Do not remove values allocated during preparation"*。PS 重执行靠 `if (!item->fixed)` 守卫**跳过** fix_fields（8.0 的 "resolve once"）。全代码库只有三处显式复位 `fixed`：`Item_field::reset_field()`（仅 prepared CREATE TABLE）、`Item_outer_ref` 构造、IN→EXISTS 二次变换前。

### 3.7 ★ `Item::fix_fields()` 基类与主要子类覆写

#### 3.7.1 基类实现只有 5 行

```cpp
// [F2] sql/item.cc
bool Item::fix_fields(THD *, Item **) {
  assert(is_contextualized());

  // We do not check fields which are fixed during construction
  assert(fixed == 0 || basic_const_item());
  fixed = true;
  return false;
}
```

★ **全文只有 5 行**——这是整个体系最反直觉处：

- **`assert(is_contextualized())`** —— 第 0 步前置条件。来自 `Parse_tree_node_tmpl::is_contextualized()`，**仅 DEBUG 生效**（`contextualized` 字段被 `#ifndef NDEBUG` 包裹）。
- **`fixed = true`** —— 基类**只负责"打戳"**；所有实质工作（名字解析、递归、定型、权限、折叠）都在子类。
- **永远返回 false** —— 基类不做任何可能失败的事。

**为什么能这么简单**：`Item` 子类树是"叶子 vs 树节点"二分——

| 类别 | 例子 | 是否调 `fix_fields` |
|---|---|---|
| **叶子**（值/类型构造期已知） | `Item_int` / `Item_string` / `Item_null` / `Item_cache` | **不调用**——构造函数里直接 `fixed = true` |
| **树节点** | `Item_func` / `Item_cond` / `Item_sum` / `Item_row` | 覆写，自己递归 |
| **需要名字解析的叶子** | `Item_field` / `Item_ref` / `Item_subselect` | 覆写（要查表/别名/下钻 unit） |

字面量把动机说得最直白（`Item_string::init()` 注释）：*"it is constant => can be used without fix_fields (and frequently used)"*。

#### 3.7.2 "标准骨架"（各子类覆写的通用后序形态）

```
0. assert(is_contextualized()) / assert(!fixed)
1. check_stack_overrun()                —— 防深递归爆栈
2. 建 Condition_context 等局部解析上下文
3. 递归：for each child: if(!child->fixed) child->fix_fields(&child)
   ★ 传的是子节点在父 args/list 中的【地址 Item **】
4. 元数据上浮：used_tables_cache |=、not_null_tables_cache、set_nullable()、add_accum_properties()
5. 语义校验：名字解析 / 聚合层级 / 嵌套聚合 / 列权限
6. resolve_type(thd)                     —— 定型（8.0 的 fix_length_and_dec）
7. 常量折叠 / 语义改写（可能整体替换 *ref）
8. fixed = true
```

★★ **第 3 步传 `Item **` 是关键设计**：`Item_func` 传 `args + i`，`Item_cond` 传 `li.ref()`，`Item_sum` 传 `args + i`。这样**子节点可以在 `fix_fields` 内部把自己替换掉**（`*ref = substitution`），父节点循环里用 `*arg` 重新取值即可感知——这是 IN→EXISTS 改写的基础设施。`Item_sum_hybrid::fix_fields` 里那条注释是最佳注脚：

```cpp
  // 'item' can be changed during fix_fields
  if ((!item->fixed && item->fix_fields(thd, args)) ||
      (item = args[0])->check_cols(1))     // ★ 必须从 args[0] 重新取回
    return true;
```

#### 3.7.3 ★ `Item_sum::fix_fields`：它自己几乎什么都不做

**8.0.39 里 `Item_sum::fix_fields` 只有 11 行，且完全不置 `fixed`**：

```cpp
// [F4] sql/item_sum.cc
bool Item_sum::fix_fields(THD *thd, Item **ref [[maybe_unused]]) {
  assert(fixed == 0);
  if (m_window != nullptr) {
    if (m_window_resolved) return false;
    if (Window::resolve_reference(thd, this, &m_window)) return true;
    m_window_resolved = true;
  }
  return false;
}
```

它只处理**窗口函数引用的延迟解析**（`OVER (w)` 的窗口名 → `Window*`），用 `m_window_resolved` 幂等保护。**真正的定型分散在两层**（源码注释明写了这个契约）：

```cpp
/**
  @note This function must be called for all set functions when expressions are
        resolved. It must be invoked in prefix order, ie at the descent of this
        traversal. @see corresponding Item_sum::check_sum_func(), which should
        be called on ascent.
*/
bool Item_sum::init_sum_func_check(THD *thd);   // 下降时
bool Item_sum::check_sum_func(THD *thd, Item **ref);  // 上升时
```

`check_sum_func()` 的核心决策（`nest_level` / `aggr_level` / 嵌套校验）：

```cpp
// [F4] —— 关键节选
  if (base_query_block->nest_level == max_aggr_level) {
    if ((allow_sum_func & nest_level_map) != 0)
      aggr_query_block = base_query_block;      // 在本层聚合
  } else if (max_aggr_level >= 0 || !(allow_sum_func & nest_level_map)) {
    for (Query_block *sl = base_query_block->outer_query_block(); ...)  // 找外层
      if (allow_sum_func & ((nesting_map)1 << sl->nest_level)) aggr_query_block = sl;
  } else { aggr_query_block = base_query_block; }   // 无列引用 → 最内层

  /* ① 内层聚合挂在更外层 → 非法 */
  if (aggr_query_block == nullptr || aggr_query_block->nest_level <= max_sum_func_level) {
    my_error(ER_INVALID_GROUP_FUNC_USE, MYF(0)); return true;
  }
  /* ② 禁止嵌套聚合：下钻参数，发现同层 Item_sum 即报错 */
  for (uint i = 0; i < arg_count; i++) {
    if (args[i]->has_aggregation() &&
        WalkItem(args[i], enum_walk::SUBQUERY_POSTFIX, [this](Item *subitem) {
          if (subitem->type() != Item::SUM_FUNC_ITEM) return false;
          Item_sum *si = down_cast<Item_sum *>(subitem);
          return si->aggr_query_block == this->aggr_query_block;   // ★ 同层即非法
        })) { my_error(ER_INVALID_GROUP_FUNC_USE, MYF(0)); return true; }
  }
```

★ **`max_aggr_level` 是被 `Item_field`/`Item_ref` 反向写回的**（`thd->lex->in_sum_func->max_aggr_level = max(..., nest_level)`）——这就是"聚合函数知道自己参数里最外层的列来自哪一层"的机制。

★ **`Aggregator::setup()` 不在 fix_fields 里**——它在**执行前**由 `setup_sum_funcs()` → `Item_sum::aggregator_setup()` 触发；选 SIMPLE/DISTINCT 更晚（在 `finalize_plan.cc` 按 `need_distinct` 决定）。⇒ 临时表参数不是 fix_fields 阶段建的。

具体子类（`Item_sum_num` = SUM/AVG 系）：

```cpp
// [F4]
bool Item_sum_num::fix_fields(THD *thd, Item **ref) {
  if (super::fix_fields(thd, ref)) return true;
  if (init_sum_func_check(thd)) return true;         // 下降
  Condition_context CCT(thd->lex->current_query_block());
  set_nullable(false);
  for (uint i = 0; i < arg_count; i++) {
    if ((!args[i]->fixed && args[i]->fix_fields(thd, args + i)) || args[i]->check_cols(1))
      return true;
    set_nullable(is_nullable() || args[i]->is_nullable());
  }
  null_value = true;                                  // ★ 必须在 resolve_type 前设
  if (resolve_type(thd)) return true;
  if (check_sum_func(thd, ref)) return true;          // 上升
  fixed = true;
  return false;
}
```

#### 3.7.4 ★ `Item_subselect::fix_fields`：整棵子树一次 fix 完

```cpp
// [F5] sql/item_subselect.cc
bool Item_subselect::fix_fields(THD *thd, Item **ref) {
  assert(!fixed);
  assert(indexsubquery_engine == nullptr);
  if (check_stack_overrun(thd, STACK_MIN_SIZE, (uchar *)&res)) return true;

  if (!(res = subquery->prepare(thd))) {        // ★① 递归进入整个子查询的 prepare
    changed = true;
    accumulate_properties();                    // ★② 归约成 INNER_TABLE_BIT / OUTER_REF_TABLE_BIT
    if (substitution) {                         // ★③ IN→EXISTS 等改写：整根替换后重新 fix
      (*ref) = substitution;
      if (item_name.is_set()) substitution->item_name = item_name;
      substitution = nullptr;
      thd->where = "checking transformed subquery";
      if (!(*ref)->fixed) ret = (*ref)->fix_fields(thd, ref);   // ★ 对新 Item 再 fix 一次
      return ret;
    }
    if (unit_cols() > max_columns) { my_error(ER_OPERAND_COLUMNS, MYF(0), 1); return true; }
    if (resolve_type(thd)) goto err;
  } else goto err;

  if ((uncacheable = unit->uncacheable)) {
    if (uncacheable & UNCACHEABLE_RAND) { m_subquery_used_tables |= RAND_TABLE_BIT; ... }
  }
  assert(!has_wf());      // ★④ 窗口函数必须在子查询自己层聚合
  fixed = true;
err:
  thd->where = save_where;
  return res;
}
```

★ 四点：

1. **`subquery->prepare(thd)`** 递归进入整个子查询的 prepare，触发内部所有 Item 的 fix_fields —— 子查询是"整棵子树一次性 fix 完"的黑盒。
2. **`accumulate_properties()`** 把子查询内部的表依赖归约成两个位：`INNER_TABLE_BIT`（引用自己的表）与 `OUTER_REF_TABLE_BIT`（引用外层 = 相关）。
3. **`substitution` 分支**是"fix_fields 期间树被重写"的最典型场景：`*ref = substitution` 换掉父节点指针，再对新 Item 调 `fix_fields`。
4. **相关子查询的 `mark_as_dependent()` 不在本函数**——它由子查询内部的 `Item_field::fix_outer_field()` / `Item_ref::fix_fields` 逐列触发；本函数只**收集结果**。

#### 3.7.5 ★ `Item_ref::fix_fields`：二级指针与"借"来的 fixed

```cpp
// [F1] sql/item.h
 protected:
  Item **m_ref_item{nullptr};      // ★ 二级指针
```

★ **为什么是二级指针**：`Item_ref` 引用的通常是 **SELECT list 数组的某个槽位**（`base_ref_items[i]`）。优化期这个槽位会被整体替换（如 `split_sum_func`、IN→EXISTS），`Item_ref` 必须**自动看到新值**，所以存"槽位地址"而非"值"。

**"借" fixed 的两种途径**：

```cpp
// [F2] —— 构造期就"借"
  if (m_ref_item != nullptr && ref_item() != nullptr) {
    ref_item()->increment_ref_count();
    if (ref_item()->fixed) set_properties();   // ★ 被引用者已 fixed → 复制属性并使自己 fixed
  }
```

```cpp
// [F2] —— fix_fields 尾部（★ 关键）
  link_referenced_item();
  assert(ref_item()->fixed);        // ★★ Item_ref 的全部哲学

  /* 拒绝对聚合的非法引用：① 只允许在 HAVING；② 不能在同层 GROUP BY */
  if ((ref_item()->has_aggregation() &&
       !thd->lex->current_query_block()->having_fix_field) ||
      walk(&Item::has_aggregate_ref_in_group_by, enum_walk::SUBQUERY_POSTFIX, nullptr)) {
    my_error(ER_ILLEGAL_REFERENCE, MYF(0), full_name(), "reference to group function");
    return true;
  }
  set_properties();
  if (ref_item()->check_cols(1)) return true;
  return false;
}

void Item_ref::set_properties() {
  set_data_type(ref_item()->data_type());
  max_length = ref_item()->max_length;
  set_nullable(ref_item()->is_nullable());
  decimals = ref_item()->decimals;
  collation.set(ref_item()->collation);
  set_accum_properties(ref_item());
  unsigned_flag = ref_item()->unsigned_flag;
  fixed = true;                     // ★ 属性镜像完成后才置位
}
```

⇒ **`Item_ref` 自己不做定型，只负责找到 `m_ref_item`，然后断言对方已 fixed，把属性整份复制过来**。运行时所有 `val_*()` 都是**转发**到 `ref_item()->val_*()`，所以**只复制元数据、不复制值**。

三个变体：

| 变体 | 特点 |
|---|---|
| `Item_view_ref` | 不覆写 `fix_fields`；只在构造时修正 `m_orig_*` 与 `cached_table`，并处理外连接可空性 |
| `Item_outer_ref` | ★ **属性借用、fixed 不借用**：构造里 `set_properties(); fixed = false;` |
| `Item_aggregate_ref` | 文件内局部类；存在的唯一理由是 **print 时不打成 ref**；`update_aggr_refs()` 演示二级指针威力——改写时只改 `m_ref_item`，不需重新 fix |

#### 3.7.6 ★ `Item_param`：不在 fix_fields 定型，靠 `propagate_type`

```cpp
// [F2] sql/item.cc
bool Item_param::fix_fields(THD *, Item **) {
  assert(!fixed);
  if (param_state() == NO_VALUE) {
    // PS 首次 prepare，还没有值
    assert(data_type() == MYSQL_TYPE_INVALID);
    collation.set(default_charset());
    fixed = true;                   // ★ 带着 MYSQL_TYPE_INVALID 就置 fixed！
    return false;
  }
  if (param_state() == NULL_VALUE) { fixed = true; return false; }
  switch (data_type_actual()) { ... }   // 有实际值 → 按实际值定型
  fixed = true;
  return false;
}
```

★ **这就是 8.0 "PS 参数不定型"的入口**：`NO_VALUE` 时 `data_type()` 保持 `MYSQL_TYPE_INVALID` 就置 `fixed = true`；上层 `resolve_type()` 见到 `MYSQL_TYPE_INVALID` 就**推迟**，等父节点用 `propagate_type` **反向注入**（如 `Item_cond::fix_fields` 里对每个参数做 `item->propagate_type(thd, MYSQL_TYPE_LONGLONG)`）。

`propagate_type` 取"最大可能尺寸"策略（因为 PS 每次执行值可能不同）：字符串一律 `65535 / mbmaxlen`，DECIMAL 一律 `DECIMAL_MAX_PRECISION/DECIMAL_MAX_SCALE`。基类包装里有短路条件：

```cpp
// [F1]
    if (data_type() != MYSQL_TYPE_INVALID && !(pin && type() == PARAM_ITEM))
      return false;      // ★ 已有类型就不覆盖，除非显式 pin
```

#### 3.7.7 `Item_sp_variable` / `Item_splocal`：断言对方已 fixed

```cpp
// [F2] sql/item.cc
bool Item_sp_variable::fix_fields(THD *, Item **) {
  Item *it = this_item();          // 返回 sp_rcontext 里的真实 Item
  assert(it->fixed);               // ★
  max_length = it->max_length;
  decimals = it->decimals;
  unsigned_flag = it->unsigned_flag;
  collation.set(it->collation);
  set_data_type(it->data_type());
  fixed = true;
  return false;
}
```

★ 语义：SP 变量的**类型声明时已知**（构造函数里就 `set_data_type(sp_var_type)`），fix_fields 只是把"声明类型"与"运行期实际值 item"对齐。

#### 3.7.8 `Item_cond::fix_fields`：多出四件事

```cpp
// [F6] sql/item_cmpfunc.cc（结构节选）
  /* ① AND-OR 扁平化：AND(F1, AND(F2,F3)) → AND(F1,F2,F3) */
  while ((item = li++)) {
    Item_cond *cond;
    while (item->type() == Item::COND_ITEM && (cond = down_cast<Item_cond *>(item)) &&
           cond->functype() == func_type && !cond->list.is_empty()) {
      li.replace(cond->list);  cond->list.clear();  item = *li.ref();
    }
    if (ignore_unknown()) item->apply_is_true();
    if ((!item->fixed && item->fix_fields(thd, li.ref())) || (item = *li.ref())->check_cols(1))
      return true;

    /* ② 常量折叠：cond AND FALSE → ALWAYS FALSE；cond OR TRUE → ALWAYS TRUE */
    if (ref != nullptr && select->first_execution && item->const_item() &&
        !item->walk(&Item::is_non_const_over_literals, enum_walk::POSTFIX, nullptr) &&
        !thd->lex->is_view_context_analysis() && ignore_unknown() &&
        !select->has_ft_funcs() && can_remove_cond) {
      if (remove_const_conds(thd, item, &new_item)) return true;
      ...
    }
    /* ③ 类型反传：AND/OR 的操作数必须是布尔 */
    if (item->propagate_type(thd, MYSQL_TYPE_LONGLONG)) return true;

    used_tables_cache |= item->used_tables();
    /* ④ not_null_tables_cache 的 AND/OR 语义差异 */
    if (func_type == COND_AND_FUNC && ignore_unknown())
      not_null_tables_cache |= item->not_null_tables();   // AND：任一项非空
    else
      not_null_tables_cache &= item->not_null_tables();   // OR：必须全部非空
  }
  if (resolve_type(thd)) return true;
  fixed = true;
```

★ 折叠触发条件很严：`ref != nullptr`（允许改写）、`first_execution`（PS 只在首次）、`const_item()`、不含 `?`/SP 变量/用户变量、非视图分析、无全文函数。★ 折叠求值期间**临时压入 ignore/strict 错误处理器**（常量表达式溢出不应终止 prepare）。

#### 3.7.9 速查表

| 类 | 覆写 `fix_fields` | 比基类多什么 | `fixed` 置位处 |
|---|---|---|---|
| `Item`（基类） | — | 只打戳 + 两条 assert | 函数内 |
| `Item_func` | ✅ | 栈守卫、逐参 `fix_func_arg`、`resolve_type` | 尾 |
| `Item_cond` | ✅ | +扁平化、折叠、`propagate_type`、AND/OR 的 not_null 差异 | 尾 |
| `Item_field` | ✅ | 名字解析、列权限、`set_field` | 权限检查后 |
| `Item_ref` | ✅ | 解析 `m_ref_item`、聚合引用合法性、`set_properties()` | `set_properties()` 内 |
| `Item_sum` | ✅（极简） | 仅窗口引用解析 | **不置位** |
| `Item_sum_num/_hybrid` | ✅ | `init_sum_func_check` → fix 参数 → `resolve_type` → `check_sum_func` | 尾 |
| `Item_subselect` | ✅ | `unit->prepare()`、`accumulate_properties`、`substitution` 重 fix | 尾 |
| `Item_sp_variable` | ✅ | 从运行期 item 抄属性（`assert(it->fixed)`） | 尾 |
| `Item_param` | ✅ | 按 `param_state()` 三分支（定型靠 `propagate_type`） | 尾（可能带 `INVALID`） |
| `Item_cache` / 字面量 | ❌ | — | **构造函数内** |

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
- 三个伪表位让"非常量"不被误判为常量（见 4.3）

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

### 3.8 ★ 树改写三件套与"改写后如何重新定型"

理论基础章（92 行）已讲 `walk`/`transform`/`compile` 的存在，本节讲**改写之后 `fixed` 怎么办**——这是 3.7 的配套。

#### 3.8.1 三者的分工（源码注释原文要点）

```cpp
// [F8] sql/sql_const.h
enum class enum_walk {
  PREFIX = 0x01, POSTFIX = 0x02, SUBQUERY = 0x04,
  SUBQUERY_PREFIX = 0x05, SUBQUERY_POSTFIX = 0x06
};
using Item_processor = bool (Item::*)(unsigned char *);
```

| 方法 | 阶段 | 语义 | 可回滚？ |
|---|---|---|---|
| `transform()` | **prepare 期** | "all transformations are **permanent**; they are not rolled back" | ❌ 永久 |
| `compile()` | **优化期** | "supposed to be used during the **optimization** stage… recorded using `THD::change_item_tree()` so that they can be **rolled back after execution**" | ✅ 可回滚 |
| `walk()` | 任意 | 只读遍历（PREFIX/POSTFIX，可选下钻 SUBQUERY）；processor 返回 **true 表示中止** |

```cpp
// [F1] —— compile 带 analyzer：自顶向下分析 + 自底向上改写
  virtual Item *compile(Item_analyzer analyzer, uchar **arg_p,
                        Item_transformer transformer, uchar *arg_t) {
    if ((this->*analyzer)(arg_p)) return ((this->*transformer)(arg_t));
    return this;
  }
```

现代写法用 lambda 包装（避免手写成员函数指针）：`WalkItem(item, walk, functor)` / `TransformItem(item, transformer)` / `CompileItem(item, analyzer, transformer)`（[F1] 模板）。

`Item_ref` 对 `transform`/`compile` 的覆写都遵循同一原则——**先改被引用者，若被替换则丢弃自身**：

```cpp
// [F2]
Item *Item_ref::transform(Item_transformer transformer, uchar *arg) {
  assert(ref_item() != nullptr);
  Item *new_item = ref_item()->transform(transformer, arg);
  if (new_item == nullptr) return nullptr;
  /* If the object is transformed into a new object, discard the Item_ref
     object and return the new object as result. */
  if (new_item != ref_item()) return new_item;      // ★ 被替换 → 丢弃自身
  return (this->*transformer)(arg);
}
```

#### 3.8.2 ★ 为什么用 `fix_after_pullout()` 而不是重新 `fix_fields()`

`fix_fields()` 是**重量级**：重做名字解析（`find_field_in_tables`）、重查列权限（`get_column_grant`）、重跑 `resolve_type`、重做折叠与语义改写。

而 semijoin / derived 合并只是**把表从子 query block 上提到父 query block**：

- 表名/列名/别名**都没变** → 不需重新解析
- 权限**已查过** → 不需再查
- 类型**没变** → 不需 `resolve_type`
- **唯一变了的是"层级"**：`nest_level`、`depended_from`、以及由层级折算的 `used_tables_cache`（含 `OUTER_REF_TABLE_BIT`）

⇒ 所以引入 `fix_after_pullout(parent, removed)`——**只重算"依赖层级"那部分元数据**。入口（`[F7]`）：

```cpp
void Query_block::fix_after_pullout(Query_block *parent_query_block,
                                    Query_block *removed_query_block) {
  if (where_cond())  where_cond()->fix_after_pullout(parent_query_block, removed_query_block);
  for (Table_ref *tr : m_table_nest) fix_tables_after_pullout(...);   // join 条件
  if (having_cond()) having_cond()->fix_after_pullout(...);
  for (Item *item : visible_fields()) item->fix_after_pullout(...);
  for (ORDER *order = order_list.first; order; order = order->next)
    (*order->item)->fix_after_pullout(...);
  for (ORDER *group = group_list.first; group; group = group->next)
    (*group->item)->fix_after_pullout(...);
}
```

叶子层的实质动作在 `Item_ident::fix_after_pullout`——**修 `depended_from` 与 `context`**：

```cpp
// [F2]
void Item_ident::fix_after_pullout(Query_block *parent_query_block,
                                   Query_block *removed_query_block) {
  /* 执行期临时创建的 Item_field（如条件下推 clone 的）没有解析上下文，已被执行过 */
  if (context == nullptr) { assert(type() == FIELD_ITEM); return; }

  assert(context->query_block != removed_query_block);

  if (context->query_block == parent_query_block) {
    if (parent_query_block == depended_from) {
      depended_from = nullptr;                            // ★ 不再是相关引用
      context = &parent_query_block->context;              // ★ 解析上下文上移
    }
  } else { /* 定义域在被移除块的内层 → 可能需要更新依赖 */ }
```

★ 各子类的 `fix_after_pullout` 与 `fix_fields` **形态同构**（下钻子节点 → OR 累积），但**不重新 prepare、不重新 resolve_type、不动 `fixed`**。三个要点：

- `Item_func::fix_after_pullout` **第一个提前返回是 `if (const_item()) return;`**——常量项的表依赖是"历史决定"的，不能重算，否则会把已判定为 const 的项重新变成非 const。
- `Item_sum::fix_after_pullout` 有硬约束 `assert(aggr_query_block != removed_query_block)`——被上提的 query block 不可能是聚合上下文。
- `Item_cache::fix_after_pullout` 不能走通用路径，必须显式转发给 `example` 并重算 `used_table_map`。

#### 3.8.3 什么时候才真的需要重新 `fix_fields()`

当变换**新建了 Item 树**（不是搬表）时。三处实证：

```cpp
// [F5] —— IN→EXISTS 生成 substitution 后，对新 Item 重跑
      (*ref) = substitution;
      if (!(*ref)->fixed) ret = (*ref)->fix_fields(thd, ref);

// [F7] —— 视图的 WHERE 与 ON 合并成新 Item_cond_and 后重跑
      Item_cond_and *new_cond = down_cast<Item_cond_and *>(and_conds(i1, i2));
      assert(!new_cond->fixed);                  // ★ 总是新 Item
      Item *cond_after_fix = new_cond;
      if (new_cond->fix_fields(thd, &cond_after_fix)) return true;

// [F7] —— IN→EXISTS 二次变换前【显式复位 fixed】
    subq_item->changed = false;
    subq_item->fixed = false;                    // ★ 允许对同一对象重跑 fix_fields
```

★ `subq_item->fixed = false;` 是全代码库极少数显式复位 `fixed` 的地方之一——因为要对**同一个** `Item_subselect` 重跑完整 fix_fields（重新走 `select_transformer`）。

#### 3.8.4 `cleanup()` 与 PS 重执行

```cpp
// [F1]
  /**
    Called for every Item after use (preparation and execution).
    ...
    Do not remove values allocated during preparation, destructor handles this.
  */
  virtual void cleanup() { marker = MARKER_NONE; }
```

★ **`cleanup()` 不复位 `fixed`**（注释："Do not remove values allocated during preparation"）。walk 侧的处理器反而**以 `fixed` 为门控**：

```cpp
  /// cleanup() item if it is resolved ('fixed').
  bool cleanup_processor(uchar *) { if (fixed) cleanup(); return false; }
```

⇒ `fixed` 是 cleanup 的**前提**而非**产物**。PS 重执行不重新 fix_fields，靠 `if (!item->fixed)` 守卫跳过（8.0 的 "resolve once"）：

```cpp
// [F7]
    if ((!m_where_cond->fixed && m_where_cond->fix_fields(thd, &m_where_cond)) ||
        m_where_cond->check_cols(1)) return true;
```

### 3.9 ★ `fixed` 标志的完整语义

#### 3.9.1 定义与三条不变式

```cpp
// [F1]
 public:
  bool fixed;  ///< True if item has been resolved
```

一句话：**"这个 Item 已经可以被求值了"**。置位后下列成员全部有效且此后只读：`data_type()`、`result_type()`、`max_length`、`decimals`、`unsigned_flag`、`collation`、`is_nullable()`、`used_tables_cache`、`not_null_tables_cache`、`m_accum_properties`。

1. **单调性**：除 3.8.3 的三个显式特例外，一旦为 true 不再回退。
2. **子节点先于父节点**：任何覆写都保证 `fixed = true` 写在所有子节点 fix_fields 之后。
3. **`Item_ref` 的 fixed 依赖被引用者**：`assert(ref_item()->fixed)`（`Item_ref::fix_fields` 尾）、`assert(it->fixed)`（`Item_sp_variable::fix_fields`）。

#### 3.9.2 `assert(fixed == 0 || basic_const_item())` 到底在说什么

- `fixed == 0` —— 正常：第一次被 fix。
- `|| basic_const_item()` —— 允许"**构造期就已置 `fixed = true` 的基本常量**"再次路过 fix_fields（如 `Item_int`/`Item_string`/`Item_null`/`Item_cache`）。`fix_func_arg` 里已有 `if (!(*arg)->fixed && ...)` 短路，但**没有这个守卫的路径**（直接调 `Item::fix_fields`）靠这个 `||` 不至于误报。

```cpp
// [F1] —— basic_const_item 的契约
  /**
    Returns true if this is a simple constant item like an integer, not
    a constant expression. ... It is assumed that val_xxx() does not modify
    the item's state for such items. It is also assumed that val_str() can be
    called with nullptr as argument as val_str() will return an internally
    cached const string.
  */
  virtual bool basic_const_item() const { return false; }
```

★ **`Item_ref` 显式把它关掉**（`bool basic_const_item() const override { return false; }`）——即使引用的是 `Item_int`，因为 `val_xxx()` 要穿透二级指针，不满足"不改状态 / `val_str(nullptr)` 可用"的契约。

**与 `const_item()` 的区别**：`const_item()` 是**语义上**的常量（整棵子树不依赖表，`used_tables() == 0`），用于常量折叠；`basic_const_item()` 是**结构上**的字面量，用于"能否在解析期直接取值"。

#### 3.9.3 谁在断言 `fixed`

`assert(fixed == 1)` / `assert(fixed)` 在代码库中**约 500 处**，集中在所有 `val_*()` / `get_date()` / `get_time()` / `send()` / `make_field()` 入口——即"**求值契约：只有 fixed 过的 Item 才能取值**"。分布：`item_func.cc` ~100、`item.cc` ~89、`item_cmpfunc.cc` ~54、`item_strfunc.cc` ~49、`item_timefunc.cc` ~46、`item_sum.cc` ~40、`item_geofunc.cc` ~39、`item_json_func.cc` ~33。

另有两处**反向**断言（要求**未** fixed），都在 fix_fields 入口：`Item_ref::fix_fields` 的 `assert(!fixed)`、`Item_subselect::fix_fields` 的 `assert(!fixed)`。

#### 3.9.4 绕过 `fixed` 的两个合法后门

| 后门 | 用途 |
|---|---|
| `quick_fix_field()`（`inline void quick_fix_field() { fixed = true; }`） | 优化期新建的内部 Item（已知 Field，不需解析）——注释："should be used in case where we are sure that we do not need complete fix_fields() procedure" |
| 构造期 `fixed = true` | 字面量（`Item_int`/`Item_string`/`Item_null`）与 `Item_cache`（3.7.9）——它们"is constant => can be used without fix_fields (and frequently used)" |

## 四、Item 与 Field

> ★ **完整实现剖析已独立成篇：[`../field.md`](../field.md) 第五章「与 Item / TABLE / handler / DD 的接缝」**——`Item_field::set_field()` 的属性快照、`Item::save_in_field_inner()` 的两级分派链、`Item_field` 列到列 memcpy 快路径、`create_tmp_field()` 三种造列路径，全在那篇。
>
> 本章只从 **Item 侧**讲清：为什么 `Item_field` 是唯一桥接点、"模板方法"体现在哪。**4.3 / 4.4（伪表位、`Item_ref` 家族）是 Item 独有内容，保留在本章。**

### 4.1 `Item_field`：Item 世界与 Field 世界的唯一桥接点

`Item_field` 是 Item 体系里**唯一**直接持有 `Field*` 的表达式节点。绑定动作就是 `set_field()`——它一次性把 Item 需要的全部类型信息从 Field 拷出来（**属性单向快照**）：

```cpp
void Item_field::set_field(Field *field_par) {
  field = result_field = field_par;      // ★ 非临时表场景二者是同一指针
  set_nullable(field->is_nullable() || field->is_tmp_nullable() ||
               field->table->is_nullable());
  collation.set(field_par->charset(), field_par->derivation(), field_par->repertoire());
  set_data_type(field_par->type());      // ★ data_type 从 Field 拷过来
  decimals = field->decimals();
  unsigned_flag = field_par->is_flag_set(UNSIGNED_FLAG);
  max_length = char_to_byte_length_safe(field_par->char_length(),
                                        collation.collation->mbmaxlen);
  fixed = true;                          // ★★ set_field 直接置 fixed（不必再走 fix_fields）
}
```

之后 `result_type()` 与 `val_xxx()` 都是**纯转发**给 Field：

```cpp
  Item_result result_type() const override { return field->result_type(); }
  longlong val_int() {
    assert(fixed == 1);
    if ((null_value = field->is_null())) return 0;   // ★ 先判空写 null_value，再取
    return field->val_int();
  }
```

```
SQL 文本 "t.a"
  → parser:    Item_field(table="t", field="a")   ← 只有名字，field == nullptr
  → fix_fields: find_field_in_tables() → set_field(Field *f)
                  data_type / collation / decimals / max_length  ← 从 Field 快照
                  fixed = true
  → 执行期:    val_int() / val_str() → field->val_xxx()（Field 从 record buffer 读）
```

★ `result_field` 与 `field` 的分离发生在"物化进临时表"时（`create_tmp_field()` 里调 `set_result_field()`），详见 [`../field.md`](../field.md) 5.1.6。

### 4.2 ★ `save_in_field()`：Item → Field 的写方向（模板方法）

4.1 是**读**方向（`val_*()` 转发给 `Field::val_*`）。写方向就是 `save_in_field()`——`fill_record()` 里那句 `value->save_in_field(rfield, false)` 就是它（见 [`10_dml.md`](10_dml.md) 2.1.1）。

> - **输入**：一个 `Item`（值）+ 一个 `Field`（目标列）
> - **输出**：值被写进该列在 `record[0]` 里的偏移；返回 `type_conversion_status`
> - **它自己不解析、不编码**——只做三件公共的事（装错误处理器、转调虚函数、兜底错误检查），编码交给下层的 `Field::store()`

★ **这是模板方法模式的教科书例子**：基类的 `Item::save_in_field()` **不是虚函数**——公共逻辑固定在此，只把"真正怎么存"下放到虚函数 `save_in_field_inner()`（`sql/item.h` 的注释明说"子类应覆写 `save_in_field_inner` 而不是 `save_in_field`"）。⇒ 别停在壳上，要往下挖到 `save_in_field_inner()`。

⇒ **两级分派链与完整代码见 [`../field.md`](../field.md) 5.1.4**：第一级看 `Item::result_type()`，第二级在 STRING 分支里再看目标 `Field::type()`（因为字符串是万能中间表示）；**落到 Field 的只有 4 类原语**——`store(const char*,len,cs)` / `store(double)` / `store(longlong,bool)` / `store_decimal()`，外加特化的 `store_json()` / `store_time()`。

★ 另有一条热路径旁路：`Item_field` 源和目标都是 Field 时**完全绕开分派链**，直接按 `pack_length()` 做列到列 `memcpy`（`field_conv_with_cache()`，带"上次目标 Field"缓存）——见 [`../field.md`](../field.md) 5.1.5。

### 4.3 `used_tables()` 与三个伪表位

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

### 4.4 `Item_ref` 家族：二级指针

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
| **`Item_func_case`** | ★ 最复杂的标量函数子类：一个 `args[]` 扁平编码两种语法（simple / searched），**两轮类型合并**，运行时用 `cmp_items[5]` + `value_added_map` 缓存比较器。详见下节 |

### `Item_func_case`：★ 多分支类型合并、PS 参数定形与比较器缓存

`CASE WHEN` 是 Item 体系里最有料的子类——它的复杂度**不在求值**（求值只是"选分支 + 转发"），而在**类型合并 / 参数定形 / 比较器复用**三处。

#### ① `args[]` 的扁平编码：奇偶位语义不同

```cpp
// sql/item_cmpfunc.h
class Item_func_case final : public Item_func {
  enum Item_result cached_result_type, left_result_type;  // 结果类型 / CASE 表达式的类型
  String tmp_value;
  uint ncases;                    // ★ 不是 WHEN 的个数，是 WHEN 数 × 2
  Item_result cmp_type;
  cmp_item *cmp_items[5];         // ★ 5 种 Item_result 各备一个比较器
  int first_expr_num, else_expr_num;   // ★ -1 表示"没有"（CASE 表达式 / ELSE 都可缺省）
```

★ **第一个复杂度来源**：所有子节点塞在一个扁平数组里，**奇数位和偶数位语义不同**，靠两个下标定位可选部分：

```
CASE x WHEN 1 THEN 'a' WHEN 2 THEN 'b' ELSE 'c' END

args[0]=1      (WHEN_1)     args[1]='a'   (THEN_1)
args[2]=2      (WHEN_2)     args[3]='b'   (THEN_2)
args[4]='c'    (ELSE)       ← else_expr_num = 4
args[5]=x      (CASE 表达式) ← first_expr_num = 5
                              ncases = 4  (= 2 × WHEN 数)

⇒ 遍历 THEN 要 for (arg = args + 1; arg < args + arg_count; arg += 2)   ★ 步长 2
⇒ first_expr_num == -1  ⇒ searched CASE（CASE WHEN cond THEN ...）
   first_expr_num != -1 ⇒ simple   CASE（CASE x WHEN val THEN ...）
```

⇒ **同一个类靠 `first_expr_num` 是否为 -1 承担两种语法**，所以后面所有代码都有两条分支。

#### ② `resolve_type()`：PS 参数定形的"临时换数组"技巧

`CASE x WHEN ? THEN ? END` 里两个 `?` 要定形，但**它们所属的类型组不同**（前者跟 CASE 表达式/其它 WHEN 一组，后者跟其它 THEN/ELSE 一组）。代码用了一个很妙的手段：

```cpp
// sql/item_cmpfunc.cc
bool Item_func_case::resolve_type(THD *thd) {
  Item **agg = (Item **)thd->mem_root->Alloc(sizeof(Item *) * (ncases + 1));
  if (agg == nullptr) return true;
  /*
    Choose types for dynamic parameters.
    1) CASE value WHEN [compare_value] THEN result ... [ELSE result] END
       If ? is in value/WHEN then infer from other WHENs/value. If ? is in
       THEN/ELSE then infer from other THENs/ELSE. If can't infer, use VARCHAR
       for value/WHEN, but determine type from outer context for THEN/ELSE.
  */
  // value/WHEN 这一组
  uint nagg;
  for (nagg = 0; nagg < ncases / 2; nagg++) agg[nagg] = args[nagg * 2];
  if (first_expr_num != -1) agg[nagg++] = args[first_expr_num];
  std::swap(args, agg);              // ★★ 临时把 args 换成"子集"
  std::swap(arg_count, nagg);
  if (param_type_uses_non_param(thd)) return true;   // 通用工具：用非参数项给 ? 定形
  std::swap(args, agg);              // ★★ 换回来
  std::swap(arg_count, nagg);

  /* 若所有 THEN/ELSE 都还无类型（全是 ?），本次定不了，交给外层 propagate_type */
  bool all_types_invalid = true;
  for (uint i = 0; i < ncases / 2; i++)
    if (args[i * 2 + 1]->data_type() != MYSQL_TYPE_INVALID) all_types_invalid = false;
  if (else_expr_num != -1 && args[else_expr_num]->data_type() != MYSQL_TYPE_INVALID)
    all_types_invalid = false;
  if (all_types_invalid) return false;      // ★ 提前返回，不硬凑类型

  // THEN/ELSE 这一组（同样的 swap 套路）
  for (nagg = 0; nagg < ncases / 2; nagg++) agg[nagg] = args[nagg * 2 + 1];
  if (else_expr_num != -1) agg[nagg++] = args[else_expr_num];
  std::swap(args, agg); std::swap(arg_count, nagg);
  if (param_type_uses_non_param(thd)) return true;
  std::swap(args, agg); std::swap(arg_count, nagg);

  return resolve_type_inner(thd);
}
```

★★ **`std::swap(args, agg)` 这一手是精髓**：`param_type_uses_non_param()` 是**通用工具**，只认 `args` 与 `arg_count` 两个成员。CASE 不改造它、也不加参数，而是**临时把自己的数组换成"筛选后的子集"**，调完再换回来——用"数据结构的临时篡改"复用逻辑。代价是源码 `@todo` 的吐槽：两处各分配一次 `agg`，有一次是冗余的。

#### ③ `resolve_type_inner()`：★ 两轮类型合并

```cpp
// sql/item_cmpfunc.cc
bool Item_func_case::resolve_type_inner(THD *thd) {
  Item **agg = (Item **)thd->mem_root->Alloc(sizeof(Item *) * (ncases + 1));
  if (agg == nullptr) return true;

  // ① 可空性：没有 ELSE 就一定可空；否则取所有 THEN/ELSE 的或
  bool nullable = else_expr_num == -1 || args[else_expr_num]->is_nullable();
  for (Item **arg = args + 1; arg < args + arg_count; arg += 2)   // ★ 步长 2，只走 THEN
    nullable |= (*arg)->is_nullable();
  set_nullable(nullable);

  // ② 合并所有 THEN / ELSE 的类型与 collation
  uint nagg;
  for (nagg = 0; nagg < ncases / 2; nagg++) agg[nagg] = args[nagg * 2 + 1];
  if (else_expr_num != -1) agg[nagg++] = args[else_expr_num];

  aggregate_type(make_array(agg, nagg));          // ★★ 类型合并（找能容纳所有分支的最小上界）
  assert(data_type() != MYSQL_TYPE_INVALID);

  cached_result_type = Field::result_merge_type(data_type());
  if (cached_result_type == STRING_RESULT) {
    /* Note: String result type is the same for CASE and COALESCE. */
    if (aggregate_string_properties(func_name(), agg, nagg)) return true;  // collation 合并
    /*
      Copy all THEN and ELSE items back to args[] array.
      Some of the items might have been changed to Item_func_conv_charset.
    */
    for (nagg = 0; nagg < ncases / 2; nagg++)
      change_item_tree_if_needed(&args[nagg * 2 + 1], agg[nagg]);   // ★★ 写回！
    if (else_expr_num != -1)
      change_item_tree_if_needed(&args[else_expr_num], agg[nagg++]);
  } else {
    aggregate_num_type(cached_result_type, agg, nagg);
  }

  // ③ 再合并「CASE 表达式 + 所有 WHEN」的比较类型（仅 simple CASE）
  if (first_expr_num != -1) {
    agg[0] = args[first_expr_num];
    left_result_type = agg[0]->result_type();
    for (nagg = 0; nagg < ncases / 2; nagg++) agg[nagg + 1] = args[nagg * 2];
    nagg++;
    uint found_types = collect_cmp_types(agg, nagg);
    ...
  }
```

★ 三个要点：

1. **合并是"找能容纳所有分支的最小上界"**——`THEN 1` / `THEN 2.5` / `THEN 'x'` 会合并成能容纳三者的类型；`assert(data_type() != MYSQL_TYPE_INVALID)` 保证合并一定出结果。
2. ★★ **`change_item_tree_if_needed()` 写回 `args[]`**——合并字符串时若各分支字符集不同，`aggregate_string_properties()` 会把某些分支**就地改造成 `Item_func_conv_charset`**（显式转换节点）。树被改了必须写回，否则运行时仍按旧节点求值 ⇒ **这是"prepare 期改写 Item 树"的典型例子**（呼应 3.8 节的树改写三件套）。
3. **两轮合并互不干扰**：THEN/ELSE 决定"结果是什么类型"，CASE/WHEN 决定"用什么类型做比较"。所以 `CASE x WHEN 'a' THEN 1 END`（x 是 int）要做**字符串比较**但返回 **int**。

#### ④ `find_item()`：★ 运行时的分支选择与"比较器缓存"

```cpp
// sql/item_cmpfunc.cc
Item *Item_func_case::find_item(String *) {
  uint value_added_map = 0;

  if (first_expr_num == -1) {
    // ── searched CASE：逐个求 WHEN 的真值 ──
    for (uint i = 0; i < ncases; i += 2) {
      if (args[i]->val_bool()) return args[i + 1];
      continue;
    }
  } else {
    /* Compare every WHEN argument with it and return the first match */
    for (uint i = 0; i < ncases; i += 2) {
      if (args[i]->real_item()->type() == NULL_ITEM) continue;   // WHEN NULL 永不匹配
      cmp_type = item_cmp_type(left_result_type, args[i]->result_type());  // ★ 逐 WHEN 算比较类型
      assert(cmp_type != ROW_RESULT);
      assert(cmp_items[(uint)cmp_type]);

      if (!(value_added_map & (1U << (uint)cmp_type))) {         // ★★ 该类型的比较器还没装左值
        cmp_items[(uint)cmp_type]->store_value(args[first_expr_num]);  // 装入 CASE 表达式的值
        if (current_thd->is_error()) return nullptr;
        if ((null_value = args[first_expr_num]->null_value))     // ★ NULL 不等于任何值
          return else_expr_num != -1 ? args[else_expr_num] : nullptr;
        value_added_map |= 1U << (uint)cmp_type;                 // ★ 置位，下次不再重复装
      }
      if (cmp_items[(uint)cmp_type]->cmp(args[i]) == false) return args[i + 1];  // 相等 → 返回 THEN
    }
  }
  // 全部未命中，返回 ELSE
  return else_expr_num != -1 ? args[else_expr_num] : nullptr;
}
```

★★ **`value_added_map` 位图是这段代码最精巧的地方**：

因为**每个 WHEN 的类型可能不同**（`CASE x WHEN 1 THEN 'a' WHEN 'b' THEN 'c'`），比较要按"两边协商出的 `cmp_type`"进行。MySQL 为 5 种 `Item_result` 各预置一个多态比较器 `cmp_items[5]`，但**左值（CASE 表达式）只需往每个比较器装一次**——`value_added_map` 记录"哪些类型的比较器已装好左值"，避免重复装入。

⇒ **"缓存 + 位图去重"**：扫 100 万行时 `store_value()` 最多被调用 5 次（每种类型一次），而不是 500 万次。

★ 另一处语义细节：CASE 表达式为 NULL 时直接走 ELSE——SQL 语义下 `NULL = NULL` 不为真。

#### ⑤ `val_*()`：七个出口全是"选择 + 转发"

```cpp
// sql/item_cmpfunc.cc
longlong Item_func_case::val_int() {
  assert(fixed == 1);
  StringBuffer<MAX_FIELD_WIDTH> dummy_str(default_charset());   // ★ 栈上的"哑"缓冲
  Item *item = find_item(&dummy_str);                           // ① 选分支
  if (item != nullptr) {
    const longlong res = item->val_int();                       // ② 转发给选中分支
    null_value = item->null_value;
    return res;
  }
  if (current_thd->is_error()) return error_int();
  null_value = true;                                            // ★ 无 ELSE 且未命中 → NULL
  return 0;
}
```

`val_real` / `val_str` / `val_decimal` / `val_json` / `get_date` / `get_time` **全是同一套式**——**CASE 自己不做任何计算，只负责"选分支 + 转发"**。

★ `dummy_str` 是个细节：`find_item` 在字符串比较时可能需要临时空间（转字符串、做 collation），但 `val_int` 路径并不需要字符串结果，所以传一个栈上的一次性缓冲。

#### ⑥ 复杂度对比：为什么 CASE 比 ABS 有料得多

| | `Item_func_abs` | **`Item_func_case`** |
|---|---|---|
| 子节点组织 | `args[0]`（1 个） | ★ `args[]` 扁平编码，奇偶位语义不同 + 两个可选下标 |
| 承担的语法 | 1 种 | ★ **2 种**（simple / searched，靠 `first_expr_num == -1` 切换） |
| `resolve_type` | `set_numeric_type()` 问 `args[0]` 一个类型 | ★ **两轮合并**（THEN/ELSE 一组、CASE/WHEN 一组）+ PS 参数 swap 定形 |
| prepare 期改写树 | 无 | ★★ `change_item_tree_if_needed()` 插入 `Item_func_conv_charset` |
| 求值 | `int_op()` 直接算 | ★ 先 `find_item()` 选分支再转发；★ `cmp_items[5]` + `value_added_map` 缓存比较器 |
| `val_*` 出口 | 基类按 `hybrid_type` 派 3 个 op | ★ **自己实现 7 个**（int/real/str/decimal/json/date/time） |

**一句话**：`ABS` 是"**我是整数，算法我给**"；`CASE` 是"**我不算，我只负责选——但'选'这件事涉及类型合并、字符集改写与比较器复用**"。

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

### 6.5 `Sumfunctype`：25 个值

`Item_sum` 自己有一个与 `Item_func::functype()`（122 值）**正交**的类型标签。注意 `Item_sum` 继承 `Item_func`，但**整个体系没有任何类 override `functype()`** ⇒ `SUM(x)` 的 `functype()` 是 `UNKNOWN_FUNC`，真正标识它的是 `sum_func()`。

| 值 | 类 | SQL |
|---|---|---|
| `COUNT_FUNC` / `COUNT_DISTINCT_FUNC` | **同一个** `Item_sum_count` | `COUNT(x)` / `COUNT(DISTINCT x)` |
| `SUM_FUNC` / `SUM_DISTINCT_FUNC` | `Item_sum_sum` | `SUM` |
| `AVG_FUNC` / `AVG_DISTINCT_FUNC` | `Item_sum_avg`（继承 `Item_sum_sum`） | `AVG` |
| `MIN_FUNC` / `MAX_FUNC` | `Item_sum_min` / `Item_sum_max`（共继承 `Item_sum_hybrid`） | `MIN` / `MAX` |
| `STD_FUNC` / `VARIANCE_FUNC` | `Item_sum_std` / `Item_sum_variance` | `STD`/`STDDEV`/`VAR_POP`/`VAR_SAMP` |
| `SUM_BIT_FUNC` | `Item_sum_bit` → or/and/xor | `BIT_OR`/`BIT_AND`/`BIT_XOR` |
| `UDF_SUM_FUNC` | `Item_udf_sum` → 4 个结果类型子类 | 聚合 UDF |
| `GROUP_CONCAT_FUNC` | `Item_func_group_concat` | `GROUP_CONCAT` |
| `JSON_AGG_FUNC` | `Item_sum_json` → array/object | `JSON_ARRAYAGG`/`JSON_OBJECTAGG` |
| 9 个窗口值 | `ROW_NUMBER_FUNC`…`NTH_VALUE_FUNC` | 窗口函数 |
| **`ROLLUP_SUM_SWITCHER_FUNC`** | `Item_rollup_sum_switcher` | **无直接语法**（见 6.7） |
| `GEOMETRY_AGGREGATE_FUNC` | `Item_sum_collect` | `ST_Collect` |

★ **DISTINCT 不是独立类**——靠 `with_distinct` 标志在 `sum_func()` 里二选一。而 `set_distinct()` 有个副作用：

```cpp
  void set_distinct(bool distinct) {
    with_distinct = distinct;
    allow_group_via_temp_table = !with_distinct;   // ★ 写了 DISTINCT 就禁用临时表增量更新
  }
```

### 6.6 各聚合的实现要点

**① `COUNT(*)` vs `COUNT(expr)` 的全部差异**——`COUNT(*)` 走 `PTI_count_sym`，它把自己伪装成 **`COUNT(0)`**：

```cpp
bool PTI_count_sym::itemize(Parse_context *pc, Item **res) {
  args[0] = new (pc->mem_root) Item_int(int32{0}, 1);
```

参数是一个**常量、非 NULL** 的 `Item_int(0, 1)` ⇒ `add()` 里的 `aggr->arg_is_null(false)` **永远 false** ⇒ 每行都计数。而 `COUNT(expr)` 的 NULL 行被短路掉。**没有第二条代码路径。**

`COUNT` 还有两个特性：`set_nullable(false)`（**永不为 NULL**，与 SUM/AVG 空集返回 NULL 对照）、`make_const()`（优化器可提前折叠）。

**② ★ `SUM(int_col)` 走 DECIMAL，不是 double**（常见误解）：

```cpp
  switch (args[0]->numeric_context_result_type()) {
    case REAL_RESULT:  set_data_type_double(); ...
    case INT_RESULT:
    case DECIMAL_RESULT: {
      // SUM result cannot be longer than length(arg) + length(MAX_ROWS)
      int precision = args[0]->decimal_precision() + DECIMAL_LONGLONG_DIGITS;
      set_data_type_decimal(precision, args[0]->decimals);
```

即 `hybrid_type` **只可能是 REAL 或 DECIMAL**。这就是"SUM 整数列不丢精度，但超大 SUM 报 DECIMAL 溢出而非静默回绕"的根因。decimal 分支用**双缓冲乒乓** `dec_buffs[2]` / `curr_dec_buff ^= 1`。

**③ AVG** 直接继承 `Item_sum_sum`（复用 `sum`），只多一个 `m_count`；`add()` = 父类 add + 计数。精度由 `div_precincrement`（默认 4）控制。

**④ ★ VAR/STD 的递推算法**——源码里 40 行注释是全文精华。核心是"朴素公式会**灾难性抵消**（catastrophic cancellation）"，所以用递推：

```
  Algorithm I（分组聚合，精确）:
  M_1 = x_1,  M_k = M_{k-1} + (x_k - M_{k-1}) / k
  S_1 = 0,    S_k = S_{k-1} + (x_k - M_{k-1}) × (x_k - M_k)
  variance = S_n / (n-1)

  Algorithm II（窗口移动帧，可增量）:
  K=0; n=0; ex=0; ex2=0
  add_sample(x):    n++; ex += x-K; ex2 += (x-K)²
  variance():       return (ex2 - (ex*ex)/n) / (n-1)
```

Algorithm II 由 `windowing_use_high_precision = false` 开启（**默认不开**），源码明确警告 *"Results could differ between optimized and non-optimized code path"*，且做了"负方差修正"：`if (variance < 0.0) return 0.0;`

样本 vs 总体 = 一个 `sample` 字段：`null_value = (count <= sample)`——总体方差 1 行也有值（0），样本方差 1 行是 NULL。

**⑤ MIN/MAX** 比较器**来自 `Arg_comparator`**（`setup_hybrid()` 里 `cmp->set_cmp_func(this, &arg_cache, &value, false)`），用 `m_is_min` 决定"更好"的方向。它是**唯一能在窗口里做"零比较"优化**的聚合（有 frame 时近似 FIRST/LAST_VALUE）。

**⑥ `GROUP_CONCAT`** 有几个反直觉点：
- 自建临时表，但标了 `HA_EXTRA_NO_ROWS` + `table->no_rows = true` ⇒ **这张表从不真正写行**，只是"行格式说明书 + `record[0]` 缓冲区"
- **ORDER BY / DISTINCT 时拼接是"惰性"的**：`add()` 只入树，`val_str()` 才遍历树真正拼串（`tree_walk` / `unique_filter->walk`）
- 截断按字节但**用 `well_formed_len` 修正到字符边界**
- `reset_field()` / `update_field()` 是 `assert(0)`，且构造时 `allow_group_via_temp_table = false`
- **明确不支持窗口函数**（`unsupported_as_wf()`）

### 6.7 ★★ ROLLUP 的双实现（此前完全没提）

**为什么需要"切换器"**：ROLLUP 要同时产出"本层小计"和"总计"，而它们**共享同一份 SQL 表达式、同一个 SELECT 槽位**。一个 `Item_sum` 只有一份累加状态，无法"一半行重置、一半行继续"⇒ 只能**复制 N+1 份**。

```cpp
/**
  A wrapper Item that contains a number of aggregate items, one for each level
  of rollup ... When aggregating, every aggregator is either reset or updated
  as per the correct level, and when returning a value, the correct child item
  corresponding to the current rollup level is queried.
 */
```

创建点（`create_rollup_switcher()`）造出 **N+1 个副本**（原始 + `send_group_parts` 个 `copy_or_same()` + `make_unique()`）。**索引约定**：`args[0]` = 原始 = **总计层**，索引越大越明细。

★ **同一行被"加两遍"的实现**是 `reset_and_add_for_rollup()`：

```cpp
  for (int i = 0; i < m_num_levels; ++i) {
    if (i >= last_unchanged_group_item_idx) {
      if (child(i)->reset_and_add()) return true;      // 变更层及以上：重置后重加（新组开始）
    } else {
      if (child(i)->aggregator_add()) return true;     // 更高层：继续累加
    }
  }
```

而组内后续行是 `aggregator_add_all()`——**所有副本同时 add**。⇒ **是 N+1 个独立状态，不是"一份状态两种读法"**。

★ **`real_sum_func()` 的完整机制**（三个 hook 构成"透明包装 + 可识别"）：

| hook | switcher 返回 | 用途 |
|---|---|---|
| `sum_func()` | `master()->sum_func()`（如 `SUM_FUNC`） | **逻辑身份**：按"它是哪个聚合"分派的代码照常工作 |
| `real_sum_func()` | `ROLLUP_SUM_SWITCHER_FUNC` | **物理身份**：遍历 Item 树时识别"这是包装器"并 **stop_at**（避免把副本当独立聚合再处理一遍） |
| `is_rollup_sum_wrapper()` / `unwrap_sum()` | `true` / `master()` | 需要拿"真身"的地方 |

**分组列**由另一个类 `Item_rollup_group_item` 处理（`m_min_rollup_level` = "第几层开始变 NULL"）。★ 它有个反直觉细节——为不被折叠，`used_tables()` 会人为加 `RAND_TABLE_BIT`。

**一次 `SetRollupLevel()` 同时切换两件事**：

```cpp
void AggregateIterator::SetRollupLevel(int level) {
  if (m_rollup && m_current_rollup_position != level) {
    m_current_rollup_position = level;
    for (Item_rollup_group_item *item : m_join->rollup_group_items)
      item->set_current_rollup_level(level);          // ① 哪些分组列变 NULL
    for (Item_rollup_sum_switcher *item : m_join->rollup_sums)
      item->set_current_rollup_level(level);          // ② 聚合从哪个副本取值
  }
}
```

⇒ 两者用**同一个 level 编号**对齐。

### 6.8 临时表契约与伴生类

**开关是 `allow_group_via_temp_table`**（默认 true）。★ 注意 **`reset_field`/`update_field` 压根不在 `Aggregator` 接口里**——它们属于"另一条正交路径"（增量更新 temp 表列），与 Aggregator 无关：

> *"Note that update_field/reset_field are not in that class, because they're simply not called when GROUP BY/DISTINCT can be handled with help of index on grouped fields."*

关闭者：`set_distinct(true)`、`Aggregator_distinct::setup()`、`Item_udf_sum`、`Item_func_group_concat`。关闭的后果是**整张临时表失去 group key**（`group = nullptr`）。

★ **伴生类为什么必须存在**——因为临时表列里存的**不是最终值，而是内部状态**：

| 聚合 | temp 表列内容 | 伴生类 |
|---|---|---|
| COUNT / SUM / MIN / MAX | **最终值** | 无（基类返回普通 `Item_field`） |
| **AVG** | `[sum][count]` 打包 | `Item_avg_field`（**除法在读取时做**） |
| **VAR** | `[m][s][count]` 打包 | `Item_variance_field` |
| **STD** | 同上 | `Item_std_field`（再 `sqrt`） |
| **BIT_XXX** | 位串 + 末尾标志字节 | `Item_sum_bit_field` |

只有这 4 个 override 了 `result_item()`。它们**不是 `Item_sum`**（无 `add()`/`clear()`），注释反复强调 *"fix_fields() never calls for this Item"*，且 `func_name()` 全是 `assert(0); return "avg_field";` 形态（不该出现在 SQL 文本里）。

## 七、★ Arg_comparator：比较的类型推导与函数选择

> 所有 `=` / `>` / `IN` / `BETWEEN` / `LIKE` 的**"用什么类型比、调哪个函数比"**都集中在这一处。不理解它就解释不了"为什么 `'1' = 1` 走整数比较而 `'a' = 1` 走浮点"、也解释不了"为什么隐式转换会吃掉索引"。

### 7.1 为什么需要它

MySQL 的类型规则远比标准 SQL 宽松——标准不允许 `date = 1`，MySQL 允许并**内部转换**。注释原文（`set_cmp_func` 上方）：

> *"Comparison function are expected to operate on arguments having the same data types. Since MySQL has very loosened up rules, it accepts all kind of arguments which the standard SQL does not allow, and then it converts the arguments internally to ones usable in the comparison."*

`Arg_comparator` 就是把"类型推导 + 转换 + 选比较函数"这三件事封装起来的地方：**一次 `set_cmp_func()` 把决策做完，之后每次 `compare()` 只是一次函数指针调用**（`inline int compare() { return (this->*func)(); }`）。

### 7.2 类成员全景（`item_cmpfunc.h:140-293`，154 行）

```cpp
class Arg_comparator {
  Item **left{nullptr};            // ★ 二级指针——与 PTI 的 Item **res 同构：可就地替换
  Item **right{nullptr};
  arg_cmp_func func{nullptr};      // 选中的比较函数（成员函数指针）
  Item_result_field *owner{nullptr};
  Arg_comparator *comparators{nullptr};  // 仅用于 compare_row()：递归的子比较器
  uint16 comparator_count{0};
  double precision{0.0};
  Item *left_cache{nullptr};       // 常量侧转换结果的缓存
  Item *right_cache{nullptr};
  bool set_null{true};             // 有 NULL 参数时是否置 owner->null_value

  bool try_year_cmp_func(Item_result type);
  static bool get_date_from_const(...);
  Json_scalar_holder *json_scalar{nullptr};  // 仅 compare_json() 用

 public:
  DTCollation cmp_collation;
  String value1, value2;           // 供 owner 使用的字符串缓冲

  longlong (*get_value_a_func)(THD *, Item ***item_arg, Item **cache_arg,
                               const Item *warn_item, bool *is_null){nullptr};
  longlong (*get_value_b_func)(...){nullptr};
  Item_result m_compare_type{INVALID_RESULT};   // 实际用来比较的类型
};
```

几个要点：

- ★ **`left`/`right` 是 `Item **`**——因为 `set_cmp_func` 期间可能**就地替换**操作数（如常量转成 `Item_cache_datetime`，见 7.4）
- **`func` 是成员函数指针**，最终由 `comparator_matrix[type]` 选出
- **`get_value_a/b_func`** 是"自定义取值器"——只在**日期/时间上下文**下设置，把值取成 packed 格式的 `longlong`。`use_custom_value_extractors()` 可查询是否启用
- **`comparators` + `comparator_count`** 使行比较 `(a,b) = (c,d)` **递归**成 n 个子比较器

### 7.3 ★ 类型推导：`item_cmp_type()`（`item.cc`）

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
  return REAL_RESULT;
}
```

四条规则：**相同→原样；有 ROW→ROW；INT/DECIMAL 组合→DECIMAL；其余→REAL**。

⇒ 这解释了两个经典现象：`int_col = 1.5` 走 **REAL**（第三、四条不命中 → REAL），而 `int_col = dec_col` 走 **DECIMAL**（第三条）。**一旦走了 REAL，索引就废了**（列被转成浮点，无法用索引定位）。

### 7.4 ★ `set_cmp_func()` 的七步决策（`item_cmpfunc.cc:1132-1250`）

`item_cmp_type()` 只是"默认推导"，`set_cmp_func` 会**按七步覆盖它**：

**① JSON 优先短路**（最先判断，最高优先级）

```cpp
  if (type != ROW_RESULT && (((*left)->result_type() == STRING_RESULT &&
                              (*left)->data_type() == MYSQL_TYPE_JSON) ||
                             ((*right)->result_type() == STRING_RESULT &&
                              (*right)->data_type() == MYSQL_TYPE_JSON))) {
    // Use the JSON comparator if at least one of the arguments is JSON.
    func = &Arg_comparator::compare_json;
    m_compare_type = STRING_RESULT;
    (*left)->mark_json_as_scalar();
    (*right)->mark_json_as_scalar();
    return false;      // ★ 直接返回，不再走后面的逻辑
  }
```

**② 日期短路** `can_compare_as_dates()`——★ 这是最常见的"覆盖默认推导"

```cpp
bool Arg_comparator::can_compare_as_dates(const Item *left, const Item *right) {
  if (left->type() == Item::ROW_ITEM || right->type() == Item::ROW_ITEM)
    return false;
  if (left->is_temporal_with_date() &&
      (right->result_type() == STRING_RESULT || right->is_temporal_with_date()))
    return true;
  else
    return left->result_type() == STRING_RESULT && right->is_temporal_with_date();
}
```

即：**一侧是 date/datetime/timestamp，另一侧是日期型或字符串型** ⇒ 走 `compare_datetime`。此时若字符串侧是**常量**，会把它转成 DATETIME 并**缓存**：

```cpp
    if (!(*left)->is_temporal_with_date()) {
      if (!get_date_from_const(*right, *left, &numeric_datetime) &&
          numeric_datetime != static_cast<ulonglong>(MYSQL_TIMESTAMP_ERROR)) {
        auto *cache = new Item_cache_datetime(MYSQL_TYPE_DATETIME);
        cache->store_value((*left), numeric_datetime);
        // Mark the cache as non-const to prevent re-caching.
        cache->set_used_tables(1);
        left_cache = cache;
        left = &left_cache;      // ★ 就地替换操作数
      }
    }
    ...
    func = &Arg_comparator::compare_datetime;
    get_value_a_func = &get_datetime_value;
    get_value_b_func = &get_datetime_value;
    cmp_collation.set(&my_charset_numeric);
    set_cmp_context_for_datetime();
```

⚠️ **`cache->set_used_tables(1)` 是"防再缓存"技巧**——把缓存标成非常量，避免它被再次包装。

⇒ ★ **这解释了为什么 `WHERE date_col = '2024-01-01'` 能走索引**：常量字符串在 `set_cmp_func` 期就转成 DATETIME 并缓存，比较时是"日期列 vs 日期常量"，列侧无需转换。

**③ TIME vs TIME**

```cpp
  } else if ((type == STRING_RESULT || type == REAL_RESULT) &&
             (*left)->data_type() == MYSQL_TYPE_TIME &&
             (*right)->data_type() == MYSQL_TYPE_TIME) {
    /* Compare TIME values as integers. */
    func = &Arg_comparator::compare_datetime;
    get_value_a_func = &get_time_value;
    get_value_b_func = &get_time_value;
```

**④ 字符串 vs 字符串：字符集聚合**

```cpp
  } else if (type == STRING_RESULT && (*left)->result_type() == STRING_RESULT &&
             (*right)->result_type() == STRING_RESULT) {
    DTCollation coll;
    coll.set((*left)->collation, (*right)->collation, MY_COLL_CMP_CONV);
    /*
      DTCollation::set() may have chosen a charset that's a superset of both
      and "left" and "right", so we need to convert both items.
     */
    if (agg_item_set_converter(coll, func_name, left, 1, MY_COLL_CMP_CONV, 1, true) ||
        agg_item_set_converter(coll, func_name, right, 1, MY_COLL_CMP_CONV, 1, true))
      return true;
```

★ 注意 `MY_COLL_CMP_CONV` 与注释：**聚合出的字符集可能是双方的超集，所以两侧都要转换**（不是只转一方）。两侧字符集不兼容时这里直接返回错误。

**⑤ `try_year_cmp_func(type)`** —— YEAR 类型的特判

**⑥ ★ DECIMAL vs 常量字符串：精度保护**

```cpp
  } else if (type == REAL_RESULT &&
             (((*left)->result_type() == DECIMAL_RESULT && !(*left)->const_item() &&
                (*right)->result_type() == STRING_RESULT && (*right)->const_item()) ||
               ((*right)->result_type() == DECIMAL_RESULT && !(*right)->const_item() &&
                (*left)->result_type() == STRING_RESULT && (*left)->const_item()))) {
    /*
     <non-const decimal expression> <cmp> <const string expression>
     or
     <const string expression> <cmp> <non-const decimal expression>

     Do comparison as decimal rather than float, in order not to lose precision.
    */
    type = DECIMAL_RESULT;
  }
```

即 `dec_col = '1.2345678901234567890'` 这种"非常量 decimal 列 vs 常量字符串"**故意不走 REAL**，改走 DECIMAL 以保精度（但列侧仍是列，索引可用）。

**⑦ 常量转换缓存 + 收尾**

```cpp
  left = cache_converted_constant(thd, left, &left_cache, type);
  right = cache_converted_constant(thd, right, &right_cache, type);
  return set_compare_func(owner_arg, type);
```

### 7.5 `set_compare_func()` 与 `comparator_matrix`

```cpp
bool Arg_comparator::set_compare_func(Item_result_field *item, Item_result type) {
  m_compare_type = type;
  owner = item;
  func = comparator_matrix[type];

  switch (type) {
    case ROW_RESULT: {
      uint n = (*left)->cols();
      if (n != (*right)->cols()) {
        my_error(ER_OPERAND_COLUMNS, MYF(0), n);
        return true;
      }
      if (!(comparators = new (*THR_MALLOC) Arg_comparator[n])) return true;
      comparator_count = n;
      for (uint i = 0; i < n; i++) {
        if (comparators[i].set_cmp_func(owner, (*left)->addr(i),
                                        (*right)->addr(i), set_null))
          return true;
```

★ **`comparator_matrix` 定义在 `mysqld.cc:1050`，不在 `item_cmpfunc.cc`**（一个容易找不到的地方）：

```c
arg_cmp_func Arg_comparator::comparator_matrix[5] = {
    &Arg_comparator::compare_string,      // Compare strings
    &Arg_comparator::compare_real,        // Compare float values
    &Arg_comparator::compare_int_signed,  // Compare signed int values
    &Arg_comparator::compare_row,         // Compare row values
    &Arg_comparator::compare_decimal};    // Compare decimal values
```

下标是 `Item_result`：0=STRING / 1=REAL / 2=INT / 3=ROW / 4=DECIMAL。行比较 `ROW_RESULT` 会**递归**为 n 个 `Arg_comparator`。

### 7.6 13 个 `compare_*` 实现

除矩阵里的 5 个，还有一批由 `set_cmp_func` 直接指定或按符号性选择：

| 函数 | 场景 |
|---|---|
| `compare_string` / `compare_binary_string` | 字符串（含二进制） |
| `compare_real` / `compare_real_fixed` | 浮点 |
| `compare_int_signed` / `compare_int_unsigned` / `compare_int_signed_unsigned` / `compare_int_unsigned_signed` | ★ **4 个按符号性组合**——`unsigned` 的比较规则不同（`1 > -1` 在 unsigned 语境下是 false） |
| `compare_decimal` | 定点数 |
| `compare_datetime` / `compare_time_packed` | 日期时间（packed 格式整数比较） |
| `compare_row` | 行（递归子比较器） |
| `compare_json` | JSON |

`compare_null_values()` 处理"任一为 NULL"的情况（按 `set_null` 决定是否置 `owner->null_value`）。

### 7.7 `inject_cast_nodes()`：把隐式转换显式化（WL#12108）

```cpp
  /**
     Comparison function are expected to operate on arguments having the
     same data types. ... This function transforms these internal conversions
     to explicit CASTs so that the internally executed query becomes
     compatible with the standard. At the moment nodes are injected only for
     comparisons between:

        1) temporal types and numeric data types: ... the comparison is
        normally done as DOUBLE, so the arguments which are not floating point
        will get converted to DOUBLE, and also for

        2) comparisons between temporal types: ... the comparison happens as
        DATETIME if the arguments have different data types, so ... the
        temporal arguments that are not DATETIME will get wrapped in a CAST
        to DATETIME.

     WL#12108; This function will limit itself to comparison between regular
     functions, aggregation functions and fields ... (so this excludes stored
     procedures, stored functions, GC, user defined functions, as well as
     literals).
```

⇒ 即：**为了让"实际执行的查询"符合标准语义**，把内部偷偷做的转换包成显式 `CAST` 节点注入回表达式树（用于 `EXPLAIN` 展示与语义对齐）。限定只对"执行期恒定"的节点做（排除 SP、UDF、字面量——后者由 `fold_condition` 处理）。

### 7.8 实践含义速查

| 现象 | 原因（定位到本节哪一步） |
|---|---|
| `int_col = 1.5` 走不了索引 | 7.3：`item_cmp_type(INT, REAL)` → REAL，列被转成浮点 |
| `int_col = dec_col` 精度好但慢 | 7.3：→ DECIMAL |
| `date_col = '2024-01-01'` **能**走索引 | 7.4②：常量字符串在 `set_cmp_func` 期转成 DATETIME 并缓存，列侧不动 |
| `dec_col = '1.234...'` 保精度 | 7.4⑥：故意不走 REAL 改走 DECIMAL |
| 两侧字符串字符集不同导致报错/转换 | 7.4④：`MY_COLL_CMP_CONV` 聚合，超集时**两侧都转换** |
| 含 JSON 的比较 | 7.4①：最高优先级短路到 `compare_json` |
| `(a,b) = (c,d)` | 7.5：ROW_RESULT → 递归 n 个子 `Arg_comparator` |

---

## 八、★ IN 谓词、子查询与缓存

> `IN` 是 SQL 最高频谓词之一，但它的实现分散在**两套完全不同的机制**里：常量列表走"排序数组二分"，子查询走"策略选择 + 多引擎"。这一章把它们连同 `Item_cache` 一起讲清。

> ⚠️ **先废掉三个 5.x 遗留的旧名字**（8.0.39 实测全库 0 匹配，别再写进文档）：`in_constant`（8.0 叫 **bisection**）、`subselect_single_column_engine`（默认执行器是 `SubqueryWithResult`）、`Item_func_not_in`（取反靠 `Item_func_opt_neg::negated` 位翻转，**不存在这个类**）。

### 8.1 三条路径总览

| 路径 | 决策阶段 | 决策者 | 产物/引擎 |
|---|---|---|---|
| **① 常量列表二分** | **resolve 期**（`Item_func_in::resolve_type()`）一次定死 | `bisection_possible` 四个条件 | `in_vector` 子类（排序数组 + `std::binary_search`） |
| **② 非常量列表线性比较** | 同上（bisection 不可用时） | — | `cmp_items[6]`（每种 `Item_result` 一个 `cmp_item`） |
| **③ `IN (SELECT ...)`** | **两阶段**：resolve 注册候选 → optimize 代价决策 | `Query_block::resolve_subquery()` → `JOIN::decide_subquery_strategy()` | `SubqueryWithResult` / `subselect_indexsubquery_engine` / `subselect_hash_sj_engine` |

### 8.2 ① 常量列表二分：`bisection_possible` 的四条件

```cpp
  /*
    First conditions for bisection to be possible:
     1. All types are similar, and
     2. All expressions in <in value list> are const (for execution)
     3. No JSON is compared (in such case universal JSON comparator is used)
  */
  bool bisection_possible = type_cnt == 1 &&       // 1
                            m_values_are_const &&  // 2
                            !compare_as_json;      // 3
```

注意条件 2 用的是 **`const_for_execution()`** 而非 `const_item()` ⇒ `IN (?, ?)`、含用户变量的列表**也算常量**，只是每次执行要重新填充（`m_need_populate`，`cleanup()` 里重置——**这是 PS 二次执行必须重填的原因**）。

**ROW 类型有额外限制**（注释把理由写得很清楚）：

```cpp
    if (cmp_type == ROW_RESULT &&
        !((ignore_unknown() && !negated) ||                     // 3
          (!list_contains_null() && !args[0]->is_nullable())))  // 4
      bisection_possible = false;
```

### 8.3 ★ `in_vector` 体系：为什么按类型各写一份

因为二分需要**紧凑、同构、可排序的原生数组**。MySQL 没有统一 value 对象（只有 `val_*` 协议），也没用 C++ 模板，于是每种 `Item_result` 各写一份——**代价是代码重复，收益是零装箱 + 零虚调用**。

| 类 | 存储 | 排序/查找 |
|---|---|---|
| `in_string` | `Mem_root_array<String>` + **指针数组** | `Cmp_string(cs)`（`strnncollsp`）；注释：*"String objects are not sortable, sort pointers instead."* |
| `in_longlong` | `Mem_root_array<packed_longlong>`（`{longlong val; bool unsigned_flag;}`） | `cmp_longlong()` |
| `in_datetime_as_longlong` / `in_time_as_longlong` / `in_datetime` | 同上 | 分别用 `val_date_temporal()` / `val_time_temporal()` / `get_datetime_value()`（带转换警告） |
| `in_double` / `in_decimal` | `Mem_root_array<double>` / `<my_decimal>` | 原生 `std::sort` + `std::binary_search` |
| `in_row` | `Mem_root_array<cmp_item_row>` + 指针数组 | `cmp_item_row::compare()`（递归到列） |

★ **`fill()` 显式跳过 NULL**——这是 NULL 语义的根源：

```cpp
bool in_vector::fill(Item **items, uint item_count) {
  m_used_size = 0;
  for (uint i = 0; i < item_count; i++) {
    set(m_used_size, items[i]);
    /*
      We don't put NULL values in array, to avoid erroneous matches in
      bisection.
    */
    if (!items[i]->null_value) m_used_size++;  // include this cell in array.
  }
  sort_array();
  return m_used_size < item_count;  // True = at least one null value found.
}
```

⇒ **`m_size`（分配容量）与 `m_used_size`（有效容量）必须是两个字段**，NULL 的存在性单独由 `have_null` 记录。

**`in_longlong` 的符号陷阱**（`cmp_longlong()` 注释原文）：

```cpp
  if (a->unsigned_flag != b->unsigned_flag) {
    /*
      One of the args is unsigned and is too big to fit into the
      positive signed range. Report no match.
    */
    if ((a->unsigned_flag && ((ulonglong)a->val) > (ulonglong)LLONG_MAX) ||
        (b->unsigned_flag && ((ulonglong)b->val) > (ulonglong)LLONG_MAX))
      return a->unsigned_flag ? 1 : -1;
```

**`create_item()` / `value_to_item()` / `compare_elems()` 给谁用？** 全库只有 **range optimizer** 用（`range_analysis.cc`）——把排序后的 IN 列表转成一组 `SEL_ARG` 区间。`NOT IN` 的 `SEL_TREE` 是手工拼的，且有上限：

```cpp
      const uint NOT_IN_IGNORE_THRESHOLD = 1000;
      if (op->m_const_array->m_used_size == 0 ||
          op->m_const_array->m_used_size > NOT_IN_IGNORE_THRESHOLD)
        return nullptr;
```

**"整型化"关键分支**：`date_col IN ('2024-01-01','2024-01-02')` 若常量能全部转成整型 → `in_datetime_as_longlong`；否则 → `in_datetime`（边转边告警）。源码注释：*"IN must compare INT columns and constants as int values (the same way as equality does)."*

### 8.4 ★ NULL 语义速查

| 表达式 | 推导 | 结果 |
|---|---|---|
| `x IN (NULL)` | 数组空 + `have_null=true` ⇒ `null_value = (false \|\| (true && true))` | **UNKNOWN** |
| `NULL IN (1,2)` | `args[0]->null_value` ⇒ `null_value=true` | **UNKNOWN** |
| `x NOT IN (1,NULL)`，x=3 | `have_null=true` ⇒ `null_value=true` ⇒ 返回 0 | **UNKNOWN（不是 TRUE）** |

⇒ 两个 `return` 分别是 `!null_value && tmp != negated` 与 `!null_value && negated`——**只要 `null_value` 为真就返回 0**。这就是 `NOT IN (SELECT ...)` 碰到 NULL 就返回空结果集的根源。

子查询侧的三态→二态翻译（`Item_exists_subselect::translate()`）：**同一个 UNKNOWN，`IS TRUE` 变 FALSE、`IS NOT TRUE` 变 TRUE**。

### 8.5 ★ `Item_in_optimizer` 是什么（连源码自己都在吐槽）

```
  NOTE
    It is not quite clear why the above listed functionality should be
    placed into a separate class called 'Item_in_optimizer'.
```

它**不是** `Item_func_in` 的伙伴，而是 **`Item_in_subselect` 的包装器**，做三件事：

1. **左表达式只求值一次**——IN→EXISTS 会把左表达式包成 `Item_ref` 注入子查询 WHERE/HAVING，可能多处引用；`fix_left()` 建 `Item_cache` 缓存它
2. **`NULL IN (...)` 短路**
3. **NOT / IS TRUE / IS FALSE 的翻译**：`return subqpred->translate(null_value, result);`

它还有 `result_for_null_param` 三态缓存（`UNKNOWN`/`FALSE`/`TRUE`），用于**非相关子查询 + 左值全 NULL** 时不再重跑子查询。

### 8.6 ③ `IN (SELECT ...)`：三个引擎与两阶段决策

**阶段 1（resolve）**：`select_in_like_transformer()` 里先占位：

```cpp
  /*
    If we didn't choose an execution method up to this point, we choose
    the IN=>EXISTS transformation, at least temporarily.
  */
  if (strategy == Subquery_strategy::UNSPECIFIED)
    strategy = Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT;
```

注意 **"at least temporarily"**——若已被注册为 semi-join 候选（`Query_block::resolve_subquery()`，16 条适用性条件），则跳到 semi-join 路径。

**阶段 2（optimize）**：`JOIN::decide_subquery_strategy()` 基于代价在 EXISTS 与物化间选择：

- 选 **EXISTS** → `finalize_exists_transform()`：给 unit 加 `LIMIT 1`
- 选 **物化** → `finalize_materialization_transform()`：★ **先把 IN→EXISTS 注入的条件拆掉再物化**

```cpp
  // We need to undo several changes which IN->EXISTS had done:
  /*
    The conditions added by in2exists depend on the concrete value from the
    outer query block, so they need to be removed before we materialize.
  */
  if (join->where_cond)
    join->where_cond = remove_in2exists_conds(join->where_cond);
```

**三个引擎**：

| 引擎 | 适用 | 谁装配 |
|---|---|---|
| `SubqueryWithResult` | **默认**（完整执行子查询） | 无引擎被装 → `subquery->exec()` |
| `subselect_indexsubquery_engine` | 单表 + 可用索引查找（EXPLAIN 的 `unique_subquery` / `index_subquery`） | `JOIN::replace_index_subquery()`（optimize 期，物化之后） |
| `subselect_hash_sj_engine` | 物化 + hash/index 查找 | `finalize_materialization_transform()` |

★ **`subselect_hash_sj_engine` 继承 `subselect_indexsubquery_engine`**——物化完之后，查找部分复用父类的 index lookup。

★ **策略可回滚、每次执行重选**（`Item_in_subselect::cleanup()`）：

```cpp
    case Subquery_strategy::SUBQ_EXISTS:
      /*
        Back to EXISTS_OR_MAT, so that next execution of this statement can
        choose between the two.
      */
      strategy = Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT;
```

### 8.7 `Item_subselect` 家族

6 个 Item 类 + 3 个执行体：

| 类 | 语义 |
|---|---|
| `Item_subselect` | 基类。**一定是叶子**；`substitution` 是**整个家族的改写钩子**（`fix_fields` 里 `*ref = substitution`，IN 变 `Item_in_optimizer` 就靠它）；`not_null_tables()` **恒 0** |
| `Item_singlerow_subselect` | 标量子查询。多列时 `row` 是 `Item_cache*` 数组（⇒ `Item_cache_row` 的用武之地）。`select_transformer()` 做"无表/无聚合/无 WHERE 的单列子查询直接替身"优化 |
| `Item_maxmin_subselect` | `> ALL (SELECT ...)` → `> (SELECT MIN(...))` 的改写产物 |
| `Item_exists_subselect` | EXISTS。持有 `strategy`、`value_transform`（`[NOT] IN/EXISTS [ IS [NOT] TRUE|FALSE ]`） |
| `Item_in_subselect` | **`Item_exists_subselect` 的子类**——IN 是一种 EXISTS |
| `Item_allany_subselect` | ALL/ANY，持有 `Comp_creator *func` |

★ **`fix_fields()` 为什么要调两次**（源码注释）：第一次解析子查询并创建 `Item_in_optimizer` 替身；第二次从 `Item_in_optimizer::fix_fields()` 进入，完成解析。

**semi-join 改写**：`Item_in_subselect` 被确认 semi-join 后**整个消失**（变成 semi-join nest，`strategy = SEMIJOIN`）。失败则打回 `UNSPECIFIED`，还有第二条路——**转派生表**：

```
    FROM ot WHERE ot.x IN (SELECT y FROM it1, it2)
    =>
    FROM ot LEFT JOIN (SELECT DISTINCT y FROM it1, it2) AS derived
            ON ot.x=derived.y WHERE derived.y IS NOT NULL.
```

### 8.8 ★ `Item_cache` 9 个类

解决"一个表达式要被求值 N 次，但值在一次求值窗口内不变"。实测使用场景：IN 子查询左表达式、标量子查询每列、MIN/MAX 当前极值、窗口 lead/lag/first_value、`Arg_comparator` 常量侧转换、日期常量比较、SP 变量 holder、JSON 常量。

（⚠️ **ORDER BY / GROUP BY 的排序键不是用 `Item_cache`**——走 `Filesort` 的 `Sort_param`。）

**两阶段惰性求值**：`store(item)` 只记 `example` 并置 `value_cached=false` → 首次 `val_xxx()` 触发 `cache_value()`。

★ **`Item_cache_datetime` 是唯一有"两种缓存形态"的**——为避免 DATETIME 的 INT↔STRING 来回转换，提供 `cache_value()`（字符串）与 `cache_value_int()`（packed longlong），**两个标志位互斥**，而 `result_type()` **恒为 `STRING_RESULT`**（所以 `val_str()` 里还留了"INT 存的但要返回字符串"的兜底转换）。

⚠️ **`Item_cache_str::cache_value()` 必须拷贝**（注释解释了为什么缓存不能只存指针）：

```cpp
  else if (value != nullptr && value->ptr() != buffer) {
    /*
      We copy string value to avoid changing value if 'item' is table field
      in queries like following (where t1.c is varchar): ...
    */
    value_buff.copy(*value);
```

★ **`set_used_tables(1)` 的"防再缓存"技巧**（第七章 7.4② 已埋伏笔，这里展开）：

`Item_cache::used_table_map` **初值为 0** ⇒ `used_tables() == 0` ⇒ **一个新 new 出来的 `Item_cache` 默认就是"常量"**。于是 `Arg_comparator::set_cmp_func` 末尾的：

```cpp
  left = cache_converted_constant(thd, left, &left_cache, type);
```

其中 `cache_converted_constant` 的条件是 `(*value)->const_for_execution() && type != (*value)->result_type()` ⇒ **刚建好的 `Item_cache_datetime` 会被再包一层 cache**（甚至反复）。`set_used_tables(1)` 把它标成"依赖表 0" ⇒ `const_for_execution() == false` ⇒ 放行。

同一个技巧还有两处用法：反向禁止缓存（`item_xmlfunc.cc`：`fake->set_used_tables(RAND_TABLE_BIT)`，注释 *"Don't cache fake because its value will be changed during comparison."*）、正向传播（`Item_in_optimizer::fix_left()`）。

**`Item_cache_row`**：每列一个独立 `Item_cache`；**零行子查询 → 整行全 NULL**（`store_null()`，避免读到未初始化列）；`val_*()` 全部 `illegal_method_call`，只能通过 `element_index(i)` 访问。

★ 另有一个"自我保护"：`is_non_const_over_literals()` **恒返回 true** ⇒ `Item_cache` **永远不被当成纯字面量常量**，从而阻止 `remove_const_conds` 把它求值后删掉。

### 8.9 四个此前 0 覆盖的特殊 Item 类

**① `Item_splocal`**（SP 局部变量）——值**不在 Item 里，在运行时 `sp_rcontext` 槽位里**，Item 只是"槽位号 + 类型声明"：

```cpp
Item *Item_splocal::this_item() {
  return current_thd->sp_runtime_ctx->get_item(m_var_idx);
}
```

`used_tables()` 返回 `INNER_TABLE_BIT`。`limit_clause_param` 的作用：**binlog 写语句时，LIMIT 子句里的 SP 变量不能被替换成 `NAME_CONST(...)`，否则从库解析失败**。

**② `Item_trigger_field`**（`NEW.x` / `OLD.x`）——★ 类注释解释了"为什么它的绑定不走 `fix_fields`"：

> *"For this item main part of actual binding to Field object happens not during fix_fields() call (like for Item_field) but right after parsing of trigger definition, when table is opened, with special setup_field() call."*

权限按读写分：读 → `SELECT_ACL`，写（`SET NEW.x=`）→ `UPDATE_ACL`；`read_only` 由"是否 BEFORE + NEW"决定（`!(new_row && action_time == TRG_ACTION_BEFORE)`）。

⚠️ 一个易错点（`bind_fields()` 注释）：**OLD 行的位置会变**——REPLACE 用 `TABLE::record[1]` 而 DELETE 用 `record[0]`。

**③ `Item_default_value`**（`DEFAULT(col)`）——★ `used_tables()` **返回 0（常量）**，因为默认值不依赖当前行；权限检查**委托给 `arg`**（`DEFAULT(col)` 里的 `col`），自己返回 false 不做二次检查。

**④ `Item_insert_value`**（`VALUES(col)`）——★ 用 **`RAND_TABLE_BIT` 防折叠**：

```cpp
  /*
   We use RAND_TABLE_BIT to prevent Item_insert_value from
   being treated as a constant and precalculated before execution
  */
  table_map used_tables() const override { return RAND_TABLE_BIT; }
```

（同样的手法还出现在 `Item_values_column`——保证 `VALUES ROW(...)` 永远逐行扫描、不被常量化；以及 `Item_is_not_null_test`——防止从 HAVING 移到 WHERE。）

### 8.10 `Item_cond` 与 NOT ALL/ANY

**德摩根**：`Item_cond_and::truth_transformer()` 逐参数变换后 `new Item_cond_or(list)`（OR 反之）。`truth_transform_arguments()` 对返回 null 的参数**包一层 `Item_func_not`**。

**AND/OR 的 `not_null_tables` 差异**：AND 取**并集**、OR 取**交集**（OR 里"某一项 null-rejecting"不意味着整个 OR null-rejecting）。

★ **`Item_func_not_all` / `Item_func_nop_all` 互为逆变换**——注释即规则：

```cpp
Item *Item_func_nop_all::truth_transformer(THD *, Bool_test test) {
  // "NOT (e $cmp$ ANY (SELECT ...)) -> e $rev_cmp$ ALL (SELECT ...)
  Item_func_not_all *new_item = new Item_func_not_all(args[0]);
  Item_allany_subselect *allany = down_cast<Item_allany_subselect *>(args[0]);
  allany->func = allany->func_creator(false);
  allany->all = !allany->all;
```

而 `Item_func_not_all::not_null_tables()` **恒返回 0**——注释说透：转换后的 ALL 子句"子查询为空时应返回 true（即使左值是 NULL）"，所以它对左值不 null-rejecting。

---

## 九、★ 内置函数：从"名字 → 对象"到"对象 → 结果"

> 本章覆盖内置函数的**完整生命周期**，分三段：
> - **9.1~9.7（创建）**："函数名字符串 → Item 对象"的唯一入口——加内置函数、查某函数参数个数、理解 UDF 怎么接入。
> - **9.8（定型 + 求值）**：对象被创建**之后**如何完成 `resolve_type()`（8.0 的 `fix_length_and_dec`）与 `val_*` 分派——即"new 出来之后怎么算出结果"。
> - **9.9（各族实现）**：算术/比较/逻辑/字符串/控制流/时间各族的代表实现，以及 `Item_typecast_*`、`Item_cache_*` 全族。
>
> ⚠️ **8.0.39 没有 `Create_native_func` 这个类**（全库 0 命中）——5.x 资料里的这个名字已不成立。8.0 的"内置函数"不是类，而是**一堆模板工厂单例**。

### 9.1 `Create_func`：为什么把函数创建从 bison 文法里分离出来

```cpp
class Create_func {
 public:
  virtual Item *create_func(THD *thd, LEX_STRING name,
                            PT_item_list *item_list) = 0;
 protected:
  Create_func() = default;
  virtual ~Create_func() = default;
};
```

官方注释（就在类上方）给了动机：

```
/**
  Public function builder interface.
  The parser (sql/sql_yacc.yy) uses a factory / builder pattern to
  construct an <code>Item</code> object for each function call.
  All the concrete function builders implements this interface,
  either directly or indirectly with some adapter helpers.
  Keeping the function creation separated from the bison grammar allows
  to simplify the parser, and avoid the need to introduce a new token
  for each function, which has undesirable side effects in the grammar.
*/
```

⇒ **核心是不必为每个函数名引入一个 bison token**（token 会污染文法、产生冲突）。

`create_func` 的两条约定（见其 doxygen）：失败返回 `nullptr` 且**已报过错**；**允许修改 THD/LEX**（`safe_to_cache_query`、`uncacheable(UNCACHEABLE_RAND)`、`binlog_row_based_if_mixed`…）——这是 RAND、GET_LOCK、UDF 这类非确定性函数标记自身"不可缓存/影响 binlog 格式"的官方钩子。

**继承关系（8.0.39 实测）**：

```
Create_func（纯虚 create_func）
 ├── Create_qfunc（加 create(thd, db, name, use_explicit_name, list)）
 │    └── Create_sp_func          ← 兜底：db.func() 一律当存储函数
 ├── Create_udf_func              ← UDF 无 db. 限定
 └── Function_factory / Odd_/Even_/Internal_function_factory（模板家族）
```

对外只有两个入口：

```cpp
extern Create_func *find_native_function_builder(const LEX_STRING &name);
extern Create_qfunc *find_qualified_function_builder(THD *thd);
```

### 9.2 ★ 13 个 `SQL_FN_*` 宏（不是 12 个，还有一个 `SQL_FACTORY`）

| # | 宏 | 用途 | 真实用例 |
|---|---|---|---|
| 1 | `SQL_FN(F, N)` | 定长 N 参 | `{"SIN", SQL_FN(Item_func_sin, 1)}`、`{"PI", SQL_FN(Item_func_pi, 0)}` |
| 2 | **`SQL_FACTORY(INST)`** | 有**定制 Instantiator** | `{"BIN", SQL_FACTORY(Bin_instantiator)}`、`{"WEEKDAY", SQL_FACTORY(Weekday_instantiator)}` |
| 3 | `SQL_FN_V(F, MIN, MAX)` | 参数区间 | `{"CONCAT", SQL_FN_V(..., 1, MAX_ARGLIST_SIZE)}`、`{"ATAN", SQL_FN_V(..., 1, 2)}` |
| 4 | `SQL_FN_V_THD(F, MIN, MAX)` | 区间 + 构造要 thd | `{"JSON_EXTRACT", SQL_FN_V_THD(..., 2, MAX_ARGLIST_SIZE)}` |
| 5 | `SQL_FN_V_LIST(F, MIN, MAX)` | 区间 + 整个链表 | `{"LEAST", SQL_FN_V_LIST(Item_func_min, 2, MAX_ARGLIST_SIZE)}` |
| 6 | `SQL_FN_LIST(F, N)` | 定长 + 链表 | `{"JSON_QUOTE", SQL_FN_LIST(..., 1)}` |
| 7 | `SQL_FN_V_LIST_THD(F, MIN, MAX)` | 区间 + 链表 + thd | `{"JSON_ARRAY", SQL_FN_V_LIST_THD(..., 0, MAX_ARGLIST_SIZE)}` |
| 8 | `SQL_FN_ODD(F, MIN, MAX)` | **强制奇数个参数** | `{"JSON_INSERT", SQL_FN_ODD(..., 3, MAX_ARGLIST_SIZE)}` |
| 9 | `SQL_FN_EVEN(F, MIN, MAX)` | **强制偶数个参数** | `{"JSON_ROW_OBJECT", SQL_FN_EVEN(..., 0, MAX_ARGLIST_SIZE)}` |
| 10 | `SQL_FN_INTERNAL(F, N)` | 仅供系统视图引用 | `{"CAN_ACCESS_TABLE", SQL_FN_INTERNAL(..., 2)}` |
| 11 | `SQL_FN_INTERNAL_V(F, MIN, MAX)` | internal + 区间 | `{"IS_VISIBLE_DD_OBJECT", SQL_FN_INTERNAL_V(..., 1, 3)}` |
| 12 | `SQL_FN_LIST_INTERNAL(F, N)` | internal + 链表 | `{"CAN_ACCESS_ROUTINE", SQL_FN_LIST_INTERNAL(..., 5)}` |
| 13 | `SQL_FN_LIST_INTERNAL_V(F, MIN, MAX)` | internal + 链表 + 区间 | `{"INTERNAL_TABLE_ROWS", SQL_FN_LIST_INTERNAL_V(..., 8, 9)}` |

宏定义（几个关键的）：

```cpp
#define SQL_FN(F, N) &Function_factory<Instantiator<F, N>>::s_singleton
#define SQL_FACTORY(INSTANTIATOR) &Function_factory<INSTANTIATOR>::s_singleton
#define SQL_FN_V(F, MIN, MAX) \
  &Function_factory<Instantiator<F, MIN, MAX>>::s_singleton
#define SQL_FN_ODD(F, MIN, MAX)   \
  &Odd_argcount_function_factory< \
      List_instantiator_with_thd<F, MIN, MAX>>::s_singleton
#define SQL_FN_INTERNAL(F, N) \
  &Internal_function_factory<Instantiator<F, N>>::s_singleton
```

⚠️ **一处注释与实现不符的 bug**：`SQL_FN_INTERNAL_V` 的注释写"enforces a check that the argument count is even"，但实现用的是普通 `Instantiator<F,MIN,MAX>`，**没有任何奇偶校验**（对比 `SQL_FN_EVEN` 用的是 `Even_argcount_function_factory`）。注释是从 `SQL_FN_EVEN` 复制粘贴来的。

"无穷参数"哨兵：

```cpp
constexpr auto MAX_ARGLIST_SIZE =
    std::numeric_limits<decltype(PT_item_list().elements())>::max();
```

### 9.3 ★★ `Function_factory` + `Instantiator`：这套模板解决什么问题

**`s_singleton`**：每个 `Function_factory<X>` 自带一个 `static` 成员，定义即初始化：

```cpp
template <typename Instantiator_fn>
class Function_factory : public Create_func {
 public:
  static Function_factory<Instantiator_fn> s_singleton;

  Item *create_func(THD *thd, LEX_STRING function_name,
                    PT_item_list *item_list) override {
    if (check_argcount_bounds(thd, function_name, item_list,
                              m_instantiator.Min_argcount,
                              m_instantiator.Max_argcount))
      return nullptr;
    return m_instantiator.instantiate(thd, item_list);
  }
 private:
  Function_factory() = default;
  Instantiator_fn m_instantiator;
};
```

⇒ ★ **注册 = 取一个地址**。因为 `s_singleton` 是类模板的静态数据成员，`&Function_factory<Instantiator<Item_func_sin,1>>::s_singleton` 这行本身**就是一次模板实例化**——不需要任何运行时注册代码。

**`Instantiator` 的作用**：把 `PT_item_list*` 翻译成 C++ 构造函数的重载形式。它**不是**变参模板展开，而是**类模板 + 一批手工偏特化**：

```cpp
// 主模板：把整个 list 传进去
template <typename Function_class, uint Min_argc, uint Max_argc = Min_argc>
class Instantiator {
 public:
  static const uint Min_argcount = Min_argc;
  static const uint Max_argcount = Max_argc;
  Item *instantiate(THD *thd, PT_item_list *args) {
    return new (thd->mem_root) Function_class(POS(), args);
  }
};

// 定长偏特化：逐个展开
template <typename Function_class>
class Instantiator<Function_class, 3> {
  ...
  Item *instantiate(THD *thd, PT_item_list *args) {
    return new (thd->mem_root)
        Function_class(POS(), (*args)[0], (*args)[1], (*args)[2]);
  }
};
```

`Min/Max_argcount` 是 `static const uint`（编译期常量）⇒ **`check_argcount_bounds` 被内联成常量比较，参数个数校验是编译期烧进二进制的**，不是运行时配置。

8.0.39 存在的偏特化：定长 `<F,0>`…`<F,5>`；区间 `<F,0,1> <F,1,2> <F,1,3> <F,2,3> <F,2,4> <F,2,6> <F,3,5>`。
⇒ ⚠️ 想用 `SQL_FN_V(F, 1, 4)` 但没有对应偏特化、且 `F` 没有 `(POS(), PT_item_list*)` 构造时，会**编译失败**。

**为什么需要这套（源码注释原文）**：

```
  - The parser produces PT_item_list objects of all argument lists, while the
    Item_func subclasses use overloaded constructors,
    e.g. Item_xxx_func(Item*), Item_xxx_func(Item*, Item*), etc.
  - We need to map parser tokens to classes and we don't have reflection.
```

**手工子类 vs 宏+模板**：

| | 手工方案 | 宏+模板方案 |
|---|---|---|
| 代码量 | 347 个类 × 15~25 行 ≈ 5000+ 行 | **347 行**（一行一条） |
| 参数校验 | 347 处重复写，漏写一处就少一次校验 | 只在 `Function_factory::create_func` 写**一次** |
| 单例 | 每个手写 `static` 成员定义 | 模板自动产生 |
| 体积 | — | *"Keeping the templates in anonymous namespaces enables the compiler to inline more and hence keeps the generated code leaner."* |

**定制 Instantiator 的两个真实例子**（它们能改写语法树）：

```cpp
class Bin_instantiator {   // BIN(x) → CONV(x, 10, 2)
  Item *instantiate(THD *thd, PT_item_list *args) {
    Item *i10 = new (thd->mem_root) Item_int(pos, 10, 2);
    Item *i2 = new (thd->mem_root) Item_int(pos, 2, 1);
    return new (thd->mem_root) Item_func_conv(pos, (*args)[0], i10, i2);
  }
};
```

```cpp
/// JSON_LENGTH(doc, path) → JSON_LENGTH(JSON_EXTRACT(doc, path))
class Json_length_instantiator { ... };
```

自定义的 Instantiator 只需满足三件事（`Min_argcount`、`Max_argcount`、`instantiate`），**不必继承任何东西**。

### 9.4 `func_array`：347 条，启动时建哈希

```cpp
static const std::pair<const char *, Create_func *> func_array[] = {
    {"ABS", SQL_FN(Item_func_abs, 1)},
    {"AES_ENCRYPT", SQL_FN_V(Item_func_aes_encrypt, 2, 6)},
    {"BIN", SQL_FACTORY(Bin_instantiator)},
    {"CONCAT", SQL_FN_V(Item_func_concat, 1, MAX_ARGLIST_SIZE)},
    {"SIN", SQL_FN(Item_func_sin, 1)},
    ...
    {"CEIL", SQL_FN(Item_func_ceiling, 1)},
    {"CEILING", SQL_FN(Item_func_ceiling, 1)},   // ★ 两个名字同一个类
    ...
    {"INTERNAL_IS_ENABLED_ROLE", SQL_FN_INTERNAL(..., 2)}};
```

`func_array` 是**编译期静态数组**（`static const`，元素是指向模板单例的常量地址）。哈希表不是编译期的：

```cpp
bool item_create_init() {
  native_functions_hash =
      new Native_functions_hash(std::begin(func_array), std::end(func_array));
}
```

调用点在 `mysqld.cc` 的 `init_errmessage()` 之后，属于**单线程启动阶段**（头文件注释：*"not thread safe, and is intended to be used at server startup only"*）。

**查表前统一 `std::toupper`** ⇒ **这就是 MySQL 函数名大小写不敏感的原因**：

```cpp
    std::string name(lex_name.str, lex_name.length);
    for (auto &it : name) it = std::toupper(it);
```

**★ 没有重复名检测**：`item_create_init()` 就一句范围构造，`std::unordered_map` 对重复 key 静默处理，没有任何 assert/日志/单测。唯一防线是那条 MAINTAINER 注释：

```
  MAINTAINER:
  - Keep sorted for human lookup. At runtime, a hash table is used.
  - do **NOT** conditionally (#ifdef, #ifndef) define a function *NAME*:
    doing so will cause user code that works against a `--without-XYZ` binary
    to fail with name collisions against a `--with-XYZ` binary.
  - keep 1 line per entry, it makes `grep | sort` easier
  - Use uppercase (tokens are converted to uppercase before lookup.)
```

而**同名是被刻意利用的**：`CEIL`/`CEILING` 都指向 `Item_func_ceiling`，`ATAN`/`ATAN2` 都指向 `Item_func_atan`。

### 9.5 ★ UDF 接入

**8 个类 = 2 × 4**：`{UDFTYPE_FUNCTION, UDFTYPE_AGGREGATE} × {STRING, REAL, INT, DECIMAL}`

```cpp
  switch (udf->returns) {
    case STRING_RESULT:
      if (udf->type == UDFTYPE_FUNCTION)
        func = new (thd->mem_root) Item_func_udf_str(pos, udf, item_list);
      else
        func = new (thd->mem_root) Item_sum_udf_str(pos, udf, item_list);
      break;
    case REAL_RESULT: ... Item_func_udf_float / Item_sum_udf_float ...
    case INT_RESULT:  ... Item_func_udf_int   / Item_sum_udf_int   ...
    case DECIMAL_RESULT: ...
    default: my_error(ER_NOT_SUPPORTED_YET, MYF(0), "UDF return type");
```

⚠️ 命名不对称：**REAL 对应的是 `_float` 不是 `_real`**；聚合版是 `Item_sum_udf_*`。基类分别是 `Item_udf_func`（继承 `Item_func`）与 `Item_udf_sum`（继承 `Item_sum`）。

**`Item_udf_func` 的几个关键特性**：

```cpp
  Item_udf_func(...) : Item_func(pos, opt_list), udf(udf_arg) {
    null_on_null = false;      // ★ UDF 不遵循"任一参数 NULL 则结果 NULL"
  }
  enum Functype functype() const override { return UDF_FUNC; }
  bool is_expensive() override { return true; }
  // ★ UDF 禁止用作生成列
  bool check_function_as_value_generator(uchar *checker_args) override {
    func_arg->banned_function_name = func_name();
    return true;
  }
```

**协议在哪实现**：`udf_handler`（`sql_udf.h` 声明，`item_func.cc` 实现），函数指针来自 `udf_func`：`func_init` / `func` / `func_deinit` + 聚合的 `func_clear` / `func_add`。

★ **UDF 没有参数个数检查**——证据链很硬：`struct udf_func` **没有 arity 字段**（只有 `returns`/`type`）；`fix_fields` 里 `f_args.arg_count = arg_count` 是照抄调用点给的个数；逐参数只做 `check_cols(1)`（禁止 `f((a,b))` 多列参数）。⇒ arity 不匹配**不报 SQL 错误**，责任在 UDF 的 `xxx_init` 自己检查 `args->arg_count`。

`fix_fields` 另一个要点：**每次都重新 `find_udf(..., mark_used=true)`**（防止 `DROP FUNCTION` 后指针悬空），并因此 `usage_count++`（有活跃引用时不允许卸载 .so）。

**`xxx_init` 的常量/非常量约定**（`call_init_func()` 注释原文）：

```cpp
    /*
     For a constant argument i, args->args[i] points to the argument value.
     For non-constant, args->args[i] is NULL.
    */
```

⇒ **init 期只有常量参数有值**，UDF 只能靠 `args->arg_type[i]` / `args->lengths[i]` 决定内存分配。

★ **`xxx_init` 在 PREPARE 阶段会被跳过**（`is_in_prepare && !initid.const_item` 时提前返回），只有常量 UDF 在 prepare 期 init。`xxx_init` 返回 `initid.const_item == false` 会让 `used_tables_cache |= RAND_TABLE_BIT`（**标记非确定性、不可缓存**）。

**聚合 UDF 的生命周期差异**：

| 阶段 | 普通 UDF | 聚合 UDF |
|---|---|---|
| 每进入新分组 | — | **`xxx_clear`** |
| 每行 | **`xxx`** | **`xxx_add`** |
| 取结果 | 上次返回值 | `xxx`（仍要提供，用于产出结果） |

```cpp
void Item_udf_sum::clear() { udf.clear(); }    // → (*func_clear)(&initid, &is_null, &error)
bool Item_udf_sum::add()   { udf.add(&null_value); return false; }
```

聚合 UDF 还有三个"不合作"信号：`allow_group_via_temp_table = false`；`reset_field()`/`update_field()` **空实现**；拷贝构造把 `udf.m_original = false`，**只有原始对象负责 `free_handler()`**。

### 9.6 ★ 加一个内置函数要动几处（扩展点清单）

以 `SIN`（最简单的情形：定长 1 参、纯数学、无副作用、非保留字）为例：

**① `item_create.cc` 的 `func_array` 加一行（必需）**

```cpp
    {"SIN", SQL_FN(Item_func_sin, 1)},
```

这是**唯一让名字变得可查的操作**（`item_create_init()` 会遍历数组建哈希）。

**② `item_func.h` 定义类（必需）**

```cpp
class Item_func_sin final : public Item_dec_func {
 public:
  Item_func_sin(const POS &pos, Item *a) : Item_dec_func(pos, a) {}
  double val_real() override;
  const char *func_name() const override { return "sin"; }
  enum Functype functype() const override { return SIN_FUNC; }
};
```

★ 构造函数签名 `(const POS&, Item *)` 必须与 `Instantiator<Item_func_sin,1>` 的偏特化匹配——**不匹配就是编译错误**，这是模板体系自带的静态契约检查。

**③ `item_func.cc` 实现（必需）**

```cpp
double Item_func_sin::val_real() {
  assert(fixed == 1);
  double value = args[0]->val_real();
  if ((null_value = args[0]->null_value)) return 0.0;
  return sin(value);
}
```

**④ `Functype` 枚举（可选）**——只有需要被优化器特殊识别才加（如 `NOW_FUNC`、`TYPECAST_FUNC`）。不加则基类默认返回 `UNKNOWN_FUNC`，函数照样能用。

**★ 不需要动的地方**（与直觉相反）：

- **`sql_yacc.yy` 不用改**——`SIN` 走 `IDENT_sys` 泛型函数调用路径，注册进 `func_array` 即可。**只有新函数名是保留字/关键字时才需要动 `lex.h` + `sql_yacc.yy`**
- **`item_create_init()` 不用改**——它遍历数组，自动纳入
- **`item_create.h` 不用改**——没有 `Create_func` 子类要声明

⇒ **一句话清单：`func_array` 一行 + 类定义 + `val_*()` 实现（+ 可选的 `Functype`）**。

### 9.7 与 `functype()`（122 值）的关系

**不是一一对应，而是 N:1 且稀疏**：

- `CEIL` 和 `CEILING` 两个注册名 → 同一个 `Item_func_ceiling` → **同一个 `CEILING_FUNC`**（2 : 1）
- 大量条目**根本不 override `functype()`**，返回基类默认 `UNKNOWN_FUNC`
- 反过来，**`UDF_FUNC` 一个类对应无数个 UDF 名字**（1 : N）
- 枚举里还有非 SQL 函数的值：`FUNC_SP`、`UDF_FUNC`、`DD_INTERNAL_FUNC`、`DATETIME_LITERAL`、`GSYSVAR_FUNC`…

⇒ **`functype()` 只是优化器/重写规则的"身份标签"**，只给需要被特殊处理的函数发一个；**注册（`func_array`）才是"名字 → 类"的唯一入口**。

---

## 十、参数

与 Item / 表达式直接相关的参数不多，主要有：

| 参数 | 默认 | 说明 |
|---|---|---|
| `div_precision_increment` | **4** | `/` 运算结果的小数位增量 |
| `max_allowed_packet` | 64MB | 间接约束表达式结果的字符串长度 |

表达式求值的**类型规则**主要由 SQL 标准与 `sql_mode` 决定（如 `MODE_NO_UNSIGNED_SUBTRACTION` 会让无符号减法返回负数，见 `Item_func_minus::resolve_type`）。

### 9.8 ★ `Item_func` 实现骨架：函数被创建之后怎么算出结果

> 本节补 9.1~9.7（函数工厂）留下的缺口：**`Create_func` 把 `"+"` 变成 `Item_func_plus` 之后，这个对象如何完成「定型 → 求值」的全流程**。文档此前对 `Item_func::val_int`、`set_data_type_from_item` 是 0 覆盖。
>
> 补充源码坐标：`[F1]` sql/item.h、`[F2]` sql/item_func.h、`[F3]` sql/item_func.cc、`[F4]` sql/item.cc、`[F5]` sql/sql_const.h、`[F6]` sql/item_strfunc.h。

#### 9.8.1 ★ 8.0.39 命名核实（务必以此为准）

| 你可能在别处看到的 | 8.0.39 真实情况 |
|---|---|
| `fix_length_and_dec()` | **已改名**为 `Item::resolve_type(THD*)`（纯虚） |
| `Item_func_numhybrid_func` | 实际类名 **`Item_func_numhybrid`** |
| **`Item_func_add`** | **不存在**——加号的类是 **`Item_func_plus`**；`Item_func_additive_op` 是 `+`/`-` 的公共基类 |
| `Item_decimal_func` | 不存在；DECIMAL 走 `Item_func_numhybrid` 的 `DECIMAL_RESULT` 分支 |
| `agg_result_type()` | 已被移除，等价于 `Item::aggregate_type()` |
| `agg_cmp_type()` | 仅剩 `item_cmpfunc.cc` 里的文件内 static 版本（只被 `Item_func_between` 用一次） |

#### 9.8.2 全链路总览

```
 语法树/工厂(九章)              resolver 阶段                      执行阶段
 ─────────────────            ──────────────                    ─────────
 "a + 1" → Item_func_plus
   args={a,1} arg_count=2
   hybrid_type=REAL_RESULT(初值)
        │
        ▼
   Item_func::fix_fields()
     ├─ check_stack_overrun()              防递归爆栈
     ├─ used_tables_cache = get_initial_pseudo_tables()
     ├─ for each arg: fix_func_arg()
     │     ├─ arg->fix_fields()            递归定型子树
     │     ├─ arg->check_cols(1)           标量检查
     │     ├─ set_nullable(|)              可空性上浮
     │     ├─ used_tables_cache |=         依赖表上浮（含伪表位）
     │     └─ not_null_tables_cache |=     仅当 null_on_null
     ├─ resolve_type() → resolve_type_inner()  ← 8.0 的 fix_length_and_dec
     └─ fixed = true
                                                    │
                                                    ▼
                                          val_int()/val_real()/
                                          val_decimal()/val_str()
                                                    │
                                                    ▼
                                     Item_func_numhybrid::val_*()
                                       switch (hybrid_type)
                                         INT_RESULT     → int_op()
                                         REAL_RESULT    → real_op()
                                         DECIMAL_RESULT → decimal_op()
                                         STRING_RESULT  → str_op()/date_op()
```

#### 9.8.3 `Item_func` 类骨架：数据成员即"契约"

```cpp
// [F2] sql/item_func.h
class Item_func : public Item_result_field {
 protected:
  Item **args;                          // 参数指针数组
 private:
  Item *m_embedded_arguments[2];        // ★ 小对象优化：≤2 参直接内嵌，零分配
  bool alloc_args(MEM_ROOT *mem_root, unsigned num_args) {
    if (num_args <= array_elements(m_embedded_arguments)) {
      args = m_embedded_arguments;
    } else {
      args = mem_root->ArrayAlloc<Item *>(num_args);
      if (args == nullptr) { arg_count = 0; return true; }   // OOM 时置 0，下游不解引用
    }
    arg_count = num_args;
    return false;
  }
 public:
  uint arg_count;
  ...
 protected:
  /* ★ "任一参数为 NULL 则结果为 NULL"的声明式开关（默认 true）。
     它只影响 not_null_tables_cache 的累积，并不自动替你写 NULL 检查。
     COALESCE/IFNULL/IF/CASE/ISNULL/存储过程 会显式置 false。 */
  bool null_on_null{true};
  uint allowed_arg_cols{1};             // 0 = "我的列数跟第一个参数走"
  table_map used_tables_cache{0};       // ★ 缓存，不是实时计算
  table_map not_null_tables_cache{0};
};
```

⇒ `Item_func` 相对 `Item` 新增的**全部状态**就是这四个成员。`args` 用**小对象优化**（1~2 参内嵌，覆盖绝大多数标量函数）；`used_tables_cache` / `not_null_tables_cache` 是**缓存**（resolve 期算一次，`used_tables()` 直接返回）。

`functype()` **不参与求值**，只用于"这个函数是谁"的身份识别：`eq()` 里先比 `functype()` 再比 `func_name()` 再递归 `AllItemsAreEqual`；优化器对特定 `Functype` 做特化改写。

#### 9.8.4 `Item_func::fix_fields()`：比基类多了什么

基类只有 4 行（`[F4]`）：`assert(is_contextualized()); fixed = true; return false;`

```cpp
// [F3] sql/item_func.cc
bool Item_func::fix_fields(THD *thd, Item **) {
  assert(fixed == 0 || basic_const_item());
  uchar buff[STACK_BUFF_ALLOC];
  Condition_context CCT(thd->lex->current_query_block());   // RAII：压入当前 Query_block

  used_tables_cache = get_initial_pseudo_tables();          // ★ 先并入自身固有伪表位
  not_null_tables_cache = 0;

  /* 注释：某些平台一次递归要 >22KB，故用 STACK_MIN_SIZE * 2 */
  if (check_stack_overrun(thd, STACK_MIN_SIZE * 2, buff)) return true;

  if (arg_count) {
    for (Item **arg = args, **arg_end = args + arg_count; arg != arg_end; arg++) {
      if (fix_func_arg(thd, arg)) return true;             // ★ 自底向上定型子树
    }
  }
  if (resolve_type(thd) || thd->is_error()) return true;   // 部分实现还不 error-safe
  fixed = true;
  return false;
}
```

单参数的收尾（**"元数据上浮"的全部**）：

```cpp
// [F3]
bool Item_func::fix_func_arg(THD *thd, Item **arg) {
  if ((!(*arg)->fixed && (*arg)->fix_fields(thd, arg))) return true;
  Item *item = *arg;

  if (allowed_arg_cols) {
    if (item->check_cols(allowed_arg_cols)) return true;   // 行值传标量函数 → 报错
  } else {
    assert(arg == args);
    allowed_arg_cols = item->cols();                       // 从首参现取
  }

  set_nullable(is_nullable() || item->is_nullable());      // ★ 可空性取或
  used_tables_cache |= item->used_tables();                // ★ 依赖表取或
  if (null_on_null) not_null_tables_cache |= item->not_null_tables();
  add_accum_properties(item);                              // PROP_SUBQUERY/AGGREGATION...
  return false;
}
```

#### 9.8.5 ★ 结果类型推导：`resolve_type()` 与"参数 → 结果元数据"工具箱

**三段式**：`resolve_type()` → `resolve_type_inner()` → `propagate_type()`

```cpp
// [F1] —— 纯虚，职责在注释里写得很清楚
  virtual bool resolve_type(THD *thd) = 0;

// [F2] —— Item_func 的默认实现极简（只为 PS 参数兜底）
  bool resolve_type(THD *thd) override {
    return param_type_uses_non_param(thd);
  }
  virtual bool resolve_type_inner(THD *) { assert(false); return false; }

// [F3] —— 为什么拆两个？预备语句的 '?' 第一次 resolve 时还没类型，必须事后灌进去
bool Item_func::propagate_type(THD *thd, const Type_properties &type) {
  assert(data_type() == MYSQL_TYPE_INVALID);
  for (uint i = 0; i < arg_count; i++) {
    if (args[i]->data_type() == MYSQL_TYPE_INVALID)        // ★ 先向下灌给 '?'
      if (args[i]->propagate_type(thd, type)) return true;
  }
  if (resolve_type_inner(thd)) return true;                // ★ 再回头跑一次自己的定型
  assert(data_type() != MYSQL_TYPE_INVALID);
  return false;
}
```

`MYSQL_TYPE_INVALID` 就是"我还没有类型信息"的哨兵值。`propagate_type` 让 `resolve_type_inner()` 可被复用两次。

**工具箱**：

```cpp
// [F1] ① set_data_type_from_item() —— "我的类型 = 参数的类型"（一次性拷 5 个属性）
  inline void set_data_type_from_item(const Item *item) {
    set_data_type(item->data_type());
    collation = item->collation;
    max_length = item->max_length;
    decimals   = item->decimals;
    unsigned_flag = item->unsigned_flag;
  }
```

```cpp
// [F4] ② Item::aggregate_type() —— 多参数类型合并（8.0 取代 agg_result_type）
void Item::aggregate_type(Bounds_checked_array<Item *> items) {
  uint itemno = 0;
  const uint count = items.size();
  while (itemno < count && items[itemno]->data_type() == MYSQL_TYPE_NULL)
    itemno++;                                    // ★ 字面 NULL 不参与合并
  if (itemno == count) { set_data_type(MYSQL_TYPE_NULL); return; }

  enum_field_types new_type = real_data_type(items[itemno]);
  uint8 new_dec = items[itemno]->decimals;
  bool new_unsigned = items[itemno]->unsigned_flag;
  bool mixed_signs = false;

  for (itemno = itemno + 1; itemno < count; itemno++) {
    if (items[itemno]->data_type() == MYSQL_TYPE_NULL) continue;
    new_type = Field::field_type_merge(new_type, real_data_type(items[itemno]));
    mixed_signs |= (new_unsigned != items[itemno]->unsigned_flag);
    new_dec = max<uint8>(new_dec, items[itemno]->decimals);
  }
  /* ★ 符号混合时"升档"：TINY→SHORT→INT24→LONG→LONGLONG→NEWDECIMAL */
  if (mixed_signs && is_integer_type(new_type)) { ... }

  set_data_type(real_type_to_type(new_type));
  decimals = new_dec;
  unsigned_flag = new_unsigned && !mixed_signs;
  max_length = 0;
}
```

⇒ 三条规则：① 字面 `NULL` 不参与合并；② 两两合并委托 `Field::field_type_merge()`，小数位取 `max`；③ **符号混合时整数类型向上抬一级**（避免值域截断），且 `unsigned_flag` 置 false。这是 `GREATEST/LEAST/COALESCE/CASE/UNION` 的通用合并器。

```cpp
// [F2] ③ agg_arg_charsets*() —— 三个薄封装，把 func_name() 自动填进报错信息
  bool agg_arg_charsets(DTCollation &c, Item **items, uint nitems, uint flags, int item_sep);
  /* 字符串结果类（如 CONCAT）：全数值时转 @@character_set_connection，允许 DERIVATION_NONE */
  bool agg_arg_charsets_for_string_result(DTCollation &c, Item **items, uint nitems, int item_sep = 1);
  /* 比较类（如 a=b / LIKE）：不转 @@character_set_connection，不允许 DERIVATION_NONE */
  bool agg_arg_charsets_for_comparison(DTCollation &c, Item **items, uint nitems, int item_sep = 1);
```

（`item_sep` 用于"隔一个取一个"的参数布局：`BETWEEN` 与 `IN`。）

#### 9.8.6 ★ `val_*` 的分派链：中间层提供默认实现

**结论先行**：`Item` 里四个取值方法全是**纯虚**；`Item_func` 只补了 `val_decimal`（从 `val_int` 转）与 `is_null`：

```cpp
// [F3]
my_decimal *Item_func::val_decimal(my_decimal *decimal_value) {
  assert(fixed);
  longlong nr = val_int();
  if (null_value) return nullptr;
  int2my_decimal(E_DEC_FATAL_ERROR, nr, unsigned_flag, decimal_value);
  return decimal_value;
}
// [F2]
  bool is_null() override { return update_null_value() || null_value; }
```

⇒ **任何直接继承 `Item_func`（不经中间层）的类都必须自己实现 `val_int/val_real/val_str`**（如 `Item_func_if`、`Item_func_case`）。

```
                       Item（val_int/val_real/val_str/val_decimal 全纯虚）
                         │
                    Item_result_field → Item_func（只补 val_decimal + is_null）
                         │
   ┌──────────────┬──────┴────────┬──────────────────┐
   ▼              ▼               ▼                  ▼
Item_int_func  Item_real_func  Item_str_func   Item_func_numhybrid
 INT_RESULT     REAL_RESULT     STRING_RESULT   hybrid_type 动态
   │              │               │                  │
 val_int()      val_real()      val_str()       int_op()/real_op()
 = 子类必实现    = 子类必实现    = 子类必实现    decimal_op()/str_op()
   │              │               │              date_op()/time_op()
 其余由 int 转   其余由 real 转   其余由 str 转    val_* 全由 switch 派发
```

**`Item_int_func`**——构造里就 `set_data_type_longlong()`（**整数函数在"出生"时类型已定**）：

```cpp
// [F3]
double Item_int_func::val_real() {
  return unsigned_flag ? (double)((ulonglong)val_int()) : (double)val_int();  // ★ 先判 unsigned
}
String *Item_int_func::val_str(String *str) {
  longlong nr = val_int();
  if (null_value) return nullptr;
  str->set_int(nr, unsigned_flag, collation.collation);
  return str;
}
```

**`Item_real_func`**——`val_int()` 走 `llrint_with_overflow_check()`（**不是裸截断**，溢出报错而非静默 UB）。
**`Item_str_func`**——`resolve_type()` 里只做一件事：给所有 `?` 参数默认 VARCHAR（`param_type_is_default(thd, 0, -1)`）。

**★ `Item_func_numhybrid`：本节最重要的中间层**

```cpp
// [F2]
class Item_func_numhybrid : public Item_func {
 protected:
  Item_result hybrid_type;          // 解析期算出的"实际计算域"
 public:
  Item_func_numhybrid(Item *a) : Item_func(a), hybrid_type(REAL_RESULT) {
    collation.set_numeric();
  }
  enum Item_result result_type() const override { return hybrid_type; }
  enum_field_types default_data_type() const override { return MYSQL_TYPE_DOUBLE; }
  virtual void set_numeric_type() = 0;                    // 子类实现，resolve 期调用

  /* 四个 val_* 都是"分派中间层"（见下） */
  double val_real() override;  longlong val_int() override;
  my_decimal *val_decimal(my_decimal *) override;  String *val_str(String *) override;

  /* ★ 子类唯一必须提供的东西：六个纯虚 *_op() */
  virtual longlong int_op() = 0;
  virtual double real_op() = 0;
  virtual my_decimal *decimal_op(my_decimal *decimal_value) = 0;
  virtual String *str_op(String *) = 0;
  virtual bool date_op(MYSQL_TIME *ltime, my_time_flags_t fuzzydate) = 0;
  virtual bool time_op(MYSQL_TIME *ltime) = 0;
};
```

设计是"**子类只写纯运算，类型转换由基类统一承担**"。四个 `val_*` 都是同一个 `switch` 骨架：

```cpp
// [F3]
longlong Item_func_numhybrid::val_int() {
  assert(fixed == 1);
  switch (hybrid_type) {
    case DECIMAL_RESULT: { my_decimal decimal_value, *val;
      if (!(val = decimal_op(&decimal_value))) return 0;   // null is set
      longlong result; my_decimal2int(E_DEC_FATAL_ERROR, val, unsigned_flag, &result);
      return result; }
    case INT_RESULT:   return int_op();
    case REAL_RESULT:  return llrint_with_overflow_check(real_op());
    case STRING_RESULT: {                                   // 时态再细分
      switch (data_type()) {
        case MYSQL_TYPE_DATE:      return val_int_from_date();
        case MYSQL_TYPE_DATETIME:
        case MYSQL_TYPE_TIMESTAMP: return val_int_from_datetime();
        case MYSQL_TYPE_TIME:      return val_int_from_time();
        default: break;
      }
      String *res = str_op(&str_value);
      if (!res) return 0;
      return (*(res->charset()->cset->strtoll10))(res->charset(), res->ptr(),
                                                  &(res->ptr()[res->length()]), nullptr);
    }
    default: assert(0);
  }
  return 0;
}
```

★ 每个 `val_*` 都是「**`hybrid_type` 分支 = 我的原生运算**，其余分支 = 从原生结果转换」。所以 `a+b` 在两个 INT 下只跑 `int_op()`，**不会**走 `real_op()` 再转回来——**这是 `Item_func_numhybrid` 存在的全部理由：用一次 switch 换取"精度和性能都不退化"**。返回 `0`/`0.0`/`nullptr` 时注释统一写 `// null is set`——**返回值被忽略，真正的结果看 `null_value`**。

#### 9.8.7 `const_item()`、常量折叠与三个伪表位

```cpp
// [F1] —— const_item() 没有 Item_func 版本，完全由 used_tables() 推出
  bool const_item() const { return (used_tables() == 0); }
  bool const_for_execution() const { return !(used_tables() & ~INNER_TABLE_BIT); }
```

```cpp
// [F5] sql/sql_const.h —— 三个伪表位占据 table_map 的最高 3 位
constexpr const table_map INNER_TABLE_BIT     {1ULL << (MAX_TABLES + 0)};  // 参数/访问表的子查询/函数
constexpr const table_map OUTER_REF_TABLE_BIT {1ULL << (MAX_TABLES + 1)};  // 外层引用
constexpr const table_map RAND_TABLE_BIT      {1ULL << (MAX_TABLES + 2)};  // 非确定性
```

它们各自阻断一种"提前求值"：`INNER_TABLE_BIT` → 不能跨查询复用（`const_for_execution()` 只容忍它）；`OUTER_REF_TABLE_BIT` → 每次外层行变化都要重算；`RAND_TABLE_BIT` → **连一次执行内都不能缓存**。

`get_initial_pseudo_tables()` 的真实实现（`[F2]`）：`Item_func_connection_id` 返回 `INNER_TABLE_BIT`；**`Item_func_rand` 返回 `RAND_TABLE_BIT`**。

⇒ 所以 `1+2` 是 const，**`RAND()+1` 不是**——差异全部来自 `get_initial_pseudo_tables()` 与参数伪表位的 OR 累积。

**常量折叠的唯一官方入口**：

```cpp
// [F3]
bool eval_const_cond(THD *thd, Item *cond, bool *value) {
  // 该函数在 resolving 与 optimization 两阶段都可能被调用
  assert(cond->may_evaluate_const(thd));
  *value = cond->val_bool();
  return thd->is_error();
}
```

`assert(may_evaluate_const(thd))` 保证两阶段语义不同：resolve 期只允许 `const_item()` 的项被折叠，优化期（表已锁）才允许 `const_for_execution()` 的项。

#### 9.8.8 完整例子：`Item_func_plus`（`+`）的全貌

继承链：`Item_func_plus → Item_func_additive_op → Item_num_op → Item_func_numhybrid → Item_func`

```cpp
// [F2]
class Item_func_plus final : public Item_func_additive_op {
 public:
  Item_func_plus(const POS &pos, Item *a, Item *b) : Item_func_additive_op(pos, a, b) {}
  const char *func_name() const override { return "+"; }
  // SUPPRESS_UBSAN: signed integer overflow（下面刻意用无符号回绕做检测）
  longlong int_op() override SUPPRESS_UBSAN;
  double real_op() override;
  my_decimal *decimal_op(my_decimal *) override;
  enum Functype functype() const override { return PLUS_FUNC; }
};
```

⇒ `Item_func_plus` 自己**只写了三个 `*_op()` 加 `func_name()`/`functype()`**，`val_*`/`print`/`resolve_type` 全由继承链免费提供——这就是 9.8.6 分派设计的收益。

**类型推导（`Item_num_op::set_numeric_type()`）**——这就是 SQL 的**数值提升规则**（REAL > DECIMAL > INT）：

```cpp
// [F3]
void Item_num_op::set_numeric_type(void) {
  assert(arg_count == 2);
  Item_result r0 = args[0]->numeric_context_result_type();   // ★ 不是 result_type()
  Item_result r1 = args[1]->numeric_context_result_type();
  assert(r0 != STRING_RESULT && r1 != STRING_RESULT);

  if (r0 == REAL_RESULT || r1 == REAL_RESULT) {
    assert(!args[0]->is_temporal() || !args[1]->is_temporal());
    set_data_type(MYSQL_TYPE_DOUBLE);  hybrid_type = REAL_RESULT;
    aggregate_float_properties(args, arg_count);
    max_length = float_length(decimals);
  } else if (r0 == DECIMAL_RESULT || r1 == DECIMAL_RESULT) {
    set_data_type(MYSQL_TYPE_NEWDECIMAL);  hybrid_type = DECIMAL_RESULT;
    result_precision();
  } else {
    assert(r0 == INT_RESULT && r1 == INT_RESULT);
    set_data_type(MYSQL_TYPE_LONGLONG);  decimals = 0;  hybrid_type = INT_RESULT;
    result_precision();
  }
}
```

用 `numeric_context_result_type()` 而非 `result_type()`，因为**时态类型在数值上下文里应被视作 INT/DECIMAL 而不是字符串**。

`+`/`-` 的精度规则（`[F3]`）：整数位取两者较大值 **+1**（进位位），小数位取较大值。★ 符号规则很微妙：**整数加法"或"**（有任一无符号则结果按无符号解释），**DECIMAL 加法"与"**（只有两者都无符号才无符号）——因为 DECIMAL 有更大值域可直接表示负数，而 BIGINT 需要靠 `unsigned_flag` 解释位模式。

**三个求值实现**（`[F3]`）——展示两种 NULL 处理模式：

```cpp
double Item_func_plus::real_op() {
  double val1 = args[0]->val_real();
  if (current_thd->is_error()) return error_real();
  double val2 = args[1]->val_real();                      // ★ 两个参数都要求值
  if (current_thd->is_error()) return error_real();
  if ((null_value = args[0]->null_value || args[1]->null_value)) return 0.0;
  return check_float_overflow(val1 + val2);               // ±inf → ER_DATA_OUT_OF_RANGE
}

// —— int_op()：先用 ulonglong 做"可能回绕"的加法，再按 unsigned_flag 四象限判溢出
longlong Item_func_plus::int_op() {
  longlong val0 = args[0]->val_int(); ... longlong val1 = args[1]->val_int();
  longlong res = static_cast<unsigned long long>(val0) + static_cast<unsigned long long>(val1);
  bool res_unsigned = false;
  if ((null_value = args[0]->null_value || args[1]->null_value)) return 0;
  if (args[0]->unsigned_flag) { ... } else { ... }        // 四象限推导 res_unsigned
  return check_integer_overflow(res, res_unsigned);       // 按"本 Item 的" unsigned_flag 复查
err:
  return raise_integer_overflow();
}

// —— decimal_op()：★ 逐参数短路（DECIMAL 开销大，没必要为已知 NULL 做第二次求值）
my_decimal *Item_func_plus::decimal_op(my_decimal *decimal_value) {
  val1 = args[0]->val_decimal(&value1);
  if ((null_value = args[0]->null_value)) return nullptr;
  val2 = args[1]->val_decimal(&value2);
  if ((null_value = args[1]->null_value)) return nullptr;
  if (check_decimal_overflow(my_decimal_add(E_DEC_FATAL_ERROR & ~E_DEC_OVERFLOW,
                                            decimal_value, val1, val2)) > 3)
    return error_decimal(decimal_value);
  return decimal_value;
}
```

#### 9.8.9 ★ NULL 语义：`Item_func` 上的三种处理模式

`Item_func` **没有**自动的"任一参数 NULL 则结果 NULL"钩子——`null_on_null` 只影响优化器侧的 `not_null_tables_cache`。真正的 NULL 传播**由每个 `val_*` 自己写**，但写法高度统一：

| 模式 | 写法 | 例子 |
|---|---|---|
| **A 先全求值再统一判** | `v0=args[0]->val_*(); v1=args[1]->val_*(); if ((null_value = args[0]->null_value \|\| args[1]->null_value)) return 0;` | `+ - * / %` 的 int/real 分支 |
| **B 逐参数短路** | `val=args[0]->val_decimal(); if ((null_value=args[0]->null_value)) return nullptr;` | `decimal_op()` 系列 |
| **C 取首个非 NULL**（`null_on_null=false`） | `null_value=false; for(i...){r=args[i]->val_*(); if(!args[i]->null_value) return r;} null_value=true;` | `COALESCE/IFNULL/IF/CASE/ISNULL` |

模式 C 的完整实现（`[F6]`）：

```cpp
class Item_func_coalesce : public Item_func_numhybrid {
  Item_func_coalesce(const POS &pos, Item *a, Item *b)
      : Item_func_numhybrid(pos, a, b) { null_on_null = false; }   // ★ 声明
  void set_numeric_type() override {}     // 类型来自参数聚合，不做数值提升
};

String *Item_func_coalesce::str_op(String *str) {
  assert(fixed == 1);
  null_value = false;                     // ★ 先置"非 NULL"
  for (uint i = 0; i < arg_count; i++) {
    String *res = args[i]->val_str(str);
    if (current_thd->is_error()) return error_str();
    if (res != nullptr) return res;       // 第一个非 NULL 参数胜出
  }
  null_value = true;                      // 全 NULL 才 NULL
  return nullptr;
}
```

⇒ 这正是 `null_on_null=false` 在运行期的兑现：因为 COALESCE 不保证"任一 NULL ⇒ 结果 NULL"，它在 resolve 阶段也**不会**把参数的 `not_null_tables()` 累积进自己的 `not_null_tables_cache`（`fix_func_arg` 里的 `if (null_on_null)`），优化器不能据此做 "NOT NULL" 推导。

**统一契约**：返回值只在 `null_value == false` 时有意义；字符串返回 `nullptr` 与 `null_value == true` 必须成对出现。

#### 9.8.10 小结：写一个新 `Item_func` 的最小清单

1. **选中间层**：结果固定整数 → `Item_int_func`；固定浮点 → `Item_real_func`；固定字符串 → `Item_str_func`；结果类型随参数变 → `Item_func_numhybrid`。
2. **只实现必要的 `val_*`**（或 `*_op()`），中间层替你补齐其余三个。
3. **实现 `resolve_type()` / `resolve_type_inner()`**：工具箱 = `set_data_type_from_item()`、`Item::aggregate_type()`、`agg_arg_charsets*()`、`result_precision()`。
4. **写 NULL 处理**：按 9.8.9 三选一；若语义不是"任一 NULL ⇒ NULL"，构造里 `null_on_null = false`。
5. **若非确定性/依赖会话/依赖外层引用**：重写 `get_initial_pseudo_tables()` 返回相应伪表位，`const_item()` 自动变 false。
6. **实现 `func_name()` 与 `functype()`**：前者供 `print()`/报错，后者供 `eq()` 与优化器特化。

### 9.9 ★ 函数族谱：各大族代表实现 + `Item_typecast_*` + `Item_cache_*`

> 9.8 节讲**骨架**（`args` / `resolve_type` / `val_*` 分派），本节讲**各族的具体代表实现与分类谱系**，补齐此前 0 覆盖的 `Item_typecast_*`（18 类）与 `Item_cache_*` 剩余类。
>
> 补充源码坐标：`[F7]` sql/item_strfunc.h/.cc、`[F8]` sql/item_cmpfunc.h/.cc、`[F9]` sql/item_timefunc.h/.cc、`[F10]` sql/item_geofunc.h、`[F11]` sql/item_json_func.h、`[F12]` sql/item_create.cc。

#### 9.9.1 算术族：溢出检测技巧

`Item_func_mul::int_op()` 用**"拆 32 位半字 + 取绝对值 + 事后定符号"**检测溢出（`a = a1·2³² + a0`）：

```cpp
// [F3] sql/item_func.cc
longlong Item_func_mul::int_op() {
  longlong a = args[0]->val_int(); ... longlong b = args[1]->val_int();
  if ((null_value = args[0]->null_value || args[1]->null_value)) return 0;
  if (a == 0 || b == 0) return 0;                       // 快速路径

  const bool a_negative = (!args[0]->unsigned_flag && a < 0);
  const bool b_negative = (!args[1]->unsigned_flag && b < 0);
  const bool res_unsigned = (a_negative == b_negative); // 同号 → 结果为正

  if (a_negative && a == INT_MIN64) { ... }              // |INT_MIN| 超 LLONG_MAX，特判
  if (a_negative) a = -a;  if (b_negative) b = -b;

  ulong a0 = 0xFFFFFFFFUL & a, a1 = ((ulonglong)a) >> 32;
  ulong b0 = 0xFFFFFFFFUL & b, b1 = ((ulonglong)b) >> 32;

  if (a1 && b1) goto err;                               // 条件①：高半字都非 0
  res1 = (ulonglong)a1 * b0 + (ulonglong)a0 * b1;
  if (res1 > 0xFFFFFFFFUL) goto err;                    // 条件②
  res1 <<= 32;  res0 = (ulonglong)a0 * b0;
  if (test_if_sum_overflows_ull(res1, res0)) goto err;  // 条件③
  res = res1 + res0;
  if (a_negative != b_negative) { if ((ulonglong)res > (ulonglong)LLONG_MAX) goto err; res = -res; }
  return check_integer_overflow(res, res_unsigned);
err:
  return raise_integer_overflow();
}
```

`Item_func_abs::int_op()` 的边界：`-LLONG_MIN = LLONG_MAX+1` 超范围 → `raise_integer_overflow()`；`unsigned_flag` 恒跟随入参（结果必然非负）。
`Item_func_round` 一个类同时实现 `ROUND()`/`TRUNCATE()`（`truncate` 开关），且**常量精度已知时会收紧结果精度**（`ROUND(99.999,2)`：参数 `DECIMAL(5,3)` → 结果 `DECIMAL(5,2)`，预留 1 位进位）。

**算术族全清单**

| 类名 | 基类 | 运算 | 要点 |
|---|---|---|---|
| `Item_func_numhybrid` | `Item_func` | — | 抽象，`hybrid_type` + 6 个纯虚 `*_op()` |
| `Item_func_num1` | `Item_func_numhybrid` | — | 结果类型只看**第一个**参数（`ABS/NEG/ROUND/CEIL/FLOOR`） |
| `Item_num_op` | `Item_func_numhybrid` | — | `result_precision()` 纯虚；`print` 用 `print_op`（中缀） |
| `Item_func_additive_op` | `Item_num_op` | `+` `-` | 共享精度规则（整数位取 max + 1 进位位） |
| `Item_func_plus` / `_minus` | `Item_func_additive_op` | `+` / `-` | minus 额外处理 `NO_UNSIGNED_SUBTRACTION` |
| `Item_func_mul` | `Item_num_op` | `*` | 半字拆分溢出检测；小数位**相加** |
| `Item_func_div` / `_div_int` | `Item_func_div_base` | `/` / `DIV` | 除零 → `signal_divide_by_null()` |
| `Item_func_mod` | `Item_num_op` | `%` | 除零返回 NULL **不报错** |
| `Item_dec_func` | `Item_real_func` | `EXP/LN/SQRT/SIN/...` | 一律 REAL 域 |

#### 9.9.2 比较族：`Item_func_eq` 只有两行

继承链：`Item_func_eq → Item_eq_base → Item_func_comparison → Item_bool_func2 → Item_bool_func`

```cpp
// [F8] sql/item_cmpfunc.h
class Item_bool_func2 : public Item_bool_func {
 protected:
  Arg_comparator cmp;                    // ★ 比较器，绑定到 args / args+1
  bool abort_on_null{false};             // IS TRUE 语义：UNKNOWN 当 FALSE
  Item_bool_func2(Item *a, Item *b) : Item_bool_func(a, b), cmp(args, args + 1) {}
 public:
  virtual bool set_cmp_func() { return cmp.set_cmp_func(this, args, args + 1, is_nullable()); }
  bool is_null() override { return args[0]->is_null() || args[1]->is_null(); }
  void apply_is_true() override { abort_on_null = true; }
  bool ignore_unknown() const { return abort_on_null; }
  void cleanup() override { Item_bool_func::cleanup(); cmp.cleanup(); }  // ★ 一并清理 Item_cache
};
```

```cpp
// [F8] sql/item_cmpfunc.cc
longlong Item_func_eq::val_int() {
  assert(fixed == 1);
  int value = cmp.compare();             // 三值逻辑由 null_value 携带
  return value == 0 ? 1 : 0;
}
longlong Item_func_ne::val_int() {
  int value = cmp.compare();
  return value != 0 && !null_value ? 1 : 0;              // NULL <> x → NULL（不是 TRUE）
}
longlong Item_func_equal::val_int() {                    // <=>：NULL 安全
  int value = cmp.compare();
  if (!null_value) return value == 0 ? 1 : 0;
  null_value = false;
  return longlong(cmp.compare_null_values());            // 退化成"两边是否都 NULL"
}
```

⇒ 实际比较语义全部委托给 `Arg_comparator`（见**第七章**）。`Item_func_equal`（`<=>`）在 `resolve_type()` 里 `set_nullable(false)`——它**永不返回 UNKNOWN**。

| 类名 | 运算符 | `val_int()` 要点 |
|---|---|---|
| `Item_func_eq` | `=` | `cmp.compare()==0` |
| `Item_func_equal` | `<=>` | NULL 安全；`set_cmp_func(..., true)` 强制 nullable |
| `Item_func_ne` | `<>` `!=` | `!=0 && !null_value` |
| `Item_func_lt/le/gt/ge` | `<` `<=` `>` `>=` | 各自阈值比较 |
| `Item_func_not` | `NOT` | 单参，`truth_transformer()` 下推 |
| `Item_func_isnull` | `IS NULL` | `update_used_tables()` 直接用 `args[0]` |
| `Item_func_like` | `LIKE` | 三参形式带 `ESCAPE` |
| `Item_func_between` | `[NOT] BETWEEN` | 展开成两个比较 |

#### 9.9.3 逻辑族：`Item_cond` 的短路求值

**★ 名字纠正**：成员**不是** `list_of`，而是 **`List<Item> list`**；配套开关是 `abort_on_null`（`ignore_unknown()`）。

```cpp
// [F8] sql/item_cmpfunc.cc
longlong Item_cond_and::val_int() {
  assert(fixed == 1);
  List_iterator_fast<Item> li(list);
  Item *item;
  null_value = false;
  while ((item = li++)) {
    if (!item->val_bool()) {                  // 有一个为假 → AND 必为假
      if (ignore_unknown() || !(null_value = item->null_value))
        return 0;
      // 否则：本项是 UNKNOWN 且要区分三值 → 记下 null_value，继续看后面有没有 FALSE
    }
    if (current_thd->is_error()) return error_int();
  }
  return null_value ? 0 : 1;                  // 全 TRUE → 1；含 UNKNOWN → NULL(0)
}

longlong Item_cond_or::val_int() {
  List_iterator_fast<Item> li(list);
  Item *item;
  null_value = false;
  while ((item = li++)) {
    if (item->val_bool()) { null_value = false; return 1; }   // ★ 有一个为真即短路
    if (item->null_value) null_value = true;                   // 记录见过 UNKNOWN
    if (current_thd->is_error()) return error_int();
  }
  return 0;
}
```

⇒ 一旦能确定结果立刻 return，**其余分支不再求值**。`Item_cond_and` 多一个 `COND_EQUAL cond_equal` 成员（本层的多个等值 `Item_equal` 列表）。常量折叠走 `Item_cond::remove_const_conds()` → `eval_const_cond()`。

#### 9.9.4 字符串族："成员 `String tmp_value`"模式

`Item_func_concat` 用**成员** `String tmp_value` 作结果缓冲（不是调用方传入的 `str`），因为多参数拼接必须跨参数累积：

```cpp
// [F7] sql/item_strfunc.h
class Item_func_concat : public Item_str_func {
  String tmp_value{"", 0, collation.collation};
 public:
  String *val_str(String *) override;
  bool resolve_type(THD *thd) override;
};
class Item_func_concat_ws : public Item_str_func {
  String tmp_value{"", 0, collation.collation};
 public:
  explicit Item_func_concat_ws(mem_root_deque<Item *> *list) : Item_str_func(list) {
    null_on_null = false;             // ★ CONCAT_WS 遇 NULL 跳过而非整体 NULL
  }
};
```

```cpp
// [F7] sql/item_strfunc.cc
String *Item_func_concat::val_str(String *str) {
  null_value = false;
  tmp_value.length(0);                      // ★ 复用成员缓冲，只重置长度不释放内存
  for (uint i = 0; i < arg_count; ++i) {
    String *res = eval_string_arg(collation.collation, args[i], str);
    if (res == nullptr) return error_str();  // CONCAT 遇 NULL 整体为 NULL
    if (res->length() + tmp_value.length() > thd->variables.max_allowed_packet)
      return push_packet_overflow_warning(thd, func_name());   // ★ 长度闸门
    if (tmp_value.append(*res)) return error_str();
  }
  tmp_value.set_charset(collation.collation);
  return &tmp_value;                        // ★ 返回成员，不是入参 str
}

/* CONCAT_WS：分隔符合插在"非空参数之间" */
String *Item_func_concat_ws::val_str(String *str) {
  String *sep_str = eval_string_arg(collation.collation, args[0], &tmp_sep_str);
  if (sep_str == nullptr) return error_str();         // 分隔符为 NULL → 整体 NULL
  uint non_null_args = 0;
  for (uint i = 1; i < arg_count; i++) {
    String *res = eval_string_arg(collation.collation, args[i], str);
    if (res == nullptr) { if (thd->is_error()) return error_str(); continue; }  // ★ Skip NULL
    if (non_null_args++ > 0) { if (tmp_value.append(*sep_str)) return error_str(); }
    if (tmp_value.append(*res)) return error_str();
  }
  return &tmp_value;
}
```

★ 这个"成员 `String tmp_value`"模式在字符串族中非常普遍（`Item_func_reverse`、`Item_func_replace`、`Item_load_file`、`Item_func_case`…）。好处是**跨调用复用已分配缓冲**，代价是**不可重入**——同一 Item 在嵌套求值里不能同时使用。

#### 9.9.5 控制流族：`IF()` / `CASE` = "先选分支，再转发"

两者都不用 `hybrid_type`，而是**"先选分支，再对该分支调 `val_*`"**——因此**只求值被选中的分支**（所以 `SELECT IF(FALSE, 1/0, 1)` 不报错）。

```cpp
// [F8] sql/item_cmpfunc.cc
bool Item_func_if::fix_fields(THD *thd, Item **ref) {
  args[0]->apply_is_true();          // ★ 条件参数按 IS TRUE 语义：UNKNOWN 视为 FALSE
  if (Item_func::fix_fields(thd, ref)) return true;
  update_not_null_tables();
  return false;
}

bool Item_func_if::resolve_type_inner(THD *thd) {
  args++;  arg_count--;              // ★ 技巧：临时把 args 指向"两个返回分支"
  if (param_type_uses_non_param(thd)) return true;
  args--;  arg_count++;
  set_nullable(args[1]->is_nullable() || args[2]->is_nullable());
  aggregate_type(make_array(args + 1, 2));               // 结果类型 = 两分支聚合
  cached_result_type = Field::result_merge_type(data_type());
  ...
}

longlong Item_func_if::val_int() {
  Item *arg = args[0]->val_bool() ? args[1] : args[2];   // ★ 二选一
  longlong value = arg->val_int();
  null_value = arg->null_value;
  return value;
}
```

`Item_func_case` 把 CASE 表达式**追加到 `args` 数组尾部**，用下标 `first_expr_num` / `else_expr_num` / `ncases` 描述语法结构；`find_item()` 线性匹配（带 `value_added_map` 惰性求值——CASE 表达式按比较类型每种只求值一次），返回值再调 `val_*`。

#### 9.9.6 时间族：★ `NOW()` 是 `INNER_TABLE_BIT`，不是 `RAND_TABLE_BIT`

**关键纠正**：`Item_func_now` / `Item_func_curdate` 的 `get_initial_pseudo_tables()` 返回 **`INNER_TABLE_BIT`**。返回 `RAND_TABLE_BIT` 的是 `RAND()`、非确定性 SP、`Item_func_random_bytes`。

语义区别：

- `INNER_TABLE_BIT` → **阻止在 prepare 阶段求值**（需要上下文就绪），但**仍允许缓存** ⇒ 语句内所有 `NOW()` 返回同一值（**语句级一致性**）
- `RAND_TABLE_BIT` → 阻止缓存，每次调用都可能不同

```cpp
// [F9] sql/item_timefunc.h
class Item_func_now : public Item_datetime_func {
 protected:
  virtual Time_zone *time_zone() = 0;        // local / UTC 由子类给出
 public:
  /// This function must assign a new value for each execution
  table_map get_initial_pseudo_tables() const override {
    return INNER_TABLE_BIT;                 // ★ 不是 RAND_TABLE_BIT
  }
  longlong val_date_temporal() override;
  bool get_date(MYSQL_TIME *res, my_time_flags_t) override;
};
class Item_func_now_local : public Item_func_now { /* NOW() */ };
class Item_func_now_utc final : public Item_func_now { /* UTC_TIMESTAMP() */ };
```

**语句级一致性**由 `thd->query_start_timeval_trunc(decimals)` 保证——取**语句开始时刻**并按 `decimals` 截断，而不是每次读系统时钟：

```cpp
// [F9] sql/item_timefunc.cc
bool Item_func_now::get_date(MYSQL_TIME *res, my_time_flags_t) {
  MYSQL_TIME_cache tm;
  tm.set_datetime(current_thd->query_start_timeval_trunc(decimals), decimals, time_zone());
  return tm.get_time(res);                  // ★ 语句内恒定
}
longlong Item_func_now::val_date_temporal() {
  MYSQL_TIME_cache tm;
  tm.set_datetime(current_thd->query_start_timeval_trunc(decimals), decimals, time_zone());
  return tm.val_packed();
}
```

⇒ 因为 `used_tables_cache != 0`，`const_item()` 为 false ⇒ 优化期不会把 `NOW()` 提前算成常量；又因为只有 `INNER_TABLE_BIT`，同一语句内多次调用命中缓存得到同一时刻。
（对照：`SYSDATE()` 故意走 `my_micro_time()` 取真实时钟，**不是**语句级一致；`--sysdate-is-now` 可改变此行为。）

#### 9.9.7 ★ `Item_typecast_*` 全族（18 类）

**★ 三个反直觉事实**：

1. **8.0.39 不存在统一的 `Item_typecast` 基类**——18 个类"按结果类型各自就近继承"（`Item_int_func` / `Item_func` / `Item_charset_conversion` / `Item_date_func` / `Item_time_func` / `Item_datetime_func` / `Item_json_func` / `Item_geometry_func`）。
2. **不存在 `Item_typecast_int` / `Item_typecast_binary`**——`CAST(x AS SIGNED/UNSIGNED)` 是 `Item_typecast_signed/unsigned`；`CAST(x AS BINARY)` **复用 `Item_typecast_char`**，只是 `m_cast_cs = &my_charset_bin`。
3. "长度/精度/字符集"成员**分散在各基类**，名字各不相同（`m_cast_length` / `m_cast_cs` / `m_from_cs` / `m_charset_conversion` 在 `Item_charset_conversion`；`m_explicit_cast` 在 datetime/date/time）。

**公共契约**（仅此四条）：单参数；`functype()` 返回 `TYPECAST_FUNC`；`resolve_type()` 里对 `args[0]` 调 `propagate_type(thd, <目标类型>, false, true)` 把目标类型**反向传播**给参数（对 `Item_param` 尤其关键：`CAST(? AS DATE)` 会让 `?` 也变成 DATE）；`print()` 一律还原成 `CAST(expr AS <type>)`。

**代表一：`Item_typecast_char`**（字符串域，最复杂）——字符集转换 + 截断 + `BINARY(N)` 右侧补零：

```cpp
// [F7] sql/item_strfunc.h
class Item_charset_conversion : public Item_str_func {
 protected:
  bool m_charset_conversion{false};         // 是否真的要做字符集转换
  const CHARSET_INFO *m_cast_cs;            // 目标字符集
  const CHARSET_INFO *m_from_cs{nullptr};   // 源字符集
  String m_tmp_value;                       // ★ 转换结果的私有缓冲
  longlong m_cast_length{-1};               // -1 = 不截断
 public:
  bool m_safe;                              // 该转换是否无损
};
class Item_typecast_char final : public Item_charset_conversion {
  enum Functype functype() const override { return TYPECAST_FUNC; }
  const char *func_name() const override { return "cast_as_char"; }
};
```

```cpp
// [F7] sql/item_strfunc.cc
bool Item_charset_conversion::resolve_type(THD *thd) {
  if (args[0]->data_type() == MYSQL_TYPE_INVALID)          // '?' 占位符
    if (args[0]->propagate_type(thd, Type_properties(MYSQL_TYPE_VARCHAR, m_cast_cs)))
      return true;
  collation.set(m_cast_cs, DERIVATION_IMPLICIT);
  /* ★ ASCII 兼容 + repertoire 为 ASCII → 直接把 from_cs 设为 cast_cs，省一次转换 */
  m_from_cs = (args[0]->collation.repertoire == MY_REPERTOIRE_ASCII &&
               my_charset_is_ascii_based(m_cast_cs) &&
               my_charset_is_ascii_based(args[0]->collation.collation))
                  ? m_cast_cs : args[0]->collation.collation;
  set_data_type_string((uint32)(m_cast_length >= 0 ? m_cast_length : compute_max_char_length()));
  /* ★ 只要目标是多字节字符集就强制转换（保证 well-formed）；单字节允许直接拷贝 */
  m_charset_conversion = (m_cast_cs->mbmaxlen > 1) ||
      (!my_charset_same(m_from_cs, m_cast_cs) && m_from_cs != &my_charset_bin &&
       m_cast_cs != &my_charset_bin);
  return false;
}

String *Item_charset_conversion::val_str(String *str) {
  String *res = args[0]->val_str(str);
  if (res == nullptr) { null_value = true; return nullptr; }
  if (m_charset_conversion || res->alloced_length() == 0) {
    uint dummy_err;
    if (m_tmp_value.copy(res->ptr(), res->length(), m_from_cs, m_cast_cs, &dummy_err))
      return error_str();
    res = check_well_formed_result(&m_tmp_value, false, true);   // 发 warning + 截断
    if (res == nullptr) return error_str();
  }
  res->set_charset(m_cast_cs);
  if (m_cast_length >= 0) {
    uint32 length;
    if (res->length() > (length = (uint32)res->charpos(m_cast_length))) {
      push_warning_printf(thd, Sql_condition::SL_WARNING, ER_TRUNCATED_WRONG_VALUE, ...);
      res->length(length);                                        // ★ 截断到 N 字符
    } else if (m_cast_cs == &my_charset_bin && res->length() < m_cast_length) {
      memset(res->ptr() + res->length(), 0, m_cast_length - res->length());  // ★ BINARY(N) 补 0x00
      res->length(m_cast_length);
    }
  }
  return res;
}
```

**代表二：`Item_typecast_signed`**——按 `cast_to_int_type()` 分派（字符串走带范围检查的解析器，非字符串/时态直接用 `val_int()` 语义）：

```cpp
// [F3] sql/item_func.cc
longlong Item_typecast_signed::val_int() {
  if (args[0]->cast_to_int_type() != STRING_RESULT || args[0]->is_temporal())
    { value = args[0]->val_int(); null_value = args[0]->null_value; }
  else
    value = val_int_from_str(args[0], unsigned_flag, &null_value);   // '1abc' 会 warning
  return value;
}
```

**代表三：`Item_typecast_datetime`——显式 vs 隐式的分水岭**

```cpp
// [F9] sql/item_timefunc.h
class Item_typecast_datetime final : public Item_datetime_func {
  bool detect_precision_from_arg;
  bool m_explicit_cast{true};          // true=用户写的 CAST；false=优化器注入
 public:
  Item_typecast_datetime(Item *a, bool explicit_cast)
      : Item_datetime_func(a), m_explicit_cast(explicit_cast) { detect_precision_from_arg = true; }
};
```

```cpp
// [F9] sql/item_timefunc.cc
bool Item_typecast_datetime::get_date(MYSQL_TIME *ltime, my_time_flags_t fuzzy_date) {
  ... flags |= sql_mode 相关的 NO_ZERO_DATE / NO_ZERO_IN_DATE / INVALID_DATES / FRAC_TRUNCATE ...
  if (get_arg0_date(ltime, flags)) {
    ltime->time_type = MYSQL_TIMESTAMP_DATETIME;
    if (args[0]->null_value || m_explicit_cast) return true;   // ★ 显式 CAST → NULL
    /* The implicit CAST to DATETIME returns 0-date on invalid argument */
    null_value = false;
    set_zero_time(ltime, ltime->time_type);                    // ★ 隐式 → '0000-00-00 00:00:00'
    return false;
  }
  ...
}
```

**显式 CAST 与隐式 CAST 的分工**（隐式由 `Arg_comparator::inject_cast_nodes()` 注入，见**7.7**）：

| | 显式 CAST | 隐式 CAST（WL#12108 注入） |
|---|---|---|
| 触发 | 用户 SQL 的 `CAST(x AS T)` / `CONVERT()` | `set_cmp_func()` 期间自动插入 |
| 目的 | 用户要的类型 | 让"字符串列 vs 数值常量"比较**能用上索引**（保持列在左侧） |
| 构造入口 | `[F12] create_func_cast()` | `wrap_in_cast()`（`[F8]`，传 `explicit_cast=false`） |
| 失败语义 | 返回 NULL | 返回零值而非 NULL |
| 覆盖类型 | 全部 18 个 | 仅 `datetime` / `date` / `time` / `real` |

**`Item_typecast_*` 全族清单（18 个）**

| # | 类名 | SQL 语法 | 基类 | 结果类型 |
|---|---|---|---|---|
| 1 | `Item_typecast_signed` | `CAST(x AS SIGNED)` | `Item_int_func` | LONGLONG（有符号） |
| 2 | `Item_typecast_unsigned` | `CAST(x AS UNSIGNED)` | `Item_int_func` | LONGLONG（无符号） |
| 3 | `Item_typecast_decimal` | `CAST(x AS DECIMAL(M,D))` | `Item_func` | NEWDECIMAL |
| 4 | `Item_typecast_real` | `CAST(x AS DOUBLE/FLOAT)` | `Item_func` | DOUBLE / FLOAT |
| 5 | `Item_typecast_char` | `CAST(x AS CHAR(N))` / `NCHAR` / **`BINARY(N)`** | `Item_charset_conversion` | VARCHAR / BINARY |
| 6 | `Item_typecast_date` | `CAST(x AS DATE)`，`DATE(x)` | `Item_date_func` | DATE（清时间部分） |
| 7 | `Item_typecast_time` | `CAST(x AS TIME(N))`，`TIME(x)` | `Item_time_func` | TIME（保留 day 部分） |
| 8 | `Item_typecast_datetime` | `CAST(x AS DATETIME(N))` | `Item_datetime_func` | DATETIME |
| 9 | `Item_typecast_year` | `CAST(x AS YEAR)` | `Item_int_func` | YEAR |
| 10 | `Item_typecast_json` | `CAST(x AS JSON)` | `Item_json_func` | JSON（见 `server/datatype/json.md`） |
| 11–18 | `Item_typecast_geometry` + 7 个子类型 | `CAST(x AS POINT/LINESTRING/POLYGON/MULTI*/GEOMETRYCOLLECTION)` | `Item_geometry_func` | 空间类型 |

#### 9.9.8 ★ `Item_cache_*` 全族（9 个）——惰性求值

```cpp
// [F1] sql/item.h
class Item_cache : public Item_basic_constant {
 protected:
  Item *example{nullptr};          // ★ 被缓存的表达式（"样本"）
  Item_field *cached_field{nullptr};
  bool value_cached{false};        // ★ 惰性求值开关
 public:
  Item_cache() { fixed = true; set_nullable(true); null_value = true; }  // ★ cache 自己永远 fixed

  virtual bool setup(Item *item) { example = item; max_length = item->max_length; ... }
  virtual void store(Item *item);          // 只登记样本，不做求值
  virtual bool cache_value() = 0;          // 子类：真正把 example 的值落到自己缓冲
  bool has_value();                        // ★ 惰性：首次访问才 cache_value()
  virtual void clear() { null_value = true; value_cached = false; }   // ★ 失效
  void cleanup() override { if (example != nullptr) clear(); Item::cleanup(); }
};
```

```cpp
// [F4] sql/item.cc
void Item_cache::store(Item *item) { example = item; ... value_cached = false; }  // ★ 不求值

bool Item_cache::has_value() {
  if (value_cached || cache_value())       // ★ 惰性求值发生在这里
    return !null_value;
  return false;
}

/* 工厂：按 result_type() 选类；★ 三个特例 */
Item_cache *Item_cache::get_cache(const Item *item, const Item_result type) {
  switch (type) {
    case INT_RESULT:
      if (item->data_type() == MYSQL_TYPE_BIT) return new Item_cache_bit(item->data_type());
      return new Item_cache_int(item->data_type());
    case REAL_RESULT:    return new Item_cache_real();
    case DECIMAL_RESULT: return new Item_cache_decimal();
    case STRING_RESULT:
      if (item->is_temporal()) return new Item_cache_datetime(item->data_type());
      if (item->data_type() == MYSQL_TYPE_JSON) return new Item_cache_json();
      return new Item_cache_str(item);
    case ROW_RESULT:     return new Item_cache_row();
  }
}
```

```cpp
// [F4] —— 代表：Item_cache_int
bool Item_cache_int::cache_value() {
  if (!example) return false;
  value_cached = true;                     // ★ 先置位，避免递归
  value = example->val_int();              // 真正求值
  null_value = example->null_value;
  unsigned_flag = example->unsigned_flag;  // unsigned 是运行期属性，每次刷新
  return true;
}
longlong Item_cache_int::val_int() {
  assert(fixed == 1);
  if (!has_value()) return 0;              // ★ 所有 val_* 都以 has_value() 开头
  return value;
}
```

★ **失效规则一句话**：`store()` / `clear()` / `cleanup()` 把 `value_cached` 置 false → 下次 `has_value()`（进而 `val_*`）重新 `cache_value()`；`store_null()` / `store_value()` 直接置 true 并写入值。

| # | 类名 | 存储成员 | `result_type()` | 要点 |
|---|---|---|---|---|
| 0 | `Item_cache`（抽象） | `example`/`cached_field`/`value_cached` | 由 `example` 推导 | 惰性；`cleanup()` 仅当有 `example` 时失效 |
| 1 | `Item_cache_int` | `longlong value` | INT | `val_time_temporal` 直接复用 `val_int()` |
| 2 | `Item_cache_bit` | 继承 `value` | INT | 覆写 `val_str()`：按 BIT(N) 字节序列输出 |
| 3 | `Item_cache_real` | `double value` | REAL | `val_int()` 用 `rint()` |
| 4 | `Item_cache_decimal` | `my_decimal` | DECIMAL | `val_decimal()` 直接返回成员地址 |
| 5 | `Item_cache_str` | `char buffer[...]` + `String value_buff` | STRING | ★ 必须**拷进私有缓冲**（example 可能是表字段，下次取行会覆盖） |
| 6 | `Item_cache_row` | `Item_cache **values` | ROW | `val_*` 全 `illegal_method_call()`；空子查询整行 `store_null()` |
| 7 | `Item_cache_datetime` | `String cached_string` + `longlong int_value` | STRING | ★ **双缓冲**：按 `cmp_context` 选字符串/整数路径，避免反复转换 |
| 8 | `Item_cache_json` | `Json_wrapper*` | STRING | `sort()` 只影响数组 |

#### 9.9.9 其余重要小族

| 类 | 要点 |
|---|---|
| **`Item_param`** | ★ **不是** `Item_func`，是零参数的 `Item`。值本体是 `union { longlong; double; MYSQL_TIME }` + `String str_value_ptr`。★ `data_type()` 由 `resolve_type()` 阶段被父节点 `propagate_type()` **反向传播**下来（`CAST(? AS DATE)` 让 `?` 也变 DATE）；`m_type_inherited` / `m_type_pinned` 分别标记"继承来的类型"与"钉死的类型"（如 `LIMIT ?` 必须是非负整数）。reprepare 时 `set_param_type_and_swap_value()` 用 **swap**（不是 copy）保留值，省分配与字符集转换。完整流程见 `12_prepared_statement.md` |
| **`Item_splocal`** | 存储程序局部变量，是**间接引用节点**：`m_var_idx` 是 `sp_rcontext` 变量数组下标，所有求值经 `this_item()` 重定向到运行时上下文。同时是 `Settable_routine_parameter`（可被 `SET`/`INTO` 赋值） |
| **`Item_func_get_system_var`** | `@@var` / `GLOBAL|SESSION var`。★ `get_initial_pseudo_tables()` 返回 **`INNER_TABLE_BIT`**（与 `NOW()` 同理）。`used_query_id + cache_present` 构成**语句级缓存**（同一 query_id 只读一次 sys_var）；`is_valid_for_pushdown()` 返回 false（含系统变量的表达式不可下推，因为 `print()` 打不出原始表达式） |
| **`Item_trigger_field`** | `NEW.x` / `OLD.x`。★ 它是 **`Item_field` 的子类**，求值路径完全复用 `Item_field::val_*`（直接读 Field 缓冲）。差别：① **绑定时机**——真正绑定由 `setup_field()` 在触发器表打开后完成，`fix_fields()` 只在 OLD/NEW 两个 Field 实例间二选一（`trigger_var_type`）；② **可写性**——`read_only`（只有 `BEFORE INSERT/UPDATE` 的 NEW 行可写）；③ `want_privilege` 在解析期定为 SELECT/UPDATE_ACL |
| 空间族 | `Item_geometry_func` 及其子类统一用 `val_geometry()` / `Geometry_buffer` 承载 WKB |
| JSON 族 | `Item_json_func` 的 `val_json(Json_wrapper*)` 通道 —— **整体剖析见 `server/datatype/json.md`** |

#### 9.9.10 ★ 本章纠正的常见误名

| 常见写法 | 8.0.39 真实情况 |
|---|---|
| `Item_func_add` | **不存在**；加法类是 `Item_func_plus` |
| `Item_func_numhybrid` 与 `Item_num_op` 并列 | 错：`Item_num_op : public Item_func_numhybrid` |
| `Item_typecast_int` / `_binary` | **不存在**；是 `Item_typecast_signed/unsigned`；`BINARY` 复用 `Item_typecast_char` |
| `Item_typecast` 统一基类 | **不存在**；18 个类各自就近继承 |
| `Item_cond::list_of` | 真实成员是 `List<Item> list`（另有 `abort_on_null`/`ignore_unknown()`） |
| `NOW()` 返回 `RAND_TABLE_BIT` | 错：返回 **`INNER_TABLE_BIT`**；`RAND()`/非确定性 SP/`random_bytes` 才是 `RAND_TABLE_BIT` |
| `Item_cache_datetime/time/date` | 只有 **`Item_cache_datetime`** 一个类，按 `data_type()` 区分，且同时缓存 String 与 longlong |

## 参考

- 源码：`class Item`（`sql/item.h` 的类注释是本篇最好的开篇引文）、`Item_func` / `Item_field` / `Item_ref` / `Item_sum` / `Item_subselect` / `Item_param`
- 源码：`Item_func::set_numeric_type()`、`Item::aggregate_type()`、`Item_cond::remove_const_conds()`、`Item_field::set_field()`
- MySQL 8.0 Reference Manual → *Type Conversion in Expression Evaluation*
- MySQL Internals Manual → *Item class*（社区维护，版本偏旧，注意与 8.0 的命名差异，见 1.4 纠错表）
- 相关篇：[`06_resolver_prepare.md`](06_resolver_prepare.md)（fix_fields 的调用现场）、[`07_optimize/logical/05_logical_predicate.md`](07_optimize/logical/05_logical_predicate.md)（等值传播）、[`12_prepared_statement.md`](12_prepared_statement.md)（`Item_param` 与 PS）
