# 11 EXPLAIN 与 optimizer trace：看懂执行计划

> 本篇覆盖：如何把 AccessPath 树打印成人类可读的计划（EXPLAIN），以及 optimizer trace 如何记录优化过程。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、EXPLAIN 的三种输出路径](#一explain-的三种输出路径)
- [二、传统表格格式](#二传统表格格式)
- [三、TREE / JSON：基于 AccessPath](#三tree--json基于-accesspath)
- [四、EXPLAIN ANALYZE](#四explain-analyze)
- [五、optimizer trace](#五optimizer-trace)

---

## 设计思想与理论基础

### 计划呈现的两条路线：重放 vs 转译

EXPLAIN 把 AccessPath 树（08 篇）翻译成人类可读的计划，有两条路线，本篇的结构由此而来：

- **传统表格（TRADITIONAL）**：走 `join->qep_tab`（`QEP_TAB` 链），输出"一张表一行"的扁平表。这是**历史路线**，服务于旧优化器的 QEP 结构
- **TREE / JSON**：直接遍历 AccessPath 树，输出缩进树或 JSON。这是**新路线**，与 hypergraph / AccessPath 对齐

两条路线并存，是因为 `QEP_TAB` 与 AccessPath 两套结构在 8.0 共存（07 篇的"两条创建路径"）。EXPLAIN 必须**同时支持两种数据源**——这是"计划 IR 迁移期"的典型双轨现象。

### 可观测性的两个层次：计划 vs 实际

| 工具 | 回答 | 实现 |
|---|---|---|
| EXPLAIN | "优化器**打算**怎么做" | 静态遍历计划树 |
| EXPLAIN ANALYZE | "实际**真的**这么做了，耗时多少" | **真执行一遍**，`TimingIterator` 计时 |
| optimizer trace | "为什么**选**了这个计划" | 优化器边算边记 JSON 日志 |

三者合起来才构成完整的可观测性：**计划 → 实际 → 决策依据**。这比"只有 EXPLAIN"的数据库（如 5.x 早期）强得多——5.7 引入 optimizer trace、8.0 引入 ANALYZE，都是补"为什么"和"实际上如何"这两块拼图。

### trace 的代价与取舍

optimizer trace 的机制是**在优化器关键路径埋点**（`Opt_trace_object`），默认**关闭**——因为它要记录大量中间量（每个候选计划的代价、每个表的估算行数），开着的开销和日志体积都不小。这是典型的"**可观测性按需开启**"设计，与 `optimizer_switch` 里那些默认关闭的 debug 开关同理。

---

## 先有个整体印象

优化器产出 AccessPath 树（08 篇）后，EXPLAIN 要把它翻译成三种形态：

```
EXPLAIN 输出
 ├─ FORMAT=TRADITIONAL（默认）  遍历 QEP_TAB，一张表一行
 ├─ FORMAT=TREE                遍历 AccessPath 树，缩进树形
 └─ FORMAT=JSON                遍历 AccessPath 树，JSON
```

**EXPLAIN ANALYZE** 还会**真正执行一遍**查询，用 `TimingIterator` 收集每步的实际时间与行数。

> ⚠️ 文件位置：本版本的 explain 代码在 **`sql/opt_explain.cc`**（不是 `explain.cc`）。

### EXPLAIN 的 SQL 变体（有哪些）

| SQL | 说明 |
|-----|------|
| `EXPLAIN SELECT ...` | 默认 TRADITIONAL 表格 |
| `EXPLAIN FORMAT=TREE SELECT ...` | 树形输出 |
| `EXPLAIN FORMAT=JSON SELECT ...` | JSON 输出 |
| `EXPLAIN ANALYZE SELECT ...` | 真执行一遍 + 实际耗时/行数（8.0.18+） |
| `EXPLAIN ANALYZE FORMAT=TREE SELECT ...` | 树形 + actual time |
| `EXPLAIN FOR CONNECTION <id>` | 看其他连接的 plan（已基本废弃，走 `SQLCOM_EXPLAIN_OTHER`） |
| `DESC` / `DESCRIBE` | EXPLAIN 的同义词（`DESC SELECT ...`） |
| `EXPLAIN <table>` | 等价 `DESCRIBE <table>`，看**表结构**（不是执行计划） |

> **注意**：`EXPLAIN <table>` 和 `EXPLAIN SELECT ...` 语义完全不同——前者是 `SHOW COLUMNS` 的别名（看表定义），后者才产生执行计划。别混。

---

## 一、EXPLAIN 的三种输出路径

`explain_query()` 先判断格式（`opt_explain.cc:2226-2256`）：`is_iterator_based()` 为真（TREE/JSON）走 iterator 路径（`ExplainIterator` `:2090`），否则走传统路径（`Explain_join` `:2070`）。

| 格式 | 遍历对象 | 路径 |
|------|----------|------|
| TRADITIONAL | `join->qep_tab`（QEP_TAB） | `Explain_join::shallow_explain`（`opt_explain.cc:1250`） |
| TREE | AccessPath | `PrintQueryPlan` → `ExplainAccessPath` |
| JSON | AccessPath | 同上，输出 JSON |

---

## 二、传统表格格式

`Explain_join::shallow_explain()`（`opt_explain.cc:1250`）遍历 **QEP_TAB**，不是 AccessPath：

```cpp
for (size_t t = 0, cnt = fmt->is_hierarchical() ? join->primary_tables
                                                : join->tables;
     t < cnt; t++) {
  if (explain_qep_tab(t)) return true;
}
```

`explain_qep_tab()`（`:1375`）从 `QEP_TAB` 取 `type`/`condition`/`range_scan`/`keys` 等，逐行输出。

> 这就是为什么传统 EXPLAIN 只能表达**左深树** —— 它本质是 QEP_TAB 数组，每张表一行。

### 2.1 传统表格的 12 列含义

| 列 | 含义 | 打印位置 |
|----|------|---------|
| `id` | 查询块编号（对应 `select#N`） | `explain_qep_tab` |
| `select_type` | 查询类型（`SIMPLE`/`PRIMARY`/`SUBQUERY`/`DERIVED`/`UNION`...） | `Explain_select` |
| `table` | 表名/别名（或 `<derivedN>`/`<unionN>`） | `col_table_name` |
| `type` | 访问类型（`join_type_str[]`） | `col_join_type` |
| `possible_keys` | 候选索引 | `explain_possible_keys` |
| `key` | 实际选的索引 | `explain_key_and_len` |
| `key_len` | 索引使用字节数 | `explain_key_and_len` |
| **`ref`** | **用哪个列/常量匹配索引** | `explain_ref`（见 2.2） |
| `rows` | 估算扫描行数 | `col_rows` |
| `filtered` | WHERE 过滤后剩余比例 | `col_filtered` |
| `Extra` | 附加信息（`Using where`/`Using index`/`Using temporary`...） | `col_extra` |

**`type` 列的取值**（`join_type_str[]`，`opt_explain.cc:115`）：

```cpp
const char *join_type_str[] = {
    "UNKNOWN", "system", "const",    "eq_ref",      "ref",        "ALL",
    "range",   "index",  "fulltext", "ref_or_null", "index_merge"};
```

`type` 列 = **这张表的访问方法**（access method）。它的"怎么选出来"是 07 篇《访问方法选择》的主题（const 检测 / Key_use / EQ_REF 判定 / 代价决斗），这里只讲**每个取值意味着什么、怎么读**。从好到坏：

| type | 含义 | 触发条件（简化） | 迭代器 |
|------|------|----------------|--------|
| `system` | 表只有 0 或 1 行（系统表 / 统计精确 ≤1） | `JT_SYSTEM`（`mark_const_table` 传 `key==nullptr`） | `ConstIterator` |
| `const` | 唯一键等值定位，**最多 1 行，优化期就读出** | `JT_CONST`（`mark_const_table` 传 `key!=nullptr`） | `ConstIterator` |
| `eq_ref` | 每行外层值经**唯一键**精确匹配内层**恰好 1 行** | `JT_EQ_REF` | `EQRefIterator` |
| `ref` | **非唯一键**等值匹配（一次可能匹配多行） | `JT_REF` | `RefIterator` |
| `ref_or_null` | ref + 额外查一次 NULL（`col=x OR col IS NULL`） | `JT_REF_OR_NULL` | `RefOrNullIterator` |
| `range` | 用索引查一个**范围**（`>`/`<`/`BETWEEN`/`IN`） | `JT_RANGE` | 各 range 迭代器 |
| `index` | **全索引扫描**（扫索引而非表，通常覆盖索引避免回表） | `JT_INDEX_SCAN` | `IndexScanIterator` |
| `ALL` | **全表扫描** | `JT_ALL` | `TableScanIterator` |
| `fulltext` | 全文索引 | `JT_FT` | — |
| `index_merge` | 多个索引**合并**（intersection / union / sort_union） | `JT_INDEX_MERGE` | `IndexMergeIterator` |

**`eq_ref` vs `ref` 的精确判定**（`create_ref_for_key`，`sql_select.cc:2512-2536`）——这是最容易混的一对：

```cpp
if ((actual_key_flags(keyinfo) & HA_NOSAME) == 0 ||            // ① 非唯一索引
    ((actual_key_flags(keyinfo) & HA_NULL_PART_KEY) &&
     !null_rejecting_key) ||                                   // ② 唯一但可空，且谓词不拒 NULL
    keyparts != actual_key_parts(keyinfo))                     // ③ 唯一但没用满 keypart
  set_type(JT_REF);                                            // ⇒ ref
else
  set_type(JT_EQ_REF);                                         // ⇒ eq_ref
```

三个退化成 `ref` 的情形：

1. **① 非唯一索引**：一个 key 值可能匹配多行 → 不是"恰好 1 行"，只能是 `ref`。
2. **② 唯一索引但含可空 keypart，且谓词不拒绝 NULL**：MySQL 唯一索引允许**多个 NULL**（`UNIQUE KEY (a)` 里 `a` 可以为 NULL 多行），所以一个 key 值可能匹配多行 → 退化成 `ref`。
3. **③ 唯一索引但没用满 keypart**：复合唯一键 `(a,b)` 只用 `a` 做等值，`a` 相同可能有多个 `b` → 不是唯一定位 → `ref`。

**`const` 的坑**（07 篇 2.1 核实过）：`system` 只在"表定义上就 ≤1 行"时出现；InnoDB 永久表**永远不会**走 `system`（`HA_STATS_RECORDS_IS_EXACT` 只有临时表申报），同样场景 InnoDB 显示 `const`（走唯一键路径）。

> 完整的选择机制（Key_use 数组怎么建、EQ_REF 的 fanout 为什么是 1.0、ref/range/scan 怎么代价决斗、二次 range 重估）见 [07 篇](07_optimize/physical/07_access_method.md)。

### 2.2 ref 列：怎么来的、代码哪里打印、代表什么

这是最容易看不懂的一列。**ref 列回答："用哪个值去匹配索引的每一部分"**。

打印链（`opt_explain.cc`）：

```cpp
// Explain_join::explain_ref（:1527）
bool Explain_join::explain_ref() {
  if (!tab) return false;
  return explain_ref_key(fmt, tab->ref().key_parts, tab->ref().key_copy);
}

// explain_ref_key（:617）
static bool explain_ref_key(Explain_format *fmt, uint key_parts,
                            store_key *key_copy[]) {
  if (key_parts == 0) return false;
  for (uint part_no = 0; part_no < key_parts; part_no++) {
    const store_key *const s_key = key_copy[part_no];
    if (s_key == nullptr) {
      // 常量键不需要拷贝 → 打印 "const"
      fmt->entry()->col_ref.push_back(STORE_KEY_CONST_NAME);
    } else {
      fmt->entry()->col_ref.push_back(s_key->name());   // 列名 或 "func"
    }
  }
  return false;
}
```

每个键部分（key part）对应一个 `store_key`，它的 `name()` 决定打印什么（`sql_select.h:817`）：

```cpp
virtual const char *name() const {
  if (item->type() == Item::FIELD_ITEM) {
    return item->full_name();   // 列引用 → "db.t1.a"
  } else {
    return "func";              // 表达式/函数 → "func"
  }
}
```

所以 ref 列的三种取值：

| ref 值 | 含义 | 触发场景 |
|--------|------|---------|
| `const` | 用常量匹配索引 | `WHERE a = 5`（`s_key == nullptr`，常量不需拷贝） |
| `db.t1.a` | 用另一张表的列匹配 | `t1 JOIN t2 ON t2.a = t1.a`（t2 的索引 a 用 t1.a 匹配） |
| `func` | 用函数/表达式结果匹配 | `WHERE a = func(b)` |
| `NULL` | 没有按值查找 | `type = ALL/index/range` 时（非 ref 类访问） |

**关键理解**：`ref` 列是 `key` 列的"补充说明"——`key` 告诉你"用哪个索引"，`ref` 告诉你"索引每一部分去匹配什么值"。它只对按值查找的访问类型（`const`/`eq_ref`/`ref`/`ref_or_null`）有意义；`type = ALL`（全表扫）时 ref 为 NULL。

**数据来源**：`tab->ref()` 是 `QEP_TAB` 的 `TABLE_REF` 结构，它的 `key_parts`/`key_copy[]` 由优化器在选好访问方法后填入（`pick_table_access_method` 阶段），每个 key part 一个 `store_key`，记录"用哪个 Item 的值构造这个 key part"。

### 2.3 filtered 列：WHERE 过滤后剩余比例（计算逻辑）

`filtered` 回答："这张表读出来的行，经过 WHERE/ON 条件过滤后还剩百分之几"。

打印代码（`explain_rows_and_filtered`，`opt_explain.cc:1532-1549`）：

```cpp
fmt->entry()->col_rows.set(static_cast<ulonglong>(pos->rows_fetched));
fmt->entry()->col_filtered.set(
    pos->rows_fetched                                      // rows_fetched 为 0 时
        ? static_cast<float>(100.0 * tab->position()->filter_effect)  // filtered = 100 × filter_effect
        : 0.0f);                                            //   → filtered 直接 0
```

所以 **`filtered = 100.0 × filter_effect`**。`filter_effect` 是 `POSITION` 上的字段，即"这张表上条件的**过滤系数**"（剩余比例），来源在访问方法选择阶段（07 篇 `best_access_path` / `calculate_scan_cost`）：

- 条件过滤开启（默认 `condition_fanout_filter=on`）→ `calculate_condition_filter` 用常量条件算过滤系数
- 否则 → 各种启发式（如无可用估计时 `found_records * 0.75` 的 25% 过滤假设）

两个注意点：

1. **`filtered` 只有传统 EXPLAIN 有**，且 `EXPLAIN CONNECTION`（`SQLCOM_EXPLAIN_OTHER`）会跳过（设成 NULL）。
2. **`rows × filtered%` 才是这张表真正"传给下一张表"的行数**。多表 join 时，前一张表的 `rows × filtered%` 会作为下一张表 `rows` 的基数来源——所以看 EXPLAIN 时 `rows` 是"扫描行数"，`rows × filtered%` 才是"有效行数"。

### 2.4 Extra 列：附加信息（30+ 种取值）

`Extra` 是"这一行表还有什么额外动作"。它由 `explain_extra`（`opt_explain.cc:1571`）+ `explain_extra_common`（`:992`）逐项 `push_extra(ET_XXX)` 拼出来，取值枚举在 `Extra_tag`（`opt_explain_format.h:60-98`），字符串映射在 `traditional_extra_tags[]`（`opt_explain_traditional.cc:48-84`）。**一个表可以同时挂多个 Extra**（用 `; ` 连接）。

**最核心的几个（必须懂）**：

| Extra | 含义 | 代码 |
|-------|------|------|
| `Using where` | **server 层还要再过滤**。不是"用了 WHERE"，而是"WHERE 里有存储引擎处理不了、返回行后 server 层还要逐行判断的部分"（如非 sargable 谓词、或 ON 转成 WHERE 的残留） | `explain_extra_common :1068`（`condition` 非空） |
| `Using index` | **覆盖索引**：只读索引不回表（`covering_keys.is_set`） | `explain_extra :1602` |
| `Using index condition` | **ICP**（Index Condition Pushdown）：索引条件推给存储引擎在索引层先过滤 | `explain_extra_common :993`（`pushed_idx_cond`） |
| `Using temporary` | 要建**临时表**（GROUP BY/DISTINCT 无法用索引、UNION、子查询等） | `explain_tmptable_and_filesort` |
| `Using filesort` | 要**排序**（ORDER BY 无法用索引满足） | `explain_tmptable_and_filesort` |
| `Using join buffer` | 用 **join buffer**（BNL/hash join/Batched Key Access） | `explain_extra :1664`（`OT_BNL`/`OT_BKA`） |

**`Using where` 和 `Using index condition` 的区别**（最容易混）：

- `Using index condition`（ICP）：条件**下推到存储引擎**，在**索引扫描时**就过滤（如 `WHERE a>5 AND b='x'`，索引 `(a)` 扫 `a>5` 后，`b='x'` 交给引擎在回表前判）。
- `Using where`：条件**没下推**，引擎返回行后 **server 层**再 `Item::val_*` 逐行判断（如条件里有函数、或索引没法承载的部分）。
- 两者可以**同时出现**：`Using where; Using index condition` 表示一部分下推、一部分留在 server 层。

**semi-join / anti-join 相关的（03 篇策略的 Extra 表现）**：

| Extra | 含义 | 对应策略 |
|-------|------|---------|
| `FirstMatch` | 首匹配去重 | FirstMatch 策略（`:1629`） |
| `LooseScan` | 松索引扫描去重 | LooseScan 策略（`:1621`） |
| `Start temporary` / `End temporary` | 临时表去重的起止边界 | DuplicateWeedout 策略（`:1623`/`:1627`） |
| `Not exists` | 外连接 + NOT EXISTS 优化（`LEFT JOIN ... WHERE right IS NULL` 只取无匹配行） | — |
| `Using index for group-by` | 用**松索引扫描**直接做 GROUP BY（不排序） | `explain_extra :1603`（GROUP_INDEX_SKIP_SCAN） |
| `Using index for skip scan` | **index skip scan** | `explain_extra :1608` |

**其余常见取值速查**：

| Extra | 含义 |
|-------|------|
| `Using MRR` | Multi-Range Read：多范围读取按 rowid 排序批量回表 |
| `Distinct` | DISTINCT 去重（`:1618`） |
| `Range checked for each record` | 每行动态决定用哪个索引（dynamic range，`:1062`） |
| `Using pushed condition` | NDB 引擎的条件下推（`:1078`） |
| `Full scan on NULL key` | `ref_or_null` 的 NULL 分支走全扫（`:1661`） |
| `Rematerialize` | lateral derived 表被重复物化（`:1644`） |
| `Impossible ON condition` | const 表的 ON 恒假（`:1580`） |
| `const row not found` / `unique row not found` | const/唯一键定位时没找到行（`:1574`/`:1577`） |
| `Backward index scan` | 反向索引扫描（ORDER BY DESC 用索引倒序） |
| `Recursive` | 递归 CTE（`:91`） |
| `Using secondary engine` | 走了 secondary engine（`:1693`） |

### 2.5 key / key_len / possible_keys：索引三兄弟

**`possible_keys`**（`explain_possible_keys`，`opt_explain.cc:938`）：优化器认为"**可能**用得上"的候选索引——就是 `usable_keys` 位图（`tab->keys()`）里置位的所有索引名：

```cpp
// opt_explain.cc:938
if (usable_keys.is_clear_all()) return false;                       // 无候选 → NULL
if ((table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) != 0) return false;  // 引擎不支持索引
for (uint j = 0; j < table->s->keys; j++) {
  if (usable_keys.is_set(j) && ...push_back(table->key_info[j].name))  // 逐个输出索引名
}
```

**读法**：`possible_keys` 是**候选清单，不保证最终用**。它为空（NULL）说明这张表没有任何索引可用（或引擎不支持索引），只能全表扫。

**`key`**（`explain_key_and_len`，`:1514`）：实际选中的索引。三条数据源：

```cpp
if (tab->ref().key_parts)                              // ref 类：ref 的索引
  → explain_key_and_len_index(tab->ref().key, tab->ref().key_length, tab->ref().key_parts);
else if (type == JT_INDEX_SCAN || type == JT_FT)       // index scan：tab->index()
  → explain_key_and_len_index(tab->index());
else if (type == JT_RANGE || type == JT_INDEX_MERGE ...)  // range：range_scan_path 的索引
  → explain_key_and_len_quick(range_scan_path);
```

**`key_len`**（`:979-989`）：索引使用的字节数。**最容易踩的坑：ref 类是"实际用到的 keypart 长度和"，不是整个索引长度**。它的来源是 `init_ref` 里的逐 keypart 累加（`sql_select.cc:2295`）：

```cpp
length += keyinfo->key_part[keyuse->keypart].store_length;   // 每个用到的 keypart 的存储长度
```

`store_length`（索引内每个 keypart 的存储长度）规则：

| 列类型 | store_length |
|--------|-------------|
| 定长列（`INT`/`DATE`/`CHAR(n)`×charset 字节） | `pack_length()`（如 INT = 4） |
| **可空列** | 上面 **+1 字节**（NULL 标记） |
| **变长列**（VARCHAR/TEXT/BLOB） | 数据长度 **+2 字节**（长度前缀，`HA_KEY_BLOB_LENGTH`） |

**key_len 是读 EXPLAIN 的利器**——`key_len` < 索引总长，说明**只用了索引前缀**（最左前缀没全部命中）：

```sql
CREATE INDEX idx ON t(a, b);          -- a: INT NOT NULL(4), b: VARCHAR(10) utf8mb4(10*4+2=42)
EXPLAIN SELECT * FROM t WHERE a=1 AND b='x';   -- key_len=46（4+42，两列都用了）
EXPLAIN SELECT * FROM t WHERE a=1;             -- key_len=4（只用前缀 a，b 没用上）
```

> 注意 `index` 扫描（全索引扫）时 `key_len` 是**整个索引长度**（`key_info[key].key_length`，`:975`），与 ref 类的"实际用到的长度"口径不同。

---

## 三、TREE / JSON：基于 AccessPath

核心在 **`sql/join_optimizer/explain_access_path.cc`**：

- `SetObjectMembers()`（`:934`）：巨型 `switch (path->type)`，为每个类型生成 `operation` 描述与 JSON 成员
- `ExplainAccessPath()`（`:1665`）：递归构建 JSON 树
- `PrintQueryPlan()`（`:1727`）：生成最终 JSON（`ExplainJsonToString` `:1761`）或树形文本（`Explain_format_tree`）

TREE 输出形如：

```
-> Nested loop inner join  (cost=0.55 rows=0)
    -> Filter: (t1.a = t2.a)  (cost=0.35 rows=1)
        -> Table scan on t1  (cost=0.25 rows=1)
    -> Index lookup on t2  (cost=0.30 rows=1)
```

**`cost`/`rows` 的来源**：AccessPath 的 `cost` 字段（`num_output_rows()`），即优化器估算值（07 篇）。

> ⚠️ `explain_json_format_version`（8.3 加的变量）**本版本不存在**，全库 0 匹配。

---

## 四、EXPLAIN ANALYZE

### 4.1 真正执行一遍

`explain_query()` 在 iterator 路径下先跑一遍查询，但结果丢弃（`opt_explain.cc:2242-2250`）：

```cpp
// Run the query, but with the result suppressed.
Query_result_null null_result;
unit->set_query_result(&null_result);
explain_thd->running_explain_analyze = true;
unit->execute(explain_thd);
```

### 4.2 TimingIterator 计时

`NewIterator()`（`timing_iterator.h:221`）在 `is_explain_analyze` 时把真实迭代器包进 `TimingIterator<RealIterator>`：

```cpp
template <class RealIterator, class... Args>
unique_ptr_destroy_only<RowIterator> NewIterator(THD *thd, MEM_ROOT *mem_root, Args &&... args) {
  if (thd->lex->is_explain_analyze) {
    return ... TimingIterator<RealIterator>(thd, ...);
  } else {
    return ... RealIterator(thd, ...);
  }
}
```

`IteratorProfilerImpl`（`timing_iterator.h:40`）维护：`m_num_init_calls`（loops）、`m_num_rows`、`m_elapsed_first_row`、`m_elapsed_other_rows`。

> 特殊：`MaterializeIterator` 和 `TemptableAggregateIterator` **不用** TimingIterator 包裹，自己内部带 profiler（`composite_iterators.cc:657/1608`），以正确拆分"物化时间"与"扫描时间"。

### 4.3 输出

`explain_access_path.cc:881-908` 从 `path->iterator->GetProfiler()` 取数据写 JSON：`actual_first_row_ms` / `actual_last_row_ms` / `actual_rows` / `actual_loops`。树形输出（`:2010-2058`）：

```
-> Table scan on t1  (cost=... rows=...) (actual time=0.5..1.2 rows=100 loops=1)
```

---

## 五、optimizer trace

### 5.1 开关

`SET optimizer_trace="enabled=on"`，注册在 `sys_vars.cc:3543`。配套变量：`optimizer_trace_features`(:3558)、`optimizer_trace_offset`(:3576)、`optimizer_trace_limit`(:3583)、`optimizer_trace_max_mem_size`(:3589)。

### 5.2 结构

- `Opt_trace_context`（`opt_trace_context.h:90`）：每会话一个，挂 `THD::opt_trace`
- `Opt_trace_object`（`opt_trace.h:799`）/ `Opt_trace_array`（`:829`）：RAII 管理的 JSON 构建器
- 未开启时所有 `add_*` 是快速 no-op
- 顶层是 JSON object，含 `steps` 数组（执行阶段示例 `sql_union.cc:1676-1682`）

### 5.3 特性开关

`feature_enabled()`（`opt_trace_context.h:235`）控制追踪哪些特性：`GREEDY_SEARCH` / `RANGE_OPTIMIZER` / `DYNAMIC_RANGE` / `REPEATED_SUBSELECT` / `MISC`。

### 5.4 输出

查 `information_schema.OPTIMIZER_TRACE`（每 trace 一行，字段 QUERY/TRACE/MISSING_BYTES/INSUFFICIENT_PRIVILEGES），由 `Opt_trace_iterator`（`opt_trace.h:407`）遍历。

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → EXPLAIN Output Format*（传统表格格式各列含义）
- *MySQL 8.0 Reference Manual → EXPLAIN ANALYZE*（ANALYZE 的 `actual time` / `loops` / `rows`）
- *MySQL 8.0 Reference Manual → Tracing the Optimizer*（optimizer trace 的 JSON 结构）

