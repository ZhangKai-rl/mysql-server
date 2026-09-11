# 02 Lexer：从字符流到 token 流

> 本篇覆盖：词法分析如何把一段 SQL 文本切成 token 流（第 ① 层 IR 的"原料"），是 [`03 Parser`](04_parser.md) 的输入。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、MYSQLlex 入口](#一mysqllex-入口)
- [二、lex_one_token 状态机（逐状态剖析）](#二lex_one_token-状态机逐状态剖析)
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

```cpp
int MYSQLlex(YYSTYPE *yylval, YYLTYPE *yylloc, THD *thd) {
  Lex_input_stream *lip = &thd->m_parser_state->m_lip;   // :1306
  ...
  int token = lex_one_token(yylval, thd);                // :1332 单 token 扫描
  ...
  // WITH + ROLLUP 的 LALR(2)→LALR(1) 特判                // :1336-1350
  ...
  return token;                                          // 返回给 parser
}
```

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
      case MY_LEX_OPERATOR_OR_IDENT: ...
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

### 状态 ① MY_LEX_START：跳过空白

```cpp
case MY_LEX_START:  // Start of token
  while (state_map[c = lip->yyPeek()] == MY_LEX_SKIP) {
    if (c == '\n') lip->yylineno++;      // 记行号
    lip->yySkip();
  }
  lip->restart_token();
  c = lip->yyGet();
  state = state_map[c];                  // 查表得到下一个状态
  break;
```

每个 token 都从这里开始：跳过空白，读入第一个字符，用 `state_map` 查表决定走哪个分支。

### 状态 ② MY_LEX_CHAR：单字符 token / 运算符 / 注释

```cpp
case MY_LEX_CHAR:
  if (c == '-' && lip->yyPeek() == '-' &&
      (my_isspace(cs, lip->yyPeekn(1)) || my_iscntrl(cs, lip->yyPeekn(1)))) {
    state = MY_LEX_COMMENT;              // '-- ' 是注释
    break;
  }
  if (c == '-' && lip->yyPeek() == '>') { // '->' / '->>'
    lip->yySkip();
    if (lip->yyPeek() == '>') { lip->yySkip(); return JSON_UNQUOTED_SEPARATOR_SYM; }
    return JSON_SEPARATOR_SYM;
  }
  if (c == '?' && lip->stmt_prepare_mode && !ident_map[lip->yyPeek()])
    return PARAM_MARKER;                 // 预编译语句的 '?' 占位符
  return (int)c;                         // 单字符 token（如 '(' '+' ','）
```

**这里体现了"上下文相关"**：同一个 `-`，后面是 `- `（`--` + 空白）就是注释，后面是 `>` 就是 JSON 运算符，否则是减号。单字符 token 直接返回 ASCII 码作为 token 号。

### 状态 ③ MY_LEX_IDENT：标识符 / 关键字

```cpp
case MY_LEX_IDENT:
  ...   // 循环读入，直到遇到非标识符字符
  if (lip->yyPeek() == '.') ...          // 处理 a.b 的表.列
  else {
    lip->yyUnget();
    if ((tokval = find_keyword(lip, length, c == '('))) {  // ★ 关键字判定
      lip->next_state = MY_LEX_START;
      return tokval;                     // 是关键字，返回关键字 token 号
    }
  }
  yylval->lex_str = get_token(lip, 0, length);  // 是普通标识符，拷贝字符串
  ...
  return IDENT;                          // 返回 IDENT（通用标识符）
```

**关键点**：标识符读完后，先用 `find_keyword` 查它是不是关键字（见第四节）。是 → 返回 `SELECT_SYM`/`FROM_SYM` 等；否 → 返回通用 `IDENT`，把字符串放进 `yylval->lex_str`。

### 状态 ④⑤ MY_LEX_IDENT_SEP / MY_LEX_IDENT_START：`a.b` / `.b`

处理 `db.table.col` 这种点号分隔的限定名：读完 `a` 遇到 `.`，转入 `IDENT_SEP` 期待下一个标识符；若以 `.` 开头（`.col`），转入 `IDENT_START`。这两个状态让词法能正确切出 `a`、`.`、`b` 三个 token（而非把 `a.b` 当一个 token）。

### 数字状态

`MY_LEX_INT_OR_REAL` / `MY_LEX_REAL_OR_POINT` 处理整数、小数、科学计数法（`1e5`）、`.5` 这种以点开头的实数。数字最终返回 `NUM`，值放进 `yylval->dec_lex_str`。

### 字符串状态

`MY_LEX_STRING` 处理 `'...'`、`"..."`、以及 `N'...'`（`IDENT_OR_NCHAR`）、`_utf8'...'`（`IDENT_OR_HEX`/`IDENT_OR_BIN`）等带引导符的字符串，返回 `TEXT_STRING`/`NCHAR_STRING` 等。

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

**关键成员**：
- `m_ptr`：当前读指针
- `m_end_of_query`：查询末尾
- `m_echo` / `m_cpp_ptr`：回显模式——把解析过的字符同时拷进 `m_cpp_ptr`（用于 general log / 预处理语句的规范化 SQL）
- `m_yylval`：当前 token 的语义值指针

**`yyPeek` + `yyUnget` 是 lookahead 的基石**：状态机经常"偷看下一个字符决定走哪个状态，看完再回退"，这是 LALR(2) 特判和运算符识别的实现手段。

`init()`（`:3372`）在每次 `lex_start` 时被调用，把新的 SQL 文本装进缓冲。

---

## 四、find_keyword：关键字识别

标识符读完后，`find_keyword`（`sql/sql_lex.cc`）判断它是不是关键字。关键字表由 `gen_lex_hash` 构建脚本**离线生成**（`sql/gen_lex_hash.cc` → 生成哈希表），编译期固化，运行时 O(1) 哈希查找。

```cpp
// sql/sql_lex.cc:1524（MY_LEX_IDENT 内）
if ((tokval = find_keyword(lip, length, c == '('))) {
  lip->next_state = MY_LEX_START;
  return tokval;        // SELECT_SYM / FROM_SYM / ...
}
```

**第三个参数 `c == '('` 很关键**：MySQL 允许函数名和关键字冲突（如 `LEFT` 既是函数名又是关键字）。传 `c == '('` 表示"标识符后面紧跟左括号"，`find_keyword` 据此区分：`LEFT(` 是函数调用（返回 `IDENT`），`LEFT` 单独是关键字（返回 `LEFT_SYM`）。

**关键字分三类**（`lex.h` 里的 `%token` 声明决定）：
- **保留字**（reserved）：如 `SELECT`/`FROM`/`WHERE`，不能当标识符
- **非保留关键字**：如 `MASTER`/`LOCK`，可当标识符
- **函数名关键字**：`LEFT`/`RIGHT` 等，特殊处理

关键字列表在 `sql/lex.h`（Bison 生成 `sql_yacc.h` 里的 `%token`）。

---

## 五、YYSTYPE：语义值如何回传

词法分析产出的每个 token，除了 token 号，还要把**值**回传给 parser。值的载体是 `YYSTYPE` union（`sql/parser_yystype.h:340`）：

```cpp
union YYSTYPE {
  ...
  LEX_STRING lex_str;           // 标识符 / 字符串
  const char *keyword;          // 关键字
  int num;                      // 数字
  PT_* / Item * / ...           // 语法动作产出的节点（见 04 篇）
};
```

`lex_one_token` 里 `yylval->lex_str = get_token(...)`、`yylval->charset = ...` 就是在填这个 union。`YYSTYPE` 的布局约束（`static_assert(sizeof(YYSTYPE) <= 32)`，`parser_yystype.h:718`）与 [03 篇的语义值栈](04_parser.md) 直接相关——union 要能塞进栈槽位。

---

## 六、字符集驱动的状态表

词法状态机**不是硬编码的**，而是由当前字符集 `CHARSET_INFO` 驱动的：

```cpp
const my_lex_states *state_map = cs->state_maps->main_map;  // 字符 → 初始状态
const uchar *ident_map = cs->ident_map;                     // 是否是标识符字符
```

- **`state_map[c]`**：输入字符 `c` 应该进入哪个状态（如字母→`IDENT`、数字→`NUMBER`、`'`→`STRING`、空白→`SKIP`）
- **`ident_map[c]`**：字符 `c` 是否属于标识符的合法字符（决定 `a_b` 读到 `_` 时继续还是停止）

不同字符集（utf8mb4、gbk、latin1）有各自的表，所以同一个词法器能正确处理多字节字符：标识符扫描里对多字节字符走 `my_ismbchar` / `skip_binary`（`sql_lex.cc:1489-1501`）。

---

## 七、LALR(2) 特判：词法如何替语法兜底

MySQL 的语法是 LALR(1)（`%expect 63`，见 [04 篇](04_parser.md)），但有些构造需要 **LALR(2)**——即要看**两个** token 才能决定。最典型的是 `WITH ROLLUP`：

```sql
SELECT ... FROM t GROUP BY a WITH ROLLUP
SELECT ... FROM t WITH INDEX(idx)      -- 这里 WITH 是 table hint
```

`WITH` 后面跟 `ROLLUP` 还是 `INDEX`，语法层需要看两个 token，超出了 LALR(1)。MySQL 的解法是**把特判下沉到词法层**（`sql_lex.cc:1336-1350`，在 `MYSQLlex` 里）：

```cpp
// MYSQLlex 内：扫到 WITH 后偷看下一个 token
if (token == WITH_SYM) {
  // 偷看下一个 token，若是 ROLLUP_SYM，则改成特殊的 WITH_ROLLUP_SYM
  ...
}
```

**本质**：词法层用 `yyPeek` 做 lookahead，把"两个 token"合并成"一个带上下文的 token"，让语法层回到 LALR(1) 可解。这类 hack 遍布 `MYSQLlex` 末尾，是 MySQL 几十年演进在词法层留下的痕迹。

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
