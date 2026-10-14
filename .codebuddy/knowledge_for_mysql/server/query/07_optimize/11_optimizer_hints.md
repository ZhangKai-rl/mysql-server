# 11 Optimizer Hints：`/*+ ... */` 干预优化器

> 标准 MySQL 5.7.7+ 的官方 hint 语法（`/*+ ... */`），用于给优化器提供现成的优化决策、缩小计划搜索空间。本篇讲它的完整管线：**独立 hint 语法器 → PT_hint_list 语法树 → Opt_hints 四层对象树 → 优化期按类型分发**。
>
> **边界**：hint 是"干预优化器"的机制，是横切主题，不属四阶段任何单一阶段；它最终作用到 06~08 篇的 join order / 访问方法 / range 决策点上。
>
> ⚠️ 与 Statement Outline（阿里云 RDS/PolarDB 私有特性）区分：Statement Outline 是把已知好计划**固化并强制复现**，标准 MySQL 8.0.39 没有；本文讲的是标准 MySQL 的 `/*+ */` hint。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、架构总览：双语法器 + 四层对象树](#一架构总览双语法器--四层对象树)
- [二、hint 的语法与解析](#二hint-的语法与解析)
- [三、Opt_hints 四层对象树](#三opt_hints-四层对象树)
- [四、hint 如何映射到优化器（三类）](#四hint-如何映射到优化器三类)
- [五、JOIN_ORDER 如何强制连接顺序](#五join_order-如何强制连接顺序)
- [六、INDEX 如何限制候选索引](#六index-如何限制候选索引)
- [七、SET_VAR / MAX_EXECUTION_TIME / RESOURCE_GROUP](#七set_var--max_execution_time--resource_group)
- [八、8.0.39 已废弃或未生效的 hint](#八8039-已废弃或未生效的-hint)
- [**★ 8.0.39 hint 消费点全景表**](#-8039-hint-消费点全景表哪里真的读了它)
- [**★ 真名勘误全表**](#真名勘误全表)

---

## 设计思想与理论基础

### hint 解决什么问题

优化器基于代价与统计信息决策，但三类情况下会选错：

1. **统计信息过时**（直方图缺失、数据倾斜）
2. **代价模型失真**（01 篇）
3. **优化器本身能力有限**（搜索空间被剪枝、用了启发式）

hint 就是让用户**手动覆盖优化器决策**的逃生舱。

### 为什么用独立的小语法器（双语法器）

8.0.39 的 hint 语法**不在主语法器 `sql_yacc.yy` 里**，而在独立的小 Bison 语法器 `sql/sql_hints.yy`。主词法器识别到 hintable 关键字（SELECT/INSERT/UPDATE/DELETE/REPLACE）后紧跟 `/*+` 时才唤醒它。

**为什么这么设计**：

1. **注释语法不适合塞进主语法器**——hit 写在注释里，主语法器本该当注释跳过；只有特定位置才激活旁路解析
2. **错误容忍**——hint 写错**不应该让整条 SQL 失败**（SQL 标准精神：hint 是"建议"）。独立语法器可以局部报错 + 产生 warning，不污染主语法器的错误恢复
3. **避免语法冲突**——hint 语法独立演进，不会与主语法产生 LALR 冲突

> 这是理解 MySQL hint 的关键：**它不是主语法的一部分，而是挂在注释上的旁路解析**。

### 四层对象树：为什么是 global → query block → table → key

hint 需要作用在不同粒度的对象上，所以自然形成四层：

| 层 | 作用对象 | hint 例子 |
|---|---|---|
| global | 整个语句 | `MAX_EXECUTION_TIME`、`SET_VAR`、`RESOURCE_GROUP` |
| query block | 子查询块 | `JOIN_ORDER`、`SEMIJOIN`、`SUBQUERY` |
| table | 某张表 | `NO_MERGE`、`BNL`、`MERGE` |
| key | 某个索引 | `INDEX`、`NO_INDEX`、`JOIN_INDEX` |

查找时由外向内、由粗到细（`hint_key_state` 先查 key 级，再 fallback 到 table 级）。

> 这种"分层对象树 + 按粒度查找"是 hint 系统的**通用设计**——Oracle 的 hint 体系、PostgreSQL 的 `pg_hint_plan` 插件都是同构的。

### 理论定位：hint 是"规则"对"代价"的覆盖

呼应 00 篇的 CBO/RBO 划分：

- 优化器正常路径是 **CBO**（代价驱动）
- hint 是**用户手写的规则**，直接覆盖代价决策

所以 hint 本质上是一个 **RBO 旁路**——允许人在优化器不够聪明时手工下指令。

这也解释了为什么 hint 用多了是"坏味道"：它把决策权从优化器还给了人，而**人的判断会随数据变化而过期**（数据时变了，硬编码的 hint 还在）。

### 代价与限制

- **hint 不保证生效**——优化器可能因语义限制无法满足。这是"建议"语义的必然结果，也是它被设计成 warning 而非 error 的原因
- 8.0.39 有一批已废弃或未生效的 hint（本篇第八章）
- 与 prepared statement 的交互：hint 在 **prepare 时**解析，之后不再变（本篇第七章）

---

## 一、架构总览：双语法器 + 四层对象树

**核心结论**：8.0.39 的 Optimizer Hints 是"**双语法器 + 四层对象树 + 优化期按类型分发**"的架构：

1. 主语法器 `sql_yacc.yy` 里**没有** hint 产生式；hint 的产生式在**独立的小 Bison 语法器** `sql/sql_hints.yy`。
2. 主词法器 `sql_lex.cc` 识别到 hintable 关键字（SELECT/INSERT/UPDATE/DELETE/REPLACE）后，若紧跟 `/*+`，就调小语法器 `HINT_PARSER_parse` 把注释解析成 `PT_hint_list`。
3. `PT_hint_list`/`PT_hint` 在名字解析阶段（`contextualize()`）转成**四层 `Opt_hints` 对象树**（global → query block → table → key）。
4. 优化阶段通过 `hint_key_state`、`hint_table_state`、`apply_join_order_hints`、`update_index_hint_maps` 读取这棵树。

> ⚠️ 常见误解纠正：网上资料常说的 `Opt_hints::update_parse_time_state`、`PT_hint::make_opt_hint` **在 8.0.39 不存在**。对应职责分别是各 `PT_*_hint::contextualize()` 与 `Opt_hints*::set_switch`/`register_child`。

---

## 二、hint 的语法与解析

### 2.1 主词法器识别（sql_lex.cc:905）

```cpp
// sql/sql_lex.cc:905（find_keyword 内）
lip->yylval->optimizer_hints = nullptr;
if (symbol->group & SG_HINTABLE_KEYWORDS) {   // SELECT/INSERT/UPDATE/DELETE/REPLACE
  lip->add_digest_token(symbol->tok, lip->yylval);
  if (consume_optimizer_hints(lip)) return ABORT_SYM;
  lip->skip_digest = true;
}
```

`consume_optimizer_hints`（`:865`）做"探测 + 调小语法器"：

```cpp
// 跳过空白，若接下来三字节是 / * +，则确认是 hint 注释
if (lip->yyPeekn(whitespace) == '/' && lip->yyPeekn(whitespace + 1) == '*' &&
    lip->yyPeekn(whitespace + 2) == '+') {
  Hint_scanner hint_scanner(lip->m_thd, ...);
  PT_hint_list *hint_list = nullptr;
  int rc = HINT_PARSER_parse(lip->m_thd, &hint_scanner, &hint_list);  // 小语法器
  ...
  lip->yylval->optimizer_hints = hint_list;   // 写回主 token 语义值
  return false;
}
```

**算法**：关键字后紧跟 `/*+` → 构造 `Hint_scanner`，调 `sql_hints.yy` 生成的 `HINT_PARSER_parse` 解析注释体 → 结果 `hint_list` 存进 `yylval->optimizer_hints`，主语法器再把它挂到 `PT_query_specification`/`PT_update`/`PT_delete`/`PT_insert`。

### 2.2 hint 的独立词法器：`Hint_scanner`（第二套词法器）

上节讲的是"主词法器怎么**发现** hint"。发现之后，注释体由**另一套词法器** `Hint_scanner` 扫描——这就是"双词法器"的另一半（`sql/sql_lex_hints.cc`）。

```cpp
/// Lexical scanner for hint comments.
///
/// When the main lexical scanner recognizes the "/*+" delimiter, it calls
/// the hint parser (HINT_PARSER_parse) to consume the rest of hint tokens
/// including the */ delimiter. The hint parser uses Hint_scanner as its own
/// lexer to scan hint-specific tokens.
class Hint_scanner {
```

它是**手写状态机**（与 `sql_lex.cc` 的 `MY_LEX_*` 状态机同性质，都不是 flex）：

```cpp
int Hint_scanner::scan() {
  int whitespaces = 0;
  for (;;) {
    start_token();
    switch (peek_class()) {
      case HINT_CHR_NL:        skip_newline(); whitespaces++; continue;
      case HINT_CHR_SPACE:     skip_byte();    whitespaces++; continue;
      case HINT_CHR_DIGIT:     return scan_number_or_multiplier_or_ident();
      case HINT_CHR_IDENT:     return scan_ident_or_keyword();
      case HINT_CHR_QUOTE:     return scan_quoted<HINT_CHR_QUOTE>();
      case HINT_CHR_ASTERISK:
        if (peek_class2() == HINT_CHR_SLASH) {
          ptr += 2;  // skip "*/"
          input_buf_end = ptr;
          return HINT_CLOSE;
        } else return get_byte();
      case HINT_CHR_AT:
        if (prev_token == '(' ||
            (prev_token == HINT_ARG_IDENT && whitespaces == 0))
          return scan_query_block_name();
        else return get_byte();
      ...
    }
  }
}
```

两个设计点：

1. **扫描器自我封闭**：遇到 `*/` 时除返回 `HINT_CLOSE` 外还执行 `input_buf_end = ptr`——把自己的输入边界**收缩到 `*/` 处**，物理上不可能越过注释去读主 SQL 文本
2. **空白计数有语义**：局部 `whitespaces` 只用于消歧 `@`——`t@qb`（紧贴）才是 query block 限定名，`t @qb`（有空格）里的 `@` 只是普通字符。这是手写扫描器才能做的上下文判定

**★ `Hint_scanner` 的完整成员**（⚠️ 真身在 **`sql/sql_lex_hints.h:57`**，不在 `sql_lex.h`）：

```cpp
class Hint_scanner {
  THD *thd;
  const CHARSET_INFO *cs;                            /* 客户端字符集，决定多字节判定 */
  const bool is_ansi_quotes;                         /* thd->variables.sql_mode & MODE_ANSI_QUOTES */
  size_t lineno;
  const hint_lex_char_classes *char_classes;         /* ★ 指向 256 项表 cs->state_maps->hint_map */

  const char *input_buf;
  const char *input_buf_end;
  const char *ptr;
  int prev_token;                                    /* 上一个 token，用于 '@' 与 digest 判定 */
  sql_digest_state *digest_state;                    /* 借来的 digest buffer */

 public:
  const char *raw_yytext;   /* ★ 当前 token 的「原文指针」，用于报错文本（未做字符集转换） */
  const char *yytext;       /* 当前 token 指针（可能已转换、已是长生命周期） */
  size_t yyleng;
  bool has_hints;           /* ★ 该 hint 注释是否非空 —— 决定 digest 里是否包 OPEN/CLOSE */

 public:
  Hint_scanner(THD *thd, size_t lineno_arg, const char *buf, size_t len,
               sql_digest_state *digest_state_arg);
  size_t get_lineno() const { return lineno; }
  const char *get_ptr() const { return ptr; }
  sql_digest_state *get_digest() { return digest_state; }
  void syntax_warning(const char *msg) const;

  int get_next_token() {          /* ★ 公开入口：bison 的 yylex 包装 */
    prev_token = scan();
    add_hint_token_digest();      /* ★ 每个 token 扫完立刻进 digest */
    return prev_token;
  }

 protected:
  int scan();                     /* 真正的状态机主循环 */
  template <hint_lex_char_classes Quote> int scan_quoted();
  int scan_ident();
  int scan_multiplier_or_ident();          /* SET_VAR(x=10K/M/G) 的后缀 */
  int scan_query_block_name();             /* @qb_name */
  int scan_ident_or_keyword();             /* ★ 关键字 vs 标识符的分流点 */
  int scan_number_or_multiplier_or_ident();
  int scan_fraction_digits();

  bool eof() const { return ptr >= input_buf_end; }
  char peek_byte() const { return *ptr; }
  /* ★ 全部字符判定都过这张 256 项表 */
  hint_lex_char_classes peek_class() const {
    return eof() ? HINT_CHR_EOF : char_classes[static_cast<uchar>(peek_byte())];
  }
  hint_lex_char_classes peek_class2() const {   /* 2 字符前瞻：识别 "*/" 与 '' 双写 */
    return ptr + 1 >= input_buf_end ? HINT_CHR_EOF
                                    : char_classes[static_cast<uchar>(ptr[1])];
  }
  void skip_newline() { skip_byte(); lineno++; }
  void skip_byte() { yyleng++; ptr++; }
  bool skip_mb() { ... my_ismbchar(cs, ptr, input_buf_end) ... }
  void adjust_token() { yytext = ptr; yyleng = 0; }
  void start_token() { adjust_token(); raw_yytext = ptr; }
  void add_hint_token_digest();

 private:
  void add_digest(uint token) {
    if (digest_state == nullptr) return;  // Digest buffer is full.
    Lexer_yystype fake_yylvalue;
    fake_yylvalue.lex_str.str = const_cast<char *>(yytext);
    fake_yylvalue.lex_str.length = yyleng;
    digest_state = digest_add_token(digest_state, token, &fake_yylvalue);
  }
};

/* ★★ 两套词法器的接缝 */
inline int HINT_PARSER_lex(YYSTYPE *yacc_yylval, Hint_scanner *scanner) {
  auto yylval = reinterpret_cast<Lexer_yystype *>(yacc_yylval);
  int ret = scanner->get_next_token();
  yylval->hint_string.str = scanner->yytext;
  yylval->hint_string.length = scanner->yyleng;
  return ret;
}
```

```cpp
/* sql/sql_lex_hints.cc:44 —— 构造即跳过 "/*+" */
Hint_scanner::Hint_scanner(THD *thd_arg, size_t lineno_arg, const char *buf,
                           size_t len, sql_digest_state *digest_state_arg)
    : thd(thd_arg),
      cs(thd->charset()),
      is_ansi_quotes(thd->variables.sql_mode & MODE_ANSI_QUOTES),
      lineno(lineno_arg),
      char_classes(cs->state_maps->hint_map),        /* ★ 挂上 256 项表 */
      input_buf(buf),
      input_buf_end(input_buf + len),
      ptr(input_buf + 3),  // skip "/*+"             /* ★★ 构造即跳过 "/*+" */
      prev_token(0),
      digest_state(digest_state_arg),
      raw_yytext(ptr), yytext(ptr), yyleng(0), has_hints(false) {}
```

> ★★ **`Hint_scanner` 不被 LEX 持有**——它是 `consume_optimizer_hints()` 的**栈上局部对象**，只持有 `THD*`；产物 `PT_hint_list*` 经 `lip->yylval` 回传，最终落到 `LEX::opt_hints_global`。

**★ 关键字 vs 标识符的分流**（`scan_ident_or_keyword()`，★ 完美哈希查 hint 关键字表）：

```cpp
  int scan_ident_or_keyword() {
    for (;;) {
      switch (peek_class()) {
        case HINT_CHR_IDENT:
        case HINT_CHR_DIGIT:  skip_byte(); continue;
        case HINT_CHR_MB:     return scan_ident();
        case HINT_CHR_EOF:
        default:
          const SYMBOL *symbol =
              Lex_hash::hint_keywords.get_hash_symbol(yytext, yyleng);   /* ★ 第二套哈希 */
          if (symbol) {                                                  /* 是 hint 关键字 */
            /* Override the yytext pointer to the short-living buffer with a
            long-living pointer to the same text (don't need to allocate a
            keyword string since symbol array is a global constant). */
            yytext = symbol->name;
            assert(yyleng == symbol->length);
            return symbol->tok;                                          /* 返回 1000+ 的 token */
          }
          yytext = thd->strmake(yytext, yyleng);
          return HINT_ARG_IDENT;                                         /* 是普通标识符 */
      }
    }
  }
```

> ★ `Lex_hash::hint_keywords` 是**独立生成**的完美哈希表（`sql_lex_hash.cc:41`），与 `sql_keywords` 分开。`hint_keywords_map[]` 由 `gen_lex_hash.cc` 在**构建期**生成，源码树里没有字面量。

#### ★ 它有自己的一张 256 项字符分类表

```cpp
struct lex_state_maps_st {
  enum my_lex_states main_map[256];          // 主词法器
  enum hint_lex_char_classes hint_map[256];  // hint 扫描器
};
```

两张表在**同一次初始化里并排生成**（`init_state_maps()` 里调 `hint_lex_init_maps`）。hint 表只有 13 个类别，而主表有 34 个 `MY_LEX_*` 状态并带大量**语义化**覆盖。逐项差异（这是理解"为什么必须换表"的关键）：

| 字符 | 主 `main_map` | hint `hint_map` | 后果 |
|---|---|---|---|
| `;` | `MY_LEX_SEMICOLON`（触发多语句切分） | `HINT_CHR_CHAR` | hint 注释里的 `;` **不会**切分语句 |
| `#` | `MY_LEX_COMMENT`（行注释） | `HINT_CHR_CHAR` | hint 内**不支持** `#` 注释 |
| `/` | `MY_LEX_LONG_COMMENT` | `HINT_CHR_SLASH`（switch 里无此分支→落 default） | hint 内**不支持嵌套 `/* */`**，也不支持 `/*! ... */` |
| `*` | `MY_LEX_END_LONG_COMMENT` | 只在紧跟 `/` 时返回 `HINT_CLOSE` | 只有 `*/` 有语义 |
| `:` | `MY_LEX_SET_VAR`（识别 `:=`） | 无覆盖 | hint 里 `:=` 被当成两个单字符 token |
| `x/b/n`、`$` | `MY_LEX_IDENT_OR_HEX/BIN/NCHAR/DOLLAR_QUOTE` | 纯 `HINT_CHR_IDENT` | hint 里没有 hex/bin 字面量 |
| `"` | `MY_LEX_STRING_OR_DELIMITER` | `HINT_CHR_DOUBLEQUOTE` + **自己重判** ANSI_QUOTES | 见下 |

`"` 的语义重解释在 hint 侧独立实现（`is_ansi_quotes` 成员，构造时取 `thd->variables.sql_mode & MODE_ANSI_QUOTES`）：

```cpp
const bool is_ident = (Quote == HINT_CHR_BACKQUOTE) ||
                      (is_ansi_quotes && Quote == HINT_CHR_DOUBLEQUOTE);
const int ret = is_ident ? HINT_ARG_IDENT : HINT_ARG_TEXT;
```

> ⚠️ 顺带澄清：hint 扫描器**没有** `_digits` / `_warn_level` 这类表（源码 0 命中），数字解析走 `my_strtoll10()`。

**★ 那 256 项表是「运行时生成」的，不是数组字面量**（⚠️ 不存在 `hint_lex_char_map` 这个名字；真身是 `cs->state_maps->hint_map[256]`）：

```cpp
/* mysys/sql_chars.cc:40 —— 这就是"那张表"：一个循环，不是字面量 */
static void hint_lex_init_maps(CHARSET_INFO *cs,
                               enum hint_lex_char_classes *hint_map) {
  size_t i;
  for (i = 0; i < 256; i++) {
    if (my_ismb1st(cs, i))        hint_map[i] = HINT_CHR_MB;
    else if (my_isalpha(cs, i))   hint_map[i] = HINT_CHR_IDENT;
    else if (my_isdigit(cs, i))   hint_map[i] = HINT_CHR_DIGIT;
    else if (my_isspace(cs, i)) { assert(!my_ismb1st(cs, i));
                                  hint_map[i] = HINT_CHR_SPACE; }
    else                          hint_map[i] = HINT_CHR_CHAR;
  }
  hint_map[u'*']  = HINT_CHR_ASTERISK;
  hint_map[u'@']  = HINT_CHR_AT;
  hint_map[u'`']  = HINT_CHR_BACKQUOTE;
  hint_map[u'.']  = HINT_CHR_DOT;
  hint_map[u'"']  = HINT_CHR_DOUBLEQUOTE;
  hint_map[u'$']  = HINT_CHR_IDENT;      /* ★ '$' 被显式提升为标识符字符 */
  hint_map[u'_']  = HINT_CHR_IDENT;      /* ★ '_' 同上 */
  hint_map[u'\n'] = HINT_CHR_NL;         /* ★ '\n' 单独成类，为了行数统计 */
  hint_map[u'\''] = HINT_CHR_QUOTE;
  hint_map[u'/']  = HINT_CHR_SLASH;
}
```

```cpp
/* include/sql_chars.h:74 —— 13 个类别（对比主表的 34 个 MY_LEX_* 状态） */
enum MY_ATTRIBUTE((__packed__)) hint_lex_char_classes {
  HINT_CHR_ASTERISK,     // [*]
  HINT_CHR_AT,           // [@]
  HINT_CHR_BACKQUOTE,    // [`]
  HINT_CHR_CHAR,         // default state
  HINT_CHR_DIGIT,        // [[:digit:]]
  HINT_CHR_DOT,          // '.'
  HINT_CHR_DOUBLEQUOTE,  // ["]
  HINT_CHR_EOF,          // pseudo-class
  HINT_CHR_IDENT,        // [_$[:alpha:]]
  HINT_CHR_MB,           // multibyte character
  HINT_CHR_NL,           // \n
  HINT_CHR_QUOTE,        // [']
  HINT_CHR_SLASH,        // [/]
  HINT_CHR_SPACE         // [[:space:]] excluding \n
};
```

> ⚠️ `sql/sql_lex_hints.h:49` 里声明的 `void hint_lex_init_maps(CHARSET_INFO *, hint_lex_char_classes *)` 与 `mysys/sql_chars.cc:40` 的 `static` 定义**不是同一个实体**——这是一个历史遗留的重复声明，实际生效的是后者。

#### ★ 切换协议：`consume_optimizer_hints()`

**★ 全文**（⚠️ 它是 `sql/sql_lex.cc:865` 的 **static 自由函数**，不是 LEX 成员）：

```cpp
static bool consume_optimizer_hints(Lex_input_stream *lip) {
  const my_lex_states *state_map = lip->query_charset->state_maps->main_map;
  int whitespace = 0;
  uchar c = lip->yyPeek();
  size_t newlines = 0;

  /* ① 借"主"词法器的状态表跳空白 */
  for (; state_map[c] == MY_LEX_SKIP; whitespace++, c = lip->yyPeekn(whitespace)) {
    if (c == '\n') newlines++;
  }

  /* ② 三字符前瞻：必须恰为 "/*+" */
  if (lip->yyPeekn(whitespace) == '/' && lip->yyPeekn(whitespace + 1) == '*' &&
      lip->yyPeekn(whitespace + 2) == '+') {
    lip->yylineno += newlines;
    lip->yySkipn(whitespace);

    /* ③ 栈上小词法器；digest buffer 被"借"给它 */
    Hint_scanner hint_scanner(lip->m_thd, lip->yylineno, lip->get_ptr(),
                              lip->get_end_of_query() - lip->get_ptr(),
                              lip->m_digest);
    PT_hint_list *hint_list = nullptr;
    int rc = HINT_PARSER_parse(lip->m_thd, &hint_scanner, &hint_list);   /* ★ 调独立小语法器 */
    if (rc == 2) return true;  // Bison's internal OOM error
    if (rc == 1) {
      /* 1. YYABORT in the hint parser grammar (OOM), 2. open commentary error */
      lip->start_token();  // adjust error message text pointer to "/*+"
      return true;         /* → 调用者返回 ABORT_SYM */
    }
    /* ④ 三件事同步回主词法器 */
    lip->yylineno = hint_scanner.get_lineno();                    // 行号
    lip->yySkipn(hint_scanner.get_ptr() - lip->get_ptr());        // ★ 游标吞掉整个 "/*+ ... */"
    lip->yylval->optimizer_hints = hint_list;  // NULL in case of syntax error
    lip->m_digest = hint_scanner.get_digest(); // NULL is digest buf. is full.
    return false;
  } else
    return false;                             /* 不是 hint 注释 → 照常 */
}
```

> ★★ **交接契约四条**：① 主词法器只借出 `m_digest` 指针，hint 词法器写完后通过 `get_digest()` 还回来；② 主词法器用 `yySkipn(get_ptr() - get_ptr())` 一次性吞掉小词法器消费的全部字节；③ 结果 `PT_hint_list*` 塞进**关键字 token 自己的 yylval**（不是独立 token 流）；④ `lip->skip_digest = true`。

切换点不在注释状态机，而在"hintable 关键字"识别处——只有命中 `SG_HINTABLE_KEYWORDS`（仅 `DELETE/INSERT/REPLACE/SELECT/UPDATE` 五个）后才尝试吃 hint：

```cpp
    if (symbol->group & SG_HINTABLE_KEYWORDS) {
      lip->add_digest_token(symbol->tok, lip->yylval);   // ① 先写 SELECT
      if (consume_optimizer_hints(lip)) return ABORT_SYM; // ② 再吃 hint
      lip->skip_digest = true;                            // ③ 抑制外层的重复写入
    }
```

`consume_optimizer_hints()` 的三段：

- **切前**：用主表的 `MY_LEX_SKIP` 跳空白；前瞻 3 字节必须恰为 `/ * +`，否则**不切换**（`/*+` 退化成普通注释）
- **切中**：主词法器把自己的 `lip->m_digest` **指针**交给 `Hint_scanner`，两者**共享同一个 digest buffer**；剩余整段文本交给它
- **切后**：三件事同步回来——行号、位置（`yySkipn`）、digest 指针（`lip->m_digest = hint_scanner.get_digest()`，digest 满时为 NULL）

#### ★ `skip_digest` 的完整机制

源码注释把它讲得很透（这段代码值得原样引用）：

```cpp
  /// Skip adding of the current token's digest since it is already added
  ///
  /// Usually we calculate a digest token by token at the top-level function
  /// of the lexer: MYSQLlex(). However, some complex ("hintable") tokens break
  /// that data flow: for example, the `SELECT /*+ HINT(t) */` is the single
  /// token from the main parser's point of view, and we add the "SELECT"
  /// keyword to the digest buffer right after the lex_one_token() call,
  /// but the "/*+ HINT(t) */" is a sequence of separate tokens from the hint
  /// parser's point of view, and we add those tokens to the digest buffer
  /// *inside* the lex_one_token() call. Thus, the usual data flow adds
  /// tokens from the "/*+ HINT(t) */" string first, and only than it appends
  /// the "SELECT" keyword token to that stream: "/*+ HINT(t) */ SELECT".
  /// This is not acceptable, since we use the digest buffer to restore
  /// query strings in their normalized forms, so the order of added tokens is
  /// important. Thus, we add tokens of "hintable" keywords to a digest buffer
  /// right in the hint parser and skip adding of them at the caller with the
  /// help of skip_digest flag.
  bool skip_digest;
```

即 `skip_digest` **同时解决两个问题**：顺序颠倒（`/*+...*/ SELECT` 错序）与重复写入。消费点在 `MYSQLlex()` 尾部：

```cpp
  if (!lip->skip_digest) lip->add_digest_token(token, yylval);
  lip->skip_digest = false;   // 一次性
```

#### hint token 如何进 digest

`add_hint_token_digest()` 的归一化规则：数字一律压成 `NUM`（不论 `1000` / `1.5` / `1M`）、标识符压成 `IDENT`、字符串压成 `TEXT_STRING`、**关键字保留自己的 token 号**。

所以 digest 能区分 `/*+ BKA(t) */` 与 `/*+ NO_BKA(t) */`，但**对字面值不敏感**。

注释首尾用两个构建期生成的伪 token：`TOK_HINT_COMMENT_OPEN`（打印 `/*+`）/ `TOK_HINT_COMMENT_CLOSE`（打印 `*/`）——它们是 `gen_lex_token.cc` 在 PART 6 里分配的 fake token（与 digest 篇讲的 `TOK_GENERIC_VALUE` 同族）。

> ⚠️ digest 满时**静默降级**：`if (digest_state == nullptr) return;`

**★ `add_hint_token_digest()` 全文**（⚠️ 真名不是 `add_hint_token_to_digest`）：

```cpp
void Hint_scanner::add_hint_token_digest() {
  if (digest_state == nullptr) return;  // can't add: digest buffer is full
  if (prev_token == 0 || prev_token == HINT_ERROR) return;

  if (prev_token == HINT_CLOSE) {
    if (has_hints) add_digest(TOK_HINT_COMMENT_CLOSE);   /* ★ 只有非空 hint 注释才补 CLOSE */
    return;
  }
  if (!has_hints) {  // the 1st hint in the comment
    add_digest(TOK_HINT_COMMENT_OPEN);                   /* ★ hint 注释被包在 OPEN/CLOSE 伪 token 之间 */
    has_hints = true;
  }

  switch (prev_token) {
    case HINT_ARG_NUMBER:
    case HINT_ARG_FLOATING_POINT_NUMBER:   add_digest(NUM); break;
    case HINT_ARG_IDENT:
      add_digest((peek_class() == HINT_CHR_AT) ? TOK_IDENT_AT : IDENT);  /* ★ 前瞻 '@' 区分 "t" 与 "t@qb" */
      break;
    case HINT_ARG_QB_NAME:  add_digest('@'); add_digest(IDENT); break;
    case HINT_ARG_TEXT:     add_digest(TEXT_STRING); break;
    case HINT_IDENT_OR_NUMBER_WITH_SCALE:  add_digest(NUM); break;
    default:
      if (prev_token <= UCHAR_MAX)         // Single-char token.
        add_digest(prev_token);            /* ( ) , = 等直接以字节值入 digest */
      else {                               // keyword
        /* Make sure this is a known hint keyword: */
        switch (prev_token) {              /* ★ debug 期白名单断言 */
          case BKA_HINT: case BNL_HINT: case DUPSWEEDOUT_HINT:
          case FIRSTMATCH_HINT: case INTOEXISTS_HINT: case LOOSESCAN_HINT:
          case MATERIALIZATION_HINT: case MAX_EXECUTION_TIME_HINT: case MRR_HINT:
          case NO_BKA_HINT: case NO_BNL_HINT: case NO_ICP_HINT: case NO_MRR_HINT:
          case NO_RANGE_OPTIMIZATION_HINT: case NO_SEMIJOIN_HINT: case QB_NAME_HINT:
          case SEMIJOIN_HINT: case SET_VAR_HINT: case SUBQUERY_HINT:
          case DERIVED_MERGE_HINT: case NO_DERIVED_MERGE_HINT:
          case JOIN_PREFIX_HINT: case JOIN_SUFFIX_HINT: case JOIN_ORDER_HINT:
          case JOIN_FIXED_ORDER_HINT: case INDEX_MERGE_HINT: case NO_INDEX_MERGE_HINT:
          case RESOURCE_GROUP_HINT: case SKIP_SCAN_HINT: case NO_SKIP_SCAN_HINT:
          case HASH_JOIN_HINT: case NO_HASH_JOIN_HINT:
          case INDEX_HINT: case NO_INDEX_HINT: case JOIN_INDEX_HINT:
          case NO_JOIN_INDEX_HINT: case GROUP_INDEX_HINT: case NO_GROUP_INDEX_HINT:
          case ORDER_INDEX_HINT: case NO_ORDER_INDEX_HINT:
          case DERIVED_CONDITION_PUSHDOWN_HINT: case NO_DERIVED_CONDITION_PUSHDOWN_HINT:
            break;
          default:
            assert(false);                 /* ★ 非白名单关键字入 digest = bug */
        }
        add_digest(prev_token);
      }
  }
}
```

> ★★ **这段白名单是"8.0.39 全部对外可见 hint"的权威清单之一**（共 41 个 token）。注意它包含 `MATERIALIZATION` / `FIRSTMATCH` / `LOOSESCAN` / `DUPSWEEDOUT` / `INTOEXISTS`——这些是**策略参数关键字**，不是独立 hint。

#### 错误是**警告**，不是错误

```cpp
void HINT_PARSER_error(THD *thd, Hint_scanner *scanner, PT_hint_list **,
                       const char *msg) {
  if (strcmp(msg, "syntax error") == 0)
    msg = ER_THD(thd, ER_WARN_OPTIMIZER_HINT_SYNTAX_ERROR);
  scanner->syntax_warning(msg);
}
```

级别是 `SL_WARNING`。而且语法里有 error 恢复产生式，出错后跳到 `*/` 继续，**已解析出的 hint 仍然生效**：

```
start:
          hint_list HINT_CLOSE          { *ret= $1; }
        | hint_list error HINT_CLOSE    { *ret= $1; }
        | error HINT_CLOSE              { *ret= NULL; }
```

真正让语句失败的只有 OOM（`rc == 2`）和未闭合注释（`rc == 1`）。

#### ★ 为什么必须两套词法器（7 条证据）

不只是"语法冲突"一句话，而是**对同一批字符的词法解释规则整体不同**：

1. **token 编号空间物理隔离**：`sql_hints.yy` 从 1000 起编号；bison 前缀也不同（`HINT_PARSER_*` vs `MYSQL*parse`）
2. **关键字集合物理隔离**：`SG_MAIN_PARSER` **不含** `SG_HINTS`；`MATERIALIZATION`/`FIRSTMATCH`/`MRR` 等在 hint 里是关键字、在主语法里是普通标识符。共用一张 hash 表无法表达
3. **数字形态不同**：hint 里额外支持 `1M`（`K/M/G` 后缀）与浮点——主词法器的 `MY_LEX_INT_OR_REAL` 等状态机表达不了
4. **`SET_VAR` 用 `=` 不是 `:=`**（grammar 只有 `'='`），且右侧只允许字面量（主语法 `SET` 的真子集）
5. **引号语义需重解释**（`"` 在 ANSI_QUOTES 下是标识符）
6. **注释/分隔符边界不同**：hint 里 `;` 不能终止语句、`#`/`--` 不是注释、`/* */` 不嵌套——**若复用主词法器，hint 文本里的 `;` 会被切成多语句，`#` 会吞掉后面所有内容**。这是必须换表的最强理由
7. **digest 需要在特定位置插入伪 token**（`TOK_HINT_COMMENT_OPEN` 等），只有独立扫描器才能在正确时机插入

**总结**：MySQL 的做法是「两张 256 项分类表 + 两套 bison 语法（各自 token 空间、各自关键字 hash）+ 一个共享的 `sql_digest_state` buffer + 一个切换点（`consume_optimizer_hints`）+ 一个顺序修正位（`skip_digest`）」。

### 2.3 小语法器 sql_hints.yy 的结构

```yacc
hint_list: hint | hint_list hint ;
hint:
      index_level_hint      /* INDEX/NO_INDEX 等带索引参数 */
    | table_level_hint      /* BKA/BNL/NO_ICP 等表级 */
    | qb_level_hint         /* SEMIJOIN/SUBQUERY/JOIN_ORDER 等 */
    | qb_name_hint          /* QB_NAME(x) */
    | max_execution_time_hint
    | set_var_hint
    | resource_group_hint
```

语义策略参数（`SEMIJOIN(strategy)` 的参数被 OR 成一个位图）：

```yacc
semijoin_strategy:
      FIRSTMATCH_HINT      { $$ = OPTIMIZER_SWITCH_FIRSTMATCH; }
    | LOOSESCAN_HINT       { $$ = OPTIMIZER_SWITCH_LOOSE_SCAN; }
    | MATERIALIZATION_HINT { $$ = OPTIMIZER_SWITCH_MATERIALIZATION; }
    | DUPSWEEDOUT_HINT     { $$ = OPTIMIZER_SWITCH_DUPSWEEDOUT; }
    ;
semijoin_strategies:
          %empty                { $$ = 0; }
        | semijoin_strategy     { $$ = $1; }
        | semijoin_strategies ',' semijoin_strategy { $$ = $1 | $3; }  /* ★ 按位或 */
        ;
```

**★ prologue 的三条关键设定**：

```yacc
%define api.pure         /* ★ 可重入 parser，无全局变量 */
%parse-param { class THD *thd }
%parse-param { class Hint_scanner *scanner }
%parse-param { class PT_hint_list **ret }
%lex-param   { class Hint_scanner *scanner }
%expect 0                /* ★★ 断言 0 冲突：hint 语法必须是 LALR(1) 无歧义 */

#define NEW_PTN new (thd->mem_root)   /* ★★ 与主语法器的 new(YYMEM_ROOT) 不同 —— hint 用自己的 mem_root */
```

> ⚠️ **`sql_hints.yy` 里没有 `MYSQL_YYABORT`**（该宏只定义在 `sql_yacc.yy:206`），全程裸用 `YYABORT` / `YYERROR`。`MYSQL_YYACCEPT` 全仓不存在。
> ⚠️ **`sql/sql_hints.yy.cc` 与 `sql/sql_hints.yy.h` 都不在源码树里**——它们是 bison 构建期生成的，但头文件被 `#include`。

**★ token 号从 1000 起**（★ 与主语法器的 token 空间隔离，保证 digest 稳定）：

```yacc
%token MAX_EXECUTION_TIME_HINT 1000
%token RESOURCE_GROUP_HINT 1001
%token BKA_HINT 1002   %token BNL_HINT 1003   %token DUPSWEEDOUT_HINT 1004
%token FIRSTMATCH_HINT 1005  %token INTOEXISTS_HINT 1006 %token LOOSESCAN_HINT 1007
%token MATERIALIZATION_HINT 1008 %token NO_BKA_HINT 1009 %token NO_BNL_HINT 1010
%token NO_ICP_HINT 1011  %token NO_MRR_HINT 1012
%token NO_RANGE_OPTIMIZATION_HINT 1013 %token NO_SEMIJOIN_HINT 1014
%token MRR_HINT 1015   %token QB_NAME_HINT 1016  %token SEMIJOIN_HINT 1017
%token SUBQUERY_HINT 1018 %token DERIVED_MERGE_HINT 1019 %token NO_DERIVED_MERGE_HINT 1020
%token JOIN_PREFIX_HINT 1021 %token JOIN_SUFFIX_HINT 1022 %token JOIN_ORDER_HINT 1023
%token JOIN_FIXED_ORDER_HINT 1024 %token INDEX_MERGE_HINT 1025 %token NO_INDEX_MERGE_HINT 1026
%token SET_VAR_HINT 1027 %token SKIP_SCAN_HINT 1028 %token NO_SKIP_SCAN_HINT 1029
%token HASH_JOIN_HINT 1030 %token NO_HASH_JOIN_HINT 1031
/* 实参 token */
%token HINT_ARG_NUMBER 1032 %token HINT_ARG_IDENT 1033 %token HINT_ARG_QB_NAME 1034
%token HINT_ARG_TEXT 1035 %token HINT_IDENT_OR_NUMBER_WITH_SCALE 1036
%token HINT_CLOSE 1037 %token HINT_ERROR 1038
/* 8.0 新增的 index 级 */
%token INDEX_HINT 1039 %token NO_INDEX_HINT 1040 %token JOIN_INDEX_HINT 1041
%token NO_JOIN_INDEX_HINT 1042 %token GROUP_INDEX_HINT 1043 %token NO_GROUP_INDEX_HINT 1044
%token ORDER_INDEX_HINT 1045 %token NO_ORDER_INDEX_HINT 1046
%token DERIVED_CONDITION_PUSHDOWN_HINT 1047 %token NO_DERIVED_CONDITION_PUSHDOWN_HINT 1048
%token HINT_ARG_FLOATING_POINT_NUMBER 1049
%token YYUNDEF 1150
```

> ★ 注释原文要求："*Please add new tokens right above this line. To make DIGESTS stable, it is desirable to avoid changing token number values.*"

**★★ QB 限定的两种写法**（★ 这是 hint 语法里最容易错的细节）：

```yacc
hint_param_table:        HINT_ARG_IDENT opt_qb_name   /* ★ 后缀式：BKA(t1@qb1) */
hint_param_table_ext:    hint_param_table
                       | HINT_ARG_QB_NAME HINT_ARG_IDENT   /* ★★ 前缀式：INDEX(@qb1 t1 idx) */
```

- **表级 hint** 用 `hint_param_table_list_empty_qb`（后缀式或 QB 前置）：`BKA(t1@qb1)` / `BKA(@qb1 t1, t2)`
- **索引级 hint** 用 `hint_param_table_ext`（**只有它支持 QB 前置**）：`INDEX(@qb1 t1 idx)`
- 归一逻辑在 `PT_table_level_hint::contextualize()`：
```cpp
    const LEX_CSTRING *qb_name_str =
        qb_name.length > 0 ? &qb_name : &table_name->opt_query_block;
```

**★ `SET_VAR` 的特殊处理**：

```yacc
set_var_ident:
          HINT_ARG_IDENT
        | MAX_EXECUTION_TIME_HINT    /* ★★ 特例：允许 SET_VAR(MAX_EXECUTION_TIME=...) —— 关键字名做变量名 */
        ;
set_var_num_item:
          HINT_ARG_NUMBER                    { ... Item_int(n) ... }
        | HINT_ARG_FLOATING_POINT_NUMBER     { $$ = NEW_PTN Item_float(...); }
        | HINT_IDENT_OR_NUMBER_WITH_SCALE    /* ★ 10K/10M/10G：去掉末位再解析 */
          { switch ($1.str[$1.length - 1]) {
              case 'K': multiplier = 1024; break;
              case 'M': multiplier = 1024 * 1024; break;
              case 'G': multiplier = 1024 * 1024 * 1024; break; } }
```

### 2.4 hint 能出现的位置

- **SELECT / UPDATE / DELETE / INSERT / REPLACE**：关键字后紧跟 `/*+`（`sql_yacc.yy:1988` 声明这些 token 携带 hint 列表）
- **子查询**：每个 `query_specification` 自带 `opt_hints`，子查询内也可写 hint
- **`@qb_name` 跨查询块定位**：表级/键级 hint 的 `table_name@qb_name` 语法允许外层 hint 指定作用到哪个查询块

---

## 三、Opt_hints 四层对象树

### 3.1 四层结构（opt_hints.h:149）

```
Opt_hints_global                      语句级（一个 LEX 一个）
   └── Opt_hints_qb[]                 查询块级（每个 Query_block 一个）
          └── Opt_hints_table[]       表级
                 └── Opt_hints_key[]  索引级
```

`Opt_hints` 基类（`opt_hints.h:162`）核心成员：

```cpp
class Opt_hints {
  const LEX_CSTRING *name;              // 空=global, QB名=qb级, 表名=表级, 索引名=key级
  Opt_hints *parent;                    // 父节点
  Opt_hints_map hints_map;              // hint 状态位图
  Mem_root_array<Opt_hints *> child_array;
  bool resolved;                        // 是否已绑定到真实对象
};
```

- `Opt_hints_global`（:350）：额外持 `max_exec_time`、`sys_var_hint`
- `Opt_hints_qb`（:372）：额外持 `select_number`、`subquery_hint`/`semijoin_hint`、`join_order_hints` 数组
- `Opt_hints_table`（:549）：额外持 `keyinfo_array`、6 个 compound key hint
- `Opt_hints_key`（:641）：仅重写 `append_name`

**★ 基类的两个关键方法**（★ `set_switch` 的返回值就是"冲突信号"）：

```cpp
  /* @return true if hint is already specified, false otherwise */
  bool set_switch(bool switch_state_arg, opt_hints_enum type_arg,
                  bool check_parent) {
    if (is_specified(type_arg) ||
        (check_parent && parent->is_specified(type_arg)))
      return true;                       /* ★★ 返回 true = 冲突 → 调用方发 ER_WARN_CONFLICTING_HINT */
    hints_map.set_switch(type_arg, switch_state_arg);
    return false;
  }
```

```cpp
/* opt_hints.cc:116 —— ★★ get_switch 是"沿父链向上回溯"的核心 */
bool Opt_hints::get_switch(opt_hints_enum type_arg) const {
  if (is_specified(type_arg)) return hints_map.switch_on(type_arg);

  if (opt_hint_info[type_arg].check_upper_lvl)
    return parent->get_switch(type_arg);   /* ★ 只有 check_upper_lvl=true 的才向上找 */

  return false;
}

/* ★ 按名字查子节点：O(n) 线性扫，没有哈希 */
Opt_hints *Opt_hints::find_by_name(const LEX_CSTRING *name_arg,
                                   const CHARSET_INFO *cs) const {
  for (uint i = 0; i < child_array.size(); i++) {
    const LEX_CSTRING *name = child_array[i]->get_print_name();
    if (!cmp_lex_string(*name, *name_arg, cs)) return child_array[i];
  }
  return nullptr;
}
```

**★★ 四层树的挂接过程**（`parse_tree_hints.cc`，★ 解析期建树，全用 `thd->mem_root`）：

```cpp
static Opt_hints_global *get_global_hints(Parse_context *pc) {
  LEX *lex = pc->thd->lex;
  if (!lex->opt_hints_global)                       /* ★ 第 1 层：挂在 LEX 上，懒创建 */
    lex->opt_hints_global =
        new (pc->thd->mem_root) Opt_hints_global(pc->thd->mem_root);
  if (lex->opt_hints_global) lex->opt_hints_global->set_resolved();
  return lex->opt_hints_global;
}

static Opt_hints_qb *get_qb_hints(Parse_context *pc, Query_block *select) {
  if (select->opt_hints_qb) return select->opt_hints_qb;   /* ★ 第 2 层：挂在 Query_block 上 */
  Opt_hints_global *global_hints = get_global_hints(pc);
  if (global_hints == nullptr) return nullptr;
  Opt_hints_qb *qb = new (pc->thd->mem_root)
      Opt_hints_qb(global_hints, pc->thd->mem_root, select->select_number);
  if (qb) {
    global_hints->register_child(qb);      /* ★ global -> qb */
    select->opt_hints_qb = qb;
    qb->set_resolved();
  }
  return qb;
}

static Opt_hints_table *get_table_hints(Parse_context *pc,
                                        Hint_param_table *table_name,
                                        Opt_hints_qb *qb) {
  Opt_hints_table *tab = static_cast<Opt_hints_table *>(
      qb->find_by_name(&table_name->table, table_alias_charset));  /* ★ 表名用 table_alias_charset（大小写敏感） */
  if (!tab) {
    tab = new (pc->thd->mem_root)
        Opt_hints_table(&table_name->table, qb, pc->thd->mem_root);  /* ★ 第 3 层 */
    qb->register_child(tab);
  }
  return tab;
}
/* 第 4 层 Opt_hints_key 在 PT_key_level_hint::contextualize() 里 new + tab->register_child(key) */
```

> ★★ **`get_table_hints()` 不 `set_resolved()`**——表是否真的存在要到 resolve 期 `adjust_key_hints()` 才知道。这是"未解析 hint 报 `ER_UNRESOLVED_HINT_NAME`"的机制基础。
> ⚠️ 真名是 `register_child()` / `find_by_name()` / `get_complex_hints()`；**不存在 `register_hint()` / `get_hints()` / `find_hint()` / `set_hint()`**。

**★ 系统 QB 名**（`select#N`，★ 解析视图体时用它回填）：

```cpp
Opt_hints_qb::Opt_hints_qb(Opt_hints *opt_hints_arg, MEM_ROOT *mem_root_arg,
                           uint select_number_arg)
    : Opt_hints(nullptr, opt_hints_arg, mem_root_arg),
      select_number(select_number_arg), ... {
  sys_name.str = buff;
  sys_name.length =
      snprintf(buff, sizeof(buff), "%s%x", sys_qb_prefix.str, select_number);
}                                     /* ★ %x 是十六进制！sys_qb_prefix = "select#" */
```

```cpp
/* find_qb_hints()：两种定位方式 */
  if (qb_name->length == 0) return pc->select->opt_hints_qb;   /* ① 没写 @qb → 当前 QB */
  Opt_hints_qb *qb = static_cast<Opt_hints_qb *>(
      pc->thd->lex->opt_hints_global->find_by_name(qb_name, system_charset_info)); /* ② 按 QB_NAME 找 */
  if (!qb) { /* ③ 退而按 "select#N" 系统名找（解析视图 body 需要） */
    for (Query_block *select = lex->all_query_blocks_list; select != nullptr;
         select = select->next_select_in_list()) {
      sys_name.length = snprintf(buff, sizeof(buff), "%s%x", "select#",
                                 select->select_number);
      if (!cmp_lex_string(sys_name, *qb_name, system_charset_info)) {
        qb = get_qb_hints(pc, select); break;
      }
    }
  }
  if (qb == nullptr)
    hint->print_warn(pc->thd, ER_WARN_UNKNOWN_QB_NAME, qb_name, nullptr, nullptr, nullptr);
```

### 3.2 hint 枚举与元信息（opt_hints.h:64）

```cpp
enum opt_hints_enum {
  BKA_HINT_ENUM, BNL_HINT_ENUM, ICP_HINT_ENUM, MRR_HINT_ENUM,
  NO_RANGE_HINT_ENUM, MAX_EXEC_TIME_HINT_ENUM, QB_NAME_HINT_ENUM,
  SEMIJOIN_HINT_ENUM, SUBQUERY_HINT_ENUM, DERIVED_MERGE_HINT_ENUM,
  JOIN_PREFIX_HINT_ENUM, JOIN_SUFFIX_HINT_ENUM, JOIN_ORDER_HINT_ENUM,
  JOIN_FIXED_ORDER_HINT_ENUM, INDEX_MERGE_HINT_ENUM, RESOURCE_GROUP_HINT_ENUM,
  SKIP_SCAN_HINT_ENUM, HASH_JOIN_HINT_ENUM, INDEX_HINT_ENUM,
  JOIN_INDEX_HINT_ENUM, GROUP_INDEX_HINT_ENUM, ORDER_INDEX_HINT_ENUM,
  DERIVED_CONDITION_PUSHDOWN_HINT_ENUM, MAX_HINT_ENUM
};
```

每个枚举对应 `opt_hint_info[]`（`opt_hints.cc:65`）一条元信息，三个字段：`check_upper_lvl`（可否指定在多级）、`switch_hint`（开关型 vs 复杂型）、`irregular_hint`（打印特殊）。

**★ `opt_hint_info[]` 全文**（⚠️ **`opt_hints_enum.h` 这个文件不存在**——枚举在 `opt_hints.h:64`，元信息数组在 `opt_hints.cc:65`）：

```cpp
struct st_opt_hint_info {
  const char *hint_name;  // Hint name.
  bool check_upper_lvl;   // true if upper level hint check is needed (for hints
                          // which can be specified on more than one level).
  bool switch_hint;       // true if hint is not complex.
  bool irregular_hint;    ///< true if hint requires some special handling.
                          ///< Currently it's used only for join order hints
                          ///< since they need special printing procedure.
};

struct st_opt_hint_info opt_hint_info[] = {
    {"BKA",                        true,  true,  false},
    {"BNL",                        true,  true,  false},
    {"ICP",                        true,  true,  false},
    {"MRR",                        true,  true,  false},
    {"NO_RANGE_OPTIMIZATION",      true,  true,  false},
    {"MAX_EXECUTION_TIME",         false, false, false},
    {"QB_NAME",                    false, false, false},
    {"SEMIJOIN",                   false, false, false},
    {"SUBQUERY",                   false, false, false},
    {"MERGE",                      true,  true,  false},   /* ★ 打印名是 MERGE，枚举叫 DERIVED_MERGE */
    {"JOIN_PREFIX",                false, false, true},    /* ★ irregular：需特殊打印 */
    {"JOIN_SUFFIX",                false, false, true},
    {"JOIN_ORDER",                 false, false, true},
    {"JOIN_FIXED_ORDER",           false, true,  false},
    {"INDEX_MERGE",                false, false, false},
    {"RESOURCE_GROUP",             false, false, false},
    {"SKIP_SCAN",                  false, false, false},
    {"HASH_JOIN",                  true,  true,  false},   /* ★ 有名字有语法，但无消费点 */
    {"INDEX",                      false, false, false},
    {"JOIN_INDEX",                 false, false, false},
    {"GROUP_INDEX",                false, false, false},
    {"ORDER_INDEX",                false, false, false},
    {"DERIVED_CONDITION_PUSHDOWN", true,  true,  false},
    {nullptr, false, false, false}};
```

> ★★ **"NO_" 前缀不是存在名字里的，是打印时补的**：`append_hint_type()` 里 `if (!hints_map.switch_on(type)) str->append("NO_")`。所以 `NO_BKA` 与 `BKA` 共用同一条 `opt_hint_info`，靠 `hints` 位区分。
> ★ `irregular_hint = true` 的**只有 3 个 JOIN_ORDER 系**——它们不走通用 `print()` 路径，由 `print_irregular_hints()` 单独处理（`print()` 开头就 `continue` 跳过）。
> ★ `Bitmap<64>` 的 64 位容量恰好容纳 23 个真实 hint + 哨兵。

### 3.3 `Opt_hints_map`（值位图 + 指定位图）

```cpp
class Opt_hints_map {
  Bitmap<64> hints;            // hint state（开关值）
  Bitmap<64> hints_specified;  // 是否已指定
};
```

`set_switch` 同时写两个位图：`hints` 写开关值、`hints_specified` 标记"已指定"。区分"指定为 off"与"未指定"——后者要回落到 `optimizer_switch`。

```cpp
class Opt_hints_map {
  Bitmap<64> hints;            // hint state
  Bitmap<64> hints_specified;  // true if hint is specified
 public:
  bool is_specified(opt_hints_enum type_arg) const {
    return hints_specified.is_set(type_arg);
  }
  void set_switch(opt_hints_enum type_arg, bool switch_state_arg) {
    if (switch_state_arg) hints.set_bit(type_arg);
    else                  hints.clear_bit(type_arg);
    hints_specified.set_bit(type_arg);      /* ★★ 无论如何都置"已指定" */
  }
  bool switch_on(opt_hints_enum type_arg) const {
    return hints.is_set(type_arg);
  }
};
```

> ★ **`Opt_hints_map` 只有 3 个方法**。外层的 `Opt_hints::get_switch()` 才是带"向上回溯"的版本，别混淆。

---

## 四、hint 如何映射到优化器（三类）

### 4.1 开关型（NO_ICP / NO_MRR / BKA / BNL / MERGE）

核心读取 `Opt_hints::get_switch`（`opt_hints.cc:116`）：

```cpp
bool Opt_hints::get_switch(opt_hints_enum type_arg) const {
  if (is_specified(type_arg)) return hints_map.switch_on(type_arg);
  if (opt_hint_info[type_arg].check_upper_lvl) return parent->get_switch(type_arg);  // 查父级
  return false;
}
```

对外封装（`opt_hints.cc:891`）：

```cpp
bool hint_key_state(const THD *thd, const Table_ref *table, uint keyno,
                    opt_hints_enum type_arg, uint optimizer_switch) {
  ...  // hint 已指定则覆盖，否则回落到 optimizer_switch_flag
}
```

**"hint 优先、开关兜底"**。消费实例：

- NO_ICP → `sql_select.cc:2985` `hint_key_state(..., ICP_HINT_ENUM, OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN)`
- NO_MRR → `handler.cc:6558` `hint_key_state(..., MRR_HINT_ENUM, OPTIMIZER_SWITCH_MRR)`
- NO_BNL → `sql_planner.cc:1004` `hint_table_state(..., BNL_HINT_ENUM, OPTIMIZER_SWITCH_BNL)`
- NO_MERGE → `sql_resolver.cc:3501` `hint_table_state(..., DERIVED_MERGE_HINT_ENUM, ...)`

### 4.2 枚举型（SEMIJOIN / SUBQUERY）

`Opt_hints_qb::semijoin_enabled`（`opt_hints.cc:239`）：

```cpp
bool Opt_hints_qb::semijoin_enabled(const THD *thd) const {
  if (subquery_hint) return false;                    // SUBQUERY hint 禁用半连接
  if (semijoin_hint) {
    if (semijoin_hint->switch_on()) return true;      // SEMIJOIN(...) 强制开
    if (semijoin_hint->get_args() == 0) return false; // NO_SEMIJOIN() 强制关
  }
  return thd->optimizer_switch_flag(OPTIMIZER_SWITCH_SEMIJOIN);
}
```

`sj_enabled_strategies`（:256）处理策略子集：`SEMIJOIN(a,b)` 只启用 a/b 两种策略；`NO_SEMIJOIN(a)` 从开关里剔除 a。

`subquery_strategy`（:270）：`SUBQUERY(MATERIALIZATION)` → `SUBQ_MATERIALIZATION`，`SUBQUERY(INTOEXISTS)` → `SUBQ_EXISTS`。消费点在 `Query_block::subquery_strategy`（`sql_lex.cc:4525`，见 02 篇 4.2 节）。

### 4.3 复杂型（JOIN_ORDER / INDEX）

见第五、六节。

---

## 五、JOIN_ORDER 如何强制连接顺序

### 5.1 JOIN_ORDER → 表依赖（set_join_hint_deps，opt_hints.cc:459）

```cpp
for (const Hint_param_table *hint_table = hint_table_list->begin();
     hint_table < hint_table_list->end(); hint_table++) {
  for (uint i = 0; i < join->tables; i++) {
    const Table_ref *table = join->join_tab[i].table_ref;
    if (!compare_table_name(hint_table, table)) {
      if (join->const_table_map & table->map()) break;   // const 表不参与
      JOIN_TAB *tab = &join->join_tab[i];
      tab->dependent |= hint_tab_map;        // ★ 表 i 依赖前面所有 hint 表
      hint_tab_map |= tab->table_ref->map();
      break;
    }
  }
}
```

**核心算法**：把 `JOIN_ORDER(t1,t2,t3)` 线性化成链式依赖——`t2 依赖 t1`、`t3 依赖 t1|t2`。第 2 张 hint 表依赖第 1 张，第 3 张依赖前两张，以此类推。

非 hint 表的处理（`get_other_dep`，:351）：

| hint | 非 hint 表的依赖 |
|------|------------------|
| `JOIN_PREFIX(t1,t2)` | 非 hint 表依赖全部 hint 表（hint 表必须在最前） |
| `JOIN_SUFFIX(t1,t2)` | hint 表依赖其他所有表（hint 表必须在最后） |
| `JOIN_ORDER(t1,t2)` | 非 hint 表不额外加依赖（可自由穿插） |

最后 `propagate_dependencies`（`sql_optimizer.cc:5556`）求传递闭包 + 检测环（自依赖 = 成环）。

**★ `set_join_hint_deps()` 全文**（⚠️ 它是 **static 自由函数**，不是 `Opt_hints_qb` 的成员；成员函数是 `apply_join_order_hints()`）：

```cpp
static bool set_join_hint_deps(JOIN *join,
                               const Hint_param_table_list *hint_table_list,
                               opt_hints_enum type) {
  Join_order_hint_handler hint_handler(join);   /* ★ RAII：构造时快照，析构时回滚 */
  table_map hint_tab_map = 0;
  if (hint_handler.init()) return true;

  for (const Hint_param_table *hint_table = hint_table_list->begin();
       hint_table < hint_table_list->end(); hint_table++) {
    bool hint_table_found = false;
    for (uint i = 0; i < join->tables; i++) {
      const Table_ref *table = join->join_tab[i].table_ref;
      if (!compare_table_name(hint_table, table)) {
        hint_table_found = true;
        /* Const tables are excluded from the process of dependency setting
        since they are always first in the table order. */
        if (join->const_table_map & table->map()) break;
        JOIN_TAB *tab = &join->join_tab[i];
        // Hint tables are always dependent on preceding tables
        tab->dependent |= hint_tab_map;               /* ★ 链式依赖 */
        update_nested_join_deps(join, tab, hint_tab_map);
        hint_tab_map |= tab->table_ref->map();
        break;
      }
    }
    if (!hint_table_found) {
      print_join_order_warn(join->thd, type, hint_table);   /* ★ 表名找不到 → ER_UNRESOLVED_HINT_NAME */
      return true;
    }
  }

  // Add dependencies that are related to non-hint tables
  for (uint i = 0; i < join->tables; i++) {
    JOIN_TAB *tab = &join->join_tab[i];
    table_map dependent_tables =
        get_other_dep(type, hint_tab_map, tab->table_ref->map());
    update_nested_join_deps(join, tab, dependent_tables);
    tab->dependent |= dependent_tables;
  }

  if (join->propagate_dependencies()) return true;   /* ★★ 成环 → 返回 true */

  hint_handler.no_restore_deps();                    /* ★ 成功：阻止析构回滚 */
  return false;
}
```

**★ 回滚靠析构函数**（`Join_order_hint_handler`，★ 这是全篇最优雅的一处设计）：

```cpp
class Join_order_hint_handler {
  JOIN *join;
  table_map *orig_dep_array;  ///< Original table dependencies
 public:
  bool init() {
    orig_dep_array = (table_map *)join->thd->alloc(sizeof(table_map) * join->tables);
    if (orig_dep_array == nullptr) return true;
    for (uint i = 0; i < join->tables; i++)
      orig_dep_array[i] = join->join_tab[i].dependent;    /* ★ 快照 */
    return false;
  }
  void no_restore_deps() { orig_dep_array = nullptr; }
  ~Join_order_hint_handler() {
    if (orig_dep_array == nullptr) return;
    for (uint i = 0; i < join->tables; i++)
      join->join_tab[i].dependent = orig_dep_array[i];    /* ★★ 提前 return 时自动恢复 */
  }
};
```

> ★★★ **`set_join_hint_deps()` 有三条失败路径（表名找不到 / 成环 / OOM），全部靠这个析构函数把 `JOIN_TAB::dependent` 整体还原**。只有成功时才调 `no_restore_deps()` 让快照失效。

**★★ `get_other_dep()`：三种 join-order hint 的语义差异**：

```cpp
static table_map get_other_dep(opt_hints_enum type, table_map hint_tab_map,
                               table_map table_map) {
  switch (type) {
    case JOIN_PREFIX_HINT_ENUM:
      if (hint_tab_map & table_map)  return 0;              // Hint table: 无额外依赖
      return hint_tab_map;                                  // ★ 其他表依赖全部 hint 表 → hint 表必在最前
    case JOIN_SUFFIX_HINT_ENUM:
      if (hint_tab_map & table_map)  return ~hint_tab_map;  // ★ hint 表依赖所有其他表 → hint 表必在最后
      return 0;
    case JOIN_ORDER_HINT_ENUM:
      return 0;                                             // ★ 只约束 hint 表之间，其他表可自由穿插
    default: assert(0); break;
  }
  return 0;
}
```

**★ `propagate_dependencies()`：Warshall 传递闭包 + 自依赖检测**：

```cpp
bool JOIN::propagate_dependencies() {
  for (uint i = 0; i < tables; i++) {
    if (!join_tab[i].dependent) continue;
    for (uint j = 0, tab = 0; j < tables; j++, tab++) { ... Warshall ... }
  }
  for (JOIN_TAB *tab = join_tab; tab < tab_end; tab++) {
    if ((tab->dependent & tab->table_ref->map())) return true;  /* ★★ 自依赖 == 成环 */
  }
  return false;
}
```

> ★ 复杂度 O(n³)，注释原文："*However, most of the iterations will be shortcircuited when there are no dependencies to propagate.*"

### 5.2 依赖如何被遵守（best_extension_by_limited_search）

在 06 篇的主循环里（`sql_planner.cc:2770`）：

```cpp
if ((remaining_tables & real_table_bit) &&
    !(eq_ref_extended & real_table_bit) &&
    !(remaining_tables & s->dependent) &&   // ★ 依赖已满足才可扩展
    (!idx || !check_interleaving_with_nj(s))) {
```

`!(remaining_tables & s->dependent)` 表示"表 s 依赖的所有表都已不在 remaining（即已在前缀中）"。因为 JOIN_ORDER 已经把 `dependent` 设成它前面的 hint 表集合，贪心搜索就**不可能**把 hint 顺序打乱——违反依赖的表在候选过滤时被跳过。

### 5.3 JOIN_FIXED_ORDER 等价于 STRAIGHT_JOIN

`parse_tree_hints.cc:266`：

```cpp
case JOIN_FIXED_ORDER_HINT_ENUM:
  if (qb->get_switch(JOIN_PREFIX_HINT_ENUM) || ...)  // 与 JOIN_ORDER 系列互斥
    conflict = true;
  else
    pc->select->add_base_options(SELECT_STRAIGHT_JOIN);  // 等价 STRAIGHT_JOIN
  break;
```

`JOIN_FIXED_ORDER` 直接设 `SELECT_STRAIGHT_JOIN` 位，语义**完全等价**于 STRAIGHT_JOIN：保持 FROM 书写顺序。在 `choose_table_order`（06 篇 2.2）里走 `straight_join` 分支 → `optimize_straight_join` 只算访问方式、不搜 join order。

应用时机（`sql_optimizer.cc:5377`）：

```cpp
if (query_block->opt_hints_qb &&
    !(query_block->active_options() & SELECT_STRAIGHT_JOIN))  // fixed order 已由 straight 分支处理
  query_block->opt_hints_qb->apply_join_order_hints(this);
```

---

## 六、INDEX 如何限制候选索引

### 6.1 update_index_hint_map（集合运算，opt_hints.cc:599）

```cpp
void Opt_hints_table::update_index_hint_map(Key_map *keys_to_use,
                                            Key_map *available_keys_to_use,
                                            opt_hints_enum type_arg) {
  if (is_resolved(type_arg)) {
    Key_map *keys_specified_in_hint = get_compound_key_hint(type_arg)->get_key_map();
    if (get_switch(type_arg)) {
      if (keys_specified_in_hint->is_clear_all())
        keys_to_use->merge(*available_keys_to_use);       // INDEX(t) 全可用
      else {
        keys_to_use->merge(*keys_specified_in_hint);      // INDEX(t i1,i2)
        keys_to_use->intersect(*available_keys_to_use);   // ∩ 可用集
      }
    } else {
      if (keys_specified_in_hint->is_clear_all()) keys_to_use->clear_all();
      else keys_to_use->subtract(*keys_specified_in_hint); // NO_INDEX(t i1)
    }
  }
}
```

**集合运算**：`INDEX(t i1,i2)` = 指定键 ∩ 可用集；`NO_INDEX(t i1)` = 从候选减掉。结果写进 `TABLE::keys_in_use_for_query/group_by/order_by`（`update_index_hint_maps`，:641）。

### 6.2 range/ref 如何读取

- range 优化器（**`range_optimizer/range_optimizer.cc:382`**，⚠️ `sql/opt_range.cc` 已不存在）遍历 `keys_to_use`
- `NO_RANGE_OPTIMIZATION(t idx)` 额外在 `hint_key_state(..., NO_RANGE_HINT_ENUM, 0)` 剔除
- `find_best_ref` 遍历 `keys_in_use_for_query`，`NO_INDEX` 移除的索引不进 ref 候选
- `INDEX_MERGE` / `SKIP_SCAN` 用 `compound_hint_key_enabled`（`opt_hints.cc:932`）按键过滤

**★ `hint_key_state()` 全文**（★ 所有 key 级 hint 的统一读取口）：

```cpp
/* opt_hints.cc:845 —— 三类的代码判据就在这里 */
static bool get_hint_state(Opt_hints *hint, Opt_hints *parent_hint,
                           opt_hints_enum type_arg, bool *ret_val) {
  assert(parent_hint);
  if (opt_hint_info[type_arg].switch_hint) {
    if (hint && hint->is_specified(type_arg)) {
      *ret_val = hint->get_switch(type_arg);
      return true;
    }
    if (opt_hint_info[type_arg].check_upper_lvl &&
        parent_hint->is_specified(type_arg)) {        /* ★ 可向上继承 QB 级 */
      *ret_val = parent_hint->get_switch(type_arg);
      return true;
    }
  } else {
    if (hint && hint->is_specified(type_arg) &&
        /* This check is necessary because function idx_merge_key_enabled()
        can operate only with resolved hint. For unresolved hint function,
        idx_merge_key_enabled() always returns 'true' to emulate absence
        of the hint. */
        hint->is_resolved(type_arg)) {                /* ★ 复杂型必须已 resolve */
      *ret_val = hint->get_switch(type_arg);
      return true;
    }
  }
  return false;
}

bool hint_key_state(const THD *thd, const Table_ref *table, uint keyno,
                    opt_hints_enum type_arg, uint optimizer_switch) {
  Opt_hints_table *table_hints = table->opt_hints_table;
  if (table_hints && keyno != MAX_KEY) {
    Opt_hints_key *key_hints = table_hints->keyinfo_array.size() > 0
                                   ? table_hints->keyinfo_array[keyno]   /* ★ O(1) 取 */
                                   : NULL;
    bool ret_val = false;
    if (get_hint_state(key_hints, table_hints, type_arg, &ret_val))
      return ret_val;                                 /* key 级优先 */
  }
  return thd->optimizer_switch_flag(optimizer_switch); /* ★ 无 hint → 回落 optimizer_switch */
}
```

> ★★ **`optimizer_switch` 参数传 0 时恒为 false**——这是 `NO_RANGE_OPTIMIZATION` / `MRR()` 这类"无对应 switch"的 hint 的用法。
> ★ `keyinfo_array[keyno]` 的下标直接就是 `KEY` 的序号，由 `adjust_key_hints()` 在 resolve 期按下标建好。

**★ 每张表怎么拿到自己的 hint**（⚠️ 真名是 `Table_ref::opt_hints_table`，不是 `TABLE::opt_hints_qb`）：

```cpp
/* sql/table.h:3507 */
  /** Table level optimizer hints for this table.  */
  Opt_hints_table *opt_hints_table{nullptr};
  /* Hints for query block of this table. */
  Opt_hints_qb *opt_hints_qb{nullptr};
```

```cpp
/* sql_resolver.cc:1212 —— resolve 期 setup_tables() 里绑定 */
    if (opt_hints_qb && !tr->opt_hints_table) {
      tr->opt_hints_table = opt_hints_qb->adjust_table_hints(tr);   /* ★ 按 alias 匹配 */
    }
    if (!tr->opt_hints_table ||
        // Ignore old index hint processing if new style hints are specified.
        !tr->opt_hints_table->update_index_hint_maps(thd, tr->table)) {
      if (tr->process_index_hints(thd, table)) return true;         /* ★ 回退到旧式 USE/IGNORE INDEX */
    }
```

> ⚠️ **`check_index_hint()` / `is_index_hinted()` 都不存在**（全仓 0 命中）。旧式的真名是 `Table_ref::process_index_hints()`（`sql/table.cc:6356`），新式的真名是 `Opt_hints_table::update_index_hint_maps()`。两者**互斥**：新式生效就不走旧式。
> ★ `adjust_table_hints()` 用 `tr->alias`（**表别名**）在 `table_alias_charset` 下查找——所以 hint 里必须写别名。

**★ 复合型（带索引列表）hint 的判定**：

```cpp
bool compound_hint_key_enabled(const TABLE *table, uint keyno,
                               opt_hints_enum type_arg) {
  Opt_hints_table *table_hints = table->pos_in_table_list->opt_hints_table;
  if (table_hints && table_hints->is_resolved(type_arg)) {
    if (table_hints->get_compound_key_hint(type_arg)->is_key_map_clear_all())
      return table_hints->get_compound_key_hint(type_arg)->get_pt_hint()->switch_on();
      /* ★ 没列索引 → 对所有 keyno 返回 hint 开关值 */
    return table_hints->get_compound_key_hint(type_arg)->is_set_key_map(keyno) ==
           table_hints->get_compound_key_hint(type_arg)->get_pt_hint()->switch_on();
      /* ★ 列了索引：INDEX(x) → keyno∈x 为真；NO_INDEX(x) → keyno∉x 为真 */
  }
  return true;
}
```

```cpp
/* opt_hints.cc:573 —— 「复合 hint」共 6 个 */
bool is_compound_hint(opt_hints_enum type_arg) {
  return (type_arg == INDEX_MERGE_HINT_ENUM || type_arg == SKIP_SCAN_HINT_ENUM ||
          type_arg == INDEX_HINT_ENUM || type_arg == JOIN_INDEX_HINT_ENUM ||
          type_arg == GROUP_INDEX_HINT_ENUM || type_arg == ORDER_INDEX_HINT_ENUM);
}
```

> ★★ **INDEX 与 JOIN_INDEX|GROUP_INDEX|ORDER_INDEX 互斥**（`Glob_index_key_hint::is_hint_conflicting()`）：`INDEX(t i)` 与 `JOIN_INDEX(t j)` 同时给会报 `ER_WARN_CONFLICTING_HINT`。

---

## 七、SET_VAR / MAX_EXECUTION_TIME / RESOURCE_GROUP

### 7.1 SET_VAR（语句级临时改变量）

`PT_hint_sys_var::contextualize`（`parse_tree_hints.cc:524`）校验变量存在且标了 `HINT_UPDATEABLE`，注册进 `Sys_var_hint`。

执行前 `update_vars`（`sql_parse.cc:3350`）、执行后 `restore_vars`（`:4879`）：

```cpp
void Sys_var_hint::restore_vars(THD *thd) {
  for (Hint_set_var *hint_var : var_list) {
    if (hint_var->save_value) {
      std::swap(var->value, hint_var->save_value);  // 换回旧值
      var->update(thd);
      std::swap(var->value, hint_var->save_value);  // 换回 hint 值供重复执行
    }
  }
}
```

用"记录旧值 → 语句末换回"的方式实现，PS 重复执行时也能正确恢复。从库不应用 SET_VAR hint（`thd->slave_thread` 检查）。

**★ SET_VAR 的完整机制**（⚠️ **不走 `LEX::var_list`**——那是 `SELECT ... INTO @var` 的列表）：

```cpp
class Hint_set_var {           /* opt_hints.h:659 —— set_var + 旧值 */
 public:
  set_var *var;      // Pointer to set_var object
  Item *save_value;  // Original variable value
};

class Sys_var_hint {           /* ★ 挂在 Opt_hints_global::sys_var_hint 上 */
  Mem_root_array<Hint_set_var *> var_list;
 public:
  bool add_var(THD *thd, const System_variable_tracker &var_tracker, Item *sys_var_value);
  void update_vars(THD *thd);
  void restore_vars(THD *thd);
  void print(const THD *thd, String *str);
};
```

```cpp
void Sys_var_hint::update_vars(THD *thd) {
  if (thd->slave_thread) return;                    /* ★ 从库不应用 */
  Set_var_error_handler error_handler(false);       /* ★ 保留警告 */
  for (uint i = 0; i < var_list.size(); i++) {
    thd->push_internal_handler(&error_handler);
    set_var *var = var_list[i]->var;
    if (!var->resolve(thd) && !var->check(thd)) {
      auto f = [thd](const System_variable_tracker &, sys_var *v) -> Item * {
        return v->copy_value(thd);                  /* ★ 把旧值克隆成 Item* */
      };
      Item *save_value =
          var->m_var_tracker.access_system_variable<Item *>(thd, f).value_or(nullptr);
      if (!var->update(thd)) var_list[i]->save_value = save_value;
    }
    thd->pop_internal_handler();
    error_handler.reset_state();
  }
}

void Sys_var_hint::restore_vars(THD *thd) {
  Set_var_error_handler error_handler(true);        /* ★ 完全静音 */
  ...
      std::swap(var->value, hint_var->save_value);  // 换回旧值
      var->update(thd);
      std::swap(var->value, hint_var->save_value);  /* ★★ 再换回来 —— 供下次 EXECUTE */
}
```

> ★★ 最后那个"换回来"的注释原文：*"Restore hint value for further executions"*。这就是 PS 能重复应用 SET_VAR 的原因。
> ★ `add_var()` 里硬编码 `new set_var(OPT_SESSION, ...)`——**SET_VAR 只能改会话级**。
> ★ 校验链：变量必须存在 + `is_hint_updateable()`（即 `sys_var::flags & HINT_UPDATEABLE`）。`System_variable_tracker::is_hint_updateable()` 还要求 `m_tag == STATIC`——**keycache / plugin / component 变量一律不可**。

**★ 调用时机**：

| 时点 | 位置 |
|---|---|
| 执行前 | `sql_parse.cc:3350` → `update_vars()` |
| 执行后（含 error 路径） | `sql_parse.cc:4882` → `restore_vars()` |
| PREPARE 前/后 | `sql_prepare.cc:2589/2597`（prepare 完立刻恢复，不污染会话） |

### 7.2 MAX_EXECUTION_TIME

仅 SELECT、非存储程序、非子查询（`parse_tree_hints.cc:497`），写 `lex->max_execution_time`。消费点（`sql_select.cc:164`）：

```cpp
return (thd->lex->max_execution_time ? thd->lex->max_execution_time
                                     : thd->variables.max_execution_time);
```

优先级：hint > session 变量 `max_execution_time`（默认 0 = 无超时）。

**★ 三条硬限制**（⚠️ 真名是 **`LEX::max_execution_time`**，**不在 THD**）：

```cpp
bool PT_hint_max_execution_time::contextualize(Parse_context *pc) {
  if (pc->thd->lex->sql_command != SQLCOM_SELECT ||  // ① 只支持 SELECT
      pc->thd->lex->sphead ||                        // ② 不在存储程序/触发器/事件里
      pc->select != pc->thd->lex->query_block)       // ③ 不在子查询里
  {
    push_warning(pc->thd, Sql_condition::SL_WARNING,
                 ER_WARN_UNSUPPORTED_MAX_EXECUTION_TIME, ...);
    return false;
  }
  ...
  pc->thd->lex->max_execution_time = milliseconds;   /* ★★ 解析期就写进 LEX，优化器完全不参与 */
  global_hint->set_switch(switch_on(), type(), false);
  global_hint->max_exec_time = this;                 /* 仅为打印保留 */
  return false;
}
```

```cpp
/* ★ 定时器真名是 set_statement_timer() / thd_timer_set() —— 不存在 thd->set_timer() / Timer 类 */
bool set_statement_timer(THD *thd) {
  ulong max_execution_time = get_max_execution_time(thd);
  thd->timer = thd_timer_set(thd, thd->timer_cache, max_execution_time);
  thd->timer_cache = nullptr;
  if (thd->timer) thd->status_var.max_execution_time_set++;
  else            thd->status_var.max_execution_time_set_failed++;
  return thd->timer;
}
```

```cpp
static inline bool is_timer_applicable_to_statement(THD *thd) {
  return (thd->lex->sql_command == SQLCOM_SELECT &&
          (have_statement_timeout == SHOW_OPTION_YES) && !thd->slave_thread &&
          !thd->timer &&
          (thd->lex->max_execution_time || thd->variables.max_execution_time) &&
          !thd->sp_runtime_ctx);
}
```

> ★ 装载点 `sql_select.cc:698`（`Sql_cmd_dml::execute()`），卸载点 `:824` / `:871` 调 `reset_statement_timer()`。

### 7.3 RESOURCE_GROUP

`PT_hint_resource_group::contextualize`（:566）写 `thd->resource_group_ctx()`，执行前 `switch_resource_group_if_needed`（`sql_prepare.cc:3659`），执行后恢复。

**★ 全文**（⚠️ **不存在 `LEX::resource_group_name`**）：

```cpp
bool PT_hint_resource_group::contextualize(Parse_context *pc) {
  auto res_grp_mgr = resourcegroups::Resource_group_mgr::instance();
  if (!res_grp_mgr->resource_group_support()) {
    pc->thd->resource_group_ctx()->m_warn = WARN_RESOURCE_GROUP_UNSUPPORTED;   /* 平台不支持 */
    return false;
  }
  if (pc->thd->lex->sphead || pc->select != pc->thd->lex->query_block) {
    pc->thd->resource_group_ctx()->m_warn = WARN_RESOURCE_GROUP_UNSUPPORTED_HINT;  /* SP/子查询 */
    return false;
  }
  memcpy(pc->thd->resource_group_ctx()->m_switch_resource_group_str,
         m_resource_group_name.str, m_resource_group_name.length);   /* ★★ 真名 */
  pc->thd->resource_group_ctx()
      ->m_switch_resource_group_str[m_resource_group_name.length] = '\0';
  return false;
}
```

> ★ 语法里 `RESOURCE_GROUP` 是**唯一用 `YYERROR`（而非 `YYABORT`）**的地方——`check_resource_group_name_len()` 失败只跳过本产生式，不终止整个 hint 解析。
> ★ 告警统一在 `sql_parse.cc:3030` 分发（`WARN_RESOURCE_GROUP_*` 五档）。

---

## ★★ 8.0.39 hint 消费点全景表（哪里真的读了它）

这是本篇最实用的一张表——**hint 在哪里被读取**，而不是在哪里被解析：

| hint | 消费位置（真实） | 时点 |
|---|---|---|
| JOIN_PREFIX / JOIN_SUFFIX / JOIN_ORDER | `sql_optimizer.cc:5377` → `opt_hints.cc:459` → 写 `JOIN_TAB::dependent` → `sql_planner.cc:2772` 检查 | optimize |
| JOIN_FIXED_ORDER | **解析期** `parse_tree_hints.cc:272` `add_base_options(SELECT_STRAIGHT_JOIN)` | parse |
| STRAIGHT_JOIN（关键字） | `sql_planner.cc:1971` 读 `active_options() & SELECT_STRAIGHT_JOIN` | optimize |
| INDEX / JOIN_INDEX / GROUP_INDEX / ORDER_INDEX | `sql_resolver.cc:1222` → `opt_hints.cc:641` | resolve |
| NO_ICP | `sql_select.cc:2987`（`QEP_TAB::push_index_cond`） | 执行期 |
| NO_RANGE_OPTIMIZATION | `range_optimizer/range_optimizer.cc:388` | optimize |
| MRR / NO_MRR | `handler.cc:6559`（`dsmrr_init`）、`:6972`（`choose_mrr_impl`） | 执行期 |
| BKA / NO_BKA | `sql_optimizer.cc:3460/3632`、`handler.cc:6975` | optimize |
| BNL / NO_BNL | `sql_planner.cc:1007`（**代价估算**）、`sql_optimizer.cc:3458/3621` | optimize |
| MERGE / NO_MERGE | `sql_resolver.cc:3497`（`Query_block::merge_derived`） | **resolve** |
| SEMIJOIN / NO_SEMIJOIN | `sql_lex.cc:4552/4583` → `opt_hints.cc:239/256` | resolve |
| SUBQUERY | `sql_lex.cc:4538` → `opt_hints.cc:270` | resolve |
| SKIP_SCAN | `sql_optimizer.cc:8173`、`index_skip_scan_plan.cc:258` | optimize |
| INDEX_MERGE | `range_optimizer.cc:751/773/1057`、`opt_hints.cc:953` | optimize |
| DERIVED_CONDITION_PUSHDOWN | `sql_derived.cc:1030` | optimize |
| MAX_EXECUTION_TIME | 解析期写 `lex->max_execution_time`；执行期 `sql_select.cc:700` | parse + 执行 |
| RESOURCE_GROUP | 解析期 `parse_tree_hints.cc:581`；执行期 `sql_parse.cc:3030` | parse + 执行 |
| SET_VAR | `sql_parse.cc:3352` / `:4882` | 执行 |
| **HASH_JOIN / NO_HASH_JOIN** | **无消费点** | — |

> ★★★ **`JOIN::optimize()` 只读 join-order 类 hint**（注释原文："*Apply join order hints, with the exception of JOIN_FIXED_ORDER and STRAIGHT_JOIN*"）。其余 hint 分散在 resolve / 执行 / 语句级三个时点——这是"为什么有些 hint 改了却没效果"的第一排查项。

**★ hint 在 EXPLAIN / optimizer trace 里的可见性**（★ 一个常被误解的点）：

- **普通 `EXPLAIN` 的输出列（id/type/possible_keys…）不含任何 hint 字段**；
- hint 只出现在**重写后的查询文本**里：`Query_block::print_hints()`（`sql_lex.cc:3095`）→ `print()` 递归；
- **被忽略的 join-order hint 会主动消失**：`print_irregular_hints()` 里 `if (join_order_hints_ignored & (1ULL << i)) continue;`（注释："*Skip hint printing in EXPLAIN message*"）；
- 未解析的 hint 只在 `QT_NORMALIZED_FORMAT` 下打印（供 query rewrite 插件用）；
- **optimizer trace 里只有 3 条 hint 痕迹**：`"no_range_optimization hint"`、`"skip_scan_hint"`、`"index_merge_hint"`。`sql/opt_trace.cc` 中 `hint` 0 命中。

**★★ prepared statement 二次执行**（★ 结论：hint 全保留，每次 EXECUTE 重新生效）：

1. hint 树挂在 `LEX`/`Query_block`/`Table_ref` 上，PS 生命周期内常驻；
2. 每次 EXECUTE 重跑 `make_join_plan()`，而 `tab->dependent` 每次被重新初始化（`sql_optimizer.cc:5495` `tab->dependent = tl->dep_tables;`）→ `apply_join_order_hints()` 每次重新写依赖；
3. SET_VAR 的 `restore_vars()` 末尾 swap 回来 → 可重复应用；
4. ⚠️ **一个可观察的副作用**：`join_order_hints_ignored` **只置位从不清除**（`opt_hints.cc:210` 构造时置 0，`:525` 只 `|=`）。若某次因成环/表未解析导致 hint 被忽略，该位**一直保留**，后续执行的重写文本里该 hint 都不再打印。

---

## 八、8.0.39 已废弃或未生效的 hint

| hint | 状态 |
|------|------|
| `MAX_STATEMENT_TIME` | 5.7.7 旧名，8.0 **已彻底移除**（全库 0 匹配），用 `MAX_EXECUTION_TIME` 替代 |
| `HASH_JOIN` / `NO_HASH_JOIN` | 枚举仍存在、语法器仍登记，但 **8.0.39 无任何代码消费**（grep `HASH_JOIN_HINT_ENUM` 无 `hint_table_state` 调用）。⚠️ 且**不是**"由 hypergraph 按 `optimizer_switch hash_join` 决定"——`OPTIMIZER_SWITCH_HASH_JOIN` 在 `sql/` 下只有常量定义与默认值两处、**无任何消费点**，两条优化器路径都不读它。开关与 hint 双双失效 |
| 旧式 `USE INDEX`/`FORCE INDEX`/`IGNORE INDEX` | 不是 `/*+ */` optimizer hint；若新式 INDEX hint 生效会被跳过（`sql_resolver.cc:1220`） |

**★ 其它"静默失效"的场景**（★ 都不报错、不告警，最难排查）：

| 场景 | 结果 |
|---|---|
| `/*+ */` 写在非 hintable 关键字后（如 `FROM /*+ ... */`） | 被当成普通注释丢弃，**完全不生效** |
| `JOIN_ORDER` + `STRAIGHT_JOIN` | `sql_optimizer.cc:5378` 直接跳过 join-order hint（**无 warning**） |
| `MERGE` + 显式 `ALGORITHM=TEMPTABLE` | `sql_resolver.cc:3487` 直接 return，hint 被忽略 |
| JOIN_ORDER 成环 / 表名拼错 | hint 静默丢弃 + `dependent` 回滚；只有表名拼错会报 `ER_UNRESOLVED_HINT_NAME` |
| `BNL` 想关掉 hash join | 做不到——BNL 只管 `JOIN_CACHE::ALG_BNL`，hash join 是 `AccessPath::HASH_JOIN` 另一条路 |
| `CREATE VIEW` 里写非 index 级 hint | `PT_hint::supports_view()` 只允许 `INDEX..ORDER_INDEX`，其余被 `PT_hint_list::contextualize` 丢弃 |

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `LEX::consume_optimizer_hints()` | **不存在**（它不是 LEX 成员）→ `sql_lex.cc:865` 的 **static 自由函数** |
| `class Hint_scanner` 在 `sql/sql_lex.h` | **不在** → 独立头 **`sql/sql_lex_hints.h:57`** |
| `sql_hints.yy.cc` 是源文件 | **不存在** → bison 构建期生成；`sql_hints.yy.h` 同样不存在（却被 `#include`） |
| 256 项表是 `hint_lex_char_map` 数组字面量 | **不存在该名字** → `cs->state_maps->hint_map[256]`，由 `hint_lex_init_maps()` **运行时循环填充** |
| 主扫描函数叫 `get_token()` | → **`Hint_scanner::scan()`**（protected）；公开入口是 `get_next_token()` |
| `add_hint_token_to_digest()` | → **`add_hint_token_digest()`** |
| hint 语法器用 `MYSQL_YYABORT` | **不存在**（只在 `sql_yacc.yy:206`）→ 全程裸用 `YYABORT`/`YYERROR`。`MYSQL_YYACCEPT` 全仓不存在 |
| 主语法有 `optimizer_hints` 非终结符 | **不存在** → 只是 `Lexer_yystype` 的 union 成员；hint 列表**挂在关键字 token 自己的 yylval 上** |
| `LEX::current_hint` | **不存在**（0 命中）→ LEX 侧只有 `opt_hints_global` |
| 主词法器在注释分支里识别 `/*+` | **不是** → 只在 `find_keyword()` 里对 `SG_HINTABLE_KEYWORDS` 关键字**后向窥探**；普通 `/* */` 分支只认 `/*!` |
| `Hint_scanner` 由 LEX 持有 | **不是** → `consume_optimizer_hints()` 的**栈上局部对象**，只持有 `THD*` |
| `skip_digest` = "hint token 不进 digest" | **恰恰相反**——hint token **是**进 digest 的；`skip_digest` 只防**关键字 token（SELECT）被重复追加** |
| `sql/opt_hints_enum.h` | **不存在** → 枚举在 `opt_hints.h:64`，元信息数组在 `opt_hints.cc:65` |
| 有 `class Parser` 负责 hint 解析 | **不存在** → 独立 bison 语法 `sql_hints.yy` + 独立词法器 `Hint_scanner` |
| `Opt_hints::register_hint()` / `get_hints()` / `find_hint()` / `set_hint()` | **都不存在** → `register_child()` / `get_complex_hints()` / `find_by_name()` / `set_switch()` |
| 基类成员 `m_hint_type` / `m_name` / `m_hints` | **这些名字不存在** → `name` / `parent` / `hints_map` / `child_array` / `resolved` / `resolved_children` |
| `Opt_hints_qb::set_join_hint_deps()` | **不是成员函数** → static 自由函数 `set_join_hint_deps()`（`opt_hints.cc:459`）；成员函数是 `apply_join_order_hints()` |
| `substitute_hint()` | **不存在** → 视图 hint 迁移靠 `PT_hint::supports_view()` + `sql_view.cc:1645` + `find_qb_hints()` 的 `select#N` 回填 |
| `enum class Semijoin_preference` | **不存在** → 直接用 `OPTIMIZER_SWITCH_FIRSTMATCH/LOOSE_SCAN/MATERIALIZATION/DUPSWEEDOUT` 位 |
| `enum class Subquery_strategy` | 存在，但在 **`sql/item_subselect.h:398`**，不在 opt_hints 文件里 |
| `Opt_hints_table::get_key_hint()` | **不存在** → `get_compound_key_hint(type)` |
| `MAX_EXECUTION_TIME` 在 THD 成员 | **不在** → `LEX::max_execution_time`（`sql_lex.h:4361`） |
| `OPTIMIZER_SWITCH_BNL` 已废弃 / 8.0 移除了 BNL | **未移除** → 仍被 `sql_planner.cc:1007`、`sql_optimizer.cc:3458/3621` 真实读取 |
| `HASH_JOIN` hint 控制 hash join | **死 hint** → 无任何消费点 |
| `thd->set_timer()` / `Timer` 类 | **都不存在** → `set_statement_timer()`（`sql_select.cc:203`）+ `thd_timer_set()`；结构体叫 `THD_timer_info` |
| `LEX::resource_group_name` | **不存在** → `THD::resource_group_ctx()->m_switch_resource_group_str` |
| SET_VAR 走 `LEX::var_list` | **错** → `LEX::var_list` 是 `SELECT ... INTO @var`；SET_VAR 走 `opt_hints_global->sys_var_hint` |
| `TABLE::opt_hints_qb` | → **`Table_ref::opt_hints_table`** + `Table_ref::opt_hints_qb`（`table.h:3507-3510`） |
| `check_index_hint()` / `is_index_hinted()` | **都不存在** → 旧式 `Table_ref::process_index_hints()`，新式 `Opt_hints_table::update_index_hint_maps()` |
| `JOIN::const_table` | → **`JOIN::const_tables`**（数量）+ **`JOIN::const_table_map`**（位图） |
| `MRR_iter` 类 | → **`MultiRangeRowIterator`** + **`DsMrr_impl`**（`handler.cc`） |
| `sql/opt_range.cc` | **已不存在** → `sql/range_optimizer/range_optimizer.cc` |
| `sql/sql_tmp_table.cc` 有 hint 读取点 | **无**（该文件 `hint`/`HINT_ENUM`/`opt_hints` 全 0 命中） |

**★ 三个"结构性事实"**

1. **`/*+ */` 不是注释，是"挂在关键字 token 上的一个语义值"**。主语法器里**没有** `optimizer_hints` 非终结符，`PT_hint_list*` 塞在 `SELECT_SYM`/`INSERT_SYM`/… 五个关键字**自己的 yylval** 里（`%type <lexer.optimizer_hints> SELECT_SYM INSERT_SYM ...`）。这解释了为什么 hint 只能出现在这五个关键字之后，也解释了为什么写错位置会**静默失效**（退化成普通注释，不报错）。

2. **`skip_digest` 修的是一个"顺序" bug，不是"要不要进 digest"**。hint token 由 `Hint_scanner::add_hint_token_digest()` 写入，而 `SELECT` 关键字由 `find_keyword()` **先**写入——因为 hint 扫描发生在 `lex_one_token()` **内部**，比外层的 `add_digest_token(SELECT)` 更早。若不抑制，digest 会变成 `/*+ HINT(t) */ SELECT`（错序）。所以 `skip_digest` 是**一次性**开关，用完立刻 `= false`。

3. **hint 的"生效"分散在四个时点，而 `JOIN::optimize()` 只管 join-order 一类**。解析期（JOIN_FIXED_ORDER → `SELECT_STRAIGHT_JOIN`；MAX_EXECUTION_TIME → `lex->max_execution_time`；RESOURCE_GROUP → `resource_group_ctx()`）、resolve 期（INDEX 系、MERGE、SEMIJOIN/SUBQUERY）、optimize 期（join-order、BNL/BKA、SKIP_SCAN、INDEX_MERGE）、执行期（NO_ICP、MRR、SET_VAR）。**"改了 hint 没效果"的第一排查项就是：这个 hint 的消费点在哪个时点，你的场景是否到达了那个时点。**

> 注：`sql/sql_hints.yy`、`sql/opt_hints.cc` 等文件中的部分 `// 中文` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**内核月报**
- **2020/09《MySQL · Optimizer · Optimizer Hints》** —— hint 机制的四层结构、三类影响方式、系统价值（本文的直接来源）

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizer Hints*（`/*+ */` 语法、各 hint 语义、`QB_NAME`）
- *MySQL 8.0 Reference Manual → Index-Level Optimizer Hints*（8.0.20+）

**相关文档**
- JOIN_ORDER 依赖如何被搜索遵守见 [`physical/06_join_order.md`](physical/06_join_order.md)
- INDEX 如何影响 range/ref 见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)、[`physical/07_access_method.md`](physical/07_access_method.md)
- SEMIJOIN/SUBQUERY 策略见 [`logical/03_semijoin.md`](logical/03_semijoin.md)、[`logical/02_subquery.md`](logical/02_subquery.md)
