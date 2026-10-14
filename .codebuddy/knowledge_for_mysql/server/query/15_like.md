# LIKE 深度解析

> 基于 MySQL 8.0.39 源码。涵盖 `LIKE` 的语法与 Item 模型、通配符匹配的回溯算法、`LIKE → range` 转换、索引可用性边界、ICP 下推、代价估算，以及与全文检索（FTS）的边界。
>
> **边界**：本篇讲 `LIKE` 这个算子自身的全链路（求值 → 优化 → 执行）。索引访问机制本身（覆盖索引、ICP 的通用框架、MRR）见 [`../../index/access.md`](../../index/access.md)；range 优化器的 SEL_ARG 体系见 [`07_optimize/`](07_optimize/)；全文检索见 [`../../feat/fts.md`](../../feat/fts.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [一、语法与 Item_func_like](#一语法与-item_func_like)
  - [二、escape 的求值规则](#二escape-的求值规则)
  - [三、★ 通配符匹配算法：回溯递归](#三-通配符匹配算法回溯递归)
  - [四、大小写、_ 与尾部空格](#四大小写_-与尾部空格)
  - [五、★ LIKE → range 的转换](#五-like--range-的转换)
  - [六、索引可用性的边界](#六索引可用性的边界)
  - [七、ICP：LIKE 在引擎内求值](#七icplike-在引擎内求值)
  - [八、代价估算：11.11% 与"用不上直方图"](#八代价估算1111-与用不上直方图)
  - [九、三种执行位置与 EXPLAIN 怎么看](#九三种执行位置与-explain-怎么看)
  - [十、与全文检索（FTS）的边界](#十与全文检索fts的边界)
- [相关的系统变量](#相关的系统变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 先从一个真实的慢查询开始

```sql
-- 500 万行的日志表，content 上有普通 B-tree 索引
EXPLAIN SELECT * FROM logs WHERE content LIKE '%timeout%';
```

```
+----+-------------+-------+------+---------------+------+---------+------+---------+-------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows    | Extra       |
+----+-------------+-------+------+---------------+------+---------+------+---------+-------------+
|  1 | SIMPLE      | logs  | ALL  | NULL          | NULL | NULL    | NULL | 4832917 | Using where |
+----+-------------+-------+------+---------------+------+---------+------+---------+-------------+
```

而把模式改成 `LIKE 'timeout%'`，立刻变成 `type=range`。同一个算子，一个字符的位置之差就决定了"全表扫 500 万行"还是"扫几百行"——本篇要讲清的就是这条分界线在哪、为什么在那。

三个先抛出来的结论：

1. **`LIKE` 的匹配算法是教科书式的回溯递归**，没有 KMP、没有 NFA 构造。多个非相邻 `%` 时**最坏指数级**，唯一的防护是递归栈深度检查。
2. **`LIKE` 能不能用上索引，取决于"匹配集合在 B-tree 上有序连续"**：`'abc%'` 的匹配集是字典序上的一段连续区间，`'%abc'` 的匹配集在整个键空间里散布各处，无法用有限区间覆盖。
3. **`LIKE` 与 FTS 不是"性能差一点的同一件事"**：LIKE 是字符级子串匹配（无分词、无打分），FTS 是词级倒排检索。二者能互相替代的只有"包含某个词"这一种形态。

### 是什么

`LIKE` 是 SQL 的标准字符串模式匹配算子，两个通配符：`%`（任意长度）、`_`（恰好一个字符），可选 `ESCAPE` 子句指定转义符。

在 MySQL 里它是一枚普通的 `Item_func_like`（`Item_bool_func2` 的子类），最终调用的是**字符集虚表里的 `wildcmp` 回调**——也就是说，匹配语义（大小写、重音、多字节）完全由排序规则决定，而算法骨架只有一套。

**★ 三个语义常量就是 LIKE 的全部"宪法"**（`include/mf_wcomp.h:27`）：

```cpp
/**
  Character constant for the escape character in a wildcard pattern
  (SQL style).
*/
const char wild_prefix = '\\';

/**
  Character constant for wildcard representing any one character
  (SQL style).
*/
const char wild_one = '_';

/**
  Character constant for wildcard representing zero or more
  characters (SQL style).
*/
const char wild_many = '%';
```
> ★ 这三个常量在 `sql/` 与 `strings/` 中共享——LIKE 与 GRANT 里库名/表名的通配（如 `GRANT ... ON db\_%.*`）基于同一套符号。

**★★ LIKE 走的是 `my_wildcmp` 宏，不是 `wild_compare()`**（⚠️ 常见混淆）：

```cpp
/* include/m_ctype.h:761 */
#define my_wildcmp(cs, s, se, w, we, e, o, m) \
  ((cs)->coll->wildcmp((cs), (s), (se), (w), (we), (e), (o), (m)))
```
```cpp
/* include/mf_wcomp.h:53 —— wild_compare 的官方自述 */
  @deprecated This function is not charset-aware, and should not be
  used in new code.
```
> ★★ `wild_compare()` 定义在 `mysys/mf_wcomp.cc:121`，**不感知字符集**，只服务 rpl_filter / auth cache 等内部场景。**LIKE 从不使用它**。
> ★ `my_wildcmp` 是 collation handler 的**可插拔方法**：8bit / bin / mb / mb_bin / unicode / uca / gb18030 各有实现。

```cpp
/* sql/item_func.h:196 —— range optimizer 靠这个分流 */
    LIKE_FUNC,
```
```cpp
/* sql/sql_yacc.yy:863 */
%token  LIKE 530                          /* SQL-2003-R */
```

### 用途

- 子串 / 前缀 / 单字符的模糊匹配（`LIKE '%x%'`、`LIKE 'x%'`、`LIKE 'a_'`）
- 可以走 B-tree 的**前缀范围扫描**（`LIKE 'abc%'`），这是唯一能用索引的形态
- 与 FTS 的分工：需要"相关度排序 / 分词 / 词距"时用 `MATCH ... AGAINST`；需要"任意形态子串 / 精确转义 / 非文本列"时用 LIKE

**★ 语法产生式**（`sql/sql_yacc.yy:10343`，★ NOT LIKE 是外层包 `Item_func_not`）：

```cpp
        | bit_expr LIKE simple_expr
        {
          $$ = NEW_PTN Item_func_like(@$, $1, $3);
        }
        | bit_expr LIKE simple_expr ESCAPE_SYM simple_expr %prec LIKE
        {
          $$ = NEW_PTN Item_func_like(@$, $1, $3, $5);
        }
        | bit_expr not LIKE simple_expr
        {
          auto item = NEW_PTN Item_func_like(@$, $1, $4);
          $$ = NEW_PTN Item_func_not(@$, item);          /* ★★ NOT LIKE = NOT(LIKE) */
        }
        | bit_expr not LIKE simple_expr ESCAPE_SYM simple_expr %prec LIKE
        {
          auto item = NEW_PTN Item_func_like(@$, $1, $4, $6);
          $$ = NEW_PTN Item_func_not(@$, item);
        }
```
```cpp
/* sql/sql_yacc.yy:1448 —— LIKE 与其它比较符同优先级，左结合 */
%left   EQ EQUAL_SYM GE GT_SYM LE LT NE IS LIKE REGEXP IN_SYM
```
> ★★ pattern 侧是 `simple_expr` 不是 `bit_expr`——保证 `'a' LIKE b` 里 `b` 是 pattern。
> ★ **NOT LIKE 永远无法转 range**：`Item_func_not` 包在外面，`get_mm_tree` 看到的顶层 func 是 NOT 而不是 LIKE。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0 | 默认排序规则改为 `utf8mb4_0900_ai_ci`（UCA 9.0.0），LIKE 走 `my_wildcmp_uca_impl`；随之带来 **NO PAD** 语义 |
| 8.0 | `Item_func::fix_length_and_dec()` 改名为 `resolve_type()`（旧资料里的名字已不存在） |
| 8.0 | 无 `PTI_like` 占位节点，yacc action 直接构造 `Item_func_like` |
| 8.0 | 默认 ICP 开启（`optimizer_switch` 默认集合含 `index_condition_pushdown`） |

**★ 版本演进的源码证据（7 条，全部 grep 实证）**：

**(a) 为向后兼容**故意违反 SQL 标准**（`sql/item_cmpfunc.cc:6395`）：

```cpp
  // Use backslash as escape character if the escape clause evaluates to NULL.
  // (For backward compatibility. The SQL standard says the LIKE expression
  // should evaluate to NULL in this case.)
  if (escape_item->null_value) {
    m_escape = '\\';
    return false;
  }
```

**(b) 官方 TODO：当前设计牺牲了可优化场景**（`item_cmpfunc.cc:718`）：

```cpp
  /*
    If the escape item is const, evaluate it now, so that the range optimizer
    can try to optimize LIKE 'foo%' into a range query.

    TODO: If we move this into escape_is_evaluated(), which is called later,
          we might be able to optimize more cases.
  */
```

**(c) 8.0 引入 NO PAD collation 后 LIKE range 语义变化**（`strings/ctype-mb.cc:679`）：

```cpp
        For LIKE 'a%', assuming min_sort_char='\0' and max_sort_char='\xff':

        "a" is the smallest possible string for NO PAD.
        "a\0\0..." is the smallest possible string for PAD SPACE.
        "a\xff\xff..." is the biggest possible string.
```

**(d) 8.0 新增 `escape_is_const` 跨执行复用**（`item_cmpfunc.cc:6365`）：

```cpp
void Item_func_like::cleanup() {
  if (!escape_is_const) escape_evaluated = false;   /* ★ 常量 escape 求值一次后跨执行复用 */
  Item_bool_func2::cleanup();
}
```

**(e) Bug#29175461：escape 本身是通配符时的语义修正**（`mysql-test/t/func_like.test:246`）：

```
--echo # Bug#29175461: LIKE RESULT ERROR IF ESCAPE IS % OF WILDCARD CHARACTERS
SET NAMES big5;
select 1 where 'b%a' like '%%a' escape '%';
select 1 where 'b_a' like '__a' escape '_';
```
> ★ 对应 `val_int()` 里 `(escape() == wild_many) ? -1 : wild_many` 的写法。该 bug 横跨 6 种 wildcmp 实现逐一验证。

**(f) 历史渊源**：`strings/ctype-czech.cc:46` 的注释显示 `my_like_range` 与 `my_strxfrm` **同源**（1997 年）：

```cpp
        This file defines functions my_strxfrm and my_strcoll for
        ... Also defined here you will find function my_like_range that returns
        index range strings for LIKE expression and the
        MY_STRXFRM_MULTIPLY set to value 4 ...
```

**(g) 旧接口被标记废弃**（`include/mf_wcomp.h:53` 与 `:79`）：`wild_compare_full` / `wild_compare` 均标 `@deprecated ... should not be used in new code`。

> ⚠️ **版本考证**：`resolve_type()` 确实是 8.0 对 `fix_length_and_dec()` 的重命名——`Item_func_like::fix_length_and_dec()` **在 8.0.39 不存在**。

---

## 理论基础

### 设计思想与权衡

**① 为什么是"逐字符集回调 + 回溯"，而不是一个统一的匹配器？**

MySQL 把 `wildcmp` 做成了 `CHARSET_INFO::coll` 的一个函数指针，每个字符集/排序规则各实现一份（8bit、bin、mb、mb_bin、unicode、uca、gb18030…）。

- **收益**：大小写、重音、多字节、contractions（如捷克语 `ch`）这些语义天然落在字符集层，LIKE 自己完全不用管；换一个排序规则就等于换一套"比较权重函数"。
- **代价**：算法被复制了 7+ 份，且每一份都是同一个回溯实现——**没有一份做了 KMP 预处理**。

**② 为什么不做 KMP / Boyer-Moore / NFA？**

- KMP/BM 只对**纯字面量模式**有效，一旦有 `%` 就必须退化为"找锚点 + 递归剩余"，预处理表失效。
- 正确的通用做法是 Thompson 构造（NFA 模拟，O(n·m) 保证），MySQL 没做。理由可以推断：LIKE 的典型模式很短（`'abc%'`），锚点扫描 + 剪枝在实践中足够快；而构造 NFA 需要额外内存分配，对"每行求值一次"的高频路径反而是负担。
- **代价落在哪**：多 `%` 且长串时**指数级回溯**。源码自己承认（`strings/ctype-uca.cc` 注释："This takes exponential time in the worst case"）。唯一的防护是 `my_string_stack_guard`（递归深度/栈检查），超限时**返回"不匹配"而不是报错**——注意这是一个静默的正确性降级，不是报错。

**③ 为什么 `LIKE '%abc'` 不能走索引（以及为什么不是"优化器偷懒"）？**

B-tree 的键按排序规则全局有序，一次 range 扫描只能覆盖**字典序上连续的若干区间**：

- `LIKE 'abc%'` 的匹配集 = `{abc, abca, …, abc⿰}` ≈ `[abc, abd)` —— **连续**，一个区间即可（略宽，靠复检兜底）
- `LIKE '%abc'` 的匹配集 = `{abc, xabc, yabc, zabc, …}` —— **在整个键空间散布**，要么全扫，要么无穷多个区间

这不是实现缺陷，是"B-tree 只能表达有序连续区间"的结构性限制。同理可推：`LIKE 'a%b%c'` 也只能转成 `[a, b)` 这样一个粗区间（前缀之外的部分没法用）。

**④ 为什么转出来的 range 一律标记 `inexact`？**

因为 Unicode 排序规则存在 contractions 等现象，使得"字符前缀"与"排序权重区间"并不严格一一对应。源码注释直说 "LIKE is tricky to get 100% exact… we currently always set that LIKE range scans are inexact"。**后果**：`LIKE 'abc%'` 转 range 后仍必须保留原谓词复检——这就是 EXPLAIN 里 `type=range` 却同时带 `Using where` 的原因。

**⑤ 代价模型为什么不看模式内容？**

`get_filtering_effect` 直接返回一个 BETWEEN 的常量系数，注释给的理由是"逐字扫描模式串找通配符太贵，而 LIKE 多数带通配符，所以不查"。**后果**：`LIKE '%a'`（几乎不过滤）和 `LIKE 'a very long specific prefix%'`（过滤性极强）拿到的选择率完全一样。

**★★ `val_int()` 只有单一实现，没有"三个分支"**（`sql/item_cmpfunc.cc:6271`，⚠️ 常见说法是错的）：

```cpp
longlong Item_func_like::val_int() {
  assert(fixed);

  if (!escape_evaluated && eval_escape_clause(current_thd)) return error_int();
                                            /* ① escape 只求值一次（幂等） */

  const CHARSET_INFO *cs = cmp.cmp_collation.collation;
                                            /* ② 比较 collation 由 Arg_comparator 聚合而来 */

  String *res = eval_string_arg(cs, args[0], &cmp.value1);
  if (args[0]->null_value) { null_value = true; return 0; }
                                            /* ③ NULL LIKE x → NULL */
  String *res2 = eval_string_arg(cs, args[1], &cmp.value2);
  if (args[1]->null_value) { null_value = true; return 0; }
                                            /* ④ x LIKE NULL → NULL */
  null_value = false;
  if (current_thd->is_error()) return 0;

  return my_wildcmp(cs, res->ptr(), res->ptr() + res->length(), res2->ptr(),
                    res2->ptr() + res2->length(), escape(),
                    (escape() == wild_one) ? -1 : wild_one,
                                            /* ★ escape 就是 '_' → 传 -1 禁用 _ 通配符 */
                    (escape() == wild_many) ? -1 : wild_many)
                                            /* ★ escape 就是 '%' → 传 -1 禁用 %（Bug#29175461） */
             ? 0
             : 1;                           /* ★ my_wildcmp 返回 0=匹配，Item 约定 1=true */
}
```
> ★ 常量 pattern 的"优化"发生在**别处**：`select_optimize()`（能否建 range）与 `check_covering_prefix_keys()`（能否用前缀/覆盖索引）。

**★★ 为什么 ICP 不推到聚簇索引**（`sql/sql_select.cc:2979`，★ 8 条准入规则里最有说服力的一条）：

```cpp
  /*
    6. The index is not a clustered index. The performance improvement
       of pushing an index condition on a clustered key is much lower
       than on a non-clustered key. This restriction should be
       re-evaluated when WL#6061 is implemented.
  */
```
> ★★ 因为聚簇索引里**索引项就是整行**，下推条件省不掉回表，只省了 server↔引擎一次函数调用，反而白付了"物化索引项"的代价。

**★★ ICP 的收益不只是省回表，还省锁**（`row0sel.cc:5463`）：

```cpp
  switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
    case ICP_NO_MATCH:
      prebuilt->try_unlock(true);      /* ★★ 先释放刚加的行锁，再跳下一条 */
      goto next_rec;
```

**★ 多表 UPDATE/DELETE 禁用 ICP 的原因**（`sql/sql_select.cc:2960`，正确性考量）：

```cpp
  /*
    3. The query is not a multi-table update or delete statement. The reason
       for this requirement is that the same handler will be used
       both for doing the select/join and the update. The pushed index
       condition might then also be applied by the storage engine
       when doing the update part and result in either not finding
       the record to update or updating the wrong record.
  */
```

### 理论溯源

| 理论 / 经典算法 | 源码落点 |
|---|---|
| **Glob / 通配符匹配**（Unix shell 传统，回溯实现） | `my_wildcmp_8bit_impl`（`strings/ctype-simple.cc:753`）、`my_wildcmp_uca_impl`（`strings/ctype-uca.cc:2205`） |
| **KMP（Knuth-Morris-Pratt, 1977）** —— 作为"没被采用"的对照：只对纯字面量有效，遇 `%` 失效 | 全仓库无 KMP 实现；`%` 的锚点扫描是朴素线性（`ctype-simple.cc:803`） |
| **排序权重比较（collation weight）** —— 大小写/重音不敏感靠权重表而非字符转换 | `likeconv` 宏 = `(cs)->sort_order[...]`（`ctype-simple.cc:750`）；UCA 用 `my_uca_charcmp`（`ctype-uca.cc:2269`） |
| **B-tree 的有序性与区间扫描** | `LIKE → SEL_ARG` 的单闭区间（`range_analysis.cc:1528`） |
| **索引条件下推（ICP）** —— 减少回表的经典优化 | `row_search_idx_cond_check`（`row0sel.cc:3782`） |

**★ 源码自述：这套算法是 "globbing"，且官方明确说"新代码不要用"**（`include/mf_wcomp.h:45`）：

```cpp
/**
  Performs wildcard matching, aka globbing, on the input string with
  the given wildcard pattern, and the specified wildcard characters.
  ...
  @deprecated This function is not charset-aware, and should not be
  used in new code.
  ...
  @return 0 if match, 1 otherwise
*/
int wild_compare_full(...);
```
> ★ "globbing"（Unix shell 传统）是官方用词。LIKE 走的是**同一算法的字符集感知版本**（`my_wildcmp_*`），不是这个 deprecated 函数。

**★ 回溯递归的完整骨架**（`mysys/mf_wcomp.cc:44`，★ 与 `my_wildcmp_*` 结构相同，可作为最简模型阅读）：

```cpp
int wild_compare_full(const char *str, int strlen, const char *wildstr,
                      int wildlen, bool str_is_pattern, char w_prefix,
                      char w_one, char w_many) {
  while (wildstr < wildend) {
    while (wildstr < wildend && *wildstr != w_many && *wildstr != w_one) {
      if (*wildstr == w_prefix && wildstr + 1 < wildend) {
        wildstr++;
        if (str_is_pattern && str < strend && *str++ != w_prefix) return 1;
      }
      if (str == strend || *wildstr++ != *str++) return 1;
    }
    if (wildstr == wildend) return str < strend;   /* pattern 耗尽：str 还剩 → 不匹配 */
    if (*wildstr++ == w_one) {
      if (str == strend || (str_is_pattern && *str == w_many)) return 1;
      if (*str++ == w_prefix && str_is_pattern) str++;
    } else { /* Found '*' */
      while (str_is_pattern && str < strend && *str == w_many) str++;
      for (; wildstr < wildend && (*wildstr == w_many || *wildstr == w_one);
           wildstr++) {
        if (*wildstr == w_many) {
          while (str_is_pattern && str < strend && *str == w_many) str++;
        } else {
          if (str_is_pattern && str + 1 < strend && *str == w_prefix) str += 2;
          else if (str == strend) return 1;
        }
      }
      if (wildstr == wildend) return 0; /* '*' as last char: OK */
      if ((cmp = *wildstr) == w_prefix && wildstr + 1 < wildend &&
          !str_is_pattern)
        cmp = wildstr[1];
      for (;; str++) {
        while (str < strend && *str != cmp) str++;   /* ★★ 锚点线性扫描 */
        if (str == strend) return 1;
        if (wild_compare_full(str, strend - str, wildstr, wildend - wildstr,
                              str_is_pattern, w_prefix, w_one, w_many) == 0)
          return 0;                                   /* ★★ 回溯递归 —— 指数复杂度根源 */
      }
    }
  }
  return str < strend;
}
```
> ★★ 关键两步：**锚点推进**（把 `%` 后第一个确定字符当作锚，线性扫到下一个候选位置）+ **递归剩余**（每个候选起点递归一次）。这就是最坏 O(n^k) 的来源。

**★ 一个重要的实用优化：合并连续通配符**（`strings/ctype-simple.cc:793`）：

```cpp
      wildstr++;
      /* Remove any '%' and '_' from the wild search string */
      for (; wildstr != wildend; wildstr++) {
        if (*wildstr == w_many) continue;
        if (*wildstr == w_one) {
          if (str == str_end) return (-1);
          INC_PTR(cs, str, str_end);
          continue;
        }
        break; /* Not a wild character */
      }
      if (wildstr == wildend) return (0); /* Ok if w_many is last */
```
> ★ `'%%%'` 与 `'%%_'` 被折叠成单个 `%`，显著减少递归深度。

**★ 最坏复杂度的唯一防护：栈熔断**（`sql/mysqld.cc:3964`，★ 源码明说 LIKE 有"病态用例"）：

```cpp
/**
  This function is used to check for stack overrun for pathological
  cases of regular expressions and 'like' expressions.
  The call to current_thd is quite expensive, so we try to avoid it
  for the normal cases.
  The size of each stack frame for the wildcmp() routines is ~128 bytes,
  so checking *every* recursive call is not necessary.
*/
extern "C" {
static int check_enough_stack_size(int recurse_level) {
  uchar stack_top;
  if (recurse_level % 16 != 0) return 0;        /* ★ 每 16 层才真检查一次 */
  THD *my_thd = current_thd;
  if (my_thd != nullptr)
    return check_stack_overrun(my_thd, STACK_MIN_SIZE * 4, &stack_top);
  return 0;
}
}
```
> ★ 挂接点：`sql/mysqld.cc:5099` `my_string_stack_guard = check_enough_stack_size;`
> ★ 声明：`include/m_ctype.h:229` `extern int (*my_string_stack_guard)(int);`；初值 `strings/ctype.cc:65` = nullptr。
> ★★ 单帧约 **128 字节**，每 16 层检查一次，要求剩余栈 ≥ `4 × STACK_MIN_SIZE`。**超限返回"不匹配"而非报错**——这是一个**静默的正确性降级**。

### 算法与数据结构

| 项 | 值 / 结构 |
|---|---|
| 匹配算法 | 回溯递归（无预处理、无记忆化） |
| 最好情况 | 无通配符：O(n) 线性 |
| 最坏情况 | k 个非相邻 `%`：**O(n^k) 指数级** |
| 防护 | `my_string_stack_guard` 递归深度检查，超限返回"不匹配" |
| 匹配粒度 | utf8mb4 下 `_` 匹配**一个字符**（1~4 字节），不是一字节 |
| range 形态 | 单个闭区间 `[min_key, max_key]`，`inexact = true` |
| 选择率 | 固定 `max(1/rows_in_table, 0.1111)`，不看模式内容、不用直方图 |

---

## 核心实现

### 主链路

```
【SQL 语法】sql_yacc.yy:10343  bit_expr LIKE simple_expr [ESCAPE simple_expr]
              → NEW_PTN Item_func_like(@$, $1, $3[, $5])

【名字解析】Item_func_like::fix_fields      item_cmpfunc.cc:6353   打开"可用前缀索引"标记
            Item_func_like::resolve_type    item_cmpfunc.cc:682    聚合 collation + 求值 escape
              ├─ agg_arg_charsets_for_comparison()                 三个参数一起聚合
              ├─ eval_escape_clause()       item_cmpfunc.cc:6376
              └─ check_covering_prefix_keys() item_cmpfunc.cc:6326

【优化器】Item_func_like::select_optimize   item_cmpfunc.cc:6303   ★首字符 %/_ → OPTIMIZE_NONE
            → get_mm_tree                    range_analysis.cc:845  闸门：OPTIMIZE_NONE 直接返回
            → get_full_func_mm_tree          range_analysis.cc:749
            → get_mm_leaf（LIKE 分支）       range_analysis.cc:1455
              └─ my_like_range → cs->coll->like_range → my_like_range_mb（ctype-mb.cc:661）
              └─ *inexact = true（:1521）+ SEL_ARG(min_str, max_str)

【执行】三种位置之一：
  ① range 内定位    IndexRangeScanIterator           index_range_scan.cc:168
  ② ICP 引擎内求值  row_search_idx_cond_check        row0sel.cc:3782
                      → innobase_index_cond          ha_innodb.cc:23508
                        → pushed_idx_cond->val_int() → Item_func_like::val_int
  ③ server 层过滤   FilterIterator                   composite_iterators.h:78

【求值核心】Item_func_like::val_int          item_cmpfunc.cc:6271
              └─ my_wildcmp(cs, ...)         m_ctype.h:761 → cs->coll->wildcmp
                  └─ my_wildcmp_8bit_impl / _mb / _uca / _unicode ...（回溯）
```

### 一、语法与 Item_func_like

产生式（`sql/sql_yacc.yy:10343-10360`）：

```yacc
| bit_expr LIKE simple_expr                      { $$ = NEW_PTN Item_func_like(@$, $1, $3); }
| bit_expr LIKE simple_expr ESCAPE_SYM simple_expr %prec LIKE
                                                 { $$ = NEW_PTN Item_func_like(@$, $1, $3, $5); }
| bit_expr not LIKE simple_expr                  { auto item = NEW_PTN Item_func_like(@$, $1, $4);
                                                   $$ = NEW_PTN Item_func_not(@$, item); }
```

要点：

- **`PTI_like` 不存在**。语法 action 直接构造 `Item_func_like`（与 `MATCH` 一样属于"语法阶段即可确定"的少数 Item）。
- **`NOT LIKE` 没有专门类**，是 `Item_func_not` 包一层 `Item_func_like`——这一点后面会决定它永远不能转 range。
- `ESCAPE` 是第 3 个参数 `args[2]`，`%prec LIKE` 用于解决与 `NOT` 的优先级冲突。
- 左操作数必须是 `bit_expr`（LIKE 优先级低于 `=`、`IN`、`BETWEEN`）。

**类定义位置容易找错**：`Item_func_like` 在 **`sql/item_cmpfunc.h:2347`**，不在 `item_func.h`（后者只有 `LIKE_FUNC` 枚举）。它继承 `Item_bool_func2`，因此复用比较器的两个 `String` 缓冲：

```cpp
class Item_func_like final : public Item_bool_func2 {
  bool escape_is_const = false;
  bool escape_evaluated = false;
  int m_escape;                                   // 0 = 无转义符
public:
  longlong val_int() override;
  enum Functype functype() const override { return LIKE_FUNC; }
  optimize_type select_optimize(const THD *thd) override;
  /// 有 ESCAPE 时，相同输入也不保证相同结果 ⇒ 不能像 a=a 那样静态短路
  cond_result eq_cmp_result() const override { return COND_OK; }
  bool resolve_type(THD *) override;              // 注意：不是 fix_length_and_dec
  float get_filtering_effect(...) override;
};
```

`resolve_type`（`item_cmpfunc.cc:682`）做三件关键的事：

```cpp
max_length = 1;                                                    // 结果是布尔
if (agg_arg_charsets_for_comparison(cmp.cmp_collation, args, arg_count)) return true;   // ★1
args[0]->cmp_context = STRING_RESULT;                              // ★2 永远按字符串比
args[1]->cmp_context = STRING_RESULT;
if (arg_count > 2) {
  if (!args[2]->const_for_execution()) {                           // ★3 ESCAPE 必须常量
    my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE"); return true;
  }
}
if (!escape_was_used_in_parsing() || args[2]->const_item()) {
  escape_is_const = true;
  if (eval_escape_clause(thd)) return true;
  if (check_covering_prefix_keys(thd)) return true;
}
```

- ★1 **比较用的排序规则在这里定**：把**全部参数（含 escape）**按 SQL 标准的 coercibility 聚合（`COLLATE` 显式 > 列 > 字面量）。这个 `cs` 同时决定后面走哪一份 `wildcmp` 实现。
- ★2 LIKE 永远是字符串比较，不会像其它 `Item_bool_func2` 那样把常量转成数值。
- ★3 非常量 ESCAPE 直接报错。

`val_int()`（`item_cmpfunc.cc:6271`）很短：

```cpp
longlong Item_func_like::val_int() {
  if (!escape_evaluated && eval_escape_clause(current_thd)) return error_int();
  const CHARSET_INFO *cs = cmp.cmp_collation.collation;
  String *res = eval_string_arg(cs, args[0], &cmp.value1);
  if (args[0]->null_value) { null_value = true; return 0; }
  String *res2 = eval_string_arg(cs, args[1], &cmp.value2);
  if (args[1]->null_value) { null_value = true; return 0; }
  null_value = false;
  if (current_thd->is_error()) return 0;
  return my_wildcmp(cs, res->ptr(), res->ptr() + res->length(),
                    res2->ptr(), res2->ptr() + res2->length(), escape(),
                    (escape() == wild_one) ? -1 : wild_one,      // ★ escape 若是 '_' 就关掉它
                    (escape() == wild_many) ? -1 : wild_many) ? 0 : 1;
}
```

三值逻辑：任一参数为 NULL → 结果 NULL。返回值映射：`my_wildcmp` 返回 `0 = 匹配`，所以 `? 0 : 1` 把"匹配"变成 SQL 的 TRUE。

### 二、escape 的求值规则

`eval_escape_clause`（`item_cmpfunc.cc:6376-6460`）的规则表（这些边界很值得记）：

| 情况 | 行为 |
|---|---|
| 没写 ESCAPE | 默认 `'\\'`；`NO_BACKSLASH_ESCAPES` 模式下为 **0**（无转义符） |
| `ESCAPE NULL` | 仍用 `'\\'`（源码注释说明这是向后兼容，SQL 标准应为 NULL） |
| `ESCAPE ''` | `m_escape = 0`（无转义符）；`NO_BACKSLASH_ESCAPES` 下报错 |
| `ESCAPE 'ab'`（多字符） | 报 `ER_WRONG_ARGUMENTS` |
| 多字节字符集 | 存 **Unicode code point**（经 `mb_wc` 转换） |
| 单字节字符集 | 存本机编码字节，字符集不同则 `copy_and_convert` |

模式串里的转义语义：`\%` → 字面 `%`；`\_` → 字面 `_`；`\\` → 字面 `\`。**模式末尾的孤立转义符按自身处理**（`LIKE 'abc\'` 的尾部 `\` 就是字面反斜杠）。

**★ `eval_escape_clause()` 核心原文**（`sql/item_cmpfunc.cc:6376`，⚠️ 成员真名是 `int m_escape`，**不是** `escape_item`）：

```cpp
bool Item_func_like::eval_escape_clause(THD *thd) {
  assert(!escape_evaluated);
  escape_evaluated = true;

  const bool no_backslash_escapes =
      thd->variables.sql_mode & MODE_NO_BACKSLASH_ESCAPES;

  // No ESCAPE clause is specified. The default escape character is backslash,
  // unless NO_BACKSLASH_ESCAPES mode is enabled.
  if (!escape_was_used_in_parsing()) {
    m_escape = no_backslash_escapes ? 0 : '\\';     /* 规则 1 */
    return false;
  }

  Item *escape_item = args[2];                      /* 只是局部变量，不是成员 */
  String buf;
  const String *escape_str = escape_item->val_str(&buf);
  if (thd->is_error()) return true;

  // Use backslash as escape character if the escape clause evaluates to NULL.
  // (For backward compatibility. The SQL standard says the LIKE expression
  // should evaluate to NULL in this case.)
  if (escape_item->null_value) {
    m_escape = '\\';                                /* 规则 2：故意违反标准 */
    return false;
  }

  // An empty escape sequence means there is no escape character. An empty
  // escape sequence is not accepted in NO_BACKSLASH_ESCAPES mode.
  if (escape_str->is_empty()) {
    if (no_backslash_escapes) {
      my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE");
      return true;                                  /* 规则 3a */
    }
    m_escape = 0;                                   /* 规则 3b：取消转义 */
    return false;
  }

  // Accept at most one character.
  if (escape_str->numchars() > 1) {
    my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE");
    return true;                                    /* 规则 4：是 numchars 不是字节数 */
  }

  // For multi-byte character sets, we store the Unicode code point of the
  // escape character.
  if (use_mb(cmp.cmp_collation.collation)) {         /* 规则 5a */
    const CHARSET_INFO *cs = escape_str->charset();
    my_wc_t wc;
    int rc = cs->cset->mb_wc(cs, &wc, pointer_cast<const uchar *>(escape_str_ptr),
                             pointer_cast<const uchar *>(escape_str_ptr) +
                                 escape_str->length());
    if (rc <= 0) { my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE"); return true; }
    m_escape = wc;
    return false;
  }

  // For single-byte character sets, we store the native code instead of the
  // Unicode code point. ...
  const CHARSET_INFO *cs = cmp.cmp_collation.collation;
  size_t unused;
  if (escape_str->needs_conversion(escape_str->length(), escape_str->charset(),
                                   cs, &unused)) {    /* 规则 5b */
    char ch;
    uint errors;
    size_t cnvlen = copy_and_convert(&ch, 1, cs, escape_str_ptr,
                                     escape_str->length(),
                                     escape_str->charset(), &errors);
    if (cnvlen == 0) { my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE"); return true; }
    m_escape = static_cast<uchar>(ch);
  } else {
    m_escape = static_cast<uchar>(escape_str_ptr[0]);  /* 规则 5c */
  }
  return false;
}
```

**★ 两个非法 escape 的报错点**（都报 `ER_WRONG_ARGUMENTS`，消息 `"Incorrect arguments to %s"`，`%s = "ESCAPE"`）：

```cpp
/* item_cmpfunc.cc:711 —— resolve_type 里：每行都变的 ESCAPE 直接拒绝 */
    // ESCAPE clauses that vary per row are not valid:
    if (!args[2]->const_for_execution()) {
      my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE");
      return true;
    }
```
```cpp
/* item_cmpfunc.cc:739 —— 标量子查询被替换为派生表列后可能变成非常量，需重新校验 */
Item *Item_func_like::replace_scalar_subquery(uchar *) {
  // Replacing a scalar subquery with a reference to a column in a derived table
  // could change the constness. ...
  if (escape_was_used_in_parsing() && !args[2]->const_for_execution()) {
    my_error(ER_WRONG_ARGUMENTS, MYF(0), "ESCAPE");
    return nullptr;
  }
  return this;
}
```

### 三、★ 通配符匹配算法：回溯递归

调用的是宏 `my_wildcmp`（`include/m_ctype.h:761`），展开为 `cs->coll->wildcmp`。各字符集的实现：

| 实现 | 文件:行 | 挂载到 |
|---|---|---|
| `my_wildcmp_8bit_impl` | `strings/ctype-simple.cc:753` | latin1、ascii 等 8bit |
| `my_wildcmp_bin_impl` | `strings/ctype-bin.cc:309` | `binary`、`*_bin` |
| `my_wildcmp_mb_impl` | `strings/ctype-mb.cc:227` | big5/sjis/gbk/euc_kr 等多字节 |
| `my_wildcmp_unicode_impl` | `strings/ctype-utf8.cc:4847` | utf8mb4_general_ci / unicode_ci |
| **`my_wildcmp_uca_impl`** | **`strings/ctype-uca.cc:2205`** | **`utf8mb4_0900_ai_ci`（8.0 默认）** |

以最有代表性的 8bit 版为例（骨架，保留全部关键分支）：

```c
static int my_wildcmp_8bit_impl(const CHARSET_INFO *cs, const char *str,
                                const char *str_end, const char *wildstr_arg,
                                const char *wildend_arg, int escape, int w_one,
                                int w_many, int recurse_level) {
  int result = -1;                                     /* Not found, using wildcards */

  if (my_string_stack_guard && my_string_stack_guard(recurse_level)) return -1;   /* ① 栈保护 */

  while (wildstr != wildend) {
    while (*wildstr != w_many && *wildstr != w_one) {   /* ② 字面量前缀段 */
      if (*wildstr == escape && wildstr + 1 != wildend) wildstr++;
      if (str == str_end || likeconv(cs, *wildstr++) != likeconv(cs, *str++))
        return (1);                                     /* No match */
      if (wildstr == wildend)
        return (str != str_end);                        /* Match if both are at end */
      result = 1;                                       /* Found an anchor char */
    }
    if (*wildstr == w_one) {                            /* ③ '_' 段 */
      do {
        if (str == str_end) return (result);
        INC_PTR(cs, str, str_end);
      } while (++wildstr < wildend && *wildstr == w_one);
      if (wildstr == wildend) break;
    }
    if (*wildstr == w_many) {                           /* ④ '%' 段 —— 回溯发生地 */
      uchar cmp;
      wildstr++;
      for (; wildstr != wildend; wildstr++) {           /* 合并连续的 % 和 _ */
        if (*wildstr == w_many) continue;
        if (*wildstr == w_one) {
          if (str == str_end) return (-1);
          INC_PTR(cs, str, str_end);
          continue;
        }
        break;                                          /* 找到锚点字符 */
      }
      if (wildstr == wildend) return (0);               /* '%' 在末尾 ⇒ 直接匹配 */
      if (str == str_end) return (-1);

      if ((cmp = *wildstr) == escape && wildstr + 1 != wildend) cmp = *++wildstr;
      INC_PTR(cs, wildstr, wildend);
      cmp = likeconv(cs, cmp);
      do {
        while (str != str_end && (uchar)likeconv(cs, *str) != cmp) str++;   /* 线性找锚点 */
        if (str++ == str_end) return (-1);
        int tmp = my_wildcmp_8bit_impl(cs, str, str_end, wildstr, wildend_arg,
                                       escape, w_one, w_many, recurse_level + 1);  /* ★ 递归 */
        if (tmp <= 0) return (tmp);                     /* 剪枝：纯字面量失败就不再右移 */
      } while (str != str_end);
      return (-1);
    }
  }
  return (str != str_end ? 1 : 0);                      /* ⑤ 收尾：两边必须同时到尾 */
}
```

逐段解释：

1. **① 栈保护**：`my_string_stack_guard` 是函数指针（`strings/ctype.cc:65` 定义），在 `mysqld.cc` 里赋值为 `check_enough_stack_size`。每递归一层 `recurse_level + 1`，栈不够就返回 `-1`（不匹配）。**这是 DoS 防护，也是唯一的防护——它是静默降级，不报错。**
2. **② 字面量前缀**：逐字符比较，遇到转义符就跳过它取下一个字符。注意 ⑤ 之外的第二处收尾 `return (str != str_end)`——**两边必须同时耗尽**才匹配。
3. **③ `_`**：消耗输入串的一个字符（多字节下用 `INC_PTR` 推进整个字符）。连续 `___` 合并处理。
4. **④ `%`（回溯核心）**：
   - 先把 `%` 后面连续的 `%`、`_` 合并掉（`%%%%` ≡ `%`）；
   - 若 `%` 就是最后一个字符 → 直接匹配（`%` 吃掉剩余全部，含空）；
   - 否则取 `%` 后的**第一个字面字符作为锚点**，在输入串里**线性扫描**找它；
   - 找到后**递归**匹配"锚点之后的剩余模式 vs 剩余输入"；
   - 递归返回 `1`（纯字面量不匹配）时停止右移试错（剪枝），返回 `0`/`-1` 则直接返回。
5. **⑤ 收尾**：模式扫完之后，输入串必须也刚好耗尽——这是"LIKE 对尾部空格敏感"的根源。

**复杂度**：无通配符 O(n)；单个 `%` 最坏 O(n·m)；**k 个非相邻 `%` 最坏 O(n^k)**。典型病态输入形如 `col LIKE 'a%a%a%a%a%a'` 配 `'aaaaaaaaaaaaaaaaaaaaaaaaX'`。源码在 `ctype-uca.cc` 的注释里直接写了 "This takes exponential time in the worst case"。**没有任何记忆化**。

### 四、大小写、_ 与尾部空格

**大小写不是 LIKE 的开关，而是"比较用的权重函数"**：

```c
#define likeconv(s, A) (uchar)(s)->sort_order[(uchar)(A)]     /* ctype-simple.cc:750 */
```

> 权重从哪来、`sort_order` / `strnxfrm` / UCA 三级权重 / PAD 与 NO PAD 的完整机制，是字符集子系统的内容，见 [`../infra/charset.md`](../infra/charset.md)（尤其是「排序处理器」与「UCA 权重算法」两节）。本篇只关心"LIKE 如何用它们"。

- `latin1_swedish_ci` 的 `sort_order` 把 A–Z 映射到 a–z ⇒ 不敏感
- `*_bin` 里该宏被改成恒等映射（`ctype-bin.cc:306`）⇒ 敏感
- `utf8mb4_general_ci` 用 `my_tosort_unicode` 把码点映射为 Unicode 权重再比
- `utf8mb4_0900_ai_ci` 用 `my_uca_charcmp`（UCA 9.0.0 权重，含重音不敏感）

也就是说：**换一个排序规则就等于换一套权重表，算法骨架不动**。

**`_` 匹配一个字符，不是一个字节**（utf8mb4 下 1~4 字节）。所以 `'中' LIKE '_'` 为真，`'中' LIKE '__'` 为假。Unicode 实现先把码点解出来（`mb_wc`）再比较；多字节实现用 `INC_PTR` 推进整个字符。

**尾部空格：LIKE 是敏感的，且 PAD/NO PAD 都一样。**

```sql
SELECT 'abc ' LIKE 'abc';   -- 0（FALSE）
SELECT 'abc ' =    'abc';   -- 1（TRUE，PAD SPACE 下）
```

区别在于：等值比较走 `strnncollsp`，会 `skip_trailing_space()`；而所有 `wildcmp` 实现的收尾都是严格的"两边同时到尾"，**没有任何 skip_trailing_space 调用**。这就是 MySQL 文档"trailing spaces are significant for LIKE"的实现根源。

**★★ `_` 匹配"一个字符"的源码证明：`INC_PTR` 宏**（★ 单字节 vs 多字节两个版本）：

```cpp
/* strings/ctype-simple.cc:751 —— 单字节：一个字符 = 一个字节 */
#define INC_PTR(cs, A, B) (A)++
```
```cpp
/* strings/ctype-mb.cc:222 —— ★★ 多字节：一次前进"整个字符的字节数" */
#define INC_PTR(cs, A, B) \
  A += (my_ismbchar(cs, A, B) ? my_ismbchar(cs, A, B) : 1)
```
> ★★ utf8mb4 下 `_` 一次前进 `my_ismbchar()` 返回的字节数（中文 = 3 字节，emoji = 4 字节）。所以 `'中' LIKE '_'` 为真、`'中' LIKE '__'` 为假。

**★ `_` 必须消耗恰好一个字符**（`ctype-simple.cc:781`，★ 到串尾而 pattern 还有 `_` → 失败）：

```cpp
    if (*wildstr == w_one) {
      do {
        if (str == str_end) /* Skip one char if possible */
          return (result);                 /* ★ 没字符可消耗 → 失败 */
        INC_PTR(cs, str, str_end);
      } while (++wildstr < wildend && *wildstr == w_one);
      if (wildstr == wildend) break;
    }
```

**★★ 尾部空格的"两边同时到尾"判据**（`ctype-simple.cc:770`）：

```cpp
      if (str == str_end || likeconv(cs, *wildstr++) != likeconv(cs, *str++))
        return (1); /* No match */
      if (wildstr == wildend)
        return (str != str_end); /* Match if both are at end */
                                 /* ★★ pattern 耗尽时要求 str 也耗尽
                                    → 'abc ' LIKE 'abc'：str 多一个空格 → 不匹配 */
      result = 1;                /* Found an anchor char */
```
```cpp
/* 整个函数的最后一行，同样是这个判据（ctype-simple.cc:823） */
  return (str != str_end ? 1 : 0);
```

**★ PAD SPACE vs NO PAD 的实际差别在 `my_like_range`**（`strings/ctype-mb.cc:686`，★ 影响 min_length 而不是匹配本身）：

```cpp
      if ((cs->state & MY_CS_BINSORT) || cs->pad_attribute == NO_PAD) {
        *min_length = static_cast<size_t>(min_str - min_org);
                                    /* ★ NO PAD：min_length 就是前缀长度（如 3） */
        /*
          Pad with spaces, because for CHAR searches, our returned min_length
          is ignored and min_str is put directly into the value to search for.
        */
        do { *min_str++ = ' '; } while (min_str != min_end);
      } else {
        *min_length = res_length;   /* ★ PAD SPACE：min_length 取满列长 */
        do { *min_str++ = static_cast<char>(cs->min_sort_char); }
        while (min_str != min_end);
      }
```

**★ 一个 8.0 的副作用：NO PAD 下不能裁剪比较**（`sql/sql_optimizer.cc:8941`）：

```cpp
    if (field->type() == MYSQL_TYPE_STRING &&
        field->charset()->pad_attribute == NO_PAD) {
      /*
        For "NO PAD" collations on CHAR columns, this function must return
        false, because removal of trailing space in CHAR columns makes the
        table value and the index value compare differently. As the column
        strips trailing spaces, it can return false candidates. Further
        comparison of the actual table values is required.
       */
      return false;
    }
```
> ★ 即 `utf8mb4_0900_ai_ci`（NO PAD）下，索引值可能只是"假候选"，**必须回表再比一次**。

> 实践坑：PAD SPACE 排序规则下的 `CHAR(N)` 列，实际存储会补空格，`char_col LIKE 'abc'` 可能因为真实值是 `'abc  '` 而不命中，但 `char_col = 'abc'` 会命中。

### 五、★ LIKE → range 的转换

**第一道闸门**在 `Item_func_like::select_optimize`（`item_cmpfunc.cc:6303`）：

```cpp
if (!args[1]->may_evaluate_const(thd)) return OPTIMIZE_NONE;   // 模式非常量
String *res2 = args[1]->val_str(&cmp.value2);
if (!res2) return OPTIMIZE_NONE;
if (!res2->length()) return OPTIMIZE_OP;
char first = res2->ptr()[0];
return (first == wild_many || first == wild_one) ? OPTIMIZE_NONE    // ★ '%' 或 '_' 开头 → 放弃
                                                 : OPTIMIZE_OP;
```

`get_mm_tree`（`range_analysis.cc:899-906`）拿到 `OPTIMIZE_NONE` 就直接 `return nullptr`——**根本进不到构造区间的逻辑**。

**主体**在 `get_mm_leaf` 的 LIKE 分支（`range_analysis.cc:1455-1533`）：

```cpp
if (type == Item_func::LIKE_FUNC) {
  Item_func_like *like_func = down_cast<Item_func_like *>(cond_func);
  // We can only optimize with LIKE if the escape string is known.
  if (!like_func->escape_is_evaluated()) goto end;
  ...
  like_error = my_like_range(field->charset(), res->ptr(), res->length(),
                             like_func->escape(), wild_one, wild_many,
                             field_length, (char *)min_str + offset,
                             (char *)max_str + offset, &min_length, &max_length);
  if (like_error) goto end;                              // 无法构造区间
  ...
  /* LIKE is tricky to get 100% exact, especially with Unicode collations ... */
  *inexact = true;                                       // ★★ 必须复检
  SEL_ARG *root = new (alloc) SEL_ARG(field, min_str, max_str, ...);
}
```

可转 range 的条件汇总：

| 条件 | 结论 |
|---|---|
| 模式**首字符**是 `%` 或 `_` | **不可转**（闸门直接拒） |
| 模式是常量 | 必需（`may_evaluate_const`） |
| ESCAPE 是常量（或省略） | 必需（`escape_is_evaluated()`）；非常量 ESCAPE 不可转 |
| 字段是字符串类型 | 必需（走 `my_like_range`，要求 `STRING_RESULT`） |
| `Field::optimize_range()` 为真 | InnoDB 普通索引（含前缀索引）都带 `HA_READ_RANGE` ⇒ **前缀索引也能做 range**；`Field_enum` 硬编码 false |
| `NOT LIKE` | **永不转**（`Item_func_not(LIKE)`，`arguments()[0]` 不是 `FIELD_ITEM`） |

**区间怎么构造**（`my_like_range_mb`，`strings/ctype-mb.cc:661`；8bit 版是 `my_like_range_simple`）：

- 逐字符拷贝到 min 和 max，遇到 `_` → min 填最小排序字符、max 填最大排序字符；
- 遇到 `%` → 前缀部分保留，min 用最小字符填满剩余、max 用 `pad_max_char()` 填满剩余，**提前返回**；
- 无通配符 → `min == max`，退化成等值区间（会被识别为 `EQ_RANGE`，走 `HA_READ_KEY_EXACT`）。

`pad_max_char`（`ctype-mb.cc:602`）的坑：utf8mb4 的 `max_sort_char = 0xFFFF`，要用 `cs->cset->wc_mb()` 编码成多字节（`EF BF BF`）循环填充；尾部放不下整个字符时补空格。

**`res_length` 是字节数不是字符数**：`maxcharlen = res_length / cs->mbmaxlen`，所以"能放进 range 的前缀字符数" = `key_part->length / mbmaxlen`。若模式前缀超过索引前缀长度（如 `col(10)` 上 `LIKE 'abcdefghijklm%'`），循环会 `break`，最终退化成 `min == max == "abcdefghij"` 的等值区间——作为超集仍正确，靠 `inexact` 复检兜底。

**执行时怎么变成读标志**（`index_range_scan.cc:168-189`）：

```cpp
start_key->flag = ((cur->flag & NEAR_MIN) ? HA_READ_AFTER_KEY
                 : (cur->flag & EQ_RANGE) ? HA_READ_KEY_EXACT : HA_READ_KEY_OR_NEXT);
end_key->flag = (cur->flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
```

LIKE 的 `max_flag == 0` ⇒ `end_key->flag = HA_READ_AFTER_KEY`（把 max_key 自身也包含进来），这正是需要"填满最大字符的上界"的原因。

> **常见误解澄清**：`HA_READ_PREFIX_LAST` **不参与** LIKE→range。它只用于 MIN/MAX 优化、loose index scan、反向扫描、auto_increment。

**★★★ 转换核心：`get_mm_leaf()` 的 LIKE 分支**（`sql/range_optimizer/range_analysis.cc:1455`，⚠️ **`sql/opt_range.cc` 已不存在**，已拆包为 `sql/range_optimizer/`）：

```cpp
  if (type == Item_func::LIKE_FUNC) {
    ...
    if (!optimize_range) goto end;             /* 前提 A：索引支持 HA_READ_RANGE */
    if (!(res = value->val_str(&tmp))) {
      tree = new (alloc) SEL_ROOT(alloc, SEL_ROOT::Type::IMPOSSIBLE);
      goto end;                                /* pattern 为 NULL → 永假 */
    }
    if (field->cmp_type() != STRING_RESULT)
      goto end;  // Can only optimize strings   /* 前提 C：列必须字符串语义 */
    ...
    Item_func_like *like_func = down_cast<Item_func_like *>(cond_func);

    // We can only optimize with LIKE if the escape string is known.
    if (!like_func->escape_is_evaluated()) goto end;   /* 前提 D：escape 必须已求值 */

    field_length -= null_bytes;
    like_error = my_like_range(
        field->charset(), res->ptr(), res->length(), like_func->escape(),
        wild_one, wild_many, field_length, (char *)min_str + offset,
        (char *)max_str + offset, &min_length, &max_length);   /* ★★ 核心：交给 charset 层 */
    if (like_error) goto end;

    // LIKE is tricky to get 100% exact, especially with Unicode collations
    // (which can have contractions etc.), and will frequently be a bit too
    // broad. To be safe, we currently always set that LIKE range scans are
    // inexact and must be rechecked by means of a filter afterwards.
    *inexact = true;                          /* ★★ 永远标记 inexact */
    ...
    SEL_ARG *root = new (alloc)
        SEL_ARG(field, min_str, max_str, !(key_part->flag & HA_REVERSE_SORT));
                                              /* ★ 产物：单个 SEL_ARG，闭区间 [min, max] */
```

**★★ `%` 开头不能转 range 的真正拦截点**（`range_analysis.cc:904`，★ 比 `my_like_range` 更上游）：

```cpp
  Item_func *cond_func = (Item_func *)cond;
  if (cond_func->functype() == Item_func::BETWEEN ||
      cond_func->functype() == Item_func::IN_FUNC)
    inv = ((Item_func_opt_neg *)cond_func)->negated;
  else {
    Item_func::optimize_type opt_type = cond_func->select_optimize(thd);
    if (opt_type == Item_func::OPTIMIZE_NONE) return nullptr;
                                     /* ★★ LIKE '%x' 在此被彻底拒之门外 */
  }
```
```cpp
/* item_cmpfunc.cc:6322 —— select_optimize 的第一道闸门 */
Item_func::optimize_type Item_func_like::select_optimize(const THD *thd) {
  if (!args[1]->may_evaluate_const(thd)) return OPTIMIZE_NONE;
  if (!evaluate_during_optimization(args[1], thd->lex->current_query_block()))
    return OPTIMIZE_NONE;
  String *res2 = args[1]->val_str(&cmp.value2);
  if (!res2) return OPTIMIZE_NONE;
  if (!res2->length())  // Can optimize empty wildcard: column LIKE ''
    return OPTIMIZE_OP;
  char first = res2->ptr()[0];
  return (first == wild_many || first == wild_one) ? OPTIMIZE_NONE
                                                   : OPTIMIZE_OP;
                                     /* ★★ 首字符是 % 或 _ → 直接判死 */
}
```
> ★⚠️ 纠正：`my_like_range()` 对 `'%x'` **其实是能算的**（返回全量区间），真正拒绝它的是上面这段。文档里说"`my_like_range` 失败"不准确。

**★ `'abc%'` → 区间的推导**（`strings/ctype-simple.cc:825`，`my_like_range_simple`）：

```cpp
  for (; ptr != end && min_str != min_end && charlen > 0; ptr++, charlen--) {
    if (*ptr == escape && ptr + 1 != end) {    /* ① 转义符：吃掉，下一字符按字面写入 min/max */
      ptr++;
      *min_str++ = *max_str++ = *ptr;
      continue;
    }
    if (*ptr == w_one) {                       /* ② '_'：min 放最小字符，max 放最大排序字符 */
      *min_str++ = '\0';
      *max_str++ = (char)cs->max_sort_char;
      continue;
    }
    if (*ptr == w_many) {                      /* ③★ '%'：从此处吞掉一切，立即返回 */
      *min_length = ((cs->state & MY_CS_BINSORT) ? (size_t)(min_str - min_org)
                                                 : res_length);
      *max_length = res_length;
      do { *min_str++ = 0; *max_str++ = (char)cs->max_sort_char; }
      while (min_str != min_end);
      return false;
    }
    *min_str++ = *max_str++ = *ptr;            /* ④ 常量字符：min/max 同时写入 */
  }
  *min_length = *max_length = (size_t)(min_str - min_org);
  while (min_str != min_end)
    *min_str++ = *max_str++ = ' '; /* Because if key compression */  /* ⑤ 无通配符：等值 */
```
> ★ `'abc%'` → `[abc\0\0…, abc\xff\xff…)`；`'abc'` → 等值；`'a_c'` → `[a\0c, a\xffc]`。

**★ 多字节的收缩字母（contraction）特判**（`strings/ctype-mb.cc:680`，★ 宁可放宽也不能漏行）：

```cpp
      /*
        Special case for collations with contractions.
        For example, in Chezh, 'ch' is a separate letter
        which is sorted between 'h' and 'i'.
        If the pattern 'abc%', 'c' at the end can mean:
        - letter 'c' itself,
        - beginning of the contraction 'ch'.
        If we simply return this LIKE range:
         'abc\min\min\min' and 'abc\max\max\max'
        then this query: SELECT * FROM t1 WHERE a LIKE 'abc%'
        will only find values starting from 'abc[^h]',
        but won't find values starting from 'abch'.
        We must ignore contraction heads followed by w_one or w_many.
        ...
        For example, for Czech 'abc%', we will return LIKE range,
        which is equal to LIKE range for 'ab%'.
      */
      if (contraction_flags && ptr + 1 < end &&
          my_uca_can_be_contraction_head(contraction_flags, (uchar)*ptr)) {
        if (ptr[1] == w_one || ptr[1] == w_many) {
          goto fill_max_and_min;      /* ★ 保守地退回成更短的常量前缀 */
        }
```

**★ 前缀索引与覆盖索引**（`item_cmpfunc.cc:6326`，`check_covering_prefix_keys`）：

```cpp
    if (second_arg->const_item()) {
      size_t prefix_length = 0;
      String *wild_str = second_arg->val_str(&cmp.value2);
      if (my_is_prefixidx_cand(wild_str->charset(), wild_str->ptr(),
                               wild_str->ptr() + wild_str->length(), escape(),
                               wild_many, &prefix_length))
                                     /* ★ 'abc%' → 3；'%abc' → false */
        field->table->update_covering_prefix_keys(field, prefix_length,
                                                  &covering_keys);
      else
        // Not comparing to a prefix, remove all prefix indexes
        field->table->covering_keys.subtract(field->part_of_prefixkey);
                                     /* ★ 前缀不够 → 前缀索引不能当覆盖索引 */
    }
```
> ★ `Item_func_like::fix_fields()` 里有一行独有的调用（`item_cmpfunc.cc:6355`）：
> ```cpp
>   args[0]->real_item()->set_can_use_prefix_key();   /* ★ 唯独 LIKE 会给左值打这个标记 */
> ```

### 六、索引可用性的边界

| 形态 | 能否用索引 | 说明 |
|---|---|---|
| `LIKE 'abc%'` | ✅ `range` | 匹配集连续 |
| `LIKE 'abc'`（无通配符） | ✅ `range`（退化为 `EQ_RANGE`） | `HA_READ_KEY_EXACT` |
| `LIKE 'a%b%c'` | ✅ 但只有 `[a, b)` 这么粗 | 第二个 `%` 之后无法用，靠复检 |
| `LIKE '%abc'` / `LIKE '_abc'` | ❌ | 首字符闸门拒；匹配集散布 |
| `NOT LIKE 'abc%'` | ❌ | `Item_func_not` 包一层，永不转 range |
| `LIKE 'abc%'` 且有前缀索引 `col(10)` | ✅ 可做 range | 但前缀超长时该索引会被从 `covering_keys` 剔除（`table.cc:7970`），不能做覆盖扫描 |
| `LIKE` 用在 FULLTEXT 索引上 | ❌ | range 优化器显式 `continue` 跳过（`range_optimizer.cc:395`）；InnoDB 对 FULLTEXT 的 `index_flags` 返回 0（`ha_innodb.cc:6366`） |

**`LIKE '%abc'` 在覆盖索引下会走"全索引扫描"**：优化器在 `make_join_readinfo`（`sql_optimizer.cc:2958`）里把 `JT_ALL` 升级成 `JT_INDEX_SCAN`（条件是 keyread 允许 + 有覆盖索引 + 非物化表），因为索引记录比整行窄、省 IO 且不回表。**但它依然要逐行求值 LIKE**，不是"用索引定位"。

**不存在 `LIKE → ref/eq_ref` 的路径**：`add_key_field`（`sql_optimizer.cc:7213`）里 `if (!eq_func) return false;`，而 LIKE 的 `eq_func` 恒为 false ⇒ 从不生成 `Key_use`。LIKE 在该函数里只登记 `keys()`/`const_keys`/`key_dependent` 和一条 `SARGABLE_PARAM`。

**★ `Field::optimize_range()` 返回 `bool`，不是枚举**（`sql/field.cc:2137`，⚠️ 常见说法是错的）：

```cpp
bool Field::optimize_range(uint idx, uint part) const {
  return table->file->index_flags(idx, part, true) & HA_READ_RANGE;
}
```
> ★ `HA_READ_RANGE` 只有 B-tree 类引擎置位；HASH / FULLTEXT 不置位 ⇒ LIKE 建不了 range。
> ★ `Field_enum` 硬编码 `return false`（`sql/field.h:4384`）——ENUM/SET 按整数排序，字符串 range 无效。
> ⚠️ `OPTIMIZE_NONE / OPTIMIZE_KEY / OPTIMIZE_OP / OPTIMIZE_NULL / OPTIMIZE_EQUAL` 是 **`Item_func::optimize_type`**（`sql/item_func.h:309`），与 `Field::optimize_range()` 无关。

**★ 第二道闸门：`comparable_in_index()`**（`sql/range_optimizer/range_optimizer.cc:1260`）：

```cpp
bool comparable_in_index(Item *cond_func, const Field *field,
                         const Field::imagetype itype,
                         Item_func::Functype comp_type, const Item *value) {
  /*
    Usually an index cannot be used if the column collation differs
    from the operation collation. However, a case insensitive index
    may be used for some binary searches:

       WHERE latin1_swedish_ci_column = 'a' COLLATE lati1_bin;
       WHERE latin1_swedish_ci_colimn = BINARY 'a '
  */
  if ((field->result_type() == STRING_RESULT &&
       field->match_collation_to_optimize_range() &&
       value->result_type() == STRING_RESULT && itype == Field::itRAW &&
       field->charset() != cond_func->compare_collation() &&
       !((comp_type == Item_func::EQUAL_FUNC ||
          comp_type == Item_func::EQ_FUNC) &&
         cond_func->compare_collation()->state & MY_CS_BINSORT)))
    return false;   /* ★★ collation 不一致 → 索引不可用
                        这解释了 col LIKE 'x' COLLATE utf8mb4_bin 会丢索引 */
  ...
  /*
    We can't use indexes when comparing to a JSON value. For example,
    the string '{}' should compare equal to the JSON string "{}". ...
  */
  if (value->result_type() == STRING_RESULT &&
      value->data_type() == MYSQL_TYPE_JSON && !value->basic_const_item())
    return false;
  return true;
}
```

**★ range 优化器显式跳过 FULLTEXT**（`range_optimizer.cc:395`，★ 第十节的硬证据）：

```cpp
      if (key_info->flags & HA_FULLTEXT) {
        trace_idx_details.add("usable", false).add_alnum("cause", "fulltext");
        continue;  // ToDo: ft-keys in non-ft ranges, if possible   SerG
      }
```
> ★ 所以 **FTS 永远不可能产生 range scan**，也永远不可能和 range 做 index merge——注释说明这是"至今没实现"而非设计必然。

### 七、ICP：LIKE 在引擎内求值

LIKE 是 ICP 最典型的受益者。判定与形态**无关**（不是"看是不是 LIKE"，而是通用的"字段是否都在该索引里"——`uses_index_fields_only`，`sql_optimizer.cc:6509`）。

**但 ICP 的调用点只有两处**（`sql_select.cc:3320-3390`），这是最易搞错的地方：

```cpp
case JT_EQ_REF: case JT_REF_OR_NULL: case JT_REF: case JT_SYSTEM: case JT_CONST:
  if (覆盖索引) table->set_keyread(true);
  else qep_tab->push_index_cond(...);        // ★ REF 族（非覆盖）才推
  break;
case JT_ALL: ... [[fallthrough]];
case JT_INDEX_SCAN:                          // ★ 不推 ICP
  ...
case JT_RANGE: case JT_INDEX_MERGE:
  if (!table->no_keyread && type == JT_RANGE) {     // ★ 排除 INDEX_MERGE
    if (覆盖) set_keyread(true);
    if (!table->key_read) qep_tab->push_index_cond(...);   // RANGE（非覆盖）才推
  }
```

| 访问类型 | 推 ICP？ |
|---|:---:|
| `JT_REF` 族（非覆盖） | ✅ |
| `JT_RANGE`（非覆盖） | ✅ |

**★★★ 整个 ICP 的枢纽只有 5 行：引擎直接调 server 的 `Item::val_int()`**（`ha_innodb.cc:23502`）：

```cpp
/** InnoDB index push-down condition check
 @return ICP_NO_MATCH, ICP_MATCH, or ICP_OUT_OF_RANGE */
ICP_RESULT innobase_index_cond(ha_innobase *h) {
  assert(h->pushed_idx_cond);
  assert(h->pushed_idx_cond_keyno != MAX_KEY);

  if (h->end_range && h->compare_key_icp(h->end_range) > 0) {
    /* caller should return HA_ERR_END_OF_FILE already */
    return ICP_OUT_OF_RANGE;
  }

  // note: handler 用了 sql 层的 Item
  return h->pushed_idx_cond->val_int() ? ICP_MATCH : ICP_NO_MATCH;
}
```
> ★★ **下推给引擎的不是编译后的字节码，也不是引擎自己实现的比较器，而是一棵活的 Item 树**——`Item_func_like::val_int()` 会在 InnoDB 的行循环里被**逐行调用**。这解释了为什么 ICP 能支持 LIKE，也解释了代价：引擎与 server 之间没有 ABI 隔离。

**★ 返回值枚举只有 3 个**（`include/my_icp.h:35`，⚠️ **不存在** `ICP_COND_USES_ONLY_INDEX_FIELDS`）：

```cpp
typedef enum icp_result {
  /** Index tuple doesn't satisfy the pushed index condition (the engine
  should discard the tuple and go to the next one) */
  ICP_NO_MATCH,
  /** Index tuple satisfies the pushed index condition (the engine should
  fetch and return the record) */
  ICP_MATCH,
  /** Index tuple is out of the range that we're scanning, e.g. if we're
  scanning "t.key BETWEEN 10 AND 20" and got a "t.key=21" tuple (the engine
  should stop scanning and return HA_ERR_END_OF_FILE right away). */
  ICP_OUT_OF_RANGE
} ICP_RESULT;
```
> ★ "只用索引字段"的判定走的是**另一个机制**：`Item::MARKER_ICP_COND_USES_INDEX_ONLY`（`sql/item.h:3446`），是 Item 的 marker 而非 handler 返回值。

**★ InnoDB 全盘接受，不挑拣**（`ha_innodb.cc:23820`）：

```cpp
class Item *ha_innobase::idx_cond_push(uint keyno, class Item *idx_cond) {
  assert(keyno != MAX_KEY);
  assert(idx_cond != nullptr);

  pushed_idx_cond = idx_cond;
  pushed_idx_cond_keyno = keyno;
  in_range_check_pushed_down = true;
  /* We will evaluate the condition entirely */
  return nullptr;              /* ★★ nullptr = "整个条件我都求值" */
}
```
> ★ 因为 server 侧 `make_cond_for_index()` 已经保证条件只用索引字段，引擎没有再挑拣的动机。
> ★ 基类默认实现（`sql/handler.h:5933`）是 `return idx_cond`（原样退回 = 不接管），所以不支持 ICP 的引擎什么都不用做。

**★ 下推不是免费的：求值前必须物化索引项**（`row0sel.cc:3810`）：

```cpp
  /* Convert to MySQL format those fields that are needed for
  evaluating the index condition. */
  if (prebuilt->blob_heap != nullptr) {
    mem_heap_empty(prebuilt->blob_heap);
  }
  for (i = 0; i < prebuilt->idx_cond_n_cols; i++) {
    const mysql_row_templ_t *templ = &prebuilt->mysql_template[i];
    if (templ->is_virtual) continue;      /* 跳过虚拟列 */
    if (!row_sel_store_mysql_field(mysql_rec, prebuilt, rec, prebuilt->index,
                                   prebuilt->index, offsets,
                                   templ->icp_rec_field_no, templ,
                                   ULINT_UNDEFINED, nullptr,
                                   prebuilt->blob_heap)) {
      return (ICP_NO_MATCH);
    }
  }
```
> ★ 对应 handler.h 注释要求的 "it must read the content of the index entry into the record buffer"。

**★ 未下推时零开销**（`row0sel.cc:3800`）：

```cpp
  if (!prebuilt->idx_cond) {
    return (ICP_MATCH);        /* ★ 短路 */
  }
  MONITOR_INC(MONITOR_ICP_ATTEMPTS);
```

**★★ ICP 的一个正确性妥协**（`row0sel.cc:3826`，★ 值得记住）：

```cpp
  /* We assume that the index conditions on
  case-insensitive columns are case-insensitive. The
  case of such columns may be wrong in a secondary
  index, if the case of the column has been updated in
  the past, or a record has been deleted and a record
  inserted in a different case. */
  result = innobase_index_cond(prebuilt->m_mysql_handler);
```
> ★★ CI 列在二级索引里的 case 可能是"脏的"（历史 UPDATE 过 / 删后插入不同大小写），所以 **ICP 在二级索引上做 CI 比较时结果可能与聚簇索引不一致**。

**★★ ICP 收益最大的一处：省掉回表 + undo 遍历**（`row0sel.cc:5394`）：

```cpp
      if (!srv_read_only_mode &&
          !lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        /* We should look at the clustered index.
        However, as this is a non-locking read,
        we can skip the clustered index lookup if
        the condition does not match the secondary
        index entry. */
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH:
            goto next_rec;                /* ★★ 省掉一次随机 IO + undo 链遍历 */
          case ICP_OUT_OF_RANGE:
            err = DB_RECORD_NOT_FOUND;
            goto idx_cond_failed;
          case ICP_MATCH:
            goto requires_clust_rec;
        }
      }
```

**★★ `uses_index_fields_only`：LIKE 能被下推的真正原因**（`sql/sql_optimizer.cc:6509`）：

```cpp
    case Item::FUNC_ITEM: {
      Item_func *item_func = (Item_func *)item;
      const Item_func::Functype func_type = item_func->functype();

      if (func_type == Item_func::DD_INTERNAL_FUNC) return false;
      if (func_type == Item_func::TRIG_COND_FUNC && ...) return false;

      /* This is a function, apply condition recursively to arguments */
      if (item_func->argument_count() > 0) {
        Item **item_end = (item_func->arguments()) + item_func->argument_count();
        for (Item **child = item_func->arguments(); child != item_end; child++) {
          if (!uses_index_fields_only(*child, tbl, keyno, other_tbls_ok))
            return false;
        }
      }
      return true;      /* ★★ 不看 functype 是不是 LIKE_FUNC，只递归检查参数 */
    }
    ...
    case Item::FIELD_ITEM: {
      const Item_field *item_field = down_cast<const Item_field *>(item);
      if (item_field->field->table != tbl) return other_tbls_ok;
      return item_field->field->part_of_key.is_set(keyno) &&
             item_field->field->type() != MYSQL_TYPE_GEOMETRY &&
             item_field->field->type() != MYSQL_TYPE_BLOB;
    }
```
> ★★ **`FUNC_ITEM` 分支不看 functype** —— `LIKE`、`=`、`>`、甚至 `UPPER(col) LIKE ...` 在 ICP 眼里**一视同仁**。所以**没有针对 LIKE 的特殊代码 = LIKE 天然可下推**。
> ★ 反例：BLOB / GEOMETRY 字段被显式排除（索引的是前缀，不完整）。
> ⚠️ **真名核查**：该函数在 **`sql/sql_optimizer.cc:6509`**；ICP 的**判断入口**是 **`QEP_TAB::push_index_cond()`（`sql/sql_select.cc:2920`）**，不在 `sql_optimizer.cc`。
| `JT_ALL` | ❌ |
| `JT_INDEX_SCAN` | ❌ |
| `JT_INDEX_MERGE` | ❌ |
| 聚簇索引 / CONST/SYSTEM / 多表 UPDATE·DELETE / 虚拟生成列索引 | ❌ |
| hypergraph 优化器 | **完全没有 ICP**（`join_optimizer/` 里无 `pushed_idx_cond`） |

**收益的本质是"少回表"**：

```
row_search_mvcc 扫到一条二级索引记录
  → row_search_idx_cond_check()          row0sel.cc:3782
      → 只把 ICP 需要的列转成 MySQL 格式（idx_cond_n_cols，row0sel.cc:3810）
      → innobase_index_cond()            ha_innodb.cc:23508
          → pushed_idx_cond->val_int()   → Item_func_like::val_int → my_wildcmp
      → ICP_NO_MATCH  ⇒ goto next_rec    ★ 不回聚簇索引
      → ICP_MATCH     ⇒ 才回表
```

所以三种情形要分清：

| 场景 | 访问类型 | ICP | 说明 |
|---|---|---|---|
| `col LIKE '%abc'`，有覆盖索引 | `JT_INDEX_SCAN` + keyread | ❌ | 本来就不需要回表，ICP 无用武之地 |
| `col LIKE '%abc'`，无覆盖索引 | `JT_ALL` | ❌ | 全表扫 + server 层 filter |
| **`a=5 AND b LIKE '%abc'`，`idx(a,b)` 非覆盖** | `JT_REF`/`JT_RANGE` | ✅ | ★ **`LIKE '%abc'` 唯一能吃到 ICP 的形态** |
| `col LIKE 'abc%'`（非覆盖） | `JT_RANGE` | ✅ | range 收窄 + ICP 过滤区间内多余行 |

默认 **ON**（`optimizer_switch` 默认集合含 `index_condition_pushdown`，`sys_vars.cc:202`；bit 定义在 `sql_const.h:197`）。引擎前提是 `HA_DO_INDEX_COND_PUSHDOWN`（InnoDB 普通索引都有，`ha_innodb.cc:6370`；FULLTEXT 为 0）。

### 八、代价估算：11.11% 与"用不上直方图"

```cpp
// Item_func_like::get_filtering_effect   item_cmpfunc.cc:6244
/* Filtering effect is similar to that of BETWEEN because
 * "col like abc%" is similar to "col between abc and abd" ...
 * Now we're left with "col like <string_no_wildcards>" which should
 * have filtering effect like equality, but it would be costly to
 * look through the whole string searching for wildcards ... */
return fld->get_cond_filter_default_probability(rows_in_table, COND_FILTER_BETWEEN);
```

- `COND_FILTER_BETWEEN = 0.1111f`（`sql/item.h:108`）
- 实际取 `max(1 / rows_in_table, 0.1111)`（`Item_field::get_cond_filter_default_probability`）
- ⇒ **LIKE 的过滤系数固定约 11.11%，与模式形态完全无关**

**★ `11.11%` 的精确出处**（⚠️ **不在** `opt_range.cc` 也不在 `sql_optimizer.cc`）：

```cpp
/* sql/item.h:93 —— 一组"没有更好估计时"的默认过滤系数 */
#define COND_FILTER_ALLPASS 1.0f
#define COND_FILTER_EQUALITY 0.1f         /* 等值 = 10% */
#define COND_FILTER_INEQUALITY 0.3333f    /* 不等值 = 33.33% */
#define COND_FILTER_BETWEEN 0.1111f       /* ★★ sql/item.h:108 —— 这就是 11.11% */
#define COND_FILTER_STALE -1.0f
```
> ★ 注意 `COND_FILTER_BETWEEN` 是**通用常量**，LIKE 只是"借用"了它。

**★ 对照物：IS NOT NULL 是用了直方图的**（`item_cmpfunc.cc:6215`，★ 同位置形成鲜明对比）：

```cpp
  double selectivity;
  if (!get_histogram_selectivity(thd, fld->field, args, arg_count,
                                 histograms::enum_operator::IS_NOT_NULL, this,
                                 fld->field->table->s, &selectivity))
    return static_cast<float>(selectivity);
```

**★★ 为什么用不上直方图：算子枚举里根本没 LIKE**（`sql/histograms/histogram.h:151`）：

```cpp
/**
  The different operators we can ask histogram statistics for selectivity
  estimations.
*/
enum class enum_operator {
  EQUALS_TO,
  GREATER_THAN,
  LESS_THAN,
  IS_NULL,
  IS_NOT_NULL,
  LESS_THAN_OR_EQUAL,
  GREATER_THAN_OR_EQUAL,
  NOT_EQUALS_TO,
  BETWEEN,
  NOT_BETWEEN,
  IN_LIST,
  NOT_IN_LIST
};                 /* ★ 13 个值，没有任何字符串模式匹配项 */
```
> ★★ **结论**：LIKE 的过滤系数在架构上只能拿到硬编码的 `0.1111`，直方图在**类型系统层面**就不支持它。

**★ `get_filtering_effect()` 的完整实现**（`item_cmpfunc.cc:6244`，★ 注释里那句"'%abc' 在逆序意义下也等价"很有意思）：

```cpp
  const Item_field *fld =
      contributes_to_filter(read_tables, filter_for_table, fields_to_ignore);
  if (!fld) return COND_FILTER_ALLPASS;   /* 非单表单列 LIKE（如 t1.a LIKE t2.b）→ 全通过 */
  /*
    Filtering effect is similar to that of BETWEEN because
    * "col like abc%" is similar to "col between abc and abd"
      The same applies for 'abc_'
    * "col like %abc" can be seen as
      "reverse(col) like cba%" (see above)
    ...
    Now we're left with col like string_with_no_wildcards which should
    have filtering effect like equality, but it would be costly to
    look through the whole string searching for wildcards and since
    LIKE is mostly used for wildcards this isn't checked.
  */
  return fld->get_cond_filter_default_probability(rows_in_table,
                                                  COND_FILTER_BETWEEN);
```
> ★ 明确的性能取舍：`LIKE 'abc'`（无通配符）本应按等值 10% 估，但为了省一次全串扫描，**故意不检查**，统一按 BETWEEN。

转了 range 的情况另算：走 `records_in_range` → InnoDB 的 `btr_estimate_n_rows_in_range`（两次下潜 + 按路径分叉估算叶子页数）。但 LIKE 不是 `EQ_RANGE`，所以不会走"用统计代替 index dive"的捷径（那条捷径只在 `is_eq_range_pred` 且 range 数超 `eq_range_index_dive_limit` 时启用）。

### 九、三种执行位置与 EXPLAIN 怎么看

| 位置 | 载体 | EXPLAIN 表现 |
|---|---|---|
| ① range 内定位 | `IndexRangeScanIterator` | `type=range`，`key=...` |
| ② ICP（引擎内过滤） | `pushed_idx_cond` → `innobase_index_cond` | `Extra: Using index condition` |
| ③ server 层过滤 | `FilterIterator`（`AccessPath::FILTER`） | `Extra: Using where` |

对照表：

| SQL 形态 | 访问 | LIKE 求值位置 |
|---|---|---|
| `col LIKE 'abc%'`（有索引，覆盖） | `range` + keyread | ① + ③（无 ICP，覆盖就不需要） |
| `col LIKE 'abc%'`（有索引，非覆盖） | `range` | ① + ② + ③ |
| `col LIKE '%abc'`（有覆盖索引） | `JT_INDEX_SCAN` + keyread | ③ only |
| `col LIKE '%abc'`（无覆盖索引） | `JT_ALL` | ③ only |
| `a=5 AND b LIKE '%abc'`（`idx(a,b)` 非覆盖） | `JT_REF`/`JT_RANGE` | **② 引擎内** |
| `col NOT LIKE 'abc%'` | `JT_ALL`/`JT_INDEX_SCAN` | ③ only |

因为 `inexact = true`，① **必须**搭配 ② 或 ③ 复检——这就是 `type=range` 同时带 `Using where` 的原因。

**★ `Using index condition` 的判定**（`sql/opt_explain.cc:992`，★ 是"实际下推成功"不是"开关打开"）：

```cpp
bool Explain_table_base::explain_extra_common(int range_scan_type, uint keyno) {
  if (keyno != MAX_KEY && keyno == table->file->pushed_idx_cond_keyno &&
      table->file->pushed_idx_cond) {
    StringBuffer<160> buff(cs);
    if (fmt->is_hierarchical() && can_print_clauses()) {
      table->file->pushed_idx_cond->print(explain_thd, &buff, cond_print_flags);
    }
    if (push_extra(ET_USING_INDEX_CONDITION, buff))
      return true;
  }
```
> ★ 条件文本只在 `EXPLAIN FORMAT=JSON`（`is_hierarchical()`）里打印。

**★ `Using where` 来自"下推后的残件"**（`opt_explain.cc:1062`）：

```cpp
  } else if (condition) {
    if (fmt->is_hierarchical() && can_print_clauses()) {
      Lazy_condition *c = new (explain_thd->mem_root) Lazy_condition(condition);
      if (c == nullptr) return true;
      fmt->entry()->col_attached_condition.set(c);
    } else if (push_extra(ET_USING_WHERE))
      return true;
  }
```
> ★ `condition` = `tab->condition_optim()`，是被 `push_index_cond` **重写后的残件**。

**★ 残件回写就是"三种执行位置"的分配点**（`sql/sql_select.cc:3062`）：

```cpp
      if (row_cond) {
        and_conditions(&row_cond, idx_remainder_cond);
        idx_remainder_cond = row_cond;
      }
      set_condition(idx_remainder_cond);   /* ★★ 残件留在 server 层做 WHERE */
      trace_obj->add("table_condition_attached", idx_remainder_cond);
```
> ★★ 所以：**`Using where` = 下推后还剩东西要在 server 层算**；两者**可以同时出现**（条件被切成两半）。

**★ `Using index`（覆盖索引）判定完全独立于 ICP**（`opt_explain.cc:1607`）：

```cpp
        table->key_read || tab->keyread_optim()) {
      ...
        if (push_extra(ET_USING_INDEX)) return true;
```
> ★ 所以 `Using index` 与 `Using index condition` 可同时出现，也可各自单独出现。

**★ 三个 tag 是独立的枚举成员**（`sql/opt_explain_traditional.cc:48`）：

```cpp
static const char *traditional_extra_tags[ET_total] = {
    nullptr,                            // ET_none
    "Using temporary",                  // ET_USING_TEMPORARY
    "Using filesort",                   // ET_USING_FILESORT
    "Using index condition",            // ET_USING_INDEX_CONDITION  ★ 索引 3
    "Using",                            // ET_USING
    "Range checked for each record",    // ET_RANGE_CHECKED_FOR_EACH_RECORD
    "Using pushed condition",           // ET_USING_PUSHED_CONDITION
    "Using where",                      // ET_USING_WHERE              ★ 索引 7
    ...
    "Using index",                      // ET_USING_INDEX              ★ 索引 10
```
> ★ JSON 格式里 `ET_USING_INDEX_CONDITION` 对应 `"index_condition"`（`opt_explain_json.cc:68`）。

**★★ `make_cond_for_index`：AND 逐项挑拣、OR 全有或全无**（`sql/sql_select.cc:2796`）：

```cpp
  if (cond->type() == Item::COND_ITEM) {
    if (((Item_cond *)cond)->functype() == Item_func::COND_AND_FUNC) {
      ...
      while ((item = li++)) {
        Item *fix = make_cond_for_index(item, table, keyno, other_tbls_ok);
        if (fix) {                        /* ★ AND：挑得动的留下，挑不动的丢掉 */
          new_cond->argument_list()->push_back(fix);
          used_tables |= fix->used_tables();
        }
        ...
    } else /* It's OR */
    {
      ...
      while ((item = li++)) {
        Item *fix = make_cond_for_index(item, table, keyno, other_tbls_ok);
        if (!fix) return nullptr;         /* ★★ OR：任一子项不可下推 → 整个 OR 返回 nullptr */
        new_cond->argument_list()->push_back(fix);
      }
```
> ★★ 重要语义差异：`WHERE a LIKE 'x%' OR b = 1`，只要 `b` 不在索引里，**整个 OR 都下不去**。

### 十、与全文检索（FTS）的边界

**LIKE 没有任何全文检索能力**：求值核心只有 `my_wildcmp` 一行——没有分词器、没有停用词、没有倒排表、没有打分、没有词位置。对照 `Item_func_match` 返回的是 `double`（相关度），而 `Item_func_like` 返回的是布尔。

**二者不能互相借用索引**（双向都堵死）：

- range 优化器显式跳过 FULLTEXT 索引（`range_optimizer.cc:395`，`cause: "fulltext"`）
- InnoDB 对 FULLTEXT 索引的 `index_flags` 返回 **0**（`ha_innodb.cc:6366`）⇒ 既无 `HA_READ_RANGE` 也无 `HA_DO_INDEX_COND_PUSHDOWN`

**能力对照**：

| | LIKE | FTS（`MATCH ... AGAINST`） |
|---|---|---|
| 匹配对象 | **字符/码点子串**（不管词边界） | **词（token）**，经分词器切出 |
| 索引 | 普通 B-tree（或全扫） | 倒排索引（11 张辅助表） |
| 前缀 | `LIKE 'abc%'` → B-tree range，字符前缀 | `AGAINST('abc*' IN BOOLEAN MODE)`，词前缀 |
| 打分排序 | 无 | 有（`tf × idf²`） |
| 词距 / 短语 | 无 | 有（`"a b"`、`@N`） |
| 布尔运算符 | 无 | 有（`+ - ~ > < ( )`） |
| 非文本列（数字/日期/JSON/ENUM） | ✅ 可以 | ❌ 只限 CHAR/VARCHAR/TEXT |
| 任意形态子串（`%a%b_`） | ✅ | ❌（FTS 不支持前导通配符） |
| 转义 / 精确语义控制 | ✅ `ESCAPE`、显式 `COLLATE` | ❌ |
| **利用 B-tree 有序性**（顺带满足 ORDER BY / GROUP BY / 前缀分页） | ✅ | ❌（FTS 结果按 rank，不按键序） |
| 事务内可见 | 立即可见 | **提交后才可见**（见 [`../../feat/fts.md`](../../feat/fts.md) §12.5-D） |
| 中文 | 逐字符匹配，能用但无语义 | 需 ngram / MeCab parser |

**交叉点举例**（`col = 'abcdef'`）：

- `col LIKE 'abc%'` → 匹配（字符前缀）
- `AGAINST('abc*' IN BOOLEAN MODE)` → 匹配（词 `abcdef` 以 abc 开头）
- `col = 'xabc'`：`LIKE '%abc'` 匹配，但 `abc*` **不**匹配（不是词开头）

**选型判据**：

- 需要"相关度排序 / 大文本高性能 / 词距"→ FTS
- 需要"任意子串 / 单字符 / 转义 / 非文本列 / 借助索引有序性"→ LIKE
- 只是"包含某个词"且数据量不大：LIKE 全扫往往就够了，但**数据量一大就必须换 FTS**（LIKE 是 O(N) 全扫，FTS 是倒排命中集）

**★★ 架构性差异：一个是 bool 一个是 double，一个有状态一个无状态**（`sql/item_func.h:3396`）：

```cpp
class Item_func_match final : public Item_real_func {   /* ★★ 返回 double 相关度 */
 public:
  Item *against;
  uint key, flags;
  bool score_from_index_scan{false};
  DTCollation cmp_collation;
  FT_INFO *ft_handler;          /* ★★ 有状态、跨行存活的句柄 —— LIKE 完全没有 */
  Table_ref *table_ref;
  /**
     Master item means that if identical items are present in the
     statement, they use the same FT handler. ...
  */
  Item_func_match *master;
  Item *concat_ws;
  String value;
  String search_value;

  bool fix_fields(THD *thd, Item **ref) override;
  bool eq(const Item *, bool binary_cmp) const override;
  longlong val_int() override { return val_real() != 0.0; }
  double val_real() override;   /* ★★ 相关度，不是 0/1 */

  bool fix_index(const THD *thd);
  bool init_search(THD *thd);
  ...
```
> ★★ 对比 `Item_func_like : Item_bool_func`（无状态、返回 bool）。**`FT_INFO *ft_handler` 这个跨行存活的句柄是 FTS 必须走独立 `JT_FT` 访问类型的根本原因**。

**★ 三条 LIKE 没有的硬约束**（`sql/item_func.cc:7569`，★ 这是"为什么不能把 FTS 当 LIKE 的扩展"的答案）：

```cpp
  for (uint i = 0; i < arg_count; i++) {
    item = args[i] = args[i]->real_item();
    if (item->type() != Item::FIELD_ITEM ||
        /* Cannot use FTS index with outer table field */
        item->is_outer_reference()) {
      my_error(ER_WRONG_ARGUMENTS, MYF(0), "MATCH");
      return true;
    }
    ...
  }
  /*
    Check that all columns come from the same table.
    ...
  */
  if ((used_tables_cache & ~INNER_TABLE_BIT) != item->used_tables())
    key = NO_SUCH_KEY;
```
```cpp
  if (Item_func::fix_fields(thd, ref) || fix_func_arg(thd, &against) ||
      !against->const_for_execution()) {
    ...
    my_error(ER_WRONG_ARGUMENTS, MYF(0), "AGAINST");
    return true;
  }
```
> ★★ ① MATCH 参数**必须是裸 `FIELD_ITEM`** 且**不能是外层引用**（LIKE 对参数类型毫无要求，`UPPER(x) LIKE ...` 完全合法）；② 所有列**必须同表**；③ `AGAINST` 参数必须 `const_for_execution()`。

**★ InnoDB 没有 FTS_DOC_ID 时直接放弃覆盖索引**（`item_func.cc:7624`，★ LIKE 从不干涉覆盖索引决策）：

```cpp
  if ((table->file->ha_table_flags() & HA_CAN_FULLTEXT_EXT)) {
    Field *doc_id_field = table->fts_doc_id_field;
    if (doc_id_field)
      update_table_read_set(doc_id_field);
    else {
      for (uint i = 0; i < arg_count; i++)
        update_table_read_set(((Item_field *)args[i])->field);
      /*
        Prevent index only access by non-FTS index if table does not have
        FTS_DOC_ID column, find_relevance does not work properly without
        FTS_DOC_ID value. ...
      */
      table->covering_keys.clear_all();      /* ★★ 全清 */
    }
```

**★ FULLTEXT 索引不支持部分匹配**（`item_func.cc:7723`，★ 对比 LIKE 只要最左前缀就能转 range）：

```cpp
  for (keynr = 0; keynr <= mkeys; keynr++) {
    // partial keys doesn't work
    if (max_cnt < arg_count ||
        max_cnt < table->key_info[ft_to_key[keynr]].user_defined_key_parts)
      continue;
    key = ft_to_key[keynr];
    return false;
  }
```

**★ `val_real()`：走 `FT_INFO::please` 虚函数表**（`item_func.cc:7760`）：

```cpp
double Item_func_match::val_real() {
  if (ft_handler == nullptr) return -1.0;
  TABLE *const table = table_ref->table;
  if (key != NO_SUCH_KEY && table->has_null_row())  // NULL row from outer join
    return 0.0;

  if (get_master()->score_from_index_scan) {
    assert(table->file->ft_handler == ft_handler);
    return ft_handler->please->get_relevance(ft_handler);   /* ① 引擎已算好 */
  }
  if (key == NO_SUCH_KEY) {
    String *a = concat_ws->val_str(&value);
    if ((null_value = (a == nullptr)) || !a->length()) return 0;
    return ft_handler->please->find_relevance(ft_handler, (uchar *)a->ptr(),
                                              a->length());
  }
  return ft_handler->please->find_relevance(ft_handler, table->record[0], 0);  /* ② 临时算 */
}
```

**★ 语法：三种模式 + `AGAINST` 是表达式**（`sql/sql_yacc.yy:10541` / `:11026`）：

```cpp
        | MATCH ident_list_arg AGAINST '(' bit_expr fulltext_options ')'
          {
            $$= NEW_PTN Item_func_match(@$, $2, $5, $6);
          }
```
```cpp
fulltext_options:
          opt_natural_language_mode opt_query_expansion
          { $$= $1 | $2; }              /* FT_NL / FT_EXPAND */
        | IN_SYM BOOLEAN_SYM MODE_SYM
          { $$= FT_BOOL; }
        ;
```
> ★ `bit_expr` 是**表达式**不是字符串字面量——与 LIKE 的 `simple_expr` 不同。

**★ FTS 结果按 rank 排序返回，不是按键序**（`ha_innodb.cc:11098`，★ 与 LIKE 的流式返回根本不同）：

```cpp
  if (result->current == nullptr) {
    if (result->rankings_by_id != nullptr) {
      /* Now that we have the complete result, we
      need to sort the document ids on their rank
      calculation. */
      fts_query_sort_result_on_rank(result);
      result->current =
          const_cast<ib_rbt_node_t *>(rbt_first(result->rankings_by_rank));
    }
  } else {
    result->current = const_cast<ib_rbt_node_t *>(
        rbt_next(result->rankings_by_rank, result->current));
```
> ★ `ft_read()` 迭代的是 `rankings_by_rank` 这棵**红黑树**。

**★ 一个容易被忽略的锁差异**（`ha_innodb.cc:10899`）：

```cpp
  /* FTS queries are not treated as autocommit non-locking selects.
  This is because the FTS implementation can acquire locks behind
  the scenes. This has not been verified but it is safer to treat
  them as regular read only transactions for now. */
  if (!trx_is_started(trx)) {
    ++trx->will_lock;
  }
```
> ★ `MATCH ... AGAINST` 会开启一个**真正的只读事务**，而纯 LIKE 扫描可以是无锁的 autocommit select。

**★ MATCH 不能用于生成列**（`item_func.h:3472`，★ 显式自我禁止）：

```cpp
  bool check_function_as_value_generator(uchar *checker_args) override {
    Check_function_as_value_generator_parameters *func_arg = ...;
    func_arg->banned_function_name = func_name();
    return true;
  }
```

**★ EXPLAIN 里 FTS 独有的 Extra**（`opt_explain.cc:1112`，★ `Ft_hints:` 是 LIKE 永远看不到的）：

```cpp
  if (type == JT_FT &&
      (table->file->ha_table_flags() & HA_CAN_FULLTEXT_HINTS)) {
    StringBuffer<64> buff(cs);
    Ft_hints *ft_hints = tab->ft_func()->get_hints();
    if (ft_hints->get_flags() & FT_SORTED) buff.append("sorted");
    else if (ft_hints->get_flags() & FT_NO_RANKING) buff.append("no_ranking");
    ...
        case FT_OP_GT:
          len = snprintf(buf, sizeof(buf) - 1, "rank > %.0g",
                         ft_hints->get_op_value());
```
> ★ 文案是 `"Ft_hints:"`（`opt_explain_traditional.cc:77`）。

```cpp
/* include/my_base.h:108 —— FULLTEXT 索引的算法标记 */
  HA_KEY_ALG_FULLTEXT = 4 /* FULLTEXT. */
```
> ⚠️ **不存在 `FT_RANGE`**——range 优化器里没有全文相关的 range 类型。

---

## 相关的系统变量

| 变量 / 开关 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `optimizer_switch` 的 `index_condition_pushdown` | **on** | GLOBAL/SESSION | ICP 总开关；关闭后 LIKE 只能在 server 层过滤（`sys_vars.cc:202`，bit 在 `sql_const.h:197`） |
| `NO_BACKSLASH_ESCAPES`（sql_mode） | 关闭 | GLOBAL/SESSION | 开启后 LIKE 的默认转义符变为 **0（无转义符）**，`\` 成为普通字符 |
| `eq_range_index_dive_limit` | 200 | GLOBAL/SESSION | 只影响 `EQ_RANGE`；`LIKE 'abc'`（无通配符）会受它影响，`LIKE 'abc%'` 不会 |
| 列的 `COLLATE` | — | — | 决定大小写/重音语义与走哪份 `wildcmp` 实现（8.0 默认 `utf8mb4_0900_ai_ci` → `my_wildcmp_uca_impl`） |

> 与 FTS 相关的 `innodb_ft_*` / `ft_*` 系列变量见 [`../../feat/fts.md`](../../feat/fts.md)「相关的系统变量」。

**★ FTS 变量补充**（⚠️ **两套并存、默认值不同**，这是最容易踩的点）：

| 变量 | 文件:行 | 默认值 | 说明 |
|---|---|---|---|
| **`ft_min_word_len`** | `sql/sys_vars.cc:2290` | **4** | ⚠️ **8.0.39 仍存在**（MyISAM 专用），`READ_ONLY` |
| `ft_max_word_len` | `sys_vars.cc:2282` | 84 | MyISAM，`VALID_RANGE(10, HA_FT_MAXCHARLEN)` |
| `ft_boolean_syntax` | `sys_vars.cc:2274` | `DEFAULT_FTB_SYNTAX` | `ON_CHECK(check_ftb_syntax)` |
| `ft_query_expansion_limit` | `sys_vars.cc:2298` | **20** | `VALID_RANGE(0, 1000)` |
| `ft_stopword_file` | `sys_vars.cc:2304` | nullptr | `READ_ONLY NON_PERSIST` |
| **`innodb_ft_min_token_size`** | `ha_innodb.cc:22512` | **3** | ★★ 与 `ft_min_word_len` 的 4 **不同** |
| `innodb_ft_max_token_size` | `ha_innodb.cc:22518` | `FTS_MAX_WORD_LEN_IN_CHAR` | — |
| `innodb_ft_cache_size` | `ha_innodb.cc:22496` | 8000000 | `READONLY`，范围 1600000–80000000 |
| `innodb_ft_result_cache_limit` | `ha_innodb.cc:22507` | 2000000000 | 可动态改 |
| `innodb_ft_num_word_optimize` | `ha_innodb.cc:22524` | 2000 | 范围 1000–10000 |
| `innodb_ft_sort_pll_degree` | `ha_innodb.cc:22530` | 2 | 范围 1–16 |
| `innodb_ft_enable_stopword` | `ha_innodb.cc:1088` | **true** | ★ **SESSION** 级（`MYSQL_THDVAR_BOOL`） |
| `innodb_ft_user_stopword_table` | `ha_innodb.cc:1098` | 空 | ★ **SESSION** 级（`MYSQL_THDVAR_STR`） |

```cpp
/* sql/sys_vars.cc:2290 —— MyISAM 侧，改了必须重建索引 */
static Sys_var_ulong Sys_ft_min_word_len(
    "ft_min_word_len",
    "The minimum length of the word to be included in a FULLTEXT index. "
    "Note: FULLTEXT indexes must be rebuilt after changing this variable",
    READ_ONLY NON_PERSIST GLOBAL_VAR(ft_min_word_len), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, HA_FT_MAXCHARLEN), DEFAULT(4), BLOCK_SIZE(1));
```
```cpp
/* storage/innobase/handler/ha_innodb.cc:22512 —— InnoDB 侧，默认值 3 */
static MYSQL_SYSVAR_ULONG(
    ft_min_token_size, fts_min_token_size, PLUGIN_VAR_RQCMDARG,
    "InnoDB Fulltext search minimum token size in characters", nullptr,
    innodb_ft_min_token_size_validate, 3, 0, 16, 0);
```
> ⚠️ **不存在 `like_range_*` 系列系统变量。**

**★ `optimizer_switch` 的 ICP flag**（`sql/sys_vars.cc:3453`，★ 名字数组必须与 `sql_const.h` 同序）：

```cpp
/* 注释原话：@b BEWARE! These must have the same order as the #defines in sql_const.h! */
static const char *optimizer_switch_names[] = {
    "index_merge", "index_merge_union", "index_merge_sort_union",
    "index_merge_intersection", "engine_condition_pushdown",
    "index_condition_pushdown",      /* ★★ 就是它 */
    ...
```
> ★ `HINT_UPDATEABLE` ⇒ 支持 `/*+ NO_ICP(t1) */` 按表关闭。判定代码：`hint_key_state(thd, table_ref, keyno, ICP_HINT_ENUM, OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN)`（`sql_select.cc:3010`）。

---

## Misc

### 易混淆概念对比

| 对比 | 说明 |
|---|---|
| **`my_wildcmp` vs `wild_compare`** | `my_wildcmp`（`m_ctype.h:761`）是 SQL `LIKE` 用的；`wild_compare` / `wild_case_compare`（`mf_wcomp.h`）**与 LIKE 无关**，只用于 `GRANT` / `SHOW` 里库表名的通配匹配，且已标注 deprecated、不感知字符集 |
| **`LIKE` vs `RLIKE`/`REGEXP`** | 完全不同的实现：LIKE 是通配符回溯，REGEXP 是正则引擎（ICU）。二者语法、能否用索引（都不能）、`NOT` 的处理方式都不同 |
| **`SOUNDS LIKE`** | 与 LIKE 无关，是 `SOUNDEX(a) = SOUNDEX(b)` 的语法糖（`item_func_eq`） |
| **LIKE 的尾部空格 vs `=` 的尾部空格** | `'abc ' LIKE 'abc'` = 0，`'abc ' = 'abc'` = 1（PAD SPACE 下）。LIKE 严格全长匹配，不裁剪尾部空格 |
| **`Item_func_like` 的位置** | 在 `sql/item_cmpfunc.h:2347`，**不在** `item_func.h` |
| **`fix_length_and_dec` vs `resolve_type`** | 8.0 已改名；`Item_func_like::fix_length_and_dec` 不存在 |
| **`HA_READ_PREFIX_LAST`** | 不参与 LIKE→range，只用于 MIN/MAX、loose scan、反向扫描、auto_increment |
| **`_` 匹配"字符"而非"字节"** | `INC_PTR` 宏：单字节 `(A)++`（`ctype-simple.cc:751`）；多字节 `A += my_ismbchar(...)`（`ctype-mb.cc:222`） |

**★ `REGEXP` / `RLIKE` 的真名与实现**（⚠️ 常见说法 `Item_func_regexp` 不完整）：

```cpp
/* sql/item_regexp_func.h:63 —— ★ 8.0 的 REGEXP 走 ICU，不是老的 Henry Spencer regex */
#include <unicode/uregex.h>
```
```cpp
/* :86 —— Item_func_regexp 是抽象基类 */
/**
  Base class for all regular expression function classes. Is responsible for
  creating the Regexp_facade object.
*/
class Item_func_regexp : public Item_func {
  ...
  /// Resolves the collation to use for comparison. ...
  /// For all regular expression functions, i.e. REGEXP_INSTR, REGEXP_LIKE,
  /// REGEXP_REPLACE and REGEXP_SUBSTR, it goes that the first two arguments
  /// have to agree on a common collation. This collation is used to control
  /// case-sensitivity.
```
```cpp
/* :285 —— ★ REGEXP / RLIKE 用的是这个子类 */
class Item_func_regexp_like : public Item_func_regexp {
 public:
  Item_func_regexp_like(const POS &pos, PT_item_list *opt_list)
      : Item_func_regexp(pos, opt_list) {
    set_data_type_bool();
  }
  ...
  const char *func_name() const override { return "regexp_like"; }
  bool is_bool_func() const override { return true; }
```
> ★★ 继承链差异：`Item_func_like` 直接继承 `Item_bool_func`；`Item_func_regexp_like` 继承 `Item_func_regexp`（因为要在 `fix_fields()` 里建 ICU facade），再靠 `set_data_type_bool()` 伪装成 bool。
> ★ `func_name()` 返回 `"regexp_like"` —— 所以 `EXPLAIN FORMAT=JSON` 里看到的是 `regexp_like(...)`。

| 维度 | LIKE | `=` | `REGEXP`/`RLIKE` |
|---|---|---|---|
| Item 类 | `Item_func_like` | `Item_func_eq` | `Item_func_regexp_like` |
| 基类 | `Item_bool_func` | `Item_bool_func`（经 `Item_bool_func2`） | `Item_func_regexp` → `Item_func` |
| 匹配引擎 | 自研通配（charset 的 `wildcmp`） | `strnncollsp()` | **ICU `uregex`** |
| 能否转 range | 有前缀常量时可 | 可 | **不可** |

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| LIKE→range 在 `sql/opt_range.cc` 的 `get_mm_leaf()` | **文件不存在** → 已拆包为 `sql/range_optimizer/`；`get_mm_leaf()` 在 `range_analysis.cc:1370`，LIKE 分支在 `:1455` |
| `val_int()` 有"常量/非常量/特殊优化"三个分支 | **不存在** → 8.0.39 是**单一统一实现**（`item_cmpfunc.cc:6271`） |
| LIKE 用 `wild_compare()` | **不用于 LIKE** → 定义在 `mysys/mf_wcomp.cc:121`，注释自述 "not charset-aware / deprecated"；LIKE 走 `my_wildcmp` 宏 |
| escape 成员叫 `escape_item` | **不存在** → 真名 `int m_escape`；`escape_item` 只是局部变量 |
| `Item_func_like::fix_length_and_dec()` | **不存在** → 真名 `resolve_type()`（`item_cmpfunc.cc:682`） |
| 有 `cmp_collation` 成员 | **不存在独立成员** → `cmp.cmp_collation.collation`（`Arg_comparator cmp`） |
| `Field::optimize_range()` 返回 `OPTIMIZE_*` | **返回 `bool`**（`field.cc:2137`）；`OPTIMIZE_*` 属于 `Item_func::optimize_type`（`item_func.h:309`） |
| `11.11%` 在 `opt_range.cc` / `sql_optimizer.cc` | **都不在** → `sql/item.h:108` `#define COND_FILTER_BETWEEN 0.1111f` |
| `LIKE '%x'` 是因为 `my_like_range()` 失败 | **错** → `my_like_range` 对 `'%x'` 能算（返回全量区间）；真正拦截在 `select_optimize()` 返回 `OPTIMIZE_NONE`，`get_mm_tree()` 立即 `return nullptr`（`range_analysis.cc:904`） |
| LIKE 能利用直方图 | **不能** → `enum_operator` 13 个值**没有 LIKE**；`get_filtering_effect()` 根本没调 `get_histogram_selectivity()` |
| `_` 匹配一个字节 | **匹配一个字符** → 多字节用 `INC_PTR` 整体跳过 `my_ismbchar()` 的字节数 |
| `ICP_COND_USES_ONLY_INDEX_FIELDS` 返回值枚举 | **不存在** → 返回值枚举是 `ICP_RESULT{ICP_NO_MATCH,ICP_MATCH,ICP_OUT_OF_RANGE}`（`include/my_icp.h:35`）；"只用索引字段"走 `Item::MARKER_ICP_COND_USES_INDEX_ONLY` |
| ICP 判断在 `sql/sql_optimizer.cc` | **错** → 判断入口是 `QEP_TAB::push_index_cond()`（`sql/sql_select.cc:2920`） |
| `sql/sql_executor.cc` 的 `do_icp_` | **不存在** → 该文件与 ICP 相关的只有 `case JT_FT:`（3735） |
| InnoDB 的 `idx_cond_push` 会挑拣条件 | **不会** → 全盘接受，`return nullptr` |
| "LIKE 被 ICP 下推"有专门代码 | **没有** → `uses_index_fields_only()` 对 `FUNC_ITEM` 递归参数一律 true，不区分 LIKE |
| `ft_min_word_len` 已废弃 | **仍存在**（`sys_vars.cc:2290`，默认 4，MyISAM）；InnoDB 用 `innodb_ft_min_token_size`（默认 **3**），两套并存 |
| `REGEXP` 实现类是 `Item_func_regexp` | **是抽象基类** → `REGEXP`/`RLIKE` 用子类 `Item_func_regexp_like`；底层是 **ICU** |
| `MATCH ... AGAINST` 能转 range | **不能** → range 优化器显式 `continue` 跳过 FULLTEXT（注释 `// ToDo: ft-keys in non-ft ranges`） |
| `Item_func_match` 有 `fix_length_and_dec()` | **未检索到** → 走 `fix_fields()` / `fix_index()` / `init_search()` 三段式 |
| 存在 `FT_RANGE` | **不存在** |
| `sql/strings/m_ctype.c`、`ctype-*.c` | 路径已变 → `strings/ctype-simple.cc` / `ctype-mb.cc` / `ctype-bin.cc` / `ctype-uca.cc` / `ctype-utf8.cc` / `ctype-gb18030.cc`（均 `.cc`） |

### 几条实践结论

1. **想让 `LIKE` 走索引，模式就不能以 `%` 或 `_` 开头**——这是唯一有效的改写方向（如业务上可行，用"前缀匹配 + 前置列过滤"替代 `%x%`）。
2. **多个 `%` 且输入很长时有指数级回溯风险**，且防护是"静默返回不匹配"而非报错——对不可信输入要限制模式复杂度或长度。
3. **`EXPLAIN` 里 `type=range` + `Using where` 并存是 LIKE 的正常形态**（`inexact` 导致的复检），不要以为是优化失败。
4. **ICP 的收益是少回表**，所以"覆盖索引"和"ICP"是互斥的两种优化：覆盖了就不推 ICP，因为本来就不回表。
5. `LIKE` 的选择率恒为 11.11%，**优化器无法区分** `'%a'` 和 `'averylongprefix%'`——后者实际过滤性极强但估算一样，可能导致 join order 选错。

---

## 参考

**论文 / 经典算法**

- Knuth, D. E., Morris, J. H., Pratt, V. R. *Fast Pattern Matching in Strings*. SIAM J. Comput. 1977. —— 作为"未被采用"的对照：KMP 的预处理只对纯字面量有效，遇 `%` 即失效，这是 MySQL 选择朴素回溯的合理性依据（代价见「理论基础」）。
- Thompson, K. *Regular Expression Search Algorithm*. CACM 1968. —— 正确的通用做法（NFA 模拟，保证 O(n·m)），MySQL 未采用；这是"多 `%` 指数级回溯"的另一面。
- Unicode Collation Algorithm（UCA 9.0.0）. —— `utf8mb4_0900_ai_ci` 的权重来源，决定 `my_wildcmp_uca_impl` 的比较语义。

**官方文档**

- *MySQL 8.0 Reference Manual → String Comparison Functions and Operators*（`LIKE` 语义、`ESCAPE`、尾部空格说明）
- *MySQL 8.0 Reference Manual → Index Condition Pushdown Optimization*
- *MySQL 8.0 Reference Manual → Full-Text Search Functions*（与 FTS 的边界）

**相关文档**

- 上游（表达式体系）：[`13_item_expression.md`](13_item_expression.md)（`Item` 的类型体系、`resolve_type` 与 `fix_fields` 两阶段）
- 索引侧（ICP / 覆盖索引 / MRR 的通用框架）：[`../../index/access.md`](../../index/access.md)
- 优化器（range 的 SEL_ARG 体系、range 优化器为何跳过 FULLTEXT）：[`07_optimize/`](07_optimize/)
- 对照组（全文检索）：[`../../feat/fts.md`](../../feat/fts.md)——倒排索引、分词、`tf × idf²` 打分、局限清单
