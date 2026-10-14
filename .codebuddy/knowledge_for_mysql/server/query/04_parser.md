# 04 Parser：从 token 流到 Parse Tree

> 本篇覆盖：语法分析如何把 token 流变成 **Parse Tree（第 ① 层 IR）**，以及它的内存归属。词法分析（字符流 → token 流）见 [02 Lexer](02_lexer.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、sql_yacc.yy 的整体结构](#一sql_yaccyy-的整体结构)
- [二、四个关键宏](#二四个关键宏)
- [三、Bison 生成物在哪](#三bison-生成物在哪cmake-构建期代码生成)
- [四、LEX 结构](#四lex-结构)
- [五、关键产生式速查](#五关键产生式速查)
- [六、Parse Tree 节点体系](#六parse-tree-节点体系)
- [七、PTI_\* 的双重身份](#七pti_-的双重身份)
- [八、Parse Tree 的内存管理](#八parse-tree-的内存管理)
- [示例：一条 SQL 产生哪些 PT 节点](#示例一条-sql-产生哪些-pt-节点)

---

## 设计思想与理论基础

### 分阶段编译：词法 → 语法 → 语义

Parser 处于经典编译原理（Aho & Ullman 龙书）**三段式的前两段**：

| 阶段 | 本篇 | 输入 → 输出 | 特点 |
|---|---|---|---|
| 词法分析 | [02 Lexer](02_lexer.md) | 字符流 → token 流 | 无结构、正则可描述 |
| 语法分析 | 本篇主体 | token 流 → Parse Tree | **上下文无关**、上下文无关文法描述 |
| 语义分析 | 下一篇 04 | Parse Tree → 逻辑树 | 上下文相关、需要符号表 |

**为什么分阶段**：词法的正则文法无法表达 SQL 的括号嵌套、递归结构，而上下文无关文法（CFG）又无法表达"这个 ident 是列名还是表名"这类依赖上下文的判断。分层让每一层只处理它能形式化描述的那部分。

### 为什么用 LALR(1) 生成器，而非手写递归下降

MySQL 用 **Bison**（Yacc 的 GNU 版）从 `sql_yacc.yy` 生成 LALR(1) 表驱动解析器，而不是手写递归下降。权衡：

- **语法规模**：SQL 有上千条产生式、大量优先级/结合性声明（`%left`/`%prec`）。手写递归下降要为每条规则写函数，维护成本极高；表驱动的状态机统一处理
- **代价**：LALR(1) 的表对"归约/移入冲突"敏感（`%expect N` 声明了允许的冲突数），报错信息不如手写解析器友好
- **遗留痕迹**：`sql_yacc.yy` 里大量 `/* must be last */`、优先级 hack，是几十年增量演进的必然结果——没有一个"从头设计"的 SQL 解析器能躲开这些

### 为什么要有 PT / PTI 两层

8.0 引入了 `PT_*`（Parse Tree，纯语法节点）和 `PTI_*`（Parse Tree Item，表达式的语法节点）两套类，而不是像 5.x 那样在语义动作里直接 `new Item`。这层设计的意义：

- **语法与语义解耦**：Parse Tree 是纯语法的，不碰 `Field`、不碰 `TABLE`，所以同一个 Parse Tree 能被"contextualize"成不同的语义对象（也支撑了 `PREPARE` 一次、`EXECUTE` 多次）
- **可复用/可重写**：后续的 query rewrite 插件（`sql_query_rewrite.cc`）在 Parse Tree 层做改写，比在 Item 层安全得多

### 演进

- **5.x**：语义动作直接建 `Item`，语法树与语义对象纠缠，Parse Tree 概念不清晰
- **8.0**：正式确立 PT/PTI 类体系（`sql/parse_tree_nodes.h`），Parse Tree 成为独立的第 ① 层 IR——这是后面 contextualize（04）能独立成篇的前提

---

## 先有个整体印象

Parser 做的事就一件：**按语法规则把 token 流归约成一棵树**。

这棵树是**纯语法的、上下文无关的** —— 它只知道"这里有个 SELECT、有个 FROM、有个 WHERE"，**不知道** `t` 是哪张表、`a` 是哪一列、这个列存不存在。把语法树"落地"成有语义的结构，是下一篇 [04 contextualize](05_contextualize.md) 的事。

```
SELECT a FROM t WHERE b > 1 ORDER BY c
   │  词法
   ▼
[SELECT][ident a][FROM][ident t][WHERE][ident b][>][NUM 1][ORDER][BY][ident c]
   │  语法归约
   ▼
PT_select_stmt
  └─ PT_query_expression
      └─ PT_query_specification
          ├─ PT_select_item_list → [PTI_expr_with_alias → PTI_simple_ident_ident(a)]
          ├─ [PT_table_factor_table_ident(t)]
          ├─ PTI_where → PTI_comp_op(PTI(b), >, Item_int(1))
          └─ PT_order → PT_order_list → [PT_order_expr(PTI(c), ASC)]
```

---

## 一、sql_yacc.yy 的整体结构

文件共 **18306 行**，是 MySQL 源码里最大的文件之一。★ 下行号表已逐条 grep 校正（原表有 6 行左右的系统性偏移）：

| 区段 | 行号 | 内容 |
|------|------|------|
| `%{` 前导区开始 | `:32` | |
| 关键宏 | `:36-46` | `YYP` / `YYLIP` / `YYPS` / `YYCSCL` / `YYMEM_ROOT` / `YYCLIENT_NO_SCHEMA` / `YYINITDEPTH` / `YYMAXDEPTH` / `Lex` / `Select` |
| `#include` 区 | `:48-171` | 约 124 个；`parser_yystype.h` 在 **`:122`** |
| `my_yyoverflow` 前置声明 | `:479` | 实现在 **`sql_parse.cc:5086`**（不在本文件） |
| `%}` 前导区结束 | `:513` | |
| `%start start_entry` | `:515` | |
| `%parse-param` ×2 / `%lex-param` | `:525 / 526 / 528` | **入参/出参** |
| `%define api.pure` | `:529` | 可重入（**全文唯一一条 `%define`**） |
| **两条铁律注释** | **`:530-534`** | `We do not accept any reduce/reduce conflicts` |
| `%expect 63` | `:535` | 预期 63 个移进/归约冲突 |
| `MAINTAINER:` token 规则注释 | `:537-585` | **讲的是 token 维护，不是冲突**（见下） |
| `%token` 段 | `:591` ~ `:1410` | 编号 258→1204 |
| 优先级段 | `:1428-1464` | 含 `CONDITIONLESS_JOIN`(1441)、`SUBQUERY_AS_EXPR`(1460)、`EMPTY_FROM_CLAUSE`(1463)、`INTO`(1464) |
| `%type` 段 | `:1466` ~ `:2212` | 205 条 |
| **`%%`** | **`:2214`** | **全文只有一个 `%%`**：规则区 2215 到文件末尾，**没有 epilogue 段** |
| `start_entry:` | `:2238-2282` | **6 个分支** |
| `sql_statement:` | `:2284-2327` | 3 个分支 |
| **`simple_statement_or_begin`** | **`:2334-2337`** | **`*parse_tree = $1;` 在 `:2335`** ← **输出 Parse Tree 根** |

**量化事实**（`sql/sql_yacc.yy`）：18306 行 / 961 条产生式 / 205 条 `%type` / 53 处 `%prec` / 878 处 `NEW_PTN`（约 877 次使用）/ 562 处 `MYSQL_YYABORT`。

#### 前导区宏完整源码

```cpp
// sql/sql_yacc.yy:32-46
%{
/*
Note: YYTHD is passed as an argument to yyparse(), and subsequently to yylex().
*/
#define YYP (YYTHD->m_parser_state)                  // Parser_state*：整个解析器的状态容器
#define YYLIP (& YYTHD->m_parser_state->m_lip)       // Lex_input_stream*：词法输入流（含原始 SQL buffer）
#define YYPS (& YYTHD->m_parser_state->m_yacc)       // Yacc_state*：bison 三栈的动态副本，my_yyoverflow 用它
#define YYCSCL (YYLIP->query_charset)                // 客户端字符集
#define YYMEM_ROOT (YYTHD->mem_root)                 // MEM_ROOT*：所有 NEW_PTN 的分配目标
#define YYCLIENT_NO_SCHEMA (YYTHD->get_protocol()->has_client_capability(CLIENT_NO_SCHEMA))

#define YYINITDEPTH 100                              // bison 静态栈初始深度（槽数）
#define YYMAXDEPTH 3200                              /* Because of 64K stack */
#define Lex (YYTHD->lex)                             // LEX*：全局语法/语义上下文
#define Select Lex->current_query_block()            // Query_block*：当前 SELECT 的 query block
```

> ★ `Lex` / `Select` 定义在 `:45-46`，在 `:48-171` 那一大段 `#include` **之前**——有意如此，因为下面的宏体（`CONTEXTUALIZE` 等）要用它们。

**栈溢出处理**：`.yy` 里只有宏与声明，实现在 `sql/sql_parse.cc:5086`：

```yacc
// sql/sql_yacc.yy:190-204
#define yyoverflow(A,B,C,D,E,F,G,H)           \
  {                                           \
    ulong val= *(H);                          \
    if (my_yyoverflow((B), (D), (F), &val))   \
    {                                         \
      yyerror(NULL, YYTHD, NULL, (const char*) (A));\   /* A 是 bison 传来的 "memory exhausted" */ \
      return 2;                               \
    }                                         \
    else                                      \
    {                                         \
      *(H)= (YYSIZE_T)val;                    \
    }                                         \
  }
```

```cpp
// sql/sql_parse.cc:5083
#define MY_YACC_INIT 1000  // Start with big alloc
#define MY_YACC_MAX 32000  // Because of 'short'

bool my_yyoverflow(short **yyss, YYSTYPE **yyvs, YYLTYPE **yyls,
                   ulong *yystacksize) {
  Yacc_state *state = &current_thd->m_parser_state->m_yacc;
  ulong old_info = 0;
  assert(state);
  if ((uint)*yystacksize >= MY_YACC_MAX) return true;
  if (!state->yacc_yyvs) old_info = *yystacksize;   // 只有第一次（静态栈→动态栈）才需手工 memcpy
  *yystacksize = set_zone((*yystacksize) * 2, MY_YACC_INIT, MY_YACC_MAX);
  if (!(state->yacc_yyvs =
            (uchar *)my_realloc(key_memory_bison_stack, state->yacc_yyvs,
                                *yystacksize * sizeof(**yyvs),
                                MYF(MY_ALLOW_ZERO_PTR | MY_FREE_ON_ERROR))) ||
      !(state->yacc_yyss = (uchar *)my_realloc(...)) ||
      !(state->yacc_yyls = (uchar *)my_realloc(...)))
    return true;
  if (old_info) {
    /*
      Only copy the old stack on the first call to my_yyoverflow(),
      when replacing a static stack (YYINITDEPTH) by a dynamic stack.
      For subsequent calls, my_realloc already did preserve the old stack.
    */
    memcpy(state->yacc_yyss, *yyss, old_info * sizeof(**yyss));
    memcpy(state->yacc_yyvs, *yyvs, old_info * sizeof(**yyvs));
    memcpy(state->yacc_yyls, *yyls, old_info * sizeof(**yyls));
  }
  *yyss = (short *)state->yacc_yyss;
  *yyvs = (YYSTYPE *)state->yacc_yyvs;
  *yyls = (YYLTYPE *)state->yacc_yyls;
  return false;
}
```

> ★ 解决的问题：bison 默认用 `alloca` 扩栈（深嵌套 SQL 会爆 64K 线程栈）。MySQL 改成**堆上** `my_realloc`，按 1000 起步、每次 ×2、封顶 32000，三块内存挂在 `Yacc_state` 上随 THD 释放。

**`MYSQL_YYABORT` 是三个，不是两个**（`:206-220`）：

```cpp
#define MYSQL_YYABORT YYABORT                 // 纯中止，不发错误消息（错误已在别处 push）

#define MYSQL_YYABORT_ERROR(...)              \ // 先 my_error() 再中止，用于语法动作内报具体 errno
  do                                          \
  {                                           \
    my_error(__VA_ARGS__);                    \
    MYSQL_YYABORT;                            \
  } while(0)

#define MYSQL_YYABORT_UNLESS(A)               \ // 条件不成立 → 报通用语法错 + 中止
  if (!(A))                                   \
  {                                           \
    YYTHD->syntax_error();                    \
    MYSQL_YYABORT;                            \
  }
```

> ⚠️ `MYSQL_YYABORT_UNLESS` **末尾没有 `do{}while(0)`**，是裸 `if`——在 `if/else` 里用有 dangling-else 风险。

**★ `yyerror` 的真实名字是 `MYSQLerror`**（`:299-311`）——因为 CMake 传了 `--name-prefix=MYSQL`（`sql/CMakeLists.txt:1340`），`yyparse→MYSQLparse`、`yylex→MYSQLlex`、`yyerror→MYSQLerror`：

```cpp
// sql/sql_yacc.yy:280-297（doxygen 摘录）
/**
  @brief Bison callback to report a syntax/OOM error
  ...
  This function is not invoked when the parser is requested to abort by
  semantic action code by means of YYABORT or YYACCEPT macros..
  In semantic actions, please use syntax_error or my_error to push an error
  into the error stack and MYSQL_YYABORT to abort from the parser.
*/
static
void MYSQLerror(YYLTYPE *location, THD *thd, Parse_tree_root **, const char *s)
{
  if (strcmp(s, "syntax error") == 0) {
    thd->syntax_error_at(*location);
  } else if (strcmp(s, "memory exhausted") == 0) {
    my_error(ER_DA_OOM, MYF(0));
  } else {
    // Find omitted error messages in the generated file (sql_yacc.cc) and fix:
    assert(false);
    my_error(ER_UNKNOWN_ERROR, MYF(0));
  }
}
```

> ★ `ER_PARSE_ERROR`（"You have an error in your SQL syntax near ..."）**不在这里**。`MYSQLerror` 只发 `ER_SYNTAX_ERROR` / `ER_DA_OOM`；真正拼 `ER_PARSE_ERROR` 的是 `THD::vsyntax_error_at()`（`sql/sql_class.cc:2820`），它用 `location.raw.start`（**原始 buffer** 位置，不是预处理后的 `cpp` 位置）取出行号。

### `%expect 63` 说明什么

63 个移进/归约（shift/reduce）冲突，说明 MySQL 的语法**不是纯 LALR(1) 可解析的**。`%expect 63` 的含义是"**我们已知、且接受这 63 个 SR 冲突**"——bison 会在生成时核对实际冲突数，**多了或少了都报错**，所以这个数字是"锁死的契约"。

#### 两条铁律

`%expect 63` 上方就是两条铁律（原文注释，**`:530-534`**）：

```yacc
// sql/sql_yacc.yy:530
/*
  1. We do not accept any reduce/reduce conflicts
  2. We should not introduce new shift/reduce conflicts any more.
*/
```

> ⚠️ **勘误**：这段注释**没有 `MAINTAINER:` 标题**。带 `MAINTAINER:` 标题的是**下面另一段**（`:537-585`），内容完全是另一回事——**token 维护规则**（4 条），与冲突无关：

```yacc
// sql/sql_yacc.yy:537
/*
   MAINTAINER:

   1) Comments for TOKENS.
   For each token, please include in the same line a comment that contains
   one or more of the following tags:
   SQL-2015-N / SQL-2015-R / SQL-2003-R / SQL-2003-N / SQL-1999-R / SQL-1999-N
   MYSQL / MYSQL-FUNC / INTERNAL / OPERATOR / FUTURE-USE
   This makes the code grep-able, and helps maintenance.

   2) About token values
   Token values are assigned by bison, in order of declaration.
   Token values are used in query DIGESTS.
   To make DIGESTS stable, it is desirable to avoid changing token values.
   In practice, this means adding new tokens at the end of the list,
   in the current release section (8.0), instead of adding them in the middle.
   Failing to comply with instructions below will trigger build failure,
   as this process is enforced by gen_lex_token.

   3) Instructions to add a new token:
   Add the new token at the end of the list, in the MySQL 8.0 section.

   4) Instructions to remove an old token:
   Do not remove the token, rename it as follows:
   %token OBSOLETE_TOKEN_<NNN> /* was: TOKEN_FOO * /
   where NNN is the token value (found in sql_yacc.h)
   For example, see OBSOLETE_TOKEN_820
*/
```

> ★ 第 2 条解释了为什么 `sql_yacc.yy` 里有一批 `OBSOLETE_TOKEN_NNN`：token 编号进了 **query digest**（见 [`14_sql_digest.md`](14_sql_digest.md)），删掉会让 digest 漂移，所以只能废弃不能删除。

| 铁律 | 含义 | 为什么 |
|---|---|---|
| **不接受任何 RR 冲突** | reduce/reduce 冲突是**语法真正的歧义**（同一个输入能归约成两个不同的非终结符），必须逐个消除 | RR 冲突意味着语义不定，几乎总是 bug |
| **不再引入新 SR 冲突** | shift/reduce 冲突数锁死在 63，新增语法规则不能让这个数变大 | SR 冲突通常能被优先级声明消解，是可管理的历史包袱 |

★ 这两条是"改语法"时的红线：**你加一条规则，如果让冲突数从 63 变成 64，bison 会直接报错**，你必须在不改动 `%expect` 的前提下消解它。

#### 冲突的两个来源

1. **LALR(2) 而非 LALR(1)**：典型是 `WITH ROLLUP`——`WITH` 后面到底是 CTE 还是 ROLLUP 修饰符，需要多看一个 token。MySQL 在词法层特判绕过（`sql/sql_lex.cc` 的 `MYSQLlex` 内做 `WITH + ROLLUP` 的 LALR(2)→LALR(1) 归约）。

2. **SQL 语法本身的二义性**：`STRAIGHT_JOIN`、表达式优先级（`a + b * c`、`NOT a IS NULL`）等，靠**优先级声明**解决。

#### 优先级声明体系：一整条"优先级阶梯"

`sql_yacc.yy` 的 `%left`/`%right`/`%nonassoc` 声明构成一条**从低到高的优先级阶梯**（先声明的低，后声明的高）。**完整原文**（`:1428-1464`）：

```yacc
%left KEYWORD_USED_AS_IDENT                                 // :1428 关键字被当作标识符用（最低）
%nonassoc TEXT_STRING                                       // :1429
%left KEYWORD_USED_AS_KEYWORD                               // :1430 关键字作为关键字
%right UNIQUE_SYM KEY_SYM                                   // :1437
%left UNION_SYM EXCEPT_SYM                                  // :1439 UNION/EXCEPT 同级
%left INTERSECT_SYM                                         // :1440 SQL 标准里 INTERSECT 高于 UNION/EXCEPT
%left CONDITIONLESS_JOIN                                    // :1441 无 ON 的 JOIN（见 joined_table 注释）
%left   JOIN_SYM INNER_SYM CROSS STRAIGHT_JOIN NATURAL LEFT RIGHT ON_SYM USING   // :1442
%left   SET_VAR                                             // :1443 ★ 本文件"最低优先级"惯用假 token
%left   OR_SYM OR2_SYM                                      // :1444
%left   XOR                                                 // :1445
%left   AND_SYM AND_AND_SYM                                 // :1446
%left   BETWEEN_SYM CASE_SYM WHEN_SYM THEN_SYM ELSE         // :1447
%left   EQ EQUAL_SYM GE GT_SYM LE LT NE IS LIKE REGEXP IN_SYM  // :1448 比较运算
%left   '|'                                                 // :1449
%left   '&'                                                 // :1450
%left   SHIFT_LEFT SHIFT_RIGHT                              // :1451
%left   '-' '+'                                             // :1452
%left   '*' '/' '%' DIV_SYM MOD_SYM                         // :1453
%left   '^'                                                 // :1454
%left   OR_OR_SYM                                           // :1455 || 比 ^ 还高
%left   NEG '~'                                             // :1456 一元负号/取反
%right  NOT_SYM NOT2_SYM                                    // :1457
%right  BINARY_SYM COLLATE_SYM                              // :1458 COLLATE 几乎最高
%left  INTERVAL_SYM                                         // :1459
%left SUBQUERY_AS_EXPR                                      // :1460 ★ 子查询当表达式用的假 token
%left '(' ')'                                               // :1461
%left EMPTY_FROM_CLAUSE                                     // :1463 空 FROM（配合 INTO 二义性）
%right INTO                                                 // :1464 最高
```

★ 这条阶梯的**顺序就是 SQL 表达式的优先级语义**：`OR` 低于 `AND`、`AND` 低于比较、比较低于算术、算术低于一元负号、`NOT` 高于 `AND`（`%right`）……MySQL 的表达式优先级不是散落在语义层，而是**由这段 `%left`/`%right` 声明一次性定义**的。

#### `%prec`：给单条产生式"临时改优先级"

53 处 `%prec` 是消解冲突的**具体手段**——当某条产生式不能简单继承它最后一个 token 的优先级时，显式指定：

| `%prec` token | 场景 | 说明 |
|---|---|---|
| `SUBQUERY_AS_EXPR` | `query_expression_parens` | 括号里的查询当表达式用 |
| `EMPTY_FROM_CLAUSE` | 空 `%empty` 的 FROM 子句 | 空产生式没有 token，必须显式给优先级 |
| `KEYWORD_USED_AS_KEYWORD` | `BIT_SYM` 等在特定上下文 | 同名字符在不同上下文当关键字/标识符 |
| `KEYWORD_USED_AS_IDENT` | `DATE_SYM`/`TIME_SYM`/`TIMESTAMP_SYM` 等 | 这些词在多数上下文是标识符（所以优先级最低） |
| `NEG` | 一元 `-` / `~` / `not2` | 一元运算符的优先级与二元 `-` 区分 |
| `SET_VAR` | `bool_pri` / `predicate` / `bit_expr` / `simple_expr` 的纯提升产生式 | `@@var` 赋值表达式的优先级 |
| `CONDITIONLESS_JOIN` | 无条件 join | `STRAIGHT_JOIN` 等 |

★ 理解 `%prec` 是读 `sql_yacc.yy` 的关键：**产生式末尾的 `%prec X` 意思是"这条产生式的优先级按 token X 算"**。为什么 `bit_expr '-' bit_expr %prec '-'` 要显式写 `%prec '-'`？因为该产生式右部以 `bit_expr` 结尾，bison 默认取**最后一个终结符** `bit_expr` 的优先级，那是错的——所以显式指定 `'-'`。

#### 冲突排查方法（新增语法规则时）

1. 加规则后直接 `bison` 编译，看 `%expect 63` 是否被破坏（冲突数≠63 会报错）；
2. 用 `bison --report=all` 生成 `.output` 文件，定位冲突在哪两个状态、哪两条产生式；
3. 看冲突是 SR 还是 RR：**RR 必须消**（改文法），SR 可以用 `%prec` 或调整优先级阶梯消解；
4. 实在无法用优先级消解、且确认语义无害的，才考虑改 `%expect`——但这是最后手段，且要更新注释说明新增冲突的来龙去脉。

★ 顺带：`cmake/bison.cmake` 里 `--warnings=all,no-yacc,no-precedence` 意味着 bison 会**警告"无用优先级声明"**，但 MySQL 显式关掉了 `no-precedence` 告警（历史遗留的 `--yacc` 兼容），所以代码里那些"看起来多余的 `%prec`"可能不会触发告警。

---

## 二、四个关键宏

这四个宏是理解"parse 与 contextualize 如何交织"的钥匙：

```cpp
// sql/sql_yacc.yy:222
#define NEW_PTN new(YYMEM_ROOT)          // YYMEM_ROOT = YYTHD->mem_root（:40）
```

```cpp
// sql/sql_yacc.yy:228-235
#define CONTEXTUALIZE(x)                                    \
  do {                                                      \
    std::remove_reference<decltype(*x)>::type::context_t pc(YYTHD, Select); \
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->contextualize(&pc)))    \
      MYSQL_YYABORT;                                                    \
  } while(0)
```

**`ITEMIZE` 完整源码**（`:248-258`）——★ 与 `CONTEXTUALIZE` 的**结构性差异只有一处**：它用裸 `Parse_context`，而 `CONTEXTUALIZE` 用节点自己的 `context_t` typedef：

```cpp
// sql/sql_yacc.yy:248
/**
  Item::itemize() function call wrapper
*/
#define ITEMIZE(x, y)                                                   \
  do                                                                    \
  {                                                                     \
    Parse_context pc(YYTHD, Select);                                    \   // ← 254：裸 Parse_context
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->itemize(&pc, (y))))     \   // ← 256：x = Item*, y = Item**（出参）
      MYSQL_YYABORT;                                                    \
  } while(0)
```

**`MAKE_CMD` 完整源码**（`:260-270`）：

```cpp
// sql/sql_yacc.yy:260
/**
  Parse_tree_root::make_cmd() wrapper to raise postponed error message on OOM

  @note x may be NULL because of OOM error.
*/
#define MAKE_CMD(x)                                    \
  do                                                   \
  {                                                    \
    if (YYTHD->is_error() || Lex->make_sql_cmd(x))     \   // ← 268：x 是 Parse_tree_root*
      MYSQL_YYABORT;                                   \
  } while(0)
```

> ★ `268` 行先判 `YYTHD->is_error()`：OOM 时 `x` 可能为 `NULL`，`make_sql_cmd(NULL)` 本身合法（会发 OOM 错误），但**已经**有错误就不该被覆盖——这就是注释 `@note x may be NULL` 的含义（**不在宏里判 `x == NULL`**）。

**`MAKE_CMD` 的直接调用点**（7 处，已校正行号）：`:4382, 4948, 8450, 8462, 8473, 16377, 16440` —— 这些语句（DDL、管理语句等）在**语法动作内部**就转成 `Sql_cmd`，`*parse_tree` 保持 NULL。

> **重要**：绝大多数语句**不走** `MAKE_CMD`，而是走 `simple_statement_or_begin: { *parse_tree = $1; }`（`:2335`）把 root 交出去，由 **`THD::sql_parser()`（`sql_class.cc:3077`：`if (root != nullptr && lex->make_sql_cmd(root))`）** 统一调用。`MAKE_CMD` 只用于必须在语法动作内立即完成的情况。
>
> ★ `THD::sql_parser()` 的注释原文说明了"root 为 NULL 也是成功"的情形：*"If 'root' is NULL on success, then the parser has already called lex->make_sql_cmd() internally."*

> ★ 还有一族**不是宏**的手写等价物 `contextualize_array(pc, $N)`，出现在 `:3499, 3519, 3538, 8291, 8313, 8332, 8359, 8382, 12961, 12981, 13003`（给数组/列表版节点批量 contextualize）。**真正的"上下文化"绝大多数走 `contextualize_array`，而不是 `CONTEXTUALIZE` 宏**——后者全文只有 7 处调用（6 处 `CONTEXTUALIZE(...)` + 1 处 `CONTEXTUALIZE_VIEW($1)` 在 `:17529`）。

**`will_contextualize` 标志**（`sql/sql_lex.h:4372`）：

```cpp
/**
  Used to inform the parser whether it should contextualize the parse
  tree. When we get a pure parser this will not be needed.
*/
bool will_contextualize;
```

- 初值 `true`（`sql_lex.cc:3604`）
- **现状**：8.0.39 服务端代码里**没有任何地方把它置 false**，唯一赋值在单测里（`unittest/gunit/character_set_deprecation-t.cc:44`）
- 设计意图：支持"纯语法解析、推迟 contextualize"（view 定义、PS 二次解析、SP 预解析）。注释 "When we get a pure parser this will not be needed" 说明这是历史遗留开关

---

## 三、Bison 生成物在哪：CMake 构建期代码生成

**`sql_yacc.cc` / `sql_yacc.h` 不在源码树里，是 CMake 配置阶段调用 Bison 从 `sql_yacc.yy` 生成的。** 这是"构建期代码生成"——你 clone 下来的源码只有 `.yy` 语法源文件，`.cc` 是 `cmake` 时现生成的。

### 两条构建路径（`sql/CMakeLists.txt:1306-1363`）

CMake 先用 `FIND_PACKAGE(BISON)`（`cmake/bison.cmake`）探测系统有没有 bison，然后二选一：

```cmake
IF(BISON_FOUND)                    # 有 bison（开发机 / git clone）
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT OFF)
ELSE()                             # 没 bison（release tarball 源码包）
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT ON)
ENDIF()
OPTION(USE_BISON_RESULTS_FROM_MAKE_DIST ...)
```

| 路径 | 触发条件 | 做法 |
|------|---------|------|
| **本地生成** | `BISON_FOUND`（装了 bison） | `BISON_TARGET(mysql_parser ...)` 从 `.yy` 现生成 `.cc` |
| **copy 预生成** | 没装 bison，或显式 `-DUSE_BISON_RESULTS_FROM_MAKE_DIST=ON` | 把 source tarball 里**已经预生成好的** `sql_yacc.cc/h` copy 到 build 目录 |

**为什么要预生成打进源码包**：发行版源码包要让用户"不装 bison 也能编译"，所以 release 时把生成物塞进 tarball（`CMakeLists.txt:1319-1333` 的 `FOREACH(genfile sql_yacc.h sql_yacc.cc ...)` 检查这些文件是否存在）。

**Bison 段落完整原文**（`sql/CMakeLists.txt:1305-1363`）：

```cmake
# We have already included cmake/bison.cmake, and done FIND_PACKAGE(BISON)
IF(BISON_FOUND)
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT OFF)
ELSE()
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT ON)
ENDIF()

# bison should be REQUIRED, but we have to support source tarball build.
# https://dev.mysql.com/doc/refman/8.0/en/source-installation.html
OPTION(USE_BISON_RESULTS_FROM_MAKE_DIST
  "Use bison results from source distribution tarball"
  ${USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT}
  )

IF(USE_BISON_RESULTS_FROM_MAKE_DIST)
  # No bison found, see if generated files are available as sources.
  FOREACH(genfile sql_yacc.h sql_yacc.cc sql_hints.yy.h sql_hints.yy.cc)
    IF(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${genfile}")
      MESSAGE(FATAL_ERROR "Cannot find ${CMAKE_CURRENT_SOURCE_DIR}/${genfile}")
    ENDIF()
    ADD_CUSTOM_COMMAND(
      OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${genfile}
      COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_CURRENT_SOURCE_DIR}/${genfile}
        ${CMAKE_CURRENT_BINARY_DIR}/${genfile}
      COMMENT "Copying file ${genfile}"
      VERBATIM
      )
  ENDFOREACH()
ELSE()
  # note: for parser
  BISON_TARGET(mysql_parser                                      # :1336
    ${CMAKE_CURRENT_SOURCE_DIR}/sql_yacc.yy
    ${CMAKE_CURRENT_BINARY_DIR}/sql_yacc.cc
    COMPILE_FLAGS
    "--name-prefix=MYSQL --yacc ${BISON_FLAGS_WARNINGS} ${BISON_NO_LINE_OPT}"  # :1340
    DEFINES_FILE ${CMAKE_CURRENT_BINARY_DIR}/sql_yacc.h          # :1341
    )                                                            # :1342

  BISON_TARGET(hints_parser                                      # :1344
    ${CMAKE_CURRENT_SOURCE_DIR}/sql_hints.yy
    ${CMAKE_CURRENT_BINARY_DIR}/sql_hints.yy.cc
    COMPILE_FLAGS
    "--name-prefix=HINT_PARSER_ --yacc ${BISON_FLAGS_WARNINGS} ${BISON_NO_LINE_OPT}"
    DEFINES_FILE ${CMAKE_CURRENT_BINARY_DIR}/sql_hints.yy.h
    )                                                            # :1350

  MESSAGE(STATUS "BISON outputs ${BISON_mysql_parser_OUTPUTS}")
  MESSAGE(STATUS "BISON outputs ${BISON_hints_parser_OUTPUTS}")

  # Some targets, for easier debugging.
  # The new Xcode build system does not support multiple targets that
  # DEPEND on the same outputs.
  IF(NOT APPLE_XCODE)
    ADD_CUSTOM_TARGET(GenBison_mysql DEPENDS ${BISON_mysql_parser_OUTPUTS})
    ADD_CUSTOM_TARGET(GenBison_hints DEPENDS ${BISON_hints_parser_OUTPUTS})
  ENDIF()

ENDIF()                                                          # :1363
```

> ⚠️ **勘误**：`sql/CMakeLists.txt` 里**没有** `FIND_PACKAGE(BISON)`。真正的探测在 **`cmake/bison.cmake:116`**，由顶层 `CMakeLists.txt:1938 INCLUDE(bison)` 引入（`sql/CMakeLists.txt:1305` 那句注释说的正是"已经做过了"）。
>
> ★ `dist` 目标显式依赖生成器（`sql/CMakeLists.txt:1442`）——打包前必须先生成：
> ```cmake
> ADD_CUSTOM_TARGET(dist
>   COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/make_dist.cmake
>   DEPENDS GenBison_mysql GenBison_hints
>   WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
> )
> ```

关键参数：

| 参数 | 作用 |
|------|------|
| `--name-prefix=MYSQL` | 所有 `yy*` 符号加前缀 ⇒ `yyparse→MYSQLparse`、`yylex→MYSQLlex`（见 02 篇） |
| `--yacc` | 兼容传统 yacc 语法（而不是 POSIX yacc） |
| `${BISON_FLAGS_WARNINGS}` | 警告开关 |
| `${BISON_NO_LINE_OPT}` | 控制 `#line` 指令 |
| `DEFINES_FILE` | 额外生成 `sql_yacc.h`（token 枚举 + `YYSTYPE` 声明，被 lexer/parser 代码 include） |

### 还有第二个 Bison 生成器

`sql/CMakeLists.txt:1344-1350` 还有一个 `hints_parser`（`sql_hints.yy`，`--name-prefix=HINT_PARSER_`），用于解析 `/*+ optimizer hint */` 注释里的 hint 语法，生成 `sql_hints.yy.cc`。它和主 parser 是两套独立的 LALR 状态机。

### 生成物长什么样

`sql_yacc.cc` 是 Bison 产出的**表驱动 parser**，含：
- **状态转移表**（`yypact`/`yydefact`/`yypgoto`/`yydefgoto`/`yytable`/`yycheck` 等常量数组）
- **归约动作**（`case N:` 里的语义动作代码，即 `.yy` 文件里 `{ }` 里的 `NEW_PTN`/`CONTEXTUALIZE` 等，被原样内联）
- **三个栈**（状态栈/语义值栈/位置栈，见本篇"八、Parse Tree 的内存管理"）

`sql_yacc.h` 是 token 定义头：`enum yytokentype`（`SELECT_SYM`/`FROM_SYM`/`IDENT`...）+ `YYSTYPE` 联合 + `MYSQLparse` 声明。lexer（`sql_lex.cc`）和 parser 都 include 它，共享 token 编号和语义值类型。

#### 两个 flag 的确切来源

```cmake
# cmake/bison.cmake:142 —— BISON_FLAGS_WARNINGS 的定义（CACHE INTERNAL）
# TODO(tdidriks): replace with "--warnings=all"
# Legacy backward compatibility suppressions:
# * no-yacc: for --yacc
# * no-precedence: for useless precedence or/and associativity rules
SET(BISON_FLAGS_WARNINGS
  "--warnings=all,no-yacc,no-precedence"
  CACHE INTERNAL "BISON 3.x flags")
```

> ⚠️ **勘误**：**`BISON_NO_LINE_OPT` 不在 `cmake/bison.cmake` 里**。它在顶层 **`CMakeLists.txt:1473`**，且**只在 `REPRODUCIBLE_BUILD` 打开时**才被设置为 `--no-lines`（否则未定义、展开为空）：
>
> ```cmake
> IF(REPRODUCIBLE_BUILD)                      # :1456
>   ...
>   SET(BISON_NO_LINE_OPT "--no-lines")       # :1473  禁止生成 #line 指令
>   SET_PROPERTY(GLOBAL PROPERTY RULE_LAUNCH_COMPILE
>     "${CMAKE_SOURCE_DIR}/scripts/invoke-with-relative-paths.pl")
> ENDIF()                                     # :1477
> ```
> 理由：`#line` 里的绝对路径会破坏可复现构建。消费者只有 4 处：`sql/CMakeLists.txt:669`（debug_lo_parser）、`:1340`（mysql_parser）、`:1348`（hints_parser）、`storage/ndb/tools/CMakeLists.txt:78`。

`cmake/bison.cmake` 还做了三件"找 bison"的事：macOS 上系统 bison 卡在 2.3，所以先在 `/opt/bison-3.8.2/bin` 与 Homebrew prefix 里找；Windows 上找 `winflexbison3`（`win_bison`）；最后才 `FIND_PACKAGE(BISON)`，并要求 **最低 3.0.4**（`MIN_BISON_VERSION_REQUIRED`，`cmake/bison.cmake:25`）。

#### `YYSTYPE` 与 `YYLTYPE`：都在别处定义

> ★ `sql_yacc.yy` 里**没有 `%union`**。`YYSTYPE` 是手写在 **`sql/parser_yystype.h:341-717`** 的 union（238 个成员），并有硬约束：

```cpp
static_assert(sizeof(YYSTYPE) <= 32, "YYSTYPE is too big");   // :719
```

> ★ 这个 32 字节上限正是 §9.3"能放指针就放指针"的原因——值型成员只能是 `int` / `ulonglong` / enum / 小 struct。union 开头几段：

```cpp
union YYSTYPE {
  Lexer_yystype lexer;  // terminal values from the lexical scanner
  /*
    Hint parser section (sql_hints.yy)
  */
  opt_hints_enum hint_type;
  PT_hint *hint;
  ...
  /*
    Main parser section (sql_yacc.yy)
  */
  int num;
  ulong ulong_num;
  ulonglong ulonglong_number;
  LEX_CSTRING lex_cstr;
  Table_ident *table;
  Item *item;
  Item_num *item_num;
  mem_root_deque<Item *> *item_list;
  ...
  Query_options select_options;
  PT_limit_clause *limit_clause;
  Parse_tree_node *node;
  PT_group *group;
  PT_order *order;
  PT_table_reference *table_reference;
  Mem_root_array_YY<PT_table_reference *> table_reference_list;   // ★ 值型（外壳在栈槽位）
  ...
  Parse_tree_root *top_level_node;
  ...
};
```

```cpp
// sql/parse_location.h:43 —— 位置类型也是自定义的，不是 %locations
struct Symbol_location {
  const char *start;  // token start
  const char *end;    // the 1st byte after the token
  bool is_empty() const { return length() == 0; }
  size_t length() const { return static_cast<size_t>(end - start); }
};

/**
  Bison "location" class
*/
struct YYLTYPE {
  Symbol_location cpp;  // token location in the preprocessed buffer
  Symbol_location raw;  // token location in the raw buffer
  bool is_empty() const { return cpp.is_empty(); }
};
typedef YYLTYPE POS;
#define YYLTYPE_IS_DECLARED 1  // signal Bison that we have our own YYLTYPE
```

> ★ **`cpp` vs `raw` 双位置**：`raw` 指向**客户端发来的原始 buffer**，`cpp` 指向预处理后的 buffer。`MYSQLerror` → `syntax_error_at(*location)` → `THD::vsyntax_error_at()` 用的是 **`raw`**（`sql_class.cc:2786`），因为报错要显示给用户看到的原文位置。这也解释了 `query_specification` 里那个 `@5.raw.is_empty()`——它判断的是"原始 SQL 里有没有写 FROM"。

---

## 四、LEX 结构

> 注意：8.0 里 `LEX` 是 **`struct`** 不是 class：

```cpp
// sql/sql_lex.h:3876
struct LEX : public Query_tables_list {
  friend bool lex_start(THD *thd);

  Query_expression *unit;                 ///< 最外层 Query_expression    :3879
  Query_block *query_block;               ///< 第一个 Query_block         :3881
  Query_block *all_query_blocks_list;     ///< 全部 Query_block 的链表     :3882
 private:
  Query_block *m_current_query_block;     ///< 解析期"当前"QB             :3885
 public:
  inline Query_block *current_query_block() const { ... }        // :3888
  inline void set_current_query_block(Query_block *select) { ... } // :3897
```

| 成员 | 位置 |
|------|------|
| `unit` | `sql/sql_lex.h:3879` |
| `query_block` | `:3881` |
| `all_query_blocks_list` | `:3882` |
| `m_current_query_block` + 访问器 | `:3885 / 3888 / 3897` |
| `sql_command` | `:2675`（在基类 **`class Query_tables_list`**，`sql_lex.h:2665`） |
| `m_sql_cmd` | `:4090` |
| `parsing_options`（类型 `st_parsing_options`） | 成员 `:4182`；类型 `:3306-3312` |
| `will_contextualize` | `:4372` |
| `make_sql_cmd` 声明 | `:4562` |

> ⚠️ **勘误**：`LEX` 中**不存在** `m_with_clause`。WITH 子句挂在 **`Query_expression::m_with_clause`**（`sql/sql_lex.h:967`）；`LEX` 只有前向声明 `class PT_with_clause;`（`sql_lex.h:112`）。

**`struct LEX` 完整成员区**（`sql/sql_lex.h:3876-4626`；只省略了与解析无关的权限/DDL/binlog 字段，解析期主线全部保留）：

```cpp
struct LEX : public Query_tables_list {                          // ★ 关键字就是 struct
  friend bool lex_start(THD *thd);

  Query_expression *unit;                 ///< Outer-most query expression
  /// @todo: query_block can be replaced with unit->first-select()
  Query_block *query_block;               ///< First query block
  Query_block *all_query_blocks_list;     ///< List of all query blocks
 private:
  /* current Query_block in parsing */
  Query_block *m_current_query_block;     // ★ 真正的"当前 QB"，私有
 public:
  inline Query_block *current_query_block() const {
    return m_current_query_block;
  }
  /*
    We want to keep current_thd out of header files, so the debug assert
    is moved to the .cc file.
  */
  void assert_ok_set_current_query_block();
  inline void set_current_query_block(Query_block *select) {
#ifndef NDEBUG
    assert_ok_set_current_query_block();
#endif
    m_current_query_block = select;
  }
  bool is_explain() const { return explain_format != nullptr; }
  bool is_explain_analyze = false;

  /**
    Whether the currently-running query should be (attempted) executed in
    the hypergraph optimizer. This will not change after the query is
    done parsing, ...
   */
  bool using_hypergraph_optimizer() const {
    return m_using_hypergraph_optimizer;
  }
  ...
  THD *thd;                                // ★ make_sql_cmd 里用它传给 make_cmd(thd)

  /* Optimizer hints */
  Opt_hints_global *opt_hints_global;
  ...
  /// Table being inserted into (may be a view)
  Table_ref *insert_table;
  /// Leaf table being inserted into (always a base table)
  Table_ref *insert_table_leaf;
  ...
  /**
    List of placeholders ('?') for parameters of a prepared statement. Because
    we append to this list during parsing, it is naturally sorted by
    position of the '?' in the query string. ...
  */
  List<Item_param> param_list;
  ...
  /*
    A stack of name resolution contexts for the query. This stack is used
    at parse time to set local name resolution contexts for various parts
    of a query. For example, in a JOIN ... ON (some_condition) clause the
    Items in 'some_condition' must be resolved only against the operands
    of the the join, and not against the whole clause. ...
  */
  List<Name_resolution_context> context_stack;   // ★ 名字解析上下文栈
  ...
  /**
    This field is used as a work field during resolving to validate
    the use of aggregate functions. ...
    Each query block is assigned a nesting level. This field is a bit field
    that contains the value one in the position of that nesting level if
    aggregate functions are allowed for that query block.
  */
  nesting_map allow_sum_func;
  /**
    Windowing functions are not allowed in HAVING ... Nor are they allowed
    inside grouped aggregate function arguments. One bit per query block ...
  */
  nesting_map m_deny_window_func;
  ...
  Sql_cmd *m_sql_cmd;                      // ★ make_sql_cmd() 的产物落在这里

  /*
    Usually `expr` rule of yacc is quite reused but some commands better
    not support subqueries which comes standard with this rule, like
    KILL, HA_READ, CREATE/ALTER EVENT etc. Set this to `false` to get
    syntax error back.
  */
  bool expr_allows_subquery{true};
  /**
    If currently re-parsing a CTE's definition, this is the offset in bytes
    of that definition in the original statement which had the WITH
    clause. Otherwise this is 0.
  */
  uint reparse_common_table_expr_at;
  bool reparse_derived_table_condition{false};
  std::vector<uint> reparse_derived_table_params_at;
  ...
  st_parsing_options parsing_options;      // ★ 解析期开关（见下面类型定义）
  ...
 private:
  /**
    Set to true when execution has started (after parsing, tables opened and
    query preparation is complete. Used to track arena state for SPs).
  */
  bool m_exec_started;
  bool m_exec_completed;
  sp_pcontext *sp_current_parsing_ctx;
  ulonglong m_statement_options{0};
 public:
  bool is_exec_started() const { return m_exec_started; }
  void set_exec_started() { m_exec_started = true; }
  ...
  /**
    Used to inform the parser whether it should contextualize the parse
    tree. When we get a pure parser this will not be needed.
  */
  bool will_contextualize;                 // ★ CONTEXTUALIZE/ITEMIZE/make_sql_cmd 的守门开关

  LEX();
  virtual ~LEX();

  /// Destroy contained objects, but not the LEX object itself.
  void destroy() {
    if (unit == nullptr) return;
    unit->destroy();
    unit = nullptr;
    query_block = nullptr;
    all_query_blocks_list = nullptr;
    m_current_query_block = nullptr;
    destroy_values_map();
  }

  /// Reset query context to initial state
  void reset();

  /// Create an empty query block within this LEX object.
  Query_block *new_empty_query_block();

  /// Create query expression object that contains one query block.
  Query_block *new_query(Query_block *curr_query_block);

  /// Create query block and attach it to the current query expression.
  Query_block *new_set_operation_query(Query_block *curr_query_block);

  /// Create top-level query expression and query block.
  bool new_top_level_query();

  /// Create query expression and query block in existing memory objects.
  void new_static_query(Query_expression *sel_query_expression,
                        Query_block *select);
  Query_expression *create_query_expr_and_block(
      THD *thd, Query_block *current_query_block, Item *where_clause,
      Item *having_clause, enum_parsing_context ctx);
  ...
  /**
    @brief check if the statement is a single-level join
    @return true  The statement doesn't contain subqueries, unions and
                  stored procedure calls.
  */
  bool is_single_level_stmt() {
    /*
      This check exploits the fact that the last added to all_select_list is
      on its top. So query_block (as the first added) will be at the tail
      of the list.
    */
    if (query_block == all_query_blocks_list &&
        (sroutines == nullptr || sroutines->empty())) {
      assert(!all_query_blocks_list->next_select_in_list());
      return true;
    }
    return false;
  }
  ...
  bool make_sql_cmd(Parse_tree_root *parse_tree);   // ★ THD::sql_parser 的第二步
};
```

**`st_parsing_options` 只有两个 bool**（`sql/sql_lex.h:3301`）：

```cpp
/*
  st_parsing_options contains the flags for constructions that are
  allowed in the current statement.
*/
struct st_parsing_options {
  bool allows_variable;
  bool allows_select_into;

  st_parsing_options() { reset(); }
  void reset();
};

// sql/sql_lex.cc:179
void st_parsing_options::reset() {
  allows_variable = true;
  allows_select_into = true;
}
```

**`will_contextualize` 的初值在构造函数，不在 `lex_start()`**（`sql/sql_lex.cc:3585`）：

```cpp
LEX::LEX()
    : unit(nullptr),
      query_block(nullptr),
      all_query_blocks_list(nullptr),
      m_current_query_block(nullptr),
      result(nullptr),
      thd(nullptr),
      opt_hints_global(nullptr),
      plugins(PSI_NOT_INSTRUMENTED),
      insert_update_values_map(nullptr),
      option_type(OPT_DEFAULT),
      drop_temporary(false),
      sphead(nullptr),
      contains_plaintext_password(false),
      keep_diagnostics(DA_KEEP_UNSPECIFIED),
      is_lex_started(false),
      in_update_value_clause(false),
      will_contextualize(true) {                 // ★★ 唯一初值赋值处
  reset_query_tables_list(true);
}
```

> ⚠️ **勘误**：`lex_start()` 本体在 **`sql/sql_lex.cc:511`**（不是 3604），且它调的 `LEX::reset()`（`:418`）**不碰** `will_contextualize`。

```cpp
// sql/sql_lex.cc:502
/**
  Call lex_start() before every query that is to be prepared and executed.
  Because of this, it's critical not to do too many things here. ...
*/
bool lex_start(THD *thd) {                       // ★ 本体在 :511
  DBUG_TRACE;
  LEX *lex = thd->lex;

  lex->thd = thd;                                // ★ LEX::thd 的绑定点（make_sql_cmd 要用）
  lex->reset();                                  // ★ reset() 不重置 will_contextualize
  thd->init_cost_model();

  const bool status = lex->new_top_level_query();   // ★ 建出顶层 unit + query_block
  assert(lex->current_query_block() == nullptr);
  lex->m_current_query_block = lex->query_block;    // ★ 绕过 setter —— 因为 lex_start 是 friend
  ...
  return status;
}
```

**`LEX::make_sql_cmd`（在 `:4962`，不是 4969）**：

```cpp
/**
  Uses parse_tree to instantiate an Sql_cmd object and assigns it to the Lex.
  ...
*/
bool LEX::make_sql_cmd(Parse_tree_root *parse_tree) {
  if (!will_contextualize) return false;            // ★ 只做语法检查时直接跳过，不生成 Sql_cmd

  m_sql_cmd = parse_tree->make_cmd(thd);            // ★★ contextualize 发生在 make_cmd() 内部
  if (m_sql_cmd == nullptr) return true;

  assert(m_sql_cmd->sql_command_code() == sql_command);   // ★ 解析期设的 sql_command 必须一致

  return false;
}
```

**`THD::sql_parser()` 完整源码**（`sql/sql_class.cc:3048`，★ 不是 3061）：

```cpp
bool THD::sql_parser() {
  /*
    SQL parser function generated by YACC from sql_yacc.yy.
    ...
    The second (output) parameter "root" returns the new parse tree.
    It is undefined (unchanged) on error. If "root" is NULL on success,
    then the parser has already called lex->make_sql_cmd() internally.
  */
  extern int MYSQLparse(class THD * thd, class Parse_tree_root * *root);
  // ★ 签名与 %parse-param 一一对应（:525/:526）

  Parse_tree_root *root = nullptr;                  // :3061 局部出参
  if (MYSQLparse(this, &root) || is_error()) {      // ★ 第一步：token 流 → Parse Tree
    /*
      Restore the original LEX if it was replaced when parsing
      a stored procedure. We must ensure that a parsing error
      does not leave any side effects in the THD.
    */
    cleanup_after_parse_error();
    return true;
  }
  if (root != nullptr && lex->make_sql_cmd(root)) { // ★★ 第二步：Parse Tree → AST（含 contextualize）
    return true;
  }
  return false;
}
```

解析期被反复调用的四个构造入口（`sql/sql_lex.cc`）：

```cpp
// :587
Query_block *LEX::new_empty_query_block() {
  Query_block *select =
      new (thd->mem_root) Query_block(thd->mem_root, nullptr, nullptr);
  if (select == nullptr) return nullptr;
  select->parent_lex = this;
  return select;
}

// :597 —— 只创建最顶层的一个 QE 和一个 QB
Query_expression *LEX::create_query_expr_and_block(
    THD *thd, Query_block *current_query_block, Item *where, Item *having,
    enum_parsing_context ctx) {
  if (current_query_block != nullptr &&
      current_query_block->nest_level >= MAX_SELECT_NESTING) {
    my_error(ER_TOO_HIGH_LEVEL_OF_NESTING_FOR_SELECT, MYF(0));
    return nullptr;
  }
  auto *const new_expression = new (thd->mem_root) Query_expression(ctx);
  if (new_expression == nullptr) return nullptr;
  auto *const new_query_block =
      new (thd->mem_root) Query_block(thd->mem_root, where, having);
  if (new_query_block == nullptr) return nullptr;

  // Link the new query expression below the current query block, if any
  if (current_query_block != nullptr)
    new_expression->include_down(this, current_query_block);  // 设置 master/slave 关系
  new_query_block->include_down(this, new_expression);

  new_query_block->parent_lex = this;
  new_query_block->include_in_global(&this->all_query_blocks_list);
  new_expression->set_query_term(new_query_block);
  return new_expression;
}

// :773
bool LEX::new_top_level_query() {
  assert(unit == nullptr && query_block == nullptr);
  assert(result == nullptr);

  query_block = new_query(nullptr);                 // nullptr = 最外层 QB
  if (query_block == nullptr) return true;
  unit = query_block->master_query_expression();    // unit 由 QB 反查得到
  return false;
}

// :695 —— UNION/EXCEPT/INTERSECT 的第二个及以后分支
Query_block *LEX::new_set_operation_query(Query_block *curr_query_block) {
  assert(unit != nullptr && query_block != nullptr);
  bool const outer_most = curr_query_block->master_query_expression() == unit;
  /*
     Only the last SELECT can have INTO. ...
  */
  if (outer_most && result) {
    my_error(ER_MISPLACED_INTO, MYF(0));
    return nullptr;
  }
  Query_block *const select = new_empty_query_block();
  if (!select) return nullptr;
  select->include_neighbour(this, curr_query_block);   // ★ 挂成"兄弟"QB，共享同一个 QE
  ...
  return select;
}
```

---

## 五、关键产生式速查

★ 下行号已逐条校正（原表有约 6 行的系统性偏移）：

| 语法结构 | 产生式行号 | 产出的 PT 对象 |
|----------|-----------|---------------|
| **`start_entry`**（`%start`） | `:2238-2282` | **6 个分支**，只有 `sql_statement` 走正常 SQL |
| `sql_statement` | `:2284-2327` | 3 个分支（空 / `;` 结尾 / 无 `;`） |
| **输出根节点** | **`:2335`** | `*parse_tree = $1`（在 `simple_statement_or_begin:2334-2337`） |
| `select_stmt` | `:9830` | `PT_select_stmt`（`:9833`）；带 locking 版 `:9836` |
| `select_stmt_with_into` | `:9875` | INTO 二义性大注释 `:9843-9874` |
| `query_expression` | `:9926` | `PT_query_expression`（`:9931`）；带 WITH 版 `:9937` |
| `query_expression_body` | `:9942` | `PT_union`(9953) / `PT_except`(9958) / `PT_intersect`(9963) |
| `query_expression_parens` | `:9968` | |
| `query_primary` | `:9973` | |
| **`query_specification`** | **`:9994`** | `PT_query_specification`（`:10007` / `:10029`） |
| `opt_from_clause` / `from_clause` / `from_tables` | `:10040 / 10045 / 10048` | |
| `select_item_list` | `:10166` | 3 分支，含 `'*'` |
| `select_item` | `:10188` | `PTI_expr_with_alias`（`:10192`） |
| **`expr`** | **`:10211`** | 10 分支 |
| `bool_pri` | `:10260` | `PTI_comp_op` 在 `:10271` |
| `predicate` | `:10289` | 16 分支 |
| `bit_expr` | `:10385` | 13 分支 |
| `simple_expr` | `:10483` | 30 分支 |
| `table_reference` | `:11845` | 3 分支（含 ODBC `{oj ...}`） |
| **`joined_table`** | **`:11942`** | `PT_joined_table_on`(11945/11955) / `using`(11949/11959) / `cross_join`(11965)；**结合性大注释 `:11864-11941`** |
| `table_factor` | `:12042` | 6 分支（含 `table_function` 12050） |
| `single_table` | `:12068` | `PT_table_factor_table_ident`（`:12071`） |
| `derived_table` | `:12080` | `PT_derived_table`（`:12090` / `:12100`）；LATERAL 版 |
| `where_clause` | `:12395` | `PTI_where`（`:12396`） |
| `opt_having_clause` | `:12399` | `PTI_having`（`:12403`） |
| `order_clause` / `order_list` | `:12586 / 12593` | `PT_order`(12589) / `PT_order_list`(12601) |
| `group_list` | `:12527` | |
| `opt_limit_clause` | `:12618` | |
| `order_expr` | `:14839` | `PT_order_expr`（`:14842`） |
| `grouping_expr` | `:14846` | `PT_order_expr`（方向恒 `ORDER_NOT_RELEVANT`） |
| `simple_ident` | `:14853` | **只有 2 分支**；`PTI_simple_ident_ident`（`:14856`） |
| `simple_ident_q` | `:14869` | `PTI_simple_ident_q_2d`(14872) / `_q_3d`(14878) |
| `row_subquery` / `table_subquery` / `subquery` | `:17322 / 17326 / 17330` | `PT_subquery`（`:17333`） |

先看一条最能说明"PT 节点长什么样"的产生式——`query_specification` 的**无 INTO 分支**（`:10018-10038`），它的语义动作把 8 个语法符号一次性塞进 `PT_query_specification`：

```yacc
// sql/sql_yacc.yy:10018
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
            // ★★ @4.raw.is_empty() —— 原始 SQL 里没写 FROM ⇒ "隐式 FROM"（SELECT 1 走无表路径）
          }
        ;
```

同层的 `predicate`（`:10289`，16 分支）是 `IN` / `BETWEEN` / `LIKE` / `REGEXP` / `MEMBER OF` 的所在：

```yacc
predicate:
          bit_expr IN_SYM table_subquery
          { $$= NEW_PTN Item_in_subselect(@$, $1, $3); }
        | bit_expr not IN_SYM table_subquery
          {
            Item *item= NEW_PTN Item_in_subselect(@$, $1, $4);
            $$= NEW_PTN PTI_truth_transform(@$, item, Item::BOOL_NEGATED);
          }
        | bit_expr IN_SYM '(' expr ')'                          // ★ SQL2003 note184 单值例外
          { $$= NEW_PTN PTI_handle_sql2003_note184_exception(@$, $1, true, $4); }
        | bit_expr IN_SYM '(' expr ',' expr_list ')'
          {
            if ($6 == NULL || $6->push_front($4) || $6->push_front($1))
              MYSQL_YYABORT;
            $$= NEW_PTN Item_func_in(@$, $6, false);
          }
        | bit_expr not IN_SYM '(' expr ')'                       { ... }
        | bit_expr not IN_SYM '(' expr ',' expr_list ')'
          { ... $$= NEW_PTN Item_func_in(@$, $7, true); }        // true = NOT IN
        | bit_expr MEMBER_SYM opt_of '(' simple_expr ')'         // ★ 8.0 新增：x MEMBER OF (json_array)
          { $$= NEW_PTN Item_func_member_of(@$, $1, $5); }
        | bit_expr BETWEEN_SYM bit_expr AND_SYM predicate        // ★ 上界用 predicate 而非 bit_expr
          { $$= NEW_PTN Item_func_between(@$, $1, $3, $5, false); }
        | bit_expr not BETWEEN_SYM bit_expr AND_SYM predicate
          { $$= NEW_PTN Item_func_between(@$, $1, $4, $6, true); }
        | bit_expr SOUNDS_SYM LIKE bit_expr                      // ★ SOUNDS LIKE ⇒ soundex(a)=soundex(b)
          {
            Item *item1= NEW_PTN Item_func_soundex(@$, $1);
            Item *item4= NEW_PTN Item_func_soundex(@$, $4);
            if ((item1 == NULL) || (item4 == NULL)) MYSQL_YYABORT;
            $$= NEW_PTN Item_func_eq(@$, item1, item4);
          }
        | bit_expr LIKE simple_expr     { $$ = NEW_PTN Item_func_like(@$, $1, $3); }
        | bit_expr LIKE simple_expr ESCAPE_SYM simple_expr %prec LIKE
          { $$ = NEW_PTN Item_func_like(@$, $1, $3, $5); }
        | bit_expr not LIKE simple_expr
          { auto item = NEW_PTN Item_func_like(@$, $1, $4);
            $$ = NEW_PTN Item_func_not(@$, item); }
        | bit_expr not LIKE simple_expr ESCAPE_SYM simple_expr %prec LIKE { ... }
        | bit_expr REGEXP bit_expr                               // ★ 8.0 起 RLIKE/REGEXP 统一走 regexp_like
          {
            auto args= NEW_PTN PT_item_list;
            args->push_back($1);
            args->push_back($3);
            $$= NEW_PTN Item_func_regexp_like(@1, args);
          }
        | bit_expr not REGEXP bit_expr
          {
            ...
            Item *item= NEW_PTN Item_func_regexp_like(@$, args);
            $$= NEW_PTN PTI_truth_transform(@$, item, Item::BOOL_NEGATED);
          }
        | bit_expr %prec SET_VAR
        ;
```

> ★ **`BETWEEN` 的上界用 `predicate` 而不是 `bit_expr`**——所以 `a BETWEEN 1 AND 2 AND 3` 能解析（AND 被 BETWEEN 吃掉当上界），这是 SQL 标准要求的（`BETWEEN` 优先级低于 `AND` 的结果）。

### 5.1 入口三连：`start_entry` → `sql_statement` → `simple_statement_or_begin`

> ⚠️ **勘误**：`*parse_tree = $1;` **不在** `start_entry` 里，而在 **`simple_statement_or_begin:2335`**。`start_entry` 的另外 5 个分支由 `GRAMMAR_SELECTOR_*` 合成 token 触发，是**内部子解析器入口**（表达式 / 分区 / 生成列 / CTE / 派生表），结果写 `YYP->result`，**不写 `parse_tree`**。

```yacc
// sql/sql_yacc.yy:2238
start_entry:
          sql_statement                                        // ① 正常 SQL
        | GRAMMAR_SELECTOR_EXPR bit_expr END_OF_INPUT          // ② 只解析一个表达式
          {
            ITEMIZE($2, &$2);
            static_cast<Expression_parser_state *>(YYP)->result= $2;
          }
        | GRAMMAR_SELECTOR_PART partition_clause END_OF_INPUT  // ③ PARTITION BY 串
          {
            /*
              We enter here when translating partition info string into
              partition_info data structure.
            */
            CONTEXTUALIZE($2);
            static_cast<Partition_expr_parser_state *>(YYP)->result=
              &$2->part_info;
          }
        | GRAMMAR_SELECTOR_GCOL IDENT_sys '(' expr ')' END_OF_INPUT   // ④ 生成列
          {
            // Check gcol expression for the "PARSE_GCOL_EXPR" prefix:
            if (!is_identifier($2, "PARSE_GCOL_EXPR"))
              MYSQL_YYABORT;
            auto gcol_info= NEW_PTN Value_generator;
            if (gcol_info == NULL)
              MYSQL_YYABORT; // OOM
            ITEMIZE($4, &$4);
            gcol_info->expr_item= $4;
            static_cast<Gcol_expr_parser_state *>(YYP)->result= gcol_info;
          }
        | GRAMMAR_SELECTOR_CTE table_subquery END_OF_INPUT      // ⑤ CTE 定义体
          {
            static_cast<Common_table_expr_parser_state *>(YYP)->result= $2;
          }
        | GRAMMAR_SELECTOR_DERIVED_EXPR expr END_OF_INPUT       // ⑥ 派生表/视图表达式
         {
           ITEMIZE($2, &$2);
           static_cast<Derived_expr_parser_state *>(YYP)->result= $2;
         }
        ;
```

```yacc
// sql/sql_yacc.yy:2284
sql_statement:
          END_OF_INPUT
          {
            THD *thd= YYTHD;
            if (!thd->is_bootstrap_system_thread() &&
                !thd->m_parser_state->has_comment())
            {
              my_error(ER_EMPTY_QUERY, MYF(0));   // ★ 只有 bootstrap 线程或"纯注释"串才允许空
              MYSQL_YYABORT;
            }
            thd->lex->sql_command= SQLCOM_EMPTY_QUERY;
            YYLIP->found_semicolon= NULL;
          }
        | simple_statement_or_begin
          {
            Lex_input_stream *lip = YYLIP;
            if (YYTHD->get_protocol()->has_client_capability(CLIENT_MULTI_QUERIES) &&
                lip->multi_statements && !lip->eof())
            {
              /*
                We found a well formed query, and multi queries are allowed:
                - force the parser to stop after the ';'
                - mark the start of the next query for the next invocation
                  of the parser.
              */
              lip->next_state= MY_LEX_END;        // ★ 多语句：记下分号位置，下次 parse 从这里继续
              lip->found_semicolon= lip->get_ptr();
            }
            else
            {
              /* Single query, terminated. */
              lip->found_semicolon= NULL;
            }
          }
          ';'
          opt_end_of_input
        | simple_statement_or_begin END_OF_INPUT
          {
            /* Single query, not terminated. */
            YYLIP->found_semicolon= NULL;
          }
        ;

opt_end_of_input:
          %empty
        | END_OF_INPUT
        ;

// ★★ 唯一的解析树出口
simple_statement_or_begin:
          simple_statement      { *parse_tree= $1; }   // :2335
        | begin_stmt                                    // :2336 BEGIN 不产生 parse_tree
        ;
```

> ★ `begin_stmt` 分支**不赋值** `*parse_tree`（保持 NULL），因为 `BEGIN...END` 存储过程体会被塞进 `lex->sphead`。这与 `THD::sql_parser()` 的注释对应：*"If 'root' is NULL on success, then the parser has already called lex->make_sql_cmd() internally."*

### 5.2 SELECT 骨架

```yacc
// :9830
select_stmt:
          query_expression
          {
            $$ = NEW_PTN PT_select_stmt($1);
          }
        | query_expression locking_clause_list
          {
            $$ = NEW_PTN PT_select_stmt(NEW_PTN PT_locking($1, $2),
                                        nullptr, true);
          }
        | select_stmt_with_into
        ;

// :9875
select_stmt_with_into:
          '(' select_stmt_with_into ')'   { $$ = $2; }
        | query_expression into_clause
          {
            $$ = NEW_PTN PT_select_stmt($1, $2);
          }
        | query_expression into_clause locking_clause_list
          {
            $$ = NEW_PTN PT_select_stmt(NEW_PTN PT_locking($1, $3), $2, true);
          }
        | query_expression locking_clause_list into_clause
          {
            $$ = NEW_PTN PT_select_stmt(NEW_PTN PT_locking($1, $2), $3);
          }
        ;

// :9926
query_expression:
          query_expression_body
          opt_order_clause
          opt_limit_clause
          {
            $$ = NEW_PTN PT_query_expression($1.body, $2, $3);
          }
        | with_clause
          query_expression_body
          opt_order_clause
          opt_limit_clause
          {
            $$= NEW_PTN PT_query_expression($1, $2.body, $3, $4);
          }
        ;

// :9942 —— 值类型是 {body, is_parenthesized} 二元组
query_expression_body:
          query_primary                       { $$ = {$1, false}; }
        | query_expression_parens %prec SUBQUERY_AS_EXPR
                                                { $$ = {$1, true}; }   // ★ 假 token 压掉 S/R 冲突
        | query_expression_body UNION_SYM union_option query_expression_body
          {
            $$ = {NEW_PTN PT_union($1.body, $3, $4.body, $4.is_parenthesized), false};
          }
        | query_expression_body EXCEPT_SYM union_option query_expression_body
          {
            $$ = {NEW_PTN PT_except($1.body, $3, $4.body, $4.is_parenthesized), false};
          }
        | query_expression_body INTERSECT_SYM union_option query_expression_body
          {
            $$ = {NEW_PTN PT_intersect($1.body, $3, $4.body, $4.is_parenthesized), false};
          }
        ;
```

> ★ `query_expression` 上方 `:9894-9925` 有一段 32 行注释，解释 `(SELECT ...)` 到底是"带括号的 query expression"还是"子查询"——**靠 `SUBQUERY_AS_EXPR` 这个假 token 解决**：它优先级低于括号，于是 parser 会尽量吃括号；只有看到 `+` 这类 token 才认定是子查询。同一段注释还说明 **WITH 规则为什么写成两条而不是 `opt_with_clause`**——否则 Bison 会把两条塞进同一状态，被迫提前归约。

### 5.3 表达式优先级链：`expr` → `bool_pri` → `predicate` → `bit_expr` → `simple_expr`

这条链是"SQL 表达式优先级如何落进文法"的关键——**每一层向下传递时都用 `%prec SET_VAR`**（SET_VAR 是优先级阶梯里最低的假 token）。

```yacc
// :10211（10 分支）
expr:
          expr or expr %prec OR_SYM
          {
            $$= flatten_associative_operator<Item_cond_or,
                                             Item_func::COND_OR_FUNC>(
                                                 YYMEM_ROOT, @$, $1, $3);
            // ★ 把 a OR b OR c 摊平成 n 元 Item_cond_or，而不是嵌套二元树
          }
        | expr XOR expr %prec XOR
          {
            /* XOR is a proprietary extension */
            $$ = NEW_PTN Item_func_xor(@$, $1, $3);
          }
        | expr and expr %prec AND_SYM
          {
            $$= flatten_associative_operator<Item_cond_and,
                                             Item_func::COND_AND_FUNC>(
                                                 YYMEM_ROOT, @$, $1, $3);
          }
        | NOT_SYM expr %prec NOT_SYM
          { $$= NEW_PTN PTI_truth_transform(@$, $2, Item::BOOL_NEGATED); }
        | bool_pri IS TRUE_SYM %prec IS
          { $$= NEW_PTN PTI_truth_transform(@$, $1, Item::BOOL_IS_TRUE); }
        | bool_pri IS not TRUE_SYM %prec IS  { ... Item::BOOL_NOT_TRUE ... }
        | bool_pri IS FALSE_SYM %prec IS     { ... Item::BOOL_IS_FALSE ... }
        | bool_pri IS not FALSE_SYM %prec IS { ... Item::BOOL_NOT_FALSE ... }
        | bool_pri IS UNKNOWN_SYM %prec IS   { $$= NEW_PTN Item_func_isnull(@$, $1); }
        | bool_pri IS not UNKNOWN_SYM %prec IS { $$= NEW_PTN Item_func_isnotnull(@$, $1); }
        | bool_pri %prec SET_VAR             // ★ 向下传递
        ;

// :10260（6 分支）
bool_pri:
          bool_pri IS NULL_SYM %prec IS      { $$= NEW_PTN Item_func_isnull(@$, $1); }
        | bool_pri IS not NULL_SYM %prec IS  { $$= NEW_PTN Item_func_isnotnull(@$, $1); }
        | bool_pri comp_op predicate
          {
            $$= NEW_PTN PTI_comp_op(@$, $1, $2, $3);
            // ★ $2 是"比较器创建函数指针"（comp_eq_creator 等，见 comp_op:10468-10476）
          }
        | bool_pri comp_op all_or_any table_subquery %prec EQ
          {
            if ($2 == &comp_equal_creator)
              /*
                We throw this manual parse error rather than split the rule
                comp_op into a null-safe and a non null-safe rule, since doing
                so would add a shift/reduce conflict. ...
              */
              YYTHD->syntax_error_at(@2);     // ★ 禁止 "<=> ANY/ALL(...)"——手工报错而非拆规则
            $$= NEW_PTN PTI_comp_op_all(@$, $1, $2, $3, $4);
          }
        | predicate %prec SET_VAR
        ;
```

```yacc
// :10385（13 分支）—— 注意加减与位运算"同级"都写在 bit_expr 里，靠 %left 顺序区分
bit_expr:
          bit_expr '|' bit_expr %prec '|'            { $$= NEW_PTN Item_func_bit_or(@$, $1, $3); }
        | bit_expr '&' bit_expr %prec '&'            { $$= NEW_PTN Item_func_bit_and(@$, $1, $3); }
        | bit_expr SHIFT_LEFT bit_expr %prec SHIFT_LEFT  { ... }
        | bit_expr SHIFT_RIGHT bit_expr %prec SHIFT_RIGHT { ... }
        | bit_expr '+' bit_expr %prec '+'            { $$= NEW_PTN Item_func_plus(@$, $1, $3); }
        | bit_expr '-' bit_expr %prec '-'            { $$= NEW_PTN Item_func_minus(@$, $1, $3); }
        | bit_expr '+' INTERVAL_SYM expr interval %prec '+'
          { $$= NEW_PTN Item_date_add_interval(@$, $1, $4, $5, 0); }
        | bit_expr '-' INTERVAL_SYM expr interval %prec '-'
          { $$= NEW_PTN Item_date_add_interval(@$, $1, $4, $5, 1); }
        | bit_expr '*' bit_expr %prec '*'            { $$= NEW_PTN Item_func_mul(@$, $1, $3); }
        | bit_expr '/' bit_expr %prec '/'            { $$= NEW_PTN Item_func_div(@$, $1, $3); }
        | bit_expr '%' bit_expr %prec '%'            { $$= NEW_PTN Item_func_mod(@$, $1, $3); }
        | bit_expr DIV_SYM bit_expr %prec DIV_SYM    { $$= NEW_PTN Item_func_div_int(@$, $1, $3); }
        | bit_expr MOD_SYM bit_expr %prec MOD_SYM    { $$= NEW_PTN Item_func_mod(@$, $1, $3); }
        | bit_expr '^' bit_expr                      { $$= NEW_PTN Item_func_bit_xor(@$, $1, $3); }
        //  ★ '^' 是位异或，不是幂
        | simple_expr %prec SET_VAR
        ;
```

```yacc
// :10483（30 分支，节选关键分支）
simple_expr:
          simple_ident
        | function_call_keyword
        | function_call_nonkeyword
        | function_call_generic
        | function_call_conflict
        | simple_expr COLLATE_SYM ident_or_text %prec NEG     // ★ COLLATE 用最高优先级 NEG，紧贴表达式
          {
            warn_on_deprecated_user_defined_collation(YYTHD, $3);
            $$= NEW_PTN Item_func_set_collation(@$, $1, $3);
          }
        | literal_or_null
        | param_marker { $$= $1; }                            // ★ '?' 占位符
        | set_function_specification                          // SUM/COUNT 等聚合
        | window_func_call
        | simple_expr OR_OR_SYM simple_expr                    // ★ 8.0 起 '||' 是字符串拼接
          { $$= NEW_PTN Item_func_concat(@$, $1, $3); }
        | '+' simple_expr %prec NEG   { $$= $2; }              // 一元 +（直接丢弃）
        | '-' simple_expr %prec NEG   { $$= NEW_PTN Item_func_neg(@$, $2); }
        | '~' simple_expr %prec NEG   { $$= NEW_PTN Item_func_bit_neg(@$, $2); }
        | not2 simple_expr %prec NEG  { $$= NEW_PTN PTI_truth_transform(@$, $2, Item::BOOL_NEGATED); }
        | row_subquery                { $$= NEW_PTN PTI_singlerow_subselect(@$, $1); }
        | '(' expr ')'                { $$= $2; }
        | '(' expr ',' expr_list ')'  { $$= NEW_PTN Item_row(@$, $2, $4->value); }
        | EXISTS table_subquery       { $$= NEW_PTN PTI_exists_subselect(@$, $2); }
        | '{' ident expr '}'          { $$= NEW_PTN PTI_odbc_date(@$, $2, $3); }   // ODBC 日期转义
        | MATCH ident_list_arg AGAINST '(' bit_expr fulltext_options ')'
          { $$= NEW_PTN Item_func_match(@$, $2, $5, $6); }
        | BINARY_SYM simple_expr %prec NEG
          {
            push_deprecated_warn(YYTHD, "BINARY expr", "CAST");   // ★ 已废弃，提示用 CAST
            $$= create_func_cast(YYTHD, @2, $2, ITEM_CAST_CHAR, &my_charset_bin);
          }
        | CAST_SYM '(' expr AS cast_type opt_array_cast ')'
          { $$= create_func_cast(YYTHD, @3, $3, $5, $6); }
        | CASE_SYM opt_expr when_list opt_else END
          { $$= NEW_PTN Item_func_case(@$, $3, $2, $4); }
        | DEFAULT_SYM '(' simple_ident ')'    { $$= NEW_PTN Item_default_value(@$, $3); }
        | VALUES '(' simple_ident_nospvar ')' { $$= NEW_PTN Item_insert_value(@$, $3); }
        | INTERVAL_SYM expr interval '+' expr %prec INTERVAL_SYM
          /* we cannot put interval before - */                   // ★ 注释解释了为何没有 "INTERVAL n unit -"
          { $$= NEW_PTN Item_date_add_interval(@$, $5, $2, $3, 0); }
        | simple_ident JSON_SEPARATOR_SYM TEXT_STRING_literal     // ★ col->'$.path'
          { ... $$= NEW_PTN Item_func_json_extract(YYTHD, @$, $1, path); }
        | simple_ident JSON_UNQUOTED_SEPARATOR_SYM TEXT_STRING_literal  // ★ col->>'$.path'
          { ... $$= NEW_PTN Item_func_json_unquote(@$, extr); }
        ;
```

### 5.4 `joined_table` 与那段 41 行结合性注释（`:11864-11941`）

```yacc
/*
  Join operations are normally left-associative, as in
    t1 JOIN t2 ON t1.a = t2.a JOIN t3 ON t3.a = t2.a
  This is equivalent to
    (t1 JOIN t2 ON t1.a = t2.a) JOIN t3 ON t3.a = t2.a
  They can also be right-associative without parentheses, e.g.
    t1 JOIN t2 JOIN t3 ON t2.a = t3.a ON t1.a = t2.a
  Which is equivalent to
    t1 JOIN (t2 JOIN t3 ON t2.a = t3.a) ON t1.a = t2.a

  In MySQL, JOIN and CROSS JOIN mean the same thing, i.e.:
  - A join without a <join specification> is the same as a cross join.
  - A cross join with a <join specification> is the same as an inner join.

  For the join operation above, this means that the parser can't know until it
  has seen the last ON whether `t1 JOIN t2` was a cross join or not. The only
  way to solve the abiguity is to keep shifting the tokens on the stack, and
  not reduce until the last ON is seen. We tell Bison this by adding a fake
  token CONDITIONLESS_JOIN which has lower precedence than all tokens that
  would continue the join. ... This gives a right-deep join tree for join
  *with* conditions, which is what is expected.

  The challenge here is that t1 JOIN t2 *could* have been a cross join, we
  just don't know it until afterwards. ... The solution is to build the tree
  downwards instead of upwards, as is normally done. ...

    There is only one rule to pay attention to: If the right-hand side of a
    cross join is a join tree, find its left-most leaf (which is a table
    name). Then replace this table name with a cross join of the left-hand
    side of the top cross join, and the right hand side with the original
    table.

  Natural joins are also syntactically conditionless, but we need to make sure
  that they are never right associative. We handle them in their own rule
  natural_join, which is left-associative only. ...
*/
joined_table:                                                   // :11942
          table_reference inner_join_type table_reference ON_SYM expr
          { $$= NEW_PTN PT_joined_table_on($1, @2, $2, $3, $5); }
        | table_reference inner_join_type table_reference USING '(' using_list ')'
          { $$= NEW_PTN PT_joined_table_using($1, @2, $2, $3, $6); }
        | table_reference outer_join_type table_reference ON_SYM expr
          { $$= NEW_PTN PT_joined_table_on($1, @2, $2, $3, $5); }
        | table_reference outer_join_type table_reference USING '(' using_list ')'
          { $$= NEW_PTN PT_joined_table_using($1, @2, $2, $3, $6); }
        | table_reference inner_join_type table_reference
          %prec CONDITIONLESS_JOIN                              // ★ 压到最后才归约
          {
            auto this_cross_join= NEW_PTN PT_cross_join($1, @2, $2, NULL);
            if ($3 == NULL) MYSQL_YYABORT; // OOM
            $$= $3->add_cross_join(this_cross_join);             // ★★ "往右下插"，不是往栈顶加
          }
        | table_reference natural_join_type table_factor         // ★ 右侧只能是 table_factor ⇒ 保证左结合
          { $$= NEW_PTN PT_joined_table_using($1, @2, $2, $3); }
        ;
```

> ★★ `add_cross_join()` 是这段注释的落地：正常做法是"在栈顶加一个 cross join 节点"，但那会得到**右深树**，语义错误；正确做法是**沉到右子树的最左叶子下面**（注释里的 ASCII 图）。

### 5.5 表名与列引用

```yacc
// :12042 —— 前面 :12019-12041 有 23 行设计说明（为何把括号拆成 single_table_parens 等三条规则）
table_factor:
          single_table
        | single_table_parens
        | derived_table { $$ = $1; }
        | joined_table_parens         { $$= NEW_PTN PT_table_factor_joined_table($1); }
        | table_reference_list_parens { $$= NEW_PTN PT_table_reference_list_parens($1); }
        | table_function { $$ = $1; }           // ★ JSON_TABLE（8.0 新增）
        ;

// :12068
single_table:
          table_ident opt_use_partition opt_table_alias opt_key_definition
          {
            $$= NEW_PTN PT_table_factor_table_ident($1, $2, $3, $4);
          }
        ;

// :12080
derived_table:
          table_subquery opt_table_alias opt_derived_column_list
          {
            /*
              The alias is actually not optional at all, but being MySQL we
              are friendly and give an informative error message instead of
              just 'syntax error'.
            */
            if ($2.str == nullptr)
              my_message(ER_DERIVED_MUST_HAVE_ALIAS,
                         ER_THD(YYTHD, ER_DERIVED_MUST_HAVE_ALIAS), MYF(0));
            $$= NEW_PTN PT_derived_table(false, $1, $2, &$3);   // false = 非 LATERAL
          }
        | LATERAL_SYM table_subquery opt_table_alias opt_derived_column_list
          {
            if ($3.str == nullptr) my_message(ER_DERIVED_MUST_HAVE_ALIAS, ...);
            $$= NEW_PTN PT_derived_table(true, $2, $3, &$4);    // true = LATERAL
          }
        ;

// :14853 —— ⚠️ 只有 2 个分支！_q_2d / _q_3d 在独立的 simple_ident_q 里
simple_ident:
          ident
          {
            $$= NEW_PTN PTI_simple_ident_ident(@$, to_lex_cstring($1));
          }
        | simple_ident_q
        ;

simple_ident_q:                                                 // :14869
          ident '.' ident
          {
            $$= NEW_PTN PTI_simple_ident_q_2d(@$, $1.str, $3.str);       // tbl.col
          }
        | ident '.' ident '.' ident
          {
            if (check_and_convert_db_name(&$1, false) != Ident_name_check::OK)
              MYSQL_YYABORT;
            $$= NEW_PTN PTI_simple_ident_q_3d(@$, $1.str, $3.str, $5.str);  // db.tbl.col
          }
        ;

// :14839
order_expr:
          expr opt_ordering_direction
          { $$= NEW_PTN PT_order_expr($1, $2); }
        ;

grouping_expr:                                                  // :14846
          expr
          { $$= NEW_PTN PT_order_expr($1, ORDER_NOT_RELEVANT); }  // ★ GROUP BY 复用 PT_order_expr
        ;

// :17322
row_subquery:   subquery ;
table_subquery: subquery ;

subquery:                                                       // :17330
          query_expression_parens %prec SUBQUERY_AS_EXPR
          { $$= NEW_PTN PT_subquery(@$, $1); }
        ;
```

> ★ **`grouping_expr` 与 `order_expr` 产出同一个类 `PT_order_expr`，只差一个 `direction`**——GROUP BY 的方向恒为 `ORDER_NOT_RELEVANT`（三态之一）。这正是 [`15_groupby_distinct_order.md`](07_optimize/15_groupby_distinct_order.md) 里 DISTINCT→GROUP BY 改写要手动设 `ORDER_NOT_RELEVANT` 的原因。

---

## 六、Parse Tree 节点体系

> ★ **完整剖析已独立成篇：[`11_parse_tree.md`](11_parse_tree.md)**——基类（`Parse_tree_root` / `Parse_tree_node_tmpl<Context>`）、两套正交递归（`contextualize()` / `itemize()`）、`PT_*` 的 17 个范畴与代表实现、与 yacc 的对应、`PT_query_specification` 深剖，全在那篇。
>
> 本节只保留**文法视角**下必须知道的三点：一个产生式动作到底造出什么、两类节点怎么区分、以及常用节点的坐标速查。

### 6.1 ★ 从文法角度看：一个产生式动作造出什么

语法阶段的产物只有"语法形状"，它有三类，混在同一棵树里：

| 动作 | 产物 | 后续走哪条递归 |
|---|---|---|
| `NEW_PTN PT_xxx(...)` | **纯语法节点**（继承 `Parse_tree_node`） | `contextualize()`（结构层） |
| `NEW_PTN PTI_xxx(...)` | **解析期占位 Item**（继承 `Parse_tree_item : public Item`） | `itemize()`（表达式层），用完**就地替换** |
| 少数 `NEW_PTN Item_xxx(...)` | **直接造真 Item**（如 `Item_func_plus`、`Item_int`） | 归约时信息已足够，无需替换 |

⇒ 所以一棵 PT 树是**混合树**：纯语法节点 + 占位 Item + 成品 Item 混在一起。完整树形见 [`11_parse_tree.md`](11_parse_tree.md)「先有个整体印象」1.3。

### 6.2 基类速览（逐行剖析见专篇第一章）

> ⚠️ **勘误**：`Parse_tree_root` **不在** `parse_tree_node_base.h`，它在 **`sql/parse_tree_nodes.h:154`**；`Parse_context` 在 **`sql/parse_tree_node_base.h:126`**（不是 `parse_tree_nodes.h`）。

```cpp
// sql/parse_tree_nodes.h:154
/**
  Base class for all top-level nodes of SQL statements
  @ingroup ptn_stmt
*/
class Parse_tree_root {
  Parse_tree_root(const Parse_tree_root &) = delete;
  void operator=(const Parse_tree_root &) = delete;

 protected:
  virtual ~Parse_tree_root() = default;
  Parse_tree_root() = default;

 public:
  virtual Sql_cmd *make_cmd(THD *thd) = 0;      // ← 唯一的纯虚接口
};
```

```cpp
// sql/parse_tree_node_base.h:126
/**
  Environment data for the contextualization phase
*/
struct Parse_context {
  THD *const thd;                      ///< Current thread handler
  MEM_ROOT *mem_root;                  ///< Current MEM_ROOT
  Query_block *select;                 ///< Current Query_block object
  mem_root_deque<QueryLevel> m_stack;  ///< Aids query term tree construction
  /// Call upon parse completion.
  /// @returns true on error, else false
  bool finalize_query_expression();
  Parse_context(THD *thd, Query_block *sl);
  bool is_top_level_union_all(Surrounding_context op);
};
```

```cpp
// sql/parse_tree_node_base.h:96 —— QueryLevel：contextualize 期的临时容器
class Query_term;
enum Surrounding_context {
  SC_TOP, SC_QUERY_SPECIFICATION, SC_TABLE_VALUE_CONSTRUCTOR,
  SC_QUERY_EXPRESSION, SC_SUBQUERY,
  SC_UNION_DISTINCT, SC_UNION_ALL,
  SC_INTERSECT_DISTINCT, SC_INTERSECT_ALL,
  SC_EXCEPT_DISTINCT, SC_EXCEPT_ALL
};

struct QueryLevel {
  Surrounding_context m_type;
  /** elts 在 contextualize 过程中充当临时容器：
      收集子节点 → 延迟组合 → 支持多路操作（A UNION B UNION C）→ 构建 Query_term 树 */
  mem_root_deque<Query_term *> m_elts;
  bool m_has_order{false};
  QueryLevel(MEM_ROOT *mem_root, Surrounding_context sc, bool has_order = false)
      : m_type(sc), m_elts(mem_root), m_has_order(has_order) {}
};
```

```cpp
// sql/parse_tree_node_base.h:145
/**
  Base class for parse tree nodes (excluding the Parse_tree_root hierarchy)
*/
template <typename Context>
class Parse_tree_node_tmpl {
  friend class Item;  // for direct access to the "contextualized" field

  Parse_tree_node_tmpl(const Parse_tree_node_tmpl &);  // undefined
  void operator=(const Parse_tree_node_tmpl &);        // undefined

#ifndef NDEBUG
 private:
  bool contextualized;  // true if the node object is contextualized
#endif                  // NDEBUG

 public:
  typedef Context context_t;      // ★★ CONTEXTUALIZE 宏用的就是这个 typedef

  static void *operator new(size_t size, MEM_ROOT *mem_root,
                            const std::nothrow_t &arg
                            [[maybe_unused]] = std::nothrow) noexcept {
    return mem_root->Alloc(size);                      // ★ placement new 到 MEM_ROOT
  }
  static void operator delete(void *ptr [[maybe_unused]],
                              size_t size [[maybe_unused]]) {
    TRASH(ptr, size);                                  // ★ 只做填充标记，不释放
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

  /**
    Do all context-sensitive things and mark the node as contextualized
    @retval false success
    @retval true  syntax/OOM/etc error
  */
  virtual bool contextualize(Context *pc) {
    uchar dummy;
    if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)) return true;

#ifndef NDEBUG
    assert(!contextualized);
    contextualized = true;      // ★ 打标记，防止重复 contextualize
#endif
    return false;
  }

  /// syntax_error() 的延迟版（三条重载）
  void error(Context *pc, const POS &pos) const {
    pc->thd->syntax_error_at(pos);
  }
  void error(Context *pc, const POS &pos, const char *msg) const {
    pc->thd->syntax_error_at(pos, "%s", msg);
  }
  void errorf(Context *pc, const POS &pos, const char *format, ...) const;
};

typedef Parse_tree_node_tmpl<Parse_context> Parse_tree_node;   // :256
```

> ★ `check_stack_overrun()` 出现在 `contextualize()` 的第一行——因为 Parse Tree 可能很深（嵌套子查询），递归 contextualize 有爆栈风险，每个节点都先探一次栈。
> ★ `contextualized` 标记**只在 debug 构建存在**（`#ifndef NDEBUG`），release 下没有这层保护。


### 6.3 常用节点坐标速查（`PT_*`）

全部在 `sql/parse_tree_nodes.h`（实现在 `parse_tree_nodes.cc`）：

| 类 | 类定义行 | `contextualize` 实现 | 做了什么 |
|---|---------|---------------------|---------|
| `PT_select_stmt` | 1690 | `parse_tree_nodes.cc:705`（是 `make_cmd` 不是 contextualize） | 顶层入口，最后返回 `Sql_cmd_select` |
| `PT_query_expression` | 1463 | `parse_tree_nodes.cc:3988` | 压 QueryLevel、处理 ORDER/LIMIT、组装 Query_term |
| `PT_query_specification` | 1344 | `parse_tree_nodes.cc:1170` | **核心**：填 fields / table_list / where / having / group / order |
| `PT_select_item_list` | — | `parse_tree_nodes.cc:3613` | `pc->select->fields = value` |
| `PT_table_factor_table_ident` | 424 | `parse_tree_nodes.cc:3639` | `add_table_to_list()` 建 `Table_ref`（`:3645`） |
| `PT_derived_table` | 484 | `parse_tree_nodes.cc:1331` | 递归 contextualize 子查询、建 `Table_ident`、处理 LATERAL |
| `PT_joined_table_on` | 583 | `parse_tree_nodes.cc:3679` | 建 name resolution context、`add_join_on` |
| `PT_joined_table_using` | 596 | `parse_tree_nodes.cc:3706` | `add_join_natural` |
| `PT_cross_join` | 571 | `parse_tree_nodes.cc:3673` | `nest_last_join` |
| `PT_order` | 202 | `parse_tree_nodes.cc:288` | 切 `parsing_place=CTX_ORDER_BY`、填 `order_list` |
| `PT_group` | — | `parse_tree_nodes.cc:252` | 填 `group_list`，方向改 `ORDER_NOT_RELEVANT` |
| `PT_limit_clause` | — | `parse_tree_nodes.cc:3619` | `select_limit` / `offset_limit` |
| `PT_subquery` | — | `parse_tree_nodes.cc:4109` | `lex->new_query()` 建子 QE+QB |
| `PT_with_clause` | — | `parse_tree_nodes.cc:1945` | 挂 `m_with_clause` |

### 6.4 表达式节点坐标速查（`PTI_*`）

在 `sql/parse_tree_items.h`，继承 `Parse_tree_item`（`parse_tree_helpers.h`）——**双重身份**，见下节：

| 类 | .h 行 | itemize 实现 | 转化结果 |
|---|-------|-------------|---------|
| `PTI_comp_op` | 59 | `parse_tree_items.cc:166` | `Item_func_gt/eq/lt/...`（`:171`） |
| `PTI_simple_ident_ident` | 99 | `:353` | `Item_field`（`:379`）或 `Item_ref`（`:381`） |
| `PTI_simple_ident_q_2d` / `_3d` | 134 / 115 | `:409 / 388` | table.col / db.table.col |
| `PTI_singlerow_subselect` | 434 | `:313` | 标量子查询 |
| `PTI_exists_subselect` | 446 | `:320` | EXISTS |
| `PTI_expr_with_alias` | 489 | `:335` | 设置 `item_name` |
| `PTI_where` / `PTI_having` | 558 / 564 | `:684`（`PTI_context`） | 切 parsing_place + `make_condition` |
| `PTI_truth_transform` | 46 | `:474` | IS TRUE / IS FALSE / NOT |
| `PTI_text_literal` | 239 | — | **直接继承 `Item_string`**（非占位符） |
| `PTI_count_sym` | 412 | `:582` | **直接继承 `Item_sum_count`** |

> 这些节点的 `contextualize()` / `itemize()` 完整实现，见 [`11_parse_tree.md`](11_parse_tree.md) 第三章。

## 七、PTI_* 的双重身份

这是 MySQL 与很多数据库不同的一点（也是"没有独立 AST 层"的根因）。

继承链：

```
Parse_tree_item  :  public Item
                            ↑
Item  :  public Parse_tree_node      (item.h:882)
```

所以 `PTI_*` 既是语法节点、又是表达式节点。**`Parse_tree_item` 完整定义**（`sql/parse_tree_helpers.h:58`）：

```cpp
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

  enum Type type() const override { return INVALID_ITEM; }
  double val_real() override {
    assert(0);
    return 0;
  }
  longlong val_int() override {
    assert(0);
    return 0;
  }
  String *val_str(String *) override {
    assert(0);
    return nullptr;
  }
  my_decimal *val_decimal(my_decimal *) override {
    assert(0);
    return nullptr;
  }
  bool get_date(MYSQL_TIME *, uint) override {
    assert(0);
    return false;
  }
  bool get_time(MYSQL_TIME *) override {
    assert(0);
    return false;
  }
};
```

> ★★ **所有取值方法都是 `assert(0)`**——这就是"占位符"最直白的证据：一个 `PTI_*` 对象**永远不该被求值**。如果哪天你在 `val_int()` 里撞上这个断言，说明某个 `PTI` 没被 `itemize()` 替换掉就漏进了 Item 树（典型 bug）。
>
> ★ 对比：`PTI_text_literal`、`PTI_count_sym` **直接继承** `Item_string` / `Item_sum_count`，它们不经过 `Parse_tree_item`——因为字符串字面量和 `COUNT` 的语义在语法阶段就确定了，不需要占位。

### 设计思想：为什么要有 PTI（语法与语义解耦）

`PTI_*` 的存在，本质是解决一个硬约束：**语法分析器遇到表达式时，还不知道它是什么语义**。

解析 `WHERE b > 1` 时，语法分析器只知道"有个比较运算 `>`，左边 `b`，右边 `1`"。但它**不知道**：

- `b` 是列、还是函数、还是聚合？—— 要查表定义（语义阶段才知道）
- `>` 是普通比较、还是子查询的 `> ANY`？—— 也依赖上下文

所以语法阶段**不能**直接 `new Item_func_gt(...)`（那是语义对象，需要知道操作数类型），只能先 new 一个占位符 `PTI_comp_op`，只记录"左操作数 + 比较符 + 右操作数"这三个**语法事实**。

这个"占位符"设计带来三个好处：

1. **语法与语义解耦**：Parse Tree 是纯语法的，不碰 `Field`/`TABLE`。同一棵 Parse Tree 能被 contextualize 成不同的语义对象——`PTI_simple_ident_ident` 可能变成 `Item_field`（列）也可能变成 `Item_ref`（别名），取决于上下文。
2. **支撑 PREPARE 一次、EXECUTE 多次**：纯语法的 Parse Tree 可以缓存，每次 EXECUTE 重新 contextualize（不同的绑定变量 → 不同的语义对象）。
3. **可改写**：query rewrite 插件（`sql_query_rewrite.cc`）在 Parse Tree 层改写，比在 Item 层安全——语法树还没绑定语义，改写不会破坏类型/引用。

> 所以 `PTI_*` 是"**语法与语义解耦**"设计思想的产物，C++ 的多重继承（`Parse_tree_item : Item : Parse_tree_node`）+ 占位符模式只是实现手段，不是设计本身。对比 `PTI_text_literal`、`PTI_count_sym`：它们直接继承 `Item_string`/`Item_sum_count`，因为字符串字面量和 COUNT 的语义在语法阶段就确定了，不需要占位符。

### 例子：`PTI_comp_op` 如何"变身"

语法动作（`sql_yacc.yy:10265`）只创建一个占位节点，记录"左操作数 / 比较类型 / 右操作数"：

```cpp
// sql/parse_tree_items.cc:166
bool PTI_comp_op::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      right->itemize(pc, &right))
    return true;

  *res = (*boolfunc2creator)(false)->create(left, right);   // :171
  return *res == nullptr;
}
```

**关键在 `:171`**：`PTI_comp_op` 本身被**丢弃**，`*res` 换成新建的 `Item_func_gt` / `Item_func_eq` / …

这就是"占位符 → 真实 Item"的典型模式：**PTI 节点是"待定类型的壳"，itemize 时决定壳里装什么，然后把壳换掉**。

### 例子：`PTI_where` 如何切上下文

```cpp
// sql/parse_tree_items.cc:684
bool PTI_context::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  pc->select->parsing_place = m_parsing_place;      // :687  CTX_WHERE / CTX_HAVING
  if (expr->itemize(pc, &expr)) return true;
  if (!expr->is_bool_func()) {
    expr = make_condition(pc, expr);                // :692  非布尔 → 包一层 truth test
    if (expr == nullptr) return true;
  }
  pc->select->parsing_place = CTX_NONE;             // :698
  expr->apply_is_true();                            // :700
  *res = expr;
  return false;
}
```

`parsing_place` 很重要：它决定 `Item::itemize` 里的统计（`select_n_having_items++`）和后续名字解析的合法性判断。

---

## 八、Parse Tree 的内存管理

```cpp
// sql/sql_yacc.yy:40
#define YYMEM_ROOT (YYTHD->mem_root)
// sql/sql_yacc.yy:222
#define NEW_PTN new(YYMEM_ROOT)
```

对应 placement new（`parse_tree_node_base.h:163`）：

```cpp
static void *operator new(size_t size, MEM_ROOT *mem_root,
                          const std::nothrow_t & = std::nothrow) noexcept {
  return mem_root->Alloc(size);        // :166
}
static void operator delete(void *ptr, size_t size) {
  TRASH(ptr, size);                    // :170  只做内存填充标记，不释放
}
```

**三条结论**：

1. **所有 PT 节点都在 `thd->mem_root`**，从不逐个 `delete`
2. contextualize 完成后 **PT 节点不再被任何指针引用**（`THD::sql_parser` 里的 `root` 是局部变量，`sql_class.cc:3061`）
3. 语句结束 `mem_root->ClearForRecy()`（`sql_parse.cc:2522`）整块回收 —— 详见 [01](01_protocol_to_dispatch.md#八mem_root为什么-parse-tree-从不-delete)

### 9.1 reentrant：解析器如何拿到 mem_root

Bison 生成的解析器是 **reentrant（可重入）** 的——不依赖全局变量，状态全通过参数传递。`sql_yacc.yy:519-523`：

```yacc
%parse-param { class THD *YYTHD }                 // yyparse 第一个参数是 THD*
%parse-param { class Parse_tree_root **parse_tree } // 第二个参数是解析树出参
%lex-param   { class THD *YYTHD }                 // yylex 也拿到 THD*
%define api.pure                                   // 可重入（线程安全）
```

`api.pure` 是 key：传统 Bison 用全局变量 `yylval`/`yychar` 传状态，多线程会踩踏；`api.pure` 改成参数传递，每个线程传自己的 THD。宏链把 THD 连到 mem_root（`sql_yacc.yy:40/222`）：

```
YYMEM_ROOT = YYTHD->mem_root
NEW_PTN    = new(YYMEM_ROOT) = new(YYTHD->mem_root)   ← placement new 到 mem_root
```

### 9.2 语义值栈：`YYSTYPE` 数组，在函数栈上

Bison 的解析器维护三个栈（生成的 `sql_yacc.cc:25057-25069`）：

```c
yy_state_t yyssa[YYINITDEPTH];   // ① 状态栈（语法状态机当前状态）
YYSTYPE    yyvsa[YYINITDEPTH];   // ② 语义值栈（每个符号的值）★
YYLTYPE    yylsa[YYINITDEPTH];   // ③ 位置栈（符号在源码的位置）
```

**语义值栈是 `YYSTYPE` 数组，初始在函数栈上**（`yyvsa[100]` 局部数组）；不够时 `YYSTACK_ALLOC`（= `alloca`，`sql_yacc.cc:2642`）扩容，**依然在栈上**，上限 `YYMAXDEPTH=3200`。每个语法符号占一个槽位，槽位内容是 `YYSTYPE` union（`parser_yystype.h:340`）。

### 9.3 指针型 vs 值型：union 成员的两种形态

关键前提：**语义值栈槽位在栈上**。槽位里的 union 成员分两种形态，对应两种内存归属：

| | 指针型 | 值型 |
|---|---|---|
| union 成员 | `Item *item`、`Table_ident *table` | `Mem_root_array_YY<...> table_reference_list` |
| 槽位存什么 | 8 字节指针 | **整个对象外壳**（4 成员 ~32 字节） |
| 对象住哪 | mem_root（堆） | **栈上的 union 槽位** |
| 数据住哪 | 对象内部 → mem_root | 外壳的 `m_array` → mem_root |

```
指针型：槽位(栈) ──8B 指针──→ 对象(mem_root 堆)

值型：  槽位(栈) 存整个 Mem_root_array_YY 外壳
            ├ m_root   ──→ 指向 THD::mem_root
            └ m_array  ──→ 元素数据(mem_root 堆)
```

**"值型"不是"数据在栈上"，而是"控制块（外壳）在栈上、数据在堆上"**。`Mem_root_array_YY` 的外壳（`m_root/m_array/m_size/m_capacity` 四个成员）躺在栈槽位里；它管理的元素数据（`m_array` 指向的连续内存）在 mem_root 堆上。

### 9.4 union 要求平凡类型 → 为什么 `_YY` 无 ctor/dtor

C++ union 的成员**必须是平凡类型（trivial）**——不能有非平凡构造/析构，因为 union 不知道当前激活哪个成员、无法自动调用谁的 ctor/dtor。

所以：
- **能放指针就放指针**（`Item *`、`Table_ident *`）——指针平凡
- **必须放值类型时**，就得让容器本身变平凡：`Mem_root_array_YY` 故意不写 ctor/dtor（`mem_root_array.h:415` 注释 `No CTOR/DTOR for this class!`），用 `init()` 手动初始化代替

```yacc
// sql_yacc.yy:7885（index_options 规则，值型用法）
%empty { $$.init(YYMEM_ROOT); }
  | fulltext_index_options
    {
      $$.init(YYMEM_ROOT);          // 在栈槽位里就地初始化外壳
      if ($$.push_back($1)) ...     // push_back 时数据才从 mem_root 分配
    }
```

对比有 ctor/dtor 的 `mem_root_deque`，只能作指针型：

```yacc
// sql_yacc.yy:2679
$$ = NEW_PTN mem_root_deque<Item *>(YYMEM_ROOT);   // 对象在 mem_root，槽位存指针
```

### 9.5 `init` 做了什么：只初始化外壳，不分配数据

```cpp
// mem_root_array.h:74-81
void init(MEM_ROOT *root) {
  m_root = root;       // 记住 mem_root 指针
  m_array = nullptr;   // 数据指针置空（还没分配）
  m_size = 0;          // 0 个元素
  m_capacity = 0;      // 0 容量
}
```

`init` 只初始化外壳 4 个成员，**一行内存都不分配**。真正的数据内存，到第一次 `push_back → reserve → m_root->Alloc(n * sizeof(T))`（`mem_root_array.h:155-158`）才从 mem_root 分配。

### 9.6 完整生命周期闭环

```
① yyparse 入口：状态栈/语义值栈/位置栈 都在函数栈上（yyvsa 等局部数组）
② lexer 填 token 值：yylex(yylval, yythd) 写 *yylval（数字→num、关键字→lexer.keyword）
③ 归约执行动作：
     指针型 → $$ = NEW_PTN X(args)，对象在 mem_root，槽位存指针
     值型   → $$.init(YYMEM_ROOT)，外壳在槽位就地 init，数据 push_back 时才分配
④ 根规则归约：$$ 是 Parse_tree_root*，经 parse_tree 出参传回
⑤ 语句结束：THD::mem_root 整体释放 → 指针型对象 + 值型数据 + Item 全回收
            栈上的值型外壳随函数返回自然消亡
```

**闭环关键**：指针型对象生命周期 = mem_root（语句级）；值型外壳生命周期 = 栈帧（归约期），但外壳里 `m_array` 的数据生命周期 = mem_root（语句级）。两者殊途同归——所有"数据"最终在 mem_root，语句结束统一回收；区别只在"控制块（外壳）"住栈上还是堆上。

---

## 示例：一条 SQL 产生哪些 PT 节点

`SELECT a FROM t WHERE b > 1 ORDER BY c`

★ 行号已按 8.0.39 校正：

| 语法规则 | 行号 | 产出 |
|---------|------|------|
| `simple_ident: ident` | `:14854-14857` | `PTI_simple_ident_ident(a)`（`:14856`） |
| `select_item` | `:10188-10194` | `PTI_expr_with_alias(PTI(a), "")`（`:10192`） |
| `select_item_list` | `:10166-10186` | `PT_select_item_list` |
| `single_table` | `:12068-12073` | `PT_table_factor_table_ident(t)`（`:12071`） |
| `where_clause: WHERE expr` | `:12395-12397` | `PTI_where(PTI_comp_op(PTI(b), >, Item_int(1)))`（`PTI_comp_op` 产生式 `:10271`） |
| `order_expr` | `:14839-14844` | `PT_order_expr(PTI(c), ORDER_ASC)`（`:14842`） |
| `order_list` | `:12593-12606` | `PT_order_list`（`:12601`） |
| `order_clause` | `:12586-12591` | `PT_order`（`:12589`） |
| `query_specification` | `:9994-10038` | `PT_query_specification`（`:10007`，无 INTO 分支 `:10029`） |
| `query_expression` | `:9926-9940` | `PT_query_expression`（`:9931`） |
| `select_stmt` | `:9830-9841` | `PT_select_stmt`（`:9833`） |
| `sql_statement` | `:2284-2327` | 分号/EOF 处理 |
| `start_entry` | `:2238` | 分支① → `sql_statement` |
| `simple_statement_or_begin` | **`:2335`** | **`*parse_tree = PT_select_stmt*`** |

完整链路：`start_entry(2238)` → `sql_statement(2284)` → `simple_statement_or_begin(2334)` → **`*parse_tree = $1`(2335)** → `simple_statement` → `select_stmt(9830)` → `PT_select_stmt`。

产出后，这棵树会被交给 `LEX::make_sql_cmd` → `PT_select_stmt::make_cmd` → **contextualize**（见 [04](05_contextualize.md)）。

---


## 参考

**书籍 / 论文**
- **Aho & Ullman《Principles of Compiler Design》（龙书）** —— 词法分析、语法分析、Parse Tree 的经典分阶段思想

**官方文档**
- *MySQL 8.0 Reference Manual → Lexical Structure*
- *MySQL 8.0 Reference Manual → Keywords and Reserved Words*

