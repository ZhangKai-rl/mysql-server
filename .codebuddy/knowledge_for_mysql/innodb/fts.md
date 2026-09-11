# InnoDB FTS（全文检索）深度解析

> 基于 MySQL 8.0.39 源码。涵盖 MATCH AGAINST 的 SQL 层实现、FTS 辅助表体系、倒排索引编码、分词、fts_cache 与后台 sync/optimize、布尔模式 AST、相关性排序算法、事务与崩溃恢复。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [一、先会用：三种搜索模式](#一先会用三种搜索模式)
- [二、server 层：MATCH AGAINST 如何走到引擎](#二server-层match-against-如何走到引擎)
- [三、FTS 索引的物理结构：11 张辅助表](#三fts-索引的物理结构11-张辅助表)
- [四、FTS_DOC_ID：文档的身份](#四fts_doc_id文档的身份)
- [五、倒排列表 ilist 与 VLC 编码](#五倒排列表-ilist-与-vlc-编码)
- [六、写入路径与 fts_cache](#六写入路径与-fts_cache)
- [七、分词（tokenize）与 ngram](#七分词tokenize与-ngram)
- [八、后台线程：sync 与 optimize](#八后台线程sync-与-optimize)
- [九、删除与空间回收](#九删除与空间回收)
- [十、查询路径与布尔模式 AST](#十查询路径与布尔模式-ast)
- [十一、相关性排序（rank）算法](#十一相关性排序rank算法)
- [十二、事务、崩溃恢复与 DDL](#十二事务崩溃恢复与-ddl)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

InnoDB FTS 是**倒排索引（inverted index）**在 InnoDB 中的实现：把每个文档切成词，建立 `word → [doc_id, position...]` 的映射，查询时按词找到文档并按相关性打分。

它不是 B-tree 索引的变体，而是**一套独立的存储子系统**：由 11 张"辅助表"（auxiliary table）承载，加上一块内存倒排表（`fts_cache_t`）和一个后台线程。

### 用途

- 文本字段的关键词搜索（替代 `LIKE '%x%'`，后者无法用索引）
- 带相关性排序的搜索（这是 FTS 相对 LIKE 的核心价值）
- 中文/日文场景通过 ngram parser 支持（无需分词词典）

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | **InnoDB 支持 FULLTEXT 索引**（此前只有 MyISAM） |
| 5.7.6 | 引入 **ngram parser**（CJK 全文检索，按 N 字滑窗切词）；MeCab parser（日文） |
| 8.0 | FTS 元数据并入 Data Dictionary；**隐藏的 FTS_DOC_ID 列与索引自动创建**；建索引改用并行分词/归并（`ddl/ddl0fts.cc`，取代旧的 `row0ftsort.cc`） |

---

## 理论基础

### 算法与数据结构

| 算法/结构 | 体现 |
|-----------|------|
| **倒排索引** | `word → (doc_id, positions)`，存于 `FTS_*_index_1..6` |
| **VLC 变长整数 + 增量编码** | ilist 里 doc_id 与 position 都存"与前一个的差值"，再按 7bit 变长编码（与 JSONB 的 varint 同族思路） |
| **红黑树** | 内存 cache 的 `words` 树；查询结果的 `rankings_by_id` / `rankings_by_rank` 两棵树 |
| **位图（bitmap）** | `fts_ranking_t::words` —— 用 bitmap 记录"该文档命中了哪些词"，替代每个文档一棵红黑树，省内存（`fts0que.cc:518-592`） |
| **归并扫描** | 布尔模式的集合交/并/差；proximity 的多路位置归并（`fts0que.cc:4157`） |
| **TF-IDF** | 打分公式，见第十一章（**注意是 tf × idf²，不是 BM25**） |

### 设计模式

| 模式 | 体现 |
|------|------|
| **插件 / 策略** | 分词器是 `st_mysql_ftparser` 插件（内置 / ngram / MeCab 可换），`fts0plugin.cc` |
| **生产者-消费者** | DML 产生 token 入 cache，后台 `fts_optimize_thread` 消费（sync） |
| **延迟写 / 批量刷** | 新增文档不立即写辅助表，先入内存 cache，攒够再批量 INSERT |
| **快照 + 分阶段提交** | OPTIMIZE 时 `DELETED → BEING_DELETED` 快照，再逐索引重写，最后统一清理（保证崩溃可续做） |
| **墓碑（tombstone）** | 删除只记 doc_id 到 DELETED 表，查询时过滤，真正清除留给 OPTIMIZE |
| **访问者** | 布尔 AST 用 `fts_ast_visit()` 遍历求值（`fts0ast.cc:520`） |

### 类似实现对比

| 系统 | 差异 |
|------|------|
| **Lucene / Elasticsearch** | 用**不可变 segment** + 周期性 merge；打分是 **BM25**（有 k1/b 参数、有文档长度归一化、tf 饱和）。索引与数据分离 |
| **InnoDB FTS** | 倒排表就是**普通 InnoDB B-tree 表**，随用户表一起事务恢复；打分是 **tf × idf²**（无归一化、无饱和）；删除靠墓碑表 + OPTIMIZE 重写 |
| **PostgreSQL tsvector** | `tsvector` 类型 + GIN 索引，支持 ts_rank（多种归一化）；支持前缀匹配与权重标签 |
| **MyISAM FT** | 老实现，词长受 `ft_min_word_len` 控制，无事务，表级锁 |

### 历史背景

5.6 之前 InnoDB 表要做全文检索，只能"挂一个 MyISAM 从表存文本 + 在 MyISAM 上建 FULLTEXT"，再 JOIN 回来 —— 事务、崩溃恢复、复制全是坑。5.6 把 FTS 做进 InnoDB 的核心难点是：**倒排索引不能建成一棵普通 B-tree 索引**（它不是行级的一列一值），于是 InnoDB 选择了"**用一堆普通表来模拟倒排索引**"的路线 —— 这就是辅助表体系的由来，也决定了它后续所有特性（内存 cache、后台 sync、墓碑删除、OPTIMIZE 重写）。

---

## 一、先会用：三种搜索模式

### 1.1 建索引

```sql
CREATE TABLE articles (
  id      INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
  title   VARCHAR(200),
  body    TEXT,
  FULLTEXT (title, body)          -- 也可以 WITH PARSER ngram
) ENGINE=InnoDB;
```

> 8.0 会自动加一个**隐藏**的 `FTS_DOC_ID BIGINT UNSIGNED NOT NULL` 列和隐藏的唯一索引 `FTS_DOC_ID_INDEX`，不需要自己建（见第四章）。

### 1.2 三种模式

| 模式 | 写法 | 语义 |
|------|------|------|
| **自然语言**（默认） | `MATCH(t,b) AGAINST('数据库')` | 按相关性打分，返回 rank > 0 的行 |
| **布尔** | `AGAINST('+MySQL -Oracle' IN BOOLEAN MODE)` | 支持操作符，只判命中不排序 |
| **查询扩展** | `AGAINST('database' WITH QUERY EXPANSION)` | 两遍搜索：先搜，再把命中文档的词加入查询重搜 |

```sql
-- 自然语言：按相关性排序输出
SELECT id, MATCH(title, body) AGAINST('database') AS score
FROM articles
WHERE MATCH(title, body) AGAINST('database')
ORDER BY score DESC;
```

### 1.3 布尔模式操作符

| 操作符 | 含义 | 例子 |
|--------|------|------|
| （无） | 可选，出现则加分 | `apple banana` |
| `+` | **必须包含** | `+apple +juice` |
| `-` | **必须不包含** | `apple -macintosh` |
| `~` | 出现则**降权**（负相关） | `~apple` |
| `>` `<` | 提高 / 降低该词对 rank 的贡献 | `+apple >juice` |
| `*` | 前缀通配（放词尾） | `data*` |
| `" "` | 短语（引号内视为整体） | `"some words"` |
| `@N` | 距离：两词之间最多隔 N 个词 | `"MySQL InnoDB" @5` |
| `()` | 子表达式分组 | `+(apple banana) -pie` |

---

## 二、server 层：MATCH AGAINST 如何走到引擎

### 2.1 语法与模式标志

语法产生式在 `sql_yacc.yy:10535`：

```
MATCH ident_list_arg AGAINST '(' bit_expr fulltext_options ')'
  →  Item_func_match(@$, $2, $5, $6)
```

三种模式不是枚举，而是**位标志宏**（`include/ft_global.h:106-110`）：

```c
#define FT_NL 0         /* 自然语言 */
#define FT_BOOL 1       /* 布尔 */
#define FT_SORTED 2     /* 内部按 rank 排序 */
#define FT_EXPAND 4     /* 查询扩展 */
#define FT_NO_RANKING 8 /* 跳过打分 */
```

布尔模式独立成一支产生式（`sql_yacc.yy:11024`），所以**布尔模式不能叠加 QUERY EXPANSION**。

### 2.2 Item_func_match

定义在 `sql/item_func.h:3396`。核心是 `val_real()`（`item_func.cc:7760`）—— 它返回相关性分数，也是 `WHERE MATCH(...)` 判真假的依据（`Item_func_match_predicate::val_int()` = `val_real() != 0`，`item_cmpfunc.cc:7112`）。

**打分有两条路径**：

| 场景 | 路径 | 实现 |
|------|------|------|
| FT 索引扫描 | `get_relevance()` — 直接取当前结果节点的 rank | `innobase_fts_retrieve_ranking` `ha_innodb.cc:21571` |
| filesort / 随机访问 | `find_relevance()` — 按 doc_id 回查 | `innobase_fts_find_ranking` `ha_innodb.cc:21601` |

### 2.3 使用限制（这些是硬约束，写代码时必须知道）

| 限制 | 报错 | 位置 |
|------|------|------|
| AGAINST 必须是**常量**（不能是列、子查询） | `ER_WRONG_ARGUMENTS` "AGAINST" | `item_func.cc:7556` |
| MATCH 参数必须是**列**，不能是表达式/外引用 | `ER_WRONG_ARGUMENTS` "MATCH" | `item_func.cc:7567` |
| 所有列必须来自**同一张表** | 同上 | `item_func.cc:7590` |
| **MATCH 的列集合必须与某个 FULLTEXT 索引完全一致**（不能多也不能少） | `ER_FT_MATCHING_KEY_NOT_FOUND` | `item_func.cc:7739`（判定 `:7680-7731`） |
| 不能用于生成列 / 函数索引 | `check_function_as_value_generator` | `item_func.h:3472` |
| 不能在物化子查询上执行 | `ER_NO_FT_MATERIALIZED_SUBQUERY` | `item_func.cc:7494` |

> **8.0 允许 `ORDER BY MATCH(...)` 且可能免排序**（`sql_optimizer.cc:1733` `test_if_ft_index_order`），所以"FT 只能用 WHERE"是过时的说法。

### 2.4 优化器如何选上 FT 索引

- `add_ft_keys()`（`sql_optimizer.cc:7822`）把可下推的 FT 谓词变成 `Key_use`，键部用一个特殊值 **`FT_KEYPART`**（`= MAX_REF_PARTS + 10`，`sql_select.h:105`）
- 代价估算在 `find_best_ref`（`sql_planner.cc:207`），FT 分支在 `:669-685`：`access_type = fulltext`，`cur_fanout = 1.0`
- 计划类型 **`JT_FT`**（`sql_opt_exec_shared.h:221`）；`FT_SELECT` 是 5.x 遗留名，本版本不存在
- 执行迭代器 **`FullTextSearchIterator`**（`ref_row_iterators.cc:593`），`Init()` 调 `ft_init()`，`Read()` 调 `ha_ft_read()`
- 可下推的谓词形态共 5 种（单 MATCH、MATCH>c、MATCH>=c、c<MATCH、c<=MATCH），见 `IsSargableFullTextIndexPredicate` `join_optimizer.cc:5117`

### 2.5 handler 接口

```c
// include/ft_global.h:47-70
struct _ft_vft {
  int (*read_next)(FT_INFO *, char *);
  float (*find_relevance)(FT_INFO *, uchar *, uint);
  void (*close_search)(FT_INFO *);
  float (*get_relevance)(FT_INFO *);
  void (*reinit_search)(FT_INFO *);
};
struct _ft_vft_ext {
  uint (*get_version)();
  ulonglong (*get_flags)();
  ulonglong (*get_docid)(FT_INFO_EXT *);
  ulonglong (*count_matches)(FT_INFO_EXT *);
};
#define FTS_DOC_ID_COL_NAME "FTS_DOC_ID"
#define FTS_NGRAM_PARSER_NAME "ngram"
```

InnoDB 只实现了 5 槽中的 3 个（`ha_innodb.cc:605`）：`read_next` 和 `reinit_search` 为 `nullptr`。

| 方法 | 位置 | 说明 |
|------|------|------|
| `ft_init()` | `ha_innodb.cc:10894` | `rnd_init(false)` |
| `ft_init_ext()` | `ha_innodb.cc:10914` | 构造 `NEW_FT_INFO`，调 `fts_query()`（`:11016`） |
| `ft_init_ext_with_hints()` | `ha_innodb.cc:11042` | 处理 `FT_NO_RANKING` → limit 下推 |
| `ft_read()` | `ha_innodb.cc:11098` | 遍历 `rankings_by_rank`，按 doc_id 回表 |

> `ft_end()`（`ha_innodb.cc:11234`）和 `ft_update()` 是遗留/不存在的接口：实际资源释放走 `_ft_vft::close_search`，DML 维护走 `fts_trx_add_op`（不走 handler 虚函数）。

---

## 三、FTS 索引的物理结构：11 张辅助表

### 3.1 总览

一个 FTS 索引对应 **6 张倒排分片表 + 5 张公共表**：

| 表 | 后缀常量 | 用途 |
|----|----------|------|
| `fts_<tid>_<iid>_index_1..6` | `fts_index_selector[]` | **倒排索引**：word → ilist |
| `fts_<tid>_deleted` | `FTS_SUFFIX_DELETED` | 已删除、尚未从 INDEX 表清除的 doc_id |
| `fts_<tid>_deleted_cache` | `FTS_SUFFIX_DELETED_CACHE` | 删除的内存缓存落盘中转 |
| `fts_<tid>_being_deleted` | `FTS_SUFFIX_BEING_DELETED` | 本轮 OPTIMIZE 正在处理的 doc_id |
| `fts_<tid>_being_deleted_cache` | `FTS_SUFFIX_BEING_DELETED_CACHE` | 同上，cache 版本 |
| `fts_<tid>_config` | `FTS_SUFFIX_CONFIG` | KV 配置（`synced_doc_id`、`optimize_checkpoint_limit` 等） |

常量定义：`fts0fts.cc:127-158`；数量 `FTS_NUM_AUX_INDEX = 6`、`FTS_NUM_AUX_COMMON = 5`（`fts0fts.h:106/109`）。

表名形如 `db/fts_0000000000000abc_0000000000000def_index_1`（拼接见 `fts0sql.cc:60-189`）。

```
用户表 articles (InnoDB, 有 FULLTEXT(title, body))
   │
   ├─ fts_<tid>_<iid>_index_1   ─┐
   ├─ fts_<tid>_<iid>_index_2    │  6 张倒排分片表
   ├─ ...                        │  (word, first_doc_id, last_doc_id, doc_count, ilist)
   └─ fts_<tid>_<iid>_index_6   ─┘
   ├─ fts_<tid>_deleted           墓碑：已删 doc_id
   ├─ fts_<tid>_deleted_cache
   ├─ fts_<tid>_being_deleted     OPTIMIZE 工作中的墓碑快照
   ├─ fts_<tid>_being_deleted_cache
   └─ fts_<tid>_config            KV：synced_doc_id / optimize_checkpoint_limit / ...
```

### 3.2 为什么要 6 张分片表

**不是为了每个索引一张，而是每个索引 6 张**，目的是缩小单棵 B-tree 并让建索引时能并行排序。

选择规则 `fts_select_index()`（`fts0types.ic:178`）：

```c
static inline ulint fts_select_index(const CHARSET_INFO *cs, const byte *str, ulint len) {
  if (fts_is_charset_cjk(cs)) {
    selected = fts_select_index_by_hash(cs, str, len);   // CJK：首字符 hash % 6
  } else {
    selected = fts_select_index_by_range(cs, str, len);  // 其他：按排序权重区间
  }
```

- 非 CJK：按首字符排序权重 `strnxfrm` 落在哪个区间 —— 边界 `{9, 65, 70, 75, 80, 85}`（`fts0fts.cc:151`），即 <9 → index_1，9..64 → index_2，… 65~69 是 'A'-'E' 之类
- CJK（gbk/big5/ujis/sjis/euckr 等，见 `fts0types.ic:99-113`）：首字符分布集中，改用 `hash % 6` 避免倾斜

### 3.3 INDEX 表的结构

等价 SQL（源码注释 `fts0fts.h:528-535`）：

```sql
CREATE TABLE FTS_<prefix>_INDEX_[1-6](
  word          VARCHAR(FTS_MAX_WORD_LEN),
  first_doc_id  INT NOT NULL,
  last_doc_id   UNSIGNED NOT NULL,
  doc_count     UNSIGNED INT NOT NULL,
  ilist         VARBINARY NOT NULL,
  UNIQUE CLUSTERED INDEX ON (word, first_doc_id))
```

实际由 `fts_create_one_index_table()`（`fts0fts.cc:1978`）用 InnoDB 内部接口建（不是 SQL DDL）。

**注意同一 word 可能有多行**：一行（一个 node）的 ilist 上限是 `FTS_ILIST_MAX_SIZE = 64KB`（`fts0priv.h:74`），超了就新起一行，靠 `first_doc_id` 区分。

### 3.4 公共表的结构

```sql
CREATE TABLE FTS_<prefix>_DELETED              (doc_id BIGINT UNSIGNED, UNIQUE CLUSTERED INDEX ON doc_id)
CREATE TABLE FTS_<prefix>_DELETED_CACHE        (同上)
CREATE TABLE FTS_<prefix>_BEING_DELETED        (同上)
CREATE TABLE FTS_<prefix>_BEING_DELETED_CACHE  (同上)
CREATE TABLE FTS_<prefix>_CONFIG               (key CHAR(50), value CHAR(200), UNIQUE CLUSTERED INDEX ON key)
```

CONFIG 默认键值（`fts0fts.cc:161-176`）：`cache_size_in_mb=256`、`optimize_checkpoint_limit=180`（秒）、`synced_doc_id=0`、`total_deleted_count=0`、`table_state=0`。

---

## 四、FTS_DOC_ID：文档的身份

倒排索引里标识文档的就是 `doc_id`（`uint64_t`）。

**8.0 不需要手工建**：`ha_innodb.cc:14906-14923` 会自动添加**隐藏**列和**隐藏**唯一索引：

```cpp
  /* Add hidden FTS_DOC_ID column */
  col->set_hidden(dd::Column::enum_hidden_type::HT_HIDDEN_SE);
  col->set_name(FTS_DOC_ID_COL_NAME);
  col->set_type(dd::enum_column_types::LONGLONG);
  col->set_nullable(false);
  col->set_unsigned(true);
  ...
  /* Add hidden FTS_DOC_ID_INDEX */
  dd_set_hidden_unique_index(dd_table->add_index(), FTS_DOC_ID_INDEX_NAME, fts_doc_id);
```

**若要显式建**，必须严格满足（`ha_innodb.cc:14865-14905`）：

| 要求 | 否则报错 |
|------|----------|
| 列名恰好 `FTS_DOC_ID` | `ER_INNODB_FT_WRONG_DOCID_COLUMN` |
| 类型 `BIGINT UNSIGNED NOT NULL` | 同上 |
| 名为 `FTS_DOC_ID_INDEX` 的 UNIQUE 索引、只含该列、非降序 | `ER_INNODB_FT_WRONG_DOCID_INDEX` |

**doc_id 分配**（不是 AUTO_INCREMENT）：

```c
// fts0fts.cc:2781
  mutex_enter(&cache->doc_id_lock);
  *doc_id = ++cache->next_doc_id;      // 内存计数器
  mutex_exit(&cache->doc_id_lock);
```

`next_doc_id` 初值来自 CONFIG 表的 `synced_doc_id` 与表中实际最大 doc_id 的较大者（`fts_init_doc_id` `fts0fts.cc:4913`）。每次 sync 后把水位写回 CONFIG（`:2930`）。

> **doc_id 必须递增**，且跳跃不能超过 `FTS_DOC_ID_MAX_STEP = 65535`（`fts0fts.h:116`）。

---

## 五、倒排列表 ilist 与 VLC 编码

### 5.1 VLC 变长整数

`fts0vlc.ic`：每字节低 7 位存数据，**高位 0x80 表示"这是最后一个字节"**（与 JSONB 的 0x80 续行位方向相反！）：

```c
// fts0vlc.ic:64
static inline ulint fts_encode_int(ulint val, byte *buf) {
  ...
  /* High-bit on means "last byte in the encoded integer". */
  *buf |= 0x80;
```

最多 5 字节：`≤127`→1B，`≤16383`→2B，`≤2097151`→3B，`≤268435455`→4B，否则 5B。

### 5.2 ilist 布局

```
[VLC(doc_id - prev_doc_id)]  [VLC(pos1 - 0)] [VLC(pos2 - pos1)] ... [0x00]
 └── 每段一个文档 ─────────────────────────────────────────────────────┘
```

- doc_id 和 position 都是**增量（delta）编码**，第一个文档的 prev 是 0
- 位置列表以 `0x00` 结尾（解码时 `while (*ptr)` 靠"真实 VLC 首字节非 0"来识别）
- 编码：`fts_cache_node_add_positions()` `fts0fts.cc:1052`
- 解码：`fts_query_filter_doc_ids()` `fts0que.cc:2958`

**举例**：word = "mysql"，出现在 doc 5 的第 0、10 位，doc 9 的第 3 位 →

```
05 00 0A 00     (doc 5: Δdoc=5, pos 0→Δ0, pos 10→Δ10)
04 83 00        (doc 9: Δdoc=4, pos 3→Δ3)
```
（示意，实际每字节高位会置 0x80）

---

## 六、写入路径与 fts_cache

### 6.1 关键：FTS 不走二级索引插入路径

`row_ins` 的索引循环**显式跳过** `DICT_FTS`（`row0ins.cc:3567`）：

```cpp
  while (node->index != nullptr) {
    if (node->index->type != DICT_FTS) {      // ← FTS 索引跳过
      err = row_ins_index_entry_step(node, thr);
```

代替它的是**事务级记账**：INSERT 时只调 `fts_trx_add_op(trx, table, doc_id, FTS_INSERT, nullptr)`（`row0mysql.cc:1704`），**只记 doc_id，不分词、不写 cache**。

真正的工作在 **COMMIT 时**：`trx_commit_low` → `fts_commit(trx)`（`trx0trx.cc:2211`）→ `fts_commit_table` → `fts_add` → `fts_add_doc_by_id` → 分词 → `fts_cache_add_doc`。

### 6.2 完整写入栈

```
ha_innobase::write_row                      ha_innodb.cc:9004
 └─ row_insert_for_mysql                    row0mysql.cc:1736
     └─ row_ins_step（跳过 DICT_FTS）        row0ins.cc:3567
     └─ fts_trx_add_op(FTS_INSERT)          row0mysql.cc:1704   ← 只记账

[COMMIT] trx_commit_low                     trx0trx.cc:2200
 └─ fts_commit                              fts0fts.cc:3246
     └─ fts_commit_table（独立后台 trx）     fts0fts.cc:3193
         └─ fts_add                         fts0fts.cc:3028
             └─ fts_add_doc_by_id           fts0fts.cc:3568
                 ├─ fts_fetch_doc_from_rec（读聚簇记录）      :3375
                 │   └─ fts_tokenize_document（分词）        :4805
                 └─ fts_cache_add_doc（只写内存）            :1164
```

### 6.3 fts_cache_t：内存倒排表

`fts0types.h:141-208`，**每表一份，进程内全局共享**（不是每事务）：

```c
struct fts_cache_t {
  rw_lock_t lock;                 /* 保护整个 buffer */
  ib_mutex_t doc_id_lock;         /* 覆盖 Doc ID */
  ib_vector_t *deleted_doc_ids;   /* 已删除 doc id */
  ib_vector_t *indexes;           /* 每个 FTS 索引一个 fts_index_cache_t */
  ib_vector_t *get_docs;          /* 从表读文档所需信息 */
  ulint total_size;               /* 所有 node 的 ilist 总字节，过大就 SYNC */
  fts_sync_t *sync;               /* 落盘用的 sync 结构 */
  doc_id_t next_doc_id;
  doc_id_t synced_doc_id;         /* 已 sync 到 CONFIG 的 Doc ID */
  ulint deleted;                  /* 上次 optimize 以来删除的文档数 */
  ulint added;                    /* 上次 optimize 以来新增的文档数 */
  fts_stopword_t stopword_info;
};
```

`indexes[i]` 是一个 `fts_index_cache_t`，里面 `words` 是一棵**按 word 排序的红黑树**（`fts_index_cache_init` `fts0fts.cc:473`），value 是 `fts_tokenizer_word_t`（word 文本 + `fts_node_t` 数组）。

### 6.4 SYNC：什么时候刷盘

触发条件（`fts0fts.cc:3691`）：

```c
  if ((cache->total_size - cache->total_size_before_sync > fts_max_cache_size / 10 || fts_need_sync) &&
      !cache->sync->in_progress) {
    need_sync = true;
```

- 单表阈值：`innodb_ft_cache_size`（默认 8MB），超过 1/10 就请求
- 全局阈值：`innodb_ft_total_cache_size`（默认 640MB），后台线程每 5 秒检查（`fts_is_sync_needed` `fts0opt.cc:2700`）

SYNC 做的事（`fts_sync` `fts0fts.cc:4370`）：**中序遍历 words 红黑树**（天然有序）→ 对每个未 sync 的 node 调 `fts_write_node` 执行 `INSERT INTO fts_*_index_N`（`fts0fts.cc:3999`）→ 更新 CONFIG 的 `synced_doc_id` → 清空并重建 cache。

---

## 七、分词（tokenize）与 ngram

### 7.1 插件接口

分词器是 `st_mysql_ftparser` 插件（`include/mysql/plugin_ftparser.h`），回调协议：

```c
// fts0fts.cc:4778
  param.mysql_parse    = fts_tokenize_document_internal;   // InnoDB 提供的内置切词
  param.mysql_add_word = fts_tokenize_add_word_for_parser; // 收集词
  param.mode = MYSQL_FTPARSER_SIMPLE_MODE;                 // 文档分词
  parser->parse(&param);
```

| parser | 位置 | 说明 |
|--------|------|------|
| **内置（default）** | `fts0plugin.cc:58-67` | 直接回落到 InnoDB 的 `fts_tokenize_document_internal`，按 `true_word_char()` 切词（字母/数字/下划线） |
| **ngram** | `plugin/fulltext/ngram_parser/plugin_ngram.cc:199` | 按 N 个字符滑窗切词，`ngram_token_size` 默认 **2**（范围 1..10，只读变量） |
| **MeCab** | `plugin/fulltext/mecab_parser/` | 日文，需要外部词典 |

ngram 滑窗核心（`plugin_ngram.cc:87`）：凑够 `ngram_token_size` 个字符产出一个 token，然后前进 1 个字符。

### 7.2 停用词

- 默认 36 个（Google stopword 列表）：`fts_default_stopword[]` `fts0fts.cc:119-124`，含 `a/an/the/i/is/it/of/that/this/…`
- 加载：`fts_load_default_stopword`（`:294`）/ `fts_load_user_stopword`（`:390`）/ 总入口 `fts_load_stopword`（`:5968`）
- 用户表：`innodb_ft_server_stopword_table` / `innodb_ft_user_stopword_table`，会话级优先于全局
- 长度过滤：`innodb_ft_min_token_size`（默认 3）/ `innodb_ft_max_token_size`（默认 84）

> **注意**：server 层的 `ft_min_word_len`（默认 4）对 **InnoDB 无效**，要用 `innodb_ft_min_token_size`（默认 3）。这是最常见的踩坑点。

---

## 八、后台线程：sync 与 optimize

### 8.1 线程本体

`fts_optimize_thread`（`fts0opt.cc:2813`），由 `fts_optimize_init()`（`:2948`）在 `srv0start.cc:2529` 启动。

主循环（`fts0opt.cc:2831-2922`）：

```c
    if (!done && ib_wqueue_is_empty(wq) && n_tables > 0 && n_optimize > 0) {
      slot = ...;
      if (slot->state != FTS_STATE_EMPTY) {
        slot->state = FTS_STATE_RUNNING;
        fts_optimize_table_bk(slot);           // ① 后台 optimize
      }
    } else if (n_optimize == 0 || !ib_wqueue_is_empty(wq)) {
      msg = ib_wqueue_timedwait(wq, FTS_QUEUE_WAIT);   // ② 等消息，5 秒超时
      if (msg == nullptr) {
        if (fts_is_sync_needed(tables)) fts_need_sync = true;
        continue;
      }
      switch (msg->type) { ... case FTS_MSG_SYNC_TABLE: fts_optimize_sync_table(...); }
```

**两类工作**：

| 类型 | 触发 | 函数 |
|------|------|------|
| **sync**（刷 cache） | DML 阈值 / 全局内存超限 / 每 5s 检查 | `fts_optimize_sync_table` `fts0opt.cc:2796` → `fts_sync_table` `fts0fts.cc:4478` |
| **optimize**（回收删除） | `cache->deleted >= 10,000,000` 且距上次 ≥5 分钟 | `fts_optimize_table_bk` `fts0opt.cc:2290` → `fts_optimize_table` `:2332` |

**`OPTIMIZE TABLE` 不由后台线程执行** —— 它在用户线程同步跑（`ha_innodb.cc:18142-18143`），且只有在 `innodb_optimize_fulltext_only=ON` 时才做 FTS 优化，否则退化成 ALTER TABLE 重建。

唤醒靠 `ib_wqueue_t` 工作队列（不存在 `fts_optimize_wakeup` 这个函数名）。

---

## 九、删除与空间回收

### 9.1 DELETE 只写墓碑

```c
// fts0fts.cc:3112（fts_delete）
  graph = fts_parse_sql(&fts_table, info,
                        "BEGIN INSERT INTO $deleted VALUES (:doc_id);");
```

- 删除时**不碰 INDEX 表**，只把 doc_id 写进 `DELETED` 辅助表（由独立后台事务提交，持久）
- 查询时把 `DELETED` + `DELETED_CACHE` + `cache->deleted_doc_ids` 三处的 doc_id 合并过滤（`fts0que.cc:3684-3709`）
- UPDATE = delete + insert：两个 op 在 `fts_trx_table_add_op` 里被合并成 `FTS_MODIFY`（`fts0fts.cc:2589`），commit 时走 `fts_modify` = `fts_delete` + `fts_add`（`:3137`）

### 9.2 OPTIMIZE：四张墓碑表的轮转

```
   ┌─────────────┐   快照     ┌──────────────────┐
   │   DELETED   │ ────────▶ │  BEING_DELETED   │
   │DELETED_CACHE│           │BEING_DELETED_CACHE│
   └─────────────┘           └──────────────────┘
         ▲                            │
         │ 新删除继续写这里            │ ② 按这个列表重写 INDEX 表
         │                            ▼
         │                    过滤掉这些 doc_id
         │                            │
         └── ③ 全部完成后才清空 DELETED 和 BEING_* ──┘
```

三条内联 SQL（`fts0opt.cc:256-275`）：

```c
fts_init_delete_sql    = "INSERT INTO $being_deleted SELECT doc_id FROM $deleted;
                          INSERT INTO $being_deleted_cache SELECT doc_id FROM $deleted_cache;"
fts_delete_doc_ids_sql = "DELETE FROM $deleted WHERE doc_id = :doc_id1;
                          DELETE FROM $deleted_cache WHERE doc_id = :doc_id2;"
fts_end_delete_sql     = "DELETE FROM $being_deleted; DELETE FROM $being_deleted_cache;"
```

主流程 `fts_optimize_table`（`fts0opt.cc:2332`）：

| 步骤 | 函数 |
|------|------|
| 建上下文(trx / to_delete) | `fts_optimize_create` `:1483` |
| DELETED → BEING_DELETED 快照 | `fts_optimize_create_deleted_doc_id_snapshot` `:2030` |
| 读快照 | `fts_optimize_read_deleted_doc_id_snapshot` `:2139` |
| 逐索引重写 | `fts_optimize_indexes` `:2170` → `fts_optimize_index` `:1787` → `fts_optimize_word` `:1268` |
| 清理 | `fts_optimize_purge_snapshot` `:2232` |

**为什么这样设计**：快照阶段立即提交，之后的删除写 DELETED 不影响本轮；若中途崩溃，下次看到 `BEING_DELETED` 有残留就**续做**（`fts0opt.cc:2348`，`DB_DUPLICATE_KEY` 被当成功 `:2357`），不会漏删也不会重复。

**并行度**：`fts_optimize_index` 是**单线程**的（`fts0opt.cc:785` 对 6 张分片表是顺序 for 循环）。`innodb_ft_sort_pll_degree` 只作用于**建索引时**的并行分词/归并（`ddl0fts.cc:1539/1603`），不作用于 OPTIMIZE。

每批处理的词数 = `innodb_ft_num_word_optimize`（默认 2000），单批时间上限 = CONFIG 的 `optimize_checkpoint_limit`（默认 180 秒）。

---

## 十、查询路径与布尔模式 AST

### 10.1 主入口

`ha_innobase::ft_init_ext`（`ha_innodb.cc:10914`）→ **`fts_query()`（`fts0que.cc:3619`）**。

```
fts_query                                  fts0que.cc:3619
 ├─ fts_table_fetch_doc_ids（DELETED / DELETED_CACHE）   :3687, 3696
 ├─ fts_query_parse                                       :3545
 │   └─ fts_lexer_create(mode) → fts_parse() → ftsparse()（bison）
 ├─ fts_ast_visit(FTS_NONE, ast, fts_query_visitor, ...)  :3763
 │   └─ fts_query_visitor                                 :2728
 │       ├─ TERM   → fts_query_execute                    :2666
 │       │            └─ fts_query_union / intersect / difference
 │       │                ├─ fts_query_cache（查内存 cache）      :1233
 │       │                └─ fts_index_fetch_nodes（查 INDEX 表） fts0opt.cc:462
 │       │                    └─ fts_query_filter_doc_ids（解码 ilist） :2958
 │       └─ TEXT   → fts_query_phrase_search（短语 / @distance）  :2497
 ├─ fts_expand_query（仅 FTS_EXPAND，第二遍搜索）          :3773
 ├─ fts_query_calculate_idf                                :3778
 └─ fts_query_get_result                                   :3784
     └─ fts_query_calculate_ranking                        :3431
```

### 10.2 三种模式在 InnoDB 侧的差别

| 模式 | 差别 |
|------|------|
| NL vs BOOLEAN | **只换词法/语法分析器**：BOOLEAN 用 `fts0blex.l`（识别 `+ - ~ < > * @ () ""`），NL 用 `fts0tlex.l`（所有词当普通 term，等价全 union）。没有独立的算法分支 |
| FTS_EXPAND | 真正的功能分支：第一遍搜完，把命中文档重新分词、去掉已搜词，再 union 搜一遍（`fts_expand_query` `fts0que.cc:3909`） |
| FT_SORTED | **InnoDB 侧不使用**（只 SQL 层用）。InnoDB 无论哪种模式都会建 `rankings_by_rank` |
| FT_NO_RANKING | 允许 limit 下推（`ha_innodb.cc:11047`） |

### 10.3 布尔语法与 AST

语法在 `fts0pars.y`，bison 生成的入口是 `ftsparse()`（`fts0pars.cc:1152`，`#define yyparse ftsparse` 在 `:65`）。

节点类型（`fts0ast.h:41-52`）：`FTS_AST_OPER / NUMB / TERM / TEXT / PARSER_PHRASE_LIST / LIST / SUBEXP_LIST`。
操作符（`fts0ast.h:55-85`）：`FTS_NONE / IGNORE / EXIST / NEGATE / INCR_RATING / DECR_RATING / DISTANCE / *_SKIP`。

**最有意思的设计：三遍遍历**（`fts_ast_visit` `fts0ast.cc:520-649`）。源码注释给的语义：

```
'a +b -c d +e -f'
  → first  pass: union   {a, d}
  → exist  pass: intersect {+b, +e}
  → ignore pass: difference {-c, -f}
```

为什么要分遍？因为布尔语义要求"`+` 是交集、`-` 是差集"，如果一次性遍历就会把并集和交集混在一起。实现上：第一遍把带 `+`/`-` 的节点标成 `*_SKIP` 并跳过，之后再分别跑 exist pass 和 ignore pass。

集合运算映射（`fts_query_execute` `fts0que.cc:2666`）：

```c
  switch (query->oper) {
    case FTS_NONE: case FTS_NEGATE: case FTS_INCR_RATING: case FTS_DECR_RATING:
      fts_query_union(query, token);      break;
    case FTS_EXIST:  fts_query_intersect(query, token);  break;
    case FTS_IGNORE: fts_query_difference(query, token); break;
```

### 10.4 词如何定位

`fts_select_index()` 选中分片表 → 在 `FTS_INDEX_TABLE_IND`（聚簇唯一索引 `(word, first_doc_id)`）上用 InnoDB 内部 SQL 查询（`fts0opt.cc:497-514`）：

```sql
SELECT word, doc_count, first_doc_id, last_doc_id, ilist
FROM $table_name WHERE word LIKE :word ORDER BY first_doc_id;
```

- 精确词 → `LIKE 'xxx'` 退化为 key lookup
- 通配符 `data*` → `fts_query_get_token`（`fts0que.cc:2697`）拼成 `"data%"`，走**前缀范围扫描**
- 先查内存 cache（`fts_query_cache` `fts0que.cc:1233`），再查磁盘

### 10.5 结果如何取回行

`ha_innobase::ft_read()`（`ha_innodb.cc:11098`）：

1. 遍历 `result->rankings_by_rank`（已按 rank 排序的红黑树）
2. 用 `innobase_fts_create_doc_id_key`（`:11059`）构造 doc_id 搜索 tuple
3. 切到 `FTS_DOC_ID_INDEX`，`row_search_for_mysql` 回聚簇索引取整行

> 若 `read_just_key`（覆盖索引优化），则不回表，直接返回 doc_id。

---

## 十一、相关性排序（rank）算法

### 11.1 结论：不是 BM25，是 tf × idf²

全仓库没有 `bm25` / `k1` / `b` / 文档长度归一化。InnoDB 的公式是：

$$\operatorname{rank}(d,q) \;=\; \sum_{t \in q \cap d} \underbrace{\operatorname{tf}(t,d)}_{\text{原始词频，不饱和}} \;\times\; \underbrace{\operatorname{idf}(t)^2}_{\text{IDF 被乘两次}}$$

$$\operatorname{idf}(t) \;=\; \log_{10}\!\left(\frac{N}{\operatorname{df}(t)}\right)$$

其中：

- `N` = `dict_table_get_n_rows(index->table)` —— **InnoDB 的统计行数估计值**，不是精确值（`fts0que.cc:3678`）
- `df(t)` = INDEX 表的 `doc_count` 列
- **没有**文档长度归一化、没有余弦归一化、没有 `tf/(tf+k1)` 饱和

### 11.2 源码

IDF（`fts_query_calculate_idf` `fts0que.cc:3215`）：

```c
      if (total_docs == word_freq->doc_count) {
        /* QP assume ranking > 0 if we find a match. Since Log10(1) = 0,
        we cannot make IDF a zero value if do find a word in all documents.
        So let's make it an arbitrary very small number */
        word_freq->idf = log10(1.0001);
      } else {
        word_freq->idf = log10(total_docs / (double)word_freq->doc_count);
      }
```

打分（`fts_query_calculate_ranking` `fts0que.cc:3251`，**核心两行 3284-3286**）：

```c
    weight = (double)doc_freq->freq * word_freq->idf;         // tf * idf
    ranking->rank += (fts_rank_t)(weight * word_freq->idf);   // 再乘一次 idf ⇒ tf * idf²
```

底数是 **log10**（不是 ln）。`doc_freq->freq` 是**原始出现次数**（解码 ilist 时每解出一个 position 就 `++freq`，`fts0que.cc:3026`）。

### 11.3 布尔算子的调整与唯一的 clamp

`fts_query_change_ranking`（`fts0que.cc:733`）—— `~ / < / >` 只在这里生效：

```c
    ranking->rank += downgrade ? RANK_DOWNGRADE : RANK_UPGRADE;    // ∓1.0
    /* Allow at most 2 adjustment ... */
    if (ranking->rank >= 1.0F)      ranking->rank = 1.0F;
    else if (ranking->rank <= -1.0F) ranking->rank = -1.0F;
```

**唯一的归一化就是 `[-1.0, +1.0]` 这个 clamp**，且它只作用于布尔算子产生的预置分（发生在打分之前）。**最终 rank 不做任何归一化**，实际值可以远大于 1（`fts0fts.h:303` 注释里写的 "Rank is between 0..1" 是历史遗留）。

### 11.4 单 term 快路径

若整棵 AST 只有一个 TERM 且非 EXPAND，`fts_query_can_optimize`（`fts0que.cc:3592`）置 `FTS_OPT_RANKING`，跳过 doc_id 集合构造，直接用 `doc_freqs` 打分（`fts0que.cc:3367-3421`）：

```c
      ranking->rank = static_cast<fts_rank_t>(ranking->rank * word_freq->idf *
                                              word_freq->idf);   // 同样的 tf * idf²
```

---

## 十二、事务、崩溃恢复与 DDL

### 12.1 辅助表是普通 InnoDB 表，记 redo

`fts_create_one_common_table`（`fts0fts.cc:1802`）用 `row_create_table_for_mysql` 建，flags2 只加 `DICT_TF2_AUX`（不带 TEMPORARY）。所有 DML 走正常 B-tree 路径，有 redo/undo。

### 12.2 但辅助表的 DML 用**独立事务**，与用户事务不原子

```c
// fts0fts.cc:3200（fts_commit_table）
  trx_t *trx = trx_allocate_for_background();     // ← 独立后台事务
  ...
  fts_sql_commit(trx);                            // :3236 立即提交
```

调用点在 `trx_commit_low`（`trx0trx.cc:2211`），即在用户事务 commit **内部**另起一个后台 trx 并立即提交。源码注释（`:2213-2217`）明确承认这个设计"temporarily tolerate"：若 crash 发生在两者之间，可能重复插入 DELETED 导致 `DB_DUPLICATE_KEY`，目前被容忍。

**cache 不按事务记录来源**：一旦 commit 成功，token 已进入全局 cache，无法回滚 —— 这也是为什么 FTS 的记账要在 commit 时才做。

回滚场景：`fts_trx_add_op` 只记账不动 cache，未提交就自然丢弃；语句级回滚由 `fts_savepoint_rollback_last_stmt`（`fts0fts.cc:5679`，入口 `trx0roll.cc:315`）抵消。

### 12.3 崩溃恢复：靠 synced_doc_id 懒重建

`fts_cache` 是纯内存结构，崩溃后未 sync 的 token **确实丢失**，但**可以自动补回，不需要重建索引**：

- sync 时"索引数据"与"水位 `synced_doc_id`"在**同一事务**提交（`fts_sync_commit` `fts0fts.cc:4264`）
- 恢复时 `fts_init_index`（`fts0fts.cc:6218`）从 `FTS_DOC_ID_INDEX` 扫出 `doc_id > synced_doc_id` 的行，重新分词放回 cache（`fts_init_recover_doc` `:6120`）

```c
// fts0fts.cc:6247
  start_doc = cache->synced_doc_id;
  ...
  fts_doc_fetch_by_doc_id(nullptr, start_doc, get_doc->index_cache->index,
                          FTS_FETCH_DOC_BY_ID_LARGE, fts_init_recover_doc, get_doc);
```

关键点：

- 走的是 `FTS_DOC_ID_INDEX` 的**范围扫描，不是全表扫描**
- 是 **lazy** 的，触发点：`fts_init_doc_id`（`:4913`）、`fts_add_doc_by_id`（`:3587`）、`ha_innobase::ft_init_ext`（`ha_innodb.cc:11008`）
- 幂等：由 `fts_status |= ADDED_TABLE_SYNCED`（`:6281`）保护
- **启动时没有全量 FTS 修复流程** —— `srv0start.cc` 里只做 `fts_optimize_init()`（`:2529`）

### 12.4 DDL

| 操作 | 路径 |
|------|------|
| DROP TABLE | `row_drop_ancillary_fts_tables`（`row0mysql.cc:3668`）→ `fts_drop_tables`（`fts0fts.cc:1643`）= 5 张公共表 + 每个 FTS 索引 6 张分片表 |
| TRUNCATE | 8.0 是 drop + create，辅助表整体删除再重建（`ha_innodb.cc:14560`） |
| RENAME | `fts_rename_aux_tables`（`fts0fts.cc:1385`） |
| DROP INDEX | `fts_drop_index`（`fts0fts.cc:707`） |

### 12.5 已知限制

| 限制 | 位置 |
|------|------|
| **分区表不支持 FULLTEXT** | `handler0alter.cc:10110-10123`，`ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING` |
| **临时表不支持** | `ha_innodb.cc:13172`，`ER_INNODB_NO_FT_TEMP_TABLE` |
| 一次只能建一个 FTS 索引 | `handler0alter.cc:4479`，`ER_INNODB_FT_LIMIT` |
| 虚拟列 / 函数索引不支持 FULLTEXT | `ha_innodb.cc:11991`；`ER_FULLTEXT_FUNCTIONAL_INDEX` |
| `FTS_DOC_ID_INDEX` 不能降序 | `ha_innodb.cc:13187` |
| `ft_min_word_len` / `ft_boolean_syntax` 对 InnoDB 无效 | 用 `innodb_ft_min_token_size`；`fts0tokenize.h:43` 是静态常量 |

---

## 核心调用栈

**写入**

```
ha_innobase::write_row                     ha_innodb.cc:9004
 └─ row_insert_for_mysql                   row0mysql.cc:1736
     ├─ row_ins_step（跳过 DICT_FTS）       row0ins.cc:3567
     └─ fts_trx_add_op                     row0mysql.cc:1704 → fts0fts.cc:2637
[COMMIT]
 trx_commit_low                            trx0trx.cc:2200
 └─ fts_commit                             fts0fts.cc:3246
     └─ fts_commit_table（独立 trx）        fts0fts.cc:3193
         └─ fts_add                        fts0fts.cc:3028
             └─ fts_add_doc_by_id          fts0fts.cc:3568
                 ├─ fts_fetch_doc_from_rec fts0fts.cc:3375
                 │   └─ fts_tokenize_document      fts0fts.cc:4805
                 │       └─ parser->parse（内置 fts0plugin.cc:65 / ngram plugin_ngram.cc:199）
                 └─ fts_cache_add_doc      fts0fts.cc:1164
                     └─ (超阈值) fts_optimize_request_sync_table   fts0fts.cc:3722
[后台]
 fts_optimize_thread                       fts0opt.cc:2813
 └─ fts_optimize_sync_table                fts0opt.cc:2796
     └─ fts_sync_table → fts_sync          fts0fts.cc:4478 / 4370
         ├─ fts_sync_index                 fts0fts.cc:4204
         │   └─ fts_sync_write_words       fts0fts.cc:4071（中序遍单词树）
         │       └─ fts_write_node         fts0fts.cc:3954 → INSERT INTO index_N
         └─ fts_sync_commit                fts0fts.cc:4264（更新 synced_doc_id + 清 cache）
```

**查询**

```
JOIN::optimize → init_ftfuncs              sql_base.cc:10319
 └─ Item_func_match::init_search           item_func.cc:7433
     └─ ha_innobase::ft_init_ext           ha_innodb.cc:10914
         └─ fts_query                      fts0que.cc:3619
             ├─ fts_query_parse            fts0que.cc:3545（bison ftsparse）
             ├─ fts_ast_visit              fts0ast.cc:520（三遍遍历）
             │   └─ fts_query_visitor      fts0que.cc:2728
             │       └─ fts_query_execute  fts0que.cc:2666（union/intersect/difference）
             ├─ fts_query_calculate_idf    fts0que.cc:3215
             └─ fts_query_calculate_ranking fts0que.cc:3251
FullTextSearchIterator::Read               ref_row_iterators.cc:593
 └─ ha_innobase::ft_read                   ha_innodb.cc:11098
     └─ row_search_for_mysql（FTS_DOC_ID_INDEX 回表）  ha_innodb.cc:11177
打分：Item_func_match::val_real            item_func.cc:7760
       └─ innobase_fts_retrieve_ranking    ha_innodb.cc:21571
```

**OPTIMIZE**

```
ha_innobase::optimize（innodb_optimize_fulltext_only=ON）  ha_innodb.cc:18142
 ├─ fts_sync_table                         fts0fts.cc:4478
 └─ fts_optimize_table                     fts0opt.cc:2332
     ├─ fts_optimize_create_deleted_doc_id_snapshot   fts0opt.cc:2030
     ├─ fts_optimize_read_deleted_doc_id_snapshot     fts0opt.cc:2139
     ├─ fts_optimize_indexes → fts_optimize_index     fts0opt.cc:2170 / 1787
     │   └─ fts_optimize_word → fts_optimize_write_word  fts0opt.cc:1268 / 1343
     └─ fts_optimize_purge_snapshot                   fts0opt.cc:2232
```

---

## 相关的系统变量/状态变量

### 系统变量（默认值已逐一核实）

**InnoDB 侧**（`ha_innodb.cc`）：

| 变量 | 默认 | 作用域 | 说明 |
|------|------|--------|------|
| `innodb_ft_cache_size` | **8000000**（8MB） | GLOBAL，只读 | 单表 FTS cache 上限，超 1/10 触发 sync |
| `innodb_ft_total_cache_size` | **640000000**（640MB） | GLOBAL，只读 | 全局 cache 总上限 |
| `innodb_ft_min_token_size` | **3** | GLOBAL，只读 | 最小 token 长度（**对 InnoDB 生效的是这个，不是 ft_min_word_len**） |
| `innodb_ft_max_token_size` | **84** | GLOBAL，只读 | 最大 token 长度 |
| `innodb_ft_num_word_optimize` | **2000** | GLOBAL，可动态 | 每次 OPTIMIZE 处理的词数 |
| `innodb_ft_sort_pll_degree` | **2** | GLOBAL，只读 | **建索引时**并行分词/归并线程数（不影响 OPTIMIZE） |
| `innodb_ft_result_cache_limit` | **2000000000**（2GB） | GLOBAL，可动态 | 单查询中间结果缓存上限 |
| `innodb_ft_server_stopword_table` | NULL | GLOBAL，可动态 | 服务器级自定义停用词表 |
| `innodb_ft_user_stopword_table` | NULL | GLOBAL+SESSION | 会话级自定义停用词表（优先于全局） |
| `innodb_ft_enable_stopword` | **true** | GLOBAL+SESSION | 建索引时是否启用停用词 |
| `innodb_ft_aux_table` | NULL | GLOBAL，可动态 | 指定 I_S 表要看哪张基表（**必须是有 FTS 索引的表**） |
| `innodb_ft_enable_diag_print` | **false** | GLOBAL，可动态 | 额外 FTS 诊断输出 |
| `innodb_optimize_fulltext_only` | **false** | GLOBAL，可动态 | `OPTIMIZE TABLE` 只优化 FTS 而不重建表 |

**server 侧**（`sql/sys_vars.cc`）：

| 变量 | 默认 | 说明 |
|------|------|------|
| `ft_min_word_len` | **4** | **主要服务 MyISAM**，InnoDB 用 `innodb_ft_min_token_size` |
| `ft_max_word_len` | **84** | 同上 |
| `ft_query_expansion_limit` | **20** | 查询扩展用的最佳匹配数 |
| `ft_stopword_file` | NULL | 停用词文件 |
| `ft_boolean_syntax` | `"+ -><()~*:\"\"&|"` | **对 InnoDB 不生效**——InnoDB 用 `fts0tokenize.h:43` 的静态常量 |

**ngram 插件**：`ngram_token_size` 默认 **2**，范围 1..10，只读（`plugin_ngram.cc:250`）。

### 状态变量

**8.0.39 中没有任何 `Innodb_ft_*` 状态变量**（`export_var_t` 与 `innodb_status_variables[]` 中均无 FTS 字段）。

FTS 的可观测性靠 **INFORMATION_SCHEMA 表**（都必须先设 `innodb_ft_aux_table`）：

| I_S 表 | 内容 |
|--------|------|
| `INNODB_FT_INDEX_CACHE` | 内存 cache 中尚未 sync 的倒排条目 |
| `INNODB_FT_INDEX_TABLE` | 已落盘的倒排索引内容 |
| `INNODB_FT_DELETED` | 已删除但尚未清除的 doc_id |
| `INNODB_FT_BEING_DELETED` | 本轮 OPTIMIZE 正在处理的 doc_id |
| `INNODB_FT_CONFIG` | `synced_doc_id` / `cache_size_in_mb` / `optimize_checkpoint_limit` / `total_deleted_count` 等 |
| `INNODB_FT_DEFAULT_STOPWORD` | 默认停用词表 |

（读这些表需要 `PROCESS` 权限，`i_s.cc:2304`）

---

## Misc

### 术语表

| 术语 | 含义 |
|------|------|
| **doc_id** | 文档标识（uint64），存在隐藏列 `FTS_DOC_ID` 上 |
| **ilist** | 倒排列表：某个 word 对应的 (doc_id, positions) 序列，VLC + 增量编码 |
| **node** | ilist 的一个物理行（上限 64KB），一个 word 可有多个 node |
| **aux table（辅助表）** | 承载 FTS 索引的 11 张内部表 |
| **fts_cache** | 每表一份的内存倒排表，新增文档先进这里 |
| **sync** | 把 cache 批量刷进 INDEX 辅助表 |
| **optimize** | 按墓碑表重写 INDEX 表，真正删除已删文档的条目 |
| **token / word** | 分词后的词；`fts_tokenizer_word_t`（内存）vs `fts_word_t`（磁盘） |
| **stopword** | 停用词，不进索引 |
| **rank** | 相关性分数 = Σ tf × idf² |

### 易混淆概念对比

| 对比 | 说明 |
|------|------|
| **`ft_min_word_len` vs `innodb_ft_min_token_size`** | 前者（默认 4）对 **InnoDB 无效**；InnoDB 用后者（默认 3） |
| **`innodb_ft_sort_pll_degree` 的作用范围** | 只作用于**建索引时**的并行分词/归并；**OPTIMIZE 是单线程** |
| **`OPTIMIZE TABLE` 是否优化 FTS** | 只有 `innodb_optimize_fulltext_only=ON` 时才做，否则退化成 ALTER TABLE 重建 |
| **DELETE 后空间是否释放** | 不释放。只是写墓碑；真正清除要 OPTIMIZE。后台自动触发需 `deleted >= 1000万` 且间隔 ≥5 分钟 |
| **FTS 索引 vs 普通二级索引** | FTS 索引**不走** `row_ins_sec_index_entry`，而是事务级记账 + commit 时进 cache |
| **VLC 的 0x80** | InnoDB FTS 里 0x80 = "**最后一个字节**"；JSONB 的 0x80 = "**还有后续字节**"。方向相反 |
| **`_ft_vft` 的 5 个槽** | InnoDB 只实现了 3 个，`read_next` / `reinit_search` 是 nullptr |
| **rank 是否归一化** | 不归一化（除布尔算子的 [-1,1] clamp）。实际值可远大于 1 |

### FAQ

**Q：为什么我建了 FULLTEXT 索引，`MATCH` 却报 "Can't find FULLTEXT index matching the column list"？**
A：MATCH 的列集合必须与该 FULLTEXT 索引**完全一致**（不能是子集也不能是超集）。`fix_index`（`item_func.cc:7660`）同时检查 `max_cnt < arg_count` 和 `max_cnt < user_defined_key_parts`，两者都不满足才匹配。

**Q：中文全文检索怎么做？**
A：`CREATE FULLTEXT INDEX idx ON t(col) WITH PARSER ngram;`，并按需设 `ngram_token_size`（默认 2，只读变量，改了要重启）。ngram 按 N 字滑窗切词，不需要词典。注意 CJK 字符集下 6 张分片表的选择改用 hash%6（`fts0types.ic:183`）。

**Q：删了很多行，为什么索引文件不缩小？**
A：删除只写 DELETED 墓碑表，INDEX 表没动。需要 `SET GLOBAL innodb_optimize_fulltext_only=ON` 然后 `OPTIMIZE TABLE t`。后台线程自动回收的门槛是 1000 万个已删文档。

**Q：崩溃后 FTS 索引会丢吗？**
A：不会永久丢。辅助表是普通 InnoDB 表（有 redo）；内存 cache 丢的部分，靠 CONFIG 里的 `synced_doc_id` 在下次使用时按 `doc_id > synced_doc_id` 范围扫描重新分词补回（`fts_init_index` `fts0fts.cc:6218`）。

**Q：FTS 的打分和 Elasticsearch 一样吗？**
A：不一样。ES/Lucene 用 **BM25**（有 k1/b 参数、有文档长度归一化、tf 饱和）；InnoDB 用 **tf × idf²**（底数 log10、无归一化、tf 是原始词频）。所以同样的语料，两者的排序结果和分数绝对量都不同，`MATCH` 的分数只能用于**相对比较**，不能当绝对阈值用。

**Q：FTS 辅助表能直接 SELECT 吗？**
A：不能。它们是 `DICT_TF2_AUX` 标记的隐藏表。想看内容只能通过 I_S 的 `INNODB_FT_*` 表（且要先设 `innodb_ft_aux_table`）。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/sql_yacc.yy:10535` / `:11020` | MATCH AGAINST 语法 / fulltext_options |
| `include/ft_global.h:106-110` | `FT_NL/FT_BOOL/FT_SORTED/FT_EXPAND/FT_NO_RANKING` |
| `include/ft_global.h:47-81` | `_ft_vft` / `_ft_vft_ext` / `FTS_DOC_ID_COL_NAME` |
| `sql/item_func.h:3396` / `item_func.cc:7760` | `Item_func_match` / `val_real` |
| `sql/item_func.cc:7660` | `fix_index`（列集合必须完全匹配的判定） |
| `sql/sql_optimizer.cc:7822` | `add_ft_keys` |
| `sql/sql_planner.cc:669-685` | FT 分支的代价估算 |
| `sql/iterators/ref_row_iterators.cc:593` | `FullTextSearchIterator` |
| `ha_innodb.cc:10894/10914/11042/11098` | `ft_init` / `ft_init_ext` / `..._with_hints` / `ft_read` |
| `ha_innodb.cc:605-646` | `ft_vft_result` / `ft_vft_ext_result` 虚函数表 |
| `ha_innodb.cc:21571/21601` | `innobase_fts_retrieve_ranking` / `find_ranking` |
| `fts/fts0fts.cc:127-158` | 辅助表后缀常量 / `fts_index_selector` 分片边界 |
| `fts/fts0fts.cc:161-176` | CONFIG 默认值 |
| `fts/fts0fts.cc:119-124` | `fts_default_stopword`（36 词） |
| `fts/fts0fts.cc:1802/1978` | `fts_create_one_common_table` / `fts_create_one_index_table` |
| `fts/fts0fts.cc:2637/3028/3048/3137` | `fts_trx_add_op` / `fts_add` / `fts_delete` / `fts_modify` |
| `fts/fts0fts.cc:3568/3375/4805` | `fts_add_doc_by_id` / `fts_fetch_doc_from_rec` / `fts_tokenize_document` |
| `fts/fts0fts.cc:1164/1052` | `fts_cache_add_doc` / `fts_cache_node_add_positions`（ilist 编码） |
| `fts/fts0fts.cc:4370/4071/3954/4264` | `fts_sync` / `fts_sync_write_words` / `fts_write_node` / `fts_sync_commit` |
| `fts/fts0fts.cc:6218/6120` | **`fts_init_index`（崩溃恢复）** / `fts_init_recover_doc` |
| `fts/fts0fts.cc:2764/4913/2930` | doc_id 分配 / 初始化 / 持久化 |
| `fts/fts0opt.cc:2813/2948` | `fts_optimize_thread` / `fts_optimize_init` |
| `fts/fts0opt.cc:2332/2030/2170/2232` | `fts_optimize_table` / 快照 / 重写索引 / 清理 |
| `fts/fts0opt.cc:256-275` | 墓碑表轮转的三条 SQL |
| `fts/fts0opt.cc:462` | `fts_index_fetch_nodes` |
| `fts/fts0que.cc:3619` | **`fts_query` 主入口** |
| `fts/fts0que.cc:3215/3251` | **`fts_query_calculate_idf` / `fts_query_calculate_ranking`** |
| `fts/fts0que.cc:2666/2728` | `fts_query_execute` / `fts_query_visitor` |
| `fts/fts0que.cc:2958` | `fts_query_filter_doc_ids`（ilist 解码） |
| `fts/fts0que.cc:3909` | `fts_expand_query`（查询扩展） |
| `fts/fts0que.cc:2497/4157/1541` | 短语搜索 / 位置归并 / proximity 判定 |
| `fts/fts0ast.cc:520` | `fts_ast_visit`（三遍遍历） |
| `fts/fts0pars.y` / `fts0pars.cc:65,1152` | 布尔语法 / `ftsparse` |
| `fts/fts0blex.l` / `fts0tlex.l` | 布尔词法 / 自然语言词法 |
| `include/fts0vlc.ic:64/114` | `fts_encode_int` / `fts_decode_vlc` |
| `include/fts0types.ic:178` | `fts_select_index`（分片选择） |
| `include/fts0types.h:141-208` | `fts_cache_t` |
| `include/fts0fts.h:55-116` | `FTS_DOC_ID*` 常量 / `FTS_NUM_AUX_INDEX=6` / `FTS_ILIST_MAX_SIZE` |
| `include/dict0mem.h:104/272-298/2141` | `DICT_FTS` / `DICT_TF2_FTS*` / `fts_doc_id_index` |
| `plugin/fulltext/ngram_parser/plugin_ngram.cc:199,250` | ngram parse / `ngram_token_size` 定义 |
| `ddl/ddl0fts.cc:1106/1539/1603` | 建索引时写 node / 并行分词 / 并行归并 |
| `handler/i_s.cc:2224-3383` | `INNODB_FT_*` 系列 I_S 表 |
