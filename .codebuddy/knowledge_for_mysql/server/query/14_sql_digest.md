# 14 SQL Digest：语句归一化与指纹

> 本篇剖析 digest 的完整机制：**SQL 如何被归一化、归一化结果如何被哈希、指纹被谁消费**。核心文件 `sql/sql_digest.cc`（归一化与哈希）、`sql/sql_lex.cc`（token 收集）、`storage/perfschema/`（消费方）。定位是横切篇：计算在 SQL 层，消费在 PFS。

## 目录

- [一句话机制](#一句话机制)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、归一化的精确规则](#一归一化的精确规则)
  - [1.8 归一化的实现：就地归约状态机](#18-归一化的实现就地归约状态机)
- [二、token 收集机制](#二token-收集机制)
- [三、哈希与存储](#三哈希与存储)
  - [3.2.1 字节流的实际编码](#321-字节流的实际编码)
  - [3.2.2 完整例子：SELECT a FROM t WHERE b=1](#322-一个完整例子select-a-from-t-where-b1)
  - [3.4 反解析：从字节流还原 DIGEST_TEXT](#34-反解析从字节流还原-digest_text)
- [四、两条独立通道：token 流 vs QT_NORMALIZED_FORMAT](#四两条独立通道token-流-vs-qtnormalizedformat)
  - [4.1 第二条通道的实现](#41-第二条通道的实现itemprint-逐个吐-)
  - [4.2 两套通道的实际差异](#42-两套通道的实际差异)
  - [4.3 STATEMENT_DIGEST_TEXT() 走哪条](#43-statement_digest_text-走哪条--token-通道)
- [五、PFS 消费方](#五pfs-消费方)
  - [5.1 从 THD 到 PFS：传递路径](#51-从-thd-到-pfsdigest-的传递路径)
  - [5.2 聚合 key = (schema, digest)](#52-聚合-key--schema-digest且只有这两个)
  - [5.3 存储结构与槽分配算法](#53-存储结构与槽分配算法)
  - [5.6 DIGEST_TEXT 是懒计算的](#56-digest_text-是懒计算的)
  - [5.7 max_digest_length = 0 的三层禁用](#57-max_digest_length--0-的三层禁用)
- [六、版本演进：5.7 → 8.0](#六版本演进57--80)
- [七、已知限制](#七已知限制)
- [参考](#参考)

---

## 一句话机制

> digest **不是"重写 SQL 文本"**，而是词法器在解析时把 token 流（token id + 标识符原文）**边收集边归约**地写入一个字节数组；语句解析成功后对**该字节数组**做 SHA-256 得 DIGEST（32 字节）；需要展示时再从字节数组**重放**出归一化文本 DIGEST_TEXT（字面量变成 `?`）。

```
解析时（MYSQLlex 逐 token）           解析成功后               读取 PFS 行时
token → digest_add_token()   →   compute_digest_hash()  →  compute_digest_text()
        └─ 就地归约（字面量→?）       └─ SHA-256(字节流)       └─ 从字节流重放文本
```

---

## 设计思想与理论基础

### 为什么需要 digest：SQL 指纹

`WHERE a=1` 和 `WHERE a=2` 是**同一个查询形状**的两次执行。性能分析关心的是形状的聚合统计（执行了多少次、总耗时、平均行数），而不是每一条参数不同的实例。

digest 就是**SQL 指纹**：把"形状相同、参数不同"的语句归一化到同一个桶，使单次语句统计聚合成有意义的总量。`events_statements_summary_by_digest` 的一切（COUNT_STAR / SUM_TIMER_WAIT / 直方图）都建立在这个指纹上。

### 与 SPM 计划身份的关系：同思想、不同体系

[`07_optimize/14_plan_stability.md`](07_optimize/14_plan_stability.md) 讲过 Oracle SPM 的 SQL signature、PolarDB 的参数化 SQL 文本。三者都是"归一化文本/指纹"这一思想的实现，但用途不同：

| | MySQL digest | Oracle signature / PolarDB 参数化 |
|---|---|---|
| 归一化结果 | token 字节流 → SHA-256 | 归一化文本 / 参数占位文本 |
| 用途 | **观测与统计**（PFS 聚合、sys 视图） | **计划匹配的 key**（baseline） |
| 参与计划选择 | ❌ 无（源码中无任何代码路径） | ✅ 是 |

MySQL 的 digest 哈希**唯一的消费方是 PFS**（及 `STATEMENT_DIGEST()` 函数、parser service）。

### 归一化丢了什么

丢掉：**所有参数值**——字符串内容、数字大小、IN 列表元素个数（`IN (?)` 和 `IN (...)` 只区分单值/多值，`IN (1,2)` 与 `IN (1,...,1000)` 同 digest）。

保留：完整语句骨架——表名、列名、函数名、hint、运算符。

**对性能分析的意义**：同一 digest 下可能混着执行计划截然不同的执行（IN 列表 2 个值和 5000 个值可能走不同索引），聚合值是"形状平均"，单个病态执行会藏匿。PFS 的缓解是 `QUERY_SAMPLE_TEXT`：每个 digest 槽保留一条**真实原文**，且倾向保留等待时间最高的那个样本。

---

## 一、归一化的精确规则

### 1.1 字面量 → `?`（与长度无关）

`digest_add_token()` 对所有字面量类 token 一律改写为 `TOK_GENERIC_VALUE`：

```cpp
case LEX_HOSTNAME: case TEXT_STRING: case NCHAR_STRING: case PARAM_MARKER: {
  token = TOK_GENERIC_VALUE;      // 打印为 "?"
  store_token(digest_storage, token);
  break;
}
case NUM: case LONG_NUM: case ULONGLONG_NUM: case DECIMAL_NUM:
case FLOAT_NUM: case BIN_NUM: case HEX_NUM: { ... }
```

⚠️ 常见误传"不同长度字符串用不同占位符避免碰撞"——**源码无此机制**。1 字节和 1MB 的字符串字面量在 digest 里完全相同，都是 `?`。

### 1.2 一元正负号被吸收，二元保留（最精妙的一条）

```cpp
/*
  We need to differentiate:
  - a <unary minus> operator ... from - a <binary minus> operator
  to only reduce "a = -1" to "a = ?", and not change "b - 1" to "b ?"
*/
if (lex_token_array[last_token2].m_start_expr) {   // 前一个 token 会开启表达式
  token = TOK_GENERIC_VALUE;
  digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;  // pop 掉符号
  found_unary = true;
}
```

- `WHERE a = -1` → `WHERE a = ?`（负号连数字坍缩）
- `WHERE a - 1 > 0` → `WHERE a - ? > 0`（二元保留）

判别依据是**前一个 token 的 `m_start_expr` 标志**（`(`、`,`、`SELECT`、`AND`、`BETWEEN`、`+`、`-` 等会开启表达式的 token，在 `gen_lex_token.cc` 硬编码）。

### 1.3 标识符保留原名，大小写敏感

```cpp
case IDENT: case IDENT_QUOTED: case TOK_IDENT_AT: {
  ...
  store_token_identifier(digest_storage, token, yylen, yytext);   // 原文字节
```

- 表名/列名**不归一化**，重放时反引号原样输出
- **无 case-fold**：`SELECT * FROM T` 与 `SELECT * FROM t` 是**两个不同 digest**（在大小写不敏感系统上是隐性的"同表不同桶"）
- 关键字无所谓大小写（按 token id 存，统一大写打印）

### 1.4 注释、空格删除；hint 例外

- 普通注释（`-- `、`/* */`、`#`）在词法状态机 `MY_LEX_COMMENT` 被跳过，从不产生 token
- 空格同理；重放时按 `m_append_space` 补空格（如 `@` 后不补，保证 `@@variable`）
- **例外：hint 注释 `/*+ ... */` 进入 digest**（见 2.2）

### 1.5 IN 列表折叠

- `x IN (1,2)` → `x IN (?)`
- `x IN (1,2,3...)` → `x IN (...)`（`TOK_IN_GENERIC_VALUE_EXPRESSION`）

目的是让超长 IN 列表不按元素数量膨胀 token 数组，而非按字面量长度区分。

### 1.6 NULL 的二义性：词法不够，yacc 补刀

词法器分不清"字面量 NULL"和"`IS NULL` 运算符"，必须靠 yacc 规则在特定语法位置显式归约：

```yacc
/* For the digest computation, in this context only, NULL is considered
   a literal, hence reduced to '?'
   REDUCE: TOK_GENERIC_VALUE := NULL_SYM */
lip->reduce_digest_token(TOK_GENERIC_VALUE, NULL_SYM);
```

`IS NULL` 的 `NULL` 保持运算符不变。这说明**"纯 token 流"归一化对少数语法点不完备**，需要解析器介入。

### 1.7 其他规则

- `(?)` → `TOK_ROW_SINGLE_VALUE`；`(?, ...)` → `(?) /* , ... */`
- 结尾分号在词法器返回 token 0 时被弹掉 → DIGEST_TEXT 不带尾分号
- 归一化文本最多输出 `max_digest_length`（默认 1024）字节即截断，**无 `...` 后缀**

### 1.7.1 ★ `digest_add_token()` 完整源码（1.1–1.7 全部规则的唯一实现处）

> 前面 1.1–1.7 讲的所有规则，**都只是这一个 `switch` 的分支**。它是"归一化"的全部实现（`sql/sql_digest.cc:380-621`）：

```cpp
sql_digest_state *digest_add_token(sql_digest_state *state, uint token,
                                   Lexer_yystype *yylval) {
  sql_digest_storage *digest_storage = nullptr;

  digest_storage = &state->m_digest_storage;

  /*
    Stop collecting further tokens if digest storage is full or
    if END token is received.
  */
  if (digest_storage->m_full || token == END_OF_INPUT) {
    return nullptr;      // ★ nullptr 会让调用方 m_digest 置空 → 之后彻底短路
  }

  /*
    Take last_token 2 tokens collected till now. These tokens will be used
    in reduce for normalisation. Make sure not to consider ID tokens in reduce.
  */
  uint last_token;
  uint last_token2;

  switch (token) {
    case NUM:
    case LONG_NUM:
    case ULONGLONG_NUM:
    case DECIMAL_NUM:
    case FLOAT_NUM:
    case BIN_NUM:
    case HEX_NUM: {
      bool found_unary;
      do {
        found_unary = false;
        peek_last_two_tokens(digest_storage, state->m_last_id_index,
                             &last_token, &last_token2);

        if ((last_token == '-') || (last_token == '+')) {
          /*
            We need to differentiate:
            - a <unary minus> operator
            - a <unary plus> operator
            from
            - a <binary minus> operator
            - a <binary plus> operator
            to only reduce "a = -1" to "a = ?", and not change "b - 1" to "b ?"

            Binary operators are found inside an expression,
            while unary operators are found at the beginning of an expression,
            or after operators.

            To achieve this, every token that is followed by an <expr>
            expression in the SQL grammar is flagged. See sql/sql_yacc.yy See
            sql/gen_lex_token.cc

            For example,
            "(-1)" is parsed as "(", "-", NUM, ")", and
            lex_token_array["("].m_start_expr is true, so reduction of the "-"
            NUM is done, the result is "(?)".
            "(a-1)" is parsed as "(", ID, "-", NUM, ")", and
            lex_token_array[ID].m_start_expr is false, so the operator is
            binary, no reduction is done, and the result is "(a-?)".
          */
          if (lex_token_array[last_token2].m_start_expr) {
            /*
              REDUCE:
              TOK_GENERIC_VALUE := (UNARY_PLUS | UNARY_MINUS) (NUM | LOG_NUM |
              ... | FLOAT_NUM)

              REDUCE:
              TOK_GENERIC_VALUE := (UNARY_PLUS | UNARY_MINUS) TOK_GENERIC_VALUE
            */
            token = TOK_GENERIC_VALUE;
            digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;   // ★ 弹出一元 +/-，游标回退 2
            found_unary = true;                                // ★ do-while 支持 "- -1"
          }
        }
      } while (found_unary);
    }
      /* fall through, for case NULL_SYM below */
      [[fallthrough]];      // ★ 数字字面量归约完，继续往下走 TOK_GENERIC_VALUE 归一化
    case LEX_HOSTNAME:
    case TEXT_STRING:
    case NCHAR_STRING:
    case PARAM_MARKER: {
      /*
        REDUCE:
        TOK_GENERIC_VALUE := BIN_NUM | DECIMAL_NUM | ... | ULONGLONG_NUM
      */
      token = TOK_GENERIC_VALUE;      // ★ 所有字面量 → '?'（PARAM_MARKER 同形）

      peek_last_two_tokens(digest_storage, state->m_last_id_index, &last_token,
                           &last_token2);

      if ((last_token2 == TOK_GENERIC_VALUE ||
           last_token2 == TOK_GENERIC_VALUE_LIST) &&
          (last_token == ',')) {
        /*
          REDUCE:
          TOK_GENERIC_VALUE_LIST :=
            TOK_GENERIC_VALUE ',' TOK_GENERIC_VALUE
          REDUCE:
          TOK_GENERIC_VALUE_LIST :=
            TOK_GENERIC_VALUE_LIST ',' TOK_GENERIC_VALUE
        */
        digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;   // ★ 弹出 '?' 与 ',' 共 4 字节
        token = TOK_GENERIC_VALUE_LIST;                        // ★ 折叠点 1 → "?, ..."
      }
      store_token(digest_storage, token);
      break;
    }
    case ')': {
      peek_last_two_tokens(digest_storage, state->m_last_id_index, &last_token,
                           &last_token2);

      if (last_token == TOK_GENERIC_VALUE && last_token2 == '(') {
        /*
          REDUCE:
          TOK_ROW_SINGLE_VALUE := '(' TOK_GENERIC_VALUE ')'
        */
        digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
        token = TOK_ROW_SINGLE_VALUE;                          // ★ 折叠点 2 → "(?)"

        /* Read last two tokens again */
        peek_last_two_tokens(digest_storage, state->m_last_id_index,
                             &last_token, &last_token2);

        if ((last_token2 == TOK_ROW_SINGLE_VALUE ||
             last_token2 == TOK_ROW_SINGLE_VALUE_LIST) &&
            (last_token == ',')) {
          digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
          token = TOK_ROW_SINGLE_VALUE_LIST;                   // ★ 折叠点 3 → "(?) /* , ... */"
        } else if (last_token == IN_SYM) {
          /*
            REDUCE:
            TOK_IN_GENERIC_VALUE_EXPRESSION := IN_SYM TOK_ROW_SINGLE_VALUE
          */
          digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;
          token = TOK_IN_GENERIC_VALUE_EXPRESSION;             // ★ 折叠点 4 → "IN (...)"
        }
      } else if (last_token == TOK_GENERIC_VALUE_LIST && last_token2 == '(') {
        /*
          REDUCE:
          TOK_ROW_MULTIPLE_VALUE := '(' TOK_GENERIC_VALUE_LIST ')'
        */
        digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
        token = TOK_ROW_MULTIPLE_VALUE;                        // ★ 折叠点 5 → "(...)"

        /* Read last two tokens again */
        peek_last_two_tokens(digest_storage, state->m_last_id_index,
                             &last_token, &last_token2);

        if ((last_token2 == TOK_ROW_MULTIPLE_VALUE ||
             last_token2 == TOK_ROW_MULTIPLE_VALUE_LIST) &&
            (last_token == ',')) {
          digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
          token = TOK_ROW_MULTIPLE_VALUE_LIST;                 // ★ 折叠点 6 → "(...) /* , ... */"
        } else if (last_token == IN_SYM) {
          /*
          REDUCE:
          TOK_IN_GENERIC_VALUE_EXPRESSION := IN_SYM TOK_ROW_MULTIPLE_VALUE
          */
          digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;
          token = TOK_IN_GENERIC_VALUE_EXPRESSION;
        }
      }
      store_token(digest_storage, token);
      break;
    }
    case IDENT:
    case IDENT_QUOTED:
    case TOK_IDENT_AT: {
      Lexer_yystype *lex_token = yylval;
      char *yytext = lex_token->lex_str.str;
      size_t yylen = lex_token->lex_str.length;

      /*
        REDUCE:
          TOK_IDENT := IDENT | IDENT_QUOTED
        The parser gives IDENT or IDENT_TOKEN for the same text,
        depending on the character set used.
        We unify both to always print the same digest text,
        and always have the same digest hash.
      */
      if (token != TOK_IDENT_AT) {
        token = TOK_IDENT;     // ★ 加不加反引号不影响 DIGEST（统一成 TOK_IDENT）
      }
      store_token_identifier(digest_storage, token, yylen, yytext);

      /* Update the index of last identifier found. */
      state->m_last_id_index = digest_storage->m_byte_count;  // ★ 护栏推进
      break;
    }
    case 0: {
      if (digest_storage->m_byte_count < SIZE_OF_A_TOKEN) break;
      unsigned int temp_tok;
      read_token(digest_storage, digest_storage->m_byte_count - SIZE_OF_A_TOKEN,
                 &temp_tok);
      if (temp_tok == ';') {
        digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;   // ★ 弹掉尾部分号
      }
      break;
    }
    default: {
      store_token(digest_storage, token);   // 关键字/运算符：原样入流，无归约
      break;
    }
  }

  return state;
}
```

> ★ **六个折叠点全在这一个函数里**（`)` 分支尤其密集：它连调两次 `peek_last_two_tokens`，所以才能做"二级折叠"）。`compute_digest_text()` **不做任何折叠**，只负责把已折叠好的 token 查表打印出来——这点常被说反。

---

### 1.8 归一化的实现：就地归约状态机

前面 1.1–1.7 讲的是"规则"（归约成什么），本节讲"机制"（怎么归约）。这是整个 digest 最容易讲浅的地方——**它不是"先攒下全部 token 再统一处理"，而是每来一个 token 就回看已写入的字节，命中规则就把写游标退回去重写**。

**"就地"的含义是回退写游标**：

```cpp
digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;   // 退 2 字节
store_token(digest_storage, token);                // 再写归约后的 token
```

`SIZE_OF_A_TOKEN` 是 2。`m_byte_count` 既是长度也是写游标，所以"归约"= 把游标往回拨，再覆盖写。

#### 回看的护栏：`m_last_id_index`

标识符是变长存储（token + 长度 + 原文），"回退 2 字节拿到上一个 token"这个假设对它不成立。所以标识符写完后记下位置，所有 peek 都不许越过它：

```cpp
// sql/sql_digest_stream.h:36
/**
  State data storage for @c digest_start, @c digest_add_token.
  This structure extends the @c sql_digest_storage structure
  with temporary state used only during parsing.
*/
struct sql_digest_state {
  /**
    Index, in the digest token array, of the last identifier seen.
    Reduce rules used in the digest computation can not
    apply to tokens seen before an identifier.
    @sa digest_add_token
  */
  int m_last_id_index;
  sql_digest_storage m_digest_storage;

  inline void reset(unsigned char *token_array, uint length) {
    m_last_id_index = 0;
    m_digest_storage.reset(token_array, length);
  }

  inline bool is_empty() { return m_digest_storage.is_empty(); }
};
```

`peek_last_two_tokens()` 的每个分支都以 `last_id_index + SIZE_OF_A_TOKEN <= peek_index` 为条件，否则返回 `TOK_UNUSED`。**完整源码**：

```cpp
// sql/sql_digest.cc:304
static inline uint peek_token(const sql_digest_storage *digest, uint index) {
  uint token;
  assert(index + SIZE_OF_A_TOKEN <= digest->m_byte_count);
  assert(digest->m_byte_count <= digest->m_token_array_length);

  token =
      ((digest->m_token_array[index + 1]) << 8) | digest->m_token_array[index];
  return token;
}

// sql/sql_digest.cc:318
/**
  Function to read last two tokens from token array. If an identifier
  is found, do not look for token before that.
*/
static inline void peek_last_two_tokens(
    const sql_digest_storage *digest_storage, uint last_id_index, uint *t1,
    uint *t2) {
  uint byte_count = digest_storage->m_byte_count;
  uint peek_index = byte_count;

  if (last_id_index + SIZE_OF_A_TOKEN <= peek_index) {
    /* Take last token. */
    peek_index -= SIZE_OF_A_TOKEN;
    *t1 = peek_token(digest_storage, peek_index);

    if (last_id_index + SIZE_OF_A_TOKEN <= peek_index) {
      /* Take 2nd token from last. */
      peek_index -= SIZE_OF_A_TOKEN;
      *t2 = peek_token(digest_storage, peek_index);
    } else {
      *t2 = TOK_UNUSED;      // ★ 再往前就碰到标识符边界了
    }
  } else {
    *t1 = TOK_UNUSED;
    *t2 = TOK_UNUSED;
  }
}
```

> ★★ **护栏的真正作用**：`m_last_id_index` 只在 `IDENT` 分支被推进到当前 `m_byte_count`（`sql_digest.cc:598`）。所以 `(a-1)` 里读到 `-` 要回看时，`last_id_index` 已经卡在 `a` 之后——**`-` 后面那个位置之前不允许再看**，于是 `t2` 拿到 `TOK_UNUSED`，`m_start_expr` 为 false，`-` 被判为二元。这就是"一元吸收、二元保留"能成立的结构性保证，而不只是靠一张表。

三 token 版（`digest_reduce_token` 专用，`sql_digest.cc:346`）：

```cpp
static inline void peek_last_three_tokens(
    const sql_digest_storage *digest_storage, uint last_id_index, uint *t1,
    uint *t2, uint *t3) {
  uint byte_count = digest_storage->m_byte_count;
  uint peek_index = byte_count;

  if (last_id_index + SIZE_OF_A_TOKEN <= peek_index) {
    peek_index -= SIZE_OF_A_TOKEN;
    *t1 = peek_token(digest_storage, peek_index);

    if (last_id_index + SIZE_OF_A_TOKEN <= peek_index) {
      peek_index -= SIZE_OF_A_TOKEN;
      *t2 = peek_token(digest_storage, peek_index);

      if (last_id_index + SIZE_OF_A_TOKEN <= peek_index) {
        peek_index -= SIZE_OF_A_TOKEN;
        *t3 = peek_token(digest_storage, peek_index);
      } else {
        *t3 = TOK_UNUSED;
      }
    } else {
      *t2 = TOK_UNUSED;
      *t3 = TOK_UNUSED;
    }
  } else {
    *t1 = TOK_UNUSED;
    *t2 = TOK_UNUSED;
    *t3 = TOK_UNUSED;
  }
}
```

#### 七处回退（这是"就地归约"的全部形态）

| # | 触发 | 回退量 | 归约结果 |
|---|---|---|---|
| 1 | 数字字面量前是 `+`/`-` 且前一个 token 的 `m_start_expr` 为真 | `1×2`（**循环**） | 一元符号被吸收 |
| 2 | 字面量前是 `VALUE` 或 `VALUE_LIST` 加 `,` | `2×2` | `TOK_GENERIC_VALUE_LIST` |
| 3 | `)` 前是 `VALUE` + `(` | `2×2` | `TOK_ROW_SINGLE_VALUE` |
| 4 | 上一步后再 peek，前是 `ROW_*_VALUE(_LIST)` + `,` | `2×2` | `TOK_ROW_SINGLE_VALUE_LIST` |
| 5 | 上一步后再 peek，前是 `IN_SYM` | `1×2` | `TOK_IN_GENERIC_VALUE_EXPRESSION` |
| 6 | `NULL_SYM` 被 yacc 判定为字面量（1.6 的补刀） | `1×2` 或 `2×2` | `TOK_GENERIC_VALUE` |
| 7 | 收到 `token == 0`（语句结束）且末 token 是 `;` | `1×2` | 去掉尾部分号 |

**第 1 条是唯一带循环的**——处理 `--1`、`+-+1` 这类连续一元符号：

```cpp
bool found_unary;
do {
  found_unary = false;
  peek_last_two_tokens(digest_storage, state->m_last_id_index,
                       &last_token, &last_token2);
  if ((last_token == '-') || (last_token == '+')) {
    if (lex_token_array[last_token2].m_start_expr) {
      token = TOK_GENERIC_VALUE;
      digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;
      found_unary = true;
    }
  }
} while (found_unary);
```

每轮吃掉一个一元符号（游标退 2），再回看新的前两个 token，直到前一个非符号 token 的 `m_start_expr` 为 false。这就是 1.2 那条"一元吸收、二元保留"的落地方式——**判据是 `lex_token_array[last_token2].m_start_expr`，一张构建期生成的表**（见 3.4）。

**第 3–5 条的两级回退**最值得看：`)` 分支里 `peek_last_two_tokens()` 被调用了**两次**——第一次判断能否折叠成 `ROW_*_VALUE`，折叠后必须重新回看，才能判断列表或 `IN` 的二级折叠。

`IN (1,2,3)` 的完整字节流演化：

```
IN_SYM ( 1  →  IN_SYM '(' TOK_GENERIC_VALUE
     , 2    →  IN_SYM '(' TOK_GENERIC_VALUE ',' TOK_GENERIC_VALUE
             → 回退 4 字节 → IN_SYM '(' TOK_GENERIC_VALUE_LIST
     , 3    →  IN_SYM '(' TOK_GENERIC_VALUE_LIST ',' TOK_GENERIC_VALUE
             → 回退 4 字节 → IN_SYM '(' TOK_GENERIC_VALUE_LIST
     )      →  回退 4 字节 → IN_SYM TOK_ROW_MULTIPLE_VALUE
             →  再回退 2 字节（吃掉 IN_SYM）→ TOK_IN_GENERIC_VALUE_EXPRESSION
```

最终流里只剩 **1 个 token**，打印为 `IN (...)`。注意最后一步只退 `SIZE_OF_A_TOKEN`：`TOK_ROW_*_VALUE` 是在 `)` 这一步就地改写出来的，不再额外占位。

#### 第 6 条：归约时目标不在栈顶怎么办

`digest_reduce_token()`（yacc 侧主动调用，唯一调用点是 `null_as_literal` 规则）面对的情况是：归约发生时 `NULL_SYM` **不一定是最后一个 token**（如 `SELECT NULL IS NULL` 里 `IS` 已先写入）。解法是弹出来、归约、再压回去：

```cpp
  } else {
    assert(last_token2 == token_right);
    digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
    store_token(digest_storage, token_left);
    token_to_push = last_token;      // 把 TOKEN_Y 暂存
  }
  ...
  if (token_to_push != TOK_UNUSED) {
    store_token(digest_storage, token_to_push);   // 再压回去
  }
```

**`digest_reduce_token()` 完整源码**（`sql/sql_digest.cc:623-704`）：

```cpp
sql_digest_state *digest_reduce_token(sql_digest_state *state, uint token_left,
                                      uint token_right) {
  sql_digest_storage *digest_storage = nullptr;

  digest_storage = &state->m_digest_storage;

  /*
    Stop collecting further tokens if digest storage is full.
  */
  if (digest_storage->m_full) {
    return nullptr;
  }

  uint last_token;
  uint last_token2;
  uint last_token3;
  uint token_to_push = TOK_UNUSED;

  peek_last_two_tokens(digest_storage, state->m_last_id_index, &last_token,
                       &last_token2);

  /*
    There is only one caller of digest_reduce_token(),
    see sql/sql_yacc.yy, rule literal := NULL_SYM.
    REDUCE:
      token_left := token_right
    Used for:
      TOK_GENERIC_VALUE := NULL_SYM
  */

  if (last_token == token_right) {
    /*
      Current stream is like:
        TOKEN_X TOKEN_RIGHT .
      REDUCE to
        TOKEN_X TOKEN_LEFT .
    */
    digest_storage->m_byte_count -= SIZE_OF_A_TOKEN;
    store_token(digest_storage, token_left);        // ★ NULL 就在末尾：直接替换
  } else {
    /*
      Current stream is like:
        TOKEN_X TOKEN_RIGHT TOKEN_Y .
      Pop TOKEN_Y
        TOKEN_X TOKEN_RIGHT . TOKEN_Y
      REDUCE to
        TOKEN_X TOKEN_LEFT . TOKEN_Y
    */
    assert(last_token2 == token_right);
    digest_storage->m_byte_count -= 2 * SIZE_OF_A_TOKEN;
    store_token(digest_storage, token_left);
    token_to_push = last_token;                     // ★ 暂存 TOKEN_Y
  }

  peek_last_three_tokens(digest_storage, state->m_last_id_index, &last_token,
                         &last_token2, &last_token3);

  if ((last_token3 == TOK_GENERIC_VALUE ||
       last_token3 == TOK_GENERIC_VALUE_LIST) &&
      (last_token2 == ',') && (last_token == TOK_GENERIC_VALUE)) {
    /*
      REDUCE:
      TOK_GENERIC_VALUE_LIST := TOK_GENERIC_VALUE ',' TOK_GENERIC_VALUE
      REDUCE:
      TOK_GENERIC_VALUE_LIST := TOK_GENERIC_VALUE_LIST ',' TOK_GENERIC_VALUE
    */
    digest_storage->m_byte_count -= 3 * SIZE_OF_A_TOKEN;
    store_token(digest_storage, TOK_GENERIC_VALUE_LIST);
    // ★ 只有 yacc 侧主动归约才做得到：NULL 不是词法器的 NUM/TEXT_STRING
  }

  if (token_to_push != TOK_UNUSED) {
    store_token(digest_storage, token_to_push);     // 再压回去
  }

  return state;
}
```

`sql_yacc.yy` 里那个**唯一调用点**（`sql/sql_yacc.yy:14727`，`literal: NULL_SYM`）：

```yacc
            /*
              For the digest computation, in this context only,
              NULL is considered a literal, hence reduced to '?'
              REDUCE:
                TOK_GENERIC_VALUE := NULL_SYM
            */
            lip->reduce_digest_token(TOK_GENERIC_VALUE, NULL_SYM);
            $$= NEW_PTN Item_null(@$);
          }
        ;
```

#### 缓冲满 = 停止收集，不是打省略号

`store_token()` / `store_token_identifier()` 放不下就置 `m_full = true`，**之后 `digest_add_token()` 直接 `return nullptr`**：

```cpp
if (digest_storage->m_full || token == END_OF_INPUT) {
  return nullptr;
}
```

而调用方会把 `nullptr` 写回 `m_digest`，从此整条语句的收集**彻底关闭**：

```cpp
void Lex_input_stream::add_digest_token(uint token, Lexer_yystype *yylval) {
  if (m_digest != nullptr) {
    m_digest = digest_add_token(m_digest, token, yylval);
  }
}
```

⚠️ 注意：`"..."`、`/* , ... */` 这类省略号标记**只来自 `TOK_*_LIST` 归约**，不代表缓冲截断。MTR 有硬证据——`--max-digest-length=2`（只够装第一个 token）下：

```
SELECT statement_digest( 'SELECT 1' ) = statement_digest( 'SELECT 1 FROM DUAL' );
→ 1
```

即 `SELECT 1` 与 `SELECT 1 FROM DUAL` 的 digest **完全相同**（字节流都只剩 `SELECT`）。这是"停止收集"而非"文本截断"的决定性证据。

---

## 二、token 收集机制

### 2.1 边解析边收集 + 就地归约

`MYSQLlex()` → `lex_one_token()` 每返回一个 token 前统一写入：

```cpp
if (!lip->skip_digest) lip->add_digest_token(token, yylval);
lip->skip_digest = false;
```

写入格式是紧凑字节流：

```
<non-id-token> <non-id-token> <id-token> <id_len> <id_text> ...
例如：SELECT * FROM T1;  →  <SELECT_TOKEN> <*> <FROM_TOKEN> <ID_TOKEN> <2> <T1>
```

`digest_add_token` 收到每个 token 时立即 peek 栈顶 2~3 个 token 做局部归约（`TOK_GENERIC_VALUE_LIST := TOK_GENERIC_VALUE ',' TOK_GENERIC_VALUE` 等），**从不对原始文本做全文重写**。`m_digest == nullptr` 时完全零开销（不收集）。

### 2.2 skip_digest 的唯一场景：hint

不是"跳过 hint"，而是"hint 已由 hint parser 内部写入，避免外层重复/乱序"：

```cpp
/*
  ...the "/*+ HINT(t) */" is a sequence of separate tokens from the hint
  parser's point of view, and we add those tokens to the digest buffer
  *inside* the lex_one_token() call. Thus, the usual data flow adds
  tokens from the "/*+ HINT(t) */" string first, and only than it appends
  the "SELECT" keyword token to that stream: "/*+ HINT(t) */ SELECT".
  This is not acceptable... the order of added tokens is important.
*/
```

hint 内部 token（`/*+` 写 `TOK_HINT_COMMENT_OPEN`，hint 参数里的数字/文本照常归一化成 `?`）由 `Hint_scanner::add_hint_token_digest()` 写入。**所以不同 hint 会改变 digest**（hint 是语句形状的一部分）。

**`skip_digest` 完整声明**（`sql/sql_lex.h:3608`，注释即上面那段原文）：

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

  void add_digest_token(uint token, Lexer_yystype *yylval);
  void reduce_digest_token(uint token_left, uint token_right);
```

**置位处**（`sql/sql_lex.cc:905`，`find_keyword()`）：

```cpp
static int find_keyword(Lex_input_stream *lip, uint len, bool function) {
  const char *tok = lip->get_tok_start();

  const SYMBOL *symbol =
      function ? Lex_hash::sql_keywords_and_funcs.get_hash_symbol(tok, len)
               : Lex_hash::sql_keywords.get_hash_symbol(tok, len);

  if (symbol) {
    lip->yylval->keyword.symbol = symbol;
    lip->yylval->keyword.str = const_cast<char *>(tok);
    lip->yylval->keyword.length = len;

    if ((symbol->tok == NOT_SYM) &&
        (lip->m_thd->variables.sql_mode & MODE_HIGH_NOT_PRECEDENCE))
      return NOT2_SYM;
    if ((symbol->tok == OR_OR_SYM) &&
        !(lip->m_thd->variables.sql_mode & MODE_PIPES_AS_CONCAT)) {
      push_deprecated_warn(lip->m_thd, "|| as a synonym for OR", "OR");
      return OR2_SYM;
    }

    lip->yylval->optimizer_hints = nullptr;
    if (symbol->group & SG_HINTABLE_KEYWORDS) {
      lip->add_digest_token(symbol->tok, lip->yylval);   // ★ 先写 "SELECT"，保证顺序正确
      if (consume_optimizer_hints(lip)) return ABORT_SYM; // ★ hint parser 在内部写 /*+ ... */
      lip->skip_digest = true;                           // ★★ 置位：MYSQLlex 这次别再写一遍
    }

    return symbol->tok;
  }
  return 0;
}
```

**消费处**（`sql/sql_lex.cc:1367`，`MYSQLlex()` 尾部）：

```cpp
  yylloc->cpp.end = lip->get_cpp_ptr();
  yylloc->raw.end = lip->get_ptr();
  if (!lip->skip_digest) lip->add_digest_token(token, yylval);
  lip->skip_digest = false;                     // ★ 一次性标志，读完立刻清零
  return token;
}
```

hint 扫描器把 digest 状态交回主 lexer（`sql/sql_lex.cc:896`）：

```cpp
    lip->yylineno = hint_scanner.get_lineno();
    lip->yySkipn(hint_scanner.get_ptr() - lip->get_ptr());
    lip->yylval->optimizer_hints = hint_list;   // NULL in case of syntax error
    lip->m_digest = hint_scanner.get_digest();  // NULL is digest buf. is full.
    return false;
```

> ⚠️ **不存在 `SKIP_DIGEST` 宏**——只有 `Lex_input_stream::skip_digest` 这个 bool 成员。

### 2.3 预编译参数

`stmt_prepare_mode` 下 `?` 返回 `PARAM_MARKER`，它在 `digest_add_token` 里同样归为 `TOK_GENERIC_VALUE` → 打印 `?`。即**预编译参数与字面量在 digest 中同形**。PREPARE 阶段同样完整算 digest（`sql/sql_prepare.cc`）。

### 2.4 收集时机

- **token 收集**：解析时（逐 token）
- **哈希计算**：解析成功后（`MYSQL_DIGEST_END` → `compute_digest_hash`）
- **DIGEST_TEXT 生成**：懒计算，仅读 PFS 表行时（`PFS_digest_row::make_row`）

---

## 三、哈希与存储

### 3.1 SHA-256，输入是 token 字节流

```cpp
void compute_digest_hash(const sql_digest_storage *digest_storage,
                         unsigned char *hash) {
  static_assert(DIGEST_HASH_SIZE == SHA256_DIGEST_LENGTH, ...);
  SHA_EVP256(digest_storage->m_token_array, digest_storage->m_byte_count, hash);
}
```

- 输入是**归一化后的 token 字节数组**（不是归一化文本字符串）
- 算法沿革：MD5（5.7 之前，非 FIPS 弃用）→ **SHA-256**（8.0 起，FIPS 合规）

### 3.2 存储格式

**全部宏与结构定义**（`sql/sql_digest.h:34` / `:94`）：

```cpp
#define MAX_DIGEST_STORAGE_SIZE (1024 * 1024)   // token 数组绝对上限 1MB

/**
  Write SHA-256 hash value in a string to be used as DIGEST for the statement.
*/
#define DIGEST_HASH_TO_STRING(_hash, _str)                                    \
  (void)sprintf(_str,                                                         \
                "%02x%02x%02x%02x%02x%02x%02x%02x"                            \
                ... /* 32 个 %02x */ ...                                       \
                _hash[0], _hash[1], ... _hash[31])

/// SHA-256 = 32 bytes of binary = 64 printable characters.
#define DIGEST_HASH_TO_STRING_LENGTH 64

/*
  Various hashes considered for digests.   ← ★ 哈希算法"选型史"注释原文
  MD5:     128 bits, used up to MySQL 5.7, abandoned in MySQL 8.0, non FIPS compliant.
  SHA1:    160 bits, non FIPS compliant in strict mode, not used
  SHA2-224:224 bits, non FIPS compliant in strict mode, not used
  SHA2-256:256 bits, FIPS compliant, Used starting with MySQL 8.0
  SHA2-384:384 bits
  SHA2-512:512 bits
*/

/**
  DIGEST hash size, in bytes. 256 bits, for SHA256.
*/
#define DIGEST_HASH_SIZE 32                      // ★ 不是 5.7 的 16 字节 MD5

ulong get_max_digest_length();                   // 对应 sql_digest.cc:49 的全局 max_digest_length
```

```cpp
// sql/sql_digest.h:94
/**
  Structure to store token count/array for a statement
  on which digest is to be calculated.
*/
struct sql_digest_storage {
  bool m_full;                    // ★ buffer 满则停止收集（digest_add_token 立刻 return nullptr）
  size_t m_byte_count;            // ★ 既是长度也是"写指针"；归约时直接减它来弹出 token
  unsigned char m_hash[DIGEST_HASH_SIZE];
  /** Character set number. */
  uint m_charset_number;          // ★ compute_digest_text 靠它 get_charset() 做标识符转码
  /**
    Token array.
    Token array is an array of bytes to store tokens received during parsing.
    Following is the way token array is formed.
    ... <non-id-token> <non-id-token> <id-token>
    <id_len> <id_text> ... For Example: SELECT * FROM T1;
    <SELECT_TOKEN> <*> <FROM_TOKEN> <ID_TOKEN> <2> <T1>

    @note Only the first @c m_byte_count bytes are initialized,
      out of @c m_token_array_length.
  */
  unsigned char *m_token_array;   // 字节流本体
  /* Length of the token array to be considered for DIGEST_TEXT calculation. */
  size_t m_token_array_length;

  sql_digest_storage() { reset(nullptr, 0); }

  inline void reset(unsigned char *token_array, size_t length) {
    m_token_array = token_array;
    m_token_array_length = length;
    reset();
  }

  inline void reset() {
    m_full = false;
    m_byte_count = 0;
    m_charset_number = 0;
    memset(m_hash, 0, DIGEST_HASH_SIZE);
  }

  inline bool is_empty() { return (m_byte_count == 0); }

  inline void copy(const sql_digest_storage *from) {
    /*
      Keep in mind this is a dirty copy of something that may change,
      as the thread producing the digest is executing concurrently,
      without any lock enforced.
    */
    size_t byte_count_copy = m_token_array_length < from->m_byte_count
                                 ? m_token_array_length
                                 : from->m_byte_count;   // ★ 按目标容量截断
    if (byte_count_copy > 0) {
      m_full = from->m_full;
      m_byte_count = byte_count_copy;
      m_charset_number = from->m_charset_number;
      memcpy(m_token_array, from->m_token_array, m_byte_count);
      memcpy(m_hash, from->m_hash, DIGEST_HASH_SIZE);
    } else {
      m_full = false;
      m_byte_count = 0;
      m_charset_number = 0;
    }
  }
};
```

- `performance_schema` 表里 `DIGEST VARCHAR(64)`（64 字符 hex），`DIGEST_TEXT LONGTEXT`
- 字符集：解析时记 `m_charset_number`，重放 TEXT 时把标识符从原字符集转成 utf8mb3 再打印
- ⚠️ **不存在 `Token_Digest_Store` 类型**（全仓库 0 命中），8.0.39 只有 `sql_digest_storage` 与 `sql_digest_state`
- ⚠️ `SIZE_OF_A_TOKEN`（=2）定义在 **`sql/sql_digest.cc:47`**，不在头文件里

`THD` 与 `Lex_input_stream` 侧的 `m_digest` 声明：

```cpp
// sql/sql_class.h:2409
  /** Current statement digest. */
  sql_digest_state *m_digest;              // ★ 嵌套解析（SP/trigger/PREPARE）会临时换掉再恢复
  /** Current statement digest token array. */
  unsigned char *m_token_array;            // ★ 真正的字节流缓冲区，由 PFS 分配
  /** Top level statement digest. */
  sql_digest_state m_digest_state;         // 顶层语句那一份（内嵌对象）

// sql/sql_lex.h:3773
  sql_digest_state *m_digest;
```

绑定处（`sql/sql_parse.cc:7117`）——注意字符集就是在这里记的：

```cpp
        parser_state->m_lip.m_digest = thd->m_digest;
        parser_state->m_lip.m_digest->m_digest_storage.m_charset_number =
            thd->charset()->number;
```

#### 3.2.1 字节流的实际编码

源码注释给的格式说明：

```
... <non-id-token> <non-id-token> <id-token> <id_len> <id_text> ...
例如：SELECT * FROM T1;
<SELECT_TOKEN> <*> <FROM_TOKEN> <ID_TOKEN> <2> <T1>
```

具体编码规则（两条，都由 `store_token*` 决定）：

| 元素 | 编码 | 占字节 |
|---|---|---|
| 普通 token | **固定 2 字节，小端**（`dest[0]=tok&0xff; dest[1]=(tok>>8)&0xff`） | 2 |
| 标识符 | token(2) + **长度前缀 2 字节小端** + 原文裸字节（不补 NUL） | `4 + len` |

所以 token **不是变长编码**——这保证了 1.8 里"回退 2 字节"总是安全的（只要不越过标识符边界，而那正是 `m_last_id_index` 的用途）。

**四个读写原语的完整源码**（`sql/sql_digest.cc:52-157`）——这就是上面两条编码规则的全部实现：

```cpp
// sql/sql_digest.cc:44 —— 必须定义它，lex_token.h 里的 lex_token_array[] 才带出定义
#define LEX_TOKEN_WITH_DEFINITION
#include "sql/lex_token.h"

#define SIZE_OF_A_TOKEN 2                         // ★ 定义在 .cc 里，不在 .h 里

ulong max_digest_length = 0;                      // PFS 启动时写入
ulong get_max_digest_length() { return max_digest_length; }

// :52 读一个 token
inline uint read_token(const sql_digest_storage *digest_storage, uint index,
                       uint *tok) {
  size_t safe_byte_count = digest_storage->m_byte_count;   // ★ 先快照：并发读时可能被别的线程改动

  if (index + SIZE_OF_A_TOKEN <= safe_byte_count &&
      safe_byte_count <= digest_storage->m_token_array_length) {   // ★ 双重护栏
    const unsigned char *src = &digest_storage->m_token_array[index];
    *tok = src[0] | (src[1] << 8);                        // 小端拼回 uint16
    return index + SIZE_OF_A_TOKEN;
  }

  /* The input byte stream is exhausted. */
  *tok = 0;
  return MAX_DIGEST_STORAGE_SIZE + 1;                     // ★ 哨兵：任何合法索引都不可能 > 1MB
}

// :71 写一个 token
inline void store_token(sql_digest_storage *digest_storage, uint token) {
  /* WRITE: ok to assert, storing a token is race free. */
  assert(digest_storage->m_byte_count <= digest_storage->m_token_array_length);

  if (digest_storage->m_byte_count + SIZE_OF_A_TOKEN <=
      digest_storage->m_token_array_length) {
    unsigned char *dest =
        &digest_storage->m_token_array[digest_storage->m_byte_count];
    dest[0] = token & 0xff;
    dest[1] = (token >> 8) & 0xff;
    digest_storage->m_byte_count += SIZE_OF_A_TOKEN;
  } else {
    digest_storage->m_full = true;   // ★ 写不下不打错，只置 m_full
  }
}

// :90 读标识符
inline uint read_identifier(const sql_digest_storage *digest_storage,
                            uint index, const char **id_string,
                            int *id_length) {
  uint new_index;
  uint safe_byte_count = digest_storage->m_byte_count;

  /* READ: never assert on data, reading can be racy when used concurrently
   * (pfs). */
  /*
    token + length + string are written in an atomic way,
    so we do always expect a length + string here
  */
  uint bytes_needed = SIZE_OF_A_TOKEN;                 // 标识符布局：[len:2][text:N]
  if ((safe_byte_count <= digest_storage->m_token_array_length) &&
      (index + bytes_needed) <= safe_byte_count) {
    const unsigned char *src = &digest_storage->m_token_array[index];
    uint length = src[0] | (src[1] << 8);
    bytes_needed += length;
    if ((index + bytes_needed) <= safe_byte_count) {
      *id_string = pointer_cast<const char *>(src) + 2;  // ★ 跳过 2 字节长度，零拷贝
      *id_length = length;
      new_index = index + bytes_needed;
      return new_index;
    }
  }
  return MAX_DIGEST_STORAGE_SIZE + 1;
}

// :129 写标识符
inline void store_token_identifier(sql_digest_storage *digest_storage,
                                   uint token, size_t id_length,
                                   const char *id_name) {
  assert(digest_storage->m_byte_count <= digest_storage->m_token_array_length);

  size_t bytes_needed = 2 * SIZE_OF_A_TOKEN + id_length;   // 4 + N
  if (digest_storage->m_byte_count + bytes_needed <=
      (unsigned int)digest_storage->m_token_array_length) {
    unsigned char *dest =
        &digest_storage->m_token_array[digest_storage->m_byte_count];
    dest[0] = token & 0xff;
    dest[1] = (token >> 8) & 0xff;
    dest[2] = id_length & 0xff;       // 长度前缀，小端
    dest[3] = (id_length >> 8) & 0xff;
    if (id_length > 0) {
      memcpy((char *)(dest + 4), id_name, id_length);
    }
    digest_storage->m_byte_count += bytes_needed;
  } else {
    digest_storage->m_full = true;
  }
}
```

> ★ 注意读写两侧的**不对称**：`store_*` 可以 `assert`（"WRITE: ok to assert, storing a token is race free"），而 `read_*` 一律不 assert——*"READ: never assert on data, reading can be racy when used concurrently (pfs)"*。这是 PFS 脏读的源码级自白。

#### 3.2.2 一个完整例子：`SELECT a FROM t WHERE b=1`

token 值取自 yacc 的显式声明（`%token FROM 452`、`IDENT 482`、`SELECT_SYM 748`、`WHERE 890` …），fake token 从 1100 起由 `gen_lex_token` 按声明顺序分配（`TOK_GENERIC_VALUE`=1100、`TOK_IDENT`=1106 …）。

归一化后的完整字节数组（`m_byte_count = 31`）：

```
偏移   字节                    含义
----   ----------------------  ------------------------------------------
0x00   EC 02                   SELECT_SYM        = 748
0x02   52 04  01 00  61        TOK_IDENT(1106) + len=1 + "a"    (7 bytes)
0x09   C4 01                   FROM              = 452
0x0B   52 04  01 00  74        TOK_IDENT(1106) + len=1 + "t"    (7 bytes)
0x12   7A 03                   WHERE             = 890
0x14   52 04  01 00  62        TOK_IDENT(1106) + len=1 + "b"    (7 bytes)
0x1B   3D 00                   '='               = 61
0x1D   4C 04                   TOK_GENERIC_VALUE = 1100  （原 NUM "1"）
```

共 31 字节，其余 993 字节（默认 1024）是未初始化垃圾，**不参与哈希**。对应 `DIGEST_TEXT`：

```
SELECT `a` FROM `t` WHERE `b` = ?
```

（标识符一律加反引号，运算符前后带空格——空格规则见 3.4.3。）

MTR 实测对照（`func_digest.result`）：

```
SELECT `a` + `b` , `a` - `b` FROM `t1` , `t2` , `t3` WHERE `a` = `c`
```

注意 `, ` 的形态——逗号**前后都有空格**。

### 3.3 digest 值是被刻意冻结的 ABI

★ **这是本篇与 [`04_parser.md`](04_parser.md) 的交叉点**：`sql/sql_yacc.yy:537` 那段 `MAINTAINER:` 注释说的"token 值进了 digest、新 token 只能追加在末尾、由 `gen_lex_token` 强制执行"，执行者就是下面这个文件和它的 `static_assert`。

**硬断言原文**（`sql/gen_lex_token.cc:238`）：

```cpp
static_assert(YYUNDEF == 1150,
              "YYUNDEF must be stable, because raw token numbers are used in "
              "PFS digest calculations");
```

> 注意措辞 **"raw token numbers"**——没有映射层、没有版本化间接，编号本身就是哈希输入的一对字节。

#### 3.3.1 编号空间的"分区宪法"（`sql/gen_lex_token.cc:33-86` 注释全文）

```cpp
/*
  MAINTAINER:

  Tokens printed in sql/lex_token.h do come from several sources:
  - tokens from sql_yacc.yy
  - tokens from sql_hints.yy
  - fake tokens for digests.

  All the token values are mapped in the same space,
  indexed by the token value directly.

  To account for enhancements and new tokens,
  gap are created, so that adding a token from one source
  does not change values of tokens from other sources.

  This is done to ensure stability in digest computed values.

  As of now (8.0.0), the mapping looks like this:
  - PART 1: [0 .. 255] tokens of single-character lexemes
  - PART 2: [256 .. ...] tokens < YYUNDEF from sql_yacc.yy
  - PART 3: [... .. 999] reserved for sql_yacc.yy new tokens < YYUNDEF
  - PART 4: [1000 .. ...] tokens from sql_hints.yy
  - PART 5: [... .. 1099] reserved for sql_hints.yy new tokens
  - PART 6: [1100 .. ...] digest special fake tokens
  - PART 7: [... .. 1149] reserved for new digest special fake tokens
  - PART 8: [1150 .. ...] tokens > YYUNDEF from sql_yacc.yy

  Should gen_lex_token fail when tokens are exhausted
  (maybe you are reading this comment because of a fprintf(stderr) below),
  the options are as follows, by order of decreasing desirability:

  1) Reuse OBSOLETE_TOKEN_XXX instead of consuming new token values

  2) Consider if you really need to create a new token,
  instead of re using an existing one.

  Keep in mind that syntax sugar in the parser still adds
  to complexity, by making the parser tables bigger,
  so adding tokens all the time is not a good practice.

  3) Expand boundary values for
     - range_for_sql_hints
     - range_for_digests
  and record again all the MTR tests that print a DIGEST,
  because DIGEST values have now changed.

  While at it, because digests have changed anyway,
  please seriously consider to clean up and reorder:
  - all the tokens in sql/sql_yacc.yy in one nice list,
  ordered alphabetically, removing obsolete values if any.
  - likewise for sql/sql_hints.yy
*/
```

> ⚠️ **勘误**：只有 **PART 1..PART 8**。文档/资料里常见的"PART 9"只是 `print_tokens()` 输出到末尾时的一句哨兵注释 `/* PART 9: End of token list. */`（`:460`），**不是一个编号区间**。

#### 3.3.2 `range` 类：编号配额的执行者（`sql/gen_lex_token.cc:149`）

```cpp
struct range {
  range(const char *title, int start, int end)
      : title{title}, start{start}, end{end}, max_seen{0} {}

  void set_token(int tok, const char *str, int line) {
    if (tok <= 0) {
      fprintf(stderr, "%s:%d: Bad token found\n", __FILE__, line);
      exit(1);
    }

    if (tok > end) {                       // ★★ 越界即 exit(1)：这就是"token 配额耗尽"的报错点
      fprintf(stderr,
              "%s:%d: Token reserve for %s exhausted: %d (should be <= %d).\n"
              "Please see MAINTAINER instructions in sql/gen_lex_token.cc\n",
              __FILE__, line, title, tok, end);
      exit(1);
    }

    if (tok >= MY_MAX_TOKEN) {
      fprintf(stderr,
              "%s:%d: Added that many new keywords ? Increase MY_MAX_TOKEN\n",
              __FILE__, line);
      exit(1);
    }

    if (tok > max_seen) max_seen = tok;

    compiled_token_array[tok].m_initialized = true;
    compiled_token_array[tok].m_token_string = str;
    compiled_token_array[tok].m_token_length = strlen(str);
    compiled_token_array[tok].m_append_space = true;   // ★ 默认补空格
    compiled_token_array[tok].m_start_expr = false;    // ★ 默认不开启表达式
  }

  int add_token(const char *str, int line) {   // digest fake token 走这里：自动 max_seen+1
    set_token(max_seen ? max_seen + 1 : start, str, line);
    return max_seen;
  }

  void print(const char *header1, const char *header2 = nullptr) const {
    puts(header1);
    for (int tok = start; tok <= max_seen; tok++) print_token(tok);
    if (header2 == nullptr) return;
    puts(header2);
    for (int tok = max_seen + 1; tok <= end; tok++)
      printf("/* reserved %03d for %s */  { \"\", 0, false, false},\n", tok, title);
  }
  ...
};
```

四个 range 变量与第二条硬约束（`:238-252`）：

```cpp
static_assert(YYUNDEF == 1150, "YYUNDEF must be stable, ...");
range range_for_sql_yacc2{"sql/sql_yacc.yy (before YYUNDEF)", YYUNDEF, MY_MAX_TOKEN};
                                                   // 实际 1150..1251  → PART 8

range range_for_digests{"digest specials", 1100, range_for_sql_yacc2.start - 1};
                                                   // 实际 1100..1149  → PART 6/7

static_assert(MAX_EXECUTION_TIME_HINT == 1000,
              "MAX_EXECUTION_TIME_HINT should be equal to 1000");   // ★ 第二条硬钉
range range_for_sql_hints{"sql/sql_hints.yy", MAX_EXECUTION_TIME_HINT,
                          range_for_digests.start - 1};             // 实际 1000..1099 → PART 4/5

range range_for_sql_yacc1{"sql/sql_yacc.yy (after YYUNDEF)", 256,
                          range_for_sql_hints.start - 1};           // 实际 256..999 → PART 2/3
```

> ⚠️ **源码的一处笔误**：`range_for_sql_yacc1`（256..999）的 title 写成 `"(after YYUNDEF)"`，但实际区间是 **before YYUNDEF**；`range_for_sql_yacc2`（1150..1251）反过来。数值是对的，**title 字符串写反了**——只影响报错文本，不影响行为。

`MY_MAX_TOKEN = 1251`（`:145`）。第三处硬约束钉住 5.7 的最后一个 token（`sql/gen_lex_token.cc:483`）：

```cpp
/*
  ZEROFILL_SYM is the last token in the MySQL 5.7 token list,
  see sql/sql_yacc.yy
  The token value is frozen and should not change,
  to avoid changing query digest values.
*/
static const int zerofill_expected_value = 906;

static_assert(!(ZEROFILL_SYM < zerofill_expected_value),
              "Token deleted. "
              "Please read MAINTAINER instructions in sql/sql_yacc.yy");
static_assert(!(ZEROFILL_SYM > zerofill_expected_value),
              "Token added in the wrong place. "
              "Please read MAINTAINER instructions in sql/sql_yacc.yy");
```

> ★ 两个方向各一条：**删 token**（编号左移）与**在中间插 token**（编号右移）都会触发。这就是 `OBSOLETE_TOKEN_NNN` 存在的原因——删不得，只能废弃。

`YYUNDEF` 本身在 `sql_yacc.yy:1337` 显式留出 gap：

```yacc
/*
  Here is an intentional gap in token numbers.

  Token numbers starting 1000 till YYUNDEF are occupied by:
  1. hint terminals (see sql_hints.yy),
  2. digest special internal token numbers (see gen_lex_token.cc, PART 6).

  Note: YYUNDEF in internal to Bison. Please don't change its number, or change
  it in sync with YYUNDEF in sql_hints.yy.
*/
%token YYUNDEF 1150                /* INTERNAL (for use in the lexer) */
%token<lexer.keyword> JSON_VALUE_SYM 1151               /* SQL-2016-R */
```

#### 3.3.3 生成流程：三步

```cmake
# sql/CMakeLists.txt:46-90
SET(GEN_DIGEST_SOURCES
  ${CMAKE_CURRENT_BINARY_DIR}/lex_token.h            # 归类为 "digest 源"
)
...
# Gen_lex_token
# Make sure sql_yacc.h is generated before compiling gen_lex_token
MYSQL_ADD_EXECUTABLE(gen_lex_token
  gen_lex_token.cc
  DEPENDENCIES GenServerSource        # ★ 强制先由 bison 生成 sql_yacc.h / sql_hints.yy.h
  SKIP_INSTALL
  )

ADD_CUSTOM_COMMAND(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/lex_token.h
  COMMAND gen_lex_token > lex_token.h   # ★ 靠 shell 重定向生成
  DEPENDS gen_lex_token
)
```

```
① bison 生成 sql_yacc.h（含 #define YYUNDEF 1150 等）与 sql_hints.yy.h
② 编译 gen_lex_token.cc：#include 上述两个头拿【编号】，
   链接 sql/lex.h 的 symbols[] 拿【文本】（sql/gen_lex_token.cc:319-332）
③ 运行 gen_lex_token > lex_token.h：compute_tokens() 填表 → print_tokens() 输出
```

> ⚠️ **常见误解**：`gen_lex_token` **不解析任何文件**。`main(int, char **)`（`:498`）完全忽略 argv，不做 `fgets`/`strstr`。token 编号靠 `#include "sql/sql_yacc.h"`（`:32`）在**编译期**拿到，可读名字来自 `lex.h` 的 `symbols[]`（5.6/5.7 确有解析 `sql_yacc.h` 的旧实现，8.0 已简化）。

**本工作树的构建产物实证**（`clion-build-950-debug/sql/lex_token.h`）：

```c
/* 1202 */  { "URL", 3, true, false},
/* 1203 */  { "GENERATE", 8, true, false},
/* 1204 */  { "TEST_0528", 9, true, false},    /* ← 本地新增的非上游 token，见 04_parser.md */
/* PART 9: End of token list. */
/* DUMMY */ { "", 0, false, false}
};
#endif /* LEX_TOKEN_WITH_DEFINITION */
/* DIGEST specific tokens. */
#define TOK_GENERIC_VALUE 1100
...
#define TOK_UNUSED 1111
```

> ★ `TEST_0528` 落在 **1204**，位于 PART 8（≥ YYUNDEF）且是**追加**，所以 `YYUNDEF` 仍是 1150、既有编号一个没动 ⇒ 三条 `static_assert` 全过 ⇒ 已有 digest 不变。这正是"只能追加不能中间插"的现场演示。

任何"中间插入 / 删除 / 挤压分区边界"的改动都会静默改变**所有**已存 digest 值，需重录全部 MTR 测试。

### 3.4 反解析：从字节流还原 DIGEST_TEXT

哈希是单向的，`DIGEST_TEXT` 是**从 token 字节流重放出来的**——3.2.2 那个 31 字节数组能完整还原成文本。这一步叫 `compute_digest_text()`，是纯线性重放。

#### 3.4.1 `compute_digest_text()` 完整源码（`sql/sql_digest.cc:167-302`）

> ⚠️ **勘误**：常见说法"重放时把 `TOK_*_LIST` 折叠成 `...`"是**错的**——折叠全在 `digest_add_token()` 里完成（见 1.7.1 六个折叠点）。这里只是把**已折叠好的 token** 原样查表输出。

```cpp
/*
  Iterate token array and updates digest_text.
*/
void compute_digest_text(const sql_digest_storage *digest_storage,
                         String *digest_text) {
  assert(digest_storage != nullptr);
  uint byte_count = digest_storage->m_byte_count;
  String *digest_output = digest_text;
  uint tok = 0;
  uint current_byte = 0;
  lex_token_string *tok_data;

  /*
    When a space needs to be appended, set add_space to true,
    and delay actually adding the space until the next token
    is found.
    This is to prevent printing digest text
    with a trailing space character.
  */
  bool add_space = false;                    // ★ 空格规则①：延迟空格

  /* Reset existing data */
  digest_output->length(0);

  if (byte_count > digest_storage->m_token_array_length) {
    digest_output->append("\0", 1);          // ★ 脏读防御：并发下 byte_count 可能暂时越界
    return;
  }

  /* Convert text to utf8 */
  const CHARSET_INFO *from_cs =
      get_charset(digest_storage->m_charset_number, MYF(0));
  const CHARSET_INFO *to_cs = &my_charset_utf8mb3_bin;   // ★ DIGEST_TEXT 固定落在 utf8mb3_bin

  if (from_cs == nullptr) {
    /*
      Can happen, as we do dirty reads on digest_storage,
      which can be written to in another thread.
    */
    digest_output->append("\0", 1);
    return;
  }

  char id_buffer[NAME_LEN + 1] = {'\0'};     // ★ 标识符转码暂存区，上限 NAME_LEN(64) 字符
  const char *id_string;
  size_t id_length;
  bool convert_text = !my_charset_same(from_cs, to_cs);

  while (current_byte < byte_count) {        // ★ 主循环：纯粹的流重放，不做任何归约
    current_byte = read_token(digest_storage, current_byte, &tok);

    if (tok <= 0 || tok >= array_elements(lex_token_array) ||
        current_byte > max_digest_length) {
      return;                                // ★ 直接 return，不加 "..."；已写出的部分保留
    }

    tok_data = &lex_token_array[tok];        // ★ 查表（字段是 m_token_string，不是 m_token_text）

    switch (tok) {
      /* All identifiers are printed with their name. */
      case IDENT:
      case IDENT_QUOTED:
      case TOK_IDENT:
      case TOK_IDENT_AT: {                   // ★ 4 种标识符统一处理
        const char *id_ptr = nullptr;
        int id_len = 0;
        uint err_cs = 0;

        current_byte =
            read_identifier(digest_storage, current_byte, &id_ptr, &id_len);
        if (current_byte > max_digest_length) {
          /* Truncation */
          return;
        }

        if (convert_text) {
          /* Verify that the converted text will fit. */
          if (to_cs->mbmaxlen * id_len > NAME_LEN) {
            if (add_space) {
              digest_output->append(" ", 1);
              add_space = false;
            }
            digest_output->append("...", 3);   // ★ 唯一一处整体 "..."
            break;
          }
          /* Convert identifier string into the storage character set. */
          id_length = my_convert(id_buffer, NAME_LEN, to_cs, id_ptr, id_len,
                                 from_cs, &err_cs);
          id_string = id_buffer;
        } else {
          id_string = id_ptr;
          id_length = id_len;
        }

        if (id_length == 0 || err_cs != 0) {
          break;                              // 空名或转码失败：整个标识符被丢弃
        }

        if (add_space) {
          digest_output->append(" ", 1);
          add_space = false;
        }

        /* Copy the converted identifier into the digest string. */
        digest_output->append("`", 1);        // ★ 标识符一律加反引号
        if (id_length > 0) {
          digest_output->append(id_string, id_length);
        }
        if (tok == TOK_IDENT_AT)  // No space before @ in "table@query_block".
        {
          digest_output->append("`", 1);      // ★ 空格规则③
        } else {
          digest_output->append("`", 1);
          add_space = true;
        }
      } break;

      /* Everything else is printed as is. */
      default:
        if (add_space) {
          digest_output->append(" ", 1);
          add_space = false;
        }

        int tok_length = tok_data->m_token_length;

        digest_output->append(tok_data->m_token_string, tok_length);
        // ★★ 这里就是 "?" / "?, ..." / "(?)" / "(?) /* , ... */"
        //     / "(...)" / "(...) /* , ... */" / "IN (...)" 的落地处——
        //     文本由 gen_lex_token.cc:341-359 登记，此处原样打印

        if (tok_data->m_append_space) {
          add_space = true;                   // ★ 空格规则②
        }
        break;
    }
  }
}
```

关键在 `lex_token_array` 这张表。三条空格规则的落点汇总：

| 规则 | 代码位置 | 作用 |
|---|---|---|
| ① 延迟空格 `add_space` | `:186` 声明、`:288-291`、`:268-270` | 只有下一个 token 真来了才补空格 ⇒ 杜绝尾部空格 |
| ② 每 token 属性位 `m_append_space` | `:296-298`；数据源 `gen_lex_token.cc:379` | 默认 true；`'@'` 显式置 false |
| ③ `TOK_IDENT_AT` 例外 | `:277-283` | `table@query_block` 的 `@` 前不加空格 |

#### 3.4.2 `lex_token.h` 是构建期生成的，源码树里没有

⚠️ `sql/lex_token.h` **不在源码树中**（搜索 0 命中）——它由 `gen_lex_token` 工具在构建时生成（对应 `sql/CMakeLists.txt` 里的 `gen_lex_token` target）。表项是四元组：

```cpp
struct lex_token_string {
  const char *m_token_string;   // 打印出来的文本
  int m_token_length;
  bool m_append_space;          // 打印后是否补一个空格
  bool m_start_expr;            // 后面是否跟一个表达式（判定一元/二元 ±）
};
```

**`m_append_space` 和 `m_start_expr` 这两个 bool 就是全部"语法智能"的来源**——1.2 的一元/二元判定用的是 `m_start_expr`，空格排版用的是 `m_append_space`。

表按 token 号分成 8 段（完整分区见 3.3.1）。

**结构体是 `main()` 现场 `printf` 打印出来的**（`sql/gen_lex_token.cc:501-512`）——所以生成的 `lex_token.h` 里那 4 个字段**没有任何注释**，注释只存在于 `gen_lex_token_string`（`:88`）：

```cpp
int main(int, char **) {                      // ★ 完全忽略 argv：不解析任何文件
  puts(ORACLE_GPL_COPYRIGHT_NOTICE("2016"));

  printf("/*\n");
  printf("  This file is generated, do not edit.\n");
  printf("  See file sql/gen_lex_token.cc.\n");
  printf("*/\n");
  printf("struct lex_token_string\n");
  printf("{\n");
  printf("  const char *m_token_string;\n");
  printf("  int m_token_length;\n");
  printf("  bool m_append_space;\n");
  printf("  bool m_start_expr;\n");
  printf("};\n");
  printf("typedef struct lex_token_string lex_token_string;\n");

  compute_tokens();     // ① 填 compiled_token_array[]
  print_tokens();       // ② 全部输出到 stdout（CMake 重定向到 lex_token.h）

  return 0;
}
```

```cpp
// sql/gen_lex_token.cc:88 —— 带注释的"内部版"结构体
  /**
    Real lexeme string or user-specified text to output with a normalized
    query string.
  */
  const char *m_token_string;
  /** Byte length of m_token_string. */
  int m_token_length;
  /**
    If true, output ' ' after this token to a normalized query string.
    See digest_add_token().
  */
  bool m_append_space;
  /** See digest_add_token(). */
  bool m_start_expr;
  /** The structure is uninitialized if false. */
  bool m_initialized{false};
```

**`print_tokens()`**（`:431-481`）——数组被 `#ifdef LEX_TOKEN_WITH_DEFINITION` 包住，因为只有 `sql_digest.cc` 需要数组实体，`sql_yacc.yy` 与 `sql_lex_hints.cc` 只要 `TOK_*` 宏：

```cpp
static void print_tokens() {
  int tok;

  printf("#ifdef LEX_TOKEN_WITH_DEFINITION\n");
  printf("lex_token_string lex_token_array[]=\n");
  printf("{\n");
  printf("/* PART 1: character tokens. */\n");

  for (tok = 0; tok < 256; tok++) {
    printf("/* %03d */  { \"\\x%02x\", 1, %s, %s},\n", tok, tok, ...);
  }

  range_for_sql_yacc1.print(
      "/* PART 2: named tokens from sql/sql_yacc.yy (chunk 1). */",
      "/* PART 3: padding reserved for sql/sql_yacc.yy extensions. */");

  range_for_sql_hints.print(
      "/* PART 4: named tokens from sql/sql_hints.yy. */",
      "/* PART 5: padding reserved for sql/sql_hints.yy extensions. */");

  range_for_digests.print(
      "/* PART 6: Digest special tokens. */",
      "/* PART 7: padding reserved for digest special tokens. */");

  range_for_sql_yacc2.print(
      "/* PART 8: named tokens from sql/sql_yacc.yy (chunk 2). */");

  printf("/* PART 9: End of token list. */\n");   // ★ 哨兵注释，不是编号区间

  printf("/* DUMMY */ { \"\", 0, false, false}\n");
  printf("};\n");
  printf("#endif /* LEX_TOKEN_WITH_DEFINITION */\n");

  printf("/* DIGEST specific tokens. */\n");      // ★ 以下 #define 无条件输出
  printf("#define TOK_GENERIC_VALUE %d\n", tok_generic_value);
  ...
  printf("#define TOK_UNUSED %d\n", tok_unused);
}
```

#### 3.4.2.1 digest fake token 的登记（`compute_tokens()`，`:334-363`）

```cpp
  /*
    Additional FAKE tokens,
    used internally to normalize a digest text.
  */

  /* Digest tokens in 5.7 */

  tok_generic_value = range_for_digests.add_token("?", __LINE__);            // 1100
  tok_generic_value_list = range_for_digests.add_token("?, ...", __LINE__);  // 1101
  tok_row_single_value = range_for_digests.add_token("(?)", __LINE__);       // 1102
  tok_row_single_value_list =
      range_for_digests.add_token("(?) /* , ... */", __LINE__);              // 1103
  tok_row_multiple_value = range_for_digests.add_token("(...)", __LINE__);   // 1104
  tok_row_multiple_value_list =
      range_for_digests.add_token("(...) /* , ... */", __LINE__);            // 1105
  tok_ident = range_for_digests.add_token("(tok_id)", __LINE__);             // 1106
  tok_ident_at = range_for_digests.add_token("(tok_id_at)", __LINE__);       // 1107
  tok_hint_comment_open =
      range_for_digests.add_token(HINT_COMMENT_STARTER, __LINE__);           // 1108 "/*+"
  tok_hint_comment_close =
      range_for_digests.add_token(HINT_COMMENT_TERMINATOR, __LINE__);        // 1109 "*/"

  /* New in 8.0 */

  tok_in_generic_value_expression =
      range_for_digests.add_token("IN (...)", __LINE__);                     // 1110

  /* Add new digest tokens here */

  tok_unused = range_for_digests.add_token("UNUSED", __LINE__);              // 1111
```

> ★ `add_token()` 用 `max_seen + 1` 递增，所以**在 5.7 组和 8.0 组之间插一个新 fake token 会改变后面所有编号**——这也证明了 3.3 的"只能追加"原则在 fake token 段同样成立。

#### 3.4.2.2 ★ `m_start_expr` 的 35 个 token（1.2 那条规则的完整判据）

```cpp
  /*
    Define additional properties for tokens.

    List all the token that are followed by an expression.
    This is needed to differentiate unary from binary
    '+' and '-' operators, because we want to:
    - reduce <unary +> <NUM> to <?>,
    - preserve <...> <binary +> <NUM> as is.
  */
  set_start_expr_token('(');
  set_start_expr_token(',');
  set_start_expr_token(EVERY_SYM);
  set_start_expr_token(AT_SYM);
  set_start_expr_token(STARTS_SYM);
  set_start_expr_token(ENDS_SYM);
  set_start_expr_token(DEFAULT_SYM);
  set_start_expr_token(RETURN_SYM);
  set_start_expr_token(IF);
  set_start_expr_token(ELSEIF_SYM);
  set_start_expr_token(CASE_SYM);
  set_start_expr_token(WHEN_SYM);
  set_start_expr_token(WHILE_SYM);
  set_start_expr_token(UNTIL_SYM);
  set_start_expr_token(SELECT_SYM);

  set_start_expr_token(OR_SYM);
  set_start_expr_token(OR2_SYM);
  set_start_expr_token(XOR);
  set_start_expr_token(AND_SYM);
  set_start_expr_token(AND_AND_SYM);
  set_start_expr_token(NOT_SYM);
  set_start_expr_token(BETWEEN_SYM);
  set_start_expr_token(LIKE);
  set_start_expr_token(REGEXP);

  set_start_expr_token('|');
  set_start_expr_token('&');
  set_start_expr_token(SHIFT_LEFT);
  set_start_expr_token(SHIFT_RIGHT);
  set_start_expr_token('+');
  set_start_expr_token('-');
  set_start_expr_token(INTERVAL_SYM);
  set_start_expr_token('*');
  set_start_expr_token('/');
  set_start_expr_token('%');
  set_start_expr_token(DIV_SYM);
  set_start_expr_token(MOD_SYM);
  set_start_expr_token('^');
```

三组共 **35 个**（`sql/gen_lex_token.cc:390-428`）：**① 结构/语句起始**（`( , EVERY AT STARTS ENDS DEFAULT RETURN IF ELSEIF CASE WHEN WHILE UNTIL SELECT`）— **② 布尔/逻辑**（`OR OR2 XOR AND && NOT BETWEEN LIKE REGEXP`）— **③ 算术/位运算**（`| & << >> + - INTERVAL * / % DIV MOD ^`）。

> ★ **语义**：这些 token 后面**一定跟一个表达式**，所以紧随的 `+`/`-` 是**一元**号（应被吸收成 `?`）。否则为二元，保留原样。消费点就是 `sql/sql_digest.cc:441` 的 `if (lex_token_array[last_token2].m_start_expr)`。
>
> ★ 注意这个列表**不完备**的语法后果：`NOT`、`BETWEEN` 在；但 `IN`、`IS`、`ON`、`WHERE`、`HAVING`、`THEN`、`ELSE` 不在——所以 `WHERE a-1` 里的 `-` 会被判为**一元**（因为前一个 token 是标识符 `a`，`m_start_expr=false` → 其实还是二元）。真正会误判的是 `SELECT-1` 这类。

#### 3.4.2.3 PART 1 与"永不显示"的占位串

PART 1（单字符）的 `m_token_string` 就是字符本身（打印成 `"\xNN"`）。被归一化的 token 其文本是**永远不会显示**的占位串（因为收集阶段已被改写成 `TOK_GENERIC_VALUE`）：

```cpp
  /*
    Values.
    These tokens are all normalized later,
    so this strings will never be displayed.
  */
  range_for_sql_yacc1.set_token(NUM, "(num)", __LINE__);
  range_for_sql_yacc1.set_token(TEXT_STRING, "(text)", __LINE__);
  range_for_sql_yacc1.set_token(IDENT, "(id)", __LINE__);
```

同理被登记为占位串的还有 `(bin) (decimal) (float) (hex) (hostname) (long) (ulonglong) (nchar) (id_quoted)`，以及 `sql_lex.cc` 里硬编码的合成 token：`"WITH ROLLUP"`（单 token 打印两个词）、`"!"`、`"||"`、`"?"`、`":="`、`"(_charset)"`、`""`(END_OF_INPUT)、`"->"`、`"->>"`。

> ★ `WITH_ROLLUP_SYM` 是"一个 token 打印成两个词"的典型——这正是 04 篇提到的 LALR(2) 问题在词法层的解法，也是"归一化规则散布在词法器各处"（第七章限制）的实例。

#### 3.4.3 空格规则（三条）

1. **延迟空格**：`add_space` 先置位，下次 append 前才真正写空格——保证没有尾部空格：

```cpp
  /*
    When a space needs to be appended, set add_space to true,
    and delay actually adding the space until the next token
    is found.
    This is to prevent printing digest text
    with a trailing space character.
  */
```

2. **默认 `m_append_space = true`**——所以绝大多数关键字/运算符后面都带空格（这就是 `` `b` , `a` `` 里逗号前后都有空格的原因）。
3. **唯一被显式关掉的是 `@`**，为了打印 `@@variable`：

```cpp
  /*
    The lexer parses "@@variable" as '@', '@', 'variable', ...
    This is incorrect, '@' is not really a token, ...
    To work around this, digest text are printed as "@@variable".
  */
  compiled_token_array[(int)'@'].m_append_space = false;
```

#### 3.4.4 一个不对称：超长标识符

标识符**没有写入期截断**（放不下就 `m_full`），但**显示期**会缩成 `...`：

```cpp
char id_buffer[NAME_LEN + 1] = {'\0'};
...
if (to_cs->mbmaxlen * id_len > NAME_LEN) {
  digest_output->append("...", 3);
  break;
}
```

`NAME_LEN` = 64 × 3 = **192 字节**。所以：**超过 64 个字符的标识符在 `DIGEST_TEXT` 里显示成 `...`，但它仍完整地参加了 SHA-256 计算**——两个超长但不同的表名，DIGEST 不同、DIGEST_TEXT 却都显示 `...`。这是一个真实的不对称。

#### 3.4.5 失败路径

| 条件 | 行为 |
|---|---|
| `byte_count > m_token_array_length`（脏读不一致） | append `"\0"` 后返回 |
| `from_cs == nullptr`（字符集还没写完就被读） | 同上 |
| `tok <= 0` 或越界或 `current_byte > max_digest_length` | 直接 return，**已拼好的部分保留** |
| 标识符转码失败 | 静默 `break` |

PFS 侧还有一层兜底：`m_byte_count == 0` 的行（即数组满后归集到 index 0 的聚合行）**不计算 digest/digest_text**，直接置 NULL。

---

## 四、两条独立通道：token 流 vs QT_NORMALIZED_FORMAT

| | token 流 digest（本篇主角） | `QT_NORMALIZED_FORMAT` |
|---|---|---|
| 层级 | **词法级**（token 字节流） | **语法树级**（AST 重打印） |
| 入口 | `MYSQLlex` 逐 token 收集 | `THD::normalized_query()` → `lex->unit->print(QT_NORMALIZED_FORMAT)` |
| 消费方 | PFS、`STATEMENT_DIGEST()` | query rewrite 插件（`mysql_parser_get_normalized_query`） |
| 互相换算 | ❌ **无直接换算关系** | |

`QT_NORMALIZED_FORMAT` 是 `enum_query_type` 的位（`1<<9`），各常量 Item 的 `print()` 直接输出 `?`。两者是**两套归一化**，注意不要混为一谈。

### 4.1 第二条通道的实现：`Item::print` 逐个吐 `?`

入口只有一个：

```cpp
const String THD::normalized_query() {
  m_normalized_query.mem_free();
  lex->unit->print(this, &m_normalized_query, QT_NORMALIZED_FORMAT);
  return m_normalized_query;
}
```

**它走的是表达式树打印**（`Query_expression::print` → `Query_block::print` → 递归 `Item::print`），与词法 token 流完全无关：不需要 `sql_digest_storage`、不需要 `max_digest_length`、不需要解析期间收集——它是解析**完成之后**对 LEX/Item 树的一次遍历。

`?` 是在各常量 Item 的 `print()` 里逐产生的：

```cpp
void Item_int::print(const THD *, String *str,
                     enum_query_type query_type) const {
  if (query_type & QT_NORMALIZED_FORMAT) {
    str->append("?");
    return;
  }
```

`Item_string` / `Item_decimal` / `Item_float` 同构。`Item_param` 把 `QT_NORMALIZED_FORMAT` 与 `QT_NO_DATA_EXPANSION` 当同效处理。

消费方是 **Rewriter 插件**（`mysql_parser_get_normalized_query`），`enum_query_type` 的注释本身就把两条通道绑在一起了：

```cpp
  /**
    Change all Item_basic_constant to ? (used by query rewrite to compute
    digest.)
  */
  QT_NORMALIZED_FORMAT = (1 << 9),
```

### 4.2 两套通道的实际差异

| 维度 | token 流（`compute_digest_text`） | `QT_NORMALIZED_FORMAT`（`Item::print`） |
|---|---|---|
| 数据来源 | 解析期间的线性字节流 | 解析完成后的 LEX/Item 树 |
| 标识符 | 一律包反引号 `` `a` `` | `append_identifier()`，按需引号 |
| **列表折叠** | ✅ 有 `?, ...` / `(...)` / `IN (...)` | ❌ **没有**（逐项打印） |
| 一元 ± | 靠 `m_start_expr` 表判断 | 由树结构天然区分 |
| **截断** | 受 `max_digest_length` 限制，会丢 token | **不受限** |
| hint | `TOK_HINT_COMMENT_OPEN/CLOSE` 两个 fake token | 未解析的 hint **只在** `QT_NORMALIZED_FORMAT` 下打印 |

hint 这条最能说明二者是**刻意各写一套**：注释明说是因为 query rewrite 在 resolve **之前**就要拿到归一化文本。

⚠️ **源码无证据表明二者等价**——没有任何注释或断言做跨通道比较。MTR 的 `compare_digests.inc` 比较的是 `statement_digest_text()` 与 PFS 的 `digest_text`，**两者都是 token 通道**，不是跨通道。

### 4.3 `STATEMENT_DIGEST_TEXT()` 走哪条？—— **token 通道**

它**真的开了一个独立 parser + 独立 token buffer**，把 `thd->m_digest` 临时换成自己的：

```cpp
String *Item_func_statement_digest_text::val_str(String *buf) {
  ...
  Thd_parse_modifier thd_mod(thd, m_token_buffer);
  if (parse(thd, args[0], statement_string)) return error_str();
  compute_digest_text(&thd->m_digest->m_digest_storage, buf);
  return buf;
}
```

`Thd_parse_modifier` 的构造函数里 `thd->m_digest = &m_digest_state; m_digest_state.reset(token_buffer, get_max_digest_length());`，析构时还原。`STATEMENT_DIGEST()` 同理，只是最后调 `compute_digest_hash` + `DIGEST_HASH_TO_STRING`。

所以这两个函数**与当前语句自己的 digest 毫无关系**，是拿参数当一条 SQL 重新解析一遍。

---

## 五、PFS 消费方

### 5.1 从 THD 到 PFS：digest 的传递路径

SQL 层算出的 digest 要跨三层才落到汇总表：

```
① THD::m_digest（sql_digest_state）
     ↓ MYSQL_DIGEST_END → pfs_digest_end_vc()
② PSI_statement_locker_state::m_digest（const sql_digest_storage*）
     ↓ pfs_end_statement_vc() → find_or_create_digest()
③ PFS_events_statements::m_digest_storage   （current/history，供 DIGEST_TEXT 还原）
   PFS_statements_digest_stat::m_digest_storage（汇总表，供还原 + 索引）
```

**哈希是在第 ①→② 这一步才算的**（不是解析时）：

```cpp
void pfs_digest_end_vc(PSI_digest_locker *locker,
                       const sql_digest_storage *digest) {
  if (state->m_collect_flags & STATE_FLAG_DIGEST) {
    /* TODO: pfs_digest_end_v1() has side effects here, to document better */
    auto *update_digest = const_cast<sql_digest_storage *>(digest);
    compute_digest_hash(digest, update_digest->m_hash);
```

那个 TODO 说的"副作用"就是**通过 `const_cast` 把哈希写回调用方（THD 侧）那一份**。

注意第 ③ 步进汇总表时**又拷了一次 token 数组**——因为 `DIGEST_TEXT` 要能还原，而 THD 的 buffer 会被下一条语句覆盖：

```cpp
  /*
    Copy digest storage to statement_digest_stat_array so that it could be
    used later to generate digest text.
  */
  pfs->m_digest_storage.copy(digest_storage);
```

### 5.2 聚合 key = (schema, digest)，且**只有这两个**

硬证据是 key 结构体本身，以及它只被这两个字段参与哈希/比较：

```cpp
struct PFS_digest_key {
  PFS_schema_name m_schema_name;
  unsigned char m_hash[DIGEST_HASH_SIZE];
};
```

```cpp
static uint digest_hash_func(const LF_HASH *, const uchar *key, size_t key_len) {
  nr1 = murmur3_32(digest_key->m_hash, DIGEST_HASH_SIZE, nr2);
  digest_key->m_schema_name.hash(&nr1, &nr2);
  return nr1;
}
```

⚠️ **`thread` 参数只用于取 `LF_PINS`（无锁哈希的线程本地 pin），不参与 key**。推论：

- 同一 digest 在**不同库**下执行 → **两行**
- 同一库下**不同用户/host** 执行 → **一行**（不区分身份）

表上 `UNIQUE KEY (SCHEMA_NAME, DIGEST)`——**同 digest 不同 schema 分开聚合**。

语句结束时拿到/创建聚合槽，累加统计：

```cpp
if (digest_stat != nullptr) {
  digest_stat->m_stat.aggregate_value(wait_time);
  digest_stat->m_stat.m_cpu_time += cpu_time;
  ...
}
```

> 统计字段的全貌（COUNT_STAR / SUM_TIMER_WAIT / 直方图 / QUERY_SAMPLE 采样）见 [`../infra/pfs_statement.md`](../../infra/observability/pfs/pfs_statement.md)，本篇只讲指纹本身。

### 5.3 存储结构与槽分配算法

- 索引：**无锁 `LF_HASH`**（murmur3_32 哈希 + memcmp 比较 32 字节哈希 + schema 名）
- 数组：`statements_digest_stat_array`，大小 = `performance_schema_digests_size`，单调递增索引轮转分配
- 每记录 `pfs_lock`；查询样本用原子引用计数；副本是**脏拷贝**（源码注释承认并发无锁读取）

分配用**单调游标 + 线性探测**，且 index 0 是保留槽：

```cpp
while (++attempts <= digest_max) {
  safe_index = digest_monotonic_index.m_u32++ % digest_max;
  if (safe_index == 0) {
    /* Record [0] is reserved. */
    continue;
  }
  ...
}
```

**index 0 保留给"其余全部"（all else）**，注释直说：

```cpp
/**
  Current index in Stat array where new record is to be inserted.
  index 0 is reserved for "all else" case when entire array is full.
*/
```

#### ★ 数组满了怎么办：没有淘汰，新 digest 全部并入 index 0

这是最容易被说错的一点。`find_or_create_digest()` 的**两个满溢出口都指向 index 0 且都 `digest_lost++`**：

```cpp
if (digest_full) {
  /* digest_stat array is full. Add stat at index 0 and return. */
  pfs = &statements_digest_stat_array[0];
  digest_lost++;
  ...
```

⚠️ **无 LRU、无按 `first_seen`/`last_seen` 驱逐。** `digest_full` 一旦置 true 就**永久为 true**，直到 `TRUNCATE TABLE events_statements_summary_by_digest`（唯一清理点，里面 `digest_full = false;` 并把游标重置为 1）。

溢出后：新 digest 不再占新槽，统计**全部累加到 index 0 那一行**。

判断"digest 表是否溢出过"的**唯一可靠指标**是状态变量 `Performance_schema_digest_lost`（`digest_lost`）。

#### 那行为什么是 NULL

index 0 在 init 时就被标记为已占用，但**从不填 digest 内容**（`m_byte_count` 保持 0）：

```cpp
/* Set record[0] as allocated. */
statements_digest_stat_array[0].m_lock.set_allocated();
```

读表时靠 `m_byte_count == 0` 判定为特殊行，DIGEST/DIGEST_TEXT 一律 NULL：

```cpp
  /*
    "0" value for byte_count indicates special entry i.e. aggregated
    stats at index 0 of statements_digest_stat_array. So do not calculate
    digest/digest_text as it should always be "NULL".
  */
```

而且它**只在真正溢出过之后才可见**（`rnd_next` 里 `if (digest_stat->m_first_seen != 0)` 才输出）。

### 5.4 两个相关参数

| 参数 | 默认 | 作用 |
|---|---|---|
| `max_digest_length` | 1024 | THD 侧 token 数组大小；写满后 `m_full` 停止收集，TEXT 输出到此截断 |
| `performance_schema_max_digest_length` | 1024 | PFS 每条 digest 记录的 token 副本缓冲区大小 |

### 5.5 STATEMENT_DIGEST() / STATEMENT_DIGEST_TEXT()

走的是 **token 通道**（不是 `Item::print`），机制见 4.3——函数求值期用 `Thd_parse_modifier` 临时接管 `thd->m_digest`，把参数当一条 SQL 重新解析一遍。

⚠️ 一个真实的坑：`?` 在普通（非 prepare）解析模式下是语法错误，所以：

```
SELECT statement_digest( 'SELECT ?' );
ERROR HY000: Could not parse argument to digest function: ... near '?'
```

（MTR `func_digest.result` 有记录。）要测 `?` 的 digest 只能走真实预编译。

### 5.6 DIGEST_TEXT 是懒计算的

`compute_digest_text()` 全程只出现在**读路径**上（`PFS_digest_row::make_row()` 与 `table_events_statements.cc` 读行时），**没有任何一处出现在 `pfs_end_statement_vc()` 的热路径**。

热路径只做两件事：一次 token 数组 memcpy（上限 1024 字节）+ 一次 SHA-256。

**性能含义**：`SELECT * FROM events_statements_summary_by_digest` 全表扫，就是 **digest_max 次 token→文本还原**。这跟"只 SELECT 需要的列"省不掉——`DIGEST_TEXT` 是在 `make_row()` 里无条件还原的。

### 5.7 `max_digest_length = 0` 的三层禁用

| 层 | 机制 |
|---|---|
| 1 | THD **根本不分配缓冲**：`if (max_digest_length > 0) m_token_array = my_malloc(...)`，否则 `nullptr` |
| 2 | 不请求计算：`if (get_max_digest_length() != 0) m_compute_digest = true;` |
| 3 | `compute_digest_text()` 终止条件 `current_byte > max_digest_length` → 第一个 token 就 return，DIGEST_TEXT 恒空 |

两个变量都是 `READ_ONLY`、范围 `0..1048576`、默认 **1024**：`max_digest_length`（THD 侧）与 `performance_schema_max_digest_length`（PFS 侧每行副本）。

---

## 六、版本演进：5.7 → 8.0

digest **不是 8.0 才有的**——`gen_lex_token.cc` 里 fake token 被明确分成两组，注释直接点名了 5.7：

```cpp
  /* Digest tokens in 5.7 */
  tok_generic_value = range_for_digests.add_token("?", __LINE__);
  ...
  /* New in 8.0 */
  tok_in_generic_value_expression =
      range_for_digests.add_token("IN (...)", __LINE__);
```

8.0 相对 5.7 的变化（均有源码证据）：

| # | 变化 | 证据 |
|---|---|---|
| 1 | **MD5 → SHA-256**（FIPS 合规） | `sql_digest.h` 有一段"哈希算法选型史"注释：*"MD5: used up to MySQL 5.7, abandoned in MySQL 8.0, non FIPS compliant"*；`compute_digest_hash()` 用 `SHA_EVP256` + `static_assert` 兜底 |
| 2 | **新增 `IN (...)` 折叠** | `/* New in 8.0 */` 注释（见上）；即 1.5 的 IN 列表折叠是 8.0 才有的 |
| 3 | **新增 `TOK_IDENT_AT`** | `table@query_block`，配合 8.0 的 optimizer hint QB name |
| 4 | **新增 hint 通道** | `TOK_HINT_COMMENT_OPEN/CLOSE` + `skip_digest` 标志（见 2.2） |
| 5 | **token 号空间重新分段并冻结** | `static_assert(YYUNDEF == 1150, ...)`；`sql_yacc.yy` 里留了 `*token YYUNDEF 1150` 的显式 gap 注释 |
| 6 | `compute_digest_text()` **去掉了 `truncated` 出参** | `sql_parse.cc` 的 doxygen 示例里还留着 `compute_digest_md5` 和带 `truncated` 的旧签名——**化石代码**，这两个函数在 8.0 已不存在 |

**为什么哈希要换**：那段选型注释把候选都列了一遍（MD5 128bit / SHA1 160bit / SHA2-224 / SHA2-256 / SHA2-384 / SHA2-512），只有 SHA2-256 标注了 *"FIPS compliant" + "Used starting with MySQL 8.0"*。SHA1 和 SHA2-224 被排除的理由都是 *"non FIPS compliant in strict mode"*。

**token 值为什么必须冻结**：因为 **raw token number 直接进了哈希输入**（3.2.2 的 hex 里那些 `EC 02`、`C4 01` 就是）。`sql_yacc.yy` 的 MAINTAINER 注释把规矩写死了：

```
Token values are used in query DIGESTS.
To make DIGESTS stable, it is desirable to avoid changing token values.
In practice, this means adding new tokens at the end of the list,
in the current release section (8.0), instead of adding them in the middle.
```

`gen_lex_token.cc` 还有一组 `static_assert` 卡住边界（如 `ZEROFILL_SYM == 906`），并给出 token 用尽时的三种处理选项——其中选项 3 明说"需要重录所有打印 DIGEST 的 MTR 测试，因为 DIGEST values have now changed"。

> ⚠️ **源码无直接证据**说明"5.6 引入"。最早的直接证据是 5.7 的 fake token 分组与 MD5 遗留。

---

## 七、已知限制

| 限制 | 说明 |
|---|---|
| **截断前缀哈希** | token 数组满后停止收集 → 两个超长语句若前 1024 字节 token 流相同则 digest 相同 |
| **脏读** | `copy()`/`compute_digest_text()` 并发无锁读取，源码注释承认 |
| **NULL 二义性** | 词法层无法区分字面量 NULL 与 `IS NULL`，靠 yacc 补刀（1.6） |
| **同桶异执行** | 参数值丢失 → 同 digest 下混着执行计划截然不同的执行（缓解：QUERY_SAMPLE_TEXT 保留最慢样本） |
| **标识符大小写敏感** | `T` 与 `t` 两个桶 |
| **对 token 编号敏感** | token 表变更静默改变所有 digest 值（3.3） |
| **归一化规则散布** | 部分规则在词法器各处特判（如 `WITH_ROLLUP_SYM` 是单 token 打印 "WITH ROLLUP"），不只在 `sql_digest.cc` |

---

## 参考

**源码**
- `sql/sql_digest.cc` / `sql_digest.h` —— 归一化归约、哈希、重放
- `sql/sql_lex.cc` / `sql_lex_hints.cc` —— token 收集与 hint 特例
- `sql/item_strfunc.cc` —— `STATEMENT_DIGEST()` 函数
- `storage/perfschema/pfs_digest.cc` / `table_esms_by_digest.cc` —— 聚合消费
- `sql/gen_lex_token.cc` —— token 打印表与 `m_start_expr` 标志

**相关文档**
- PFS 聚合细节（统计字段/直方图/sample 采样策略）见 `infra` 篇
- 计划身份（SPM signature）见 [`07_optimize/14_plan_stability.md`](07_optimize/14_plan_stability.md)
- 预编译（PREPARE 期的 digest、参数占位符）见 [`12_prepared_statement.md`](12_prepared_statement.md)
