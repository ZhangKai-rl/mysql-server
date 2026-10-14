# 02 Lexer：从字符流到 token 流

> 本篇覆盖：词法分析如何把一段 SQL 文本切成 token 流（第 ① 层 IR 的"原料"），是 [`03 Parser`](04_parser.md) 的输入。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、MYSQLlex 入口](#一mysqllex-入口)
- [二、lex_one_token 状态机（逐状态剖析）](#二lex_one_token-状态机逐状态剖析)
  - [33 个状态的全貌（按枚举顺序）](#33-个状态的全貌按枚举顺序)
- [三、Lex_input_stream：输入缓冲与回退](#三lex_input_stream输入缓冲与回退)
- [四、find_keyword：关键字识别](#四find_keyword关键字识别)
- [五、YYSTYPE：语义值如何回传](#五yystype语义值如何回传)
- [六、字符集驱动的状态表](#六字符集驱动的状态表)
- [七、LALR(2) 特判：词法如何替语法兜底](#七lalr2-特判词法如何替语法兜底)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)

---

## 设计思想与理论基础

### 词法分析为什么独立成层

经典编译原理（龙书）三段式的**第一段**：把无结构的字符流，切成有意义的 token（标识符/关键字/数字/字符串/运算符/注释）。它独立成层的依据是：

- **词法可用正则文法描述**（标识符 `[a-zA-Z_][a-zA-Z0-9_]*`、数字 `[0-9]+` 都是正则），而 SQL 的括号嵌套、递归结构是正则表达不了的——必须交给下一层 CFG（上下文无关文法）
- **隔离复杂度**：字符集（utf8mb4/gbk）、大小写、多字节字符识别这些"字节层"的脏活，如果揉进语法分析，状态机会爆炸

### 状态机 vs 手写 if-else

MySQL 的词法**不是**简单的正则扫描，而是一台**显式状态机**（`my_lex_states` 枚举 + 每个状态一个 `case`）。原因：SQL 的词法有大量**上下文相关**的判断（`-` 可能是减号也可能是注释 `--`、`->` 和 `->>` 是 JSON 运算符、`N'...'` 是 national string、`_utf8'...'` 是字符集引导符），单纯的正则分不开，需要状态记忆。

### 演进

- **5.x**：`lex_one_token` 已经是状态机，但状态和宏更混乱
- **8.0**：状态机基本稳定，主要新增了 JSON 运算符（`->`/`->>`）、`MY_LEX_IDENT_OR_DOLLAR_QUOTE`（`$$...$$`）等新语法对应的状态

---

## 先有个整体印象

词法分析做的事就一件：**按状态机把字符流切成 token，并给每个 token 标上类型和值**。

```
SELECT a FROM t WHERE b > 1
   │  MYSQLlex() 循环调用 lex_one_token()
   ▼
[SELECT] [a] [FROM] [t] [WHERE] [b] [>] [1]
   │    │    │     │    │      │    │   │
   │    │    │     │    │      │    │   └ NUM（值 1）
   │    │    │     │    │      │    └ 运算符 '>'
   │    │    │     │    │      └ IDENT（"b"）
   │    │    │     │    └ WHERE_SYM（关键字）
   │    │    │     └ IDENT（"t"）
   │    │    └ FROM_SYM（关键字）
   │    └ IDENT（"a"）
   └ SELECT_SYM（关键字）
```

**关键**：词法阶段**不判断** `a` 是列名还是别名、`t` 是表名还是 CTE——它只回答"这是一个标识符/关键字/数字"，语义留到后面（contextualize / fix_fields）。

---

## 一、MYSQLlex 入口

Bison 生成的 parser 每需要一个 token，就调用一次 lexer。MySQL 的 lexer 入口是 `MYSQLlex`（`sql/sql_lex.cc:1304`）：

**完整源码**（`sql/sql_lex.cc:1304-1372`）——★ 它绝不是薄封装，承担 4 件事：错误短路 → lookahead 回放 → `WITH ROLLUP` 合并 → digest 收集：

```cpp
/**
  yylex() function implementation for the main parser

  @param [out] yacc_yylval   semantic value of the token being parsed (yylval)
  @param [out] yylloc        "location" of the token being parsed (yylloc)
  @param thd                 THD

  @return                    token number

  @note
  MYSQLlex remember the following states from the following MYSQLlex():

  - MY_LEX_END			Found end of query
*/
int MYSQLlex(YYSTYPE *yacc_yylval, YYLTYPE *yylloc, THD *thd) {
  auto *yylval = reinterpret_cast<Lexer_yystype *>(yacc_yylval);
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;
  int token;

  if (thd->is_error()) {
    if (thd->get_parser_da()->has_sql_condition(ER_CAPACITY_EXCEEDED))
      return ABORT_SYM;                 // ★ 错误短路：容量类错误立刻终止词法
  }

  if (lip->lookahead_token >= 0) {      // ★① lookahead 回放路径（WITH 特判留下的）
    /*
      The next token was already parsed in advance,
      return it.
    */
    token = lip->lookahead_token;
    lip->lookahead_token = -1;
    *yylval = *(lip->lookahead_yylval);
    yylloc->cpp.start = lip->get_cpp_tok_start();
    yylloc->cpp.end = lip->get_cpp_ptr();
    yylloc->raw.start = lip->get_tok_start();
    yylloc->raw.end = lip->get_ptr();
    lip->lookahead_yylval = nullptr;
    lip->add_digest_token(token, yylval);
    return token;
  }

  token = lex_one_token(yylval, thd);   // ★② 真正的单 token 扫描
  yylloc->cpp.start = lip->get_cpp_tok_start();
  yylloc->raw.start = lip->get_tok_start();

  switch (token) {
    case WITH:                          // ★③ LALR(2) → LALR(1) 的唯一特判
      /*
        Parsing 'WITH' 'ROLLUP' requires 2 look ups,
        which makes the grammar LALR(2).
        Replace by a single 'WITH_ROLLUP' token,
        to transform the grammar into a LALR(1) grammar,
        which sql_yacc.yy can process.
      */
      token = lex_one_token(yylval, thd);     // ★★ 唯一"多读一个 token"的地方
      switch (token) {
        case ROLLUP_SYM:
          yylloc->cpp.end = lip->get_cpp_ptr();
          yylloc->raw.end = lip->get_ptr();
          lip->add_digest_token(WITH_ROLLUP_SYM, yylval);
          return WITH_ROLLUP_SYM;             // 合并成一个 token
        default:
          /*
            Save the token following 'WITH'
          */
          lip->lookahead_yylval = lip->yylval;    // ★ 不是 ROLLUP → 存起来下次回放
          lip->yylval = nullptr;
          lip->lookahead_token = token;
          yylloc->cpp.end = lip->get_cpp_ptr();
          yylloc->raw.end = lip->get_ptr();
          lip->add_digest_token(WITH, yylval);
          return WITH;
      }
      break;
  }

  yylloc->cpp.end = lip->get_cpp_ptr();       // ★④ 公共收尾：位置 + digest
  yylloc->raw.end = lip->get_ptr();
  if (!lip->skip_digest) lip->add_digest_token(token, yylval);
  lip->skip_digest = false;
  return token;
}
```

> ★★ **职责边界**：`yylloc`（`YYLTYPE`）与 **digest 收集**都由 `MYSQLlex()` 统一处理，`lex_one_token()` **完全不管**——后者只负责"扫出一个 token 号 + 填 `yylval`"。这条分工解释了为什么 `lex_one_token` 里看不到任何位置/摘要代码。

**调用关系**：`MYSQLparse`（Bison 生成的）→ 需要 token 时回调 `MYSQLlex` → `lex_one_token` 扫一个 token → 返回 token 号（如 `SELECT_SYM`）。

**为什么叫 `MYSQLlex` 而不是 `yylex`**：`sql/CMakeLists.txt:1336-1342` 用 `--name-prefix=MYSQL` 让 Bison 把所有 `yy*` 符号加前缀：

```cmake
BISON_TARGET(mysql_parser
    ${CMAKE_CURRENT_SOURCE_DIR}/sql_yacc.yy
    ${CMAKE_CURRENT_BINARY_DIR}/sql_yacc.cc
    COMPILE_FLAGS "--name-prefix=MYSQL --yacc ...")
```

⇒ `yyparse → MYSQLparse`、`yylex → MYSQLlex`、`yylval → MYSQLlval`。

---

## 二、lex_one_token 状态机（逐状态剖析）

`lex_one_token`（`sql/sql_lex.cc:1374`）是整个词法的核心，一台显式状态机：

```cpp
static int lex_one_token(Lexer_yystype *yylval, THD *thd) {
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;
  const CHARSET_INFO *cs = thd->charset();
  const my_lex_states *state_map = cs->state_maps->main_map;   // 字符 → 状态
  const uchar *ident_map = cs->ident_map;                      // 是否标识符字符

  lip->start_token();
  state = lip->next_state;        // 上一个 token 留下的"续接状态"
  lip->next_state = MY_LEX_START;

  for (;;) {
    switch (state) {
      case MY_LEX_START: ...       // ① 跳过空白，进入真实 token
      case MY_LEX_CHAR: ...        // ② 单字符 token / 运算符 / 注释
      case MY_LEX_IDENT: ...       // ③ 标识符 / 关键字
      case MY_LEX_IDENT_SEP: ...   // ④ 标识符后跟 '.'（a.b）
      case MY_LEX_IDENT_START: ... // ⑤ '.' 后跟标识符（.b）
      ...
      case MY_LEX_NUMBER_IDENT: ...
      case MY_LEX_INT_OR_REAL: ... // 数字
      case MY_LEX_REAL_OR_POINT: ...
      case MY_LEX_STRING: ...      // 字符串
      case MY_LEX_COMMENT: ...     // 注释
      case MY_LEX_LONG_COMMENT: ...
    }
  }
}
```

#### 33 个状态的全貌（按枚举顺序）

**枚举完整定义**（`include/sql_chars.h:38-72`）——⚠️ 注意它**不在** `sql/sql_lex.h`，也不叫 `enum_lex_state`（该名字全仓 0 命中）：

```cpp
enum MY_ATTRIBUTE((__packed__)) my_lex_states {
  MY_LEX_START,
  MY_LEX_CHAR,
  MY_LEX_IDENT,
  MY_LEX_IDENT_SEP,
  MY_LEX_IDENT_START,
  MY_LEX_REAL,
  MY_LEX_HEX_NUMBER,
  MY_LEX_BIN_NUMBER,
  MY_LEX_CMP_OP,
  MY_LEX_LONG_CMP_OP,
  MY_LEX_STRING,
  MY_LEX_COMMENT,
  MY_LEX_END,
  MY_LEX_NUMBER_IDENT,
  MY_LEX_INT_OR_REAL,
  MY_LEX_REAL_OR_POINT,
  MY_LEX_BOOL,
  MY_LEX_EOL,
  MY_LEX_LONG_COMMENT,
  MY_LEX_END_LONG_COMMENT,
  MY_LEX_SEMICOLON,
  MY_LEX_SET_VAR,
  MY_LEX_USER_END,
  MY_LEX_HOSTNAME,
  MY_LEX_SKIP,
  MY_LEX_USER_VARIABLE_DELIMITER,
  MY_LEX_SYSTEM_VAR,
  MY_LEX_IDENT_OR_KEYWORD,
  MY_LEX_IDENT_OR_HEX,
  MY_LEX_IDENT_OR_BIN,
  MY_LEX_IDENT_OR_NCHAR,
  MY_LEX_IDENT_OR_DOLLAR_QUOTE,
  MY_LEX_STRING_OR_DELIMITER
};
```

> ★ **源码里没有任何逐项注释**——就是 33 个裸枚举名。这就是为什么"这个状态到底干嘛的"只能从 `lex_one_token` 的 `case` 里反推。
>
> ★ `MY_ATTRIBUTE((__packed__))` 让它只占 1 字节——因为 `state_map` 是 `my_lex_states[256]` 的数组，每字符集一份。

配套结构（`include/sql_chars.h:91`）：

```cpp
struct lex_state_maps_st {
  enum my_lex_states main_map[256];
  enum hint_lex_char_classes hint_map[256];   // ★ hint 子词法器另一张表
};

typedef struct lex_state_maps_st lex_state_maps_st;

bool init_state_maps(CHARSET_INFO *cs);
```

下面逐条剖析了 5 个主干状态、概括了数字/字符串/注释三组。这里先给一张**全枚举对照表**，方便反查"某个 token 为什么被这么切"：

| # | 状态 | 归类 | 说明 |
|---|---|---|---|
| 1 | `MY_LEX_START` | 入口 | 跳过空白，进入真实 token（**本文详解**） |
| 2 | `MY_LEX_CHAR` | 入口 | 单字符 token / 运算符 / 注释起点（**本文详解**） |
| 3 | `MY_LEX_IDENT` | 标识符 | 标识符或关键字（**本文详解**） |
| 4 | `MY_LEX_IDENT_SEP` | 标识符 | 标识符后跟 `.`（`a.b`）（**本文详解**） |
| 5 | `MY_LEX_IDENT_START` | 标识符 | `.` 后跟标识符（`.b`）（**本文详解**） |
| 6 | `MY_LEX_REAL` | 数字 | 已确定是实数 |
| 7 | `MY_LEX_HEX_NUMBER` | 数字/串 | 十六进制字面量引导（`x'..'` / `0x`） |
| 8 | `MY_LEX_BIN_NUMBER` | 数字/串 | 二进制字面量引导（`b'..'` / `0b`） |
| 9 | `MY_LEX_CMP_OP` | 运算符 | 比较运算符 |
| 10 | `MY_LEX_LONG_CMP_OP` | 运算符 | 多字符比较符（`<=` `>=` `<>` `!=`） |
| 11 | `MY_LEX_STRING` | 字符串 | 字符串字面量（**本文概括**） |
| 12 | `MY_LEX_COMMENT` | 注释 | `#` / `--` 行注释（**本文概括**） |
| 13 | `MY_LEX_END` | 入口 | 结束 |
| 14 | `MY_LEX_NUMBER_IDENT` | 数字 | 数字后紧跟标识符 |
| 15 | `MY_LEX_INT_OR_REAL` | 数字 | 整数/实数待定（**本文概括**） |
| 16 | `MY_LEX_REAL_OR_POINT` | 数字 | 小数点后待定：是 `.5` 还是 `.field` |
| 17 | `MY_LEX_BOOL` | 字面量 | 布尔/特殊字面量引导 |
| 18 | `MY_LEX_EOL` | 入口 | 行尾 |
| 19 | `MY_LEX_LONG_COMMENT` | 注释 | `/*` 长注释（**本文概括**） |
| 20 | `MY_LEX_END_LONG_COMMENT` | 注释 | 长注释内，寻找 `*/` |
| 21 | `MY_LEX_SEMICOLON` | 语句边界 | 分号——多语句切分就靠它（详见 [`01_protocol_to_dispatch.md`](01_protocol_to_dispatch.md) 多语句部分） |
| 22 | `MY_LEX_SET_VAR` | 变量 | `SET` 的赋值符（`:=` / `=`） |
| 23 | `MY_LEX_USER_END` | 变量/账号名 | **`@` 本身**——下一个字符决定转入 `SYSTEM_VAR` / `HOSTNAME` / `START` |
| 24 | `MY_LEX_HOSTNAME` | 账号名 | 账号名里 `@host` 部分（见 `server/auth/`） |
| 25 | `MY_LEX_SKIP` | 入口 | 跳过（**与 `MY_LEX_CHAR` 共用一个 case**，`sql_lex.cc:1405`） |
| 26 | `MY_LEX_USER_VARIABLE_DELIMITER` | **引用标识符** | ⚠️ **命名严重误导**：它是**反引号 `` ` ``**（不是用户变量！），产出 `IDENT_QUOTED`；`ANSI_QUOTES` 时双引号也复用此分支 |
| 27 | `MY_LEX_SYSTEM_VAR` | 变量 | `@@` 系统变量（见 [`../infra/variables.md`](../infra/variables.md)） |
| 28 | `MY_LEX_IDENT_OR_KEYWORD` | 标识符 | **只在 `@@` 之后**由 `MY_LEX_SYSTEM_VAR` 转入（`sql_lex.cc:2025-2028`），允许 `@@GLOBAL.var` |
| 29 | `MY_LEX_IDENT_OR_HEX` | 标识符 | 标识符或 `x'..'` |
| 30 | `MY_LEX_IDENT_OR_BIN` | 标识符 | 标识符或 `b'..'` |
| 31 | `MY_LEX_IDENT_OR_NCHAR` | 标识符 | 标识符或 `N'..'` |
| 32 | `MY_LEX_IDENT_OR_DOLLAR_QUOTE` | 标识符 | 美元引用（`$$...$$`） |
| 33 | `MY_LEX_STRING_OR_DELIMITER` | 字符串 | 字符串或分隔符（标准 SQL 的 `''` 转义） |

⚠️ **8.0 早期版本有 `MY_LEX_OPERATOR_OR_IDENT`，8.0.39 已移除**（全仓 0 命中）。旧资料与本文早前版本里的这个状态已不成立。

作为对照，hint 子词法器有**自己一套**字符类枚举（`include/sql_chars.h:74-89`）——这印证了"hint 是一套完全独立的词法/语法栈"：

```cpp
enum MY_ATTRIBUTE((__packed__)) hint_lex_char_classes {
  HINT_CHR_ASTERISK,
  HINT_CHR_SLASH,
  HINT_CHR_IDENT,
  HINT_CHR_MB,
  HINT_CHR_DIGIT,
  HINT_CHR_QUOTE,
  HINT_CHR_BACKTICK,
  HINT_CHR_DOT,
  HINT_CHR_AT,
  HINT_CHR_COMMA,
  HINT_CHR_OPENBRACE,
  HINT_CHR_CLOSEBRACE,
  HINT_CHR_OPENPAREN,
  HINT_CHR_CLOSEPAREN,
  HINT_CHR_SPACE,
  HINT_CHR_OTHER
};
```

> ★ 15 个类 vs 主词法器的 33 个状态——hint 语法面窄得多（`/*+ ... */` 里只有名字、`(`、`)`、`,`、`@`、数字、字符串）。两张表同在 `lex_state_maps_st` 里（`main_map[256]` + `hint_map[256]`），由 `init_state_maps()` / `hint_lex_init_maps()` 各填一份。

> 未逐条剖析的多是边缘 token（`x'..'` / `b'..'` 引导符、`user@host`、`@@var`、美元引用）。它们各自归属不同功能域，上面表里的"去哪找"已给出指向。

### `lex_one_token()` 完整源码（4 段首尾相接）

> ⚠️ **勘误**：早前版本说"数字最终返回 `NUM`，值放进 `yylval->dec_lex_str`"——**错的**。词法器**从不把字面量转成数值**，`yylval->lex_str` 里始终是**原文字符串**；token 类型只由 `int_token()` 按"去符号去前导零后的**位数** + 逐位字典序"判定（见本节末）。真正的 `my_strtoll10` / `my_strntod` 发生在**语法层** `Item_int::init()` / `Item_float::init()`。

**第 1/4 段（`:1374-1468`）：入口、循环骨架、START / CHAR / SKIP / IDENT_OR_NCHAR / _DOLLAR_QUOTE / _HEX / _BIN**

```cpp
static int lex_one_token(Lexer_yystype *yylval, THD *thd) {
  uchar c = 0;
  bool comment_closed;
  int tokval, result_state;
  uint length;
  enum my_lex_states state;
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;
  const CHARSET_INFO *cs = thd->charset();
  const my_lex_states *state_map = cs->state_maps->main_map;  // ★ 每个字符集一份
  const uchar *ident_map = cs->ident_map;

  lip->yylval = yylval;  // The global state

  lip->start_token();
  state = lip->next_state;        // ★★ 状态机的"状态"跨 token 保存在 lip->next_state
  lip->next_state = MY_LEX_START;
  for (;;) {
    switch (state) {
      case MY_LEX_START:  // Start of token
        // Skip starting whitespace
        while (state_map[c = lip->yyPeek()] == MY_LEX_SKIP) {
          if (c == '\n') lip->yylineno++;

          lip->yySkip();
        }

        /* Start of real token */
        lip->restart_token();     // ★ 去掉前导空白，m_tok_start 前移
        c = lip->yyGet();
        state = state_map[c];     // ★★ 查表：字符 → 初始状态
        break;
      case MY_LEX_CHAR:  // Unknown or single char token
      case MY_LEX_SKIP:  // This should not happen
        if (c == '-' && lip->yyPeek() == '-' &&
            (my_isspace(cs, lip->yyPeekn(1)) ||
             my_iscntrl(cs, lip->yyPeekn(1)))) {
          state = MY_LEX_COMMENT;  // ★ "-- " 必须后跟空白/控制字符，所以 a--b 不是注释
          break;
        }

        if (c == '-' && lip->yyPeek() == '>')  // '->'
        {
          lip->yySkip();
          lip->next_state = MY_LEX_START;
          if (lip->yyPeek() == '>') {
            lip->yySkip();
            return JSON_UNQUOTED_SEPARATOR_SYM;   // '->>'
          }
          return JSON_SEPARATOR_SYM;              // '->'
        }

        if (c != ')') lip->next_state = MY_LEX_START;  // Allow signed numbers

        /*
          Check for a placeholder: it should not precede a possible identifier
          because of binlogging: when a placeholder is replaced with its value
          in a query for the binlog, the query must stay grammatically correct.
        */
        if (c == '?' && lip->stmt_prepare_mode && !ident_map[lip->yyPeek()])
          return (PARAM_MARKER);

        return ((int)c);           // ★★ 单字符 token 直接把 ASCII 码当 token 号

      case MY_LEX_IDENT_OR_NCHAR:
        if (lip->yyPeek() != '\'') {
          state = MY_LEX_IDENT;    // N 后不是单引号 → 它就是普通标识符开头
          break;
        }
        /* Found N'string' */
        lip->yySkip();  // Skip '
        if (!(yylval->lex_str.str = get_text(lip, 2, 1))) {
          state = MY_LEX_CHAR;  // Read char by char
          break;
        }
        yylval->lex_str.length = lip->yytoklen;
        return (NCHAR_STRING);

      case MY_LEX_IDENT_OR_DOLLAR_QUOTE:
        state = MY_LEX_IDENT;
        push_deprecated_warn_no_replacement(
            lip->m_thd, "$ as the first character of an unquoted identifier");
        break;

      case MY_LEX_IDENT_OR_HEX:
        if (lip->yyPeek() == '\'') {  // Found x'hex-number'
          state = MY_LEX_HEX_NUMBER;
          break;
        }
        [[fallthrough]];
      case MY_LEX_IDENT_OR_BIN:
        if (lip->yyPeek() == '\'') {  // Found b'bin-number'
          state = MY_LEX_BIN_NUMBER;
          break;
        }
        [[fallthrough]];
```

**第 2/4 段（`:1469-1764`）：IDENT / IDENT_SEP / NUMBER_IDENT / IDENT_START / USER_VARIABLE_DELIMITER / INT_OR_REAL / REAL / HEX / BIN**

```cpp
      case MY_LEX_IDENT:
        const char *start;
        if (use_mb(cs)) {
          result_state = IDENT_QUOTED;
          switch (my_mbcharlen(cs, lip->yyGetLast())) { ... }
          while (ident_map[c = lip->yyGet()]) {   // ★ ident_map 高速吞标识符字符
            switch (my_mbcharlen(cs, c)) { ... }
          }
        } else {
          for (result_state = c; ident_map[c = lip->yyGet()]; result_state |= c)
            ;
          /* If there were non-ASCII characters, mark that we must convert */
          result_state = result_state & 0x80 ? IDENT_QUOTED : IDENT;
        }
        length = lip->yyLength();
        start = lip->get_ptr();
        if (lip->ignore_space) {
          /*
            If we find a space then this can't be an identifier. We notice this
            below by checking start != lex->ptr.
          */
          for (; state_map[c] == MY_LEX_SKIP; c = lip->yyGet()) {
            if (c == '\n') lip->yylineno++;
          }
        }
        if (start == lip->get_ptr() && c == '.' && ident_map[lip->yyPeek()])
          lip->next_state = MY_LEX_IDENT_SEP;     // ★ db.tbl.col
        else {  // '(' must follow directly if function
          lip->yyUnget();
          if ((tokval = find_keyword(lip, length, c == '('))) {   // ★★ 关键字判定
            lip->next_state = MY_LEX_START;  // Allow signed numbers
            return (tokval);                 // Was keyword
          }
          lip->yySkip();  // next state does a unget
        }
        yylval->lex_str = get_token(lip, 0, length);

        /*
           Note: "SELECT _bla AS 'alias'"
           _bla should be considered as a IDENT if charset haven't been found.
           So we don't use MYF(MY_WME) with get_charset_by_csname to avoid
           producing an error.
        */
        if (yylval->lex_str.str[0] == '_') {
          auto charset_name = yylval->lex_str.str + 1;
          const CHARSET_INFO *underscore_cs =
              get_charset_by_csname(charset_name, MY_CS_PRIMARY, MYF(0));
          if (underscore_cs) {                    // ★ MYF(0)：查不到就当普通 IDENT，不报错
            lip->warn_on_deprecated_charset(underscore_cs, charset_name);
            if (underscore_cs == &my_charset_utf8mb4_0900_ai_ci) {
              underscore_cs = thd->variables.default_collation_for_utf8mb4;
            }
            yylval->charset = underscore_cs;
            lip->m_underscore_cs = underscore_cs; // ★ 暂存，下一个 TEXT_STRING 用
            lip->body_utf8_append(lip->m_cpp_text_start,
                                  lip->get_cpp_tok_start() + length);
            return (UNDERSCORE_CHARSET);
          }
        }

        lip->body_utf8_append(lip->m_cpp_text_start);
        lip->body_utf8_append_literal(thd, &yylval->lex_str, cs,
                                      lip->m_cpp_text_end);
        return (result_state);  // IDENT or IDENT_QUOTED

      case MY_LEX_IDENT_SEP:  // Found ident and now '.'
        yylval->lex_str.str = const_cast<char *>(lip->get_ptr());
        yylval->lex_str.length = 1;
        c = lip->yyGet();  // should be '.'
        if (uchar next_c = lip->yyPeek(); ident_map[next_c]) {
          lip->next_state =
              MY_LEX_IDENT_START;  // Next is an ident (not a keyword)
          // ★★ 点号之后的名字【绝不查关键字表】：db.select 里的 select 是列名
          if (next_c == '$')
            push_deprecated_warn_no_replacement(
                lip->m_thd, "$ as the first character of an unquoted identifier");
        } else  // Probably ` or "
          lip->next_state = MY_LEX_START;

        return ((int)c);           // '.' 自身作为 '.' token 返回

      case MY_LEX_NUMBER_IDENT:  // number or ident which num-start
        if (lip->yyGetLast() == '0') {
          c = lip->yyGet();
          if (c == 'x') {
            while (my_isxdigit(cs, (c = lip->yyGet()))) ;
            if ((lip->yyLength() >= 3) && !ident_map[c]) {
              yylval->lex_str = get_token(lip, 2, lip->yyLength() - 2);
              return (HEX_NUM);          // 0x...
            }
            lip->yyUnget();
            state = MY_LEX_IDENT_START;
            break;
          } else if (c == 'b') {
            while ((c = lip->yyGet()) == '0' || c == '1') ;
            if ((lip->yyLength() >= 3) && !ident_map[c]) {
              yylval->lex_str = get_token(lip, 2, lip->yyLength() - 2);
              return (BIN_NUM);          // 0b...
            }
            lip->yyUnget();
            state = MY_LEX_IDENT_START;
            break;
          }
          lip->yyUnget();
        }

        while (my_isdigit(cs, (c = lip->yyGet()))) ;
        if (!ident_map[c]) {  // Can't be identifier
          state = MY_LEX_INT_OR_REAL;
          break;
        }
        if (c == 'e' || c == 'E') {
          // The following test is written this way to allow numbers of type 1e1
          if (my_isdigit(cs, lip->yyPeek()) || (c = (lip->yyGet())) == '+' ||
              c == '-') {  // Allow 1E+10
            if (my_isdigit(cs, lip->yyPeek())) {
              lip->yySkip();
              while (my_isdigit(cs, lip->yyGet())) ;
              yylval->lex_str = get_token(lip, 0, lip->yyLength());
              return (FLOAT_NUM);
            }
          }
          lip->yyUnget();
        }
        [[fallthrough]];
      case MY_LEX_IDENT_START:  // We come here after '.'
        result_state = IDENT;
        ... while (ident_map[c = lip->yyGet()]) ...
        if (c == '.' && ident_map[lip->yyPeek()])
          lip->next_state = MY_LEX_IDENT_SEP;
        yylval->lex_str = get_token(lip, 0, lip->yyLength());
        ...
        return (result_state);

      case MY_LEX_USER_VARIABLE_DELIMITER:  // ★ 实际是反引号 `ident`
      {
        uint double_quotes = 0;
        char quote_char = c;  // Used char
        for (;;) {
          c = lip->yyGet();
          if (c == 0) {
            lip->yyUnget();
            return ABORT_SYM;  // Unmatched quotes
          }
          int var_length;
          if ((var_length = my_mbcharlen(cs, c)) == 1) {
            if (c == quote_char) {
              if (lip->yyPeek() != quote_char) break;
              c = lip->yyGet();          // ★ `` 转义成一个字面 `
              double_quotes++;
              continue;
            }
          } else if (use_mb(cs)) { ... }
        }
        if (double_quotes)
          yylval->lex_str = get_quoted_token(
              lip, 1, lip->yyLength() - double_quotes - 1, quote_char);
        else
          yylval->lex_str = get_token(lip, 1, lip->yyLength() - 1);
        if (c == quote_char) lip->yySkip();  // Skip end `
        lip->next_state = MY_LEX_START;
        ...
        return (IDENT_QUOTED);
      }
      case MY_LEX_INT_OR_REAL:  // Complete int or incomplete real
        if (c != '.') {         // Found complete integer number.
          yylval->lex_str = get_token(lip, 0, lip->yyLength());
          return int_token(yylval->lex_str.str, (uint)yylval->lex_str.length);
        }
        [[fallthrough]];
      case MY_LEX_REAL:  // Incomplete real number
        while (my_isdigit(cs, c = lip->yyGet())) ;
        if (c == 'e' || c == 'E') {
          c = lip->yyGet();
          if (c == '-' || c == '+') c = lip->yyGet();  // Skip sign
          if (!my_isdigit(cs, c)) { state = MY_LEX_CHAR; break; }
          while (my_isdigit(cs, lip->yyGet())) ;
          yylval->lex_str = get_token(lip, 0, lip->yyLength());
          return (FLOAT_NUM);
        }
        yylval->lex_str = get_token(lip, 0, lip->yyLength());
        return (DECIMAL_NUM);

      case MY_LEX_HEX_NUMBER:  // Found x'hexstring'
        lip->yySkip();
        while (my_isxdigit(cs, (c = lip->yyGet()))) ;
        if (c != '\'') return (ABORT_SYM);
        lip->yySkip();
        length = lip->yyLength();
        if ((length % 2) == 0) return (ABORT_SYM);  // ★ 奇数个十六进制位
        yylval->lex_str = get_token(lip, 2, length - 3);
        return (HEX_NUM);

      case MY_LEX_BIN_NUMBER:  // Found b'bin-string'
        lip->yySkip();
        while ((c = lip->yyGet()) == '0' || c == '1') ;
        if (c != '\'') return (ABORT_SYM);
        lip->yySkip();
        length = lip->yyLength();
        yylval->lex_str = get_token(lip, 2, length - 3);
        return (BIN_NUM);
```

**第 3/4 段（`:1765-1932`）：CMP_OP / LONG_CMP_OP / BOOL / STRING_OR_DELIMITER / STRING / COMMENT / LONG_COMMENT**

```cpp
      case MY_LEX_CMP_OP:  // Incomplete comparison operator
        if (state_map[lip->yyPeek()] == MY_LEX_CMP_OP ||
            state_map[lip->yyPeek()] == MY_LEX_LONG_CMP_OP)
          lip->yySkip();                 // ★ = ! > 后若还跟 =!<> 则再吃一个（!=, >=, <>）
        if ((tokval = find_keyword(lip, lip->yyLength() + 1, false))) {
          lip->next_state = MY_LEX_START;
          return (tokval);
        }
        state = MY_LEX_CHAR;  // Something fishy found
        break;

      case MY_LEX_LONG_CMP_OP:  // Incomplete comparison operator
        if (state_map[lip->yyPeek()] == MY_LEX_CMP_OP ||
            state_map[lip->yyPeek()] == MY_LEX_LONG_CMP_OP) {
          lip->yySkip();
          if (state_map[lip->yyPeek()] == MY_LEX_CMP_OP) lip->yySkip();  // ★ <=>, <<, <=
        }
        if ((tokval = find_keyword(lip, lip->yyLength() + 1, false))) {
          lip->next_state = MY_LEX_START;
          return (tokval);
        }
        state = MY_LEX_CHAR;
        break;

      case MY_LEX_BOOL:
        if (c != lip->yyPeek()) {      // ★ && / || 必须成对，单个 & | 落回 CHAR
          state = MY_LEX_CHAR;
          break;
        }
        lip->yySkip();
        tokval = find_keyword(lip, 2, false);  // Is a bool operator
        lip->next_state = MY_LEX_START;
        return (tokval);

      case MY_LEX_STRING_OR_DELIMITER:
        if (thd->variables.sql_mode & MODE_ANSI_QUOTES) {
          state = MY_LEX_USER_VARIABLE_DELIMITER;  // ★ ANSI_QUOTES：双引号当标识符
          break;
        }
        /* " used for strings */
        [[fallthrough]];
      case MY_LEX_STRING:  // Incomplete text string
        if (!(yylval->lex_str.str = get_text(lip, 1, 1))) {
          state = MY_LEX_CHAR;  // Read char by char
          break;
        }
        yylval->lex_str.length = lip->yytoklen;
        lip->body_utf8_append(lip->m_cpp_text_start);
        lip->body_utf8_append_literal(
            thd, &yylval->lex_str,
            lip->m_underscore_cs ? lip->m_underscore_cs : cs,  // ★ _utf8'...' 在此生效
            lip->m_cpp_text_end);
        lip->m_underscore_cs = nullptr;   // 一次性，用完即清
        return (TEXT_STRING);

      case MY_LEX_COMMENT:  //  Comment
        thd->m_parser_state->add_comment();
        while ((c = lip->yyGet()) != '\n' && c) ;   // ★ '# ' 与 '-- ' 吃到行尾
        lip->yyUnget();        // Safety against eof
        state = MY_LEX_START;  // Try again
        break;

      case MY_LEX_LONG_COMMENT: /* Long C comment? */
        if (lip->yyPeek() != '*') {
          state = MY_LEX_CHAR;  // Probable division  ★ '/' 后不是 '*' → 除号
          break;
        }
        thd->m_parser_state->add_comment();
        lip->yyUnget();
        lip->save_in_comment_state();

        if (lip->yyPeekn(2) == '!') {          // ★★ 可执行注释 /*! ... */
          lip->in_comment = DISCARD_COMMENT;
          lip->set_echo(false);                // ★ 关闭回显：/*! 三字符不进 cpp 缓冲
          lip->yySkip(); lip->yySkip(); lip->yySkip();

          /*
            The special comment format is very strict:
            '/' '*' '!', followed by exactly
            1 digit (major), 2 digits (minor), then 2 digits (dot).
            32302 -> 3.23.02
            50032 -> 5.0.32
            50114 -> 5.1.14
          */
          char version_str[6];
          if (my_isdigit(cs, (version_str[0] = lip->yyPeekn(0))) && ... ) {
            if (!my_isspace(cs, lip->yyPeekn(5))) {
              push_warning(thd, ..., ER_WARN_NO_SPACE_VERSION_COMMENT, ...);
            }
            version_str[5] = 0;
            ulong version = strtol(version_str, nullptr, 10);

            if (version <= MYSQL_VERSION_ID) {
              lip->yySkipn(5);
              /* Expand the content of the special comment as real code */
              lip->set_echo(true);   // ★★ 版本够新 → 内容当真代码展开
              state = MY_LEX_START;
              break;
            } else {
              /*
                Patch and skip the conditional comment to avoid it
                being propagated infinitely (eg. to a slave).
              */
              char *pcom = lip->yyUnput(' ');  // ★★ 把 '!' 覆写成空格（唯一原地改写查询文本处）
              comment_closed = !consume_comment(lip, 1);
              if (!comment_closed) *pcom = '!';  // 未闭合则还原
            }
          } else {
            /* Not a version comment. */
            state = MY_LEX_START;
            lip->set_echo(true);
            break;
          }
        } else {
          if (lip->in_comment != NO_COMMENT) {
            push_warning(..., ER_WARN_DEPRECATED_NESTED_COMMENT_SYNTAX);
          }
          lip->in_comment = PRESERVE_COMMENT;
          lip->yySkip();  // Accept /
          lip->yySkip();  // Accept *
          comment_closed = !consume_comment(lip, 0);
        }
        /* Unbalanced comments with a missing '*' '/' are a syntax error */
        if (!comment_closed) return (ABORT_SYM);
        state = MY_LEX_START;  // Try again
        lip->restore_in_comment_state();
        break;
```

**第 4/4 段（`:1933-2059`）：END_LONG_COMMENT / SET_VAR / SEMICOLON / EOL / END / REAL_OR_POINT / USER_END / HOSTNAME / SYSTEM_VAR / IDENT_OR_KEYWORD**

```cpp
      case MY_LEX_END_LONG_COMMENT:
        if ((lip->in_comment != NO_COMMENT) && lip->yyPeek() == '/') {
          lip->yyUnget();
          /* Accept '*' '/', with the proper echo */
          lip->set_echo(lip->in_comment == PRESERVE_COMMENT);  // ★ /*! */ 的 */ 不回显
          lip->yySkipn(2);
          lip->set_echo(true);

          /*
            C-style comments are replaced with a single space (as it
            is in C and C++).  If there is already a whitespace
            character at this point in the stream, the space is
            not inserted.

            See also ISO/IEC 9899:1999 §5.1.1.2
            ("Programming languages — C")
          */
          if (!my_isspace(cs, lip->yyPeek()) &&
              lip->get_cpp_ptr() != lip->get_cpp_buf() &&
              !my_isspace(cs, *(lip->get_cpp_ptr() - 1)))
            lip->cpp_inject(' ');       // ★★ 注释替换成一个空格（只写 cpp，不动 raw）
          lip->in_comment = NO_COMMENT;
          state = MY_LEX_START;
        } else
          state = MY_LEX_CHAR;  // Return '*'   ★ 不在注释里 → 乘法
        break;

      case MY_LEX_SET_VAR:  // Check if ':='
        if (lip->yyPeek() != '=') { state = MY_LEX_CHAR; break; }
        lip->yySkip();
        return (SET_VAR);

      case MY_LEX_SEMICOLON:  // optional line terminator
        state = MY_LEX_CHAR;  // Return ';'
        break;

      case MY_LEX_EOL:
        if (lip->eof()) {
          lip->yyUnget();  // Reject the last '\0'
          lip->set_echo(false);
          lip->yySkip();
          lip->set_echo(true);
          if (lip->in_comment != NO_COMMENT) return (ABORT_SYM);
          lip->next_state = MY_LEX_END;  // Mark for next loop
          return (END_OF_INPUT);         // ★ 第一次到 EOL 返回 END_OF_INPUT(≠0)
        }
        state = MY_LEX_CHAR;
        break;

      case MY_LEX_END:
        lip->next_state = MY_LEX_END;
        return (0);  // ★ 第二次才返回 0 = YYEOF

      /* Actually real shouldn't start with . but allow them anyhow */
      case MY_LEX_REAL_OR_POINT:
        if (my_isdigit(cs, lip->yyPeek()))
          state = MY_LEX_REAL;         // .5
        else {
          state = MY_LEX_IDENT_SEP;    // return '.'
          lip->yyUnget();              // Put back '.'
        }
        break;

      case MY_LEX_USER_END:  // end '@' of user@hostname
        switch (state_map[lip->yyPeek()]) {
          case MY_LEX_STRING:
          case MY_LEX_USER_VARIABLE_DELIMITER:
          case MY_LEX_STRING_OR_DELIMITER:
            break;                                  // @'user' / @`user` / @"user"
          case MY_LEX_USER_END:
            lip->next_state = MY_LEX_SYSTEM_VAR;    // ★ @@ → 系统变量
            break;
          default:
            lip->next_state = MY_LEX_HOSTNAME;      // ★ @host → 主机名
            break;
        }
        yylval->lex_str.str = const_cast<char *>(lip->get_ptr());
        yylval->lex_str.length = 1;
        return ((int)'@');

      case MY_LEX_HOSTNAME:
        for (c = lip->yyGet();
             my_isalnum(cs, c) || c == '.' || c == '_' || c == '$';
             c = lip->yyGet()) ;
        yylval->lex_str = get_token(lip, 0, lip->yyLength());
        return (LEX_HOSTNAME);

      case MY_LEX_SYSTEM_VAR:
        yylval->lex_str.str = const_cast<char *>(lip->get_ptr());
        yylval->lex_str.length = 1;
        lip->yySkip();  // Skip '@'
        lip->next_state =
            (state_map[lip->yyPeek()] == MY_LEX_USER_VARIABLE_DELIMITER
                 ? MY_LEX_START              // @@`var`
                 : MY_LEX_IDENT_OR_KEYWORD); // @@var_name
        return ((int)'@');

      case MY_LEX_IDENT_OR_KEYWORD:
        /*
          We come here when we have found two '@' in a row.
          We should now be able to handle:
          [(global | local | session) .]variable_name
        */
        for (result_state = 0; ident_map[c = lip->yyGet()]; result_state |= c) ;
        result_state = result_state & 0x80 ? IDENT_QUOTED : IDENT;
        if (c == '.') lip->next_state = MY_LEX_IDENT_SEP;   // @@global.var
        length = lip->yyLength();
        if (length == 0) return (ABORT_SYM);  // Names must be nonempty.
        if ((tokval = find_keyword(lip, length, false))) {
          lip->yyUnget();   // Put back 'c'
          return (tokval);  // Was keyword   ★ @@ 后允许 GLOBAL/SESSION/LOCAL
        }
        yylval->lex_str = get_token(lip, 0, length);
        ...
        return (result_state);
    }
  }
}
```

### 三个填充 `yylval` 的辅助函数（`sql/sql_lex.cc`）

```cpp
// :974 —— 普通 token：从 raw buffer 拷一份到 THD mem_root
static LEX_STRING get_token(Lex_input_stream *lip, uint skip, uint length) {
  LEX_STRING tmp;
  lip->yyUnget();  // ptr points now after last token char
  tmp.length = lip->yytoklen = length;
  tmp.str = lip->m_thd->strmake(lip->get_tok_start() + skip, tmp.length);

  lip->m_cpp_text_start = lip->get_cpp_tok_start() + skip;
  lip->m_cpp_text_end = lip->m_cpp_text_start + tmp.length;

  return tmp;
}

// :993 —— 带引号 token：顺手把 `` 折叠成 `
static LEX_STRING get_quoted_token(Lex_input_stream *lip, uint skip,
                                   uint length, char quote) {
  ...
  for (; to != end;) {
    if ((*to++ = *from++) == quote) {
      from++;  // Skip double quotes
      lip->m_cpp_text_start++;
    }
  }
  ...
}

// :1023 —— 字符串字面量：唯一做转义解的地方
static char *get_text(Lex_input_stream *lip, int pre_skip, int post_skip) {
  uchar c, sep;
  uint found_escape = 0;
  const CHARSET_INFO *cs = lip->m_thd->charset();

  lip->tok_bitmap = 0;
  sep = lip->yyGetLast();  // String should end with this
  while (!lip->eof()) {
    c = lip->yyGet();
    lip->tok_bitmap |= c;
    { ... use_mb(cs) && my_ismbchar(...) → skip_binary(l - 1); continue; }
    if (c == '\\' && !(lip->m_thd->variables.sql_mode &
                       MODE_NO_BACKSLASH_ESCAPES)) {   // ★ NO_BACKSLASH_ESCAPES 在这里生效
      found_escape = 1;
      if (lip->eof()) return nullptr;
      lip->yySkip();
    } else if (c == sep) {
      if (c == lip->yyGet()) {   // ★ '' 是两个单引号 → 折叠成一个
        found_escape = 1;        // duplicate. Remember for delete
        continue;
      } else
        lip->yyUnget();
      ...
      if (!found_escape) {
        lip->yytoklen = (uint)(end - str);
        memcpy(start, str, lip->yytoklen);   // ★ 无转义：直接 memcpy，零开销
        start[lip->yytoklen] = 0;
      } else {
        for (to = start; str != end; str++) {
          ...
          switch (*++str) {
            case 'n': *to++ = '\n'; break;
            case 't': *to++ = '\t'; break;
            case 'r': *to++ = '\r'; break;
            case 'b': *to++ = '\b'; break;
            case '0': *to++ = 0; break;      // Ascii null
            case 'Z': *to++ = '\032'; break; // ^Z must be escaped on Win32
            case '_':
            case '%':
              *to++ = '\\';  // ★ remember prefix for wildcard
              [[fallthrough]];
            default:
              *to++ = *str;
              break;
          }
          ...
        }
      }
      return start;
    }
  }
  return nullptr;  // unexpected end of query
}
```

### 数字 token 的判定：`int_token()`（`:1178`）

```cpp
/*
** Calc type of integer; long integer, longlong integer or real.
** Returns smallest type that match the string.
** When using unsigned long long values the result is converted to a real
** because else they will be unexpected sign changes because all calculation
** is done with longlong or double.
*/
static const char *long_str = "2147483647";              static const uint long_len = 10;
static const char *signed_long_str = "-2147483648";
static const char *longlong_str = "9223372036854775807"; static const uint longlong_len = 19;
static const char *signed_longlong_str = "-9223372036854775808"; static const uint signed_longlong_len = 19;
static const char *unsigned_longlong_str = "18446744073709551615"; static const uint unsigned_longlong_len = 20;

static inline uint int_token(const char *str, uint length) {
  if (length < long_len)  // quick normal case
    return NUM;                       // ★ 位数 < 10 一定是 NUM
  bool neg = false;

  if (*str == '+') { str++; length--; }        // Remove sign and pre-zeros
  else if (*str == '-') { str++; length--; neg = true; }
  while (*str == '0' && length) { str++; length--; }
  if (length < long_len) return NUM;

  uint smaller, bigger;
  const char *cmp;
  if (neg) {
    if (length == long_len) {
      cmp = signed_long_str + 1;
      smaller = NUM;      // If <= signed_long_str
      bigger = LONG_NUM;  // If >= signed_long_str
    } else if (length < signed_longlong_len)
      return LONG_NUM;
    else if (length > signed_longlong_len)
      return DECIMAL_NUM;             // ★ 负数超 -2^63 下界 → DECIMAL（不能无符号表示）
    else {
      cmp = signed_longlong_str + 1;
      smaller = LONG_NUM;
      bigger = DECIMAL_NUM;
    }
  } else {
    if (length == long_len) { cmp = long_str; smaller = NUM; bigger = LONG_NUM; }
    else if (length < longlong_len) return LONG_NUM;
    else if (length > longlong_len) {
      if (length > unsigned_longlong_len) return DECIMAL_NUM;
      cmp = unsigned_longlong_str; smaller = ULONGLONG_NUM; bigger = DECIMAL_NUM;
    } else { cmp = longlong_str; smaller = LONG_NUM; bigger = ULONGLONG_NUM; }
  }
  while (*cmp && *cmp++ == *str++) ;
  return ((uchar)str[-1] <= (uchar)cmp[-1]) ? smaller : bigger;  // ★ 逐位比 ASCII
}
```

> ★★ **词法层只判类型、不做转换**：`yylval->lex_str` 里始终是字面量原文。真正的解析在语法层（`sql/sql_yacc.yy`）：
> ```cpp
> int64_literal:                                   // :14789
>           NUM           { $$ = NEW_PTN Item_int(@$, $1); }
>         | LONG_NUM      { $$ = NEW_PTN Item_int(@$, $1); }
>         | ULONGLONG_NUM { $$ = NEW_PTN Item_uint(@$, $1.str, $1.length); }
>         ;
> ```
> `Item_int::init()`（`sql/item.cc:3240`）才调 `my_strtoll10`；`Item_float::init()`（`:6971`）调 `my_strntod`；`Item_decimal`（`:3299`）调 `str2my_decimal`。

---

## 三、Lex_input_stream：输入缓冲与回退

`lex_one_token` 的所有读字符操作都通过 `Lex_input_stream`（`sql/sql_lex.h:3355`），它封装了**缓冲区 + 游标 + 回显**：

| 方法 | 作用 |
|------|------|
| `yyGet()` | 读一个字符并前进（`:3412`） |
| `yyPeek()` | 偷看下一个字符，**不前进**（`:3428`） |
| `yyPeekn(n)` | 偷看第 n 个字符（`:3437`） |
| `yySkip()` / `yySkipn(n)` | 前进 1 / n 个字符（`:3455`/`:3467`） |
| `yyUnget()` / `yyUnput(ch)` | 回退一个字符 / 塞回一个字符（`:3447`/`:3482`） |
| `skip_binary(n)` | 跳过 n 字节（多字节字符用）（`:3399`） |

**`yyPeek` + `yyUnget` 是 lookahead 的基石**：状态机经常"偷看下一个字符决定走哪个状态，看完再回退"，这是 LALR(2) 特判和运算符识别的实现手段。

#### 双缓冲：raw / cpp

★ 这是本类最容易被忽略的设计——它维护**两条并行的缓冲**，前面的 `get_ptr()` / `get_cpp_ptr()`、`yylloc->raw` / `yylloc->cpp` 都源于此：

```cpp
/**
  This class represents the character input stream consumed during lexical
  analysis.

  In addition to consuming the input stream, this class performs some comment
  pre processing, by filtering out out-of-bound special text from the query
  input stream.

  Two buffers, with pointers inside each, are maintained in parallel. The
  'raw' buffer is the original query text, which may contain out-of-bound
  comments. The 'cpp' (for comments pre processor) is the pre-processed buffer
  that contains only the query text that should be seen once out-of-bound data
  is removed.
*/
```

> ★ 所以 `/*!80000 ... */` 的"展开"动作的实质是：`/*!` 三字符**不写进 cpp**（`set_echo(false)`），内容照写 —— 于是 general log / binlog 里看到的就是去掉标记后的真代码。而 `MY_LEX_END_LONG_COMMENT` 里的 `cpp_inject(' ')` 只写 cpp，**raw 不动**。

#### 完整定义（`sql/sql_lex.h:3355-3796`；省略部分只含 `body_utf8_*` 系列与 `warn_on_deprecated_*`）

```cpp
class Lex_input_stream {
 public:
  explicit Lex_input_stream(uint grammar_selector_token_arg)
      : grammar_selector_token(grammar_selector_token_arg) {}

  bool init(THD *thd, const char *buff, size_t length);
  void reset(const char *buff, size_t length);

  /**
    Set the echo mode.

    When echo is true, characters parsed from the raw input stream are
    preserved. When false, characters parsed are silently ignored.
    @param echo the echo mode.
  */
  void set_echo(bool echo) { m_echo = echo; }

  void save_in_comment_state() {
    m_echo_saved = m_echo;
    in_comment_saved = in_comment;
  }
  void restore_in_comment_state() {
    m_echo = m_echo_saved;
    in_comment = in_comment_saved;
  }

  /** Skip binary from the input stream. @param n number of bytes to accept. */
  void skip_binary(int n) {
    assert(m_ptr + n <= m_end_of_query);
    if (m_echo) {
      memcpy(m_cpp_ptr, m_ptr, n);
      m_cpp_ptr += n;
    }
    m_ptr += n;
  }

  /** Get a character, and advance in the stream. */
  unsigned char yyGet() {
    assert(m_ptr <= m_end_of_query);
    char c = *m_ptr++;
    if (m_echo) *m_cpp_ptr++ = c;    // ★★ 回显：同时写进 cpp 缓冲
    return c;
  }

  unsigned char yyGetLast() const { return m_ptr[-1]; }

  /** Look at the next character to parse, but do not accept it. */
  unsigned char yyPeek() const { assert(m_ptr <= m_end_of_query); return m_ptr[0]; }

  /** Look ahead at some character to parse. */
  unsigned char yyPeekn(int n) const {
    assert(m_ptr + n <= m_end_of_query);
    return m_ptr[n];
  }

  /**
    Cancel the effect of the last yyGet() or yySkip().
    Note that the echo mode should not change between calls to yyGet / yySkip
    and yyUnget. The caller is responsible for ensuring that.
  */
  void yyUnget() { m_ptr--; if (m_echo) m_cpp_ptr--; }

  /** Accept a character, by advancing the input stream. */
  void yySkip() {
    assert(m_ptr <= m_end_of_query);
    if (m_echo) *m_cpp_ptr++ = *m_ptr++;
    else m_ptr++;
  }

  /** Accept multiple characters at once. */
  void yySkipn(int n) {
    assert(m_ptr + n <= m_end_of_query);
    if (m_echo) { memcpy(m_cpp_ptr, m_ptr, n); m_cpp_ptr += n; }
    m_ptr += n;
  }

  char *yyUnput(char ch) {
    *--m_ptr = ch;
    if (m_echo) m_cpp_ptr--;
    return m_ptr;                    // ★ 返回被写回的位置（版本注释 patch 时要靠它）
  }

  /**
    Inject a character into the pre-processed stream.

    Note, this function is used to inject a space instead of multi-character
    C-comment. Thus there is no boundary checks here (basically, we replace
    N-chars by 1-char here).
  */
  char *cpp_inject(char ch) { *m_cpp_ptr = ch; return ++m_cpp_ptr; }

  bool eof() const { return (m_ptr >= m_end_of_query); }
  bool eof(int n) const { return ((m_ptr + n) >= m_end_of_query); }

  const char *get_buf() const { return m_buf; }
  const char *get_cpp_buf() const { return m_cpp_buf; }
  const char *get_end_of_query() const { return m_end_of_query; }

  /** Mark the stream position as the start of a new token. */
  void start_token() {
    m_tok_start = m_ptr; m_tok_end = m_ptr;
    m_cpp_tok_start = m_cpp_ptr; m_cpp_tok_end = m_cpp_ptr;
  }

  /** Adjust the starting position of the current token.
      This is used to compensate for starting whitespace. */
  void restart_token() { m_tok_start = m_ptr; m_cpp_tok_start = m_cpp_ptr; }

  const char *get_tok_start() const { return m_tok_start; }
  const char *get_cpp_tok_start() const { return m_cpp_tok_start; }
  const char *get_tok_end() const { return m_tok_end; }
  const char *get_cpp_tok_end() const { return m_cpp_tok_end; }
  const char *get_ptr() const { return m_ptr; }
  const char *get_cpp_ptr() const { return m_cpp_ptr; }

  /** Get the length of the current token, in the raw buffer. */
  uint yyLength() const {
    /*
      The assumption is that the lexical analyser is always 1 character ahead,
      which the -1 account for.
    */
    assert(m_ptr > m_tok_start);
    return (uint)((m_ptr - m_tok_start) - 1);
  }

  /** Current thread. */
  THD *m_thd;
  /** Current line number. */
  uint yylineno;
  /** Length of the last token parsed. */
  uint yytoklen;
  /** Interface with bison, value of the last token parsed. */
  Lexer_yystype *yylval;

  /**
    LALR(2) resolution, look ahead token.
    Value of the next token to return, if any,
    or -1, if no token was parsed in advance.
    Note: 0 is a legal token, and represents YYEOF.
  */
  int lookahead_token;

  /** LALR(2) resolution, value of the look ahead token.*/
  Lexer_yystype *lookahead_yylval;

  bool skip_digest;
  void add_digest_token(uint token, Lexer_yystype *yylval);
  void reduce_digest_token(uint token_left, uint token_right);

  /**
    True if this scanner tokenizes a partial query (partition expression,
    generated column expression etc.)
  */
  bool is_partial_parser() const { return grammar_selector_token >= 0; }

  const CHARSET_INFO *query_charset;

 private:
  /** Pointer to the current position in the raw input stream. */
  char *m_ptr;
  const char *m_tok_start;      ///< Starting position of the last token parsed
  const char *m_tok_end;        ///< Ending position of the previous token
  const char *m_end_of_query;   ///< End of the query text
  const char *m_buf;            ///< Beginning of the query text
  size_t m_buf_length;

  /** Echo the parsed stream to the pre-processed buffer. */
  bool m_echo;
  bool m_echo_saved;

  char *m_cpp_buf;              ///< Pre-processed buffer
  char *m_cpp_ptr;              ///< Pointer in the pre-processed stream
  const char *m_cpp_tok_start;
  const char *m_cpp_tok_end;

  char *m_body_utf8;            ///< UTF8-body buffer created during parsing
  char *m_body_utf8_ptr;
  const char *m_cpp_utf8_processed_ptr;

 public:
  /** Current state of the lexical analyser. */
  enum my_lex_states next_state;

  /**
    Position of ';' in the stream, to delimit multiple queries.
    This delimiter is in the raw buffer.
  */
  const char *found_semicolon;

  /** Token character bitmaps, to detect 7bit strings. */
  uchar tok_bitmap;

  /** SQL_MODE = IGNORE_SPACE. */
  bool ignore_space;            // ⚠️ 不是 m_ignore_spaces

  /** true if we're parsing a prepared statement: in this mode
      we should allow placeholders. */
  bool stmt_prepare_mode;
  /** true if we should allow multi-statements. */
  bool multi_statements;

  /** State of the lexical analyser for comments. */
  enum_comment_state in_comment;
  enum_comment_state in_comment_saved;

  const char *m_cpp_text_start;
  const char *m_cpp_text_end;

  /**
    Character set specified by the character-set-introducer.
    NOTE: this member must be used within MYSQLlex() function only.
  */
  const CHARSET_INFO *m_underscore_cs;

  /** Current statement digest instrumentation. */
  sql_digest_state *m_digest;

  /**
    The synthetic 1st token to prepend token stream with.

    This token value tricks parser to simulate multiple %start-ing points.
    Currently the grammar is aware of 4 such synthetic tokens:
    1. GRAMMAR_SELECTOR_PART for partitioning stuff from DD,
    2. GRAMMAR_SELECTOR_GCOL for generated column stuff from DD,
    3. GRAMMAR_SELECTOR_EXPR for generic single expressions from DD/.frm.
    4. GRAMMAR_SELECTOR_CTE for generic subquery expressions from CTEs.
    5. -1 when parsing with the main grammar (no grammar selector available).

    @note yylex() is expected to return the value of type int:
          0 is for EOF and everything else for real token numbers.
          Bison, in its turn, generates positive token numbers.
          So, the negative grammar_selector_token means "not a token".
          In other words, -1 is "empty value".
  */
  const int grammar_selector_token;   // ★★ 与 04 篇 start_entry 的 6 个分支一一对应

  bool text_string_is_7bit() const { return !(tok_bitmap & 0x80); }
};
```

```cpp
// sql/sql_lex.h:3317 —— 注释三态
enum enum_comment_state {
  /** Not parsing comments. */
  NO_COMMENT,
  /**
    Parsing comments that need to be preserved.
    (Copy '/' '*' and '*' '/' sequences to the preprocessed buffer.)
    Typically, these are user comments '/' '*' ... '*' '/'.
  */
  PRESERVE_COMMENT,
  /**
    Parsing comments that need to be discarded.
    (Don't copy '/' '*' '!' and '*' '/' sequences to the preprocessed buffer.)
    Typically, these are special comments '/' '*' '!' ... '*' '/',
    or '/' '*' '!' 'M' 'M' 'm' 'm' 'm' ... '*' '/', where the comment
    markers should not be expanded.
  */
  DISCARD_COMMENT
};
```

#### `init()` / `reset()`（`sql/sql_lex.cc:203` / `:231`）

```cpp
/**
  Perform initialization of Lex_input_stream instance.

  Basically, a buffer for a pre-processed query. This buffer should be large
  enough to keep a multi-statement query. The allocation is done once in
  Lex_input_stream::init() in order to prevent memory pollution when
  the server is processing large multi-statement queries.
*/
bool Lex_input_stream::init(THD *thd, const char *buff, size_t length) {
  query_charset = thd->charset();
  m_cpp_buf = (char *)thd->alloc(length + 1);
  if (m_cpp_buf == nullptr) return true;
  m_thd = thd;
  reset(buff, length);
  return false;
}

/**
  Prepare Lex_input_stream instance state for use for handling next SQL
  statement.

  It should be called between two statements in a multi-statement query.
  The operation resets the input stream to the beginning-of-parse state,
  but does not reallocate m_cpp_buf.
*/
void Lex_input_stream::reset(const char *buffer, size_t length) {
  yylineno = 1;
  yytoklen = 0;
  yylval = nullptr;
  lookahead_token = grammar_selector_token;   // ★ 主语法时 = -1 ⇒ "无预读"
  static Lexer_yystype dummy_yylval;
  lookahead_yylval = &dummy_yylval;
  skip_digest = false;
  /*
    Lex_input_stream modifies the query string in one special case (sic!).
    yyUnput() modifises the string when patching version comments.
    This is done to prevent newer slaves from executing a different
    statement than older masters.

    For now, cast away const here. This means that e.g. SHOW PROCESSLIST
    can see partially patched query strings. ...
  */
  m_ptr = const_cast<char *>(buffer);
  m_tok_start = nullptr;
  m_tok_end = nullptr;
  m_end_of_query = buffer + length;
  m_buf = buffer;
  m_buf_length = length;
  m_echo = true;
  m_cpp_tok_start = nullptr;
  m_cpp_tok_end = nullptr;
  m_body_utf8 = nullptr;
  m_cpp_utf8_processed_ptr = nullptr;
  next_state = MY_LEX_START;
  found_semicolon = nullptr;
  ignore_space = m_thd->variables.sql_mode & MODE_IGNORE_SPACE;  // ★ IGNORE_SPACE 在此固化
  stmt_prepare_mode = false;
  multi_statements = true;
  in_comment = NO_COMMENT;
  m_underscore_cs = nullptr;
  m_cpp_ptr = m_cpp_buf;
}
```

> ★ `lookahead_token = grammar_selector_token` 这句很妙：主语法时它是 `-1`，于是 `MYSQLlex` 里 `if (lip->lookahead_token >= 0)` 判不出来，走正常扫描；而 5 个 `GRAMMAR_SELECTOR_*`（≥0）会被当作"第一个 token"直接吐给 parser —— 这就是 04 篇 `start_entry` 能靠同一个 `yyparse` 支持 6 个起始点的机制。

#### `get_lineno()`：O(n) 重扫，不是增量维护

```cpp
// sql/sql_lex.cc:1129
uint Lex_input_stream::get_lineno(const char *raw_ptr) const {
  assert(m_buf <= raw_ptr && raw_ptr <= m_end_of_query);
  if (!(m_buf <= raw_ptr && raw_ptr <= m_end_of_query)) return 1;

  uint ret = 1;
  const CHARSET_INFO *cs = m_thd->charset();
  for (const char *c = m_buf; c < raw_ptr; c++) {
    uint mb_char_len;
    if (use_mb(cs) && (mb_char_len = my_ismbchar(cs, c, m_end_of_query))) {
      c += mb_char_len - 1;  // skip the rest of the multibyte character
      continue;              // we don't expect '\n' there
    }
    if (*c == '\n') ret++;
  }
  return ret;
}
```

> 常态行号靠 `yylineno` 在 `MY_LEX_START`（`:1395`）、`MY_LEX_IDENT`（`:1517`）、`MY_LEX_COMMENT` 处**手工 ++**；`get_lineno()` 只在报错 / hint 场景调用，所以 O(n) 可接受。

---

## 四、find_keyword：关键字识别

标识符读完后，`find_keyword`（`sql/sql_lex.cc`）判断它是不是关键字。关键字表由 `gen_lex_hash` 构建脚本**离线生成**（`sql/gen_lex_hash.cc` → 生成哈希表），编译期固化，运行时 O(1) 哈希查找。

**完整源码**（`sql/sql_lex.cc:905-936`）：

```cpp
static int find_keyword(Lex_input_stream *lip, uint len, bool function) {
  const char *tok = lip->get_tok_start();

  const SYMBOL *symbol =
      function ? Lex_hash::sql_keywords_and_funcs.get_hash_symbol(tok, len)
               : Lex_hash::sql_keywords.get_hash_symbol(tok, len);
  // ★★ function=true ⇒ 查"关键字+函数"表；否则只查纯关键字表

  if (symbol) {
    lip->yylval->keyword.symbol = symbol;
    lip->yylval->keyword.str = const_cast<char *>(tok);
    lip->yylval->keyword.length = len;

    if ((symbol->tok == NOT_SYM) &&
        (lip->m_thd->variables.sql_mode & MODE_HIGH_NOT_PRECEDENCE))
      return NOT2_SYM;                      // ★ 同一个符号按 sql_mode 换 token
    if ((symbol->tok == OR_OR_SYM) &&
        !(lip->m_thd->variables.sql_mode & MODE_PIPES_AS_CONCAT)) {
      push_deprecated_warn(lip->m_thd, "|| as a synonym for OR", "OR");
      return OR2_SYM;                       // ★ || 默认当 OR2_SYM
    }

    lip->yylval->optimizer_hints = nullptr;
    if (symbol->group & SG_HINTABLE_KEYWORDS) {   // ★ 只有这 5 个 DML 关键字才去吃 /*+ ... */
      lip->add_digest_token(symbol->tok, lip->yylval);
      if (consume_optimizer_hints(lip)) return ABORT_SYM;
      lip->skip_digest = true;              // ★ 置位（MYSQLlex 收尾时会消费）
    }

    return symbol->tok;
  }
  return 0;                                 // 0 = 不是关键字
}
```

`/*+ ... */` 的识别不在主词法循环里，而在 `consume_optimizer_hints()`（`sql/sql_lex.cc:865-903`）：

```cpp
static bool consume_optimizer_hints(Lex_input_stream *lip) {
  const my_lex_states *state_map = lip->query_charset->state_maps->main_map;
  int whitespace = 0;
  uchar c = lip->yyPeek();
  size_t newlines = 0;

  for (; state_map[c] == MY_LEX_SKIP;
       whitespace++, c = lip->yyPeekn(whitespace)) {
    if (c == '\n') newlines++;              // 跨过空白，记住换行数以维护行号
  }

  if (lip->yyPeekn(whitespace) == '/' && lip->yyPeekn(whitespace + 1) == '*' &&
      lip->yyPeekn(whitespace + 2) == '+') {   // ★ 逐字符比对 "/*+"
    lip->yylineno += newlines;
    lip->yySkipn(whitespace);  // skip whitespace

    Hint_scanner hint_scanner(lip->m_thd, lip->yylineno, lip->get_ptr(),
                              lip->get_end_of_query() - lip->get_ptr(),
                              lip->m_digest);   // ★ 换一套词法器
    PT_hint_list *hint_list = nullptr;
    int rc = HINT_PARSER_parse(lip->m_thd, &hint_scanner, &hint_list);
    if (rc == 2) return true;  // Bison's internal OOM error
    if (rc == 1) {
      lip->start_token();  // adjust error message text pointer to "/*+"
      return true;         // → find_keyword() 返回 ABORT_SYM，整句报语法错
    }
    lip->yylineno = hint_scanner.get_lineno();
    lip->yySkipn(hint_scanner.get_ptr() - lip->get_ptr());   // ★ 主 lexer 跳过 hint 正文
    lip->yylval->optimizer_hints = hint_list;   // NULL in case of syntax error
    lip->m_digest = hint_scanner.get_digest();  // NULL is digest buf. is full.
    return false;
  } else
    return false;   // 不是 "/*+" → 后面的 /* */ 按普通注释处理
}
```

两个对外辅助函数（`:938-970`）：

```cpp
bool is_keyword(const char *name, size_t len) {
  assert(len != 0);
  return Lex_hash::sql_keywords.get_hash_symbol(name, len) != nullptr;
}

bool is_lex_native_function(const LEX_STRING *name) {
  assert(name != nullptr);
  return Lex_hash::sql_keywords_and_funcs.get_hash_symbol(
             name->str, (uint)name->length) != nullptr;
}
```

> ⚠️ 这两个函数 doxygen 的 `RETURN VALUES` 注释**写反了**（"0 name is a keyword / 1 name isn't"），实际返回 `true` 表示**是**关键字/函数。

**第三个参数 `c == '('` 很关键**：MySQL 允许函数名和关键字冲突（如 `LEFT` 既是函数名又是关键字）。传 `c == '('` 表示"标识符后面紧跟左括号"，`find_keyword` 据此区分：`LEFT(` 是函数调用（返回 `IDENT`），`LEFT` 单独是关键字（返回 `LEFT_SYM`）。

**关键字分三类**（`lex.h` 里的 `%token` 声明决定）：
- **保留字**（reserved）：如 `SELECT`/`FROM`/`WHERE`，不能当标识符
- **非保留关键字**：如 `MASTER`/`LOCK`，可当标识符
- **函数名关键字**：`LEFT`/`RIGHT` 等，特殊处理

#### 关键字登记表 `symbols[]`：SYM 四宏与 SG 四分类

`find_keyword` 查的是**运行时哈希表**，但那张哈希表不是手写的，而是由 `sql/lex.h` 里的**登记表 `symbols[]`** 经 `gen_lex_hash` 离线生成的。理解 lexer 必须分清这两层：

**真实源码**（`sql/lex.h:27-64`）——★ 注意 `NO_YACC_SYMBOLS` 这个开关，它是"同一个 `lex.h` 被服务端和构建工具各用一次"的关键：

```cpp
/* This file includes all reserved words and functions */

#include "m_string.h"
#include "sql/lex_symbol.h"

/* We don't want to include sql_yacc.h into gen_lex_hash */
#ifdef NO_YACC_SYMBOLS
#define SYM_OR_NULL(A) 0          // ★ 构建期工具 gen_lex_hash.cc 定义了它
                                  //   ⇒ 生成哈希表时所有 token 值都是 0，只保留 name/length/group
#else

#include "sql/sql_hints.yy.h"
#include "sql/sql_yacc.h"

#define SYM_OR_NULL(A) A          // ★ 编译 server 时，token 用 bison 生成的枚举常量
#endif

#define HINT_COMMENT_STARTER "/*+"
#define HINT_COMMENT_TERMINATOR "*/"

#define SYM(T, A) STRING_WITH_LEN(T), SYM_OR_NULL(A), SG_KEYWORDS
#define SYM_FN(T, A) STRING_WITH_LEN(T), SYM_OR_NULL(A), SG_FUNCTIONS
#define SYM_HK(T, A) STRING_WITH_LEN(T), SYM_OR_NULL(A), SG_HINTABLE_KEYWORDS
#define SYM_H(T, A) STRING_WITH_LEN(T), SYM_OR_NULL(A), SG_HINTS
```

> ★ `STRING_WITH_LEN(T)` 展开为 `T, (uint)(sizeof(T)-1)`，正好填满 `SYMBOL` 的前两个字段；所以 `{SYM("SELECT", SELECT_SYM)}` 是把 `{name, length, tok, group}` 四个字段一气呵成初始化。四个宏**唯一的区别**就是第 4 个字段 `group`。

```cpp
// sql/lex.h:61 —— symbols[] 定义起点，结束于 :919
static const SYMBOL symbols[] = {
    /*
     Insert new SQL keywords after that commentary (by alphabetical order):
    */
    {SYM("&&", AND_AND_SYM)},
    {SYM("<", LT)},
    {SYM("<=", LE)},
    {SYM("<>", NE)},
    {SYM("!=", NE)},
    {SYM("=", EQ)},
    {SYM(">", GT_SYM)},
    {SYM(">=", GE)},
    {SYM("<<", SHIFT_LEFT)},
    {SYM(">>", SHIFT_RIGHT)},
    {SYM("<=>", EQUAL_SYM)},
    {SYM("ACCESSIBLE", ACCESSIBLE_SYM)},
    {SYM("ACCOUNT", ACCOUNT_SYM)},
    ...
```

> ⚠️ **勘误**：`symbols[]` 里**不止关键字**——开头这 11 条全是**运算符**（`&&` `<` `<=` `<>` `!=` `=` `>` `>=` `<<` `>>` `<=>`）。这正是 `MY_LEX_CMP_OP` / `MY_LEX_LONG_CMP_OP` / `MY_LEX_BOOL` 分支最后要调 `find_keyword()` 的原因：多字符运算符是靠**查符号表**认出来的，不是硬编码。

四类分区（`:827-919`）与实测计数：

```cpp
    /*
      Place keywords that accept optimizer hints below this comment.
    */
    {SYM_HK("DELETE", DELETE_SYM)},      // :830
    {SYM_HK("INSERT", INSERT_SYM)},
    {SYM_HK("REPLACE", REPLACE_SYM)},
    {SYM_HK("SELECT", SELECT_SYM)},
    {SYM_HK("UPDATE", UPDATE_SYM)},      // :834  ← 只有这 5 个
    ...
    {SYM_FN("CAST", CAST_SYM)},          // :843
    {SYM_FN("MID", SUBSTRING)},          // ★ 别名：MID 与 SUBSTRING 同 token
    {SYM_FN("SUBSTR", SUBSTRING)},
    ...
    {SYM_H("BKA", BKA_HINT)},            // :877
    {SYM_H("MAX_EXECUTION_TIME", MAX_EXECUTION_TIME_HINT)},
    ...
    {SYM_H("NO_DERIVED_CONDITION_PUSHDOWN",
           NO_DERIVED_CONDITION_PUSHDOWN_HINT)}};   // :919
```

| 宏 | 元素数 | 分区行号 |
|---|---:|---|
| `SYM` | **759** | 65–826 |
| `SYM_HK` | **5** | 830–834 |
| `SYM_FN` | **35** | 839–873 |
| `SYM_H` | **42** | 877–919 |
| **合计** | **841** | |

★ 四个宏对应四种 **`SG_*` 分组**（`SYMBOL::group` 字段），这是"一个 token 可能被登记多次"的原因：

| 宏 | 分组 | 含义 |
|---|---|---|
| `SYM` | `SG_KEYWORDS` | 普通关键字 |
| `SYM_FN` | `SG_FUNCTIONS` | 函数名（`LEFT` 等，可与关键字冲突，靠 `c=='('` 区分） |
| `SYM_HK` | `SG_HINTABLE_KEYWORDS` | **hintable 关键字**：SELECT/INSERT/UPDATE/DELETE/REPLACE —— 后面紧跟 `/*+` 时唤醒 hint 语法器（见 [`07_optimize/11_optimizer_hints.md`](07_optimize/11_optimizer_hints.md)） |
| `SYM_H` | `SG_HINTS` | hint **注释内部**的关键字（`JOIN_ORDER`、`INDEX` 等 hint 名） |

★ **由此理解一个关键点：`SYM_HK` 的登记是 hint 机制在 lexer 侧的入口**——它把"SELECT/INSERT/UPDATE/DELETE/REPLACE"标成 hintable，这样 lexer 才能在这些关键字之后检测 `/*+` 并切入 hint 语法器。而 `SYM_H` 让 hint 注释内部能用与主语法器不同的关键字集合。这两组宏的登记位置，就是"hint 如何接入词法层"的答案。

### 4.1 `SYM_GROUP` 是枚举不是宏，且 `SG_MAIN_PARSER` 不含 hints

```cpp
// sql/lex_symbol.h:29
enum SYM_GROUP {
  SG_KEYWORDS = 1 << 0,           // SQL keywords and reserved words
  SG_FUNCTIONS = 1 << 1,          // very special native SQL functions
  SG_HINTABLE_KEYWORDS = 1 << 2,  // SQL keywords that accept optimizer hints
  SG_HINTS = 1 << 3,              // optimizer hint parser keywords

  /* All tokens of the main parser: */
  SG_MAIN_PARSER = SG_KEYWORDS | SG_HINTABLE_KEYWORDS | SG_FUNCTIONS
};                                // ★★ 注意不含 SG_HINTS

struct SYMBOL {
  const char *name;
  const unsigned int length;
  const unsigned int tok;
  /** group mask, see SYM_GROUP enum for bits. */
  int group;
};

struct LEX_SYMBOL {
  const SYMBOL *symbol;   // 查表命中的静态 SYMBOL 指针
  char *str;              // 指向查询原文中的该 token（非 0 结尾！要配 length 用）
  unsigned int length;
};
```

### 4.2 `Lex_hash`：⚠️ 不是哈希函数，是 Knuth 数字搜索 trie

> ⚠️ **勘误**：资料里常说"完美哈希"。`gen_lex_hash.cc:29-77` 的文件头明写算法出自 **Knuth《TAOCP》Vol.3 §6.3 "Digital searching"**——是**按长度分层的逐字符 trie**，构建期穷举出无冲突布局，运行时零探测。没有 murmur、没有 CRC、没有 gperf。

```cpp
// sql/gen_lex_hash.cc:29（文件头注释节选）
Algorithm implemented from a description in
"The Art of Computer Programming" by Donald E. Knuth
Volume 3 "Sorting and searching",
chapter 6.3 "Digital searching"

as illustration of data structures, imagine next table:
static SYMBOL symbols[] = { { "ADD", ...}, { "AND", ...}, { "DAY", ...} };

for this structure, presented program generate next searching-structure:

+-----------+-+-+-+
|       len |1|2|3|        ← ★ 第 0 层按【长度】索引
+-----------+-+-+-+
|first_char |0|0|a|
|last_char  |0|0|d|
|link       |0|0|+|
                 |
                 V
       +----------+-+-+-+--+
       |    1 char|a|b|c|d |    ← 每个节点是 [first_char, last_char] 区间 + 子节点基址
       +----------+-+-+-+--+
       ...
                   V
            symbols[2] ( "DAY" )

for optimization, link is the 16-bit index in 'symbols' or search-array..
So, we can read full search-structure as 32-bit word
```

```cpp
// sql/sql_lex_hash.h:31
class Lex_hash {
 private:
  const unsigned char *hash_map;        // 指向生成的 xxx_map[] 字节表（每 4 字节一个节点）
  const unsigned int entry_max_len;     // 该表最长符号长度

 public:
  Lex_hash(const unsigned char *hash_map_arg, unsigned int entry_max_len_arg)
      : hash_map(hash_map_arg), entry_max_len(entry_max_len_arg) {}

  const struct SYMBOL *get_hash_symbol(const char *s, unsigned int len) const;

  static const Lex_hash sql_keywords;             // SG_KEYWORDS | SG_HINTABLE_KEYWORDS
  static const Lex_hash sql_keywords_and_funcs;   // SG_MAIN_PARSER
  static const Lex_hash hint_keywords;            // ★ 第三个！给 Hint_scanner 用
};
```

```cpp
// sql/sql_lex_hash.cc:37 —— 三个静态实例的定义（★ 不在 sql_lex.cc）
const Lex_hash Lex_hash::sql_keywords(sql_keywords_map, sql_keywords_max_len);
const Lex_hash Lex_hash::sql_keywords_and_funcs(sql_keywords_and_funcs_map,
                                                sql_keywords_and_funcs_max_len);
const Lex_hash Lex_hash::hint_keywords(hint_keywords_map, hint_keywords_max_len);
```

**查找实现**（`sql/sql_lex_hash.cc:88`）——32 位字解码 + `to_upper_lex` 大小写折叠：

```cpp
inline int lex_casecmp(const char *s, const char *t, uint len) {
  while (len-- != 0 && to_upper_lex[(uchar)*s++] == to_upper_lex[(uchar)*t++]) ;
  return (int)len + 1;               // 全相等时 len 已减到 (uint)-1，+1 → 0
}

const SYMBOL *Lex_hash::get_hash_symbol(const char *s, unsigned int len) const {
  const char *cur_str = s;

  if (len == 0) { ...return nullptr; }
  if (len > entry_max_len) return nullptr;    // ★ 第一道剪枝：比最长符号还长，必不是关键字

  uint32 cur_struct = uint4korr(hash_map + ((len - 1) * 4));   // ★ 按长度定层

  for (;;) {
    uchar first_char = (uchar)cur_struct;     // 节点布局：[b0]=first_char [b1]=last_char [b2..3]=link

    if (first_char == 0) {                    // ★ 叶子
      uint16 ires = (uint16)(cur_struct >> 16);
      if (ires == array_elements(symbols)) return nullptr;   // 哨兵 = 符号总数
      const SYMBOL *res = symbols + ires;
      uint count = (uint)(cur_str - s);       // trie 已吃掉的字符数
      return lex_casecmp(cur_str, res->name + count, len - count) ? nullptr : res;
    }

    uchar cur_char = (uchar)to_upper_lex[(uchar)*cur_str];   // ★ 大小写折叠
    if (cur_char < first_char) return nullptr;
    cur_struct >>= 8;
    if (cur_char > (uchar)cur_struct) return nullptr;        // b1 = last_char
    cur_struct >>= 8;
    cur_struct = uint4korr(hash_map +
                           (((uint16)cur_struct + cur_char - first_char) * 4));  // ★ children[c - first_char]
    cur_str++;
  }
}
```

> ★ `to_upper_lex[]`（`sql_lex_hash.cc:49-73`）是一张 latin1 专用的大写化表（0x61-0x7A→0x41-0x5A，0xC0-0xFE→0xC0-0xDE）。**关键字的大小写不敏感就靠它**——不用 locale，所以是 ASCII/latin1 语义。

### 4.3 构建期生成（CMake）

```cmake
# sql/CMakeLists.txt:1371
# Gen_lex_hash
MYSQL_ADD_EXECUTABLE(gen_lex_hash gen_lex_hash.cc SKIP_INSTALL)

ADD_CUSTOM_COMMAND(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/lex_hash.h
  COMMAND gen_lex_hash > lex_hash.h     # ★ 靠 stdout 重定向 —— 所以 main() 里全是 printf
  DEPENDS gen_lex_hash
)
```

`main()` 依次输出三张表（`sql/gen_lex_hash.cc:305`）：

```cpp
int main(int, char **) {
  /* Broken up to indicate that it's not advice to you, gentle reader. */
  printf("/*\n\n  Do " "not " "edit " "this " "file " "directly!\n\n*/\n");
  puts(ORACLE_GPL_COPYRIGHT_NOTICE("2000"));

  /* Print sql_keywords_and_funcs_map[] and sql_keyword_and_funcs_max_len: */
  if (hash_map_info().print_hash_map("sql_keywords_and_funcs", SG_MAIN_PARSER)) exit(1);
  printf("\n");
  /* Print sql_keywords_map[] and sql_keywords_max_len values: */
  if (hash_map_info().print_hash_map("sql_keywords",
                                     SG_KEYWORDS | SG_HINTABLE_KEYWORDS)) exit(1);
  printf("\n");
  /* Print hint_keywords_map[] and hint_keywords_max_len values: */
  if (hash_map_info().print_hash_map("hint_keywords", SG_HINTS)) exit(1);
  exit(0);
}
```

同一 group 内**不允许重名**（trie 无法表达冲突），`check_duplicates()` 会直接 `exit(1)`：

```cpp
bool check_duplicates(uint group_mask) {
  std::set<const char *> names;
  ...
    if (!names.insert(cur->name).second) {
      printf("\ngen_lex_hash fatal error : "
             "Unfortunately gen_lex_hash can not generate a hash,\n since "
             "your lex.h has duplicate definition for a symbol \"%s\"\n\n", cur->name);
      return true;
    }
  ...
}
```

**本工作树构建产物实证**（`clion-build-950-debug/sql/lex_hash.h`，8758 行 / 148 KB）：

```cpp
static const unsigned char sql_keywords_and_funcs_map[17552]= {
'<', '>', 38, 0,      // 长度 1 层：first='<' last='>' link=38
'!', '|', 41, 0,      // 长度 2 层
...
};
const unsigned int sql_keywords_and_funcs_max_len=38;

static const unsigned char sql_keywords_map[16572]= { ... };
const unsigned int sql_keywords_max_len=38;

static const unsigned char hint_keywords_map[724]= {
0,   0,   73, 3,     // 长度 1 层没有 hint 关键字 → 空槽(0,0) + 哨兵 0x0349 = 841 = symbols 元素总数
'B', 'M', 29, 0,
...
};
const unsigned int hint_keywords_max_len=29;
```

> ★ 三张表大小差异说明一切：主符号表 17.5 KB / 关键字表 16.5 KB（差 1 KB 就是 35 个 `SYM_FN` 函数），hint 表只有 724 字节。

关键字列表的**权威来源是 `sql/lex.h` 的 `symbols[]`**；Bison 生成的 `sql_yacc.h` 里的 `%token` 只是另一份与它**保持同步**的声明（`SYM_OR_NULL(A)` 里的 `A` 就是那个 token 常量）。

---

## 五、YYSTYPE：语义值如何回传

词法分析产出的每个 token，除了 token 号，还要把**值**回传给 parser。

⚠️ **这里有两层 union，容易混**：词法器自己写的是 **`Lexer_yystype`**（窄接口，5 个成员），而 Bison 的语义值栈用的是 **`YYSType`**（`sql/parser_yystype.h:341`，238 个成员）。`MYSQLlex` 的签名收的是 `YYSTYPE *yacc_yylval`，第一件事就是 `reinterpret_cast` 成 `Lexer_yystype *`：

```cpp
int MYSQLlex(YYSTYPE *yacc_yylval, YYLTYPE *yylloc, THD *thd) {
  auto *yylval = reinterpret_cast<Lexer_yystype *>(yacc_yylval);
```

> ★ 这个 cast 成立的前提：`Lexer_yystype` 是 `YYSTYPE` 的**第一个成员**（`lexer`），见 [04 篇 §三](04_parser.md)。所以词法器只写 `YYSTYPE::lexer` 这一小块，语法动作再去读别的成员。

**`Lexer_yystype` 完整定义**（`sql/lexer_yystype.h:24`）：

```cpp
union Lexer_yystype {          // ★ 一次只用其中一个分支
  LEX_STRING lex_str;          // IDENT / TEXT_STRING / HEX_NUM / BIN_NUM / NCHAR_STRING / LEX_HOSTNAME
  LEX_SYMBOL keyword;          // 命中关键字：{symbol, str, length}，见 find_keyword() sql_lex.cc:913
  const CHARSET_INFO *charset; // UNDERSCORE_CHARSET（_utf8'..' 的引导符）
  PT_hint_list *optimizer_hints;  // hint 解析产物，挂在关键字 token 上带去语法分析
  LEX_CSTRING hint_string;     // hint 内部的裸字符串片段
};
```

**每个分支是谁填的**：

| 分支 | 填它的代码 | 对应 token |
|---|---|---|
| `lex_str` | `get_token()` / `get_quoted_token()` / `get_text()` | `IDENT` `IDENT_QUOTED` `TEXT_STRING` `NCHAR_STRING` `HEX_NUM` `BIN_NUM` `LEX_HOSTNAME` |
| `keyword` | `find_keyword()`（`:913-915`） | 所有关键字与运算符 |
| `charset` | `MY_LEX_IDENT` 的 `_` 分支（`:1549`） | `UNDERSCORE_CHARSET` |
| `optimizer_hints` | `consume_optimizer_hints()`（`:897`） | hintable 关键字 |

`YYSTYPE` 的布局约束（`static_assert(sizeof(YYSTYPE) <= 32)`，`parser_yystype.h:719`）与 [04 篇的语义值栈](04_parser.md) 直接相关——union 要能塞进栈槽位。`Lexer_yystype` 只有 5 个成员、最大也不过 24 字节（3 个指针），远小于这个上限。

---

## 六、字符集驱动的状态表

词法状态机**不是硬编码的**，而是由当前字符集 `CHARSET_INFO` 驱动的：

```cpp
const my_lex_states *state_map = cs->state_maps->main_map;  // 字符 → 初始状态
const uchar *ident_map = cs->ident_map;                     // 是否是标识符字符
```

- **`state_map[c]`**：输入字符 `c` 应该进入哪个状态（如字母→`IDENT`、数字→`NUMBER_IDENT`、`'`→`STRING`、空白→`SKIP`）
- **`ident_map[c]`**：字符 `c` 是否属于标识符的合法字符（决定 `a_b` 读到 `_` 时继续还是停止）

不同字符集（utf8mb4、gbk、latin1）有各自的表，所以同一个词法器能正确处理多字节字符：标识符扫描里对多字节字符走 `my_ismbchar` / `skip_binary`（`sql_lex.cc:1489-1501`）。

> ⚠️ `state_map` **不是** `sql_lex.cc` 里的全局数组——它是 `cs->state_maps->main_map`，**每个字符集一份**，由 `init_state_maps()` 在启动时填好（`mysys/sql_chars.cc:68`）。`lex_one_token` 里那行只是取个局部 const 指针。

#### `init_state_maps()` 完整源码（`mysys/sql_chars.cc:68-134`）

```cpp
bool init_state_maps(CHARSET_INFO *cs) {
  uint i;
  uchar *ident_map;
  enum my_lex_states *state_map = nullptr;

  lex_state_maps_st *lex_state_maps = (lex_state_maps_st *)my_once_alloc(
      sizeof(lex_state_maps_st), MYF(MY_WME));
  if (lex_state_maps == nullptr) return true;  // OOM

  cs->state_maps = lex_state_maps;
  state_map = lex_state_maps->main_map;

  if (!(cs->ident_map = ident_map = (uchar *)my_once_alloc(256, MYF(MY_WME))))
    return true;  // OOM

  hint_lex_init_maps(cs, lex_state_maps->hint_map);

  /* Fill state_map with states to get a faster parser */
  for (i = 0; i < 256; i++) {
    if (my_isalpha(cs, i))
      state_map[i] = MY_LEX_IDENT;
    else if (my_isdigit(cs, i))
      state_map[i] = MY_LEX_NUMBER_IDENT;
    else if (my_ismb1st(cs, i))
      /* To get whether it's a possible leading byte for a charset. */
      state_map[i] = MY_LEX_IDENT;          // ★ 多字节首字节也当标识符
    else if (my_isspace(cs, i))
      state_map[i] = MY_LEX_SKIP;
    else
      state_map[i] = MY_LEX_CHAR;           // ★ 兜底：运算符/括号/未知
  }
  state_map[(uchar)'_'] = state_map[(uchar)'$'] = MY_LEX_IDENT;
  state_map[(uchar)'\''] = MY_LEX_STRING;
  state_map[(uchar)'.'] = MY_LEX_REAL_OR_POINT;
  state_map[(uchar)'>'] = state_map[(uchar)'='] = state_map[(uchar)'!'] = MY_LEX_CMP_OP;
  state_map[(uchar)'<'] = MY_LEX_LONG_CMP_OP;   // ★ < 单独一类：<=> << <=
  state_map[(uchar)'&'] = state_map[(uchar)'|'] = MY_LEX_BOOL;
  state_map[(uchar)'#'] = MY_LEX_COMMENT;
  state_map[(uchar)';'] = MY_LEX_SEMICOLON;
  state_map[(uchar)':'] = MY_LEX_SET_VAR;
  state_map[0] = MY_LEX_EOL;                    // '\0' → 查询尾哨兵
  state_map[(uchar)'/'] = MY_LEX_LONG_COMMENT;  // 可能是 /* 或除法
  state_map[(uchar)'*'] = MY_LEX_END_LONG_COMMENT;
  state_map[(uchar)'@'] = MY_LEX_USER_END;
  state_map[(uchar)'`'] = MY_LEX_USER_VARIABLE_DELIMITER;   // ★★ 命名误导：这是反引号
  state_map[(uchar)'"'] = MY_LEX_STRING_OR_DELIMITER;

  /*
    Create a second map to make it faster to find identifiers
  */
  for (i = 0; i < 256; i++) {
    ident_map[i] = (uchar)(state_map[i] == MY_LEX_IDENT ||
                           state_map[i] == MY_LEX_NUMBER_IDENT);
  }

  /* Special handling of hex and binary strings */
  state_map[(uchar)'x'] = state_map[(uchar)'X'] = MY_LEX_IDENT_OR_HEX;
  state_map[(uchar)'b'] = state_map[(uchar)'B'] = MY_LEX_IDENT_OR_BIN;
  state_map[(uchar)'n'] = state_map[(uchar)'N'] = MY_LEX_IDENT_OR_NCHAR;

  /* Special handling for identifiers that start with dollar */
  state_map[(uchar)'$'] = MY_LEX_IDENT_OR_DOLLAR_QUOTE;

  return false;
}
```

> ★★ **注意 `x`/`b`/`n` 被改掉之后才算 `ident_map`**——所以 `ident_map['x'] == 0`。这正是 `MY_LEX_IDENT` 分支能用 `ident_map` 高速吞字符、而 `x'..'` 必须由 `MY_LEX_IDENT_OR_HEX` 单独特判的原因。

调用者（`sql/sql_lex.cc:162`）：

```cpp
bool lex_init() {
  DBUG_TRACE;
  for (CHARSET_INFO **cs = all_charsets;
       cs < all_charsets + array_elements(all_charsets) - 1; cs++) {
    if (*cs && (*cs)->ctype && is_supported_parser_charset(*cs)) {
      if (init_state_maps(*cs)) return true;  // OOM
    }
  }
  return false;
}
```

#### 字符类 → 状态 映射总表

| 字符 | 状态 | 产出 |
|---|---|---|
| `a-z A-Z`、多字节首字节 | `MY_LEX_IDENT` | `IDENT` / `IDENT_QUOTED` / 关键字 |
| `x`/`X` `b`/`B` `n`/`N` | `MY_LEX_IDENT_OR_HEX/BIN/NCHAR` | `HEX_NUM` `BIN_NUM` `NCHAR_STRING`（后非 `'` 则回落 IDENT） |
| `_` | `MY_LEX_IDENT` | `_utf8'..'` → `UNDERSCORE_CHARSET`，否则 IDENT |
| `$` | `MY_LEX_IDENT_OR_DOLLAR_QUOTE` | 废弃告警 + IDENT |
| `0-9` | `MY_LEX_NUMBER_IDENT` | `NUM`/`LONG_NUM`/`ULONGLONG_NUM`/`DECIMAL_NUM`/`FLOAT_NUM`/`HEX_NUM`/`BIN_NUM` |
| 空白 | `MY_LEX_SKIP` | 在 `MY_LEX_START` 吞掉；`\n` 时 `yylineno++` |
| `'` | `MY_LEX_STRING` | `TEXT_STRING` |
| `"` | `MY_LEX_STRING_OR_DELIMITER` | `ANSI_QUOTES` 时当标识符，否则字符串 |
| `` ` `` | `MY_LEX_USER_VARIABLE_DELIMITER` | `IDENT_QUOTED` |
| `.` | `MY_LEX_REAL_OR_POINT` | `.5` → `DECIMAL_NUM`；`.col` → `'.'` + IDENT |
| `>` `=` `!` | `MY_LEX_CMP_OP` | 最多再吃 1 个（`>=` `!=` `<>`） |
| `<` | `MY_LEX_LONG_CMP_OP` | 最多再吃 2 个（`<=` `<<` `<=>`） |
| `&` `\|` | `MY_LEX_BOOL` | 必须成对（`&&` `\|\|`），否则落回 `MY_LEX_CHAR` |
| `#` | `MY_LEX_COMMENT` | 行注释 |
| `/` | `MY_LEX_LONG_COMMENT` | `/*...*/` `/*!...*/`，或除法 |
| `*` | `MY_LEX_END_LONG_COMMENT` | 仅 `in_comment != NO_COMMENT` 时当注释尾，否则乘法 |
| `;` | `MY_LEX_SEMICOLON` | `';'` |
| `:` | `MY_LEX_SET_VAR` | `:=` 或 `':'` |
| `@` | `MY_LEX_USER_END` | 下一个字符决定 → `SYSTEM_VAR` / `HOSTNAME` / `START` |
| `\0` | `MY_LEX_EOL` | 首次 → `END_OF_INPUT`，再次 → `0`(YYEOF) |
| 其余 `, ( ) + - % ~ ^` 等 | `MY_LEX_CHAR` | `return (int)c` |
| 特判 `-` | `MY_LEX_CHAR` 内 | `-- `→注释、`->`→`JSON_SEPARATOR_SYM`、`->>`→`JSON_UNQUOTED_SEPARATOR_SYM` |
| 特判 `?` | `MY_LEX_CHAR` 内 | `PARAM_MARKER`（需 `stmt_prepare_mode` 且后非标识符） |

---

## 七、LALR(2) 特判：词法如何替语法兜底

MySQL 的语法是 LALR(1)（`%expect 63`，见 [04 篇](04_parser.md)），但有些构造需要 **LALR(2)**——即要看**两个** token 才能决定。最典型的是 `WITH ROLLUP`：

```sql
SELECT ... FROM t GROUP BY a WITH ROLLUP
SELECT ... FROM t WITH INDEX(idx)      -- 这里 WITH 是 table hint
```

`WITH` 后面跟 `ROLLUP` 还是 `INDEX`，语法层需要看两个 token，超出了 LALR(1)。MySQL 的解法是**把特判下沉到词法层**（`sql_lex.cc:1336-1350`，在 `MYSQLlex` 里）：

**真实源码见第一节**（`sql/sql_lex.cc:1336-1360`）：`MYSQLlex` 扫到 `WITH` 后**再调一次** `lex_one_token()`——若是 `ROLLUP_SYM` 就合并成 `WITH_ROLLUP_SYM`；否则把那个 token 存进 `lip->lookahead_token`，下次 `MYSQLlex` 直接回放。

```cpp
      token = lex_one_token(yylval, thd);        // ★★ 全词法器唯一"多读一个 token"的地方
      switch (token) {
        case ROLLUP_SYM:
          lip->add_digest_token(WITH_ROLLUP_SYM, yylval);
          return WITH_ROLLUP_SYM;                // 合并成一个 token
        default:
          lip->lookahead_yylval = lip->yylval;   // 存起来
          lip->lookahead_token = token;
          lip->add_digest_token(WITH, yylval);
          return WITH;                           // 下次 MYSQLlex 回放
      }
```

> ⚠️ **勘误**：早前版本说"这类 hack 遍布 `MYSQLlex` 末尾"——**不准确**。`MYSQLlex` 里**只有这一处**多读 token。其余"上下文相关"的判断（-→注释/JSON 符、`?`→占位符、`x'`→十六进制、`@@`→系统变量）都在 `lex_one_token` 内部用 `yyPeek`/`yyPeekn` 完成，属于**字符级** lookahead，不需要跨 token 预读。

**本质**：词法层把"两个 token"合并成"一个带上下文的 token"，让语法层回到 LALR(1) 可解。代价是 `WITH` 之后必须多扫一次，且引入 `lookahead_token` 这套回放机制（`Lex_input_stream::lookahead_token` / `lookahead_yylval`）。

> ★ 顺带解释一个细节：`reset()` 里 `lookahead_token = grammar_selector_token`。主语法时后者是 `-1`，于是 `MYSQLlex` 开头 `if (lip->lookahead_token >= 0)` 判不出来——**同一套回放机制被复用来投递 `GRAMMAR_SELECTOR_*` 那 5 个合成起始 token**（见 [04 篇 §5.1](04_parser.md)）。

---

## 核心调用栈

```
sql/sql_parse.cc:7142  thd->sql_parser()
 └─ sql/sql_class.cc:3067  MYSQLparse(THD*, Parse_tree_root**)
      └─ （Bison 生成的 sql_yacc.cc，需要 token 时回调）
          └─ sql/sql_lex.cc:1304  MYSQLlex(YYSTYPE*, YYLTYPE*, THD*)
               └─ sql/sql_lex.cc:1374  lex_one_token(yylval, thd)
                    ├─ :1394  MY_LEX_START   跳过空白
                    ├─ :1405  MY_LEX_CHAR    运算符/注释/单字符
                    ├─ :1469  MY_LEX_IDENT   标识符 → find_keyword(:1524)
                    ├─ :1570  MY_LEX_IDENT_SEP  a.b
                    ├─ :1638  MY_LEX_IDENT_START  .b
                    └─ ...    数字/字符串/注释状态
                    └─ 返回 token 号（SELECT_SYM / IDENT / NUM / ...）
```

---

## 相关的系统变量/状态变量

| 变量 | 类型 | 含义 | 对词法的影响 |
|------|------|------|-------------|
| `character_set_client` | 系统变量 | 客户端语句的字符集 | 决定 `cs->state_maps` / `ident_map`，即词法用哪张状态表 |
| `sql_mode`（`NO_BACKSLASH_ESCAPES`） | 系统变量 | 反斜杠是否作转义 | 影响字符串扫描里 `\` 的处理 |
| `sql_mode`（`ANSI_QUOTES`） | 系统变量 | 双引号是否当字符串引号 | 影响 `"..."` 是字符串还是标识符 |
| `lower_case_table_names` | 系统变量 | 表名大小写 | 影响标识符大小写归一化（词法层不直接做，但影响后续） |

> 词法层本身**没有**专门的状态变量，`Com_*` 计数在命令分发层（见 [01 篇](01_protocol_to_dispatch.md)）。

---

## Misc

- **`--` 注释 vs 减号**：MySQL 要求 `--` 后必须跟空白或控制字符才是注释（`sql_lex.cc:1407-1409`），这是标准 SQL 的兼容细节
- **`#` 注释**：`#` 到行尾也是注释（`MY_LEX_COMMENT` 处理），但仅在非 `ANSI` 模式
- **`/*! ... */` 版本注释**：`/*!40101 ... */` 是可执行注释，词法层 `MY_LEX_LONG_COMMENT` 会解析版本号，决定内部内容是否当 token 处理——这是"注释"里唯一能携带 SQL 的地方
- **JSON 运算符 `->` / `->>`**：8.0 新增，见状态 ② 的处理（`sql_lex.cc:1414-1423`）
- **`MYSQLlex` vs `lex_one_token`**：前者是 parser 回调入口（含 WITH ROLLUP 特判、预处理语句 hook），后者是"扫一个 token"的纯状态机

---

## 参考

- **Aho & Ullman《Principles of Compiler Design》（龙书）** —— 词法分析、状态机、正则文法
- *MySQL 8.0 Reference Manual → Lexical Structure（词法结构）*
- *MySQL 8.0 Reference Manual → Keywords and Reserved Words（关键字与保留字）*
