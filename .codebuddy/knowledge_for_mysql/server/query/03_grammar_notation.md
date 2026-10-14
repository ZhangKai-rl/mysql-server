# 03 SQL 语法记法：怎么读官方文档的 EBNF

> 本篇覆盖：MySQL 官方手册的语法说明（方括号/花括号/竖线/省略号）是什么记法、每个符号什么意思、如何和 `sql_yacc.yy` 的产生式一一对应。是 [04 Parser](04_parser.md) 的前置。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、BNF / EBNF 是什么](#一bnf--ebnf-是什么)
- [二、MySQL 官方文档的语法约定](#二mysql-官方文档的语法约定)
- [三、EBNF ↔ BNF：官方文档和 sql_yacc.yy 的互转](#三ebnf--bnf官方文档和-sql_yaccyy-的互转)
- [四、SELECT 语法逐行对照](#四select-语法逐行对照)
- [五、语法记法和词法/语法解析的对应](#五语法记法和词法语法解析的对应)

---

## 设计思想与理论基础

### SQL 语法有两份"副本"，是同一语法的两种记法

一个关键事实：**MySQL 的 SQL 语法在两个地方被完整描述过**：

| 副本 | 记法 | 读者 | 表达力 |
|------|------|------|--------|
| 官方手册《SELECT Statement》 | **EBNF 变体** | 人 | 强（`[]`/`{}`/`...` 直接表达可选/重复） |
| 源码 `sql_yacc.yy` | **BNF**（Bison 方言） | Bison 生成器 | 弱（可选/重复要写成辅助规则） |

两者**等价**（描述同一个语言），只是元符号不同。看懂官方文档 → 找到 `sql_yacc.yy` 对应产生式，是读 MySQL 解析源码的基本功。本篇就是讲这个"翻译"。

### 为什么会有两种记法

- **给人看**：EBNF 有"可选/重复"的缩写符号，一条规则顶 BNF 好几条，紧凑好读。
- **给机器看**：Bison 只认 `rule: alt1 | alt2;` 这种纯 BNF 形式，`[]`/`{}` 都要展开成辅助非终结符（`opt_xxx`/`xxx_list`）。所以 `sql_yacc.yy` 里到处是 `opt_` 前缀——那正是官方文档里 `[]` 的"展开形态"。

---

## 先有个整体印象

官方文档的语法说明（如用户贴的 SELECT 语法）和 `sql_yacc.yy` 的产生式是**一一对应**的：

```
官方手册（EBNF）                          sql_yacc.yy（BNF）
─────────────────────                    ─────────────────────
[FROM table_references ...]       ←→      opt_from_clause（sql_yacc.yy:10040）
select_expr [, select_expr] ...   ←→      select_item_list（:10166）
{ALL | DISTINCT | DISTINCTROW}    ←→      opt_distinct（:11565）的多条分支
```

本篇把这个对应关系逐符号、逐行讲清楚。

---

## 一、BNF / EBNF 是什么

### 1.1 BNF（Backus-Naur Form，巴科斯-诺尔范式）

描述**上下文无关文法（CFG）**的元语言，1959 年由 Backus 为 ALGOL 60 提出。只有两个元符号：

- `::=`（"定义为"）
- `|`（"或"）

BNF 表达"可选"和"重复"只能靠**多写辅助规则**：

```bnf
<select_statement> ::= SELECT <select_list> <from_clause>
<from_clause> ::= FROM <table_list>     -- 有 FROM
<from_clause> ::=                        -- 没有 FROM（用空分支表达"可选"）
```

### 1.2 EBNF（Extended BNF）

BNF 加三个缩写元符号（ISO/IEC 14977 标准化）：

| 符号 | 含义 |
|------|------|
| `[ xxx ]` | 可选（0 或 1 次） |
| `{ xxx }` | 重复（0 次或多次） |
| `( xxx )` | 分组 |

```ebnf
<select_statement> ::= SELECT <select_list> [ FROM <table_list> ]
--                                              ^ 一条规则搞定"可选"
```

### 1.3 尖括号 `<xxx>` 是"非终结符"的标记（但 MySQL 手册不用它）

标准 BNF/EBNF 里，**尖括号包起来的是"非终结符"**（还没展开成具体文本的语法成分），与之相对的是终结符（直接写的字符）。所以：

- `<query expression>`、`<query specification>` —— 非终结符（还能继续展开）
- `SELECT`、`FROM`、`+` —— 终结符（就长这样）

**但关键：你贴的 MySQL 官方手册语法里，一个尖括号都没有。** 因为 MySQL 手册**不用尖括号**，它改用**大小写**来区分终结符/非终结符：

| 记法风格 | 终结符（关键字） | 非终结符（占位符） |
|---------|----------------|------------------|
| **标准 EBNF / SQL 标准** | `SELECT`（照抄） | `<select expr>`（尖括号包） |
| **MySQL 官方手册** | `SELECT`（大写） | `select_expr`（**小写斜体**） |

所以 **"尖括号"和"小写斜体"是同一个东西（非终结符）的两种标记方式**：

- SQL 标准文档、`query_term.h` 源码注释 → 尖括号 `<query expression>`
- MySQL 官方手册 → 小写斜体 `select_expr`

这就是为什么 [05 contextualize 篇](05_contextualize.md) 里用 `<query expression>` 尖括号——它遵循的是 **SQL 标准 + 源码注释** 的记法，而不是 MySQL 手册的记法。两套记法并存，读的时候要能互相翻译：`<query expression>`（尖括号）↔ `query_expression`（MySQL 手册 / `sql_yacc.yy` 的非终结符命名）。

而 `sql_yacc.yy` 里的非终结符（`query_expression`、`select_item_list`）**既不用尖括号也不斜体**，就是小写 C 标识符——因为 Bison 语法里非终结符 vs 终结符靠"有没有 `%token` 声明"区分，不需要任何记号。它们的命名习惯，正是 SQL 标准 `<query expression>` **去掉尖括号**后的样子。

> 一句话总结三种记法的"非终结符"长相：SQL 标准 `<xxx>`（尖括号）、MySQL 手册 `xxx`（小写斜体）、`sql_yacc.yy` `xxx`（小写标识符，无记号）—— 三者指同一个东西。

---

## 二、MySQL 官方文档的语法约定

MySQL 8.0 手册《Typographical and Syntax Conventions》的约定（**注意和 ISO 标准 EBNF 有出入**）：

| 符号 | MySQL 手册含义 | ISO EBNF 含义 | 是否一致 |
|------|---------------|--------------|---------|
| `[ xxx ]` 方括号 | 可选（0 或 1 次） | 可选 | ✅ 一致 |
| `{ xxx }` 花括号 | **必选组**（必须从中选一个） | **重复**（0 次或多次） | ❌ **不同！** |
| `\|` 竖线 | 二选一 | 二选一 | ✅ 一致 |
| `xxx ...` 省略号 | 前面的元素可重复 | （无此符号，用 `{}`） | ❌ 来源不同 |
| 大写 | 关键字/终结符，照抄 | 无约定 | — |
| 小写（斜体） | 占位符（用户填，如 `tbl_name`） | 无约定 | — |

**最容易踩的坑**：MySQL 手册的 `{ }` 是"必选组"，**不是** EBNF 标准的"重复"！例如：

```ebnf
{LINES | FIELDS}
-- 意为"LINES 或 FIELDS 必须选一个"，不是"LINES 重复零次或多次"
```

而 MySQL 手册里"重复"用**省略号**表达：

```ebnf
select_expr [, select_expr] ...
--                              ^^^ 前面的 "[, select_expr]" 可重复
```

**"小写占位符"是精髓**：`select_expr`、`where_condition`、`tbl_name` 这些斜体小写字**不是关键字**，是"这里该放什么类型的东西"的占位符。它们最终由词法层的通用 token 类别（`IDENT`/`NUM`/`TEXT_STRING`）或其它语法规则展开。

---

## 三、EBNF ↔ BNF：官方文档和 sql_yacc.yy 的互转

`sql_yacc.yy`（Bison 方言）没有 `[]`/`{}`，官方文档里的缩写必须**展开成辅助非终结符**。三条翻译规则：

### 规则 1：`[ xxx ]` → `opt_xxx` 规则

```bnf
-- 官方文档：[WHERE where_condition]
-- sql_yacc.yy:
opt_where_clause:
          %empty                       -- 空分支 = "可选项没出现"
        | where_clause
        ;
```

`sql_yacc.yy` 里所有 `opt_` 前缀的非终结符，就是官方文档里对应位置的 `[ ]`。例：`opt_from_clause`(:10040)、`opt_group_clause`(:12525)、`opt_having_clause`(:12405)、`opt_window_clause`(:12485)、`opt_order_clause`(:12587)、`opt_limit_clause`(:12624)。

**这个规律可以量化**：8.0.39 的 `sql/sql_yacc.yy` 里有 **216 条** `^opt_` 开头的规则、共 **248 处** `%empty`——也就是说，官方文档里满屏的 `[...]`，在这份文件里就是这 216 条规则和 248 个空分支。

**真实源码**（`sql/sql_yacc.yy:12396` / `:10040`）：

```yacc
opt_where_clause:
          %empty { $$ = nullptr; }
        | where_clause
        ;

where_clause:
          WHERE expr    { $$ = NEW_PTN PTI_where(@2, $2); }
        ;
```

```yacc
opt_from_clause:
          %empty %prec EMPTY_FROM_CLAUSE { $$.init(YYMEM_ROOT); }
        | from_clause
        ;

from_clause:
          FROM from_tables { $$= $2; }
        ;

from_tables:
          DUAL_SYM { $$.init(YYMEM_ROOT); }
        | table_reference_list
        ;
```

> ★ **`%empty %prec EMPTY_FROM_CLAUSE` 是本文件里唯一带 `%prec` 的空分支**——它为了解决一个真实的 **shift/reduce 冲突**：解析器看到 `SELECT <list>` 后又看到 `INTO`，既可以 shift（当作"leading INTO"，后面还能接 `FROM`），也可以 reduce（当作表-less 查询，`INTO` 就成了语句末尾的 "trailing INTO"）。源码在 `:9858` 给了完整说明：

```yacc
/*
  The problem is when the parser has seen SELECT <select list>, and sees an
  INTO token. It can now either shift it or reduce what it has to a table-less
  query expression. If it shifts the token, it will accept seeing a FROM token
  next and hence the INTO will be interpreted as the leading INTO. If it
  reduces what it has seen to a table-less select, however, it will interpret
  INTO as the trailing into. But what if the next token is FROM? Obviously,
  we want to always shift INTO. We do this by two precedence declarations: We
  make the INTO token right-associative, and we give it higher precedence than
  an empty from clause, using the artificial token EMPTY_FROM_CLAUSE.

  The remaining problem is that now we allow the leading INTO anywhere, when
  it should be allowed on the top level only. We solve this by manually
  throwing parse errors whenever we reduce a nested query expression if it
  contains an into clause.
*/
```

```yacc
// sql/sql_yacc.yy:1463 —— 人为优先级声明
%left EMPTY_FROM_CLAUSE
```

### 规则 2：`xxx [, xxx] ...` → `xxx_list` 规则

```bnf
-- 官方文档：select_expr [, select_expr] ...
-- sql_yacc.yy:
select_item_list:
          select_item                          -- 至少一个
        | select_item_list ',' select_item     -- 左递归：再来一个
        ;
```

`_list` 后缀 + **左递归**，就是官方文档里"逗号分隔的可重复列表"。左递归（`list ',' item` 而不是 `item ',' list`）是 LALR 生成器的要求——左递归可以被归约压栈，不炸栈。

**真实源码**（`sql/sql_yacc.yy:10166`）——注意它比上面的抽象多一个 `'*'` 分支（`SELECT *` 的星号是**单独一条产生式**，不是 `select_item`）：

```yacc
select_item_list:
          select_item_list ',' select_item
          {
            if ($1 == NULL || $1->push_back($3))
              MYSQL_YYABORT;
            $$= $1;
          }
        | select_item
          {
            $$= NEW_PTN PT_select_item_list;
            if ($$ == NULL || $$->push_back($1))
              MYSQL_YYABORT;
          }
        | '*'
          {
            Item *item = NEW_PTN Item_asterisk(@$, nullptr, nullptr);
            $$ = NEW_PTN PT_select_item_list;
            if ($$ == nullptr || item == nullptr || $$->push_back(item))
              MYSQL_YYABORT;
          }
        ;

select_item:
          table_wild { $$= $1; }
        | expr select_alias
          {
            $$= NEW_PTN PTI_expr_with_alias(@$, $1, @1.cpp, to_lex_cstring($2));
          }
        ;

select_alias:
          %empty { $$=null_lex_str;}
        | AS ident { $$=$2; }
        | AS TEXT_STRING_validated { $$=$2; }
        ;
```

> ★ 这里能同时看到**规则 1 和规则 2 套用在一个语法点上**：`select_alias` 就是官方手册 `select_expr [[AS] alias]` 里那个 `[[AS] alias]`——**两层可选**被拆成了"外层 `opt_` 式空分支 + 内层 `AS` 也走空分支"。

### 规则 3：`{A | B | C}` → 多分支

"必选组"就是 BNF 本来就有的"多分支"，不需要缩写符号，直接写。

**真实源码**（`sql/sql_yacc.yy:17337`，`query_spec_option`）：

```yacc
query_spec_option:
          STRAIGHT_JOIN       { $$= SELECT_STRAIGHT_JOIN; }
        | HIGH_PRIORITY       { $$= SELECT_HIGH_PRIORITY; }
        | DISTINCT            { $$= SELECT_DISTINCT; }
        | SQL_SMALL_RESULT    { $$= SELECT_SMALL_RESULT; }
        | SQL_BIG_RESULT      { $$= SELECT_BIG_RESULT; }
        | SQL_BUFFER_RESULT   { $$= OPTION_BUFFER_RESULT; }
        | SQL_CALC_FOUND_ROWS {
            push_warning(YYTHD, Sql_condition::SL_WARNING,
                         ER_WARN_DEPRECATED_SYNTAX,
                         ER_THD(YYTHD, ER_WARN_DEPRECATED_SQL_CALC_FOUND_ROWS));
            $$= OPTION_FOUND_ROWS;
          }
        | ALL                 { $$= SELECT_ALL; }
        ;
```

把它聚合起来的是 `select_options` / `select_option_list`（`:10087`，又是一个"左递归列表"）：

```yacc
select_options:
          %empty
          {
            $$.query_spec_options= 0;
          }
        | select_option_list
        ;

select_option_list:
          select_option_list select_option
          {
            if ($$.merge($1, $2))
              MYSQL_YYABORT;
          }
        | select_option
        ;

select_option:
          query_spec_option
          {
            $$.query_spec_options= $1;
          }
        | SQL_NO_CACHE_SYM
          {
            push_deprecated_warn_no_replacement(YYTHD, "SQL_NO_CACHE");
            /* Ignored since MySQL 8.0. */
            $$.query_spec_options= 0;
          }
        ;
```

> ⚠️ **勘误：`DISTINCTROW` 不是一个独立的语法分支**。官方手册写 `{ALL | DISTINCT | DISTINCTROW}`，但 `sql_yacc.yy` 里**根本没有 `DISTINCTROW` 这个 token**——它在**词法符号表**里被直接映射成 `DISTINCT`（`sql/lex.h:217`）：
>
> ```c
> // sql/lex.h:217
>     {SYM("DISTINCTROW", DISTINCT)}, /* Access likes this */
> ```
>
> 即 `DISTINCTROW` 是 `DISTINCT` 的**同义词**（注释说这是为了兼容 MS Access）。所以"手册 3 个分支 → 源码只有 2 个分支"（`DISTINCT` / `ALL`），第三个在词法层就被吃掉了。
>
> ⚠️ **另一处易错**：`opt_distinct`（`:11565`）**不是** SELECT 的 `[DISTINCT]`，它是 **`GROUP_CONCAT([DISTINCT] ...)` 的可选 DISTINCT**，全文只有两行：
>
> ```yacc
> opt_distinct:
>           %empty      { $$ = 0; }
>         | DISTINCT    { $$ = 1; }
>         ;
> ```

---

## 四、SELECT 语法逐行对照

用户贴的官方 SELECT 语法，逐行对应 `sql_yacc.yy` 的 `query_specification` 产生式（`sql_yacc.yy:9994`）：

| 官方手册（EBNF） | sql_yacc.yy 对应 | 位置 |
|-----------------|-----------------|------|
| `SELECT` | `SELECT_SYM`（终结符） | query_specification 第 1 项 |
| `[ALL \| DISTINCT \| DISTINCTROW]` | `select_options` → `query_spec_option`（`DISTINCTROW` 在词法层并入 `DISTINCT`） | `:17337` |
| `[HIGH_PRIORITY]` / `[STRAIGHT_JOIN]` / `[SQL_SMALL_RESULT] ...` | `query_spec_option` 的其余分支 | `:17337` |
| `select_expr [, select_expr] ...` | `select_item_list` | `:10166` |
| `[into_option]` | `into_clause`（第 4 项，非 opt 前缀但可空） | `query_specification` 第 4 项 |
| `[FROM table_references [PARTITION partition_list]]` | `opt_from_clause` → `from_clause` | `:10040` / `:10045` |
| `[WHERE where_condition]` | `opt_where_clause` | `:12396` |
| `[GROUP BY ... [WITH ROLLUP]]` | `opt_group_clause`（含 `opt_rollup_clause`） | `:12525` |
| `[HAVING where_condition]` | `opt_having_clause` | `:12405` |
| `[WINDOW window_name AS (window_spec) ...]` | `opt_window_clause` | `:12485` |
| `[ORDER BY ...]` | `opt_order_clause` | `:12587` |
| `[LIMIT ...]` | `opt_limit_clause` | `:12624` |
| `[FOR {UPDATE \| SHARE} ...]` | locking clause（挂在 `query_expression_with_opt_locking_clauses`） | — |

核心产生式**完整源码**（`sql_yacc.yy:9994-10038`）：

```yacc
query_specification:
          SELECT_SYM
          select_options
          select_item_list
          into_clause
          opt_from_clause
          opt_where_clause
          opt_group_clause
          opt_having_clause
          opt_window_clause
          {
            $$= NEW_PTN PT_query_specification(
                                      $1,  // SELECT_SYM
                                      $2,  // select_options
                                      $3,  // select_item_list
                                      $4,  // into_clause
                                      $5,  // from
                                      $6,  // where
                                      $7,  // group
                                      $8,  // having
                                      $9,  // windows
                                      @5.raw.is_empty()); // implicit FROM
          }
        | SELECT_SYM
          select_options
          select_item_list
          opt_from_clause
          opt_where_clause
          opt_group_clause
          opt_having_clause
          opt_window_clause
          {
            $$= NEW_PTN PT_query_specification(
                                      $1,  // SELECT_SYM
                                      $2,  // select_options
                                      $3,  // select_item_list
                                      NULL,// no INTO clause
                                      $4,  // from
                                      $5,  // where
                                      $6,  // group
                                      $7,  // having
                                      $8,  // windows
                                      @4.raw.is_empty()); // implicit FROM
          }
        ;
```

**两个阅读要点**：

1. ⚠️ **勘误：`into_clause` 没有 `opt_` 前缀，也**没有** `%empty` 空分支**。真实源码是：

```yacc
// sql/sql_yacc.yy:12754
into_clause:
          INTO into_destination
          {
            $$= $2;
          }
        ;

into_destination:
          OUTFILE TEXT_STRING_filesystem
          opt_load_data_charset
          opt_field_term opt_line_term
          {
            $$= NEW_PTN PT_into_destination_outfile(@$, $2, $3, $4, $5);
          }
        | DUMPFILE TEXT_STRING_filesystem
          {
            $$= NEW_PTN PT_into_destination_dumpfile(@$, $2);
          }
        | select_var_list { $$= $1; }
        ;
```

   `[into_option]` 之所以可选，是因为 **`query_specification` 本身写了两个分支**（一个带 `into_clause`、一个不带），而不是靠空分支。之所以必须这么写，是因为 INTO 在 MySQL 里可以出现在**两个位置**：SELECT 列表之后（leading INTO）和整条语句末尾（trailing INTO）——官方手册因此把 `[into_option]` **写了两处**。末尾那处由 `select_stmt_with_into` 承担（`:9875`）：

```yacc
select_stmt_with_into:
          '(' select_stmt_with_into ')'
          {
            $$ = $2;
          }
        | query_expression into_clause
          {
            $$ = NEW_PTN PT_select_stmt($1, $2);
          }
        ...
```

   ★ 顺带解释上面生产式里那个奇怪的 `@5.raw.is_empty()`：它就是"**implicit FROM**"标志——`SELECT 1`（无 FROM）时 `opt_from_clause` 匹配的是空分支，其源码位置区间 `@5.raw` 为空，于是 `PT_query_specification` 知道这是隐式 FROM。这就是 `SELECT 1` 能走"无表"路径的语法层来源。

2. **每个 `opt_` 展开后就是一列可空项**——官方文档那一长串 `[...]` 叠 `[...]` 的竖排列表，在 Bison 里就是 `query_specification` 右侧一长串 `opt_xxx`。逐个展开如下：

```yacc
// :12405
opt_having_clause:
          %empty { $$= NULL; }
        | HAVING expr
          {
            $$= new PTI_having(@$, $2);
          }
        ;

// :12525
opt_group_clause:
          %empty { $$= NULL; }
        | GROUP_SYM BY group_list olap_opt
          {
            $$= NEW_PTN PT_group($3, $4);
          }
        ;

group_list:
          group_list ',' grouping_expr
          {
            $1->push_back($3);
            $$= $1;
          }
        | grouping_expr
          {
            $$= NEW_PTN PT_order_list();
            if ($$ == NULL)
              MYSQL_YYABORT;
            $$->push_back($1);
          }
        ;

// ★ 官方手册 "GROUP BY ... [WITH ROLLUP]" 的那个 [WITH ROLLUP]
olap_opt:
          %empty { $$= UNSPECIFIED_OLAP_TYPE; }
        | WITH_ROLLUP_SYM { $$= ROLLUP_TYPE; }
            /*
              'WITH ROLLUP' is needed for backward compatibility,
              and cause LALR(2) conflicts.
              This syntax is not standard.
              MySQL syntax: GROUP BY col1, col2, col3 WITH ROLLUP
              SQL-2003: GROUP BY ... ROLLUP(col1, col2, col3)
            */
        ;

// :12587
opt_order_clause:
          %empty { $$= NULL; }
        | order_clause
        ;

order_clause:
          ORDER_SYM BY order_list
          {
            $$= NEW_PTN PT_order($3);
          }
        ;

opt_ordering_direction:
          %empty { $$= ORDER_NOT_RELEVANT; }
        | ordering_direction
        ;

ordering_direction:
          ASC         { $$= ORDER_ASC; }
        | DESC        { $$= ORDER_DESC; }
        ;

// :12624
opt_limit_clause:
          %empty { $$= NULL; }
        | limit_clause
        ;

limit_clause:
          LIMIT limit_options
          {
            $$= NEW_PTN PT_limit_clause($2);
          }
        ;

// :12485
opt_window_clause:
          %empty
          {
            $$= NULL;
          }
        | WINDOW_SYM window_definition_list
          {
            $$= $2;
          }
        ;
```

> ★ **注意这些空分支的语义动作不是"什么都不做"，而是显式写入 `NULL` / `0` / `ORDER_NOT_RELEVANT`**——因为 `%empty` 也必须给 `$$` 赋值，否则后面 `PT_query_specification` 拿到的就是未初始化的 union 成员。`opt_ordering_direction` 的空分支给的是 `ORDER_NOT_RELEVANT`（ORDER 的"方向无关"三态之一），这正是[15 篇](07_optimize/15_groupby_distinct_order.md)里 DISTINCT→GROUP BY 改写时要手动设的那个值。

3. **`[FOR {UPDATE | SHARE} ...]` 不在 `query_specification` 里**——它挂在更外层的 `query_expression_with_opt_locking_clauses`：

```yacc
// sql/sql_yacc.yy:10117
locking_clause_list:
          locking_clause_list locking_clause
          {
            $$= $1;
            if ($$->push_back($2))
              MYSQL_YYABORT; // OOM
          }
        | locking_clause
          {
            $$= NEW_PTN PT_locking_clause_list(YYTHD->mem_root);
            if ($$ == nullptr || $$->push_back($1))
              MYSQL_YYABORT; // OOM
          }
        ;

locking_clause:
          FOR_SYM lock_strength opt_locked_row_action
          {
            $$= NEW_PTN PT_query_block_locking_clause($2, $3);
          }
        | FOR_SYM lock_strength table_locking_list opt_locked_row_action
          {
            $$= NEW_PTN PT_table_locking_clause($2, $3, $4);
          }
        | LOCK_SYM IN_SYM SHARE_SYM MODE_SYM
          {
            $$= NEW_PTN PT_query_block_locking_clause(Lock_strength::SHARE);
          }
        ;

lock_strength:
          UPDATE_SYM { $$= Lock_strength::UPDATE; }
        | SHARE_SYM  { $$= Lock_strength::SHARE; }
        ;

opt_locked_row_action:
          %empty { $$= Locked_row_action::WAIT; }
        | locked_row_action
        ;

locked_row_action:
          SKIP_SYM LOCKED_SYM { $$= Locked_row_action::SKIP; }
        | NOWAIT_SYM { $$= Locked_row_action::NOWAIT; }
        ;
```

> ★ 这正是"官方文档为了可读性把语法**摊平**在一页里，而 Bison 必须按**归约层次**拆开"的典型例子：`query_specification` 只负责 SELECT 本体，`FOR UPDATE` 属于更高层的 query expression。

---

## 五、语法记法和词法/语法解析的对应

官方文档语法里的**每个元素**，在解析链路里都有落点：

| 记法元素 | 词法层（02 篇） | 语法层（04 篇） |
|---------|----------------|----------------|
| 大写关键字（`SELECT`/`FROM`/`WHERE`） | 关键字 token：`SELECT_SYM`/`FROM_SYM`/`WHERE_SYM` | 产生式右侧的终结符 |
| 小写占位符（`select_expr`/`where_condition`/`tbl_name`） | 通用 token 类别：`IDENT`/`NUM`/`TEXT_STRING` | 非终结符（再展开成子规则） |
| `[ ]` / `{ }` / 竖排 | （词法不关心，这是语法层的结构） | `opt_xxx` / 多分支 / 产生式 |
| `, ...` 重复列表 | 词法产出 `,`（单字符 token） | `xxx_list` 左递归规则 |

### 5.1 "终结符 vs 非终结符"在 Bison 里怎么表达：`%token` vs `%type`

本篇 1.3 节说："`sql_yacc.yy` 里的非终结符**既不用尖括号也不斜体**，就是小写 C 标识符——因为 Bison 语法里非终结符 vs 终结符靠**有没有 `%token` 声明**区分"。这句话的源码依据就是这两组声明：

```yacc
// sql/sql_yacc.yy:1081 —— 终结符：%token 声明 + 显式编号 + SQL 标准出处注释
%token  SELECT_SYM 748                    /* SQL-2003-R */
```

```yacc
// sql/sql_yacc.yy:1871 —— 非终结符：%type 声明 + 语义值类型（union 成员名）
%type <query_primary>
        query_primary
        query_specification

%type <query_expression> query_expression

%type <subquery> subquery row_subquery table_subquery
```

判据一目了然：

| | 声明关键字 | 有没有编号 | 语义值类型 | 例子 |
|---|---|---|---|---|
| **终结符** | `%token` | **有**（如 `748`） | 可带（如 `%token<lexer.keyword>`） | `SELECT_SYM`、`FROM_SYM`、`DISTINCT` |
| **非终结符** | `%type` | 无 | **必须有**（`<query_primary>` 等） | `query_specification`、`select_item_list` |

对照本篇 1.3 的三套记法：

| 记法 | 终结符 | 非终结符 |
|---|---|---|
| SQL 标准 EBNF | `SELECT`（照抄） | `<query specification>`（尖括号） |
| MySQL 官方手册 | `SELECT`（大写） | `query_specification`（小写斜体） |
| `sql_yacc.yy` | `%token SELECT_SYM 748` | `%type <query_primary> query_specification` |

> ★ 还有一处**词法层的记号**：终结符 `SELECT_SYM` 比手册的 `SELECT` 多了 `_SYM` 后缀——这是 MySQL 的惯例（"symbol"），用来和 C/C++ 标识符、以及 `lex.h` 里的符号表条目区分。读 `sql_yacc.yy` 时看到 `XXX_SYM` 就知道"这是手册里的大写关键字 `XXX`"。

### 5.2 优先级声明也是"记法"的一部分

官方手册不写优先级，但 Bison 必须有它才能消解冲突。前面 `EMPTY_FROM_CLAUSE` 那段已经露过一次，完整的一组（`:1461`）：

```yacc
%left '(' ')'

%left EMPTY_FROM_CLAUSE
%right INTO
```

`INTO` 被声明为**右结合**且**优先级高于** `EMPTY_FROM_CLAUSE`——这正是"永远 shift INTO"的实现方式（见第三章规则 1 的源码注释）。

**一句话**：官方文档语法 = 语法层的"用户界面"；`sql_yacc.yy` = 语法层的"实现"；词法层负责把大写关键字变成关键字 token、把小写占位符变成 `IDENT`/`NUM` 等通用 token——三者在同一条流水线上。

**读任意一条官方语法的最佳实践**：

1. 先看 `[]`（可选项）——可以整段忽略，不影响主干
2. 再看 `{}` 和 `|`——必选的分支
3. 最后看 `...`——哪里可重复
4. 到 `sql_yacc.yy` 找同名/相近的非终结符（`opt_`/`_list` 后缀），读产生式本体

---

## 参考

- **ISO/IEC 14977** —— EBNF 国际标准（`{}` 是重复、`[]` 可选、`()` 分组）
- *MySQL 8.0 Reference Manual → Typographical and Syntax Conventions*（官方手册的语法约定：`[]`/`{}`/`|`/`...`/大小写）
- *MySQL 8.0 Reference Manual → SELECT Statement*（本篇第四节的对照对象）
- **Aho & Ullman《Principles of Compiler Design》（龙书）** —— CFG、BNF、产生式、左递归
