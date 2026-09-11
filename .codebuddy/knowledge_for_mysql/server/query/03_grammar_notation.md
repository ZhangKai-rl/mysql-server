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

### 规则 3：`{A | B | C}` → 多分支

```bnf
-- 官方文档：{ALL | DISTINCT | DISTINCTROW}
-- sql_yacc.yy（select_options 展开后）:
select_options:
          ...
        | ALL
        | DISTINCT
        | DISTINCTROW
        ...
```

"必选组"就是 BNF 本来就有的"多分支"，不需要缩写符号，直接写。

---

## 四、SELECT 语法逐行对照

用户贴的官方 SELECT 语法，逐行对应 `sql_yacc.yy` 的 `query_specification` 产生式（`sql_yacc.yy:9994`）：

| 官方手册（EBNF） | sql_yacc.yy 对应 | 位置 |
|-----------------|-----------------|------|
| `SELECT` | `SELECT_SYM`（终结符） | query_specification 第 1 项 |
| `[ALL \| DISTINCT \| DISTINCTROW]` | `select_options` 内的 `opt_distinct` | `:11565` |
| `[HIGH_PRIORITY]` / `[STRAIGHT_JOIN]` / `[SQL_SMALL_RESULT] ...` | `select_options` 的其余分支 | `select_options` 规则 |
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

核心产生式（`sql_yacc.yy:9994-10038`，两个分支 = INTO 有没有）：

```bnf
query_specification:
          SELECT_SYM
          select_options
          select_item_list
          into_clause              -- 注意：不是 opt_ 前缀（见下）
          opt_from_clause
          opt_where_clause
          opt_group_clause
          opt_having_clause
          opt_window_clause
          { $$ = NEW_PTN PT_query_specification(...); }
        | SELECT_SYM               -- 第二个分支：无 INTO
          ...
        ;
```

**两个阅读要点**：

1. **`into_clause` 没有 `opt_` 前缀**，但官方文档里 `[into_option]` 是可选的——因为 `into_clause` 规则本身就有 `%empty` 空分支（INTO 的位置特殊：语法上允许在 SELECT 列表后**或**语句末尾出现，所以官方文档把 `[into_option]` 写了两处）。
2. **每个 `opt_` 展开后就是一列可空项**——官方文档那一长串 `[...]` 叠 `[...]` 的竖排列表，在 Bison 里就是 `query_specification` 右侧一长串 `opt_xxx`。

---

## 五、语法记法和词法/语法解析的对应

官方文档语法里的**每个元素**，在解析链路里都有落点：

| 记法元素 | 词法层（02 篇） | 语法层（04 篇） |
|---------|----------------|----------------|
| 大写关键字（`SELECT`/`FROM`/`WHERE`） | 关键字 token：`SELECT_SYM`/`FROM_SYM`/`WHERE_SYM` | 产生式右侧的终结符 |
| 小写占位符（`select_expr`/`where_condition`/`tbl_name`） | 通用 token 类别：`IDENT`/`NUM`/`TEXT_STRING` | 非终结符（再展开成子规则） |
| `[ ]` / `{ }` / 竖排 | （词法不关心，这是语法层的结构） | `opt_xxx` / 多分支 / 产生式 |
| `, ...` 重复列表 | 词法产出 `,`（单字符 token） | `xxx_list` 左递归规则 |

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
