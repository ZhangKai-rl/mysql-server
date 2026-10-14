# Parse Tree：语法形状 → 语义形状的一次性编译器

> **本篇归属**：`PT_*`（≈382 个类）+ `PTI_*`（36 个类）节点族的完整剖析。
>
> **与姊妹篇的分工（★ 读之前先看这个，避免找错地方）**：
>
> | 文档 | 讲什么 |
> |---|---|
> | [`04_parser.md`](04_parser.md) | **`sql_yacc.yy` 文法本身**——产生式写法、四个关键宏、Bison 构建期代码生成、LEX 结构、PT 内存管理 |
> | [`05_contextualize.md`](05_contextualize.md) | **contextualize 流程视角**——调用链全景、`Query_block` / `Query_expression` 的创建与填充、`Query_term` 树、PT 节点在流程中如何被消费 |
> | [`13_item_expression.md`](13_item_expression.md) | **Item 侧**——`PTI_*` 变身之后的世界：`fix_fields` / `val_*` / 类型推导 |
> | **本篇** | **PT 节点体系本身**——基类与模板、两套正交递归、17 个范畴的代表实现、与 yacc 的对应、复杂类深剖 |

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象：一条 SQL 怎么变成 Parse Tree](#先有个整体印象一条-sql-怎么变成-parse-tree)
- [一、基类体系：`Parse_tree_root` 与 `Parse_tree_node_tmpl<Context>`](#一基类体系parse_tree_root-与-parse_tree_node_tmplcontext)
- [二、两套正交递归：`contextualize()` 与 `itemize()`](#二两套正交递归contextualize-与-itemize)
- [三、`PT_*` 节点族：17 个范畴与代表实现](#三pt_-节点族17-个范畴与代表实现)
- [四、与 yacc 的对应](#四与-yacc-的对应)
- [五、复杂类深剖：`PT_query_specification`](#五复杂类深剖pt_query_specification)
- [六、命名规律与量化证据](#六命名规律与量化证据)
- [八、集合运算节点：`PT_union` / `PT_except` / `PT_intersect`](#八集合运算节点pt_union--pt_except--pt_intersect)（★ `contextualize_setop` / `merge_descendants` 摊平 N 元树 / 摊平 vs 流式的取舍）
- [九、DML 语句根节点的 `make_cmd()`](#九dml-语句根节点的-make_cmd)
- [十、`PTI_*` 解析期占位 Item 全景（36 个类）](#十pti_-解析期占位-item-全景36-个类逐个讲透)（★ 完整清单 + 变身模式归类 + "为什么需要占位符"归因表）（★ `PT_insert`/`PT_update`/`PT_delete` 与 SELECT 的差异对比表）
- [七、`Parse_context` 与 `itemize()` 深度剖析](#七parse_context-与-itemize-深度剖析)（★ 环境机制：`pc->select` 切换、`m_stack` push/pop 契约、`Query_term` 组装、`itemize()` 五种变身模式）
- [参考](#参考)

---

## 设计思想与理论基础

### 为什么 8.0 要引入 Parse Tree 中间层

8.0 之前（5.7 及更早）的 yacc 动作是**直接构造语义对象**——归约到 `select_item` 时立刻 `new Item_field(...)`、直接往 `LEX` / `SELECT_LEX` 里塞字段。这带来三个问题：

1. **语法与语义耦合**：一条产生式的动作里混杂"建 Item""设上下文""报错"，改文法就要动语义代码；
2. **上下文无法延迟**：有些决策在归约那一刻做不出（例如 `a = b` 里的 `=` 到底造哪个 Item、`SYSDATE` 依赖 `sysdate_is_now` 变量），只能塞一堆"先记下来回头再处理"的临时字段；
3. **不可测试**：没有"纯语法"这一层，无法单独做语法检查（如只想验证语法而不做名字解析）。

8.0 的重构（WL#7196 一带）把这条链路切成**三段**：

```
① 归约（yacc）        只建"语法形状"：NEW_PTN PT_xxx(...) / PTI_xxx(...)
                      不查表、不解析名字、不决定 Item 类型
                              │
② contextualize       把语法形状编译成"语义形状"（AST）
   + itemize          建 Query_block / Query_expression / Table_ref / Item
                      这一层才查表（内建函数表）、解析名字、决定 Item 具体类
                              │
③ prepare/optimize    AST 上的类型推导与优化（见 13_item_expression.md）
```

⇒ **PT 树就是"语法形状"的载体**。它存在的唯一目的是**把"什么时候决定"从归约期推迟到 contextualize 期**。

### PT 与 AST 的本质区别

| | PT（Parse Tree） | AST（`Query_block` / `Item` / `Table_ref`） |
|---|---|---|
| 形状 | **"谁包着谁"**（语法嵌套） | **"谁引用谁、怎么执行"** |
| 生命周期 | **一次性**：contextualize 后即弃 | 贯穿 prepare / optimize / execute |
| 内存 | 全在 `thd->mem_root`，整体回收 | 同在 mem_root，但有精细的 `cleanup()` |
| 可重入 | **不可**（`assert(!contextualized)`） | 可（prepared statement 反复执行） |
| 类名 | `PT_*` / `PTI_*` | `Query_block` / `Item_*` / `Table_ref` |

### 三个"反直觉"

1. **PT 树不是 Item 树**——它是**混合树**：既有纯语法节点（`PT_query_specification`），也有 `Item` 子类（`PTI_*` 占位符），甚至有已成品的 `Item`（`Item_func_plus` 在归约时就被 new 出来了）。见"先有个整体印象"1.3。
2. **有两套正交递归**——`contextualize()`（结构层）与 `itemize()`（表达式层），它们在 `Item : public Parse_tree_node` 处焊接，且互相调用（子查询是切换点）。
3. **`Parse_tree_node_tmpl` 是模板**——`Context` 参数化让 DDL / 分区 / 列属性这些子文法带上各自的环境字段（`Table_ddl_parse_context` 等 5 种实例化），而不污染 `Parse_context`。

---

## 先有个整体印象：一条 SQL 怎么变成 Parse Tree

> 本节源码坐标：`[P1] sql/parse_tree_nodes.h` · `[P2] sql/parse_tree_nodes.cc` · `[P3] sql/parse_tree_node_base.h` · `[P4] sql/parse_tree_items.h` · `[P5] sql/parse_tree_items.cc` · `[P6] sql/sql_yacc.yy` · `[P7] sql/lex.h` · `[P8] sql/sql_lex.h` · `[P9] sql/parse_tree_helpers.h` · `[P10] sql/sql_lex.cc` · `[P15] sql/item_create.cc`

本节用一条具体 SQL 串起后面各章——**先建立全貌，再深入细节**。

```sql
SELECT ABS(t.a) + t.b * 2 FROM t WHERE t.c > 5
```

（这条 SQL 的下游——`fix_fields` 与 `val_*` 求值——见 [`13_item_expression.md`](13_item_expression.md) 的"先有个整体印象"。）

### 1.1 词法：这条 SQL 的每个词是什么 token

| 片段 | token（真实 token 名） | 出处 / 语义值 | 说明 |
|---|---|---|---|
| `SELECT` | `SELECT_SYM` | `[P7]` `{SYM_HK("SELECT", SELECT_SYM)}` | ★ 它是**唯一带语义值的语句关键字**：`[P6]` `%type <lexer.optimizer_hints> SELECT_SYM ...` |
| `ABS` | `IDENT_sys` | `[P7]` 的 symbols 表**没有** `"ABS"` | ★ 所以 ABS 不是关键字，也不是"关键字函数"，走通用函数调用 `function_call_generic` |
| `(` `)` `+` `*` `.` | `'('` `')'` `'+'` `'*'` `'.'` | 单字符运算符**不在** symbols 表里 | 词法器直接返回字符码作为 token |
| `t` / `a` / `b` / `c` | `IDENT_sys` | `yylval->lex_str` | 经 `find_keyword()` 查表失败后回落 |
| `2` / `5` | `NUM` | `[P10]` `int_token()` → `NUM` | 短整数；长整数是 `LONG_NUM`，超范围是 `DECIMAL_NUM` |
| `FROM` | `FROM` | `[P7]` `{SYM("FROM", FROM)}` | 注意 token 名就叫 `FROM`，无 `_SYM` 后缀 |
| `WHERE` | `WHERE` | `[P7]` `{SYM("WHERE", WHERE)}` | 同上 |
| `>` | `GT_SYM` | `[P7]` `{SYM(">", GT_SYM)}` | ★ 同名陷阱：`>=` 是 `GE`（不是 `GE_SYM`） |

```c
// [P7] sql/lex.h（symbols 表片段）
    {SYM(">", GT_SYM)},
    {SYM(">=", GE)},
    {SYM("<<", SHIFT_LEFT)},
...
    {SYM("FROM", FROM)},
...
    {SYM("WHERE", WHERE)},
...
    {SYM_HK("SELECT", SELECT_SYM)},
```

整数 `2` 的 token 判定在 `[P10]`；这里 `length < long_len`（10）直接短路返回 `NUM`：

```c
static inline uint int_token(const char *str, uint length) {
  if (length < long_len)  // quick normal case
    return NUM;           // ← '2' 在这里返回 NUM
  bool neg = false;
  if (*str == '+') { str++; length--; }
  else if (*str == '-') { ... }
  ...
  return ((uchar)str[-1] <= (uchar)cmp[-1]) ? smaller : bigger;
}
```

标识符 `ABS`/`t`/`a`/`c` 的判定在 `[P10]` 的 `lex_one_token()` → `MY_LEX_IDENT` 分支：**先当关键字查表，查不到才回落 `IDENT_sys`**：

```c
      case MY_LEX_IDENT:
        ...
        length = lip->yyLength();
        start = lip->get_ptr();
        ...
        if (start == lip->get_ptr() && c == '.' && ident_map[lip->yyPeek()])
          lip->next_state = MY_LEX_IDENT_SEP;   // ← 't' '.' 的 '.' 走这条路
        else {  // '(' must follow directly if function
          lip->yyUnget();
          if ((tokval = find_keyword(lip, length, c == '('))) {
            lip->next_state = MY_LEX_START;  // Allow signed numbers
            return (tokval);                 // Was keyword  ← SELECT/FROM/WHERE 在这里返回
          }
          lip->yySkip();
        }
        yylval->lex_str = get_token(lip, 0, length);   // ← 回落到 IDENT_sys
```

★ `ABS` 虽不是关键字，但**是内建函数**——它在 `[P15] sql/item_create.cc` 的 `SQL_FN` 表（**不是** yacc 关键字表）里，名字解析发生在 **contextualize 阶段**而非归约阶段：

```c
    {"ABS", SQL_FN(Item_func_abs, 1)},
    {"ACOS", SQL_FN(Item_func_acos, 1)},
    {"ADDTIME", SQL_FN(Item_func_add_time, 2)},
```

于是 token 流（21 个）：

```
 SELECT_SYM  IDENT_sys(ABS) '(' IDENT_sys(t) '.' IDENT_sys(a) ')' '+'
 IDENT_sys(t) '.' IDENT_sys(b) '*' NUM(2)  FROM  IDENT_sys(t)
 WHERE  IDENT_sys(t) '.' IDENT_sys(c) GT_SYM  NUM(5)
```

★ 记住一个 8.0 特色：`SELECT_SYM` 的语义值类型不是空的，而是 `PT_hint_list *`。`/*+ ... */` 提示由 `[P10]` 的 `consume_optimizer_hints()` 直接挂到 `SELECT` 这个 token 的 `yylval` 上：

```c
    lip->yylval->optimizer_hints = hint_list;   // NULL in case of syntax error
```

所以 `SELECT /*+ SET_VAR(x=1) */ ...` 的 hint 是**随 SELECT 关键字一起**进入 `PT_query_specification` 的第 1 个形参的。

### 1.2 Bison 归约序列

#### 1.2.1 优先级：`*` 为什么会先归约

```
%left   EQ EQUAL_SYM GE GT_SYM LE LT NE IS LIKE REGEXP IN_SYM
%left   '|'
%left   '&'
%left   SHIFT_LEFT SHIFT_RIGHT
%left   '-' '+'          ← 低
%left   '*' '/' '%' DIV_SYM MOD_SYM   ← 高（比 '+' 晚声明 ⇒ 更高）
%left   '^'
%left   OR_OR_SYM
%left   NEG '~'
```

算术运算全部写在同一个左递归非终结符 `bit_expr` 上，优先级完全由上面这张表决定：当栈顶是 `bit_expr(ABS(t.a)) '+' bit_expr(t.b)`、lookahead 是 `'*'` 时，因为 `'*'` 优先级 > `'+'`，Bison **移进**而不是归约；等 `bit_expr '*' bit_expr` 先归约成 `Item_func_mul`，`bit_expr '+' bit_expr` 才归约成 `Item_func_plus`。

#### 1.2.2 归约步骤表（自底向上，按实际发生顺序）

| # | 归约的产生式 | 归约结果 |
|---|---|---|
| R1–R3 | `ident: IDENT_sys`（×3: ABS / t / a） | `LEX_STRING` |
| R4 | `simple_ident_q: ident '.' ident` | `PTI_simple_ident_q_2d(@$, "t", "a")` |
| R5–R10 | `simple_ident → simple_expr → bit_expr → predicate → bool_pri → expr` | 纯透传，节点不变 |
| R11 | `select_alias: %empty` | `null_lex_str` |
| R12 | `udf_expr: expr select_alias` | `PTI_udf_expr` |
| R13–R14 | `udf_expr_list: udf_expr` → `opt_udf_expr_list` | `PT_item_list*` |
| R15 | `function_call_generic: IDENT_sys '(' opt_udf_expr_list ')'` | `PTI_function_call_generic_ident_sys` ← **ABS 在这里** |
| R16–R17 | `simple_expr: function_call_generic` → `bit_expr: simple_expr` | 左操作数就绪 |
| R18–R24 | `IDENT_sys(t) '.' IDENT_sys(b)` → `PTI_simple_ident_q_2d("t","b")` → …→ `bit_expr` | 右操作数就绪（lookahead `'*'` ⇒ 移进） |
| R25–R28 | `NUM(2)` → `int64_literal` → `literal` → `simple_expr` → `bit_expr` | `Item_int` |
| R29 | `bit_expr '*' bit_expr` | **`Item_func_mul`** |
| R30 | `bit_expr '+' bit_expr` | **`Item_func_plus`**（外层） |
| R31–R34 | `predicate → bool_pri → expr`；`select_item: expr select_alias`；`select_item_list` | `PTI_expr_with_alias` → `PT_select_item_list` |
| R35 | `select_options: %empty` | `query_spec_options = 0` |
| R36–R41 | `table_ident: ident` → `single_table` → `table_factor` → `table_reference` → `from_tables` | `Table_ident("t")` → `PT_table_factor_table_ident` |
| R42–R43 | `from_clause: FROM from_tables` → `opt_from_clause` | `Mem_root_array_YY<PT_table_reference*>` |
| R44–R48 | `t.c` → `PTI_simple_ident_q_2d("t","c")`；`5` → `Item_int`；`comp_op: GT_SYM`；`bool_pri comp_op predicate` | `PTI_comp_op` |
| R49–R50 | `where_clause: WHERE expr` → `opt_where_clause` | **`PTI_where`** |
| R51 | `opt_group_clause / opt_having_clause / opt_window_clause` 均 `%empty` | `NULL` |
| R52 | `query_specification` | **`PT_query_specification`** |
| R53–R54 | `query_primary` → `query_expression_body` | `{qs, false}` |
| R55–R56 | `opt_order_clause`/`opt_limit_clause` 为 `%empty`；`query_expression` | **`PT_query_expression`** |
| R57 | `select_stmt: query_expression` | **`PT_select_stmt`** |

#### 1.2.3 逐步的真实产生式 + 动作代码

**① 列引用 `t.a` → `PTI_simple_ident_q_2d`**（`[P6]`）

```yacc
ident:
          IDENT_sys    { $$=$1; }        // ← 词法给的 LEX_STRING，原样透传
        | ident_keyword
          {
            THD *thd= YYTHD;
            $$.str= thd->strmake($1.str, $1.length);
            if ($$.str == NULL)
              MYSQL_YYABORT;
            $$.length= $1.length;
          }
        ;

simple_ident_q:
          ident '.' ident
          {
            $$= NEW_PTN PTI_simple_ident_q_2d(@$, $1.str, $3.str);
          }
        | ident '.' ident '.' ident
          ...
        ;
```

```cpp
// [P4] —— 注意它只是"占位符"，此时还没有 Item_field
class PTI_simple_ident_q_2d : public PTI_simple_ident_q_3d {
  typedef PTI_simple_ident_q_3d super;
 public:
  PTI_simple_ident_q_2d(const POS &pos, const char *table_arg,
                        const char *field_arg)
      : super(pos, nullptr, table_arg, field_arg) {}   // db = nullptr，二维（tbl.col）
  bool itemize(Parse_context *pc, Item **res) override;
};
```

**② 函数调用 `ABS(t.a)` → `PTI_function_call_generic_ident_sys`**（`[P6]`）

```yacc
function_call_generic:
          IDENT_sys '(' opt_udf_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_ident_sys(@1, $1, $3);
          }
        | ident '.' ident '(' opt_expr_list ')'          // db.func(...) 走二维名
        | ident '.' ident '.' ident '(' opt_expr_list ')'
        ;

opt_udf_expr_list:
          %empty { $$= NULL; }
        | udf_expr_list { $$= $1; }
        ;

udf_expr_list:
          udf_expr
          {
            $$= NEW_PTN PT_item_list;                   // 实参容器 = PT_item_list
            if ($$ == NULL || $$->push_back($1))
              MYSQL_YYABORT;
          }
        | udf_expr_list ',' udf_expr
          {
            if ($1 == NULL || $1->push_back($3))
              MYSQL_YYABORT;
            $$= $1;
          }
        ;

udf_expr:
          expr select_alias
          {
            $$= NEW_PTN PTI_udf_expr(@$, $1, $2, @1.cpp);
          }
        ;
```

```cpp
// [P4] —— 只记住"函数名 + 实参 PTI 列表"，具体是 native/UDF/stored proc 到 itemize 时才定
class PTI_function_call_generic_ident_sys : public Parse_tree_item {
  typedef Parse_tree_item super;
  LEX_STRING ident;
  PT_item_list *opt_udf_expr_list;
  udf_func *udf;
 public:
  PTI_function_call_generic_ident_sys(const POS &pos, const LEX_STRING &ident_arg,
                                      PT_item_list *opt_udf_expr_list_arg)
      : super(pos), ident(ident_arg), opt_udf_expr_list(opt_udf_expr_list_arg) {}
  bool itemize(Parse_context *pc, Item **res) override;
};
```

> ★ 关键：`ABS` **不是** `function_call_nonkeyword` 里的关键字函数（那条路会直接 `NEW_PTN Item_func_abs(...)`）。它走 `IDENT_sys '(' ... ')'`，所以在归约结束时树上只有"名字 + 参数"，`Item_func_abs` 要等到 `itemize()` 里查 `SQL_FN` 表才造出来。

**③ 算术：`*` 与 `+`**（`[P6]`，同一非终结符 `bit_expr`）

```yacc
bit_expr:
          bit_expr '|' bit_expr %prec '|'
          { $$= NEW_PTN Item_func_bit_or(@$, $1, $3); }
        | bit_expr '&' bit_expr %prec '&'
          { $$= NEW_PTN Item_func_bit_and(@$, $1, $3); }
        | bit_expr SHIFT_LEFT bit_expr %prec SHIFT_LEFT
          { $$= NEW_PTN Item_func_shift_left(@$, $1, $3); }
        | bit_expr '+' bit_expr %prec '+'          // ← R30：本例的最外层
          {
            $$= NEW_PTN Item_func_plus(@$, $1, $3);
          }
        | bit_expr '-' bit_expr %prec '-'
          { $$= NEW_PTN Item_func_minus(@$, $1, $3); }
        | bit_expr '+' INTERVAL_SYM expr interval %prec '+'   // 日期算术重载（同名陷阱）
          { $$= NEW_PTN Item_date_add_interval(@$, $1, $4, $5, 0); }
        | bit_expr '*' bit_expr %prec '*'          // ← R29：本例先归约这一条
          {
            $$= NEW_PTN Item_func_mul(@$, $1, $3);
          }
        | bit_expr '/' bit_expr %prec '/'
          { $$= NEW_PTN Item_func_div(@$, $1,$3); }
        | bit_expr '%' bit_expr %prec '%'
          { $$= NEW_PTN Item_func_mod(@$, $1,$3); }
        | simple_expr %prec SET_VAR                 // ← 单值（列/函数/字面量）从这里升起
        ;
```

> ★ 注意：`Item_func_plus` / `Item_func_mul` 在这里**已经 new 出来了**（它们是真 Item，不是 `PTI_*`），但**没有 itemize**，也没有 `fix_fields`。这是 8.0 的"两段式"：语法阶段只建形状，语义（上下文绑定、递归 itemize）留到 contextualize。

**④ 字面量 `2` / `5`**

```yacc
int64_literal:
          NUM           { $$ = NEW_PTN Item_int(@$, $1); }        // ← '2' / '5'
        | LONG_NUM      { $$ = NEW_PTN Item_int(@$, $1); }
        | ULONGLONG_NUM { $$ = NEW_PTN Item_uint(@$, $1.str, $1.length); }
        ;
```

**⑤ `FROM t` → `PT_table_factor_table_ident`**

```yacc
from_clause:
          FROM from_tables { $$= $2; }
        ;

table_factor:
          single_table                              // ← 本例：直接透传（无动作代码）
        | single_table_parens
        | derived_table { $$ = $1; }
        | joined_table_parens
          { $$= NEW_PTN PT_table_factor_joined_table($1); }
        ;

single_table:
          table_ident opt_use_partition opt_table_alias opt_key_definition
          {
            $$= NEW_PTN PT_table_factor_table_ident($1, $2, $3, $4);
          }
        ;

table_ident:
          ident
          {
            $$= NEW_PTN Table_ident(to_lex_cstring($1));   // ★ Table_ident("t")，不是 PT_* 也不是 Item
            if ($$ == NULL)
              MYSQL_YYABORT;
          }
        ;
```

**⑥ `WHERE t.c > 5` → `PTI_where`**（含比较运算）

```yacc
where_clause:
          WHERE expr    { $$ = NEW_PTN PTI_where(@2, $2); }    // ← @2 = WHERE 的位置
        ;

bool_pri:
          ...
        | bool_pri comp_op predicate                    // ← R47
          {
            $$= NEW_PTN PTI_comp_op(@$, $1, $2, $3);
          }
        ;

comp_op:
          EQ     { $$ = &comp_eq_creator; }
        | EQUAL_SYM { $$ = &comp_equal_creator; }
        | GE     { $$ = &comp_ge_creator; }
        | GT_SYM { $$ = &comp_gt_creator; }              // ← '>' 在这里变成"构造函数创建器"
        | LE     { $$ = &comp_le_creator; }
        | LT     { $$ = &comp_lt_creator; }
        ...
        ;
```

```cpp
// [P4] —— PTI_where 只是"给 expr 打上 CTX_WHERE 上下文"的薄壳
class PTI_where final : public PTI_context {
 public:
  PTI_where(const POS &pos, Item *expr_arg)
      : PTI_context(pos, expr_arg, CTX_WHERE) {}
};
```

#### 1.2.4 归约树（看"谁吃掉谁"）

```
SELECT_SYM  select_options  select_item_list   opt_from_clause   opt_where_clause  …    ← query_specification
   │             │                │                  │                   │
  (hint)        %empty     PT_select_item_list    from_clause       where_clause
                           │                       │                   │
                           │                   from_tables          WHERE expr
                           │                       │                   │
                           │                  table_reference_list   PTI_comp_op
                           │                       │                 ┌──┴───┐
                           │                  table_reference     bool_pri  comp_op  predicate
                           │                       │             (t.c)      GT_SYM    (5)
                           │                  table_factor                   │
                           │                       │                  &comp_gt_creator
                           │                  single_table
                           │                       │
                           │              ┌────────┴─────────┐
                           │         table_ident        opt_use_partition
                           │         Table_ident("t")    opt_table_alias
                           │                              opt_key_definition
                           │
                    PTI_expr_with_alias
                           │
                   expr (= bit_expr '+' bit_expr)
                    ┌──────┴───────┐
        bit_expr(ABS(t.a))     bit_expr '*' bit_expr
             │                  ┌──────┴──────┐
   bit_expr(function_call….)  bit_expr(t.b)  bit_expr(2)
             │
   PTI_function_call_generic_ident_sys
      ┌──────┴───────┐
  IDENT_sys("ABS")  PT_item_list
                       │
                  PTI_udf_expr
                       │
                  PTI_simple_ident_q_2d("t","a")
```

### 1.3 最终 PT 树的形状（★ 混合树）

```
PT_select_stmt                                   [P1] Parse_tree_root，m_sql_command = SQLCOM_SELECT
└─ m_qe : PT_query_expression*                   [P1] m_order/m_limit/m_with_clause 均为 nullptr
   └─ m_body : PT_query_expression_body* (= PT_query_specification)
      └─ PT_query_specification                  [P1] 本篇第五章的主角
         ├─ opt_hints        = nullptr           （SELECT 后无 /*+ */）
         ├─ options          = Query_options{query_spec_options = 0}
         ├─ item_list        : PT_item_list* → 实际对象 PT_select_item_list
         │   └─ value[0] = PTI_expr_with_alias  ← ★ Item 子类（占位符）
         │        └─ expr : Item_func_plus      ← ★ 已经是真 Item（未 itemize/fix_fields）
         │             ├─ args[0] = PTI_function_call_generic_ident_sys("ABS")   ← 占位符
         │             │    └─ opt_udf_expr_list : PT_item_list
         │             │         └─ value[0] = PTI_udf_expr
         │             │              └─ expr = PTI_simple_ident_q_2d("t","a")   ← 占位符
         │             └─ args[1] = Item_func_mul                                ← ★ 真 Item
         │                  ├─ args[0] = PTI_simple_ident_q_2d("t","b")          ← 占位符
         │                  └─ args[1] = Item_int(2)                             ← ★ 真 Item
         ├─ opt_into1        = nullptr
         ├─ m_is_from_clause_implicit = false    （@4 FROM 位置非空）
         ├─ from_clause      : Mem_root_array_YY<PT_table_reference*>[1]
         │   └─ PT_table_factor_table_ident      ← 纯语法节点（不是 Item）
         │        ├─ table_ident = Table_ident("t")   ← 也不是 Item，是 Table_ident
         │        └─ opt_table_alias / opt_use_partition / opt_key_definition = nullptr
         ├─ opt_where_clause : PTI_where*        ← ★ Item 子类（占位符 / 上下文壳）
         │    └─ expr = PTI_comp_op              ← 占位符
         │         ├─ left  = PTI_simple_ident_q_2d("t","c")
         │         ├─ op    = &comp_gt_creator
         │         └─ right = Item_int(5)
         ├─ opt_group_clause  = nullptr
         ├─ opt_having_clause = nullptr
         └─ opt_window_clause = nullptr
```

★ **这就是 8.0 的"混合树"**——同一棵树里 `Item` 节点的地位**不统一**：

| 节点族 | 基类 | 是否是 `Item` | 归约时就知道真身吗 |
|---|---|---|---|
| `PT_select_stmt` / `PT_query_expression` / `PT_query_specification` / `PT_select_item_list` / `PT_table_factor_table_ident` / `Table_ident` | `Parse_tree_node` | ❌ 纯语法节点 | 是（结构性信息齐全） |
| `PTI_simple_ident_q_2d` / `PTI_function_call_generic_ident_sys` / `PTI_comp_op` / `PTI_where` / `PTI_udf_expr` / `PTI_expr_with_alias` | `Parse_tree_item` → **`Item`** | ✅ 是 Item，但只是**占位符** | 否（要等 `itemize()`） |
| `Item_func_plus` / `Item_func_mul` / `Item_int` | `Item` | ✅ 真 Item | 是（归约动作直接 new） |

判断方法：看它有没有 `itemize()` 覆写、有没有 `skip_itemize()` 保护。这也是为什么 contextualize 阶段必须显式遍历——`PT_item_list::contextualize()` 对每个元素调 `item->itemize(pc, &item)`，把 `PTI_*` **原地替换**：

```cpp
// [P9] —— "占位符变身"的发动机
class PT_item_list : public Parse_tree_node {
  typedef Parse_tree_node super;
 public:
  PT_item_list() : value(*THR_MALLOC) {}
  mem_root_deque<Item *> value;

  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (Item *&item : value) {          // ★ 引用！就地替换占位符
      if (item->itemize(pc, &item)) return true;
    }
    return false;
  }
  ...
};
```

### 1.4 contextualize 阶段：PT 树 → QB / QE

调用链（`[P2]`）：

```
LEX::make_sql_cmd()
  └─ PT_select_stmt::make_cmd(thd)
       ├─ Parse_context pc(thd, thd->lex->current_query_block());
       ├─ thd->lex->sql_command = m_sql_command;          // SQLCOM_SELECT
       ├─ m_qe->contextualize(&pc)                        // PT_query_expression
       │    ├─ pc->m_stack.push_back(QueryLevel(..., SC_QUERY_EXPRESSION, has_order))
       │    ├─ m_body->contextualize(pc)                  // → PT_query_specification
       │    │    ├─ pc->m_stack.push_back(QueryLevel(..., SC_QUERY_SPECIFICATION))
       │    │    ├─ pc->select->parsing_place = CTX_SELECT_LIST
       │    │    ├─ options.save_to(pc)                   // → QB::set_base_options(0)
       │    │    ├─ item_list->contextualize(pc)          // → QB::fields = value（并逐项 itemize）
       │    │    ├─ parsing_place = CTX_NONE
       │    │    ├─ contextualize_array(pc, &from_clause) // → QB 的 table_list / context
       │    │    ├─ itemize_safe(pc, &opt_where_clause)   // → PTI_where::itemize → CTX_WHERE
       │    │    ├─ pc->select->set_where_cond(...)       // → QB::m_where_cond
       │    │    ├─ pop QueryLevel；父级 elts.push_back(pc->select)
       │    │    └─ opt_hints->contextualize(pc)          // nullptr → false
       │    ├─ QueryLevel ql = m_stack.back(); expr = ql.m_elts.back(); pop
       │    ├─ switch (expr->term_type()) case QT_QUERY_BLOCK:
       │    │      contextualize_order_and_limit(pc)      // m_order/m_limit 都空 → 直接返回
       │    └─ return false
       ├─ pc.finalize_query_expression()                  // Query_expression::set_query_term(top)
       └─ return new (thd->mem_root) Sql_cmd_select(thd->lex->result);
```

```cpp
// [P2] —— 顶层入口
Sql_cmd *PT_select_stmt::make_cmd(THD *thd) {
  Parse_context pc(thd, thd->lex->current_query_block());   // ← pc.select 指向 LEX 预建好的顶层 Query_block

  thd->lex->sql_command = m_sql_command;

  if (m_qe->contextualize(&pc)) {     // ★ 一行调用驱动整棵 PT 树
    return nullptr;
  }
  ...
  if (pc.finalize_query_expression()) return nullptr;
  ...
  if (thd->lex->sql_command == SQLCOM_SELECT)
    return new (thd->mem_root) Sql_cmd_select(thd->lex->result);
  else  // (thd->lex->sql_command == SQLCOM_DO)
    return new (thd->mem_root) Sql_cmd_do(nullptr);
}
```

```cpp
// [P2] —— 三个"必填槽位"的填充点
bool PT_select_item_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;   // ← super = PT_item_list：先逐个 itemize
  pc->select->fields = value;                  // ← Query_block::fields
  return false;
}

bool PT_table_factor_table_ident::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  THD *thd = pc->thd;
  Yacc_state *yyps = &thd->m_parser_state->m_yacc;    // ★ 回读 yacc 阶段留下的锁类型

  m_table_ref = pc->select->add_table_to_list(        // ← 建 Table_ref 并挂进 QB
      thd, table_ident, opt_table_alias, 0, yyps->m_lock_type, yyps->m_mdl_type,
      opt_key_definition, opt_use_partition, nullptr, pc);
  if (m_table_ref == nullptr) return true;
  if (pc->select->add_joined_table(m_table_ref)) return true;
  return false;
}
```

```cpp
// [P5] —— PTI_* 的"变身"现场（三例）
bool PTI_simple_ident_q_3d::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  THD *thd = pc->thd;
  const char *schema =
      thd->get_protocol()->has_client_capability(CLIENT_NO_SCHEMA) ? nullptr : db;
  if (pc->select->no_table_names_allowed) { ... return true; }
  if ((pc->select->parsing_place != CTX_HAVING) ||
      (pc->select->get_in_sum_expr() > 0)) {
    *res = new (pc->mem_root) Item_field(POS(), schema, table, field);   // ★ t.a/t.b/t.c
  } else {
    *res = new (pc->mem_root) Item_ref(POS(), schema, table, field);     // HAVING 里改引 select list
  }
  if (*res == nullptr) return true;
  return (*res)->itemize(pc, res);
}

bool PTI_function_call_generic_ident_sys::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  udf = nullptr;
  if (using_udf_functions && (udf = find_udf(ident.str, ident.length)) &&
      udf->type == UDFTYPE_AGGREGATE) {
    pc->select->in_sum_expr++;
  }
  if (sp_check_name(&ident)) return true;
  /*
    names are resolved with the following order:
    - MySQL native functions,   ← ABS 命中这一档（item_create.cc 的 SQL_FN 表）
    - User Defined Functions,
    - Stored Functions (assuming the current <use> database)
  */
  Create_func *builder = find_native_function_builder(ident);
  if (builder)
    *res = builder->create_func(thd, ident, opt_udf_expr_list);   // ★ → Item_func_abs
  else { /* UDF / stored function 兜底 */ }
  return *res == nullptr || (*res)->itemize(pc, res);
}

bool PTI_context::itemize(Parse_context *pc, Item **res) {    // PTI_where / PTI_having
  if (super::itemize(pc, res)) return true;

  pc->select->parsing_place = m_parsing_place;      // ★ WHERE → CTX_WHERE

  if (expr->itemize(pc, &expr)) return true;
  if (!expr->is_bool_func()) {
    expr = make_condition(pc, expr);                // WHERE 1 → WHERE 1 <> 0
    if (expr == nullptr) return true;
  }
  assert(pc->select->parsing_place == m_parsing_place);
  pc->select->parsing_place = CTX_NONE;             // ★ 用完立刻复位
  expr->apply_is_true();
  *res = expr;
  return false;
}
```

```cpp
// [P5] —— 别名壳：把 PTI_expr_with_alias 换成内层 Item，并写上 item_name
bool PTI_expr_with_alias::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || expr->itemize(pc, &expr)) return true;
  if (alias.str) {
    expr->item_name.copy(alias.str, alias.length, system_charset_info, false);
  } else if (!expr->item_name.is_set()) {
    expr->item_name.copy(expr_loc.start, (uint)(expr_loc.end - expr_loc.start),
                         pc->thd->charset());    // ★ 无别名 → 用源码原文做列名
  }
  *res = expr;
  return false;
}
```

```cpp
// [P3] —— 收尾：把 Query_term 挂到 Query_expression
bool Parse_context::finalize_query_expression() {
  QueryLevel ql = m_stack.back();
  m_stack.pop_back();
  assert(ql.m_elts.size() == 1);
  Query_term *top = ql.m_elts.back();
  top = top->pushdown_limit_order_by();
  select->master_query_expression()->set_query_term(top);   // ★ QE 与 QB 在此连上
  if (top->validate_structure(nullptr)) return true;
  return false;
}
```

#### 1.4.1 填充结果图

```
           PT 树（解析侧）                          LEX / AST（语义侧）
   ─────────────────────────────────        ────────────────────────────────────────
   PT_select_stmt                            Sql_cmd_select
        │ make_cmd()                              ▲ 由 make_cmd() 末尾 new 出
        ▼                                         │
   PT_query_expression ───────────────► Query_expression (LEX::unit)
        │  finalize_query_expression()            │ m_query_term
        │  set_query_term(Query_block)            ▼
        │                                    Query_term (QT_QUERY_BLOCK)
        ▼                                         │
   PT_query_specification ───────────► Query_block  (LEX::current_query_block())
        │                                    ├─ base_options      = 0
        ├─ item_list ─────────────────►      ├─ fields[0] = Item_func_plus
        │    （PTI_expr_with_alias              │      ├─ args[0] = Item_func_abs
        │      已就地替换为内层 Item）          │      │      └─ args[0] = Item_field(t.a)
        │                                    │      └─ args[1] = Item_func_mul
        │                                    │             ├─ args[0] = Item_field(t.b)
        │                                    │             └─ args[1] = Item_int(2)
        ├─ from_clause ──────────────►      ├─ m_table_list = [Table_ref "t"]
        │                                    ├─ context.table_list
        │                                    │      = context.first_name_resolution_table
        ├─ opt_where_clause (PTI_where) ──►   ├─ m_where_cond = Item_func_gt
        │      已就地替换为 PTI_comp_op           │      ├─ args[0] = Item_field(t.c)
        │      再替换为 Item_func_gt             │      └─ args[1] = Item_int(5)
        ├─ opt_having_clause = nullptr ──►   ├─ m_having_cond = nullptr
        ├─ opt_group_clause  = nullptr ──►   ├─ group_list = {}
        └─ opt_hints = nullptr                └─ parsing_place = CTX_NONE（已复位）
```

### 1.5 最终产物

解析阶段结束时，三样东西就绪：

1. **`Sql_cmd_select`**：`PT_select_stmt::make_cmd()` 的最后一行 `new (thd->mem_root) Sql_cmd_select(...)`。
2. **`LEX`**：`sql_command = SQLCOM_SELECT`，`unit` 指向 `Query_expression`，`current_query_block()` 指向唯一的 `Query_block`。
3. **`Query_expression` / `Query_block`**：由 `LEX` 在**解析开始前**就预建好，contextualize 阶段只负责**填槽位**（详见 [`05_contextualize.md`](05_contextualize.md) 第二章）。

★ 此时树上已经**没有 `PTI_*` 了**——它们全部在 `itemize()` 里被就地替换：

| 归约结束时的占位符 | contextualize 之后 |
|---|---|
| `PTI_expr_with_alias` | 消失，槽位被内层 `Item_func_plus` 占据，同时写好 `item_name = "ABS(t.a) + t.b * 2"` |
| `PTI_function_call_generic_ident_sys("ABS")` | `Item_func_abs`（`find_native_function_builder` → `Create_func_abs::create_func`） |
| `PTI_udf_expr` | 消失，槽位被 `Item_field(t.a)` 占据 |
| `PTI_simple_ident_q_2d("t","a"/"t","b"/"t","c")` | `Item_field` |
| `PTI_where` | 消失，`Query_block::m_where_cond` 指向 `Item_func_gt` |
| `PTI_comp_op` | `Item_func_gt` |

⚠️ 注意这一刻的 Item **仍未被 `fix_fields()` 触碰**：`Item_field` 的 `field`/`table_ref` 还是空的，`Item_func_plus` 的 `data_type` 还没定。类型推导与列绑定发生在下一步 `Query_block::prepare()`——那正是 [`13_item_expression.md`](13_item_expression.md)「先有个整体印象」的 Stage 2。

---

## 一、基类体系：`Parse_tree_root` 与 `Parse_tree_node_tmpl<Context>`

> 本节源码坐标增补：`[P3] sql/parse_tree_node_base.h` · `[P4] sql/parse_tree_node_base.cc` · `[P11] sql/item.h` · `[P12] sql/sql_yacc.yy` · `[P13] sql/sql_lex.h` · `[P14] sql/sql_class.cc`

### 1.1 全景：PT 树在编译流水线中的位置

```
  SQL 文本
     │  词法（sql/sql_lex.cc + lex_token.h）
     ▼
  token 流 ──► bison: MYSQLparse(thd, &root)         [P14]
                   │  产生式动作里 NEW_PTN new PT_xxx(...)  [P12]
                   ▼
        ┌──────────────────────────────────────────────┐
        │  PT 树：root 为 Parse_tree_root*             │
        │  所有节点分配在 YYMEM_ROOT == thd->mem_root   │
        └──────────────────────────────────────────────┘
                   │  THD::sql_parser(): lex->make_sql_cmd(root)
                   ▼
             root->make_cmd(thd)      ← Parse_tree_root 唯一接口
                   │
                   ▼
        contextualize(&pc)  结构层递归（QE / QB / Table_ref / ORDER / LIMIT）
                   │
                   ├── 遇到表达式 → itemize(&pc, &item)  表达式层递归（Item 树）
                   │                    └── 遇到子查询 → 再回到 contextualize()
                   ▼
        LEX + Sql_cmd（AST）  ← PT 树到此"一次性消费"完毕
                   ▼
        Query_expression::prepare() / optimize() / execute()
```

### 1.2 `Parse_tree_root`：顶层只有一个纯虚接口

```cpp
// [P1]
/**
  Base class for all top-level nodes of SQL statements
  @ingroup ptn_stmt
*/
class Parse_tree_root {
  Parse_tree_root(const Parse_tree_root &) = delete;
  void operator=(const Parse_tree_root &) = delete;

 protected:
  virtual ~Parse_tree_root() = default;   // ① 析构非 public：根节点只能被 mem_root 整体回收
  Parse_tree_root() = default;

 public:
  virtual Sql_cmd *make_cmd(THD *thd) = 0;   // ② 唯一接口
};
```

- ① 拷贝构造/`operator=` 被 `delete`，且析构为 `protected`：**根节点禁止复制、禁止栈上/显式 delete**，只能由 `new (thd->mem_root)` 创建并被 MEM_ROOT 整体回收。
- ② **为什么顶层只需要 `make_cmd()` 一个接口？** 因为根节点的职责边界极其干净：bison 归约出整棵 PT 树后，交给 `LEX` 的唯一诉求是"给我一个可执行对象"。返回值约定也很统一：成功返回 `Sql_cmd*`，失败返回 `nullptr`（`m_sql_cmd == nullptr` 即判错），这样"报错"与"OOM"走同一条通道。
- 继承者覆盖全部语句类型：`PT_select_stmt`、`PT_insert`、`PT_update`、`PT_delete`、`PT_create_table_stmt`、`PT_call`、`PT_show_*` 等。

### 1.3 ★ `Parse_tree_node_tmpl<Context>`：为什么是模板

```cpp
// [P3]
template <typename Context>
class Parse_tree_node_tmpl {
  friend class Item;  // for direct access to the "contextualized" field

  Parse_tree_node_tmpl(const Parse_tree_node_tmpl &);  // undefined
  void operator=(const Parse_tree_node_tmpl &);        // undefined

#ifndef NDEBUG
 private:
  bool contextualized;  // true if the node object is contextualized  ①
#endif                  // NDEBUG

 public:
  typedef Context context_t;                            // ② 关键 typedef

  static void *operator new(size_t size, MEM_ROOT *mem_root,
                            const std::nothrow_t &arg
                            [[maybe_unused]] = std::nothrow) noexcept {
    return mem_root->Alloc(size);                        // ③ 只认 MEM_ROOT
  }
  static void operator delete(void *ptr [[maybe_unused]],
                              size_t size [[maybe_unused]]) {
    TRASH(ptr, size);                                    // ④ debug 下用垃圾填充，暴露 use-after-free
  }
  static void operator delete(void *, MEM_ROOT *,
                              const std::nothrow_t &) noexcept {}

 protected:
  Parse_tree_node_tmpl() {
#ifndef NDEBUG
    contextualized = false;
#endif
  }

 public:
  virtual ~Parse_tree_node_tmpl() = default;

#ifndef NDEBUG
  bool is_contextualized() const { return contextualized; }
#endif
```

- ① `contextualized` **只在 `!NDEBUG`（debug/assert 版）下存在**：release 构建里这个字段被彻底编译掉，节点不为一个"仅用于断言的标记"付 1 字节/节点的代价。代价是 release 下无法检测"同一节点被 contextualize 两次"。
- ② `typedef Context context_t;` 是模板化的真正使用者入口——yacc 宏 `CONTEXTUALIZE(x)` 里写 `std::remove_reference<decltype(*x)>::type::context_t pc(...)`，从而对 `Table_ddl_node` 派生类自动造出 `Table_ddl_parse_context`，对普通 `PT_*` 造出 `Parse_context`。**若不用模板，`context_t` 就无法随节点类型变化。**
- ③④ `operator new` 只接受 `MEM_ROOT*`（`nothrow` 语义，OOM 返回 `nullptr`），且**没有**真正归还内存的 `delete`——配合 `TRASH()`，语义是"只分配、整体释放"。

**`Context` 参数化解决了什么？** 8.0.39 里被实例化为 **5 种环境**：

| 实例化 | 用途 |
|---|---|
| `Parse_tree_node` = `Parse_tree_node_tmpl<Parse_context>` | 90% 以上节点（DML/查询结构） |
| `Table_ddl_node` = `...<Table_ddl_parse_context>` | CREATE/ALTER TABLE 的列、索引、表选项 |
| `PT_column_attr_base` 的基类 = `...<Column_parse_context>` | 列类型/列属性 |
| `Parse_tree_part_node` = `...<Partition_parse_context>` | 分区子句 |
| `PT_alter_tablespace_option_base` = `...<Alter_tablespace_parse_context>` | ALTER TABLESPACE 选项 |

⇒ **模板让同一个 `contextualize()` 协议在不同子文法里带上不同的"额外参数包"**，而不必给 `Parse_context` 塞一堆用不上的字段。派生 context 都是 `Parse_context` 的子类，因此 `pc->thd`、`pc->mem_root`、`pc->select` 三件套在所有实例里都可用。

```cpp
// [P3]
typedef Parse_tree_node_tmpl<Parse_context> Parse_tree_node;
```

### 1.4 ★ `contextualize()` 基类实现：逐行注释

```cpp
// [P3]
  /**
    Do all context-sensitive things and mark the node as contextualized
    @retval false  success
    @retval true   syntax/OOM/etc error
  */
  virtual bool contextualize(Context *pc) {
    uchar dummy;                                                  // ① 取栈上局部量地址
    if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy))     // ② 递归深度守卫
      return true;

#ifndef NDEBUG
    assert(!contextualized);                                      // ③ 防重入
    contextualized = true;                                        // ④ 打标记
#endif

    return false;                                                 // ⑤ 默认：什么都不做
  }
```

- ① `uchar dummy;` 不是为使用，而是为了**拿到当前栈帧的一个真实地址**。
- ② `check_stack_overrun()` 用这个地址与线程栈边界比较。**为什么必须在每一层做？** 因为 `contextualize()` 是**深度优先递归**，深度正比于语法嵌套深度；而 SQL 文本长度几乎不受限（嵌套子查询、`a AND a AND a ...`），可以轻易构造出上万层嵌套。bison 自己的 `YYMAXDEPTH`（`[P12]`，3200）只保护**归约栈**，不保护 `contextualize()` 的 C++ 递归栈——两者是两回事。
- ③ `assert(!contextualized)`：一个节点**只允许被 contextualize 一次**，直接印证"PT 节点是一次性消费"。
- ⑤ 默认返回 `false`：绝大多数叶子节点**不需要任何上下文处理**，只需在 DAG 上被"访问到"。因此派生类的标准写法是：

```cpp
bool PT_xxx::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;   // ① 必须先调基类：栈检查 + 防重入
  ...                                          // ② 再做自己的事
  return false;
}
```

### 1.5 ★ `Parse_context`：递归传递的唯一环境对象

```cpp
// [P3]
enum Surrounding_context {          // 语法上"我在谁里面"
  SC_TOP, SC_QUERY_SPECIFICATION, SC_TABLE_VALUE_CONSTRUCTOR,
  SC_QUERY_EXPRESSION, SC_SUBQUERY,
  SC_UNION_DISTINCT, SC_UNION_ALL,
  SC_INTERSECT_DISTINCT, SC_INTERSECT_ALL,
  SC_EXCEPT_DISTINCT, SC_EXCEPT_ALL
};

struct QueryLevel {
  Surrounding_context m_type;
  /** elts 在 contextualize 过程中充当临时容器：
      收集子节点（子节点把自己 push 进父节点的 elts）→ 延迟组合 → 构建 Query_term 树 */
  mem_root_deque<Query_term *> m_elts;
  bool m_has_order{false};
  QueryLevel(MEM_ROOT *mem_root, Surrounding_context sc, bool has_order = false)
      : m_type(sc), m_elts(mem_root), m_has_order(has_order) {}
};

/**
  Environment data for the contextualization phase
*/
struct Parse_context {
  THD *const thd;                      ///< Current thread handler
  MEM_ROOT *mem_root;                  ///< Current MEM_ROOT
  Query_block *select;                 ///< Current Query_block object
  mem_root_deque<QueryLevel> m_stack;  ///< Aids query term tree construction
  bool finalize_query_expression();
  Parse_context(THD *thd, Query_block *sl);
  bool is_top_level_union_all(Surrounding_context op);
};
```

```cpp
// [P4]
Parse_context::Parse_context(THD *thd_arg, Query_block *sl_arg)
    : thd(thd_arg), mem_root(thd->mem_root), select(sl_arg),
      m_stack(thd->mem_root) {
  m_stack.push_back(QueryLevel(thd->mem_root, SC_TOP));   // 初始：栈底永远是 SC_TOP
}
```

**四个字段就是 contextualize 递归的全部"可变状态"**：

- `thd`：`const` 指针，拿 LEX、报错、OOM 分配；
- `mem_root`：contextualize 期间新建对象（Item / Query_block / Table_ref）的分配源；
- `select`：**当前** `Query_block`——最"可变"的字段，进入子查询时被换成子 QB，出来再换回（见 `PT_subquery`、`PT_derived_table` 里的 `orig_query_block` 保存/恢复）；
- `m_stack`：`QueryLevel` 栈，专门服务 `Query_term` 树构造（UNION/INTERSECT/EXCEPT/括号嵌套）。

⇒ `Parse_context` 是**值语义、按指针传递、生命周期只覆盖一次 contextualize 递归**。

（★ 本章只是概览；`pc->select` 的切换场景、`m_stack` 契约、`Query_term` 组装、`is_top_level_union_all()`、5 种 `Context` 实例化的完整剖析见[第七章](#七parse_context-与-itemize-深度剖析)）。

### 1.6 `error()` / `errorf()`：带 `POS` 的延迟报错

```cpp
// [P3]
  void error(Context *pc, const POS &pos) const {
    pc->thd->syntax_error_at(pos);          // ① 只有位置、无自定义消息
  }
  void error(Context *pc, const POS &pos, const char *msg) const {
    pc->thd->syntax_error_at(pos, "%s", msg);   // ② 位置 + 固定消息
  }
  void errorf(Context *pc, const POS &pos, const char *format, ...) const
      MY_ATTRIBUTE((format(printf, 4, 5)));      // ③ 位置 + printf 风格
};

template <typename Context>
inline void Parse_tree_node_tmpl<Context>::errorf(Context *pc, const POS &pos,
                                                  const char *format, ...) const {
  va_list args;
  va_start(args, format);
  pc->thd->vsyntax_error_at(pos, format, args);
  va_end(args);
}
```

- 三个重载挂在**基类**上，所以任何子类（包括非 `Parse_context` 实例化的分支）都能 `this->error(pc, m_pos)`，`Context` 模板参数让 `pc->thd` 依然可用。
- `POS` 即 `YYLTYPE`（`sql/parse_location.h`），携带指向**原始 SQL 文本**的 `raw.start` / `raw.end`；`syntax_error_at` 用它截出出错片段。⇒ PT 节点的报错天然是"延迟"的：**语法阶段只记位置，语义阶段才真正报错**。

### 1.7 生命周期与内存管理

```cpp
// [P12]
#define YYP (YYTHD->m_parser_state)
#define YYMEM_ROOT (YYTHD->mem_root)
#define NEW_PTN new(YYMEM_ROOT)          // 所有 PT 节点的诞生方式
```

- **分配在哪？** 全部在 `thd->mem_root`，经 `operator new(size, MEM_ROOT*, nothrow)`。**OOM 时返回 `nullptr` 而不抛异常**——这就是 yacc 动作里到处写 `if ($$ == NULL) MYSQL_YYABORT;` 的原因。
- **为什么 contextualize 之后可以丢弃？** 因为产物（`Query_block` / `Query_expression` / `Table_ref` / `Item`）全部建在 `pc->mem_root` 或挂进 `LEX`/`Query_block` 成员里；PT 节点只是"配方"。唯一还被长期持有的是极少数有 `print()` 的对象（如 `PT_with_clause` 被存进 `Query_expression::m_with_clause` 以便视图打印）——例外而非常态。
- **有没有 `cleanup()`？** PT 基类**没有** `cleanup()`——不需要逐节点析构，语句结束时 `thd->mem_root` 整体归还。`Item::cleanup()` 是 **Item 树**（每次执行前后复用）的概念，与 PT 节点无关，不要混淆。

---

## 二、两套正交递归：`contextualize()` 与 `itemize()`

### 2.1 焊接点：`Item` 本身就是 `Parse_tree_node`

```cpp
// [P11]
class Item : public Parse_tree_node {
  typedef Parse_tree_node super;
  ...
  bool skip_itemize(Item **res) {
    *res = this;
    return !is_parser_item;         // ① 非 parser 直建的 Item 不必 itemize
  }
  virtual bool itemize(Parse_context *pc, Item **res);
};
```

```cpp
// [P11] sql/item.cc
bool Item::itemize(Parse_context *pc, Item **res) {
  if (skip_itemize(res)) return false;      // ① 已定型的直接返回
  if (super::contextualize(pc)) return true; // ② ★复用 contextualize 的栈检查/防重入
  pc->thd->add_item(this);                   // ③ 挂进 THD::item 列表
  if (pc->select) {
    const enum_parsing_context place = pc->select->parsing_place;
    if (place == CTX_SELECT_LIST || place == CTX_HAVING || place == CTX_ORDER_BY) {
      pc->select->select_n_having_items++;   // ④ 统计项数
    }
  }
  return false;
}
```

- ② 是**两条递归的焊接点**：`itemize()` 先调 `super::contextualize(pc)`，白拿栈溢出检查与 `contextualized` 防重入。这也是 `[P3]` 里 `friend class Item;` 的由来。
- **`res` 是"替换"语义**，不是简单回填：`*res` 可以指向一个**新分配的对象**来替换 `this`——这正是 `PTI_*` 占位符的工作方式。

### 2.2 两套递归的分工与互调

```
                    contextualize(pc)  ← 结构层
    PT_query_expression ──► PT_query_specification ──► PT_table_reference ──► ...
            │                        │                          │
            │ 建 Query_expression   │ 建 Query_block           │ 建 Table_ref / join 链
            └── ORDER/LIMIT ─────────┴── WHERE/HAVING ──────────┘
                                             │
                                             ▼ 进入表达式层
                                    itemize(pc, &item)  ← 表达式层
                                    Item_func ─► Item_field ─► ...
                                             │
                                             ▼ 遇到 (SELECT ...) / EXISTS (...)
                                    PTI_singlerow_subselect::itemize()
                                             │
                                             └──► subselect->contextualize(pc)  ← 回到结构层！
```

### 2.3 三个"过河"的辅助函数

```cpp
// [P2]
template <typename Context, typename Node>
bool contextualize_safe(Context *pc, Node node) {   // ① NULL-tolerant 单节点
  if (node == nullptr) return false;
  return node->contextualize(pc);
}

template <typename Context>
bool contextualize_safe(Context *pc, mem_root_deque<Item *> *list) {  // ② 列表版走 itemize
  if (list == nullptr) return false;
  for (Item *&item : *list) {
    if (item->itemize(pc, &item)) return true;
  }
  return false;
}

template <typename Context, typename Node, typename... Nodes>
bool contextualize_safe(Context *pc, Node node, Nodes... nodes) {  // ③ 可变参数：短路
  return contextualize_safe(pc, node) || contextualize_safe(pc, nodes...);
}

bool itemize_safe(Parse_context *pc, Item **item) {  // ④ 单 Item 的 NULL-tolerant itemize
  if (*item == nullptr) return false;
  return (*item)->itemize(pc, item);
}
```

```cpp
// [P9] —— 数组版：注意它检查 will_contextualize
template <typename Context, typename Array>
bool contextualize_array(Context *pc, Array *array) {
  for (auto it : *array) {
    if (pc->thd->lex->will_contextualize && it->contextualize(pc)) return true;
  }
  return false;
}
```

★ `Parse_context` 在两套递归中的完整角色、以及 `itemize()` 的**五种变身模式**，见[第七章](#七parse_context-与-itemize-深度剖析)。

★ `will_contextualize`（`[P13]`，`LEX` 构造里初始化为 `true`）是"纯解析模式"开关——只需要语法树而不做语义转换时置 `false`，让 `contextualize()` 整体空转。注意它只被 `contextualize_array` 和 yacc 的 `CONTEXTUALIZE` 宏检查，**不是**每个 `contextualize()` 都查，属于"外层把关"设计。

### 2.4 总入口

```cpp
// [P14]
bool THD::sql_parser() {
  Parse_tree_root *root = nullptr;
  if (MYSQLparse(this, &root) || is_error()) {     // ① bison：产出 PT 树
    cleanup_after_parse_error();                   // ② 失败：回滚 LEX 副作用
    return true;
  }
  if (root != nullptr && lex->make_sql_cmd(root)) { // ③ PT 树 → Sql_cmd
    return true;
  }
  return false;                                     // ④ 成功：PT 树此后不再被引用
}
```

```cpp
// [P13]
bool LEX::make_sql_cmd(Parse_tree_root *parse_tree) {
  if (!will_contextualize) return false;         // ① "纯解析"开关
  m_sql_cmd = parse_tree->make_cmd(thd);         // ② 唯一调用点
  if (m_sql_cmd == nullptr) return true;
  assert(m_sql_cmd->sql_command_code() == sql_command);   // ③ 命令码一致性校验
  return false;
}
```

---

## 三、`PT_*` 节点族：17 个范畴与代表实现

### 3.1 范畴总表（按文件/上下文类型归类）

| # | 范畴 | 主要文件 | 上下文类型 | 代表类 |
|---|---|---|---|---|
| 1 | 语句根节点（`make_cmd`） | `[P1]` | — (`Parse_tree_root`) | `PT_select_stmt` |
| 2 | 查询表达式 / 集合运算 | `[P1]` `[P2]` | `Parse_context` | `PT_query_expression` |
| 3 | 查询块（query specification） | `[P1]` `[P2]` | `Parse_context` | `PT_query_specification` |
| 4 | 选择列表 / Item 列表 / INTO | `[P1]` `[P9]` | `Parse_context` | `PT_select_item_list` |
| 5 | 表引用（table factor） | `[P1]` `[P2]` | `Parse_context` | `PT_table_factor_table_ident` |
| 6 | JOIN 结构 | `[P1]` `[P2]` | `Parse_context` | `PT_joined_table_on` / `PT_cross_join` |
| 7 | 派生表 / 子查询 | `[P1]` `[P2]` | `Parse_context` | `PT_derived_table` / `PT_subquery` |
| 8 | ORDER BY / GROUP BY | `[P1]` `[P2]` | `Parse_context` | `PT_order` / `PT_group` |
| 9 | LIMIT | `[P1]` `[P2]` | `Parse_context` | `PT_limit_clause` |
| 10 | WITH / CTE | `[P1]` `[P2]` | `Parse_context` | `PT_with_clause` |
| 11 | 窗口 | `[P10]` | `Parse_context` | `PT_window` |
| 12 | 锁子句（FOR UPDATE / LOCK IN SHARE MODE） | `[P1]` `[P2]` | `Parse_context` | `PT_locking_clause` |
| 13 | DML 值列表 / 赋值 | `[P1]` `[P2]` | `Parse_context` | `PT_insert_values_list` |
| 14 | SET / 系统变量 / 用户变量 | `[P1]` `[P2]` | `Parse_context` | `PT_set_variable` |
| 15 | 列类型与列属性 | `[P7]`（36 个） | `Column_parse_context` | `PT_default_column_attr` |
| 16 | 分区子句 | `[P8]`（29 个） | `Partition_parse_context` | `PT_partition_min_rows` |
| 17 | 表/索引/键/表选项 + hint + 杂项命令 | `[P1]` `[P9]` 等 | `Table_ddl_parse_context` / `Parse_context` | `PT_inline_index_definition` |

### 3.2 范畴 1 —— 语句根节点：`PT_select_stmt::make_cmd()`

```cpp
// [P1]
class PT_select_stmt : public Parse_tree_root {
  typedef Parse_tree_root super;

 public:
  PT_select_stmt(enum_sql_command sql_command, PT_query_expression_body *qe)
      : m_sql_command(sql_command), m_qe(qe), m_into(nullptr),
        m_has_trailing_locking_clauses{false} {}
  explicit PT_select_stmt(PT_query_expression_body *qe,
                          PT_into_destination *into = nullptr,
                          bool has_trailing_locking_clauses = false)
      : m_sql_command{SQLCOM_SELECT}, m_qe{qe}, m_into{into},
        m_has_trailing_locking_clauses{has_trailing_locking_clauses} {}

  Sql_cmd *make_cmd(THD *thd) override;

 private:
  enum_sql_command m_sql_command;
  PT_query_expression_body *m_qe;   // 实际是 PT_query_expression
  PT_into_destination *m_into;
  const bool m_has_trailing_locking_clauses;
};
```

```cpp
// [P2] —— 顶层入口，全链路骨架
Sql_cmd *PT_select_stmt::make_cmd(THD *thd) {
  Parse_context pc(thd, thd->lex->current_query_block());  // ① 造环境：栈底 SC_TOP

  thd->lex->sql_command = m_sql_command;                   // ② 先落 sql_command

  if (m_qe->contextualize(&pc)) {                          // ③ ★ 整棵 PT 树的递归入口
    return nullptr;
  }

  const bool has_into_clause_inside_query_block = thd->lex->result != nullptr;

  if (has_into_clause_inside_query_block && m_into != nullptr) {
    my_error(ER_MULTIPLE_INTO_CLAUSES, MYF(0));            // ④ 语义检查（非语法）
    return nullptr;
  }
  if (contextualize_safe(&pc, m_into)) {                   // ⑤ INTO 也走 contextualize
    return nullptr;
  }

  if (pc.finalize_query_expression()) return nullptr;      // ⑥ 收尾：造出 Query_term 根

  if (m_into != nullptr && m_has_trailing_locking_clauses) {
    push_warning(thd, ER_WARN_DEPRECATED_INNER_INTO);
  } else if (has_into_clause_inside_query_block &&
             thd->lex->unit->is_set_operation()) { ... }

  if (thd->lex->sql_command == SQLCOM_SELECT)              // ⑦ 产出 Sql_cmd
    return new (thd->mem_root) Sql_cmd_select(thd->lex->result);
  else  // (thd->lex->sql_command == SQLCOM_DO)
    return new (thd->mem_root) Sql_cmd_do(nullptr);
}
```

- ① `Parse_context pc(...)`：环境对象**在栈上**，随 `make_cmd` 帧存在，正好覆盖整次递归。
- ⑥ `finalize_query_expression()` 只在最外层调用一次（`PT_subquery` 内部再各自调用一次）。
- ⑦ `Sql_cmd` 是 PT 树留给执行层的**唯一产物**；`nullptr` 即失败。

### 3.3 范畴 2 —— 查询表达式：`PT_query_expression::contextualize()`

★ 这是"压 `QueryLevel` → 递归 body → 组装 `Query_term`"的标准范式，也是全族最复杂的 `contextualize` 之一。

```cpp
// [P2]
bool PT_query_expression::contextualize(Parse_context *pc) {
  pc->m_stack.push_back(
      QueryLevel(pc->mem_root, SC_QUERY_EXPRESSION, m_order != nullptr));  // ① 压层
  if (contextualize_safe(pc, m_with_clause))                              // ② WITH 先处理
    return true;

  if (Parse_tree_node::contextualize(pc) || m_body->contextualize(pc))     // ③ 递归 body
    return true;

  QueryLevel ql = pc->m_stack.back();
  Query_term *expr = ql.m_elts.back();                                     // ④ 取子节点成果
  pc->m_stack.pop_back();

  switch (expr->term_type()) {                                             // ⑤ 按类型组装
    case QT_UNARY: {
      Query_term_unary *ex = down_cast<Query_term_unary *>(expr);
      Query_expression *qe = pc->select->master_query_expression();
      // 建"后处理块"，使 ( SELECT a FROM t ) ORDER BY t.a 能去掉括号
      if (ex->set_block(qe->create_post_processing_block(ex))) return true;

      if (m_order != nullptr)
        ex->query_block()->order_list = m_order->order_list->value;
      if (m_limit != nullptr) {
        ex->query_block()->select_limit = m_limit->limit_options.limit;
        ex->query_block()->offset_limit = m_limit->limit_options.opt_offset;
      }

      Query_block *orig_query_block = pc->select;                          // ⑥ 切 QB
      LEX *lex = pc->thd->lex;
      pc->select = ex->query_block();
      lex->push_context(&pc->select->context);
      assert(pc->select->parsing_place == CTX_NONE);
      if (contextualize_safe(pc, m_order, m_limit)) return true;           // ⑦ ORDER/LIMIT
      lex->pop_context();
      pc->select = orig_query_block;                                       // ⑧ 还原 QB

      if (pc->m_stack.back().m_type == SC_QUERY_EXPRESSION) {  // 3 deep or more
        expr = new (pc->mem_root) Query_term_unary(pc->mem_root, expr);    // ⑨ 再包一层括号
        if (expr == nullptr) return true;
      }
      QueryLevel upper = pc->m_stack.back();
      pc->m_stack.pop_back();
      ql.m_elts.pop_back();
      if (upper.m_type == SC_UNION_DISTINCT || upper.m_type == SC_UNION_ALL ||
          upper.m_type == SC_EXCEPT_DISTINCT || upper.m_type == SC_EXCEPT_ALL ||
          upper.m_type == SC_INTERSECT_DISTINCT ||
          upper.m_type == SC_INTERSECT_ALL)
        ql = upper;                                                        // ⑩ 并入集合运算层
      ql.m_elts.push_back(expr);
      pc->m_stack.push_back(ql);
    } break;
    case QT_QUERY_BLOCK: { ... }   // 与 QT_UNARY 类似，见下
    case QT_UNION: case QT_EXCEPT: case QT_INTERSECT: { ... }  // 集合运算：已带 ORDER/LIMIT 时再包 unary
  }
  return false;
}
```

- ①③ 是 `QueryLevel` 栈的标准用法：**进入压层 → 递归 → 子节点把结果 push 到 `m_elts` → 取出 → 出层**。
- ⑥⑦⑧ 是"切换当前 `Query_block`"的**固定三连**：保存 `orig_query_block` → `pc->select = ...` + `lex->push_context()` → 递归 → `lex->pop_context()` + 还原。ON 子句、派生表、子查询都用同一范式。
- ⑨ `Query_term_unary` 的反复包装，正是 SQL 里"多余括号"的语义：每多一层 `SC_QUERY_EXPRESSION` 就多包一层 unary 节点。
- ⑩ 如果上层是 UNION/INTERSECT/EXCEPT，就**不另起一层**，而是把结果并进集合运算的 `elts`。

配套的 ORDER/LIMIT 下推判断：

```cpp
// [P2]
bool PT_query_expression::contextualize_order_and_limit(Parse_context *pc) {
  if (m_order == nullptr && m_limit == nullptr) return false;

  if (m_body->can_absorb_order_and_limit(m_order != nullptr,
                                         m_limit != nullptr)) {
    if (contextualize_safe(pc, m_order, m_limit)) return true;  // ① 就地消化
  } else {
    LEX *lex = pc->thd->lex;
    Query_block *orig_query_block = pc->select;
    pc->select = nullptr;                                       // ② 借用"空 QB"上下文
    lex->push_context(&pc->select->context);
    assert(pc->select->parsing_place == CTX_NONE);
    bool res = contextualize_safe(pc, m_order, m_limit);
    lex->pop_context();
    pc->select = orig_query_block;
    if (res) return true;
  }
  return false;
}
```

⇒ `can_absorb_order_and_limit()` 决定了 `(SELECT ... ) ORDER BY t.a` 里 ORDER BY 是被内层"吸收"还是必须新建后处理块——体现"**PT 树不重写，由 contextualizer 决定 AST 形状**"的设计原则。

### 3.4 范畴 4 —— 选择列表：`PT_select_item_list` 与 `PT_item_list`

```cpp
// [P9] —— 结构层节点驱动表达式层递归的典范
class PT_item_list : public Parse_tree_node {
  typedef Parse_tree_node super;

 public:
  PT_item_list() : value(*THR_MALLOC) {}    // ① *THR_MALLOC == thd->mem_root

  mem_root_deque<Item *> value;

  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (Item *&item : value) {            // ② 逐个 itemize，就地替换
      if (item->itemize(pc, &item)) return true;
    }
    return false;
  }
};

// [P2]
bool PT_select_item_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;   // ← super = PT_item_list：先逐个 itemize
  pc->select->fields = value;                  // ① 一键搬家：交给 Query_block
  return false;
}
```

- ② `Item *&item` 是**引用遍历**：`itemize` 可能把 `item` 换成新对象（PTI → 真 Item），引用保证原地写回列表——这是"两套递归耦合点"的最小样板。

### 3.5 范畴 5 —— 表引用：`PT_table_factor_table_ident`

```cpp
// [P1]
class PT_table_reference : public Parse_tree_node {
 public:
  Table_ref *m_table_ref;            // ① contextualize 的产物从这里往外传

  /**
    Lets us build a parse tree top-down, which is necessary due to the
    context-dependent nature of the join syntax. ...
  */
  virtual PT_joined_table *add_cross_join(PT_cross_join *cj);  // ② 自顶向下插左叶子
};
```

```cpp
// [P2]
bool PT_table_factor_table_ident::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  THD *thd = pc->thd;
  Yacc_state *yyps = &thd->m_parser_state->m_yacc;   // ① 回读 yacc 阶段留下的锁类型

  m_table_ref = pc->select->add_table_to_list(
      thd, table_ident, opt_table_alias, 0, yyps->m_lock_type, yyps->m_mdl_type,
      opt_key_definition, opt_use_partition, nullptr, pc);    // ② 建 Table_ref 并挂进 QB
  if (m_table_ref == nullptr) return true;
  if (pc->select->add_joined_table(m_table_ref)) return true; // ③ 加入 join 链
  return false;
}
```

- ① 又一次"语法状态回读"：`SELECT HIGH_PRIORITY` 在 `PT_query_specification` 里写入 `yyps->m_lock_type`，这里在建表引用时才消费。**yacc 的 `Yacc_state` 是语法阶段与语义阶段之间的一块共享黑板。**
- `m_table_ref` 是**输出槽**：父节点（JOIN、派生表包装）直接读 `child->m_table_ref`——这是 PT 家族"父读子产物"的通用约定（对比：`Query_term` 用 `m_stack` 传，表引用用成员槽传）。

### 3.6 范畴 6 —— JOIN：contextualize 期做"规范化改写"

★ `contextualize()` 不只是"搬运"，还做**规范化改写**（RIGHT JOIN 在 MySQL 内部一律表示为交换序的 LEFT JOIN）：

```cpp
// [P2]
bool PT_joined_table::contextualize_tabs(Parse_context *pc) {
  if (m_left_table_ref != nullptr) return false;  // ① 幂等：已做过就跳过

  bool was_right_join = m_type & JTT_RIGHT;
  if (was_right_join) {                            // ② 右连接统一改写成左连接
    m_type = static_cast<PT_joined_table_type>((m_type & ~JTT_RIGHT) | JTT_LEFT);
    std::swap(m_left_pt_table, m_right_pt_table);
  }

  if (m_left_pt_table->contextualize(pc) || m_right_pt_table->contextualize(pc))
    return true;                                   // ③ 自底向上递归两侧

  m_left_table_ref = m_left_pt_table->m_table_ref; // ④ 收集两侧产物
  m_right_table_ref = m_right_pt_table->m_table_ref;

  if (m_left_table_ref == nullptr || m_right_table_ref == nullptr) {
    error(pc, m_join_pos);                         // ⑤ 基类 error()，带 POS
    return true;
  }

  if (m_type & JTT_LEFT) {
    m_right_table_ref->outer_join = true;
    if (was_right_join) {
      m_right_table_ref->join_order_swapped = true;
      m_right_table_ref->query_block->set_right_joins();
    }
  }
  return false;
}
```

带 ON 的连接是**结构层 + 表达式层 + 名字解析上下文切换**三合一：

```cpp
// [P2]
bool PT_joined_table_on::contextualize(Parse_context *pc) {
  if (this->contextualize_tabs(pc)) return true;       // ① 先解析左右表引用

  if (push_new_name_resolution_context(pc, this->m_left_table_ref,
                                       this->m_right_table_ref)) {
    this->error(pc, this->m_join_pos);
    return true;
  }

  Query_block *sel = pc->select;
  sel->parsing_place = CTX_ON;                         // ② 置 ON 上下文

  if (super::contextualize(pc) || on->itemize(pc, &on)) return true;  // ③ 表达式层
  if (!on->is_bool_func()) {
    on = make_condition(pc, on);                       // ④ 非布尔表达式 → 包成条件
    if (on == nullptr) return true;
  }
  assert(sel == pc->select);

  add_join_on(this->m_right_table_ref, on);            // ⑤ 落到 Table_ref
  pc->thd->lex->pop_context();
  assert(sel->parsing_place == CTX_ON);
  sel->parsing_place = CTX_NONE;                       // ⑥ 复位
  m_table_ref = pc->select->nest_last_join(pc->thd);
  return m_table_ref == nullptr;
}
```

⇒ `push_new_name_resolution_context` / `pop_context` 与 `parsing_place` 的置位/复位成对出现——**ON 子句里的列只能看见左右两张表**，这个可见性规则就是在 contextualize 期间用"上下文栈 + parsing_place"表达的，而不是在 PT 树结构里表达的。

### 3.7 范畴 7 —— 派生表与子查询：`PT_subquery`

★ 这是"新建 QE + QB、切换 `pc->select`"的完整范式，也是两套递归互相调用的地方。

```cpp
// [P2]
bool PT_subquery::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  LEX *lex = pc->thd->lex;
  if (!lex->expr_allows_subquery || lex->sql_command == SQLCOM_PURGE) {
    error(pc, pos);                                  // ① 语义性语法错误：此处不允许子查询
    return true;
  }

  Query_block *child = lex->new_query(pc->select);   // ② 建子 QE + QB
  if (child == nullptr) return true;

  Parse_context inner_pc(pc->thd, child);            // ③ ★新的环境对象：select 换成 child
  inner_pc.m_stack.push_back(QueryLevel(pc->mem_root, SC_SUBQUERY));

  if (m_is_derived_table) child->linkage = DERIVED_TABLE_TYPE;

  if (qe->contextualize(&inner_pc)) return true;     // ④ 用子环境递归

  query_block = inner_pc.select->master_query_expression()->first_query_block();
  if (inner_pc.finalize_query_expression()) return true;   // ⑤ 子查询自己也要收尾
  inner_pc.m_stack.pop_back();

  lex->pop_context();
  pc->select->n_child_sum_items += child->n_sum_items;     // ⑥ 向父层"上浮"统计信息

  /* A subquery (and all the subsequent query blocks in a UNION) can add
     columns to an outer query block. Reserve space for them. */
  for (Query_block *temp = child; temp != nullptr; temp = temp->next_query_block()) {
    pc->select->select_n_where_fields += temp->select_n_where_fields;
    pc->select->select_n_having_items += temp->select_n_having_items;
  }
  return false;
}
```

- ③ `Parse_context` 是**按值新建**的（`inner_pc`），`pc->select` 不被修改——与 `PT_query_expression` 里"`pc->select = ...` + 还原"是两种风格，本质相同：环境对象界定"当前查询块"。
- ⑥ 子→父的统计量上浮，说明 contextualize 不仅"建对象"，还顺带做**跨层的容量预留**。

派生表把子查询"伪装成一张表"：

```cpp
// [P2] —— 关键抽象：Table_ident(unit) 把一个 Query_expression 包装成"表标识符"
bool PT_derived_table::contextualize(Parse_context *pc) {
  Query_block *outer_query_block = pc->select;
  outer_query_block->parsing_place = CTX_DERIVED;
  if (!m_lateral) {                                        // ① 非 LATERAL：再往外跳一层上下文
    pc->thd->lex->push_context(outer_query_block->context.outer_context);
  }
  if (m_subquery->contextualize(pc)) return true;          // ② 递归整棵子查询
  if (!m_lateral) pc->thd->lex->pop_context();
  outer_query_block->parsing_place = CTX_NONE;

  Query_expression *unit = pc->select->first_inner_query_expression();
  pc->select = outer_query_block;                          // ③ 环境回弹
  Table_ident *ti = new (pc->mem_root) Table_ident(unit);  // ④ 包装成"表"
  if (ti == nullptr) return true;

  m_table_ref = pc->select->add_table_to_list(pc->thd, ti, m_table_alias, 0,
                                              TL_READ, MDL_SHARED_READ);
  if (m_table_ref == nullptr) return true;
  if (m_lateral) {
    m_table_ref->derived_query_expression()->m_lateral_deps = OUTER_REF_TABLE_BIT;
  }
  return pc->select->add_joined_table(m_table_ref);
}
```

① 体现了 LATERAL 的语义差异：非 LATERAL 派生表**不能**看见同一 FROM 里左边的表，所以名字解析上下文要再往外跳一层；LATERAL 则保持在当前 QB。

### 3.8 范畴 8 —— ORDER BY / GROUP BY：`PT_order_expr` 的双重身份

```cpp
// [P1]
class PT_order_expr : public Parse_tree_node, public ORDER {   // ★既是 PT 节点又是 ORDER
 public:
  PT_order_expr(Item *item_arg, enum_order dir) {
    item_initial = item_arg;
    direction = (dir == ORDER_DESC) ? ORDER_DESC : ORDER_ASC;
  }
  bool contextualize(Parse_context *pc) override;
};

class PT_order_list : public Parse_tree_node {
 public:
  SQL_I_List<ORDER> value;
  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (ORDER *o = value.first; o != nullptr; o = o->next) {
      if (static_cast<PT_order_expr *>(o)->contextualize(pc)) return true;  // ① 向下还原类型
    }
    return false;
  }
};
```

- ★ `ORDER` 是 AST 侧的老结构体。让 PT 节点**直接继承它**，就省掉了"PT 节点 → 新建 ORDER → 拷贝字段"的转换，contextualize 结束后 `PT_order_expr` 直接以 `ORDER*` 身份留在 `Query_block::order_list` 里。这是"PT 与 AST 结构复用"的少数例外。
- ① 因此链表元素是 `ORDER*` 但需要调 `contextualize()`，才要向下还原成 PT 类型。

```cpp
// [P2]
bool PT_order_expr::contextualize(Parse_context *pc) {
  return super::contextualize(pc) || item_initial->itemize(pc, &item_initial);
}

bool PT_group::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  Query_block *select = pc->select;
  select->parsing_place = CTX_GROUP_BY;
  if (group_list->contextualize(pc)) return true;
  select->group_list = group_list->value;

  // group by does not have to provide ordering —— 8.0 显式去掉隐含排序
  ORDER *group = select->group_list.first;
  for (; group; group = group->next) group->direction = ORDER_NOT_RELEVANT;  // ★

  select->parsing_place = CTX_NONE;
  switch (olap) {                                        // WITH ROLLUP
    case ROLLUP_TYPE:
      if (select->linkage == GLOBAL_OPTIONS_TYPE) {
        my_error(ER_WRONG_USAGE, MYF(0), "WITH ROLLUP", "global union parameters");
        return true;
      }
      select->olap = ROLLUP_TYPE;
      break;
    ...
  }
  return false;
}
```

★ `ORDER_NOT_RELEVANT` 是 contextualize 期做的**语义规范化**：GROUP BY historically 提供排序，8.0 显式把方向置为"不要求"，把"是否排序"的决定权完全交给 ORDER BY。

### 3.9 范畴 9/10/11 —— LIMIT / WITH / 窗口（要点）

```cpp
// [P2] LIMIT：按 is_offset_first 决定 offset 与 limit 的 itemize 顺序
bool PT_limit_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  if (limit_options.is_offset_first && limit_options.opt_offset != nullptr &&
      limit_options.opt_offset->itemize(pc, &limit_options.opt_offset))
    return true;                                            // ① LIMIT off, n
  if (limit_options.limit->itemize(pc, &limit_options.limit)) return true;
  if (!limit_options.is_offset_first && limit_options.opt_offset != nullptr &&
      limit_options.opt_offset->itemize(pc, &limit_options.opt_offset))
    return true;                                            // ② LIMIT n OFFSET off

  pc->select->select_limit = limit_options.limit;
  pc->select->offset_limit = limit_options.opt_offset;
  pc->thd->lex->set_stmt_unsafe(LEX::BINLOG_STMT_UNSAFE_LIMIT);  // ③ 复制安全性标记
  return false;
}
```

①② 顺序敏感是为了让 `LIMIT ? OFFSET ?`（PS 占位符）的参数顺序与 `Item_param` 编号一致。③ 是"语义侧效应"：带 LIMIT 且无 ORDER BY 的 DML 对复制不安全，解析期就能定下。

```cpp
// [P2] WITH：contextualize 只做一件事——登记（PT 节点被长期持有的少数例外）
bool PT_with_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->select->master_query_expression()->m_with_clause = this;   // ★ 视图打印要用 print()
  return false;
}
```

```cpp
// [P10] 窗口：一个 contextualize 里同时用 contextualize + itemize 的微观样本
bool PT_window::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  if (m_partition_by != nullptr) {
    if (m_partition_by->contextualize(pc)) return true;   // ① PARTITION BY：结构层
  }
  if (m_order_by != nullptr) {
    if (m_order_by->contextualize(pc)) return true;       // ② ORDER BY：复用 PT_order
  }
  if (m_frame != nullptr) {                               // ③ 框架边界：表达式层
    for (auto bound : {m_frame->m_from, m_frame->m_to}) {
      if (bound->m_border_type == WBT_VALUE_PRECEDING ||
          bound->m_border_type == WBT_VALUE_FOLLOWING) {
        auto **bound_i_ptr = bound->border_ptr();
        if ((*bound_i_ptr)->itemize(pc, bound_i_ptr)) return true;
      }
    }
  }
  return false;
}
```

（范畴 12–17：锁子句是"模板方法"——基类 `PT_locking_clause::contextualize()` 做复制安全性标记，纯虚 `set_lock_for_tables()` 下沉给 QB 级/表级子类；DML 值列表只做 itemize 不做结构转换；SET 家族的"AST"不是 `Query_block` 而是 `LEX::var_list`；列属性/分区/表 DDL 三类用各自的 `Context` 实例化。它们共同说明：**PT 节点族覆盖了 MySQL 全部命令类型，"AST"的目标结构随命令而异**。）

### 3.10 ★ `Parse_tree_item`：`PTI_*` 的基类与"变身"契约

```cpp
// [P9]
/**
  Base class for parse-time Item objects

  Parse-time Item objects are placeholders for real Item objects: in some
  cases it is not easy or even possible to decide what exact Item class object
  we need to allocate in the parser. Parse-time Item objects are intended
  to defer real Item object allocation to the contextualization phase (see
  the Item::itemize() function).

  This wrapper class overrides abstract virtual functions of the parent
  class with dummy wrappers to make C++ compiler happy.
*/
class Parse_tree_item : public Item {
 public:
  explicit Parse_tree_item(const POS &pos) : Item(pos) {}

  enum Type type() const override { return INVALID_ITEM; }   // ① 类型无效：不可参与求值
  double val_real() override { assert(0); return 0; }        // ② 求值一律 assert(0)
  longlong val_int() override { assert(0); return 0; }
  String *val_str(String *) override { assert(0); return nullptr; }
  my_decimal *val_decimal(my_decimal *) override { assert(0); return nullptr; }
  bool get_date(MYSQL_TIME *, uint) override { assert(0); return false; }
  bool get_time(MYSQL_TIME *) override { assert(0); return false; }
};
```

★ **为什么需要它？** 有些 token 组合在**语法阶段无法判定**该造哪个 `Item`（例如 `a = b` 里的 `=`；`SYSDATE` 依赖 `sysdate_is_now` 变量）。于是先造一个"占位 Item"，它的**形状像 Item**（能进 `Item*` 列表、能参与递归），但**不可求值**（`type()` 为 `INVALID_ITEM`，所有 `val_*()` `assert(0)`）。在 `itemize()` 期间，占位符把自己**替换**成真 Item。

```cpp
// [P5] —— 三个典型替换
bool PTI_comp_op::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      right->itemize(pc, &right))
    return true;
  *res = (*boolfunc2creator)(false)->create(left, right);   // ① 由函数指针决定真 Item 类
  return *res == nullptr;
}

bool PTI_singlerow_subselect::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || subselect->contextualize(pc)) return true;  // ② 回到结构层！
  *res = new (pc->mem_root) Item_singlerow_subselect(subselect->value());
  pc->select->n_scalar_subqueries++;
  return *res == nullptr;
}
```

⇒ ② 是"两套递归互相调用"的关键证据：表达式层的 `itemize()` 里调用 `subselect->contextualize(pc)`（结构层），再把产出的 `Query_block*` 包成 `Item_singlerow_subselect`。

（36 个 `PTI_*` 的完整清单与各自的变身目标，见 [`05_contextualize.md`](05_contextualize.md) 第九章。）

---

## 四、与 yacc 的对应

（具体 SQL 的逐步产生式见"先有个整体印象"1.2.3；本节聚焦**三个宏**与"自顶向下"的特例。）

### 4.1 三个宏：`NEW_PTN` / `CONTEXTUALIZE` / `ITEMIZE`

```cpp
// [P12]
#define YYP (YYTHD->m_parser_state)
#define YYMEM_ROOT (YYTHD->mem_root)
#define NEW_PTN new(YYMEM_ROOT)          // ① 在语句 MEM_ROOT 上分配；nothrow

/**
  Parse_tree_node::contextualize() function call wrapper
*/
#define CONTEXTUALIZE(x)                                \
  do                                                    \
  {                                                     \
    std::remove_reference<decltype(*x)>::type::context_t pc(YYTHD, Select); \
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->contextualize(&pc)))    \
      MYSQL_YYABORT;                                                    \
  } while(0)

#define CONTEXTUALIZE_VIEW(x)                           \
  do                                                    \
  {                                                     \
    std::remove_reference<decltype(*x)>::type::context_t pc(YYTHD, Select); \
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->contextualize(&pc)))    \
      MYSQL_YYABORT;                                                    \
    if (pc.finalize_query_expression())                                 \
      MYSQL_YYABORT;                                                    \
  } while(0)

/**
  Item::itemize() function call wrapper
*/
#define ITEMIZE(x, y)                                                   \
  do                                                                    \
  {                                                                     \
    Parse_context pc(YYTHD, Select);                                    \
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->itemize(&pc, (y))))     \
      MYSQL_YYABORT;                                                    \
  } while(0)
```

★ **为什么需要宏？** 因为在**归约中途**就要 contextualize 的场景（视图体、表达式子树）必须就地报错并 `YYABORT`，而 `MYSQL_YYABORT` 依赖 yacc 的局部控制流，写成函数调用无法展开。

★ `CONTEXTUALIZE(x)` 里那句 `std::remove_reference<decltype(*x)>::type::context_t` 就是 1.3 里 `typedef Context context_t` 的**唯一使用者**：宏不知道 `x` 的具体类型，但通过 `context_t` 能造出正确类型的环境对象——**模板参数化在此兑现**。`CONTEXTUALIZE_VIEW` 多一步 `finalize_query_expression()`，因为视图体是一棵完整的查询表达式。

### 4.2 "自顶向下插左叶子"：JOIN 归约的特例

```yacc
// [P12]
        | table_reference inner_join_type table_reference
          %prec CONDITIONLESS_JOIN
          {
            auto this_cross_join= NEW_PTN PT_cross_join($1, @2, $2, NULL);
            if ($3 == NULL)
              MYSQL_YYABORT; // OOM
            $$= $3->add_cross_join(this_cross_join);   // ★ 不是自底向上！
          }
```

⇒ `t1 JOIN t2 JOIN t3` 归约时先拿到**右子树**，必须"自顶向下"把新的 cross join 插到**最左叶子**位置。这是 PT 树构造中少见的非自底向上操作，由 `PT_table_reference::add_cross_join()` 完成（`cj->add_rhs(this); return cj;`）。

### 4.3 `NEW_PTN` 之后的 OOM 检查是硬性约定

```yacc
// [P12]
select_item_list:
          select_item_list ',' select_item
          {
            if ($1 == NULL || $1->push_back($3))       // ★ 检查 NULL
              MYSQL_YYABORT;
            $$= $1;
          }
        | '*'
          {
            Item *item = NEW_PTN Item_asterisk(@$, nullptr, nullptr);   // ★ @$ = 本产生式位置
            $$ = NEW_PTN PT_select_item_list;
            if ($$ == nullptr || item == nullptr || $$->push_back(item))
              MYSQL_YYABORT;
          }
        ;
```

⇒ `NEW_PTN` 展开是 `new (YYTHD->mem_root)`，配合 `operator new(size, MEM_ROOT*, nothrow)`，**OOM 返回 `nullptr` 不抛异常**——所以每个 `NEW_PTN` 之后都要检查。`@$` 是 bison 的位置（`YYLTYPE`），被塞进节点供 `error(pc, pos)` 使用。

---

## 五、复杂类深剖：`PT_query_specification`

它是 PT 体系里最核心、最复杂的填充函数——一条 SELECT 的全部组成都在这里。

### 5.1 类定义（一条 SELECT 的全部组成）

```cpp
// [P1]
class PT_query_primary : public PT_query_expression_body {};   // 空壳，只为在类型体系上分一层

class PT_query_specification : public PT_query_primary {
  typedef PT_query_primary super;

  PT_hint_list *opt_hints;                       // ① SELECT 后 /*+ ... */ 的 hint 列表
  Query_options options;                         // ② ALL / DISTINCT / STRAIGHT_JOIN / HIGH_PRIORITY ...
  PT_item_list *item_list;                       // ③ SELECT 列表（实际对象多为 PT_select_item_list）
  PT_into_destination *opt_into1;                // ④ 前置 INTO（SELECT ... INTO OUTFILE ... FROM ...）
  const bool m_is_from_clause_implicit;          // ⑤ 是否"隐式 FROM"（SELECT 1; 没有 FROM）
  Mem_root_array_YY<PT_table_reference *> from_clause;  // ⑥ FROM 子句（空数组表示 DUAL）
  Item *opt_where_clause;                        // ⑦ WHERE（归约产物是 PTI_where，类型却是裸 Item*）
  PT_group *opt_group_clause;                    // ⑧ GROUP BY ... [WITH ROLLUP]
  Item *opt_having_clause;                       // ⑨ HAVING（归约产物是 PTI_having）
  PT_window_list *opt_window_clause;             // ⑩ WINDOW w AS (...)

 public:
  PT_query_specification(
      PT_hint_list *opt_hints_arg, const Query_options &options_arg,
      PT_item_list *item_list_arg, PT_into_destination *opt_into1_arg,
      const Mem_root_array_YY<PT_table_reference *> &from_clause_arg,
      Item *opt_where_clause_arg, PT_group *opt_group_clause_arg,
      Item *opt_having_clause_arg, PT_window_list *opt_window_clause_arg,
      bool implicit_from_clause)
      : opt_hints(opt_hints_arg), options(options_arg), item_list(item_list_arg),
        opt_into1(opt_into1_arg),
        m_is_from_clause_implicit{implicit_from_clause},
        from_clause(from_clause_arg), opt_where_clause(opt_where_clause_arg),
        opt_group_clause(opt_group_clause_arg),
        opt_having_clause(opt_having_clause_arg),
        opt_window_clause(opt_window_clause_arg) {
    assert(implicit_from_clause ? from_clause.empty() : true);   // 隐式 FROM ⇒ 一定没有表
  }
  /* 另有简版/最简版构造：无 hint / 无 INTO / 只有 select 列表（DO 语句等内部场景）*/

  bool contextualize(Parse_context *pc) override;       // ★ 本节主角

  bool is_set_operation() const override { return false; }        // QS 永远不是 UNION/EXCEPT/...
  bool can_absorb_order_and_limit(bool, bool) const override { return true; }  // QS 总能"吃掉"尾部 ORDER/LIMIT
};
```

配套的 `Query_options`（`[P12]`）与取值：

```cpp
struct Query_options {
  ulonglong query_spec_options;   // 位掩码：SELECT_DISTINCT / SELECT_STRAIGHT_JOIN / ...
  bool save_to(Parse_context *);  // 落到 Query_block::base_options
};
```

```yacc
// [P6]
query_spec_option:
          STRAIGHT_JOIN       { $$= SELECT_STRAIGHT_JOIN; }
        | HIGH_PRIORITY       { $$= SELECT_HIGH_PRIORITY; }
        | DISTINCT            { $$= SELECT_DISTINCT; }
        | SQL_SMALL_RESULT    { $$= SELECT_SMALL_RESULT; }
        | SQL_BIG_RESULT      { $$= SELECT_BIG_RESULT; }
        | SQL_BUFFER_RESULT   { $$= OPTION_BUFFER_RESULT; }
        | SQL_CALC_FOUND_ROWS { ... $$= OPTION_FOUND_ROWS; }
        | ALL                 { $$= SELECT_ALL; }
        ;
```

### 5.2 `contextualize()` 完整代码 + 逐行注释

```cpp
// [P2]
bool PT_query_specification::contextualize(Parse_context *pc) {
  // ① 基类：只做栈溢出检查 + NDEBUG 下置 contextualized 标记（防重复 contextualize）
  if (super::contextualize(pc)) return true;

  // ② 进入新的"查询层"。SC_QUERY_SPECIFICATION 告诉父层"我是个 query block"，
  //    供 UNION/EXCEPT 组装 Query_term 树。
  pc->m_stack.push_back(QueryLevel(pc->mem_root, SC_QUERY_SPECIFICATION));

  // ③ 上下文切换到 SELECT 列表。此后 itemize() 出的 Item 才知道自己身处 CTX_SELECT_LIST：
  //    影响 select_n_having_items++、子查询语义、以及 PTI_simple_ident_q_3d 走 Item_field 而非 Item_ref。
  pc->select->parsing_place = CTX_SELECT_LIST;

  // ④ HIGH_PRIORITY 是"语法选项但要落到锁类型"的特例：改写 Yacc_state 的加锁/MDL 级别。
  //    ★ 必须在 from_clause 之前做完，因为 add_table_to_list() 会读 yyps->m_lock_type / m_mdl_type。
  if (options.query_spec_options & SELECT_HIGH_PRIORITY) {
    Yacc_state *yyps = &pc->thd->m_parser_state->m_yacc;
    yyps->m_lock_type = TL_READ_HIGH_PRIORITY;
    yyps->m_mdl_type = MDL_SHARED_READ;
  }

  // ⑤ 其余选项（ALL/DISTINCT/STRAIGHT_JOIN/SQL_BIG_RESULT...）作为位掩码交给 Query_block
  if (options.save_to(pc)) return true;

  // ⑥ 先做 SELECT 列表。
  //    ★ 顺序原因：MySQL 允许 SELECT 列表里引用后面 FROM 才出现的别名；
  //      且 fields 里的 Item 需要 select_n_having_items 计数（CTX_SELECT_LIST 下才 +1）。
  //      副作用：PT_item_list::contextualize() 就地 itemize 每一项，PTI_* 在这里变身。
  if (item_list->contextualize(pc)) return true;   // → Query_block::fields = value

  // ⑦ 立刻复位。断言"复位的是同一个 select"，防止子节点偷偷切走了 pc->select
  assert(pc->select->parsing_place == CTX_SELECT_LIST);
  pc->select->parsing_place = CTX_NONE;

  // ⑧ 前置 INTO（SELECT ... INTO OUTFILE ... FROM ...）。nullptr 时 contextualize_safe 直接返回 false
  if (contextualize_safe(pc, opt_into1)) return true;

  // ⑨ FROM 子句。★ 必须在 WHERE 之前：WHERE 里的 t.c 要依靠 FROM 建好的名字解析上下文
  if (!from_clause.empty()) {
    if (contextualize_array(pc, &from_clause)) return true;
    pc->select->context.table_list =
        pc->select->context.first_name_resolution_table =
            pc->select->get_table_list();   // 名字解析起点指向第一张表
  }

  // ⑩ WHERE / GROUP BY / HAVING。依赖链：
  //    WHERE → FROM（表名解析）；GROUP BY → SELECT 列表（别名分组）；HAVING → GROUP BY + SELECT 列表
  if (itemize_safe(pc, &opt_where_clause) ||
      contextualize_safe(pc, opt_group_clause) ||
      itemize_safe(pc, &opt_having_clause))
    return true;

  // ⑪ 把已 itemize 完的条件写进 Query_block 的两个槽位
  pc->select->set_where_cond(opt_where_clause);
  pc->select->set_having_cond(opt_having_clause);

  /*
    Window clause is resolved under CTX_SELECT_LIST and not
    under CTX_WINDOW. Reasons being:
    1. Window functions are part of select list and the
       resolution of window definition happens along with
       window functions.
    ...（5 条理由，见源码）
  */
  // ⑫ 窗口子句故意"借用" CTX_SELECT_LIST 而不是 CTX_WINDOW —— 阅读时最容易踩的坑
  pc->select->parsing_place = CTX_SELECT_LIST;
  if (contextualize_safe(pc, opt_window_clause)) return true;
  pc->select->parsing_place = CTX_NONE;

  // ⑬ 收尾一：把自己的 Query_block 交给父层（父层之后从 elts 取出拼 Query_term 树）
  QueryLevel ql = pc->m_stack.back();
  pc->m_stack.pop_back();
  pc->m_stack.back().m_elts.push_back(pc->select);

  // ⑭ 收尾二：最后才处理 hint。★ 因为 hint 里可能引用查询块名（QB_NAME）与表名，
  //    必须等 FROM / WHERE 都建好、Opt_hints_qb 能定位到正确的 Query_block 之后
  return (opt_hints != nullptr ? opt_hints->contextualize(pc) : false);
}
```

### 5.3 调用时序图

```
进入  pc->select = 顶层 Query_block；m_stack = [SC_TOP]
  │
  ├─(1)  super::contextualize            栈检查 + 防重入
  ├─(2)  push SC_QUERY_SPECIFICATION     开一层"查询层"
  ├─(3)  parsing_place = CTX_SELECT_LIST
  ├─(4)  HIGH_PRIORITY → yyps 锁类型      ← 必须在 FROM 之前
  ├─(5)  options.save_to()                → QB::base_options
  ├─(6)  item_list->contextualize()       → QB::fields      （PTI_* 变身；select_n_having_items++）
  ├─(7)  parsing_place = CTX_NONE         ← 复位 + assert 同一 select
  ├─(8)  opt_into1
  ├─(9)  from_clause[]                   → QB::m_table_list / context.{table_list,first_name_resolution_table}
  ├─(10) opt_where_clause  (itemize)      → 内部切 CTX_WHERE 再复位
  │      opt_group_clause
  │      opt_having_clause (itemize)      → 内部切 CTX_HAVING 再复位
  ├─(11) set_where_cond() / set_having_cond()
  ├─(12) CTX_SELECT_LIST → opt_window_clause → CTX_NONE
  ├─(13) pop 自己的 QueryLevel；父层 elts.push_back(pc->select)
  └─(14) opt_hints->contextualize()      → Opt_hints_qb（放最后，等 QB 定型）
退出  parsing_place = CTX_NONE；m_stack = [SC_TOP(+父层)]
```

### 5.4 为什么它复杂

1. **成员多、槽位多**：10 个成员对应 `Query_block` 上至少 8 类槽位，而**每个槽位的填充方式都不一样**——有的是 `contextualize`（纯语法节点），有的是 `itemize`（`Item` 占位符，会**就地替换指针**）。混淆这两者会导致节点被 contextualize 两次或漏掉变身。
2. **顺序有硬依赖**：`FROM` 必须早于 `WHERE/GROUP/HAVING`（名字解析上下文）；`SELECT 列表`必须早于 `GROUP BY/HAVING`（别名可见性）；`HIGH_PRIORITY` 必须早于 `FROM`（锁类型被 `add_table_to_list()` 读取）；`hint` 必须最后（要定位 `Query_block`）。任何一步挪位都会破坏语义。
3. **作用域切换多**：`parsing_place` 在**同一个函数里被写了 4 次**，而 `WHERE`/`HAVING`/`GROUP BY`/`ON` 的切换又发生在各自节点的 `itemize()` 内部。这是一个"**跨节点的隐式状态机**"——函数签名里看不见，只能靠 `assert(pc->select->parsing_place == ...)` 兜底。
4. **跨层协作**：它不直接造 `Query_expression`/`Query_term`，而是通过 `pc->m_stack` 把 `Query_block` "上抛"给父层，由 `PT_query_expression::contextualize` 决定是 `QT_QUERY_BLOCK` 还是包一层 `Query_term_unary`。正确性依赖**父子的隐式契约**（push/pop 必须配对）。
5. **历史包袱**：`INTO` 有两个位置（前置 `opt_into1` 与尾部 `select_stmt_with_into`），于是 `query_specification` 有两条几乎重复的产生式；`m_is_from_clause_implicit` 与 `has_trailing_into_clause()` 都是为了消解这份歧义。
6. **断言密集**：`assert(pc->select->parsing_place == CTX_SELECT_LIST)`、`assert(implicit_from_clause ? from_clause.empty() : true)`、`assert(!contextualized)`——说明不变量只能靠运行时断言维护，也是它"难改"的直接证据。

### 5.5 特有机制

**① `opt_hints`（`/*+ ... */`）**：hint 不走独立产生式——词法阶段 `consume_optimizer_hints()` 把 `PT_hint_list*` 塞进 `SELECT_SYM` 的 `yylval`，靠 `%type <lexer.optimizer_hints> SELECT_SYM ...` 让 `$1` 带上它。所以 `PT_query_specification` 的第一个形参名是 `opt_hints` 而 yacc 注释写的是 `// SELECT_SYM`。

**② 选项不是独立成员**：`STRAIGHT_JOIN` / `DISTINCT` / `SQL_BIG_RESULT` 等都压进 `Query_options::query_spec_options` 这一个 `ulonglong` 位掩码，再由 `save_to()` 一次性写入 `Query_block::base_options`。

**③ `HIGH_PRIORITY` 的"跨界"**：它是 `query_spec_option` 之一，却不能只靠位掩码——它影响的是**表级锁类型**，只能通过改写 `Yacc_state`（`m_lock_type` / `m_mdl_type`）传递给后续的 `add_table_to_list()`。这是"选项不是纯位掩码"的唯一例外。

**④ `m_is_from_clause_implicit`**：`@N.raw.is_empty()` 判断 `FROM` 是否真的出现过，决定 `SELECT 1;` 这类语句走哪条查询块建立路径。

**⑤ 窗口子句的"反直觉"上下文**：`WINDOW` 子句明明有 `CTX_WINDOW` 却偏用 `CTX_SELECT_LIST`（5.2 步骤⑫的 5 条理由）——这是阅读这段代码时最容易踩的坑。

### 5.6 与 `PT_query_expression` 的责任边界（重要）

```
PT_query_expression            PT_query_specification
├─ m_with_clause  (WITH ...)   ├─ opt_hints
├─ m_body         (query body) ├─ options      (ALL/DISTINCT/STRAIGHT_JOIN/...)
├─ m_order        (ORDER BY)   ├─ item_list    (SELECT list)
└─ m_limit        (LIMIT)      ├─ opt_into1    (前置 INTO)
                               ├─ from_clause
                               ├─ opt_where_clause
                               ├─ opt_group_clause
                               ├─ opt_having_clause
                               └─ opt_window_clause
```

★ **`ORDER BY` / `LIMIT` 不在 `PT_query_specification` 上**，而在外层的 `PT_query_expression` 上。原因是 SQL 标准里它们属于**查询表达式**（可作用于 `UNION` 整体），而不是单个查询块；`can_absorb_order_and_limit()` 这套"下推/吸收"规则正是用来处理 `(SELECT ...) ORDER BY ...` 这类歧义的。

### 5.7 同名 / 版本陷阱（8.0.39 实测）

| 常见说法 | 8.0.39 的实际情况 |
|---|---|
| `PT_query_specification::m_select_item_list` | 成员名是 `item_list`（`PT_item_list *`） |
| `m_where_clause` / `m_having_clause` | 成员名是 `opt_where_clause` / `opt_having_clause` |
| 类里有 `m_order` / `m_limit` | **没有**；ORDER BY / LIMIT 在 `PT_query_expression` 上 |
| `PTI_where` 在 `parse_tree_nodes.h` | 在 **`sql/parse_tree_items.h`**（`class PTI_where final : public PTI_context`） |
| `Pushdown_select_init` 管理作用域 | 全仓库 0 命中；8.0.39 用 `Parse_context::m_stack` + `QueryLevel` + `Surrounding_context` |
| 收尾调 `select->set_braces(...)` | 全仓库 `set_braces` **0 命中**；收尾是 pop + `m_elts.push_back(pc->select)` + `opt_hints->contextualize()` |
| `ABS` 是 yacc 关键字函数 | 它是 `IDENT_sys`，走 `function_call_generic` → `PTI_function_call_generic_ident_sys`，`Item_func_abs` 由 `item_create.cc` 的 `SQL_FN` 表在 `itemize()` 里创建 |
| `>` 的 token 是 `GT` | 是 **`GT_SYM`**（`>=` 才是 `GE`，没有 `_SYM`） |
| `FROM` / `WHERE` 的 token 是 `FROM_SYM` / `WHERE_SYM` | 就是 `FROM` / `WHERE`（无后缀） |

---

## 六、命名规律与量化证据

### 6.1 命名与"身份"

| 命名 | 基类（代码证据） | 语义 |
|---|---|---|
| `PT_xxx` | `: public Parse_tree_node`（`= Parse_tree_node_tmpl<Parse_context>`） | 纯语法节点，走 `contextualize()` |
| `PTI_xxx` | `: public Parse_tree_item`，而 `Parse_tree_item : public Item : public Parse_tree_node` | 解析期占位 Item，走 `itemize()`，用完即被替换 |
| `PT_xxx_base` | 含纯虚函数 | 抽象中间层，只为归类 |
| `PT_xxx final` | `class PT_query_expression final : ...` | 继承终点，编译期禁止再派生 |
| `typedef Parse_tree_node super;` | 每个类体内首行 | 统一"调基类实现"的写法 |
| `opt_xxx` 成员 | — | 语法中的可选项，可能为 `nullptr` |
| `PT_order_expr : public Parse_tree_node, public ORDER` | `[P1]` | 双重身份：既是 PT 节点又是 AST 结构 |

"身份"还有两个运行时维度：

1. **`enum_parsing_context parsing_place`**（`CTX_*`）：标识"当前正在解析哪个子句"，由 `PT_query_specification` / `PT_group` / `PT_order` / `PT_derived_table` / `PT_joined_table_on` 置位与复位，被 `Item::itemize()` 与后续名字解析消费。
2. **`Surrounding_context` + `QueryLevel`**：标识"当前在查询表达式的哪一层"，用于 `Query_term` 树构造与 UNION ALL 流式判定。

### 6.2 量化：两套递归的规模

| 指标 | 命中 | 说明 |
|---|---|---|
| `PT_*` 类定义（`^class PT_`） | **449** 处 | 其中 `parser_yystype.h` 的 **67** 处是前向声明，实际定义约 **382** 个 |
| `PTI_*` 类定义（`^class PTI_`） | **36** 处 | + 1 处前向声明 |
| `contextualize()` 声明 | **164** 处 | 减去基类 1 处 ⇒ override **≈163** |
| `itemize()` 声明 | **92** 处 | 减去基类 ⇒ override **≈91**，其中 **33** 次集中在 `PTI_*` |
| `Context` 实例化种类 | **5** | `Parse_context` / `Table_ddl_parse_context` / `Column_parse_context` / `Partition_parse_context` / `Alter_tablespace_parse_context` |

⇒ `contextualize` 的 override（≈163）约为 `itemize`（≈91）的 **1.8 倍**，与"结构层节点比表达式层节点多"的直觉一致；而 `itemize` 的 override 里 **33 次集中在 `PTI_*`**——即"占位符替换"是表达式层递归中最活跃的一支。

### 6.3 设计要点小结

```
  ┌─────────────────────── Parse_tree_root（唯一接口 make_cmd）───────────────┐
  │  PT_select_stmt  PT_insert  PT_update  PT_delete  PT_create_table_stmt ... │
  └───────────────────────────────────────────────────────────────────────────┘
                                  │ make_cmd() 内部
                                  ▼
  ┌────── Parse_tree_node_tmpl<Context>（contextualize + error/errorf）───────┐
  │  Context = Parse_context             → Parse_tree_node      （≈90% 节点） │
  │          = Table_ddl_parse_context   → Table_ddl_node                     │
  │          = Column_parse_context      → PT_column_attr_base                │
  │          = Partition_parse_context   → Parse_tree_part_node               │
  │          = Alter_tablespace_parse_context                                 │
  └───────────────────────────────────────────────────────────────────────────┘
        ▲                                                    ▲
        │ : public Parse_tree_node                           │
   Item ─────────────► Parse_tree_item（PTI_* ×36）           │
        │ itemize() 里 super::contextualize(pc)              │
        └────────── 两套递归在 Item 处焊接 ───────────────────┘
```

1. **一个接口 + 一个模板**撑起全部 380+ 节点类；
2. **`contextualized` 只在 debug 存在**——开发期契约，不是运行期状态；
3. **`check_stack_overrun` 必须在每层做**——递归深度由不可信的 SQL 文本决定，bison 的 `YYMAXDEPTH` 只保护归约栈；
4. **两套递归正交但互调**，焊接点是 `Item : public Parse_tree_node`；
5. **PT 节点一次性消费 + MEM_ROOT 整体回收**，少数例外（`PT_with_clause`）只因为需要 `print()` 重建视图文本。

---

## 七、`Parse_context` 与 `itemize()` 深度剖析

> 前六章讲的是"有哪些节点、各自做什么"；本章讲**驱动它们的那套环境机制**——`Parse_context` 如何承载状态、`pc->select` 何时被切换、`m_stack` 的 push/pop 契约、`Query_term` 树如何组装，以及 `itemize()` 的五种变身模式。
>
> 本节源码坐标：`[P1] sql/parse_tree_nodes.h` · `[P2] sql/parse_tree_nodes.cc` · `[P3] sql/parse_tree_node_base.h` · `[P4] sql/parse_tree_node_base.cc` · `[P5] sql/parse_tree_items.cc` · `[P6] sql/parse_tree_items.h` · `[P7] sql/item.cc` · `[P8] sql/sql_lex.h` · `[P9] sql/parse_tree_helpers.h` · `[P10] sql/sql_lex.cc` · `[P11] sql/item.h` · `[P13] sql/query_term.h` · `[P14] sql/query_term.cc` · `[P15] sql/sql_yacc.yy` · `[P16] sql/parse_tree_column_attrs.h` · `[P17] sql/parse_tree_partitions.h`

### 7.1 `Parse_context`：contextualize 阶段的"环境数据"

#### 7.1.1 完整定义逐字段剖析

```cpp
// [P3]
struct Parse_context {
  THD *const thd;                      ///< Current thread handler
  MEM_ROOT *mem_root;                  ///< Current MEM_ROOT
  Query_block *select;                 ///< Current Query_block object
  mem_root_deque<QueryLevel> m_stack;  ///< Aids query term tree construction
  bool finalize_query_expression();
  Parse_context(THD *thd, Query_block *sl);
  bool is_top_level_union_all(Surrounding_context op);
};
```

**★ `THD *const thd` 为什么是 `const`？**

`const` 修饰的是**指针本身**（顶层 const），不是 `THD` 对象。语义是"一旦构造，本 context 终身绑定到该线程"，任何 `pc->thd = ...` 都是编译错误。三个理由：

1. `Parse_context` 在栈上被大量**按值创建**（见 `PT_subquery` 的 `Parse_context inner_pc(pc->thd, child);`）。`const` 让编译器保证子 context 不可能偷偷"改绑"到别的线程。
2. `thd` 是**跨全树共享的只读入口**——所有节点通过它拿 `pc->thd->lex`、`thd->mem_root`、报错。它不随递归变化，因此不该是可写的"游标"。
3. 代价是 `Parse_context` 的**拷贝赋值运算符被隐式删除**——正合设计意图：context 只能"新建"，不能"改写"。

**★ 为什么 `select` 是唯一"可变"字段？**

因为四个字段里只有它是**游标**：

| 字段 | 性质 | 随递归变化 | 说明 |
|---|---|---|---|
| `thd` | 环境常量 | 否 | 语法 + const 双重锁定 |
| `mem_root` | 环境常量 | 否 | 约定上从不改写；它是"分配器"不是状态 |
| **`select`** | **游标** | **是** | 指向"当前正在被填充的那个 `Query_block`" |
| `m_stack` | 栈状态 | 是（push/pop） | 容器内容增减，不是"游标重定向" |

⇒ 只有 `select` 的语义是"我现在的落笔点是哪一个 `Query_block`"。所以：进入 `PT_query_specification` 时所有 `add_table_to_list()` / `set_where_cond()` 都落在 `pc->select`；遇到 UNION 第二分支必须重定向，否则 `SELECT 2` 的字段会落到 `SELECT 1` 的块里。

```cpp
// [P4]
Parse_context::Parse_context(THD *thd_arg, Query_block *sl_arg)
    : thd(thd_arg), mem_root(thd->mem_root), select(sl_arg),
      m_stack(thd->mem_root) {
  m_stack.push_back(QueryLevel(thd->mem_root, SC_TOP));   // 初始：栈底永远是 SC_TOP
}
```

注意：`m_stack` 本身也用同一个 MEM_ROOT 分配，**整个 context 零 malloc、随语句 MEM_ROOT 一次性回收**；构造完 `m_stack.size() == 1`，这是 `finalize_query_expression()` 里 `assert(ql.m_elts.size() == 1)` 成立的前提。

#### 7.1.2 ★ `pc->select` 被切换的全部场景

| 场景 | 写法 | 是否复原 |
|---|---|---|
| ① `PT_query_expression`（UNARY / set-op 分支） | `pc->select = ex->query_block();` | 是（`orig_query_block`） |
| ② `PT_query_expression::contextualize_order_and_limit` | `pc->select = nullptr;` | 是 |
| ③ `PT_set_operation::contextualize_setop` | `pc->select = lex->new_set_operation_query(pc->select);` | **否**（永久前进到右分支） |
| ④ `PT_derived_table` | `pc->select = outer_query_block;` | 是（防御式） |
| ⑤ `PT_start_option_value_list_*`（SET 语句） | `pc->select = pc->thd->lex->query_block;` | 否（重同步） |

##### 场景 ①：切到"后处理块"（保存-复原三连）

```cpp
// [P2] PT_query_expression::contextualize 的 QT_UNARY 分支（节选）
      Query_block *orig_query_block = pc->select;          // ★ 保存
      LEX *lex = pc->thd->lex;
      pc->select = ex->query_block();                      // ★ 切换（后处理块）
      lex->push_context(&pc->select->context);
      assert(pc->select->parsing_place == CTX_NONE);       // ★ 兜底：必须是"干净"的块
      if (contextualize_safe(pc, m_order, m_limit)) return true;
      lex->pop_context();
      pc->select = orig_query_block;                       // ★ 复原
```

要点：`pc->select` 的切换**必须与 `lex->push_context()/pop_context()` 成对出现**。前者是"Item 登记落点"，后者是**名字解析上下文栈**（`LEX::context_stack`），与 `pc->m_stack` 是**两个完全不同的栈**。

##### 场景 ②：`pc->select = nullptr`（借"空 QB"上下文）

```cpp
// [P2]
bool PT_query_expression::contextualize_order_and_limit(Parse_context *pc) {
  if (m_order == nullptr && m_limit == nullptr) return false;
  if (m_body->can_absorb_order_and_limit(m_order != nullptr, m_limit != nullptr)) {
    if (contextualize_safe(pc, m_order, m_limit)) return true;   // ① 就地消化
  } else {
    LEX *lex = pc->thd->lex;
    Query_block *orig_query_block = pc->select;
    pc->select = nullptr;                                        // ★ 置空！
    lex->push_context(&pc->select->context);
    bool res = contextualize_safe(pc, m_order, m_limit);
    lex->pop_context();
    pc->select = orig_query_block;
    if (res) return true;
  }
  return false;
}
```

⇒ 置空的目的：`ORDER BY`/`LIMIT` 的 Item 此时**不属于任何 `Query_block`**（后处理块还没建），`pc->select == nullptr` 让 `Item::itemize()` 里的 `if (pc->select)` 自然跳过 `select_n_having_items++` 计数——这正是基类 itemize 里那段注释 *"Item constructor can be called during execution other then SQL_COM command => we should check pc->select on zero"* 的由来。

##### 场景 ③：`PT_set_operation` 单向前进（不复原）

```cpp
// [P2]
bool PT_set_operation::contextualize_setop(Parse_context *pc,
                                           Query_term_type setop_type,
                                           Surrounding_context context) {
  pc->m_stack.push_back(QueryLevel(pc->mem_root, context));
  if (super::contextualize(pc)) return true;
  if (m_lhs->contextualize(pc)) return true;    // 左分支：仍用旧 pc->select

  // ★★ 创建右分支的 Query_block 并把游标永久推到它上面（qb1 UNION qb2 的 qb2）
  pc->select = pc->thd->lex->new_set_operation_query(pc->select);
  if (pc->select == nullptr || m_rhs->contextualize(pc)) return true;
  pc->thd->lex->pop_context();   // 与 new_set_operation_query() 内部的 set_context() 配对

  QueryLevel ql = pc->m_stack.back();
  pc->m_stack.pop_back();

  Query_term_set_op *setop = nullptr;
  switch (setop_type) {   // QT_UNION / QT_EXCEPT / QT_INTERSECT
    case QT_UNION: setop = new (pc->mem_root) Query_term_union(pc->mem_root); break;
    ...
  }
  merge_descendants(pc, setop, ql);   // 折叠同类型嵌套（含 is_top_level_union_all 判断）
  setop->label_children();
  Query_expression *qe = pc->select->master_query_expression();
  if (setop->set_block(qe->create_post_processing_block(setop))) return true;
  pc->m_stack.back().m_elts.push_back(setop);   // ★ 把自己交给父层
  return false;
}
```

⇒ `A UNION B UNION C` 时游标 qb1 → qb2 → qb3，**永不回头**（回头由 `PT_query_expression` 的保存-复原负责）。

##### 场景 ④：`PT_subquery` —— 按值新建 `inner_pc`（唯一的"真隔离"）

```cpp
// [P2]（节选）
  Query_block *child = lex->new_query(pc->select);      // 建子 QE + QB
  // ★★ 按值新建：thd/mem_root 复制自父，select 换成 child，
  //    m_stack 从零开始（构造只压了 SC_TOP），再压一层 SC_SUBQUERY
  Parse_context inner_pc(pc->thd, child);
  inner_pc.m_stack.push_back(QueryLevel(pc->mem_root, SC_SUBQUERY));
  ...
  if (qe->contextualize(&inner_pc)) return true;        // 子树全用 inner_pc
  if (inner_pc.finalize_query_expression()) return true; // 子上下文自己终结
  inner_pc.m_stack.pop_back();
  assert(inner_pc.m_stack.size() == 0);                 // ★ push/pop 配对的硬兜底
```

★ 这是全套机制里**唯一真正隔离**的切换：父 `pc` 完全没被修改，子树的 `m_stack` 是全新的一摞 `[SC_TOP, SC_SUBQUERY]`。两个后果：

- 子查询内部的 UNION/括号嵌套**不可能污染外层 `m_stack`**；
- `is_top_level_union_all()` 扫到 `SC_SUBQUERY` 立刻返回 false——子 context 天然自带这一层，正是为了让"子查询里的 UNION ALL"不被误判为可流式顶层 UNION ALL。

##### 时序图（子查询不破坏父落点）

```
make_cmd:  Parse_context pc(thd, qb1)      m_stack = [ SC_TOP ]
  │
  ├ PT_query_specification::contextualize(pc)
  │   push SC_QUERY_SPECIFICATION ; parsing_place = CTX_SELECT_LIST
  │   pc->select == qb1            <-- 落点 A
  │
  ├ PTI_context(WHERE)::itemize     parsing_place = CTX_WHERE
  │   └ PTI_singlerow_subselect::itemize
  │       └ PT_subquery::contextualize(pc)
  │           child = lex->new_query(pc->select)      // qb2
  │           Parse_context inner_pc(thd, child)      // ★ 按值新建
  │           inner_pc.m_stack = [ SC_TOP, SC_SUBQUERY ]
  │           qe->contextualize(&inner_pc)            // 子树全用 inner_pc
  │           inner_pc.finalize_query_expression() ; pop -> size 0
  │           lex->pop_context()
  │       *res = new Item_singlerow_subselect(...)    // 回到父 pc
  │   pc->select == qb1            <-- ★ 落点 A 未被破坏
  │
  └ pc.finalize_query_expression()
```

#### 7.1.3 ★ `QueryLevel` 与 `m_stack` 的 push/pop 契约

```cpp
// [P3]
enum Surrounding_context {          // "语法上我在谁里面"
  SC_TOP,                      // 最顶层：整条语句的"根容器"层
  SC_QUERY_SPECIFICATION,      // 一个 SELECT 块（或 VALUES / 显式表）
  SC_TABLE_VALUE_CONSTRUCTOR,  // VALUES ROW(...) 行构造器
  SC_QUERY_EXPRESSION,         // 一对括号（可带 ORDER BY / LIMIT）形成的层
  SC_SUBQUERY,                 // 子查询边界（PT_subquery 的 inner_pc 压入）
  SC_UNION_DISTINCT, SC_UNION_ALL,
  SC_INTERSECT_DISTINCT, SC_INTERSECT_ALL,
  SC_EXCEPT_DISTINCT, SC_EXCEPT_ALL
};

struct QueryLevel {
  Surrounding_context m_type;
  /**
     elts 在 contextualize 过程中充当**临时容器**，用于：
       收集子节点  子节点把自己 push 到父节点的 elts
       延迟组合    收集完再决定如何组合成父节点
       支持多路    UNION/INTERSECT/EXCEPT 可能有多个子节点（A UNION B UNION C）
       树构建      最终从 elts 取出节点，构建 Query_term 树
  */
  mem_root_deque<Query_term *> m_elts;
  bool m_has_order{false};
};
```

**契约（父子约定）**：

1. **谁 push**：每个"会产生一个 `Query_term` 的语法层"在进入时 push 一层自己的 `QueryLevel`——`PT_query_expression`→`SC_QUERY_EXPRESSION`（`has_order = m_order != nullptr`）、`PT_query_specification`→`SC_QUERY_SPECIFICATION`、`PT_union/except/intersect`→`SC_UNION_*` 等、`PT_subquery`→在 `inner_pc` 上 push `SC_SUBQUERY`。
2. **谁 push 到 elts**：**子节点结束时**把自己的成果 push 到 `m_stack.back().m_elts`（父层的临时容器）。`PT_query_specification` 塞的是 `pc->select`（`Query_block` 本身就是 `Query_term`）；`PT_set_operation` 塞 `setop`；`PT_query_expression` 塞 `expr`。
3. **谁 pop**：push 的那一方负责 pop，且**先把 elts 取走再 pop**（`QueryLevel ql = back(); pop_back();`）。`ql` 是按值拷贝的，pop 后仍可安全使用。
4. **谁消费**：父节点消费子层 `ql.m_elts`（`PT_set_operation::merge_descendants` 把它们折进 `setop->m_children`），再把自己的 term 交给更上一层。
5. **配对失衡会怎样**：少 pop → `finalize_query_expression()` 取到的不是 `SC_TOP`，`assert(ql.m_elts.size() == 1)` 极可能炸（release 下装错根，随后 `validate_structure()` 崩或执行计划错乱）；多 pop → `pop_back()` 越界/空栈 UB。

#### 7.1.4 ★ `Query_term` 树的组装过程

```cpp
// [P13]
enum Query_term_type {
  QT_QUERY_BLOCK,   // Query specification / table value constructor / explicit table
  QT_UNARY,         // 带 ORDER BY 和/或 LIMIT 的括号层（若无二者则被 collapse）
  QT_INTERSECT, QT_EXCEPT, QT_UNION   // 集合运算，节点是 N 元的
};
```

**创建时机一览**：

| 节点 | 何时被创建 |
|---|---|
| `Query_block`（= `QT_QUERY_BLOCK`） | `lex->new_query()` / `new_set_operation_query()` / `new_top_level_query()`；由 `PT_query_specification` push 进父层 elts |
| `Query_term_unary` | ① `PT_query_expression` 看到"外面还有一层 `SC_QUERY_EXPRESSION`"时；② set-op 的 block 已被 ORDER BY/LIMIT 装饰过（括号套 set-op）时 |
| `Query_term_union / _except / _intersect` | `PT_set_operation::contextualize_setop()` 收齐左右分支后创建 |
| 后处理块（`create_post_processing_block`） | `PT_set_operation` 建 setop 后、`PT_query_expression` 的 `QT_UNARY` 分支；**它是旧的 `fake_query_block` 的替代** |
| `m_children` 折叠 | `merge_descendants()`：同类型且无 LIMIT 的嵌套 set-op 被摊平（`A UNION (B UNION C)` → 三元 union） |

**★ `pushdown_limit_order_by()`**（`[P14]`）负责"消掉冗余的括号层"：

```cpp
    case QT_UNARY: {
      auto *unary = down_cast<Query_term_unary *>(this);
      Query_block *const this_block = unary->query_block();
      Query_block *const child_block = unary->m_children[0]->query_block();
      if (this_block->order_list.elements == 0 &&
          child_block->absorb_limit_of(this_block)) {
        // (SELECT a, c FROM t1 ORDER BY a LIMIT 4) LIMIT 2
        //   ==> SELECT a, c FROM t1 ORDER BY a LIMIT 2
        return unary->m_children[0]->pushdown_limit_order_by(parent);
      } else {
        if (child_block->order_list.elements == 0 &&
            child_block->select_limit == nullptr) {
          // (SELECT a, b AS c FROM t1) ORDER BY b+1 LIMIT 5
          //   ==> SELECT a, b AS c FROM t1 ORDER BY b+1 LIMIT 5
          // ★ 注意这让名字绑定变宽松（非标准）
          child_block->order_list = this_block->order_list;
          child_block->absorb_limit_of(this_block);
          ...
          // ORDER BY 里的 Item 要改挂到内层块的上下文
          Item_ident::Change_context ctx(&child_block->context);
          for (ORDER *o = this_block->order_list.first; o; o = o->next) {
            o->item_initial->walk(&Item::change_context_processor,
                                  enum_walk::POSTFIX,
                                  reinterpret_cast<uchar *>(&ctx));
          }
          ...
          return unary->m_children[0]->pushdown_limit_order_by(parent);
        }
      }
```

#### ★ 实例：`SELECT 1 UNION SELECT 2 ORDER BY 1` 的 `m_stack` 逐步状态

```
步骤  动作（谁）                                   m_stack（栈底→栈顶）
────  ──────────────────────────────────────────  ──────────────────────────────────
 0    Parse_context pc(thd, qb1)                  [ SC_TOP{elts=[]} ]
 2    PT_query_expression push SC_QUERY_EXPRESSION
      (has_order=true, 因为有 ORDER BY 1)          [ SC_TOP, QE{has_order=T} ]
 3    PT_union push SC_UNION_DISTINCT             [ SC_TOP, QE, UNION_D ]
 4    lhs=PT_query_specification(SELECT 1)
        push QS → itemize → pop → push qb1         [ SC_TOP, QE, UNION_D{elts=[qb1]} ]
 5    pc->select = new_set_operation_query(qb1) ⇒ qb2
 6    rhs=PT_query_specification(SELECT 2)         [ SC_TOP, QE, UNION_D{elts=[qb1,qb2]} ]
 7    lex->pop_context()
 8    pop UNION_D → ql{elts=[qb1,qb2]}
        new Query_term_union; merge_descendants; label_children;
        set_block(ppb); push setop 到父层 elts     [ SC_TOP, QE{elts=[union]} ]
 9    PT_query_expression: expr=union, pop QE
        union->query_block()->order_list = ORDER BY 1
        pc->select(qb2) → ppb → itemize → 复原 qb2
        m_stack.back()==SC_TOP（不是 QE，不补 unary）
        push union 到父层 elts                     [ SC_TOP{elts=[union]} ]
10    pc.finalize_query_expression()
        ql=SC_TOP{elts=[union]}; pop → []
        assert(ql.m_elts.size()==1) ✔
        top->pushdown_limit_order_by() → 递归 qb1/qb2（QT_QUERY_BLOCK 原样返回）
        qe->set_query_term(union); union->validate_structure(nullptr)
```

⇒ 最终 `Query_expression::m_query_term` 指向 `Query_term_union{ block=ppb(ORDER BY 1), children=[qb1, qb2] }`。

#### 7.1.5 ★ `finalize_query_expression()` 逐行

```cpp
// [P4]
bool Parse_context::finalize_query_expression() {
  QueryLevel ql = m_stack.back();      // ① 取顶层（应只剩 SC_TOP 这一层）
  m_stack.pop_back();                  // ② 弹出：此时 m_stack 应为空
  assert(ql.m_elts.size() == 1);       // ③ 顶层容器里必须恰好一个 Query_term
  Query_term *top = ql.m_elts.back();
  top = top->pushdown_limit_order_by();// ④ 下推 ORDER/LIMIT，消掉冗余的 unary 层
  select->master_query_expression()->set_query_term(top);  // ⑤ 挂到 Query_expression
  if (top->validate_structure(nullptr)) return true;       // ⑥ 结构自检
  return false;
}
```

**★ `assert(ql.m_elts.size() == 1)` 为什么成立？**——整条 push/pop 契约的**收敛式归纳**：

- **基**：构造函数压入 `SC_TOP`，此时 `elts=[]`。
- **归纳**：任何一个 push 了 `QueryLevel` 的节点，在 pop 之前都**恰好**往父层 elts 塞 **1** 个元素（`PT_query_specification` 塞 1 个 `Query_block`；`PT_set_operation` 塞 1 个 `setop`——内部已把 N 个孩子折进 `setop->m_children`；`PT_query_expression` 塞 1 个 `expr`）。
- ⇒ "一层 ⇄ 一个 term"，层层归约后最外层必然只剩 1 个根。

**为什么"后处理块"不会让 elts 变多？** 因为它是 `setop`/`unary` 的**内部成员**（`set_block()`），不是独立的 elts 元素。

⑤ 用 `pc->select->master_query_expression()`（而非 `lex->unit`），让 `finalize_query_expression()` 对"顶层语句"与"子查询/派生表"**一视同仁**。

#### 7.1.6 `is_top_level_union_all()`：UNION ALL 流式执行优化

```cpp
// [P4]
bool Parse_context::is_top_level_union_all(Surrounding_context op) {
  if (op == SC_EXCEPT_ALL || op == SC_INTERSECT_ALL) return false;
  assert(op == SC_UNION_ALL);
  for (size_t i = m_stack.size(); i > 0; i--) {        // 从栈顶往下扫
    switch (m_stack[i - 1].m_type) {
      case SC_UNION_DISTINCT: case SC_INTERSECT_DISTINCT:
      case SC_INTERSECT_ALL:  case SC_EXCEPT_DISTINCT:
      case SC_EXCEPT_ALL:     case SC_SUBQUERY:
        // 上面有去重/其它集合操作/子查询边界：结果必须物化或二次处理，不能边算边吐
        return false;
      case SC_QUERY_EXPRESSION:
        if (m_stack[i - 1].m_has_order) return false;  // 括号外的 ORDER BY 阻断流式
        [[fallthrough]];
      default:;
    }
  }
  return true;
}
```

★ 用途：在 `merge_descendants()` 里决定是否把嵌套的同类型 set-op **摊平**。摊平能得到 N 元算子（减少执行器层数与临时表），但会**丧失"流式"机会**——最外层是 `UNION ALL` 时，不摊平可以让右表边读边发，省一次物化：

```cpp
// [P2] merge_descendants() 片段
if (lower->m_last_distinct > 0) {
  if (pc->is_top_level_union_all(sc)) {
    // If we merge we lose the chance to stream the right table, so don't.
    setop->m_children.push_back(elt);
    count++;
  } else { /* ... 折叠进 m_children ... */ }
}
```

⇒ `m_has_order` 只在 `PT_query_expression` push `SC_QUERY_EXPRESSION` 时被置为 `m_order != nullptr`，正是为了这里判断"括号外面还有 ORDER BY"。

#### 7.1.7 ★ 5 种 Context 实例化：加料不换协议

`contextualize()` 里唯一对 Context 的要求是 `pc->thd`（栈检查）与 `pc->thd->syntax_error_at()`（error 辅助）——**只要提供了 `thd`，就能当 Context 用**，这就是"协议"。下面是四种"加料"：

| 实例化 | 额外成员 | 用途 |
|---|---|---|
| `Parse_context`（基准） | — | ≈90% 节点 |
| `Table_ddl_parse_context` | `create_info` / `alter_info` / `key_create_info`（**全 `const` 指针**） | CREATE/ALTER TABLE 的三个落点 |
| `Column_parse_context` | `is_generated` / `cf_appliers` | 列属性：**延迟应用队列** |
| `Partition_parse_context` | **多继承** `Parser_partition_info` | 分区游标（`curr_part_elem`） |
| `Alter_tablespace_parse_context` | 只有 `thd`/`mem_root` + 继承 `Tablespace_options` | ★ **不继承 `Parse_context`**——鸭子类型 |

```cpp
// [P1] ② DDL：三个成员全是 const 指针——"加的料"在 context 生命周期内不可改指向，
//     只能改指向对象的内容
struct Table_ddl_parse_context final : public Parse_context {
  HA_CREATE_INFO *const create_info;        // 表级选项落点
  Alter_info *const alter_info;             // ALTER 操作清单落点
  KEY_CREATE_INFO *const key_create_info;   // 索引级选项落点
};
typedef Parse_tree_node_tmpl<Table_ddl_parse_context> Table_ddl_node;
```

```cpp
// [P16] ③ 列属性：cf_appliers 是"延迟施加到 CreateField 的动作队列"——
//     列属性节点不在 contextualize 时直接改 CreateField（那时字段对象可能还没定型），
//     而是往队列里塞一个 CreateFieldApplier，等字段创建后再统一施加
struct Column_parse_context : public Parse_context {
  const bool is_generated;
  std::vector<CreateFieldApplier> cf_appliers;
};
```

```cpp
// [P17] ④ 分区：多继承——一份通用协议 + 一份分区专属游标
struct Partition_parse_context : public Parse_context,
                                 public Parser_partition_info { ... };
typedef Parse_tree_node_tmpl<Partition_parse_context> Parse_tree_part_node;

// 消费示例：节点只负责"往当前元素写一个字段"
bool PT_partition_comment::contextualize(Partition_parse_context *pc) override {
  if (super::contextualize(pc)) return true;
  pc->curr_part_elem->part_comment = comment;
  return false;
}
```

★ ⑤ `Alter_tablespace_parse_context` 是极端形态：**它不继承 `Parse_context`**，只是"鸭子类型"地提供了同名的 `thd` 与 `mem_root`，从而满足 `contextualize()` 对 `pc->thd` 的编译期要求——**换协议也行，只要字段名对得上**。它继承的 `Tablespace_options` 就是 payload 本身（选项节点直接写 `pc->nodegroup_id`）。

#### 7.1.8 `Parse_context` 与 `LEX` 的关系

| LEX 侧 | Parse_context 侧 | 关系 |
|---|---|---|
| `lex->current_query_block()` | `pc->select` | **两者经常不同步**。`Select` 宏（`[P15]`）取 `current_query_block()` 构造新 context；之后 `pc->select` 在 UNION/后处理块处自由前进，而 `m_current_query_block` 不动 |
| `lex->new_query(pc->select)` | `PT_subquery` 的 `inner_pc` | 以 `pc->select` 为父创建子 QE+QB |
| `lex->new_set_operation_query(pc->select)` | `pc->select` 前进 | 创建 UNION 右分支兄弟块 |
| `lex->push_context() / pop_context()` | — | **名字解析上下文栈**（`LEX::context_stack`），与 `pc->m_stack` 是两个独立栈，必须成对 |
| `lex->will_contextualize` | — | `[P15]` 的宏靠它决定是否真的做 contextualize |

★ `push_context()` 的真正来源是 `Query_block::set_context()`：

```cpp
// [P10]
bool Query_block::set_context(Name_resolution_context *outer_context) {
  context.init();
  context.query_block = this;
  context.outer_context = outer_context;
  return parent_lex->push_context(&context);   // ★ 与 PT_subquery / PT_set_operation 末尾的 pop 配对
}
```

#### 7.1.9 `enum_parsing_context` 与 `parsing_place` 的置位/复位规则

`CTX_*` 共 30 个值（`[P3]`），常用者与规则：

| `CTX_*` | 谁置位 | 谁消费 | 谁复位 |
|---|---|---|---|
| `CTX_SELECT_LIST` | `PT_query_specification`（SELECT 列表 + WINDOW 子句） | `Item::itemize()` 计数 `select_n_having_items` | 同函数内 `= CTX_NONE` |
| `CTX_WHERE` / `CTX_HAVING` | `PTI_where` / `PTI_having`（经 `PTI_context`） | `PTI_simple_ident_q_3d` 选 `Item_ref` vs `Item_field` | `PTI_context::itemize()` 末尾 |
| `CTX_ON` | `PT_joined_table_on` | `make_condition`、名字解析 | 同函数末尾 |
| `CTX_DERIVED` | `PT_derived_table` | `LEX::new_query()` 决定 `outer_context` | 同函数末尾 |
| `CTX_GROUP_BY` / `CTX_ORDER_BY` | `PT_group` / `PT_order` | — | 同函数 |

**`assert` 如何兜底**（这个隐式状态机没有编译期保护，只能靠断言）：

```cpp
// PT_query_specification：防止嵌套 SELECT 在子块里把父块的 parsing_place 改掉
assert(pc->select->parsing_place == CTX_SELECT_LIST);
// PT_joined_table_on：ON 里不允许子 SELECT 偷换 select
assert(sel == pc->select);
assert(sel->parsing_place == CTX_ON);
// PT_query_expression：切到后处理块时必须是干净的
assert(pc->select->parsing_place == CTX_NONE);
// PTI_context：复位前确认没被别人改
assert(pc->select->parsing_place == m_parsing_place);
```

### 7.2 ★ `itemize()` 深度剖析

> ★ **常见误解澄清：不是只有 `PTI_*` 才走 `itemize()`。**
>
> `itemize()` 是**表达式层的统一递归入口**（对应结构层的 `contextualize()`）——凡是 `is_parser_item == true` 的 Item，无论是不是 `PTI_*`，都要走。区别只在"走完做什么"：
>
> | 类别 | `itemize()` 里做什么 | 例子 |
> |---|---|---|
> | `PTI_*`（占位 Item） | **变身**——`*res` 换成新对象 | `PTI_comp_op`、`PTI_simple_ident_q_3d` |
> | 真 Item（yacc 用 `POS` 直接 new） | **递归自己的 args**（让子树里的 PTI 变身），自己不变身 | `Item_func_plus`、`Item_func_xor` |
> | 工厂/运行时造的（`is_parser_item == false`） | `skip_itemize()` 立即返回（幂等 no-op） | `Item_func_now_local`、`Create_func` 产物 |
>
> **实测**：`sql/parse_tree_items.h` 有 **33** 个 `PTI_*::itemize`；而 `sql/*.cc` 里 `Item_*::itemize` 的定义有 **57** 个（`Item_ident`、`Item_field`、`Item_param`、`Item_default_value`、`Item_asterisk`、`Item_cond`、`Item_func_interval`…）——**非 PTI 的 override 反而更多**。

★ **真 Item 如何递归 args**（`Item_func::itemize()`，`[P?] sql/item_func.cc`）——这是"表达式层递归"真正展开的地方：

```cpp
bool Item_func::itemize(Parse_context *pc, Item **res) {
  if (skip_itemize(res)) return false;              // ① 工厂造的 → 幂等返回
  if (Item_result_field::itemize(pc, res)) return true;   // ② 走基类：登记 + 计数
  const bool no_named_params = !may_have_named_parameters();
  for (size_t i = 0; i < arg_count; i++) {          // ★③ 递归 args
    add_accum_properties(args[i]);
    if (args[i]->itemize(pc, &args[i])) return true;  // ★ 传 &args[i] —— 原地替换槽位
    if (no_named_params && !args[i]->item_name.is_autogenerated()) {
      my_error(functype() == FUNC_SP ? ER_WRONG_PARAMETERS_TO_STORED_FCT
                                     : ER_WRONG_PARAMETERS_TO_NATIVE_FCT,
               MYF(0), func_name());               // ④ 具名参数检查
      return true;
    }
  }
  return false;
}
```

⇒ 这就是 `SELECT ABS(t.a) + t.b * 2` 里 `Item_func_plus`（yacc 直接 new 的真 Item）如何让自己 args 里的 `PTI_simple_ident_q_2d("t","b")` 变身的答案：**`PT_item_list::contextualize()` 只对顶层元素调 `itemize()`，深层 args 的递归由 `Item_func::itemize()` 自己完成**。两套递归因此能无缝咬合。

#### 7.2.1 基类实现逐行

```cpp
// [P11]
  explicit Item(const POS &);
 private:
  /*
    Hide the contextualize*() functions: call/override the itemize()
    in Item class tree instead.
  */
  bool contextualize(Parse_context *) override {
    assert(0);       // ★ Item 不许直接走 contextualize()，只能走 itemize()
    return true;
  }
 protected:
  bool skip_itemize(Item **res) {
    *res = this;                 // 无论如何先"原样透传"
    return !is_parser_item;      // 非 parser 分配 => 不用再做任何事
  }
```

⇒ ★ 注意这个**设计上的"封印"**：`Item` 把 `contextualize()` 私有化并 `assert(0)`，强制表达式层只能走 `itemize()`。这正是"两套递归虽有共同基类、但入口被分开"的实现手段。

```cpp
// [P7] sql/item.cc
bool Item::itemize(Parse_context *pc, Item **res) {
  // ① 若本对象不是 parser 直接分配的（is_parser_item == false），
  //    说明它已是"完整"的 Item：*res = this 后直接返回。
  //    ⇒ 这让"对新造对象再 itemize 一次"成为无害的幂等写法（见 7.2.4）
  if (skip_itemize(res)) return false;

  // ② 走 Parse_tree_node_tmpl<Parse_context>::contextualize()：
  //    栈溢出检查 + debug 下 assert(!contextualized) 并置位（防二次消费）
  if (super::contextualize(pc)) return true;

  // ③ 登记到 THD 的全局 Item 链表，供语句结束/失败时统一析构
  pc->thd->add_item(this);

  /*
    Item constructor can be called during execution other then SQL_COM
    command => we should check pc->select on zero
  */
  // ④ pc->select 可能为 NULL（后处理块尚未建立，见 7.1.2 场景 ②）
  if (pc->select) {
    const enum_parsing_context place = pc->select->parsing_place;
    if (place == CTX_SELECT_LIST || place == CTX_HAVING || place == CTX_ORDER_BY) {
      pc->select->select_n_having_items++;   // 为 hidden item 预留 base_ref_items 槽位
    }
  }
  return false;
}
```

#### 7.2.2 ★ 五种"变身模式"

```
                        ┌─────────────────────────────────────────────┐
   *res 的五种写法       │  PTI_xxx::itemize(pc, res) 之后 *res 是谁？  │
                        └─────────────────────────────────────────────┘

 ① 原样透传             PTI ─────────────────────────►  PTI 自己
    *res = this         （skip_itemize 已做）例：PTI_user_variable

 ② 换成新对象           PTI ──new──►  Item_func_xxx / Item_field / Item_ref
    *res = new ...      例：PTI_comp_op、PTI_simple_ident_q_3d

 ③ 查表决定（三级回退）  PTI ──find_native_function_builder──► Item_func_xxx
    if/else if/else        └─find_udf────────────────────► Item_udf_func
                           └─find_qualified_function_builder► Item_func_sp
                        例：PTI_function_call_generic_ident_sys

 ④ 上下文壳剥离         PTI_where/having/context ──► expr（剥掉外壳）
    parsing_place 置位 → itemize → make_condition → 复位 → *res = expr

 ⑤ 子查询跨界           PTI_singlerow_subselect ──► Item_singlerow_subselect
    先 subselect->contextualize(pc)（可能压 inner_pc）
    再把结果包成 Item
```

##### 模式 ①：原样透传

```cpp
// [P6] —— PTI_user_variable 既是 PT 节点又是最终 Item（继承 Item_func_get_user_var）
class PTI_user_variable final : public Item_func_get_user_var {
  bool itemize(Parse_context *pc, Item **res) override;
};
// [P5]
bool PTI_user_variable::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;   // *res = this 已在此完成
  LEX *lex = pc->thd->lex;
  if (!lex->parsing_options.allows_variable) {
    my_error(ER_VIEW_SELECT_VARIABLE, MYF(0));   // VIEW 里不允许 @var
    return true;
  }
  lex->set_uncacheable(pc->select, UNCACHEABLE_RAND);   // 用户变量不确定 → 不可缓存
  return false;                                 // ★ 没有 *res = ...：this 就是最终 Item
}
```

⇒ 这类 `PTI_*` **不需要变身**，只需"补上下文"（这里补的是缓存性标记）。

##### 模式 ②：换成新对象

```cpp
// [P5] PTI_comp_op —— boolfunc2creator 是语法期就定好的"造函数器"
bool PTI_comp_op::itemize(Parse_context *pc, Item **res) {
  // 注意传的是 &left / &right —— 成员槽位地址！所以左/右操作数变身也能原地生效
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      right->itemize(pc, &right))
    return true;
  *res = (*boolfunc2creator)(false)->create(left, right);   // ★ 变身
  return *res == nullptr;
}
```

★ `PTI_simple_ident_q_3d` 是**`parsing_place` 影响变身目标**的教科书例子：

```cpp
// [P5]
bool PTI_simple_ident_q_3d::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  const char *schema =
      thd->get_protocol()->has_client_capability(CLIENT_NO_SCHEMA) ? nullptr : db;
  if (pc->select->no_table_names_allowed) { ...; return true; }

  // ★★ HAVING 中且不在聚合函数里 → Item_ref（引用 SELECT 列表已算出的列）
  //    否则                      → Item_field（真正读表的字段）
  if ((pc->select->parsing_place != CTX_HAVING) ||
      (pc->select->get_in_sum_expr() > 0)) {
    *res = new (pc->mem_root) Item_field(POS(), schema, table, field);
  } else {
    *res = new (pc->mem_root) Item_ref(POS(), schema, table, field);
  }
  if (*res == nullptr) return true;
  return (*res)->itemize(pc, res);          // ★ 二次 itemize，见 7.2.4
}
```

⇒ `SELECT a, COUNT(*) FROM t GROUP BY a HAVING a > 1` 里，`HAVING a` 的 `a` 是 `Item_ref`（引用第 1 列）；而 `HAVING COUNT(a) > 1` 的 `a` 在聚合内部（`in_sum_expr > 0`），走 `Item_field`。

##### 模式 ③：查表决定（三级回退）

```cpp
// [P5]
bool PTI_function_call_generic_ident_sys::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  udf = nullptr;
  // 先验：同名 UDF 若是聚合函数，先"预占" in_sum_expr（此刻还不知会不会走 UDF 分支）
  if (using_udf_functions && (udf = find_udf(ident.str, ident.length)) &&
      udf->type == UDFTYPE_AGGREGATE) {
    pc->select->in_sum_expr++;
  }
  if (sp_check_name(&ident)) return true;
  /*
    names are resolved with the following order:
    - MySQL native functions,   ← ABS 命中这一档
    - User Defined Functions,
    - Stored Functions (assuming the current <use> database)
  */
  Create_func *builder = find_native_function_builder(ident);
  if (builder)
    *res = builder->create_func(thd, ident, opt_udf_expr_list);
  else {
    if (udf) {
      if (udf->type == UDFTYPE_AGGREGATE) pc->select->in_sum_expr--;  // ★ 撤销预占
      *res = Create_udf_func::s_singleton.create(thd, udf, opt_udf_expr_list);
    } else {
      builder = find_qualified_function_builder(thd);
      *res = builder->create_func(thd, ident, opt_udf_expr_list);      // stored function
    }
  }
  return *res == nullptr || (*res)->itemize(pc, res);
}
```

★ `in_sum_expr` 的"**预占-撤销**"是这个模式最精妙的地方：UDF 与内建函数同名时，先乐观当作聚合函数计数，落定后立刻撤销。

##### 模式 ④：上下文壳剥离（`PTI_where` / `PTI_having` / `PTI_context`）

```cpp
// [P6]
class PTI_context : public Parse_tree_item {
  Item *expr;
  const enum_parsing_context m_parsing_place;   // 语法期定死：CTX_WHERE / CTX_HAVING
};
class PTI_where final : public PTI_context {
  PTI_where(const POS &pos, Item *expr_arg) : PTI_context(pos, expr_arg, CTX_WHERE) {}
};
```

```cpp
// [P5]
bool PTI_context::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;

  pc->select->parsing_place = m_parsing_place;   // ★ ① 置位
  if (expr->itemize(pc, &expr)) return true;     // ★ ② 递归（子树会读 parsing_place）
  if (!expr->is_bool_func()) {
    expr = make_condition(pc, expr);             // ★ ③ 消费：非布尔 → 包成条件
    if (expr == nullptr) return true;
  }
  assert(pc->select->parsing_place == m_parsing_place);
  pc->select->parsing_place = CTX_NONE;          // ★ ④ 复位
  expr->apply_is_true();
  *res = expr;                                   // ★ ⑤ 剥壳：壳从 Item 树里消失
  return false;
}
```

⇒ 这回答了一个常见疑问：**WHERE / HAVING 节点本身不是 Item**，它们只是"上下文壳"，itemize 完成后 self-destruct。

##### 模式 ⑤：子查询跨界

```cpp
// [P5]
bool PTI_singlerow_subselect::itemize(Parse_context *pc, Item **res) {
  // ① 先让子查询整棵 parse tree 完成 contextualize：
  //    内部会 new_query() 建子 QE/QB、建 inner_pc、跑完整棵子树、finalize、复原
  if (super::itemize(pc, res) || subselect->contextualize(pc)) return true;
  // ② 再把"子查询的 Query_expression 句柄"包成一个 Item
  *res = new (pc->mem_root) Item_singlerow_subselect(subselect->value());
  pc->select->n_scalar_subqueries++;
  return *res == nullptr;
}
```

★ 注意 `super::itemize()` 在 `subselect->contextualize()` **之前**——因为计数用当时的 `parsing_place`，而子查询 contextualize 期间 `pc->select` 可能被临时切换。

##### 附：`PTI_expr_with_alias` —— 默认列名来自**源码原文**

```cpp
// [P5]
  if (alias.str) {
    if (pc->thd->lex->sql_command == SQLCOM_CREATE_VIEW && check_column_name(alias.str)) {
      my_error(ER_WRONG_COLUMN_NAME, MYF(0), alias.str);
      return true;
    }
    expr->item_name.copy(alias.str, alias.length, system_charset_info, false);
  } else if (!expr->item_name.is_set()) {
    // ★ 无别名：把源码里这段表达式的原始文本当作列名
    //   expr_loc 是 POS 记录的 [raw.start, raw.end)，指向原始 SQL 缓冲区
    expr->item_name.copy(expr_loc.start, (uint)(expr_loc.end - expr_loc.start),
                         pc->thd->charset());
  }
  *res = expr;
```

⇒ `SELECT a+1 FROM t` 的列名叫 `a+1`，就是这段代码的功劳。

#### 7.2.3 ★ `res` 为什么必须是二级指针

`Item **res` 是**"槽位地址"**，不是"对象"。调用方把"存放这个 Item 的槽位"交出来，itemize 才能**原地替换槽位内容**：

```cpp
// [P2] —— 注意是 Item *&item（引用！）
template <typename Context>
bool contextualize_safe(Context *pc, mem_root_deque<Item *> *list) {
  if (list == nullptr) return false;
  for (Item *&item : *list) {          // item 是容器内元素的引用
    if (item->itemize(pc, &item)) return true;   // ★ 传的是该元素的地址
  }
  return false;
}

// [P9] —— PT_item_list 同理
  for (Item *&item : value) { if (item->itemize(pc, &item)) return true; }
```

```
调用前（语法刚建好）:
   PT_item_list::value            [0]                    [1]
                                   │                      │
                            PTI_expr_with_alias      PTI_simple_ident_ident
                            （PTI，临时的壳）          （PTI，临时的壳）

PT_item_list::contextualize(pc):
   for (Item *&item : value)  item->itemize(pc, &item)
                    ^^^^              ^^^^^^  ^^^^^^
                   引用            旧对象     槽位地址（&value[0]）

   PTI_expr_with_alias::itemize():  *res = expr;   ──► 原地改写 value[0]

调用后:   value[0] ──► Item_func_plus（壳没了）
          value[1] ──► Item_field
```

⇒ 若用一级指针（值传递），`*res = expr` 只改形参副本，槽位仍指向已被抛弃的 PTI 壳——执行期 `val_*()` 会走到 `Parse_tree_item` 的 `assert(0)`。

#### 7.2.4 ★ 为什么要再调一次 `(*res)->itemize(pc, res)`

模式 ②/③ 生成的新对象有两种出身：

| 出身 | 例子 | `is_parser_item` | 再调一次的效果 |
|---|---|---|---|
| 用 `POS()` 构造 | `new Item_field(POS(), ...)`、`new Item_ref(POS(), ...)` | **true** | **真正执行**：contextualize + `add_item` + 计数；若它自己也 override 了 `itemize()` 还会跑它 |
| 用非 POS 构造函数（`Create_func` 等工厂造出） | `Item_func_eq`、`Item_func_now_local` | **false** | `skip_itemize()` 立即返回 true ⇒ **幂等 no-op** |

⇒ 于是"再造一次"被写成**统一、无条件**的惯用法，既安全又覆盖两类出身。**不做会怎样？** 用 `POS()` 造的 `Item_field` 不会被 `add_item()` 登记（异常路径悬空），也不计入 `select_n_having_items`（后续拆 hidden item 时越界）。

真实例子——非 `PTI_*` 的 Item 也会 override `itemize()`（如 `Item_func_xor`）：

```cpp
bool Item_func_xor::itemize(Parse_context *pc, Item **res) {
  // 工厂用非 POS 构造函数造的（is_parser_item=false）→ 幂等兜底
  if (skip_itemize(res)) return false;
  if (super::itemize(pc, res)) return true;
  // 只有语法期用 POS() 构造时才走到这里：把非布尔参数包成条件
  if (!args[0]->is_bool_func()) { args[0] = make_condition(pc, args[0]); ... }
  if (!args[1]->is_bool_func()) { args[1] = make_condition(pc, args[1]); ... }
  return false;
}
```

#### 7.2.5 `is_parser_item` 与 `skip_itemize()`

- **`is_parser_item`**：`const bool`，由**用哪个构造函数**决定——`Item(const POS &)` → `true`；其余构造函数 → `false`。
  - `true` ⇒ "我是语法阶段由 grammar action 直接 new 出来的半成品，需要 `itemize()` 补完上下文"；
  - `false` ⇒ "我是运行时/工厂造出来的完整 Item，别碰我"。
- ★ **`Parse_tree_item`**（所有 `PTI_*` 的公共基类）的 `type()` 是 `INVALID_ITEM`，所有 `val_*()` 都是 `assert(0)`——**PTI 绝不允许活到执行期**，必须在 itemize 阶段被变身/剥壳。这条"断言防线"是整套机制的安全网。

### 7.3 两者的交界：`Parse_context` 在 itemize 中扮演什么

#### 为什么 `itemize()` 只接受 `Parse_context*`

```cpp
// [P11] 非模板，不参数化 Context
virtual bool itemize(Parse_context *pc, Item **res);
// 对照：[P3] 模板化
virtual bool contextualize(Context *pc) { ... }
```

1. **Item 只活在"查询块"语义里**：Item 需要 `pc->select`（`parsing_place` / `in_sum_expr` / `select_n_having_items` / `add_item_to_list`）。而 `Table_ddl_parse_context` / `Partition_parse_context` / `Alter_tablespace_parse_context` 的使用场景（DDL 选项、分区定义）根本不产生 Item 树。把 `itemize()` 绑死在 `Parse_context` 上，等于用类型系统声明"**只有查询上下文才能造 Item**"。
2. **避免虚表膨胀**：`itemize()` 若也模板化，每个 `PTI_*` 都要为每个 Context 生成一个虚调用点，收益为零。
3. `[P15]` 的 `ITEMIZE` 宏**硬编码** `Parse_context pc(YYTHD, Select);`。

#### `pc->select` 上的"动态作用域变量"范式

`in_sum_expr` 是最典型的例子——由外壳节点维护、内层只读：

```cpp
// [P5] PTI_in_sum_expr：进入聚合函数参数前 +1，出来 -1，然后剥壳
bool PTI_in_sum_expr::itemize(Parse_context *pc, Item **res) {
  pc->select->in_sum_expr++;
  if (super::itemize(pc, res) || expr->itemize(pc, &expr)) return true;
  pc->select->in_sum_expr--;
  *res = expr;
  return false;
}
```

⇒ 这个"加-递归-减-剥壳"四段式与 `PTI_context` 的"置位-递归-消费-复位-剥壳"五段式是**同一范式**。

```
  pc->thd        →  分配器（mem_root）+ 全局 Item 链表（add_item）+ 报错通道
  pc->select     →  落点（add_item_to_list / set_where_cond …）
                    + 动态作用域状态（parsing_place / in_sum_expr / n_child_sum_items …）
  pc->m_stack    →  itemize 阶段只读（供 is_top_level_union_all 判断；
                    只有 PTI_*_subselect 会经 PT_subquery 间接推动它）
```

### 知识库内相关文档（`Parse_context` 章）

| 主题 | 归属 |
|---|---|
| `sql_yacc.yy` 文法本身（产生式写法、四个宏、Bison 构建期生成、LEX 结构） | [`04_parser.md`](04_parser.md) |
| contextualize 流程（调用链、QB/QE 创建与填充、`Query_term` 树、`PTI_*` 36 个类全谱） | [`05_contextualize.md`](05_contextualize.md) |
| Item 侧（`PTI_*` 变身之后：`fix_fields` / `val_*` / 类型推导） | [`13_item_expression.md`](13_item_expression.md) |
| 词法与 token（`lex.h` 符号表、`int_token()`） | [`02_lexer.md`](02_lexer.md) |
| 文法记法（如何读 `.yy`） | [`03_grammar_notation.md`](03_grammar_notation.md) |

### 外部参考（`Parse_context` 章）

> ⚠️ 待补：按知识库规范，动笔前应深度检索 **MySQL 8.0 Reference Manual** 与相关 **WorkLog**（如 WL#7196 Parse tree 重构、WL#8081 contextualize 阶段），并回源码核实后补入本节。

---

## 八、集合运算节点：`PT_union` / `PT_except` / `PT_intersect`

> **边界声明**：本节只讲 **PT → AST 的构造/填充侧**。执行期如何物化、如何去重（`MaterializeIterator` 等）属 [`07_optimize/19_set_operation.md`](07_optimize/19_set_operation.md)，本节不展开。
>
> 本节源码坐标：`[P1] sql/parse_tree_nodes.h` · `[P2] sql/parse_tree_nodes.cc` · `[P3] sql/parse_tree_node_base.h` · `[P4] sql/parse_tree_node_base.cc` · `[P10] sql/sql_lex.cc` · `[P13] sql/query_term.h` · `[P14] sql/query_term.cc` · `[P15] sql/sql_yacc.yy`

### 8.1 三兄弟：极薄的子类 + 极厚的父类

8.0.31 引入 `Query_term` 树后，三个语法节点被压缩到**几乎没有实体**——它们只给父类提供一个 `Query_term_type` + `Surrounding_context` 标签：

```cpp
// [P1]
class PT_set_operation : public PT_query_expression_body {
 public:
  PT_set_operation(PT_query_expression_body *lhs, bool is_distinct,
                   PT_query_expression_body *rhs,
                   bool is_rhs_in_parentheses = false)
      : m_lhs(lhs), m_is_distinct(is_distinct), m_rhs(rhs),
        m_is_rhs_in_parentheses{is_rhs_in_parentheses} {}
  void merge_descendants(Parse_context *pc, Query_term_set_op *setop, QueryLevel &ql);
  bool can_absorb_order_and_limit(bool, bool) const override { return false; }
 protected:
  bool contextualize_setop(Parse_context *pc, Query_term_type setop_type,
                           Surrounding_context context);
  PT_query_expression_body *m_lhs;
  bool m_is_distinct;
  PT_query_expression_body *m_rhs;
  const bool m_is_rhs_in_parentheses;
};

class PT_union : public PT_set_operation { bool contextualize(Parse_context *) override; };
class PT_except : public PT_set_operation { bool contextualize(Parse_context *) override; };
class PT_intersect : public PT_set_operation { bool contextualize(Parse_context *) override; };
```

| 成员 | 含义 |
|---|---|
| `m_lhs` / `m_rhs` | 类型是**抽象基类** `PT_query_expression_body` ⇒ 可以是 `PT_query_specification`、另一个 `PT_union`，或括号包起来的 `PT_query_expression`——**嵌套由此而来**（与 8.0.31 前 `st_select_lex_unit` 的链表结构不同，现在是真正的 PT 树） |
| `m_is_distinct` | `true` ⇒ `UNION [DISTINCT]`；`false` ⇒ `... ALL` |
| ★ `m_last_distinct` | **不在本类上**！它属于 AST 侧的 `Query_term_set_op`（`[P13]`），由 `merge_descendants()` 计算并写入 |

三个子类的全部工作就是**选标签**（`[P2]`）：

```cpp
bool PT_union::contextualize(Parse_context *pc) {
  return contextualize_setop(pc, QT_UNION,
                             m_is_distinct ? SC_UNION_DISTINCT : SC_UNION_ALL);
}
bool PT_except::contextualize(Parse_context *pc) {
  return contextualize_setop(pc, QT_EXCEPT,
                             m_is_distinct ? SC_EXCEPT_DISTINCT : SC_EXCEPT_ALL);
}
bool PT_intersect::contextualize(Parse_context *pc) {
  return contextualize_setop(pc, QT_INTERSECT,
                             m_is_distinct ? SC_INTERSECT_DISTINCT : SC_INTERSECT_ALL);
}
```

### 8.2 ★ `contextualize_setop()`：三兄弟共用的骨架

```cpp
// [P2]
bool PT_set_operation::contextualize_setop(Parse_context *pc,
                                           Query_term_type setop_type,
                                           Surrounding_context context) {
  pc->m_stack.push_back(QueryLevel(pc->mem_root, context));   // ① 压本层 QueryLevel
  if (super::contextualize(pc)) return true;

  if (m_lhs->contextualize(pc)) return true;                  // ② 左分支递归

  // ★ 创建右分支的 Query_block 并把游标永久推到它上面（qb1 UNION qb2 的 qb2）
  pc->select = pc->thd->lex->new_set_operation_query(pc->select);  // ③ 前进

  if (pc->select == nullptr || m_rhs->contextualize(pc)) return true;  // ④ 右分支递归

  pc->thd->lex->pop_context();                                // ⑤ 配对 ③ 内部的 push_context

  QueryLevel ql = pc->m_stack.back();                         // ⑥ 取回本层收集到的 elts
  pc->m_stack.pop_back();

  Query_term_set_op *setop = nullptr;                         // ⑦ 按类型 new 对应 AST 节点
  switch (setop_type) {
    case QT_UNION:     setop = new (pc->mem_root) Query_term_union(pc->mem_root); break;
    case QT_EXCEPT:    setop = new (pc->mem_root) Query_term_except(pc->mem_root); break;
    case QT_INTERSECT: setop = new (pc->mem_root) Query_term_intersect(pc->mem_root); break;
    default: assert(false);
  }
  if (setop == nullptr) return true;

  merge_descendants(pc, setop, ql);                           // ⑧ ★ 摊平 + 算 distinct 下标
  setop->label_children();                                    // ⑨ 给每个孩子编号 sibling_idx

  Query_expression *qe = pc->select->master_query_expression();
  if (setop->set_block(qe->create_post_processing_block(setop))) return true;  // ⑩ 后处理块
  pc->m_stack.back().m_elts.push_back(setop);                 // ⑪ 把自己交给父层
  return false;
}
```

- ② **左分支执行期间 `pc->select` 不变**——集合运算的第一个操作数复用当前 `Query_block`。
- ③ `new_set_operation_query()` 内部 `include_neighbour()` 把新 QB **横向**挂到 `curr_query_block->next` ⇒ 所有叶子 QB 是**同一条 `next` 链**（qb1 → qb2 → qb3）；且新 QB 的 `outer_context` 取自 **first_query_block 的 outer_context**，所以第二个 QB 不能看见第一个 QB 的字段。
- ⑩ 后处理块是**老的 `fake_query_block` 的继承者**（`[P10]` `create_post_processing_block()`）：`include_standalone()`（**不进 `next` 链**）、`linkage = GLOBAL_OPTIONS_TYPE`、`select_number = INT_MAX`；★ `no_table_names_allowed` 在**非 unary 时一律 `true`** ⇒ `SELECT a FROM t1 UNION SELECT b FROM t2 ORDER BY t1.a` 会报错（ORDER BY 只能引用结果列名）。

### 8.3 ★ `merge_descendants()`：把左结合二叉树摊平成 N 元树

这是全篇最精妙的一步——**语法上是二叉左结合树，语义上重构成 N 元树**：

```cpp
// [P2]（节选骨架，注释为源码自带）
void PT_set_operation::merge_descendants(Parse_context *pc,
                                         Query_term_set_op *setop, QueryLevel &ql) {
  int count = 0;
  int64_t last_distinct = 0;
  int64_t first_distinct = std::numeric_limits<int64_t>::max();
  const Query_term_type op = setop->term_type();
  Surrounding_context sc = qt2sc(op);

  if (m_is_distinct) {
    for (Query_term *elt : ql.m_elts) {
      if (elt->term_type() == QT_QUERY_BLOCK ||  /* 1 叶子，无从合并 */
          elt->term_type() != op            ||  /* 2 另一类 set-op */
          (op == QT_EXCEPT && count > 0)) {     /* 3 EXCEPT 非首位（不是右结合）*/
        count++;
        if (first_distinct == INT64_MAX) first_distinct = 1;
        last_distinct = count - 1;
        setop->m_children.push_back(elt);
      } else if (Query_term_set_op *lower = down_cast<Query_term_set_op *>(elt);
                 elt->query_block()->select_limit == nullptr) {   /* 4 嵌套层无 LIMIT */
        last_distinct = count + lower->m_children.size() - 1;
        count = count + lower->m_children.size();
        for (auto child : lower->m_children) setop->m_children.push_back(child);  // ★ 摊平
      } else { /* 同类但带 LIMIT，不合并 */ }
    }
  } else {
    for (Query_term *elt : ql.m_elts) {
      /* ... */
      } else if (Query_term_set_op *lower = down_cast<Query_term_set_op *>(elt);
                 (count == 0 || lower->m_last_distinct == 0 || op == QT_INTERSECT)) {
        if (lower->m_last_distinct > 0) {
          if (pc->is_top_level_union_all(sc)) {
            // ★★ If we merge we lose the chance to stream the right table, so don't.
            setop->m_children.push_back(elt);
            count++;
          } else { /* ... 摊平 ... */ }
        }
        /* ... */
      }
    }
  }
  if (last_distinct > 0 && op == QT_INTERSECT)  // distinct always wins
    last_distinct = setop->m_children.size() - 1;
  setop->m_last_distinct = last_distinct;
  setop->m_first_distinct = first_distinct;
}
```

**三兄弟各自的摊平规则**：

| 操作符 | 能否摊平为 N 元 | 约束 |
|---|---|---|
| **UNION** | ✅ 积极摊平 | 同类型 + 嵌套层无 `LIMIT`；上层 ALL 且下层含 DISTINCT 时还要过 `is_top_level_union_all()` |
| **INTERSECT** | ✅ 极积极 | 唯一例外：**INTERSECT ALL 保持二元**（执行期只能算二元） |
| **EXCEPT** | ⚠️ 只摊平**最左** | `EXCEPT` 非右结合 ⇒ `(op == QT_EXCEPT && count > 0)` 直接拒绝非首位 |

```
r EXCEPT (s EXCEPT t)   → 不能摊平（(s EXCEPT t) 必须先算完）
(r EXCEPT s) EXCEPT t   → 可以摊平成三元 [r, s, t]
```

#### ★ 8.3.1 摊平 vs 流式的取舍（全篇最精妙的设计权衡）

**为什么"不摊平"有时更好？** 一旦摊成 N 元，所有操作数都要进**同一个临时表**；不摊平时，右表可以边读边发（streaming），省一次写盘。源码注释给了两个 `EXPLAIN FORMAT=tree` 对比：

```
-- 无 ORDER BY：不摊平，右表流式
SELECT * FROM t1 UNION DISTINCT SELECT * FROM t2 UNION ALL SELECT * FROM t3;
  -> Append
     -> Stream results
         -> Table scan on <union temporary>
             -> Union materialize with deduplication
                 -> Table scan on t1
                 -> Table scan on t2
     -> Stream results
         -> Table scan on t3          ← ★ t3 完全不进临时表

-- 有 ORDER BY a：摊平（is_top_level_union_all == false）
SELECT * FROM t1 UNION DISTINCT SELECT * FROM t2 UNION ALL SELECT * FROM t3 ORDER BY a;
  -> Sort: a
     -> Table scan on <union temporary>
         -> Union materialize with deduplication
             -> Table scan on t1
             -> Table scan on t2
             -> Disable deduplication
                 -> Table scan on t3   ← 只需一个临时文件
```

- `is_top_level_union_all()` 返回 `true`（可流式）⇒ **宁可不摊平也要保住右表流式**；
- 上方有 ORDER BY / DISTINCT / 子查询 ⇒ 流式无望 ⇒ **积极摊平**省临时表。

#### ★ 8.3.2 `m_last_distinct` / `m_first_distinct` 的语义

```cpp
// [P13]
  mem_root_deque<Query_term *> m_children;
  /**
     Index of last query expression which has <set-op> DISTINCT on its left. ...
     all blocks to its left are included. Which is why we only need to
     remember that query block.
  */
  int64_t m_last_distinct;
  /// Presently only needed by EXCEPT: index of the first DISTINCT set operand
  int64_t m_first_distinct;
```

⇒ **`m_children` 中下标 ≤ `m_last_distinct` 的孩子参与去重**，更大的直接追加。与 `has_mixed_distinct_operators()` 完全吻合：

```cpp
// [P14]
bool Query_term_set_op::has_mixed_distinct_operators() {
  return (m_last_distinct > 0) &&
         (static_cast<size_t>(m_last_distinct) < (m_children.size() - 1));
}
```

```
N=3, m_last_distinct=2  → 不混合：三个操作数整体去重   (A ALL B) DISTINCT C
N=3, m_last_distinct=1  → 混合  ：前两个去重，第三个直接追加  (A DISTINCT B) ALL C
N=2, m_last_distinct=0  → 不混合：纯 ALL                A UNION ALL B
```

★ INTERSECT 收尾有"**去重赢者通吃**"：`if (last_distinct > 0 && op == QT_INTERSECT) last_distinct = N-1;`——只要 N 元 INTERSECT 里有任意一个 DISTINCT，就等价于全 DISTINCT（交集后再去重 = 先去重再交集）。

### 8.4 DISTINCT vs ALL：contextualize 期定下的差异

| 维度 | `UNION DISTINCT` | `UNION ALL` |
|---|---|---|
| 压入的 `QueryLevel` | `SC_UNION_DISTINCT` | `SC_UNION_ALL` |
| 造出的 AST 类 | **完全相同**：`Query_term_union` | **完全相同** |
| `merge_descendants` 分支 | `if (m_is_distinct)`：无条件摊平 | `else`：摊平前查 `is_top_level_union_all()` |
| `m_last_distinct` | 每 push 一个孩子就递增 | 直接 push 不改；只在合并下层时继承 |

⇒ **关键结论**：DISTINCT 与 ALL 在 parse 期**不产生不同的类、不产生不同的树形**，只产生两个整数下标 + 一次"摊不摊平"的选择。"去重"动作本身完全不发生在 parse 期，只是**被标记**到 `Query_term_set_op` 上留给执行器。

### 8.5 ★ 完整例子：`SELECT 1 UNION ALL SELECT 2 UNION SELECT 3 ORDER BY 1`

**PT 树**（文法是**左递归** ⇒ **左结合**）：

```
PT_select_stmt
  └── PT_query_expression
         ├── m_body = PT_union(is_distinct = TRUE)        ← 外层 UNION（默认 DISTINCT）
         │       ├── m_lhs = PT_union(is_distinct = FALSE)   ← 内层 UNION ALL
         │       │        ├── m_lhs = PT_query_specification "SELECT 1"
         │       │        └── m_rhs = PT_query_specification "SELECT 2"
         │       └── m_rhs = PT_query_specification "SELECT 3"
         ├── m_order = PT_order (ORDER BY 1)
         └── m_limit = nullptr
```

**`m_stack` 逐步状态**（`QB1/2/3` 为叶子块，`U_all` 内层 union，`U_out` 外层）：

| # | 触发点 | `m_stack` | `pc->select` |
|---|---|---|---|
| 0 | `Parse_context pc(thd, ...)` | `[SC_TOP]` | QB1 |
| 1 | `PT_query_expression` push | `[SC_TOP, QE(has_order=true)]` | QB1 |
| 2 | 外层 `PT_union` ① | `[SC_TOP, QE, U_DISTINCT]` | QB1 |
| 3 | 内层 `PT_union` ① | `[..., U_DISTINCT, U_ALL]` | QB1 |
| 4–6 | QB1/QB2 各自 push 自己 | `U_ALL.elts=[QB1,QB2]` | QB1→**QB2** |
| 7 | 内层收尾：`U_all`（不摊平，两孩子都是叶子）⇒ `m_children=[QB1,QB2]`、`m_last_distinct=0`；造 **QB_fake1** | `U_DISTINCT.elts=[U_all]` | QB2 |
| 8–9 | 外层 ③ 前进到 QB3；QB3 push 自己 | `U_DISTINCT.elts=[U_all,QB3]` | **QB3** |
| 10 | 外层 `merge_descendants` **摊平** | 见下 | QB3 |
| 11 | 造 **QB_fake2**，push 自己 | `QE.elts=[U_out]` | QB3 |
| 12 | `PT_query_expression` case QT_UNION：挂 ORDER BY、临时 `pc->select = QB_fake2` + contextualize + 复原 | `SC_TOP.elts=[U_out]` | QB3 |
| 13 | `finalize_query_expression()`：`assert(elts.size()==1)` → `pushdown_limit_order_by()` → `set_query_term(U_out)` | `[]` | QB3 |

**第 10 步摊平细节**（外层 `m_is_distinct == true` ⇒ 走 `if` 分支）：

```
elt = U_all :  (1)(2)(3) 都不成立；QB_fake1 无 select_limit ⇒ (4) 成立，合并！
               first_distinct = lower->m_children.size() = 2
               last_distinct = 0 + 2 - 1 = 1 ; count = 2 ; children = [QB1, QB2]
elt = QB3   :  (1) 成立（叶子），不合并
               count = 3 ; last_distinct = 3 - 1 = 2 ; children = [QB1, QB2, QB3]
⇒ U_out->m_last_distinct = 2 = N-1  ⇒ has_mixed_distinct_operators() == false
⇒ 三个操作数整体去重 ✔（(QB1 ALL QB2) DISTINCT QB3 语义上就是全集去重）
```

**最终 `Query_term` 树**：

```
        Query_expression (m_query_term ─────────┐)
   slave 链:  QB1 ──next──► QB2 ──next──► QB3   │
              (SELECT 1)   (SELECT 2)   (SELECT 3)│
                    ▲                             │
   ┌────────────────┴──────────────────┐          │
   │  Query_term_union (U_out)          │◄─────────┘
   │  m_last_distinct  = 2              │
   │  m_first_distinct = 2              │
   │  m_block ──► QB_fake2 [ORDER BY 1] │
   │  m_children = [QB1(0), QB2(1), QB3(2)] │
   └────────────────────────────────────┘
   （内层 U_all 已被摊平合并，QB_fake1 成为孤儿）
```

### 8.6 `can_absorb_order_and_limit()` 为何返回 `false`

```cpp
// [P1] 三处对比
// PT_query_specification:  return true;      ← 单个 QB，ORDER BY 落在自己身上是安全的
// PT_query_expression:     m_body->is_set_operation() ? false : ...判断
// PT_set_operation:        return false;     ← ★ 一律不下推
```

**原因**：集合运算有**多个** QB，没有"当前 QB"可用；且 ORDER BY / LIMIT 语义上必须作用于**整个结果集**。因此走 `contextualize_order_and_limit()` 的 `else` 分支：临时 `pc->select = nullptr` 并 `push_context`，在**新建的后处理块**上 contextualize。

```
can_absorb == true  → 直接在 pc->select（当前 QB）上 contextualize
                      ⇒ ORDER BY 落在 QB 自己的 order_list
can_absorb == false → pc->select 置空 → push_context → 在 post-processing block 上
                      contextualize → pop_context → 还原
                      ⇒ ORDER BY 落在 Query_term_set_op::m_block 上
```

---

## 九、DML 语句根节点的 `make_cmd()`

> **边界声明**：INSERT/UPDATE/DELETE 的**执行语义**（`fill_record`、`write_row`、`Query_result_update` 等）属 [`10_dml.md`](10_dml.md)，本节只讲 **make_cmd 如何把 PT 填充进 LEX / Query_block / Sql_cmd**。
>
> 本节源码坐标：`[P1] sql/parse_tree_nodes.h` · `[P2] sql/parse_tree_nodes.cc` · `[P5] sql/parse_tree_helpers.h` · `[P8] sql/sql_lex.h` · `[P16] sql/sql_insert.h` · `[P17] sql/sql_update.h` · `[P18] sql/sql_delete.h`

**先说一个 8.0 的重要事实**：老版本的 `lex->value_list` / `lex->update_value_list` 槽位**在 8.0.39 的 `LEX` 里已不存在**——DML 的"值列表"现在一律**挂在 `Sql_cmd` 对象上**（`Sql_cmd_insert_base::insert_many_values` / `Sql_cmd_update::update_value_list`）。这是"命令对象化"重构的结果，也是理解 DML 与 SELECT `make_cmd` 差异的关键。

### 9.1 基线回顾：`PT_select_stmt::make_cmd()`

⇒ **它是唯一无条件调用 `finalize_query_expression()` 的根节点**——把 `Query_term` 树的根挂到 `Query_expression::m_query_term`，并做 `pushdown_limit_order_by()` + 结构校验。（完整代码见 3.2。）

### 9.2 ★ `PT_insert::make_cmd()`：DML 中最复杂的一个

```cpp
// [P2]（节选，保留主干与关键分支）
Sql_cmd *PT_insert::make_cmd(THD *thd) {
  LEX *const lex = thd->lex;
  Parse_context pc(thd, lex->current_query_block());   // ① ★ 不新建 QB，复用顶层块

  // ② 语法归一化：INSERT .. VALUES ROW(), ROW() .. 被等价改写成 VALUES (), () ..
  if (has_query_block() && insert_query_expression->is_table_value_constructor()) {
    row_value_list = insert_query_expression->get_row_value_list();
    insert_query_expression = nullptr;                 // ★ 改写后 has_query_block() 变 false
  }

  // ③ 定 sql_command + duplicates
  if (is_replace) {
    lex->sql_command = has_query_block() ? SQLCOM_REPLACE_SELECT : SQLCOM_REPLACE;
    lex->duplicates = DUP_REPLACE;
  } else {
    lex->sql_command = has_query_block() ? SQLCOM_INSERT_SELECT : SQLCOM_INSERT;
    lex->duplicates = DUP_ERROR;
    lex->set_ignore(ignore);
  }

  // ④ 目标表进 Query_block::m_table_list（TL_OPTION_UPDATING）+ 设锁
  Yacc_state *yyps = &pc.thd->m_parser_state->m_yacc;
  if (!pc.select->add_table_to_list(thd, table_ident, nullptr, TL_OPTION_UPDATING,
                                    yyps->m_lock_type, yyps->m_mdl_type, nullptr,
                                    opt_use_partition))
    return nullptr;
  pc.select->set_lock_for_tables(lock_option);

  // ⑤ 字段列表 (col1,...) → Item 化
  if (column_list->contextualize(&pc)) return nullptr;

  if (has_query_block()) {                             // ⑥ INSERT ... SELECT
    SQL_I_List<Table_ref> save_list;
    Query_block *const save_query_block = pc.select;
    save_query_block->m_table_list.save_and_clear(&save_list);   // ⑥a 摘走目标表
    if (insert_query_expression->contextualize(&pc)) return nullptr;
    if (pc.finalize_query_expression()) return nullptr;          // ⑥b ★ 只有这条支路调
    save_query_block->m_table_list.push_front(&save_list);       // ⑥c 目标表放回队首
    lex->bulk_insert_row_cnt = 0;
  } else {                                             // ⑦ INSERT ... VALUES
    pc.select->parsing_place = CTX_INSERT_VALUES;
    if (row_value_list->contextualize(&pc)) return nullptr;
    assert(pc.select->parsing_place == CTX_INSERT_VALUES);
    pc.select->parsing_place = CTX_NONE;
    lex->bulk_insert_row_cnt = row_value_list->get_many_values().size();
  }

  // ⑧ INSERT ... AS new ... 给 VALUES 造派生表（略）
  // ⑨ ON DUPLICATE KEY UPDATE col=x, ...
  if (opt_on_duplicate_column_list != nullptr) {
    lex->duplicates = DUP_UPDATE;                      // ★ 覆盖 ③ 里设的 DUP_ERROR
    Table_ref *first_table = lex->query_block->get_table_list();
    if (first_table->lock_descriptor().type == TL_WRITE_CONCURRENT_DEFAULT)
      first_table->set_lock({TL_WRITE_DEFAULT, THR_DEFAULT});   // ODKU 需要读后写
    pc.select->parsing_place = CTX_INSERT_UPDATE;      // ★ 另一个解析上下文
    if (opt_on_duplicate_column_list->contextualize(&pc) ||
        opt_on_duplicate_value_list->contextualize(&pc))
      return nullptr;
    pc.select->parsing_place = CTX_NONE;
  }

  if (opt_hints != nullptr && opt_hints->contextualize(&pc)) return nullptr;

  // ⑩ 产出命令对象（按 VALUES / SELECT 分支造不同子类）
  Sql_cmd_insert_base *sql_cmd = has_query_block()
      ? new (thd->mem_root) Sql_cmd_insert_select(is_replace, lex->duplicates)
      : new (thd->mem_root) Sql_cmd_insert_values(is_replace, lex->duplicates);
  if (sql_cmd == nullptr) return nullptr;

  // ⑪ ★ 把 PT 侧收集到的 Item 列表搬进 Sql_cmd（不再放 LEX）
  if (!has_query_block()) {
    sql_cmd->insert_many_values = row_value_list->get_many_values();
    sql_cmd->values_table = values_table;
    sql_cmd->values_column_list = opt_values_column_list;
  }
  sql_cmd->insert_field_list = column_list->value;
  if (opt_on_duplicate_column_list != nullptr) {
    sql_cmd->update_field_list = opt_on_duplicate_column_list->value;
    sql_cmd->update_value_list = opt_on_duplicate_value_list->value;
  }
  return sql_cmd;
}
```

**配套小节点**：`PT_item_list` 是一维 `Item` 列表（被 INSERT 复用三次：插入列 / ODKU 列 / ODKU 值）；行值是**二维**的 `PT_insert_values_list`：

```cpp
// [P1]
class PT_insert_values_list : public Parse_tree_node {
  mem_root_deque<List_item *> many_values;   // ★ deque<deque<Item*>>：一行一个 List_item
 public:
  bool contextualize(Parse_context *pc) override;
};
// [P2]
bool PT_insert_values_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  for (List_item *item_list : many_values)          // 逐行
    for (auto it = item_list->begin(); it != item_list->end(); ++it)   // 逐列
      if ((*it)->itemize(pc, &*it)) return true;    // ★ 引用遍历，原地替换
  return false;
}
```

**剖析要点**：

1. **不新建 `Query_block`**。`INSERT ... SELECT` 的 SELECT 部分**复用同一个顶层 QB**，只在期间用 `save_and_clear` / `push_front` 把目标表暂时挪出/挪回 `m_table_list`，避免 SELECT 的 `FROM` 表与目标表混在同一链表里解析。
2. **只有 `INSERT ... SELECT` 支路调 `finalize_query_expression()`**——只有它可能构造 `Query_term` 树（UNION 等）；`INSERT ... VALUES` 没有嵌套查询表达式，`m_query_term` 直接用 `LEX::new_top_level_query()` 预置的叶子 QB。
3. **`parsing_place` 两段切换**：`CTX_INSERT_VALUES`（VALUES 行值）、`CTX_INSERT_UPDATE`（ODKU 表达式，可引用 `VALUES(col)`）。
4. **ODKU 会覆盖 `lex->duplicates`**（`DUP_ERROR` → `DUP_UPDATE`）并修正锁。

### 9.3 `PT_update::make_cmd()`

```cpp
// [P2]（节选）
Sql_cmd *PT_update::make_cmd(THD *thd) {
  LEX *const lex = thd->lex;
  Query_block *const select = lex->current_query_block();
  Parse_context pc(thd, select);
  lex->duplicates = DUP_ERROR;
  lex->set_ignore(opt_ignore);
  if (contextualize_safe(&pc, m_with_clause)) return nullptr;

  if (contextualize_array(&pc, &join_table_list)) return nullptr;   // ① FROM / JOIN

  select->parsing_place = CTX_UPDATE_VALUE;                          // ② 切上下文
  if (column_list->contextualize(&pc) || value_list->contextualize(&pc))
    return nullptr;
  select->fields = column_list->value;        // ★ 左值列列表进 Query_block::fields
  assert(select->parsing_place == CTX_UPDATE_VALUE);
  select->parsing_place = CTX_NONE;

  const bool is_multitable = select->m_table_list.elements > 1;      // ③ 多表判定（运行时）
  lex->sql_command = is_multitable ? SQLCOM_UPDATE_MULTI : SQLCOM_UPDATE;
  select->set_lock_for_tables(opt_low_priority);

  if (opt_where_clause != nullptr &&
      opt_where_clause->itemize(&pc, &opt_where_clause)) return nullptr;
  select->set_where_cond(opt_where_clause);
  if (opt_order_clause != nullptr && opt_order_clause->contextualize(&pc)) return nullptr;
  if (opt_limit_clause != nullptr) { /* itemize + select_limit + BINLOG_UNSAFE */ }
  if (opt_hints != nullptr && opt_hints->contextualize(&pc)) return nullptr;

  return new (thd->mem_root) Sql_cmd_update(is_multitable, &value_list->value);
}
```

**★ 赋值结构落点**（本节核心问答）：`UPDATE t SET a = 1, b = 2` 的 `field = expr` 被拆成**两个平行的一维 `PT_item_list`**（yacc 里同步 push），parse 期分别落到两个不同地方：

```
         PT_update
        ┌─────┴──────┐
  column_list    value_list
   (PT_item_list) (PT_item_list)
        │              │
        ▼              ▼
  select->fields   Sql_cmd_update::update_value_list
  （Item_field 列表）（Item 表达式列表）
        │              │
        │ prepare 期    │
        ▼              ▼
  TABLE::write_set   每行的求值结果
  （位图：哪些列要写）
```

⇒ **`TABLE::write_set` 不是 parse 期填的**。`make_cmd` 只把左值 `Item` 存进 `Query_block::fields`；真正的位图与 `field=expr` 配对校验发生在 `Sql_cmd_update::prepare_inner()`（属 [`10_dml.md`](10_dml.md)）。

### 9.4 `PT_delete::make_cmd()`

```cpp
// [P2]（节选）
Sql_cmd *PT_delete::make_cmd(THD *thd) {
  LEX *const lex = thd->lex;
  Query_block *const select = lex->current_query_block();
  Parse_context pc(thd, select);
  lex->sql_command = is_multitable() ? SQLCOM_DELETE_MULTI : SQLCOM_DELETE;
  lex->set_ignore(opt_delete_options & DELETE_IGNORE);
  select->init_order();
  if (contextualize_safe(&pc, m_with_clause)) return nullptr;

  if (is_multitable()) {                                   // ① 逐个目标表
    for (Table_ident **i = table_list.begin(); i != table_list.end(); ++i)
      if (add_table(&pc, *i)) return nullptr;
  } else if (add_table(&pc, table_ident)) return nullptr;

  if (is_multitable()) {                                   // ② ★ 表链表"换手"
    select->m_table_list.save_and_clear(&delete_tables);
    lex->query_tables = nullptr;                           //   让 FROM/USING 重新从零收表
    lex->query_tables_last = &lex->query_tables;
  } else {
    select->m_table_nest.push_back(select->get_table_list());
  }
  /* ... yyps 锁类型改为 TL_READ_DEFAULT / MDL_SHARED_READ ... */

  if (is_multitable()) {                                   // ③ USING / JOIN 部分
    if (contextualize_array(&pc, &join_table_list)) return nullptr;
    pc.select->context.table_list =
        pc.select->context.first_name_resolution_table = pc.select->get_table_list();
  }
  /* ... where / order / limit ... */

  if (is_multitable() && multi_delete_link_tables(&pc, &delete_tables)) return nullptr;
  return new (thd->mem_root) Sql_cmd_delete(is_multitable(), &delete_tables);
}
```

- `is_multitable()` 的实现是 `table_ident == nullptr`（`[P1]`）——**由 yacc 走哪个构造函数决定**，不是运行时数表个数（`PT_update` 则是数 `m_table_list.elements`）。
- `save_and_clear(&delete_tables)` 是 DELETE 独有的"**表链表换手**"：`DELETE t1, t2 FROM t1 JOIN t2 ...` 里 `t1, t2` 先收进 `m_table_list`，再整条搬到 `delete_tables`，让 `FROM/USING` 的表从空链表重新收；最后 `multi_delete_link_tables()` 做名字匹配并回填 `updating` / 锁。

### 9.5 ★ 四者 `make_cmd()` 对比

```
 PT_select_stmt                        PT_insert
 ─────────────                         ────────
 pc(lex->current_query_block())        pc(lex->current_query_block())
 sql_command = SELECT / DO             sql_command = INSERT|REPLACE|*_SELECT
   │                                     │
   ├─ m_qe->contextualize()              ├─ 目标表 add_table_to_list()
   │    └─ PT_query_expression           ├─ column_list->contextualize()
   │         └─ query_specification      ├─ has_query_block() ?
   │            / PT_union ...           │    YES: save_and_clear 表链
   ├─ m_into->contextualize()            │          insert_qe->contextualize()
   ├─ finalize_query_expression() ★YES   │          finalize_query_expression() ★YES
   │                                     │          push_front 还原表链
   └─ Sql_cmd_select(lex->result)        │    NO : parsing_place=CTX_INSERT_VALUES
                                         │          row_value_list->contextualize()
                                         ├─ VALUES 派生表（AS new）
                                         ├─ ODKU: DUP_UPDATE + CTX_INSERT_UPDATE
                                         └─ Sql_cmd_insert_values / _select
                                              ├─ insert_field_list  ← column_list
                                              └─ update_field/value_list ← ODKU

 PT_update                             PT_delete
 ────────                              ─────────
 pc(lex->current_query_block())        pc(lex->current_query_block())
   │                                     │
   ├─ join_table_list                    ├─ add_table() ×N
   ├─ parsing_place=CTX_UPDATE_VALUE     ├─ 多表: save_and_clear(&delete_tables) ★
   ├─ column_list + value_list itemize   │        + 清空 lex->query_tables
   │    ├─ 列 → select->fields  ★        ├─ join_table_list
   │    └─ 值 → 暂存 PT 侧               ├─ where / order / limit
   ├─ is_multitable = m_table_list       ├─ multi_delete_link_tables() ★
   │       .elements > 1                 │        （名字匹配 + 置写锁）
   └─ Sql_cmd_update(&value_list->value) └─ Sql_cmd_delete(&delete_tables)
   (无 finalize)                         (无 finalize)
```

| 维度 | `PT_select_stmt` | `PT_insert` | `PT_update` | `PT_delete` |
|---|---|---|---|---|
| `sql_command` | `SELECT` / `DO` | `INSERT` / `INSERT_SELECT` / `REPLACE*` | `UPDATE` / `UPDATE_MULTI` | `DELETE` / `DELETE_MULTI` |
| 新建 `Query_block`? | 否（子查询/集合运算内部才建） | **否**（SELECT 支路复用同一个） | 否 | 否 |
| 目标表如何进 AST | — | `add_table_to_list(TL_OPTION_UPDATING)` | `join_table_list` + `set_lock_for_tables` | `add_table()` + 多表 `save_and_clear` |
| 填 `Query_block::fields`? | 由 `PT_query_specification` 填 select list | **不填**（列清单进 `Sql_cmd`） | **填**：`select->fields = column_list->value` | 不填 |
| 值列表落点 | — | `Sql_cmd_insert_base::insert_many_values` | `Sql_cmd_update::update_value_list` | —（`delete_tables` 链表） |
| `parsing_place` 切换 | `CTX_SELECT_LIST` 等 | `CTX_INSERT_VALUES` / `CTX_INSERT_UPDATE` | `CTX_UPDATE_VALUE` | 无切换 |
| 调 `finalize_query_expression()`? | **是（无条件）** | **仅 `INSERT ... SELECT` 支路** | **否** | **否** |
| 多表判定 | — | — | `m_table_list.elements > 1`（运行时） | `table_ident == nullptr`（构造时） |
| 产出 | `Sql_cmd_select` / `Sql_cmd_do` | `Sql_cmd_insert_values` / `_select` | `Sql_cmd_update` | `Sql_cmd_delete` |

**★ 一句话总结差异**：`PT_select_stmt` 的 `make_cmd` 本质是"contextualize 一棵查询表达式 + 封树（finalize）"；三个 DML 的 `make_cmd` 是"**自己手工搭那棵查询块**"——它们没有 `PT_query_expression` 包壳，直接一步步填 `Query_block` 的各个槽位（`m_table_list` / `fields` / `where` / `order_list` / `select_limit`），把 DML 特有的"目标表 / 赋值对 / 值行"搬到新建的 `Sql_cmd` 上，因此**都不需要（也大多不能）调用 `finalize_query_expression()`**——唯一例外是 `INSERT ... SELECT`，因为它体内真的挂着一棵查询表达式。

### 9.6 DDL 根节点与 `Table_ddl_parse_context`（呼应 7.1.7）

```cpp
// [P2] PT_create_table_stmt::make_cmd（节选）
  Parse_context pc(thd, lex->current_query_block());
  Table_ref *table = pc.select->add_table_to_list(thd, table_name, nullptr,
                                                  TL_OPTION_UPDATING, TL_WRITE, MDL_SHARED);
  table->open_strategy = Table_ref::OPEN_FOR_CREATE;
  lex->create_info = &m_create_info;
  // ★ 派生出 DDL 专用上下文：多了 create_info / alter_info / key_create_info
  Table_ddl_parse_context pc2(thd, pc.select, &m_alter_info);
  ...
  // 列定义 / 索引 / 约束等元素，全部用 pc2（而不是 pc）contextualize
  for (auto element : *opt_table_element_list)
    if (element->contextualize(&pc2)) return nullptr;
```

⇒ DDL 的 `make_cmd` 除了填 LEX / Query_block，还要**并行维护 `HA_CREATE_INFO` + `Alter_info`**；为此引入 `Table_ddl_parse_context`（派生自 `Parse_context`，新增三个落点指针）——这是 DDL 与 DML 在"填充侧"最大的结构性差别。

---

## 十、`PTI_*` 解析期占位 Item 全景（36 个类逐个讲透）

> 本章源码坐标：`[P5] sql/parse_tree_items.cc` · `[P6] sql/parse_tree_items.h` · `[P9] sql/parse_tree_helpers.h` · `[P10] sql/sql_yacc.yy` · `[P12] sql/item_create.cc` · `[P13] sql/item_func.cc`
>
> 第七章已讲 `Parse_tree_item` 基类、`Item::itemize()` 基类实现、`res` 二级指针、二次 itemize、**五种变身模式**框架与 7 个代表例子（本章不重复展开，只在总图中复用编号 ①②③④⑤）。

### 10.1 全景：36 个类完整清单

硬证据：`grep '^class PTI_' sql/parse_tree_items.h` ⇒ **36**；`grep 'NEW_PTN PTI_' sql/sql_yacc.yy` ⇒ 43 处实例化点（去重后 33 个类）。

⇒ **36 = 33 个可实例化类 + 2 个抽象中间层（`PTI_text_literal`、`PTI_context`）+ `PTI_having`**（后者在 yacc 里用普通 `new` 而非 `NEW_PTN`：`$$= new PTI_having(@$, $2);`）。

> ⚠️ **一处常见误解更正**：`PTI_simple_ident_q_3d` **不是**抽象中间层——它在 yacc 的 `ident '.' ident '.' ident` 规则里被直接实例化，同时又是 `_q_2d` 的父类，属**"可实例化的中间层"**。真正的抽象中间层只有 `PTI_text_literal` 与 `PTI_context`（构造均为 `protected`）。

| # | 类名 | 继承自 | 变身目标 | 模式 | 说明 |
|---|---|---|---|---|---|
| 1 | `PTI_truth_transform` | `Parse_tree_item` | `expr->truth_transformer()` 返回值 / `Item_func_not` / `Item_func_truth` | ③ | IS TRUE / IS NOT TRUE / IS FALSE / NOT |
| 2 | `PTI_comp_op` ★ | `Parse_tree_item` | `Item_func_eq/ge/gt/le/lt/ne` | ② | 二元比较（第七章代表） |
| 3 | `PTI_comp_op_all` | `Parse_tree_item` | `Item_in_subselect` / `Item_allany_subselect` + `Item_func_{not,nop}_all` | ③⑤ | `= ANY` / `<> ALL` |
| 4 | `PTI_simple_ident_ident` | `Parse_tree_item` | `Item_splocal` / `Item_field` / `Item_ref` | ② | 裸列名（先查 SP 变量） |
| 5 | `PTI_simple_ident_q_3d` ★ | `Parse_tree_item` | `Item_field` / `Item_ref` | ② | `db.tbl.col`；`_q_2d` 的父类 |
| 6 | `PTI_simple_ident_q_2d` | `_q_3d` | `Item_trigger_field` 或复用父类 | ② | `tbl.col`；触发器 `NEW./OLD.` 特判 |
| 7 | `PTI_simple_ident_nospvar_ident` | `Parse_tree_item` | `Item_field` / `Item_ref` | ② | 裸列名，**禁止**解析为 SP 变量 |
| 8 | `PTI_function_call_nonkeyword_now` | `Item_func_now_local` | `this` | ④ | `NOW` / `LOCALTIME` |
| 9 | `PTI_function_call_nonkeyword_sysdate` | `Parse_tree_item` | `Item_func_sysdate_local` **或** `Item_func_now_local` | ② | 由 `sysdate_is_now` 决定 |
| 10 | `PTI_udf_expr` | `Parse_tree_item` | `expr`（改写 `item_name`） | ① | UDF 实参带属性名 |
| 11 | `PTI_function_call_generic_ident_sys` ★ | `Parse_tree_item` | native/UDF/SP 三级 builder | ③ | `func(args)`（第七章代表） |
| 12 | `PTI_function_call_generic_2d` | `Parse_tree_item` | `Item_func_sp` | ③ | `db.func(args)` |
| 13 | **`PTI_text_literal`** | `Item_string` | —（无 `itemize()`） | 抽象 | 中间层：只存 `is_7bit` + `literal` |
| 14–16 | `PTI_text_literal_{text_string, nchar_string, underscore_charset}` | `PTI_text_literal` | `this` | ④ | `'abc'` / `N'abc'` / `_utf8'abc'` |
| 17 | `PTI_text_literal_concat` | `PTI_text_literal` | `head_str`（链表头） | ① | `'a' 'b'` 相邻字面量拼接 |
| 18 | `PTI_temporal_literal` | `Parse_tree_item` | `Item_{date,time,datetime}_literal` | ③ | `DATE/TIME/TIMESTAMP '...'` |
| 19–20 | `PTI_literal_underscore_charset_{hex,bin}_num` | `Item_string` | `this` | ④ | `_utf8 0x4142` / `_utf8 b'0101'` |
| 21 | `PTI_variable_aux_set_var` | `Item_func_set_user_var` | `this` + 入 `lex->set_var_list` | ④ | `@a := expr` |
| 22 | `PTI_user_variable` ★ | `Item_func_get_user_var` | `this` | ④ | `@a`（第七章代表） |
| 23 | `PTI_get_system_variable` | `Parse_tree_item` | `Item_func_get_system_var` | ③ | `@@global.x` / `@@x` |
| 24 | `PTI_count_sym` | `Item_sum_count` | `this`（补 `args[0]`） | ④ | `COUNT(*)` |
| 25 | `PTI_in_sum_expr` | `Parse_tree_item` | `expr` | ① | 临时抬高 `in_sum_expr` 的壳 |
| 26–27 | `PTI_singlerow_subselect` ★ / `PTI_exists_subselect` | `Parse_tree_item` | `Item_singlerow_subselect` / `Item_exists_subselect` | ⑤ | 标量子查询 / EXISTS |
| 28 | `PTI_odbc_date` | `Parse_tree_item` | 成功→三种日期字面量；失败→**`expr` 原样** | ③① | `{d '2020-01-01'}` |
| 29 | `PTI_handle_sql2003_note184_exception` | `Parse_tree_item` | `Item_in_subselect` / `Item_func_eq` / `_ne` | ⑤ | `x IN (single_expr)` 单值特例 |
| 30 | `PTI_expr_with_alias` ★ | `Parse_tree_item` | `expr`（设 `item_name`） | ① | `expr AS alias` |
| 31–32 | `PTI_int_splocal` / `PTI_limit_option_ident` | `Parse_tree_item` / `_int_splocal` | `Item_splocal` | ② | SP 整型变量 / `LIMIT ident` |
| 33 | `PTI_limit_option_param_marker` | `Parse_tree_item` | `param_marker` | ① | `LIMIT ?` |
| 34 | **`PTI_context`** ★ | `Parse_tree_item` | `expr`（`make_condition` 后） | 抽象① | 切 `parsing_place` 的壳 |
| 35–36 | `PTI_where` / `PTI_having` | `PTI_context` | 继承父类 | ① | 只注入 `CTX_WHERE` / `CTX_HAVING` |

（★ = 第七章已作为代表讲过。）

**统计**：① 原样透传 7 ｜② 换新对象 8 ｜③ 查表/派发 7 ｜④ 就地初始化 9 ｜⑤ 子查询跨界 3 ｜抽象中间层 2 ＝ **36**。

### 10.2 按用途分组（要点）

**① 字面量：为什么也需要占位符**——`'abc'` 的字符集不是语法决定的，而是**连接状态**决定的，语法阶段拿不到 `THD`：

```cpp
// [P5] 'abc'：字符集要等到 itemize 才能确定（依赖 thd 的连接级变量）
bool PTI_text_literal_text_string::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  THD *thd = pc->thd;
  const CHARSET_INFO *cs_con = thd->variables.collation_connection;
  const CHARSET_INFO *cs_cli = thd->variables.character_set_client;
  uint repertoire = is_7bit && my_charset_is_ascii_based(cs_cli)
                        ? MY_REPERTOIRE_ASCII : MY_REPERTOIRE_UNICODE30;
  if (thd->charset_is_collation_connection ||
      (repertoire == MY_REPERTOIRE_ASCII && my_charset_is_ascii_based(cs_con)))
    tmp = literal;
  else {
    if (thd->convert_string(&tmp, cs_con, literal.str, literal.length, cs_cli))
      return true;                                    // 字符集转换失败
  }
  init(tmp.str, tmp.length, cs_con, DERIVATION_COERCIBLE, repertoire);
  return false;
}
```

⇒ 这就是"字面量为什么也是 `PTI_*`"——`_utf8'abc'`（字符集语法期已知）与 `'abc'`（字符集连接期才知）走不同分支。

**② `simple_ident` 四兄弟**共享同一判断（HAVING 且不在聚合内 ⇒ `Item_ref` 而非 `Item_field`），差别只在前缀段数与是否查 SP 变量表。

**③ 变量**：`@a := expr` 的 `*res` 保持 `this`，**但自己又登记进 `lex->set_var_list`**：

```cpp
// [P5]
bool PTI_variable_aux_set_var::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  LEX *lex = pc->thd->lex;
  if (!lex->parsing_options.allows_variable) {   // 视图定义里不允许用户变量
    my_error(ER_VIEW_SELECT_VARIABLE, MYF(0));
    return true;
  }
  lex->set_uncacheable(pc->select, UNCACHEABLE_RAND);
  return lex->set_var_list.push_back(this);      // ★ *res 仍是 this，但另作了登记
}
```

**④ `PTI_count_sym`**：把 `*` 换成常量 `0` 再走 `Item_sum::itemize()`——`COUNT(*)` 实为 `COUNT(0)`：

```cpp
bool PTI_count_sym::itemize(Parse_context *pc, Item **res) {
  args[0] = new (pc->mem_root) Item_int(int32{0}, 1);
  if (args[0] == nullptr) return true;
  return super::itemize(pc, res);
}
```

**⑤ `PTI_in_sum_expr`**（"加-递归-减-剥壳"四段式）：

```cpp
bool PTI_in_sum_expr::itemize(Parse_context *pc, Item **res) {
  pc->select->in_sum_expr++;                          // ★ 进入聚合参数前 +1
  if (super::itemize(pc, res) || expr->itemize(pc, &expr)) return true;
  pc->select->in_sum_expr--;                          // ★ 出来 -1
  *res = expr;                                        // 剥壳
  return false;
}
```

### 10.3 重点深剖

#### 10.3.1 `PTI_simple_ident_q_2d`：★ 全族唯一的"跳级调用"

```cpp
// [P5]
bool PTI_simple_ident_q_2d::itemize(Parse_context *pc, Item **res) {
  THD *thd = pc->thd;
  LEX *lex = thd->lex;
  sp_head *sp = lex->sphead;

  if (sp && sp->m_type == enum_sp_type::TRIGGER &&        // ★ 只在触发器体里
      (!my_strcasecmp(system_charset_info, table, "NEW") ||
       !my_strcasecmp(system_charset_info, table, "OLD"))) {
    if (Parse_tree_item::itemize(pc, res)) return true;   // ★★ 直呼曾祖 Parse_tree_item::itemize，
                                                          //    有意跳过父类 _q_3d 的
                                                          //    no_table_names_allowed 检查与 Item_field 构造
    bool new_row = (table[0] == 'N' || table[0] == 'n');
    if (sp->m_trg_chistics.event == TRG_EVENT_INSERT && !new_row) {
      my_error(ER_TRG_NO_SUCH_ROW_IN_TRG, MYF(0), "OLD", "on INSERT");  // INSERT 没有 OLD 行
      return true;
    }
    if (sp->m_trg_chistics.event == TRG_EVENT_DELETE && new_row) {
      my_error(ER_TRG_NO_SUCH_ROW_IN_TRG, MYF(0), "NEW", "on DELETE");  // DELETE 没有 NEW 行
      return true;
    }
    const bool read_only =                                // ★ 只有 BEFORE INSERT 的 NEW 可写
        !(new_row && sp->m_trg_chistics.action_time == TRG_ACTION_BEFORE);
    Item_trigger_field *trg_fld = new (pc->mem_root)
        Item_trigger_field(POS(), new_row ? TRG_NEW_ROW : TRG_OLD_ROW, field,
                           SELECT_ACL, read_only);
    if (trg_fld == nullptr || trg_fld->itemize(pc, (Item **)&trg_fld)) return true;

    lex->sphead->m_cur_instr_trig_field_items.link_in_list(  // 登记进触发器字段链表
        trg_fld, &trg_fld->next_trg_field);
    *res = trg_fld;                             // ★ 变身目标：Item_trigger_field
  } else {
    if (super::itemize(pc, res)) return true;   // 普通 tbl.col：复用父类 _q_3d
  }
  return false;
}
```

⇒ **为什么这一族必须占位**：`tbl.col` 语法上无法区分"表列"与"触发器的 `NEW.col`"——只有 itemize 时才知道在不在编译触发器（`lex->sphead->m_type == TRIGGER`）。

#### 10.3.2 `PTI_function_call_nonkeyword_sysdate`——"为什么必须占位"的最佳范例

```cpp
// [P5]
bool PTI_function_call_nonkeyword_sysdate::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  /*
    Unlike other time-related functions, SYSDATE() is replication-unsafe
    because it is not affected by the TIMESTAMP variable. ...
  */
  THD *thd = pc->thd;
  LEX *lex = thd->lex;
  lex->set_stmt_unsafe(LEX::BINLOG_STMT_UNSAFE_SYSTEM_FUNCTION);
  if (global_system_variables.sysdate_is_now == 0)          // ★ 运行期全局系统变量决定类
    *res = new (pc->mem_root) Item_func_sysdate_local(dec);
  else
    *res = new (pc->mem_root) Item_func_now_local(dec);
  if (*res == nullptr) return true;
  lex->safe_to_cache_query = false;
  return false;
}
```

⇒ `SYSDATE(3)` 在语法树里只有一个 `uint8 dec`。它变成"真系统时间"（复制不安全）还是"受 `SET TIMESTAMP` 影响"（复制安全），取决于 **itemieze 那一刻** `global_system_variables.sysdate_is_now`。若语法期就 `new` 出具体类，改一次变量就必须重新解析——**占位符正是把"类的选择"推迟到有 `THD` 的时刻**。

#### 10.3.3 `PTI_comp_op_all`——`= ANY` 直接变成 IN 子查询

```cpp
// [P8] sql/sql_parse.cc
Item *all_any_subquery_creator(Item *left_expr,
                               chooser_compare_func_creator cmp, bool all,
                               Query_block *query_block) {
  if ((cmp == &comp_eq_creator) && !all)  //  = ANY  <=>  IN
    return new Item_in_subselect(left_expr, query_block);
  if ((cmp == &comp_ne_creator) && all)   //  <> ALL <=>  NOT IN
  {
    Item *i = new Item_in_subselect(left_expr, query_block);
    if (i == nullptr) return nullptr;
    Item *neg_i = i->truth_transformer(nullptr, Item::BOOL_NEGATED);
    if (neg_i != nullptr) return neg_i;
    return new Item_func_not(i);
  }
  Item_allany_subselect *it = new Item_allany_subselect(left_expr, cmp, query_block, all);
  if (all) return it->upper_item = new Item_func_not_all(it);  /* ALL */
  return it->upper_item = new Item_func_nop_all(it);           /* ANY/SOME */
}
```

- `x = ANY (SELECT ...)` **根本不产生** `Item_allany_subselect`，直接走半连接专用的 `Item_in_subselect`——这是优化器能选 semi-join / 物化的前提；
- `ALL` 套 `Item_func_not_all`（空集为 TRUE 的语义），`ANY/SOME` 套 `Item_func_nop_all`，`upper_item` 是"外层可见的那个 Item"；
- ★ `cmp` 是**函数指针**，所以只能**在 itemize 期做指针比较**决定分支——语法期只有指针没有语义。

#### 10.3.4 `PTI_truth_transform`——由虚函数 `truth_transformer()` 吸收

```cpp
// [P5]
static Item *change_truth_value_of_condition(Parse_context *pc, Item *expr,
                                             Item::Bool_test truth_test) {
  if (!expr->is_bool_func()) {
    expr = make_condition(pc, expr);          // ★ 非布尔先补成 `expr <> 0`
    if (expr == nullptr) return nullptr;
  }
  Item *changed = expr->truth_transformer(pc->thd, truth_test);
  if (changed != nullptr) return changed;     // ★ 表达式自己能吸收 → 直接复用/换新
  if (truth_test == Item::BOOL_NEGATED)
    return new (pc->mem_root) Item_func_not(expr);
  return new (pc->mem_root) Item_func_truth(expr, truth_test);   // 没人吸收才新建
}
```

`truth_transformer()` 是 `Item` 上的虚函数（默认 `return nullptr`），**各类自己决定能否吸收**：`Item_func_truth` 用 5×5 真值表合并两次 IS 测试（**连对象都不换**）；`Item_func_not` 直接返回内层；`Item_cond_and/or` 用德摩根律展开；比较函数用 `negated_item()`；`nop_all/not_all` 做 ANY↔ALL 互换。

#### 10.3.5 `PTI_odbc_date`——★ 唯一"变身可能失败并回退"的 `PTI_*`

```cpp
// [P5]（节选）
  *res = nullptr;                                    // ★ 先置空：透传分支靠这个哨兵判断
  if (expr->type() == Item::STRING_ITEM &&           // ① 必须是字符串字面量
      expr->collation.repertoire == MY_REPERTOIRE_ASCII) {  // ② 必须纯 ASCII
    String buf; String *tmp_str = expr->val_str(&buf);
    if (tmp_str->length() < MAX_DATE_STRING_REP_LENGTH * 4) {   // ③ 长度上限
      /* ident 就是 { } 里的 d / t / ts */
      if (ls->length == 1) { if (ls->str[0]=='d') type = MYSQL_TYPE_DATE;
                             else if (ls->str[0]=='t') type = MYSQL_TYPE_TIME; }
      else if (ls->length == 2) { if (ls->str[0]=='t' && ls->str[1]=='s')
                                    type = MYSQL_TYPE_DATETIME; }
      if (type != MYSQL_TYPE_STRING)
        *res = create_temporal_literal(pc->thd, str.ptr(), str.length(),
                                       system_charset_info, type, false);
                                       // ⑤ send_error=false：失败不报错，返回 nullptr
    }
  }
  if (*res == nullptr) *res = expr;                  // ★ 失败 → 原样透传（退化成普通字符串）
```

⇒ `{d 'garbage'}` **不会报语法错**，而是退化成普通字符串字面量。这是它同时带 ③ 与 ① 两个标记的原因。

#### 10.3.6 `PTI_text_literal_concat`——语法期只记链表，itemieze 期才拼

```cpp
// [P5]
bool PTI_text_literal_concat::itemize(Parse_context *pc, Item **res) {
  Item *tmp_head;
  if (super::itemize(pc, res) || head->itemize(pc, &tmp_head)) return true;
  assert(tmp_head->type() == STRING_ITEM);            // ★ head 经 itemize 后必是 Item_string
  Item_string *head_str = down_cast<Item_string *>(tmp_head);
  head_str->append(literal.str, literal.length);      // ★ 把"本段"追加到链表头，自身被丢弃
  if ((head_str->collation.repertoire & MY_REPERTOIRE_EXTENDED) == 0) {
    const CHARSET_INFO *cs = pc->thd->variables.collation_connection;
    head_str->collation.repertoire |=
        my_string_repertoire(cs, literal.str, literal.length);   // repertoire 逐段累积
  }
  *res = head_str;                                    // ★ 目标是 head，不是 this
  return false;
}
```

⇒ 为什么语法期不能拼：① 段的字符集要到 `head->itemize()` 之后才定；② `repertoire` 需 `THD` 才能逐段判断；③ 拼接是 O(n) 复制。结果 N 段字面量只留 **1 个 `Item_string`**，中间 N-1 个 `PTI_text_literal_concat` 全被丢弃。

### 10.4 ★ 规律性总结

#### 10.4.1 抽象中间层的判据

| 类 | 证据 | 可实例化 |
|---|---|---|
| `PTI_text_literal` | 构造在 `protected:` 下；**无** `itemize()`，只提供 `is_7bit`+`literal` | ✘ 纯抽象 |
| `PTI_context` | 构造 `protected`；`itemize()` 已实现且两子类**都不 override**，只靠构造参数注入 `CTX_*`；子类均 `final` | ✘ 纯抽象 |
| `PTI_simple_ident_q_3d` | 有自己的 `itemize()` 且被 yacc 直接实例化；成员 `db/table/field` 在 `protected:`；`_q_2d` **override** 了它 | ✔ 可实例化的中间层 |

⇒ **判据**：构造 `protected` / 无 `itemize()` ⇒ 纯抽象；有 `itemize()` 但被子类 override ⇒ 可实例化的中间层。

#### 10.4.2 "不变身" vs "必须变身"

| 类别 | 数量 | 判据 | 代表 |
|---|---|---|---|
| **不变身**（④，`*res` 仍是 `this`） | 9 | 类本身已继承**真实 Item 类**（`Item_string` / `Item_func_*` / `Item_sum_count`），语法期信息已足够，`itemize()` 只补 `init()`/上下文 | `PTI_user_variable`、`PTI_count_sym` |
| **必须变身**（①②③⑤） | 25 | 最终类依赖 itemize 期才有的信息：`THD` 状态、系统变量、`sp_pcontext`、子查询上下文 | `PTI_function_call_nonkeyword_sysdate`、`PTI_simple_ident_ident` |

⇒ 两类**共享同一句** `if (super::itemize(pc, res)) return true;`，差别只在**下一行**有没有 `*res = ...`。

#### 10.4.3 ★ "为什么需要占位符"归因表

| 归因 | 依赖的 itemize 期信息 | 涉及的类 |
|---|---|---|
| **连接/会话状态** | `collation_connection` / `character_set_client` | `PTI_text_literal_*` 系 |
| **全局系统变量** | `sysdate_is_now` | `PTI_function_call_nonkeyword_sysdate` |
| **系统变量表查找** | `System_variable_tracker` | `PTI_get_system_variable` |
| **解析位置** | `parsing_place == CTX_HAVING`、`in_sum_expr` | `simple_ident` 四兄弟、`PTI_in_sum_expr`、`PTI_context` |
| **是否在编译 SP/触发器** | `lex->sphead`、`sp_pcontext` | `PTI_simple_ident_ident`、`_q_2d`、`PTI_int_splocal` |
| **名字→实现的三级回退** | `find_native_function_builder` / `find_udf` / `find_qualified_function_builder` | `PTI_function_call_generic_*` |
| **子查询需先挂进 LEX** | `subselect->contextualize(pc)` | `PTI_singlerow_subselect`、`PTI_exists_subselect`、`PTI_comp_op_all` |
| **改写已 itemize 的子树** | `right` 可能是 `Item_singlerow_subselect` | `PTI_handle_sql2003_note184_exception` |
| **表达式吸收语义变换** | `truth_transformer()` 虚派发 | `PTI_truth_transform` |
| **"试一下，失败就还原"** | `create_temporal_literal(send_error=false)` | `PTI_odbc_date` |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

> **一句话收束**：`PTI_*` 的存在意义，就是把"**语法能确定的**"和"**必须有 `Parse_context` 才能确定的**"这两件事在时间上切开——`PTI_*` 的成员变量几乎全是 `LEX_STRING` / `POS` / `Item *` / `PT_subquery *`（语法期只负责记下名字、位置、子节点），**`itemize()` 那一行才是"对象诞生时刻"**。

> **8.0.39 核对要点**：
> - `PTI_in_subselect` **不存在**（全仓无此类）——IN 子查询由 `PTI_comp_op_all` → `all_any_subquery_creator()` 或 `PTI_handle_sql2003_note184_exception` 造出 `Item_in_subselect`；
> - `LIMIT` 相关只有两个 `PTI_*`（`PTI_limit_option_ident`、`PTI_limit_option_param_marker`）；整数常量 `LIMIT 10` 直接 `NEW_PTN Item_uint`，无 `PTI_` 包装；
> - `PTI_text_literal_underscore_charset` 等三个的 `itemize()` 是**头文件内 inline**，不在 `.cc` 里。

---

## 参考

### 知识库内相关文档

| 主题 | 归属 |
|---|---|
| `sql_yacc.yy` 文法本身（产生式写法、四个宏、Bison 构建期生成、LEX 结构） | [`04_parser.md`](04_parser.md) |
| contextualize 流程（调用链、QB/QE 创建与填充、`Query_term` 树、`PTI_*` 36 个类全谱） | [`05_contextualize.md`](05_contextualize.md) |
| Item 侧（`PTI_*` 变身之后：`fix_fields` / `val_*` / 类型推导） | [`13_item_expression.md`](13_item_expression.md) |
| 词法与 token（`lex.h` 符号表、`int_token()`、`lex_one_token()`） | [`02_lexer.md`](02_lexer.md) |
| 文法记法（如何读 `.yy`） | [`03_grammar_notation.md`](03_grammar_notation.md) |

### 外部参考

> ⚠️ 待补：按知识库规范，动笔前应深度检索 **MySQL 8.0 Reference Manual** 与相关 **WorkLog**（如 WL#7196 Parse tree 重构、WL#8081 contextualize 阶段、WL#12108 `inject_cast_nodes`），并回源码核实后补入本节。
