# 全文检索（FTS）深度解析

> 基于 MySQL 8.0.39 源码。涵盖 MATCH AGAINST 的 SQL 层实现、FTS 辅助表体系、倒排索引编码、分词、fts_cache 与后台 sync/optimize、布尔模式 AST、相关性排序算法、事务与崩溃恢复。
>
> **边界**：FTS 是同时改动 SQL 层（语法、`Item_func_match`、优化器、迭代器、filesort）与引擎层（辅助表、内存 cache、后台线程）的特性，因此单篇讲完整条链路。**引擎差异**：社区版只有 InnoDB 与 MyISAM 两家，MyISAM 只在篇内小节对比，不单独开篇。纯引擎内部机制（B-tree、redo、buffer pool）见 [`../innodb/`](../innodb/) 相应篇；server↔引擎的接口本身见 [`../server/handler.md`](../server/handler.md)；**与 `LIKE` 的选型对比**见 [`../server/query/15_like.md`](../server/query/15_like.md) §十。

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
- [十三、两层的结合点](#十三两层的结合点)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 先从一个真实的查询开始

```sql
CREATE TABLE articles (
  id    INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
  title VARCHAR(200),
  body  TEXT,
  FULLTEXT (title, body)
) ENGINE=InnoDB;

EXPLAIN SELECT id, MATCH(title, body) AGAINST('mysql 优化') AS score
FROM articles
WHERE MATCH(title, body) AGAINST('mysql 优化')
ORDER BY score DESC LIMIT 10;
```

```
+----+-------------+----------+----------+---------------+-------+------+----------+------------------+
| id | select_type | table    | type     | possible_keys | key   | rows | filtered | Extra            |
+----+-------------+----------+----------+---------------+-------+------+----------+------------------+
|  1 | SIMPLE      | articles | fulltext | title         | title |    1 |   100.00 | Ft_hints: sorted |
+----+-------------+----------+----------+---------------+-------+------+----------+------------------+
```

这一行 `type=fulltext` 背后发生的事，就是本篇要讲清的。先抛出三个反直觉的事实，它们分别对应后面的章节：

1. **FTS 索引不是"一种 B-tree 索引"**——它是**用 11 张普通 InnoDB 表拼出来的倒排索引**，连 range 优化器都显式跳过它。而且 `rows=1` 是假的：优化器对 FT 的 fanout 恒取 `1.0`，**没有任何行数估算模型**。
2. **`Ft_hints: sorted` 是 server 层下推给引擎的一组"意图"**（要不要排序、rank 阈值多少、能否截断到 LIMIT）。这是 FTS 与其它索引最大的区别：两层之间传递的不只是"键"，还有一套 **hints 协议**。
3. **删掉 100 万行，索引文件一个字节都不会小**——删除只写墓碑表，真正回收要靠 `OPTIMIZE` 重写倒排表；后台线程自动触发的门槛是 **1000 万个已删文档**。

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

### 设计思想与权衡

**① 最根本的取舍：用"一堆普通表"模拟倒排索引，而不是发明一种新索引。**

倒排索引不是行级的一列一值，塞不进 B-tree 的一行。摆在面前的有两条路：

| 方案 | 代价 |
|---|---|
| **A. 实现新的索引类型**（PostgreSQL GIN 的路子：自研页面格式、 posting list 页、新的 WAL 与恢复逻辑） | 要新写一套页面管理、崩溃恢复、并发控制 |
| **B. 用普通 InnoDB 表拼**（InnoDB 的选择：一个词一行，ilist 存 BLOB） | 写入放大、查询要读多张表、删除只能墓碑+重写 |

InnoDB 选了 B。**理由是 B 把所有"难"的部分外包给了已有的 B-tree 子系统**——事务、redo/undo、MVCC、崩溃恢复、buffer pool 全部免费获得。这不是偷懒，而是 5.6 时把 FTS 做进 InnoDB 的唯一现实路径（5.6 之前的常见做法是"挂一张 MyISAM 从表存文本"，事务和崩溃恢复全是坑）。

代价是**这篇文档后面几乎所有机制都是为它买单**：

- 一次词的插入 = 一次 B-tree 插入 + 一次 ilist 的读改写 ⇒ **必须攒批**（内存 `fts_cache` + 后台 sync）
- 删一个词要改整条 posting list ⇒ **只能墓碑 + 后台重写**（DELETED 表 + OPTIMIZE）
- 攒批 ⇒ 内存态会丢 ⇒ **靠 `synced_doc_id` 水位崩溃重放**
- 攒批用独立事务提交 ⇒ **与用户事务不原子**（源码 `trx0trx.cc` 的 `FTS-FIXME` 注释明确承认，只能"temporarily tolerate `DB_DUPLICATE_KEY`"）

**② 6 张分片表的代价：非 CJK 下 `index_1` 是个超大桶。**

分片是为了缩小单棵 B-tree 并支持建索引时并行归并，但非 CJK 走的是"首字符排序权重落在哪个区间"，边界是 `{65, 70, 75, 80, 85}` ⇒ **所有 sort weight < 65 的词（数字、符号、以及 latin1 小写字母区）全部落进 `index_1`**，只有 A–E/F–J/K–O/P–T/U–Z 才均分到后 5 张。更糟的是 CJK 判定 `fts_is_charset_cjk` 只认 **9 个具体 collation 名**（gbk/big5/ujis/sjis/cp932/eucjpms/euckr/gb18030/gb2312），**不含 utf8mb4**——所以 utf8mb4 的中文列如果不配 ngram parser，词会全部按 range 规则涌进 `index_1`，形成严重倾斜。这是分片设计最实际的 bad case。

**③ 打分选了 tf × idf² 而不是 BM25。**

没有 `k1`/`b` 参数、没有文档长度归一化、没有 tf 饱和（`fts_doc_stats_t::word_count` 虽然记了词数，查询路径**完全没用到**）。后果：**长文档天然占优**，且 rank 不是概率也不是 [0,1]（`fts0fts.h` 里 "Rank is between 0..1" 的注释早已不成立），只能用于**同一条查询内的相对比较**。

**④ server 层无法估算 FT 的行数，只能靠 hints 补救。**

FT 谓词走的是独立通道（`add_ft_keys` → `FT_KEYPART` 哨兵 → `JT_FT`），range 优化器显式跳过它。优化器对 FT 的 fanout **恒取 1.0**，成本 = 前序读取数 × 1 页读成本——基本没有估算模型。补偿手段是**把意图打包下推**：要不要排序（`FT_SORTED`）、rank 阈值（`FT_OP_GT/GE`）、能否截断到 LIMIT（`FT_NO_RANKING` + limit）。

**⑤ 打分在引擎、取值在 SQL 层 ⇒ filesort 必须退回 rowid 模式。**

`MATCH()` 的值不从 `record[0]` 读，而是"问"handler 要（`get_relevance` / `find_relevance`）。于是只要表被 MATCH 引用，filesort 就**强制禁用 addon fields**（`Addon_fields_status::fulltext_searched`）——因为 addon 模式排序后直接从 buffer 反序列化，handler 根本没被定位到该行，取到的 rank 是错的。这是"打分职责跨界"带来的直接代价。

**★ 设计意图的源码自述（三条原文）**：

**（A）为什么用辅助表 —— 建表 DDL 就写在注释里**（`fts0fts.h:495`，★ 最直白的一条）：

```cpp
/** Creates the common auxiliary tables needed for supporting an FTS index
on the given table. ...
The following tables are created.
CREATE TABLE $FTS_PREFIX_DELETED
        (doc_id BIGINT UNSIGNED, UNIQUE CLUSTERED INDEX on doc_id)
...
CREATE TABLE $FTS_PREFIX_CONFIG
        (key CHAR(50), value CHAR(200), UNIQUE CLUSTERED INDEX on key) */
```
```cpp
/** Creates the column specific ancillary tables needed for supporting an
FTS index on the given table. ...

All FTS AUX Index tables have the following schema.
CREATE TABLE $FTS_PREFIX_INDEX_[1-6](
        word            VARCHAR(FTS_MAX_WORD_LEN),
        first_doc_id    INT NOT NULL,
        last_doc_id     UNSIGNED NOT NULL,
        doc_count       UNSIGNED INT NOT NULL,
        ilist           VARBINARY NOT NULL,
        UNIQUE CLUSTERED INDEX ON (word, first_doc_id)) */
```
> ★★ 这就是"用普通表模拟倒排索引"的完整答案：倒排项被映射成**聚簇 B-tree 上的一行**，主键 `(word, first_doc_id)`，posting list 塞进 VARBINARY 的 `ilist`。**事务、redo、MVCC、buffer pool、崩溃恢复全部免费得到**。

**（B）Unicode 支持不足的自我批评**（`fts0fts.h:284`，★ 罕见的"吐槽式"注释）：

```cpp
// FIXME: Get rid of this if possible.
/** Since MySQL's character set support for Unicode is woefully inadequate
(it supports basic operations like isalpha etc. only for 8-bit characters),
we have to implement our own. We use UTF-16 without surrogate processing
as our in-memory format. This typedef is a single such character. */
typedef unsigned short ib_uc_t;
```
> ★ 因为 MySQL 的 charset API 对 Unicode 支持不足（`isalpha` 等只支持 8-bit），InnoDB FTS 不得不**自己实现一套 UTF-16（不做 surrogate 处理）**的分词字符处理。

**（C）辅助表的 DML 用独立事务**（`fts0sql.cc:301`，★ 与 12.2 呼应）：

```cpp
/** Commit a transaction.
 @return DB_SUCCESS or error code */
dberr_t fts_sql_commit(trx_t *trx) /*!< in: transaction */
{
  dberr_t error;

  error = trx_commit_for_mysql(trx);
  /* 【注】参数 trx 是 FTS 自己 trx_allocate_for_background() 出来的独立后台事务 */

  /* Commit should always succeed */
  ut_a(error == DB_SUCCESS);

  return (DB_SUCCESS);
}
```

**★ 三个关键阈值的源码落点**：

```cpp
/* storage/innobase/include/fts0fts.h:102 */
/** The number of FTS index partitions for a fulltext index. */
constexpr size_t FTS_NUM_AUX_INDEX = 6;      /* ★ constexpr 不是 #define */

/** The number of FTS AUX common table for a fulltext index. */
constexpr size_t FTS_NUM_AUX_COMMON = 5;     /* ★ 常被误当作 6 */

/** Threshold where our optimize thread automatically kicks in */
#define FTS_OPTIMIZE_THRESHOLD 10000000      /* ★ 1000 万个已删文档才自动 OPTIMIZE */

/** Threshold to avoid exhausting of doc ids. Consecutive doc id difference
should not exceed FTS_DOC_ID_MAX_STEP */
#define FTS_DOC_ID_MAX_STEP 65535            /* ★ 由 VLC 的 delta 上限反推 */
```
> ★ `FTS_DOC_ID_MAX_STEP = 65535` 的来历（`row0mysql.cc:1676` 注释明说）：*"Difference between Doc IDs are restricted within 4 bytes integer. See fts_get_encoded_len()"* —— 业务约束是从编码格式反推出来的。

**★ 为什么要"耗尽 doc_id"保护**（`row0mysql.cc:1661`）：

```cpp
      if (doc_id < next_doc_id) {
        ib::error(ER_IB_MSG_981)
            << "FTS Doc ID must be large than " << next_doc_id - 1
            << " for table " << table->name;
        err = DB_FTS_INVALID_DOCID;
        ...
      }

      /* Difference between Doc IDs are restricted within
      4 bytes integer. See fts_get_encoded_len(). Consecutive
      doc_ids difference should not exceed
      FTS_DOC_ID_MAX_STEP value. */

      if (doc_id - next_doc_id >= FTS_DOC_ID_MAX_STEP) {
        ib::error(ER_IB_MSG_982) << "Doc ID " << doc_id
                                 << " is too big. Its difference with"
                                    " largest used Doc ID "
                                 << next_doc_id - 1 << " cannot"
                                    " exceed or equal to "
                                 << FTS_DOC_ID_MAX_STEP;
        err = DB_FTS_INVALID_DOCID;
```

### 理论溯源

| 理论 / 论文 | 源码落点 |
|---|---|
| **倒排索引 + TF-IDF**（Salton 向量空间模型，1970s） | `fts_*_index_1..6` 存 `word → ilist`；`fts_query_calculate_idf` 用 `log10(N/df)` |
| **BM25 的反面教材**：InnoDB 用的是简化 TF-IDF，缺 Robertson & Zaragoza（2009）的长度归一化与 tf 饱和 | `fts_query_calculate_ranking` 两行（`tf*idf` 再 `*idf`） |
| **变长整数（VLC）** 与 protobuf varint 同族，但**续位方向相反** | `fts_encode_int` / `fts_decode_vlc` |
| **增量（delta）编码**：doc_id 有序 ⇒ 差值小 ⇒ VLC 更省 | ilist 里 doc_id 与 position 都存差值 |
| **布尔检索模型**（Salton）：`+` 是交集、`-` 是差集、无符号是并集 | `fts_ast_visit` 的三遍遍历 |
| **相关反馈（Rocchio, 1971）的简化版**：用首轮命中文档的词扩展查询 | `fts_expand_query`（查询扩展） |
| **策略模式 / 插件**：分词器可替换 | `st_mysql_ftparser`；内置 / ngram / MeCab |
| **InnoDB 内部 query graph**（源自早期内置 SQL 解析器 `pars0pars.y`） | `fts_parse_sql` / `fts_eval_sql`——不是真 SQL，不过 server 优化器、不做 binlog |

### 算法与数据结构

按"这条链路的每一步用了什么"组织，全部经源码核实。**没实现的**另见 Misc 的清单。

**（1）分词**

| 算法 | 落点 | 关键点 |
|---|---|---|
| `true_word_char` 最长连续匹配 | `fts0tokenize.h:40`、`ha_innodb.cc:7839` | 词字符 = `_MY_U \| _MY_L \| _MY_NMR` 或 `_`；两阶段扫描（跳非词字符 → 吃连续词字符）。**汉字不属于这三类**，所以内置分词器切不出中文词 |
| ngram 重叠滑窗 | `plugin_ngram.cc:87-97` | 窗口 `ngram_token_size`（默认 2）、**步长 1 字符**；遇非词字符重置窗口 |
| ngram 停用词的子串枚举 | `fts0fts.cc:4532-4580` | 红黑树只支持 `COMPARE` 不支持 `CONTAIN`，于是对 unigram…n-gram 逐档再滑一次窗 ⇒ **O(n²)** |
| MeCab 形态素解析 | `plugin/fulltext/mecab_parser/` | 日文，外部词典依赖 |

**（2）索引组织**

| 算法 | 落点 | 关键点 |
|---|---|---|
| 倒排索引 | `FTS_*_index_1..6` | `word → (doc_id, positions...)` |
| 分片路由（区间） | `fts_select_index_by_range` `fts0types.ic:120` | 非 CJK：首字符 sort weight 在 `fts_index_selector[]`（`fts0fts.cc:151`）上线性查找；分界 9/65/70/75/80/85 |
| 分片路由（哈希） | `fts_select_index_by_hash` `fts0types.ic:146` | CJK：首字符走 collation 的 `hash_sort`（种子 `nr1=1,nr2=4`），**强制截断成 32 位 ulong** 再 `% 6`（注释明说 Windows/Linux 结果不同，但必须与磁盘数据一致） |

**★ 倒排索引的三个核心结构**（`storage/innobase/include/fts0types.h`，⚠️ 真名核查：三者都存在，但都在 `fts0types.h` 不在 `fts0fts.h`）：

```cpp
/** Columns of the FTS auxiliary INDEX table */
struct fts_node_t {           /* fts0types.h:210 —— INDEX 表的一行 */
  doc_id_t first_doc_id; /*!< First document id in ilist. */
  doc_id_t last_doc_id;  /*!< Last document id in ilist. */
  byte *ilist;           /*!< Binary list of documents & word
                         positions the token appears in. */
  ulint doc_count;       /*!< Number of doc ids in ilist */
  ulint ilist_size;      /*!< Used size of ilist in bytes. */
  ulint ilist_size_alloc;/*!< Allocated size of ilist in bytes */
  bool synced;           /*!< flag whether the node is synced */
};
                    /* ★ 一个 word 因 ilist 太大被切成多个 node，每个 node 覆盖
                       一段 doc_id 区间，主键是 (word, first_doc_id) */

/** A tokenizer word. Contains information about one word. */
struct fts_tokenizer_word_t {   /* fts0types.h:243 */
  fts_string_t text;      /*!< Token text. */
  ib_vector_t *nodes;     /*!< Word node ilists, each element is fts_node_t */
};

/** Word text plus it's array of nodes as on disk in FTS index */
struct fts_word_t {             /* fts0types.h:298 */
  fts_string_t text;  /*!< Word value in UTF-8 */
  ib_vector_t *nodes; /*!< Nodes read from disk */
  ib_alloc_t *heap_alloc; /*!< For handling all allocations */
};
```

**★ 索引期（内存）结构：token 与文档**（`fts0types.h`）：

```cpp
/** A token and its positions within a document. */
struct fts_token_t {
  fts_string_t text;      /*!< token text */
  ib_vector_t *positions; /*!< 该 token 在文档中出现的所有位置 */
};

/** This type represents a single document field. ... */
struct fts_doc_t {
  fts_string_t text;    /*!< document text */
  bool found;           /*!< true if the document was found successfully */
  ib_rbt_t *tokens;     /*!< 分词结果；键 fts_string_t*，值 fts_token_t* */
  ib_alloc_t *self_heap;
  CHARSET_INFO *charset;
  st_mysql_ftparser *parser;  /*!< fts plugin parser */
  bool is_ngram;              /*!< Whether it is a ngram parser */
  ib_rbt_t *stopwords;        /*!< Stopwords */
};
```

**★★ 6 张分片表的路由算法**（`fts0types.ic:95`，★ 这是"为什么是 6"的唯一直观证据 —— 源码里**没有**写"为什么是 6"的设计说明）：

```cpp
/** Check if fts index charset is cjk */
static inline bool fts_is_charset_cjk(const CHARSET_INFO *cs) {
  if (strcmp(cs->m_coll_name, "gb2312_chinese_ci") == 0 ||
      strcmp(cs->m_coll_name, "gbk_chinese_ci") == 0 ||
      strcmp(cs->m_coll_name, "big5_chinese_ci") == 0 ||
      strcmp(cs->m_coll_name, "gb18030_chinese_ci") == 0 ||
      strcmp(cs->m_coll_name, "ujis_japanese_ci") == 0 ||
      strcmp(cs->m_coll_name, "sjis_japanese_ci") == 0 ||
      strcmp(cs->m_coll_name, "cp932_japanese_ci") == 0 ||
      strcmp(cs->m_coll_name, "eucjpms_japanese_ci") == 0 ||
      strcmp(cs->m_coll_name, "euckr_korean_ci") == 0) {
    return (true);
  } else {
    return (false);
  }
}

/** Select the FTS auxiliary index for the given character by range. */
static inline ulint fts_select_index_by_range(const CHARSET_INFO *cs,
                                              const byte *str, ulint len) {
  ulint selected = 0;
  ulint value = innobase_strnxfrm(cs, str, len);   /* ★ 首字符的排序权重 */

  while (fts_index_selector[selected].value != 0) {
    if (fts_index_selector[selected].value == value) {
      return (selected);
    } else if (fts_index_selector[selected].value > value) {
      return (selected > 0 ? selected - 1 : 0);    /* ★ 落在两边界间 → 取左边界那张 */
    }
    ++selected;
  }
  return (selected - 1);                           /* ★ 大于 'U'(85) 全落 index_6 */
}

/** Select the FTS auxiliary index for the given character by hash. */
static inline ulint fts_select_index_by_hash(const CHARSET_INFO *cs,
                                             const byte *str, ulint len) {
  ...
  /*
    Get collation hash code. Force truncation to ulong for legacy reasons;
    it gives different results for Windows and Linux, but it needs to match
    on-disk data.
   */
  uint64_t nr1 = 1;
  uint64_t nr2 = 4;
  cs->coll->hash_sort(cs, str, char_len, &nr1, &nr2);

  return (static_cast<ulong>(nr1) % FTS_NUM_AUX_INDEX);  /* ★ CJK: hash % 6 */
}

/** Select the FTS auxiliary index for the given character. */
static inline ulint fts_select_index(const CHARSET_INFO *cs, const byte *str,
                                     ulint len) {
  ulint selected;

  if (fts_is_charset_cjk(cs)) {
    selected = fts_select_index_by_hash(cs, str, len);
  } else {
    selected = fts_select_index_by_range(cs, str, len);
  }

  return (selected);
}
```
> ★★ 注意最后那段注释：CJK 的哈希值**强制截断成 32 位 ulong**，且 Windows/Linux 结果不同 —— 但必须与磁盘数据一致，所以不敢改。这是典型的历史包袱。

**★ 分片边界常量**（`fts0fts.cc:151`）：

```cpp
/** FTS auxiliary INDEX split intervals. */
const fts_index_selector_t fts_index_selector[] = {
    {9, "index_1"},  {65, "index_2"}, {70, "index_3"}, {75, "index_4"},
    {80, "index_5"}, {85, "index_6"}, {0, nullptr}};
```
> ★ `9`=控制字符、`65`='A'、`70`='F'、`75`='K'、`80`='P'、`85`='U'。非 CJK 按首字符落区间，CJK 走 hash % 6 —— **"6"这个数字同时服务两种算法**。

**（3）倒排压缩（真正落盘的压缩，只有这一套）**

| 算法 | 落点 | 关键点 |
|---|---|---|
| Delta / d-gap | `fts_cache_node_add_positions` `fts0fts.cc:1076-1135` | doc_id 与 position 都存**差值**；位置列表以 `0x00` 结尾；新文档开始时位置 delta 重置为 0 |
| VLC 变长整数 | `fts0vlc.ic:42/64/114` | 每字节 7 有效位、**高位=1 表示最后一个字节**（与 protobuf 相反）；5 字节上限 |
| ilist 惰性扩容 | `fts0fts.cc:1092-1110` | 16 → 32 → 48 阶梯，之后 ×1.2 |
| node 切分 | `FTS_ILIST_MAX_SIZE = 64KB` | 超过就新起一行（靠 `first_doc_id` 区分） |

> **★ 磁盘上没有 zlib**。zlib 只出现在 `fts0opt.cc`，是 **OPTIMIZE 流程里一个纯内存的 word 列表缓冲**（`fts_zip_read_word` `:618`、`deflateInit(level=9)` `:824`，1024B block 链表，`FTS_ZIP_BLOCK_SIZE` `fts0fts.cc`… 见 `fts0opt.cc:247`），用完即释放，**不落盘、无开关**。把"InnoDB FTS 用 zlib 压缩索引"写进文档是错的。

**（4）查询处理**

| 算法 | 落点 | 复杂度 / 关键点 |
|---|---|---|
| 集合并/交/差 | `fts_query_union` `:1278`、`intersect` `:1105`、`difference` `:1014` | 底层是**红黑树 `ib_rbt_t`**，O(log n)。交集是**另建一棵树再整棵替换**，不是"小集合 probe 大树" |
| 命中词 bitmap | `fts_ranking_words_create/add/get_next` `:509/538/596` | 用朴素 char bitmap 替代"每文档一棵词红黑树"；初始 4B、**2 倍扩容**；配 `word_vector` 做 O(1) 反查 |
| 已删 doc 过滤 | `fts_bsearch` `fts0opt.cc:978` | 二分；7 处调用，全在"deleted 有序数组"场景 |
| 前缀/通配符 | `fts_cache_find_wildcard` `fts0que.cc:930` | 红黑树**前缀比较器定位 + 双向线性扫**（先 `rbt_prev` 到头，再 `rbt_next` 到尾）。**没有 trie** |
| 位置归并（proximity） | `fts_proximity_get_positions` `fts0que.cc:4157` | 源码注释自称 "similar to the merge phase of a merge sort"——**多指针推进，只推进当前位置最小的那个词**；不是滑动窗口也不是堆。`MAX_PROXIMITY_ITEM = 128` |
| 短语候选 doc 求交 | `fts_phrase_or_proximity_search` `:4031` | 嵌套扫描，**每个 token 都从 0 重启游标**（源码自带 FIXME `:4104` 承认这点） |
| 短语精确校验 | `fts_query_match_phrase` `:1691`、`fts_query_match_phrase_terms` `:1461` | 回读原文**朴素逐 token 顺序匹配**；无 KMP / Boyer-Moore |
| `@N` 距离判定 | `fts_proximity_is_word_in_range` `:1541` | UTF-8 变长 ⇒ "字节差 ≤ N" ≠ "隔 N 个词"，只能回读原文**在字节区间内数词**；`n_word` 含首尾被匹配词本身（`@2` 才是相邻） |
| 结果排序 | `fts_query_sort_result_on_rank` `:3850` | 遍历 `rankings_by_id` 逐节点 **rbt_insert 进新树**（按 rank 有序），O(n log n)，不是 qsort |

**（5）打分**：`rank = Σ tf × idf²`，见第十一章（无 BM25、无长度归一化）。

**（6）建索引：并行分词 + 外部归并**

| 算法 | 落点 | 关键点 |
|---|---|---|
| 并行分词（按 doc_id 取模分片） | `FTS::enqueue` `ddl0fts.cc:1575`、`start_parse_threads` `:1539` | `doc_id % n_parsers` 保证同一 doc 的多字段进同一线程；每线程 6 个临时文件（每 aux index 一个），路径 `thd_innodb_tmpdir` |
| 内存排序 | `merge_sort` `ddl0buffer.cc:44` | 自顶向下归并排序，需 O(n) aux 数组；排序键 = `(word, doc_id, position)` 三元组 |
| 外部归并 | `Merge_file_sort` `ddl0impl-merge.h:43` | **`N_WAY_MERGE = 2`**——注释原话 "we stick with 2 for now"，即**平衡 2 路外部归并**，不是 k 路、不是败者树（`ddl0merge.cc` 里 `priority_queue`/`make_heap` 0 命中） |
| 流式聚合重建 ilist | `FTS::Inserter::insert_tuple` `ddl0fts.cc:1186` | 利用归并后的全局有序，word 变则 flush、doc_id 变则清位置列表；最后复用 `fts_cache_node_add_positions` 编码 |
| `innodb_ft_sort_pll_degree` | `ha_innodb.cc:22530` → `ddl::fts_parser_threads` | 默认 2，范围 [1,16]，**向上取整到 2 的幂**。注意：**分词线程数可调，归并/插入线程数固定 = 6**（aux index 数） |

**（7）运维类**

| 算法 | 落点 | 关键点 |
|---|---|---|
| doc_id 分配 | `fts_get_next_doc_id` `fts0fts.cc:2764` | `doc_id_lock` 下 `++next_doc_id`；**步长上限 `FTS_DOC_ID_MAX_STEP = 65535`**——这是 VLC 5 字节上限反推出来的业务约束（`row0mysql.cc:1676` 显式检查） |
| cache 刷盘阈值 | `fts_sync` `fts0fts.cc:4405` | **不是 LRU**：超阈值全量 flush 后清空；超限后 `unlock_cache = false` 防止 sync 饿死 |
| OPTIMIZE 分批 + 游标 checkpoint | `fts0opt.cc:1464/1744/1622` | 每批 `innodb_ft_num_word_optimize`（默认 2000）个 word，逐个把 `last_optimized_word` 持久化到 config 表 ⇒ 可中断续跑 |
| 崩溃恢复增量扫描 | `fts_init_recover_doc` `fts0fts.cc:6120` | 从 `synced_doc_id` 扫到表内最大 doc_id，重新分词灌回 cache |
| 盲查询扩展 | `fts_expand_query` `fts0que.cc:3909` | 回读命中文档原文**重新分词**（源码注释承认"没有 doc→word 正排索引"）再二轮检索 |

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

`MATCH ... AGAINST` 的 EBNF（严格按 `sql_yacc.yy` 提炼）：

```ebnf
fulltext_expr   = MATCH ident_list_arg AGAINST "(" bit_expr fulltext_options ")" ;

ident_list_arg  = ident_list | "(" ident_list ")" ;      /* MATCH(a,b) 与 MATCH a,b 等价 */

fulltext_options  = [ opt_natural_language_mode ] [ opt_query_expansion ]
                  | "IN" "BOOLEAN" "MODE" ;

opt_natural_language_mode = %empty | "IN NATURAL LANGUAGE MODE" ;
opt_query_expansion       = %empty | "WITH QUERY EXPANSION" ;
```

产生式本体在 `sql_yacc.yy:10541`，action 直接构造 Item：

```
| MATCH ident_list_arg AGAINST '(' bit_expr fulltext_options ')'
  {
    $$= NEW_PTN Item_func_match(@$, $2, $5, $6);
  }
```

> **注意：没有 `PT_match` / `PTI_match` 这类 Parse Tree 节点**。语法 action 直接 `NEW_PTN Item_func_match(...)`，是少见的"语法阶段直接产出 Item"（MATCH 的语义在语法阶段就确定了，不需要占位符）。`Item_func_match` 仍是 `Parse_tree_node` 的派生，靠 `itemize()` 接管。

三种模式不是枚举，而是**位标志宏**（`include/ft_global.h:106-110`）：

```c
#define FT_NL 0         /* 自然语言 */
#define FT_BOOL 1       /* 布尔 */
#define FT_SORTED 2     /* 内部按 rank 排序 */
#define FT_EXPAND 4     /* 查询扩展 */
#define FT_NO_RANKING 8 /* 跳过打分 */
```

**为什么布尔模式不能叠加 `WITH QUERY EXPANSION`？** 这不是语义检查，是**纯语法层面的互斥**（`sql_yacc.yy:11026-11049`）：

- 分支 1 `opt_natural_language_mode opt_query_expansion` → `FT_NL(0) | [FT_EXPAND(4)]`，得 0 或 4
- 分支 2 `IN BOOLEAN MODE` → action 里**直接赋值** `$$= FT_BOOL;`，不是 `|=`

即分支 2 根本没有位置容纳 `FT_EXPAND`，`... IN BOOLEAN MODE WITH QUERY EXPANSION` 在 bison 归约时就是 syntax error，根本走不到语义检查。语义上也确实不成立：查询扩展是"首轮检索 → 抽词 → 二轮检索"，依赖相关性排序，而布尔模式不产生可用于扩展的经典 rank。

`print()` 里也用 `else if` 体现互斥（`item_func.cc:7792`）：`FT_BOOL` 优先打印 `in boolean mode`，否则才打印 `with query expansion`。

### 2.2 Item_func_match：从解析到取分

定义在 `sql/item_func.h:3396`。关键成员：

```cpp
class Item_func_match final : public Item_real_func {
  Item *against;                 // AGAINST 的表达式（必须是执行期常量）
  uint key, flags;               // key = 选中的 FT 索引号（或 NO_SUCH_KEY）
  bool score_from_index_scan{false};  // ★ 决定 val_real() 走哪条路
  FT_INFO *ft_handler;           // 引擎返回的句柄
  Table_ref *table_ref;
  Item_func_match *master;       // 等价 MATCH 的"主"对象，共享 ft_handler
  Item *concat_ws;               // 仅 MyISAM 非索引列布尔模式用
  Ft_hints *hints;               // 下推给引擎的意图
  bool used_in_where_only;       // 决定 can_skip_ranking()
};
```

**生命周期**（四个阶段各管一件事）：

| 阶段 | 函数 | 职责 |
|---|---|---|
| 语法 | action 直接 `new` | 构造 |
| 语义归位 | `itemize()` `item_func.cc:7405` | 挂进 `Query_block::ftfunc_list`；`lex->set_using_match()`；按 `parsing_place` 置 `used_in_where_only` |
| 名字解析 | `fix_fields()` `item_func.cc:7537` | 校验"全是列、同一张表、AGAINST 是执行期常量"；**顺手 `table_ref->set_fulltext_searched()`**（filesort 的伏笔） |
| 索引匹配 | `fix_index()` `item_func.cc:7660` | 判定列集合与某个 FULLTEXT 索引完全一致 |
| 执行前 | `init_search()` `item_func.cc:7433` | 调 `ft_init_ext_with_hints()` 拿到 `FT_INFO` |

> ★ **易错**：`fix_fields` 不做索引匹配，那是 `fix_index()` 的事，由 `setup_ftfuncs()`（`sql_base.cc:10303`）在 `Query_block::prepare` 里调用。

**`fix_index` 的判定条件**（`item_func.cc:7724-7726`）——这是"必须完全一致"的真正实现：

```cpp
// partial keys doesn't work
if (max_cnt < arg_count ||                                              // A
    max_cnt < table->key_info[ft_to_key[keynr]].user_defined_key_parts)  // B
  continue;
key = ft_to_key[keynr];
```

- **A：`max_cnt >= arg_count`** —— MATCH 的每一列都在该索引里
- **B：`max_cnt >= user_defined_key_parts`** —— 索引的每一列也都被 MATCH 覆盖

A ∧ B 合起来就是"**列集合一一对应**"。失败时：若引擎允许非索引列布尔搜索（MyISAM，`allows_search_on_non_indexed_columns`）则退化成 `key = NO_SUCH_KEY` 走 `concat_ws` 拼串，否则报 `ER_FT_MATCHING_KEY_NOT_FOUND`。

**两个 MATCH 如何共享一次搜索**：`setup_ftfuncs` 里用 `eq()` 找等价的 MATCH，后来的那个 `set_master()` 指向前者；slave 的 `init_search()` 直接复用 `master->ft_handler`，只有 master 负责 `close_search`。注意 `eq()` **刻意忽略 `FT_SORTED`** 标志（`item_func.cc:7744`）——因为排序与否不改变"这是同一个搜索"。

**`val_real()` 的三条路径**（`item_func.cc:7760`）——分支条件常被误解成"布尔 vs 自然语言"，其实不是：

```cpp
if (ft_handler == nullptr) return -1.0;                        // ① 未初始化
if (key != NO_SUCH_KEY && table->has_null_row()) return 0.0;   // ② 外连接补的 NULL 行

if (get_master()->score_from_index_scan)                       // ③ 正在走 FT 索引扫描
  return ft_handler->please->get_relevance(ft_handler);
if (key == NO_SUCH_KEY) {                                      // ④ 无可用 FT 索引（MyISAM 非索引列）
  String *a = concat_ws->val_str(&value);
  return ft_handler->please->find_relevance(ft_handler, (uchar *)a->ptr(), a->length());
}
return ft_handler->please->find_relevance(ft_handler, table->record[0], 0);  // ⑤ 其余（表扫/随机访问）
```

| 分支 | 条件 | InnoDB 侧实现 |
|---|---|---|
| ③ `get_relevance` | **`score_from_index_scan == true`**（正在用 `FullTextSearchIterator`） | `innobase_fts_retrieve_ranking`（直接取引擎当前结果项的 rank） |
| ⑤ `find_relevance` | 没走 FT 索引扫描（如 filesort 后的随机访问、表扫） | `innobase_fts_find_ranking`（按 doc_id 回查） |

> **关键：`Read()` 从不"回填"rank，是惰性拉取。** `FullTextSearchIterator::Read()` 只调 `ha_ft_read()`；rank 由上层求值时 `val_real()` → `get_relevance()` 从引擎取。引擎（InnoDB 的 `NEW_FT_INFO.ft_result`）在 `ft_read` 时已经把游标停在当前结果项上。

**`Item_func_match_predicate`（`item_cmpfunc.h:762`）**：MATCH 在 WHERE 里单独作谓词时，`make_condition()`（`item_cmpfunc.cc:5459`）会把它包一层。为什么需要？若不包装，`make_condition` 会生成 `Item_func_ne(0, match)`，那是"比较"语义（**带 ranking**）；而历史行为里裸 `WHERE MATCH(...)` 是布尔语义，**允许引擎跳过 ranking**（`can_skip_ranking()`）。这层包装就是为了让这两种语义分开。

### 2.3 使用限制（这些是硬约束，写代码时必须知道）

| 限制 | 报错 | 位置 |
|------|------|------|
| AGAINST 必须是**常量**（不能是列、子查询） | `ER_WRONG_ARGUMENTS` "AGAINST" | `item_func.cc:7556` |
| MATCH 参数必须是**列**，不能是表达式/外引用 | `ER_WRONG_ARGUMENTS` "MATCH" | `item_func.cc:7567` |
| 所有列必须来自**同一张表** | 同上 | `item_func.cc:7590` |
| **MATCH 的列集合必须与某个 FULLTEXT 索引完全一致**（不能多也不能少） | `ER_FT_MATCHING_KEY_NOT_FOUND` | `item_func.cc:7739`（判定 `:7680-7731`） |
| 不能用于生成列 / 函数索引 | `check_function_as_value_generator` | `item_func.h:3472` |
| 不能在物化子查询上执行 | `ER_NO_FT_MATERIALIZED_SUBQUERY` | `item_func.cc:7494` |
| **不能引用 ROLLUP 列** | `ER_FULLTEXT_WITH_ROLLUP` | `sql_resolver.cc:371-374` |

> **"FTS 只能用在 WHERE"是过时的说法。** 8.0 里：
> - `ORDER BY MATCH(...)` 合法且**可能免排序**（`test_if_ft_index_order`，`sql_optimizer.cc:1733`）
> - `HAVING` / `GROUP BY` 里出现 MATCH **也是允许的**——`fulltext_uses_rollup_column()`（`sql_resolver.cc:5090`）明确把 "SELECT list / GROUP BY / HAVING / ORDER BY" 都列为受检查位置，说明这些位置本身合法
> - 真正被禁的只有 **ROLLUP 列**（以及上面表里那些"必须是裸列、必须同表"的约束）

### 2.4 优化器如何选上 FT 索引

**FT 走的是一条独立通道，不经过 range 优化器**——`range_optimizer.cc:395` 显式跳过带 `HA_FULLTEXT` 标志的索引（`cause: "fulltext"`）。整条通道是：

```
update_ref_and_keys()            sql_optimizer.cc:5343 → 8279
 └─ add_ft_keys()                sql_optimizer.cc:8381 → 7822   造 Key_use(keypart = FT_KEYPART)
     → find_best_ref()           sql_planner.cc:207（FT 分支 :669-685）
     → create_ref_for_key()      sql_select.cc:2419（ftkey 分支 :2434-2468）→ set_type(JT_FT)
```

- **`FT_KEYPART` 是哨兵值不是 keypart 编号**：`= MAX_REF_PARTS(16) + 10 = 26`（`sql_select.h:105`）。用它做标记，是因为 FT 要求索引用满全部列，"遍历接下来 actual_key_parts 个"仍然成立（`sql_planner.cc:1343-1349` 的注释解释）。

**代价估算几乎是空的**（`sql_planner.cc:681-684`）：

```cpp
// Actually it should be cur_fanout=0.0 (yes!) but 1.0 is probably safer
cur_read_cost = prev_record_reads(join, idx, table_deps) *
                table->cost_model()->page_read_cost(1.0);
cur_fanout = 1.0;
```

即 **fanout 恒为 1.0、成本恒为 1 次页读**——MySQL 对 FT 命中多少行没有估算模型（连注释都在自嘲）。这也是 EXPLAIN 里 `rows=1` 的来源。另外 FT 的 `cur_keytype` 排在 `NOT_UNIQUE` 之后，只要已找到 `EQ_REF` 之类就整条跳过；semijoin loosescan 也不用 fulltext。

计划类型 **`JT_FT`**（= 8，`sql_opt_exec_shared.h:221`），EXPLAIN 打印 `"fulltext"`（`join_type_str[8]`，`opt_explain.cc:117`）。`FT_SELECT` 是 5.x 遗留名，本版本不存在。

**★★★ `add_ft_keys()` 完整实现**（`sql/sql_optimizer.cc:7822`，★ 真名是**复数** `add_ft_keys`，不存在单数的 `add_ft_key`）：

```cpp
static bool add_ft_keys(Key_use_array *keyuse_array, Item *cond,
                        table_map usable_tables, bool simple_match_expr) {
  Item_func_match *cond_func = nullptr;

  if (!cond) return false;

  assert(cond->is_bool_func());

  if (cond->type() == Item::FUNC_ITEM) {
    Item_func *func = down_cast<Item_func *>(cond);
    Item_func::Functype functype = func->functype();
    if (functype == Item_func::MATCH_FUNC) {
      func = down_cast<Item_func *>(func->arguments()[0]);
      functype = func->functype();
    }
    enum ft_operation op_type = FT_OP_NO;
    double op_value = 0.0;
    if (functype == Item_func::FT_FUNC) {
      cond_func = down_cast<Item_func_match *>(func)->get_master();
      cond_func->set_hints_op(op_type, op_value);      /* ① 裸 MATCH(...) */
    } else if (func->arg_count == 2) {
      Item *arg0 = func->arguments()[0];
      Item *arg1 = func->arguments()[1];
      if (arg1->const_item() && is_function_of_type(arg0, Item_func::FT_FUNC) &&
          ((functype == Item_func::GE_FUNC &&
            (op_value = arg1->val_real()) > 0) ||      /* ② MATCH >= const, const>0 */
           (functype == Item_func::GT_FUNC &&
            (op_value = arg1->val_real()) >= 0))) {    /* ③ MATCH >  const, const>=0 */
        cond_func = down_cast<Item_func_match *>(arg0)->get_master();
        if (functype == Item_func::GE_FUNC)
          op_type = FT_OP_GE;
        else if (functype == Item_func::GT_FUNC)
          op_type = FT_OP_GT;
        cond_func->set_hints_op(op_type, op_value);
      } else if (arg0->const_item() &&                 /* ④ const <= MATCH */
                 is_function_of_type(arg1, Item_func::FT_FUNC) && ... ) {
        cond_func = down_cast<Item_func_match *>(arg1)->get_master();
        ...
      }
    }
  } else if (cond->type() == Item::COND_ITEM) {
    List_iterator_fast<Item> li(*down_cast<Item_cond *>(cond)->argument_list());

    if (down_cast<Item_cond *>(cond)->functype() == Item_func::COND_AND_FUNC) {
      Item *item;
      while ((item = li++))
        if (add_ft_keys(keyuse_array, item, usable_tables, false)) return true;
    }                       /* ★★ 只递归 AND —— OR 明确不支持 */
  }

  if (!cond_func || cond_func->key == NO_SUCH_KEY ||
      !(usable_tables & cond_func->table_ref->map()))
    return false;

  Table_ref *tbl = cond_func->table_ref;
  if (!tbl->table->keys_in_use_for_query.is_set(cond_func->key)) return false;

  cond_func->set_simple_expression(simple_match_expr);

  const Key_use keyuse(tbl, cond_func, cond_func->key_item()->used_tables(),
                       cond_func->key, FT_KEYPART,   /* ★★ keypart 填哨兵值 */
                       0,            // optimize
                       0,            // keypart_map
                       ~(ha_rows)0,  // ref_table_rows
                       false,        // null_rejecting
                       nullptr,      // cond_guard
                       UINT_MAX);    // sj_pred_no
  tbl->table->reginfo.join_tab->keys().set_bit(cond_func->key);
  return keyuse_array->push_back(keyuse);
}
```
> ★★ 三个要点：① **只支持 AND**（`COND_AND_FUNC`），所以 `WHERE MATCH(a) OR MATCH(b)` 用不上 FT 索引；② 支持 `MATCH(...) >= v / > v / <= v / < v` 四种比较，并把 `(op_type, op_value)` 写进 `Ft_hints` 下推给引擎 —— 这就是 `FT_OP_GT`/`FT_OP_GE` 的**唯一来源**；③ `keyuse` 的 keypart 填常量 `FT_KEYPART` 而非真实列号，这是 FT 在整个优化器里的"识别码"。

调用点（`sql_optimizer.cc:8380`，位于 `update_ref_and_keys()` 内）：

```cpp
  if (query_block->ftfunc_list->elements) {
    if (add_ft_keys(keyuse, cond, normal_tables, true)) return true;
  }
```
> ★ 仅当 query_block 有 FT 函数时才走；首个调用 `simple_match_expr=true`。

**★ `JT_FT` 的唯一判定入口**（`create_ref_for_key` `sql/sql_select.cc:2419`）：

```cpp
bool create_ref_for_key(JOIN *join, JOIN_TAB *j, Key_use *org_keyuse,
                        table_map used_tables) {
  const uint key = org_keyuse->key;
  const bool ftkey = (org_keyuse->keypart == FT_KEYPART);   /* ★★ 就是靠这个哨兵 */
  ...
  if (ftkey) {
    Item_func_match *ifm = down_cast<Item_func_match *>(org_keyuse->val);

    length = 0;                 /* ★ FT 的 key length 恒为 0 */
    keyparts = 1;               /* ★ 恒为 1 */
    ifm->get_master()->score_from_index_scan = true;
                                /* ★ 置位后可直接 get_relevance() 拿分，不必回表算 */
  } else /* not ftkey */
    calc_length_and_keyparts(...);
  ...
  if (ftkey) {
    Key_use *keyuse = org_keyuse;
    j->ref().items[0] = ((Item_func *)(keyuse->val))->key_item();
    /* Predicates pushed down into subquery can't be used FT access */
    j->ref().cond_guards[0] = nullptr;
    // not supported yet. SerG
    assert(!(keyuse->used_tables & ~PSEUDO_TABLE_BITS));

    j->set_type(JT_FT);                                    /* ★★ 唯一 set_type(JT_FT) 处 */
    j->set_ft_func(down_cast<Item_func_match *>(keyuse->val));
    memset(j->ref().key_copy, 0, sizeof(j->ref().key_copy[0]) * keyparts);
                                /* ★ key_copy 全清零：FT 不走"键值比较"，走 ft_handler 回调 */
    return false;
  }
```

**★ `FT_KEYPART` 哨兵常量的定义**（`sql/sql_select.h:105`）：

```cpp
/* Values in optimize */
#define KEY_OPTIMIZE_EXISTS 1
#define KEY_OPTIMIZE_REF_OR_NULL 2
#define FT_KEYPART (MAX_REF_PARTS + 10)
```
> ★ 一个超出任何合法 keypart 编号的值，用于把 FT 的 `Key_use` 混进同一个数组但能被区分出来。

**★ FT 在代价/行数估算里的两处特例**（`sql/sql_planner.cc`）：

```cpp
    enum idx_type cur_keytype =
        (keyuse->keypart == FT_KEYPART) ? FULLTEXT : NOT_UNIQUE;
```
```cpp
  if (keyuse) {
    const KEY *key = table->key_info + keyuse->key;

    if (keyuse[0].keypart == FT_KEYPART) {
      /*
        Fulltext indexes are special because keyuse->keypart does not
        contain the keypart number but a constant (FT_KEYPART)
        defining that it is a fulltext index. However, since fulltext
        search demands that all indexed keyparts are used, iterating
        over the next 'actual_key_parts' works.
      */
      for (uint i = 0; i < key->actual_key_parts; i++)
        bitmap_set_bit(&table->tmp_set, key->key_part[i].field->field_index());
    } else {
```
> ★ 因为 keypart 不是列号，计算 filtering effect 时改成遍历 `actual_key_parts` 把所有列标进 `tmp_set`。这印证"FT 要求所有索引列都被用到"。

**可下推的谓词形态共 5 种**（`IsSargableFullTextIndexPredicate` `join_optimizer.cc:5132`）：

| 形态 | 约束 | hint 算子 |
|---|---|---|
| `WHERE MATCH(c) AGAINST('s')` | — | `FT_OP_NO` |
| `MATCH(...) > const` | `const >= 0` | `FT_OP_GT` |
| `MATCH(...) >= const` | **`const > 0`** | `FT_OP_GE` |
| `const < MATCH(...)` | `const >= 0` | `FT_OP_GT` |
| `const <= MATCH(...)` | **`const > 0`** | `FT_OP_GE` |

> **为什么 `>` 和 `>=` 的约束不对称？** 因为 **FT 索引扫描只返回 score > 0 的文档**。`MATCH > 0` 对所有命中文档为真、对所有未命中（score=0）为假 ⇒ 索引结果恰好等价于谓词，可下推；而 `MATCH >= 0` 对 score=0 的文档**也为真** ⇒ 索引结果不完整，不可下推（除非 `const > 0`）。本质是"索引扫描的结果集必须恰好等于谓词的真值集"。

**不能下推的形态**：`MATCH < const` / `<=`（会含 score=0）、`= const`、`!=`、出现在 OR 分支里（`add_ft_keys` 只递归 `COND_AND_FUNC`）、`MATCH` 参与表达式（如 `MATCH(...)+1 > 0.5`）、const 不是常量或为 NULL、条件下推进子查询后（"Predicates pushed down into subquery can't be used FT access"）。

**`ORDER BY MATCH(...)` 可能免排序**（这是常见误解的反面——"FTS 只能用在 WHERE"是过时说法）：

`test_if_ft_index_order`（`sql_optimizer.cc:1733`）要求**四条同时满足**：ORDER BY 只有一项、方向是 **`ORDER_DESC`**、该项是裸 `MATCH`（`FT_FUNC` 而非被包装的 `MATCH_FUNC`）、引擎结果本身有序。命中后有两条路：

1. 已在用 FT 索引扫描（`tab->type() == JT_FT` 且是同一个 MATCH）→ 设 `FT_SORTED`，直接免排序
2. 原本没用索引 → 若有 LIMIT 且 `LIMIT <= ft_func->get_count()` 且无 WHERE，**强行把计划改造成 FT 扫描**（`set_type(JT_FT)`，`sql_optimizer.cc:2265`）

**hypergraph 优化器支持 fulltext，而且很完整**（8.0.31+ 默认仍未开启，但代码齐全）：

| 设施 | 位置 |
|---|---|
| `AccessPath::FULL_TEXT_SEARCH` | `join_optimizer/access_path.h:219` |
| `IsSargableFullTextIndexPredicate` / `FindSargableFullTextPredicates` | `join_optimizer.cc:5132 / 5203` |
| `ProposeAllFullTextIndexScans` / `ProposeFullTextIndexScan` | `join_optimizer.cc:2507 / 2568` |
| **禁止 FTS 表参与 hash join** | `join_optimizer.cc:3274` |

> hash join 被禁的理由很有意思，也点出了 FTS 的一个隐含约束：**求 MATCH 值要求 handler 真实定位在行上**，而 hash join 返回行的顺序与底层扫描不一致，于是"保守地不提议"。

两个优化器的**唯一实质差异**是 `score_from_index_scan` 的置位时机：旧优化器在 `create_ref_for_key` 选定计划时就置位；hypergraph 在整个计划定稿前一直同时保留"用/不用索引扫描"两个候选，因此推迟到 `FullTextSearchIterator` 构造函数里才置位。

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

> **`ft_end()` / `ft_update()` 在 server 层 handler 接口中不存在**（`sql/handler.h` 只有 `ft_init` / `ft_init_ext` / `ft_init_ext_with_hints` / `ha_ft_read` / `ft_read`）。InnoDB 有一个私有且**无人调用**的 `ha_innobase::ft_end()`（`ha_innodb.cc:11234`，只打日志）；MyISAM 有内部函数 `_mi_ft_update()`（`ft_update.cc:187`，行更新时同步 FT 索引，不属于 handler 接口）。资源释放实际走 `_ft_vft::close_search`，DML 侧的索引维护走 `fts_trx_add_op`，**不走 handler 虚函数**。

### 2.6 Ft_hints：server 层下推给引擎的"意图"

这是 FTS 与其它索引最大的区别：**两层之间传递的不只是键，还有一组 hints**。

`Ft_hints`（`sql/handler.h:3969` 的 C++ 包装，底层 `struct ft_hints` 在 `include/ft_global.h:117`）：

| 字段 | 含义 | 由谁设置 |
|---|---|---|
| `flags` | `FT_SORTED`（引擎需按 rank 排）/`FT_NO_RANKING`（可跳过打分）/`FT_BOOL` | 优化器：`test_if_skip_sort_order`、`optimize_fts_query`、`init_ftfuncs`（hypergraph 补设 `FT_NO_RANKING`） |
| `op_type` + `op_value` | rank 阈值（`FT_OP_GT` / `FT_OP_GE`） | `add_ft_keys` 从 `MATCH > const` 形态提取 |
| `limit` | 结果截断条数 | `set_hint_limit(join->m_select_limit)` |

InnoDB 侧的实际消费只有一处（`ha_innobase::ft_init_ext_with_hints`，`ha_innodb.cc:11042`）：

```cpp
/* TODO Implement function properly working with FT hint. */
if (hints->get_flags() & FT_NO_RANKING) {
  m_prebuilt->m_fts_limit = hints->get_limit();
} else {
  m_prebuilt->m_fts_limit = ULONG_UNDEFINED;
}
```

即**只有"不需要打分"时才把 limit 传下去**——因为要排序就必须拿到全部候选，无法提前截断。

`FT_NO_RANKING` 的设置条件（`can_skip_ranking`，`item_func.h:3586`）：没设 `FT_SORTED` **且** `used_in_where_only`（结果不出现在表达式里）**且** `op_type == FT_OP_NO`（没有比较算子）。

### 2.7 执行器：FullTextSearchIterator

`sql/iterators/ref_row_iterators.cc:593`，很短但有几个关键点：

```cpp
FullTextSearchIterator::FullTextSearchIterator(..., bool use_order, bool use_limit, ...) {
  if (thd->lex->using_hypergraph_optimizer()) {
    ft_func->score_from_index_scan = true;                       // ① hypergraph 在这里才置位
    if (table->covering_keys.is_set(ft_func->key) && !table->no_keyread)
      table->set_keyread(true);                                  // ② 覆盖索引：只读 FTS_DOC_ID
    if (use_order) ft_func->get_hints()->set_hint_flag(FT_SORTED);
    if (use_limit) ft_func->get_hints()->set_hint_limit(...);
  }
  assert(ft_func->score_from_index_scan);
}

bool FullTextSearchIterator::Init() {
  if (!table()->file->inited) {
    int error = table()->file->ha_index_init(m_ref->key, m_use_order);   // ③ 用 FT 索引号 init
    ...
  }
  m_ft_func->score_from_index_scan = true;
  return table()->file->ft_init() != 0;                                  // ④ 引擎定位到结果集开头
}

int FullTextSearchIterator::Read() {
  int error = table()->file->ha_ft_read(table()->record[0]);             // ⑤ 取一行（含回表）
  ...
}
```

五点说明：

1. **hypergraph 的置位推迟到构造函数**：因为它同时保留"用/不用索引扫描"两个候选到最后一刻（见 2.4）。
2. **覆盖索引优化**：若 FT 索引是 covering 且未禁 keyread，`set_keyread(true)` → 引擎走 `read_just_key` 分支**完全不回表**，只取 `FTS_DOC_ID`（`ha_innodb.cc:11146`）。
3. `ha_index_init(m_ref->key, m_use_order)` 传的是 **FT 索引号**，`m_use_order` 决定是否要求有序。
4. `ft_init()`（`ha_innodbase::ft_init`，`ha_innodb.cc:10894`）只做 `++trx->will_lock` + `rnd_init(false)`——真正的查询早在 `ft_init_ext` 里就做完了。
5. `ha_ft_read` 是**非虚包装**（`handler.cc:3028`），负责生成列更新，里面才调虚函数 `ft_read()`。

**主链路**：

```
JOIN::optimize
 └─ JOIN::optimize_fts_query                  sql_optimizer.cc:10839（设 hints）
     └─ init_ftfuncs                          sql_optimizer.cc:10886 → sql_base.cc:10319
         └─ Item_func_match::init_search      item_func.cc:7433
             └─ handler::ft_init_ext_with_hints  → ha_innobase（InnoDB）→ fts_query()
[执行]
FullTextSearchIterator::Init → ft_init()
FullTextSearchIterator::Read → ha_ft_read → ft_read（InnoDB：遍历 rankings_by_rank + 回表）
上层求值时 Item_func_match::val_real → get_relevance
```

### 2.8 filesort 的 FTS 特例：强制退回 rowid 模式

只要表被 MATCH 引用，filesort 就**放弃 addon fields**。判定在 `decide_addon_fields` 的**最前面**，优先级高于 `force_sort_rowids`（`sql/filesort.cc:189-196`）：

```cpp
for (TABLE *table : tables) {
  if (table->pos_in_table_list &&
      table->pos_in_table_list->is_fulltext_searched()) {
    // See comment in SortWillBeOnRowId().
    m_addon_fields_status = Addon_fields_status::fulltext_searched;
    return;
  }
}
```

标记 `is_fulltext_searched()` 的唯一置位点是 `Item_func_match::fix_fields`（`item_func.cc:7599`）。原因（`SortWillBeOnRowId`，`filesort.cc:2187`，源码注释说得很直白）：

```
MATCH() (except in "boolean mode") doesn't use the actual value,
it just goes and asks the handler directly for the current row.
Thus, we need row IDs, so that the row is positioned correctly.
```

**为什么不得不这样**：addon 模式排序后直接从 sort buffer 反序列化出整行，**不再经过 handler**；而 `MATCH()` 的 rank 要"问"handler 要（`val_real` 的两个分支都依赖引擎当前状态）。没有 `rnd_pos()` 定位，取到的就是上一行的 rank。rowid 模式排序后用 `ha_rnd_pos` 真实定位，才正确。

配套的收尾动作在 `EndFullTextIndexScan`（`sql/iterators/sorting_iterator.cc:73`）：排序完成、切回随机访问前，把 `score_from_index_scan` 置回 `false`，让 `val_real()` 改走 `find_relevance` 路径：

```cpp
static void EndFullTextIndexScan(TABLE *table) {
  if (table->file->ft_handler != nullptr) {
    for (Item_func_match &ft_func : *table->pos_in_table_list->query_block->ftfunc_list) {
      if (ft_func.master == nullptr && ft_func.ft_handler == table->file->ft_handler) {
        ft_func.score_from_index_scan = false;
        break;
      }
    }
  }
}
```

> 这是 8.0.20 之后 addon 成为默认（见 [`../server/query/runtime/01_filesort.md`](../server/query/runtime/01_filesort.md)）后，极少数**仍然强制 rowid** 的场景之一（另两个是 UPDATE/DELETE 的两阶段读、大 BLOB）。

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

- 非 CJK：按首字符排序权重落在哪个区间 —— 边界数组 `{9, 65, 70, 75, 80, 85}`（`fts0fts.cc:151`）
- CJK（见 `fts0types.ic:99-113`）：首字符分布集中，改用 `hash % 6` 避免倾斜

**非 CJK 的真实区间**（`fts_select_index_by_range`，`fts0types.ic:120`）——这里有个常见误解，必须照着循环逐行推：

```c
while (fts_index_selector[selected].value != 0) {
  if (fts_index_selector[selected].value == value) return selected;
  else if (fts_index_selector[selected].value > value)
    return (selected > 0 ? selected - 1 : 0);
  ++selected;
}
return (selected - 1);
```

| sort weight | 分片 | 典型字符（latin1_swedish_ci） |
|---|---|---|
| **< 65**（即 `value ≤ 64`） | **`index_1`** | 数字、符号、控制字符 |
| 65–69 | `index_2` | A–E（ci 下含 a–e） |
| 70–74 | `index_3` | F–J |
| 75–79 | `index_4` | K–O |
| 80–84 | `index_5` | P–T |
| ≥ 85 | `index_6` | U–Z 及更高 |

即 **`index_1` 是一个覆盖全部"非字母"字符的巨大桶**，只有后 5 张才是字母区间均分——并不是"6 个等宽区间"。边界里的 `9` 不产生独立分片（`value == 9` 与 `value < 65` 都返回 0）。

> **这是真实存在的 bad case**：CJK 判定 `fts_is_charset_cjk` 只按 collation 名精确 `strcmp` 匹配 **9 个**（gb2312/gbk/big5/gb18030/ujis/sjis/cp932/eucjpms/euckr），**不含 utf8mb4**。所以 utf8mb4 的中文列如果不配 ngram parser，词会全部按 range 规则涌进 `index_1`，造成严重的数据倾斜——这也是"中文必须配 ngram"的另一层原因（主因当然是内置 parser 按 `true_word_char` 切词，中文不是字母/数字/下划线，根本切不出词）。

### 3.3 INDEX 表的结构

实际建表代码（`fts_create_one_index_table`，`fts0fts.cc:1978`，列定义在 `:1998-2037`）：

```c
dict_mem_table_add_col(new_table, heap, "word",
    charset == &my_charset_latin1 ? DATA_VARCHAR : DATA_VARMYSQL,
    field->col->prtype, FTS_MAX_WORD_LEN /* 336 */, true);
dict_mem_table_add_col(new_table, heap, "first_doc_id", DATA_INT,
    DATA_NOT_NULL | DATA_UNSIGNED, 8, true);
dict_mem_table_add_col(new_table, heap, "last_doc_id", DATA_INT,
    DATA_NOT_NULL | DATA_UNSIGNED, 8, true);
dict_mem_table_add_col(new_table, heap, "doc_count", DATA_INT,
    DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
dict_mem_table_add_col(new_table, heap, "ilist", DATA_BLOB,
    (DATA_MTYPE_MAX << 16) | DATA_UNSIGNED | DATA_NOT_NULL, 0, true);
/* 聚簇唯一索引 FTS_INDEX_TABLE_IND 建在 (word, first_doc_id) 上 */
```

> **★ 别照抄源码注释**：`fts0fts.cc:2321-2327` 那段"等价 SQL"注释写的是 `first_doc_id INT NOT NULL` / `doc_count UNSIGNED INT`，**是陈旧的**。真实情况是 **`first_doc_id` / `last_doc_id` 都是 8 字节 BIGINT UNSIGNED NOT NULL**，只有 `doc_count` 是 4 字节。这类"注释比代码老"的情况在 FTS 里不止一处（另见 §11.3 的 `RANK_UPGRADE`）。

**注意同一 word 可能有多行**：一行（一个 node）的 ilist 上限是 `FTS_ILIST_MAX_SIZE = 64KB`（`fts0priv.h:74`），超了就新起一行，靠 `first_doc_id` 区分。所以查一个词要按 `(word, first_doc_id)` 顺序扫多个 node。

### 3.4 公共表的结构

建表代码在 `fts_create_one_common_table`（`fts0fts.cc:1802`）：

```c
if (!is_config) {                       /* deleted / deleted_cache / being_deleted / *_cache */
  dict_mem_table_add_col(new_table, heap, "doc_id", DATA_INT,
                         DATA_UNSIGNED, 8, true);   /* 注意：没有 DATA_NOT_NULL */
} else {                                /* config */
  dict_mem_table_add_col(new_table, heap, "key",   DATA_VARCHAR, 0,          50,  true);
  dict_mem_table_add_col(new_table, heap, "value", DATA_VARCHAR, DATA_NOT_NULL, 200, true);
}
/* 聚簇唯一索引 FTS_COMMON_TABLE_IND 建在 doc_id（或 key）上 */
```

CONFIG 表的初始 5 行由 `fts_config_table_insert_values_sql`（`fts0fts.cc:161-176`）写入：

| 键字符串（实际值） | 常量宏 | 定义 | 默认值 |
|---|---|---|---|
| `cache_size_in_mb` | `FTS_MAX_CACHE_SIZE_IN_MB` | `fts0fts.cc:63` | `256` |
| `optimize_checkpoint_limit` | `FTS_OPTIMIZE_LIMIT_IN_SECS` | `fts0priv.h:79` | `180`（秒） |
| `synced_doc_id` | `FTS_SYNCED_DOC_ID` | `fts0priv.h:82` | `0` |
| **`deleted_doc_count`** | `FTS_TOTAL_DELETED_COUNT` | `fts0priv.h:89` | `0` |
| `table_state` | `FTS_TABLE_STATE` | `fts0priv.h:108` | `0` |

> **★ 宏名 ≠ 字符串值**：宏叫 `FTS_TOTAL_DELETED_COUNT`，但写进 CONFIG 表的键名是 **`deleted_doc_count`**。

`table_state` 的取值（`fts0priv.h:46-57`，**不在 `fts0opt.h`**）：

```c
enum fts_table_state_enum {
  FTS_TABLE_STATE_RUNNING = 0,   /* Auxiliary tables created OK */
  FTS_TABLE_STATE_OPTIMIZING,    /* This is a substate of RUNNING */
  FTS_TABLE_STATE_DELETED        /* All aux tables to be dropped when it's safe */
};
```

> **★ 别和 `fts0opt.cc:82` 的 `fts_state_t` 混为一谈**：那里的 `FTS_STATE_LOADED/RUNNING/SUSPENDED/DONE/EMPTY` 是**后台 optimize 线程里 slot 的状态机**（`slot->state`），与 CONFIG 表的 `table_state` 毫无关系。

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

> **视角声明**：倒排列表的**存储与编码是索引结构问题**，权威剖析见 [`../index/inverted.md`](../index/inverted.md)（ilist 字节级布局、VLC 终止位与 delta、6 档分档、删除旁路表示与 optimize 重编码）。本篇作为全文检索全链路的一环保留编码细节，但**不重复那里的权威内容**——若两处冲突以 `inverted.md` 为准。

### 5.1 VLC 变长整数

`fts0vlc.ic`：每字节低 7 位存数据，**高位 0x80 表示"这是最后一个字节"**（与 JSONB 的 0x80 续行位方向相反！）：

```c
// fts0vlc.ic:64
static inline ulint fts_encode_int(ulint val, byte *buf) {
  ...
  /* High-bit on means "last byte in the encoded integer". */
  *buf |= 0x80;
```

最多 5 字节：`≤127`→1B，`≤16383`→2B，`≤2097151`→3B，`≤268435455`→4B，否则 5B，且每字节装 **7 个有效位、高位在前**（`*buf++ = val >> 7; ...`）。

> 5 字节上限意味着**值域被限死在 32 位**（`ut_ad(val <= 4294967295u)`，`fts0vlc.ic:95`），源码注释（`:53-56`）自己也承认这是一处已知的、未修的限制。

**与 InnoDB 另一套变长编码 VLQ（`mach_write_compressed`，redo/undo 用）的对比**（完整剖析见 [`../server/infra/encoding.md`](../server/infra/encoding.md)）：

| | VLC（FTS，本文件） | VLQ（`mach_write_compressed`） |
|---|---|---|
| 长度信息 | **末字节**高位 0x80 = "last byte" | **首字节前缀**（`0`/`10`/`110`/`1110`/`11110000`） |
| 每字节有效位 | 恒定 7 位 | 首字节 7 位、后续字节 8 位 |
| 解码 | 必须**扫到末字节**才知道结束 | 看首字节**即知总长** |
| 适用访问模式 | 纯顺序追加/顺序读（ilist 字节流） | 要能按字节定位边界、跨过记录（redo 解析） |

★ 同一代码库里两套变长编码并存，**各自匹配自己的访问模式**：FTS ilist 永远顺序消费，后缀标志够用；redo 解析器要随机定位下一条记录，必须前缀码。

### 5.2 ilist 布局

```
[VLC(doc_id - prev_doc_id)]  [VLC(pos1 - 0)] [VLC(pos2 - pos1)] ... [0x00]
 └── 每段一个文档 ─────────────────────────────────────────────────────┘
```

- doc_id 和 position 都是**增量（delta）编码**，第一个文档的 prev 是 0
- 位置列表以 `0x00` 结尾（解码时 `while (*ptr)` 靠"真实 VLC 首字节非 0"来识别）
- 编码：`fts_cache_node_add_positions()` `fts0fts.cc:1052`
- 解码：`fts_query_filter_doc_ids()` `fts0que.cc:2958`

**举例**：一个 node 覆盖 doc 1（位置 1、4、9）与 doc 5（位置 2），`last_doc_id` 初值 0：

| 内容 | delta | 编码字节 |
|---|---|---|
| doc 1 | 1 − 0 = 1 | `81` |
| pos 1 | 1 − 0 = 1 | `81` |
| pos 4 | 4 − 1 = 3 | `83` |
| pos 9 | 9 − 4 = 5 | `85` |
| 位置列表结束 | — | `00` |
| doc 5 | 5 − 1 = 4 | `84` |
| pos 2 | 2 − 0 = 2 | `82` |
| 位置列表结束 | — | `00` |

→ `ilist = 81 81 83 85 00 84 82 00`（8 字节）

三个易错点：

1. **每个文档的位置列表以 `0x00` 结尾**（不是整个 ilist 只在最后加一个）。
2. **位置 delta 在每个新文档开始时重置为 0**，不是跨文档累积。
3. 第一个 doc 的 delta 是相对 `node->last_doc_id` 的（新 node 为 0，所以等于绝对值），解码端用 `ut_a(pos == node->first_doc_id)` 断言了这一点（`fts0que.cc:2985`）。

**★★ VLC 三个函数的真名与完整实现**（`fts0vlc.ic`，★ 真名核查：是 `fts_encode_int` / `fts_decode_vlc` / `fts_get_encoded_len`，**不存在** `fts_vlc_encode` / `fts_encode_ulint`）：

```c
/** Return length of val if it were encoded using our VLC scheme.
 FIXME: We will need to be able encode 8 bytes value
 @return length of value encoded, in bytes */
static inline ulint fts_get_encoded_len(ulint val) /* in: value to encode */
{
  if (val <= 127) {           /* 2^7 - 1 */
    return (1);
  } else if (val <= 16383) {  /* 2^14 - 1 */
    return (2);
  } else if (val <= 2097151) {/* 2^21 - 1 */
    return (3);
  } else if (val <= 268435455) { /* 2^28 - 1 */
    return (4);
  } else {
    /* Possibly we should care that on 64-bit machines ulint can
    contain values that we can't encode in 5 bytes, but
    fts_encode_int doesn't handle them either so it doesn't much
    matter. */
    return (5);               /* ★ 最多 5 字节 = 35 位有效位 */
  }
}

/** Encode an integer using our VLC scheme and return the length in bytes.
 @return length of value encoded, in bytes */
static inline ulint fts_encode_int(
    ulint val, /* in: value to encode */
    byte *buf) /* in: buffer, must have enough space */
{
  ulint len;

  if (val <= 127) {
    *buf = (byte)val;
    len = 1;
  } else if (val <= 16383) {
    *buf++ = (byte)(val >> 7);
    *buf = (byte)(val & 0x7F);
    len = 2;
  } else if (val <= 2097151) {
    *buf++ = (byte)(val >> 14);
    *buf++ = (byte)((val >> 7) & 0x7F);
    *buf = (byte)(val & 0x7F);
    len = 3;
  } else if (val <= 268435455) {
    *buf++ = (byte)(val >> 21);
    *buf++ = (byte)((val >> 14) & 0x7F);
    *buf++ = (byte)((val >> 7) & 0x7F);
    *buf = (byte)(val & 0x7F);
    len = 4;
  } else {
    /* Best to keep the limitations of the 32/64 bit versions
    identical, at least for the time being. */
    ut_ad(val <= 4294967295u);

    *buf++ = (byte)(val >> 28);
    *buf++ = (byte)((val >> 21) & 0x7F);
    *buf++ = (byte)((val >> 14) & 0x7F);
    *buf++ = (byte)((val >> 7) & 0x7F);
    *buf = (byte)(val & 0x7F);
    len = 5;
  }

  /* High-bit on means "last byte in the encoded integer". */
  *buf |= 0x80;               /* ★★ 末字节高位置 1 = 终止标志 */

  return (len);
}

/** Decode and return the integer that was encoded using our VLC scheme.
 @return value decoded */
static inline ulint fts_decode_vlc(
    byte **ptr) /* in: ptr to decode from, this ptr is
                incremented by the number of bytes decoded */
{
  ulint val = 0;

  for (;;) {
    byte b = **ptr;

    ++*ptr;
    val |= (b & 0x7F);        /* ★ 每字节贡献 7 位有效数据 */

    /* High-bit on means "last byte in the encoded integer". */
    if (b & 0x80) {
      break;
    } else {
      val <<= 7;
    }
  }

  return (val);
}
```
> ★ 为什么终止符可以是 `0x00`：**VLC 编码的 0 是 `0x80`（低位 0 + 高位 1）**，与 `0x00` 天然可区分。

**★★★ ilist 编码主函数**（`fts0fts.cc:1052`，★ 编码就内联在这里，**不存在** `fts_encode_ilist`）：

```c
/** Add the given doc_id/word positions to the given node's ilist. */
void fts_cache_node_add_positions(
    fts_cache_t *cache,     /*!< in: cache */
    fts_node_t *node,       /*!< in: word node */
    doc_id_t doc_id,        /*!< in: doc id */
    ib_vector_t *positions) /*!< in: fts_token_t::positions */
{
  ...
  ut_ad(doc_id >= node->last_doc_id);   /* ★ ilist 内 doc_id 必须单调递增 */

  /* Calculate the space required to store the ilist. */
  doc_id_delta = (ulint)(doc_id - node->last_doc_id);
  enc_len = fts_get_encoded_len(doc_id_delta);

  last_pos = 0;
  for (i = 0; i < ib_vector_size(positions); i++) {
    ulint pos = *(static_cast<ulint *>(ib_vector_get(positions, i)));

    ut_ad(last_pos == 0 || pos > last_pos);

    enc_len += fts_get_encoded_len(pos - last_pos);  /* ★ 位置也是 delta 编码 */
    last_pos = pos;
  }

  /* The 0x00 byte at the end of the token positions list. */
  enc_len++;                            /* ★ 每段末尾 1 字节 0x00 */

  if ((node->ilist_size_alloc - node->ilist_size) >= enc_len) {
    ilist = nullptr;
    ptr = node->ilist + node->ilist_size;
  } else {
    ulint new_size = node->ilist_size + enc_len;

    /* Over-reserve space by a fixed size for small lengths and
    by 20% for lengths >= 48 bytes. */
    if (new_size < 16) {
      new_size = 16;
    } else if (new_size < 32) {
      new_size = 32;
    } else if (new_size < 48) {
      new_size = 48;
    } else {
      new_size = (ulint)(1.2 * new_size);   /* ★ >=48 字节后按 1.2 倍扩容 */
    }
    ...
    cache->total_size += new_size;          /* ★ 计入 cache 总量，SYNC 阈值看它 */
  }

  ptr_start = ptr;

  /* Encode the new fragment. */
  ptr += fts_encode_int(doc_id_delta, ptr); /* ① doc_id delta */

  last_pos = 0;
  for (i = 0; i < ib_vector_size(positions); i++) {
    ulint pos = *(static_cast<ulint *>(ib_vector_get(positions, i)));
    ptr += fts_encode_int(pos - last_pos, ptr);  /* ② 依次 position delta */
    last_pos = pos;
  }

  *ptr++ = 0;                              /* ③ 0x00 终止符 */

  ut_a(enc_len == (ulint)(ptr - ptr_start));
  ...
  node->ilist_size += enc_len;

  if (node->first_doc_id == FTS_NULL_DOC_ID) {
    node->first_doc_id = doc_id;
  }

  node->last_doc_id = doc_id;              /* ★ 下一个 doc 的 delta 基准 */
  ++node->doc_count;
}
```

**★ 何时开一个新 node（新行）**（`fts_cache_add_doc` `fts0fts.cc:1194`，★ 四个条件）：

```c
    if (fts_node == nullptr || fts_node->synced ||
        fts_node->ilist_size > FTS_ILIST_MAX_SIZE ||     /* ★ 超 64KB → 新 node */
        doc_id < fts_node->last_doc_id) {                /* ★ doc_id 回退 → 也必须新开 */
      fts_node =
          static_cast<fts_node_t *>(ib_vector_push(word->nodes, nullptr));
      memset(fts_node, 0x0, sizeof(*fts_node));
      cache->total_size += sizeof(*fts_node);
    }
```
```c
/* storage/innobase/include/fts0priv.h:73 */
/** Approx. upper limit of ilist length in bytes. */
#define FTS_ILIST_MAX_SIZE (64 * 1024)
```
> ★ `doc_id < fts_node->last_doc_id` 这一条很关键：**并发/乱序写入导致 doc_id 回退时也要切新 node**，因为 delta 编码要求单调递增。

**★ 唯一的完整"读"侧样板**（`i_s.cc:2632`，★ `INNODB_FT_INDEX_TABLE` 就是靠它把 ilist 展开成人能看的行）：

```c
    /* Decrypt the ilist, and display Dod ID and word position */
    for (ulint i = 0; i < ib_vector_size(word->nodes); i++) {
      fts_node_t *node;
      byte *ptr;
      ulint decoded = 0;
      doc_id_t doc_id = 0;            /* ★ 跨 doc 段的累加器 */

      node = static_cast<fts_node_t *>(ib_vector_get(word->nodes, i));
      ptr = node->ilist;

      while (decoded < node->ilist_size) {
        ulint pos = fts_decode_vlc(&ptr);
        doc_id += pos;                /* ① doc_id = 累加 delta */

        /* Get position info */
        while (*ptr) {                /* ② 遇 0x00 表示该 doc 位置列表结束 */
          pos = fts_decode_vlc(&ptr);
          ...
        }
        ++ptr;                        /* ③ 跳过 0x00 终止符 */
        decoded = ptr - (byte *)node->ilist;
      }
    }
```

**★ 落盘**（`fts_write_node` `fts0fts.cc:3952`，★ 辅助表访问用的是 InnoDB 内部 SQL）：

```c
  pars_info_bind_varchar_literal(info, "token", word->f_str, word->f_len);
  fts_write_doc_id((byte *)&first_doc_id, node->first_doc_id);
  fts_bind_doc_id(info, "first_doc_id", &first_doc_id);
  fts_write_doc_id((byte *)&last_doc_id, node->last_doc_id);
  fts_bind_doc_id(info, "last_doc_id", &last_doc_id);
  mach_write_to_4((byte *)&doc_count, node->doc_count);
  pars_info_bind_int4_literal(info, "doc_count", (const uint32_t *)&doc_count);

  /* Set copy_name to false since it's a static. */
  pars_info_bind_literal(info, "ilist", node->ilist, node->ilist_size,
                         DATA_BLOB, DATA_BINARY_TYPE);   /* ★ ilist 整段作 BLOB */
  ...
    *graph = fts_parse_sql(fts_table, info,
                           "BEGIN\n"
                           "INSERT INTO $index_table_name VALUES"
                           " (:token, :first_doc_id,"
                           "  :last_doc_id, :doc_count, :ilist);");
```

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
struct fts_cache_t {                      /* fts0types.h:144-208，逐字段按源码顺序 */
  rw_lock_t lock;                         /* 保护整个 cache */
  rw_lock_t init_lock;                    /* 仅保护 get_docs 的惰性创建（latch level 不同） */
  ib_mutex_t optimize_lock;               /* OPTIMIZE 用 */
  ib_mutex_t deleted_lock;                /* 保护 deleted_doc_ids / deleted / added */
  ib_mutex_t doc_id_lock;                 /* 保护 next_doc_id */
  ib_vector_t *deleted_doc_ids;           /* ★ 8.0.39 中没有任何写入点（见 §9.1） */
  ib_vector_t *indexes;                   /* 每个 FTS 索引一个 fts_index_cache_t */
  ib_vector_t *get_docs;                  /* 每个 FTS 索引一个 fts_get_doc_t */
  ulint total_size;                       /* 所有 node 的 ilist 总字节，SYNC 阈值依据 */
  uint64_t total_size_before_sync;        /* 上次发 SYNC 请求时的 total_size */
  fts_sync_t *sync;                       /* sync 状态（trx / event / in_progress …） */
  ib_alloc_t *sync_heap, *self_heap;
  doc_id_t next_doc_id;                   /* 内存计数器，++ 分配 */
  doc_id_t synced_doc_id;                 /* 已 sync 到 CONFIG 的水位 */
  doc_id_t first_doc_id;                  /* 表打开后第一个 doc id，0 = 未初始化 */
  ulint deleted;                          /* 上次 optimize 以来删除数 */
  ulint added;                            /* 上次 optimize 以来新增数 */
  fts_stopword_t stopword_info;
  mem_heap_t *cache_heap;
};
```

层次关系：

```
fts_cache_t
 └── indexes : ib_vector_t<fts_index_cache_t>
       ├── index     : dict_index_t*
       ├── words     : ib_rbt_t*   key = fts_string_t, value = fts_tokenizer_word_t
       ├── doc_stats : ib_vector_t<fts_doc_stats_t>   (doc_id, word_count)
       └── ins_graph[6] / sel_graph[6] : 6 张 aux 表的预编译 query graph

fts_tokenizer_word_t  { text : fts_string_t; nodes : ib_vector_t<fts_node_t> }
fts_node_t            { first_doc_id, last_doc_id, ilist(byte*), doc_count, ilist_size, synced }
```

`words` 是 InnoDB 自研红黑树（`ib_rbt_t`，比较函数 `innobase_fts_text_cmp`），因此 `rbt_first()` + `rbt_next()` 的中序遍历天然是**按 word 字典序升序**——SYNC 正是靠这一点做有序落盘。

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

SYNC 做的事（`fts_sync` `fts0fts.cc:4370`）：X 锁 cache → 等 `in_progress` → `fts_sync_begin` → 逐索引 `fts_sync_write_words`（**中序遍历 words 红黑树**，天然有序）→ 对每个未 sync 的 node 调 `fts_write_node` 执行 `INSERT INTO fts_*_index_N`（`fts0fts.cc:3999`）→ `fts_sync_commit` 更新 CONFIG 的 `synced_doc_id` 并一次提交 → 清空并重建 cache。

> **`fts_sync_index` 是 3 行薄封装**（`fts0fts.cc:4204` → `fts_sync_write_words` 4218），真正的遍历+落盘在 `fts_sync_write_words`（`:4071`）。另外它会在写盘中间**临时释放 cache 锁**（`:4137-4139`），避免长持锁；但若 `total_size > fts_max_cache_size`，`fts_sync` 会置 `sync->unlock_cache = false`（`:4405`），不再放锁——因为此时 sync 已经追不上写入速度了。

> **`fts_need_sync` 是全局变量**（`fts0fts.cc:78`，`bool`），不是函数；真正的判定函数是 `fts_is_sync_needed()`（`fts0opt.cc:2700`）。名字太像，容易写错。

### 6.5 fts_trx：事务级记账与状态合并

三个结构（全在 `fts0fts.h`）：

```c
struct fts_trx_t {                 /* :227 —— 挂在 trx->fts_trx */
  trx_t *trx;
  ib_vector_t *savepoints;         /* 活动 savepoint 栈，至少 1 个（隐式） */
  ib_vector_t *last_stmt;          /* 语句级，供 statement rollback 用 */
  mem_heap_t *heap;
};
struct fts_trx_table_t {           /* :248 —— 每表一个 */
  dict_table_t *table;
  fts_trx_t *fts_trx;
  ib_rbt_t *rows;                  /* 按 doc_id 索引，cell = fts_trx_row_t */
  fts_doc_ids_t *added_doc_ids;
  que_t *docs_added_graph;
};
struct fts_trx_row_t {             /* :264 —— 注意：没有 fts_trx_op_t 这个名字 */
  doc_id_t doc_id;
  fts_row_state state;             /* FTS_INSERT / FTS_MODIFY / FTS_DELETE / FTS_NOTHING / FTS_INVALID */
  ib_vector_t *fts_indexes;        /* NULL = 全部索引 */
};
```

`fts_trx_add_op`（`fts0fts.cc:2637`）会把**同一个 op 同时写两份**：事务级 `savepoints` 与语句级 `last_stmt`——后者专门供语句回滚（`fts_savepoint_rollback_last_stmt`）抵消。

**状态合并**（`fts_trx_table_add_op` `:2589` → `fts_trx_row_get_new_state` `:2382`，迁移表在 `:2433-2438`）：

| 已有 \ 新 op | I | M | D |
|---|---|---|---|
| **I** | INVALID | INSERT | NOTHING |
| **M** | INVALID | MODIFY | DELETE |
| **D** | **MODIFY** | INVALID | INVALID |

即只有"**同一个 doc_id** 上先 D 后 I"才会合成 `FTS_MODIFY`，其余按表执行（`I+D → FTS_NOTHING` 表示这条记录根本不用处理）。

> **★ 常见误解：以为 UPDATE = delete + insert = FTS_MODIFY。不是。**
> `row_fts_do_update`（`row0mysql.cc:1851-1855`）产生的是 **old_doc_id → D** 和 **new_doc_id → I** 两条**不同 key** 的记录——因为 `fts_update_doc_id`（`ha_innodb.cc:9727`）给更新后的行分配了一个**全新的 doc_id**。状态机根本碰不到面，所以 UPDATE 提交时是两个独立 op（先 `fts_delete` 再 `fts_add`）。只有"用户显式建了 `FTS_DOC_ID` 列、删掉一行再插回同一个 DOC_ID"这种场景才会真的出现 `FTS_MODIFY`。
> 顺带一提：语句回滚里 `FTS_MODIFY` 是 **no-op**，源码 `fts_undo_last_stmt`（`:5667`）留着 `/* FIXME: Check if FTS_MODIFY need to be addressed */`。

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

- 删除时**不碰 INDEX 表**，只把 doc_id 写进 `DELETED` 辅助表（由独立后台事务提交，持久）。`doc_id == 0` 直接返回，不记墓碑。
- 查询时把 **DELETED + DELETED_CACHE + `cache->deleted_doc_ids`** 三处合并、排序，再用 `fts_bsearch` 二分过滤（`fts0que.cc:3684-3709`，过滤点散落在 `fts_query_union_doc_id` :687、`fts_query_remove_doc_id` :715、`fts_query_change_ranking` :738 等）。
- UPDATE 提交时是两个独立 op（见 §6.5 的纠正）：**不是**合并成 `FTS_MODIFY`。`fts_modify`（`fts0fts.cc:3137`）= `fts_delete` + `fts_add`，只在真的出现 `FTS_MODIFY` 状态时走。

> **★ `cache->deleted_doc_ids` 在 8.0.39 是死代码**：全仓库对这个 vector 只有"创建（`fts_cache_init` :510）/ 清空（`fts_cache_clear` :954）/ 读取（`fts_sync_add_deleted_cache` :4282、`fts_cache_append_deleted_doc_ids` :5289）"，**没有任何 `ib_vector_push` 写入点**。于是：
> - 第 ③ 路过滤永远是空集；
> - `fts_sync_add_deleted_cache()` 在实际运行中永远走不到（`ib_vector_size() == 0`）；
> - **DELETED_CACHE 表在 8.0.39 里基本是空的**，查询端仍读它只是为了兼容从旧版本升级上来的数据。
>
> 所以"删除会同时写 DELETED 和内存 cache"这类说法在当前版本是过时的。

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

**并行度：OPTIMIZE 全程单线程，且全局只有一个 optimize 线程。**

证据在 `fts_optimize_init()`（`fts0opt.cc:2948`）：

```c
ut_a(fts_optimize_wq == nullptr);              // 只允许一个
...
srv_threads.m_fts_optimize = os_thread_create(fts_optimize_thread_key, 0,
                                              fts_optimize_thread, fts_optimize_wq);
```

源码注释直接写着 "For now we only support one optimize thread."。线程内部也是串行的：要么处理一条队列消息，要么 `fts_optimize_table_bk()` 处理一个 slot，没有 worker pool；对 6 张分片表也是顺序 for 循环。

> **`innodb_ft_sort_pll_degree`（默认 2）与 OPTIMIZE 完全无关**。它绑定的是 `ddl::fts_parser_threads`，全仓库只出现在 `ddl/ddl0fts.cc`、`ddl/ddl0ctx.cc`、`ha_innodb.cc` 三处，唯一真正的并行线程创建是 `FTS::start_parse_threads()`（`ddl0fts.cc:1539`），由 `ddl/ddl0builder.cc:720` 在**建索引时**调用。即它只作用于 `CREATE FULLTEXT INDEX` / `ALTER TABLE ADD FULLTEXT` / rebuild 时的并行分词与归并。

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

为什么要分遍？因为布尔语义要求"`+` 是交集、`-` 是差集"，一次性遍历就会把并集和交集混在一起。三种 pass 定义在 `fts0ast.cc:43`：

```c
enum fts_ast_visit_pass_t {
  FTS_PASS_FIRST,   /* 处理除 FTS_EXIST / FTS_IGNORE 之外的算子 */
  FTS_PASS_EXIST,   /* 处理 FTS_EXIST */
  FTS_PASS_IGNORE   /* 处理 FTS_IGNORE */
};
```

> **精确化**：不是"整棵树走三遍"，而是**"每一层内先跑 first pass，遇到 `+`/`-` 节点就地改成 `FTS_EXIST_SKIP` / `FTS_IGNORE_SKIP` 并跳过；本层普通节点处理完后，再对带 skip 标记的子 list 做 exist、ignore 两轮 revisit"**，且这个过程是**递归下探**的（`fts0ast.cc:564-646`）。另外 `FTS_AST_TEXT`（短语）在非 first pass 里直接 break（`:568-570`）。

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

1. 遍历 `result->rankings_by_rank`（按 rank 排序的红黑树）
2. 用 `innobase_fts_create_doc_id_key`（`:11059`）构造 doc_id 搜索 tuple（注意 `mach_write_to_8` 转大端存储序）
3. 切到 `FTS_DOC_ID_INDEX`，`row_search_for_mysql`（`PAGE_CUR_GE`）回聚簇索引取整行

三个补充点：

- **`rankings_by_rank` 是惰性的**：查询阶段只建 `rankings_by_id`；`rankings_by_rank` 直到 `ft_read()` 第一次被调用时才由 `fts_query_sort_result_on_rank()`（`fts0que.cc:3850`，调用点 `ha_innodb.cc:11126`）构建。这个"推迟到首次读取"的设计与 `FullTextSearchIterator` 的惰性风格一致。
- **确实有"覆盖索引"式优化**：`read_just_key == true` 时**完全不回表**（`ha_innodb.cc:11146-11153`），直接从 ranking 拿 doc_id。由 `HA_EXTRA_KEYREAD` 设置。
- **跳过已删行**：回表若返回 `DB_RECORD_NOT_FOUND`（该 doc 已被删但墓碑还没生效），`ft_read` 会 `goto next_record` 继续下一个（`:11188`）。

> 另外注意一个与直觉相反的顺序：**短语搜索是先查磁盘、后查内存缓存**（`fts_query_phrase_search` `:2575` 磁盘、`:2587` 缓存），而 union/intersect/difference 是**先缓存后磁盘**（`fts_query_union` `:1308` 缓存、`:1316` 磁盘）。原因在于短语需要按位置归并，缓存里的 node 未必完整，实现上选择了先铺满磁盘结果。

**★ `ft_read()` 回表主路径**（`ha_innodb.cc:11095`，★ 精简版保留关键分支）：

```cpp
int ha_innobase::ft_read(uchar *buf) {
  ...
  result = reinterpret_cast<NEW_FT_INFO *>(ft_handler)->ft_result;

  if (result->current == nullptr) {
    /* This is the case where the FTS query did not
    contain and matching documents. */
    if (result->rankings_by_id != nullptr) {
      /* Now that we have the complete result, we
      need to sort the document ids on their rank
      calculation. */

      fts_query_sort_result_on_rank(result);       /* ★★ 惰性排序：首次 ft_read 才建树 */

      result->current =
          const_cast<ib_rbt_node_t *>(rbt_first(result->rankings_by_rank));
    } else {
      ut_a(result->current == nullptr);
    }
  } else {
    result->current = const_cast<ib_rbt_node_t *>(
        rbt_next(result->rankings_by_rank, result->current));
  }

next_record:

  if (result->current != nullptr) {
    doc_id_t search_doc_id;
    dtuple_t *tuple = m_prebuilt->search_tuple;

    /* If we only need information from result we can return
       without fetching the table row */
    if (ft_prebuilt->read_just_key) {              /* ★ 覆盖索引式快路径 */
      const fts_ranking_t *ranking = rbt_value(fts_ranking_t, result->current);
      m_prebuilt->fts_doc_id = ranking->doc_id;
      if (m_prebuilt->fts_doc_id_in_read_set) {
        innobase_fts_store_docid(table, ranking->doc_id);
      }
      return (0);                                  /* ★ 完全不回表 */
    }

    dict_index_t *index;
    index = m_prebuilt->table->fts_doc_id_index;   /* ★ 切到 FTS_DOC_ID_INDEX */
    ut_a(index != nullptr);
    m_prebuilt->index = index;

    fts_ranking_t *ranking = rbt_value(fts_ranking_t, result->current);
    search_doc_id = ranking->doc_id;

    innobase_fts_create_doc_id_key(tuple, index, &search_doc_id);

    auto ret = innobase_srv_conc_enter_innodb(m_prebuilt);
    if (ret == DB_SUCCESS) {
      ret = row_search_for_mysql((byte *)buf, PAGE_CUR_GE, m_prebuilt,
                                 ROW_SEL_EXACT, 0);   /* ★★ doc_id → 聚簇索引行 */
      innobase_srv_conc_exit_innodb(m_prebuilt);
    }

    switch (ret) {
      case DB_SUCCESS:
        error = 0;
        break;
      case DB_RECORD_NOT_FOUND:                    /* ★ 已删但墓碑未生效 */
        result->current = const_cast<ib_rbt_node_t *>(
            rbt_next(result->rankings_by_rank, result->current));
        if (!result->current) {
          error = HA_ERR_END_OF_FILE;
        } else {
          goto next_record;                        /* ★ 跳下一个继续 */
        }
        break;
      ...
```

**★ 构造 doc_id 搜索键**（`ha_innodb.cc:11056`，★ 注意字节序转换）：

```cpp
static void innobase_fts_create_doc_id_key(
    dtuple_t *tuple, const dict_index_t *index, doc_id_t *doc_id) {
  doc_id_t temp_doc_id;
  dfield_t *dfield = dtuple_get_nth_field(tuple, 0);

  ut_a(dict_index_get_n_unique(index) == 1);
  dtuple_set_n_fields(tuple, index->n_fields);
  dict_index_copy_types(tuple, index, index->n_fields);

#ifdef UNIV_DEBUG
  /* The unique Doc ID field should be an eight-bytes integer */
  dict_field_t *field = index->get_field(0);
  ut_a(field->col->mtype == DATA_INT);
  ut_ad(sizeof(*doc_id) == field->fixed_len);
  ut_ad(!strcmp(index->name, FTS_DOC_ID_INDEX_NAME));
#endif

  /* Convert to storage byte order */
  mach_write_to_8(reinterpret_cast<byte *>(&temp_doc_id), *doc_id);
  *doc_id = temp_doc_id;
  dfield_set_data(dfield, doc_id, sizeof(*doc_id));

  dtuple_set_n_fields_cmp(tuple, 1);

  for (ulint i = 1; i < index->n_fields; i++) {
    dfield = dtuple_get_nth_field(tuple, i);
    dfield_set_null(dfield);
  }
}
```

**★ `NEW_FT_INFO` 与两张虚函数表**（`ha_innodb.h:690`、`ha_innodb.cc:590`）：

```cpp
/** Structure Returned by ha_innobase::ft_init_ext() */
typedef struct new_ft_info {
  struct _ft_vft *please;
  struct _ft_vft_ext *could_you;
  row_prebuilt_t *ft_prebuilt;
  fts_result_t *ft_result;
} NEW_FT_INFO;
```
```cpp
/* Call back function array defined by MySQL and used to
retrieve FTS results. */
const struct _ft_vft ft_vft_result = {nullptr, innobase_fts_find_ranking,
                                      innobase_fts_close_ranking,
                                      innobase_fts_retrieve_ranking, nullptr};
                    /* ★ read_next 与 reinit_search 为 nullptr —— InnoDB 不通过
                       _ft_vft 取行，而是由 FullTextSearchIterator 直接调 ft_read() */
```
```cpp
/** @return version of the extended FTS API */
static uint innobase_fts_get_version() {
  return (2);
}

/** @return Which part of the extended FTS API is supported */
static ulonglong innobase_fts_flags() {
  return (FTS_ORDERED_RESULT | FTS_DOCID_IN_RESULT);
}
```
> ★★ `FTS_ORDERED_RESULT`：告诉 server"结果已按 rank 降序"，**server 可据此省掉 filesort** —— 这是 §2.8「filesort 的 FTS 特例」的接口依据。

**★ 分数如何回到 SQL 层**（`ha_innodb.cc:21575`，★ `MATCH` 最终返回的那个 float）：

```cpp
static float innobase_fts_retrieve_ranking(FT_INFO *fts_hdl) {
  fts_result_t *result;
  row_prebuilt_t *ft_prebuilt [[maybe_unused]];

  result = reinterpret_cast<NEW_FT_INFO *>(fts_hdl)->ft_result;
  ft_prebuilt = reinterpret_cast<NEW_FT_INFO *>(fts_hdl)->ft_prebuilt;

  fts_ranking_t *ranking = rbt_value(fts_ranking_t, result->current);
  assert(ft_prebuilt->fts_doc_id == ranking->doc_id);
                      /* ★★ 两层之间唯一的"游标对齐断言"：doc_id 必须相等 */

  return (ranking->rank);   /* ★ 未做任何归一化/clamp */
}
```

---

## 十一、相关性排序（rank）算法

### 11.0 先理解 BM25：InnoDB 缺了哪两块

**BM25（Best Matching 25）** 出自 Okapi 系统（Robertson 等，1990s），是概率排序原理（Robertson & Spärck Jones, 1976）的工程落地，现在是 **Lucene / Elasticsearch / Solr / Xapian 的默认打分**。

$$\operatorname{score}(d,q) \;=\; \sum_{t \in q} \underbrace{\operatorname{IDF}(t)}_{\text{词有多罕见}} \;\times\; \underbrace{\frac{\operatorname{tf}(t,d)\,(k_1+1)}{\operatorname{tf}(t,d) + k_1\!\left(1 - b + b\,\frac{\mathrm{dl}}{\mathrm{avgdl}}\right)}}_{\text{词频证据（饱和 + 长度归一）}}$$

$$\operatorname{IDF}(t) = \ln\!\left(1 + \frac{N - \operatorname{df}(t) + 0.5}{\operatorname{df}(t) + 0.5}\right)$$

它比朴素的 `tf × idf` 多出来的，就是**两个参数**：

| 参数 | 典型值 | 作用 | 直觉 |
|---|---|---|---|
| **`k1`** | `1.2`（0.9~2.0） | **tf 饱和** | 一个词出现 100 次 ≠ 重要 100 倍；边际收益递减，上限是 `k1 + 1` |
| **`b`** | `0.75`（0~1） | **长度归一化** | 长文档天然更容易命中更多词，要按长度打折；`b=0` 不惩罚，`b=1` 完全按长度归一 |

逐项看：

- **IDF 项**：罕见词权重高。Lucene 用 `ln(1 + ...)` 是为了保证 IDF 恒 ≥ 0（避免 `df > N/2` 时出现负权重）。MySQL 用 `log10(N/df)`，且 `df == N` 时退化成 `log10(1.0001)` 这个"拍脑袋的极小值"——两边都做了兜底，但 MySQL 的兜底结果是**分数≈0**，会出现"明明匹配了却排不上"。
- **tf 饱和（`k1`）**：`tf→∞` 时该项趋于 `k1+1`。含义是词频的证据价值**次线性**，堆砌同一个词不能无限加分。
- **长度归一化（`b`）**：分母里的 `1 − b + b·dl/avgdl`，`dl > avgdl` 时放大分母（惩罚长文），`dl < avgdl` 时缩小（奖励短文）。

**数值对比**（`N=1000`、`df=100`、`avgdl=300`、`k1=1.2`、`b=0.75`；MySQL 的 idf = `log10(1000/100) = 1.0`）：

| 文档 | tf | BM25 的 tf 分量 | BM25 总分 | MySQL `tf × idf²` |
|---|---|---|---|---|
| 短文档 `dl=100` | 5 | `5·2.2 / (5 + 1.2·(0.25 + 0.25))` = **1.96** | 4.51 | **5.0** |
| 中等 `dl=500` | 5 | `11 / (5 + 1.2·1.5)` = **1.62** | 3.72 | **5.0** |
| 长文档 `dl=3000` | 5 | `11 / (5 + 1.2·7.75)` = **0.77** | 1.77 | **5.0** |

差异一眼可见：**MySQL 完全不看文档长度**，三篇同分；BM25 下短文档是长文档的 **2.5 倍**。

再看 **tf 饱和**（固定 `dl = avgdl`）：`tf=1→0.79`、`2→1.16`、`5→1.62`、`10→1.86`、`100→2.16`，趋于 `k1+1 = 2.2`；而 MySQL 是线性的 `1/2/5/10/100`。⇒ **堆砌关键词在 MySQL 里是有效的作弊手段，在 BM25 里基本无效**。

**InnoDB 缺的正是 `k1` 和 `b` 这两块**。还有第三处更隐蔽的问题（`fts0que.cc:3678`）：

```c
query.total_docs = dict_table_get_n_rows(index->table);
```

`total_docs` 用的是 **InnoDB 的统计估算行数**，不是精确文档数——连 IDF 的分母都是毛估的。三者叠加的结论是：**MySQL 的 MATCH 分数只能在同一条查询内做相对排序**，不能跨查询比较，更不能当绝对阈值用。

> 另有一处死代码：`fts_query_total_docs_containing_term`（`fts0que.cc:2078`）看起来是用来精确统计 df 的，但**全仓库无调用者**——`df` 实际只取 INDEX 表的 `doc_count` 列（即落盘 node 的记录数，未包含 cache 中未 sync 的部分）。

**相关性排序的算法家族**（BM25 只是其中一支）：

| 家族 | 代表 | 核心思想 | 谁在用 |
|---|---|---|---|
| 布尔模型 | MySQL 布尔模式、早期 IR | 只判命中，不排序 | MySQL `IN BOOLEAN MODE` |
| **向量空间模型 + TF-IDF** | Salton 1975，cosine 相似度 | 文档与查询都表示成向量，夹角越小越相关 | **MySQL InnoDB FTS（简化版 `tf × idf²`）** |
| **概率模型 / BM25** | Robertson & Spärck Jones 1976 → Okapi BM25 | 估计"相关 vs 不相关"的对数几率；tf 饱和 + 长度归一化 | **Lucene / ES / Solr / Xapian 默认** |
| 语言模型 LM | Ponte-Croft 1998；QL / Jelinek-Mercer / Dirichlet 平滑 | 估计"由该文档生成该查询"的概率 | Indri、早期 Lucene |
| DFR（Divergence from Randomness） | PL2、InL2、DFI | 用词频偏离随机分布的程度衡量信息量 | Terrier |
| 学习排序 LTR | RankNet / LambdaMART、XGBoost `rank:ndcg` | 把 BM25 分、点击、PageRank 等当特征训练排序模型 | ES LTR 插件、商业搜索引擎 |
| 稠密 / 神经检索 | DPR、ANCE、ColBERT、BGE | 双塔 embedding + 向量近邻；常与 BM25 混合（RRF 融合） | 现代 RAG、向量数据库 |

工程变体还有 **BM25F**（多字段加权，ES 的 `combined_fields` 是简化版）、**BM25+**（Lv & Zhai 2011，修长文档被过度惩罚）、pivoted length normalization（TF-IDF 侧的长度归一化变体）。PostgreSQL 的 `ts_rank` 则提供若干可选的**归一化标志**（按文档长度、唯一词数等打折），可以按需组合——比 MySQL"写死一个公式"灵活得多。

**MySQL 想用 BM25 怎么办？** 社区版 8.0.39 **没有任何实现**（全仓库搜 `bm25` / `Okapi` 零命中，没有插件点、没有 hook）。可选路径：

1. **外部引擎承担检索**（ES / Solr），MySQL 只存正排数据；
2. **读出 MATCH 分数自己重排**——但**拿不到 tf**（MySQL 只暴露最终 rank，不暴露词频与文档长度），所以只能做"按文档长度粗粒度惩罚"这类近似修正；
3. 应用层双写，把文本同步到支持 BM25 的引擎。

> 实践建议：既然 MySQL 的分数没有长度归一化，**在应用层补一个"按文本长度打折"的因子**（例如 `score / log(1 + len/avg_len)`）就能缓解大部分"长文本霸榜"的问题——这是成本最低的修正。

### 11.1 结论：不是 BM25，是 tf × idf²

全仓库没有 `bm25` / `k1` / `b` / 文档长度归一化。InnoDB 的公式是：

$$\operatorname{rank}(d,q) \;=\; \sum_{t \in q \cap d} \underbrace{\operatorname{tf}(t,d)}_{\text{原始词频，不饱和}} \;\times\; \underbrace{\operatorname{idf}(t)^2}_{\text{IDF 被乘两次}}$$

$$\operatorname{idf}(t) \;=\; \log_{10}\!\left(\frac{N}{\operatorname{df}(t)}\right)$$

其中：

- `N` = `dict_table_get_n_rows(index->table)` —— **InnoDB 的统计行数估计值**，不是精确值（`fts0que.cc:3678`）
- `df(t)` = INDEX 表的 `doc_count` 列
- **没有**文档长度归一化、没有余弦归一化、没有 `tf/(tf+k1)` 饱和

**★ 打分函数的真名核查**：真名是 **`fts_query_calculate_ranking()`**（`fts0que.cc:3251`）；**不存在** `fts_rank_doc`（全仓库 grep 无命中）。

**★★ IDF 的完整实现**（`fts0que.cc:3214`）：

```c
/** Calculate the inverse document frequency (IDF) for all the terms. */
static void fts_query_calculate_idf(fts_query_t *query) /*!< in: Query state */
{
  const ib_rbt_node_t *node;
  uint64_t total_docs = query->total_docs;

  for (node = rbt_first(query->word_freqs); node;
       node = rbt_next(query->word_freqs, node)) {
    fts_word_freq_t *word_freq;

    word_freq = rbt_value(fts_word_freq_t, node);

    if (word_freq->doc_count > 0) {
      if (total_docs == word_freq->doc_count) {
        /* QP assume ranking > 0 if we find
        a match. Since Log10(1) = 0, we cannot
        make IDF a zero value if do find a
        word in all documents. So let's make
        it an arbitrary very small number */
        word_freq->idf = log10(1.0001);      /* ★ df == N 时的兜底，不是 0 */
      } else {
        word_freq->idf = log10(total_docs / (double)word_freq->doc_count);
      }                                      /* ★★ log10，不是 ln；无 +0.5 平滑 */

      if (fts_enable_diag_print) {
        ib::info(ER_IB_MSG_515) << "'" << word_freq->word.f_str << "' -> "
                                << query->total_docs << "/"
                                << word_freq->doc_count << " "
                                << std::setw(6) << std::setprecision(5)
                                << word_freq->idf;
      }
    }
  }
}
```
> ★ `innodb_ft_enable_diag_print=ON` 可以把每个词的 `idf` 打进 error log —— **这是唯一能直接观察打分输入的手段**。

**★★★ 打分主函数完整实现**（`fts0que.cc:3251`，★ 全篇最关键的一段）：

```c
/** Calculate the ranking of the document. */
static void fts_query_calculate_ranking(
    const fts_query_t *query, /*!< in: query state */
    fts_ranking_t *ranking)   /*!< in: Document to rank */
{
  ulint pos = 0;
  fts_string_t word;

  /* At this stage, ranking->rank should not exceed the 1.0 bound */
  ut_ad(ranking->rank <= 1.0 && ranking->rank >= -1.0);
              /* ★ 只是断言：此时 rank 仅含布尔算子的 ±1 调整，尚未乘 idf */
  ut_ad(rbt_size(query->word_map) == query->word_vector->size());

  while (fts_ranking_words_get_next(query, ranking, &pos, &word)) {
              /* ★ 遍历该文档命中的词位图 */
    int ret;
    ib_rbt_bound_t parent;
    double weight;
    fts_doc_freq_t *doc_freq;
    fts_word_freq_t *word_freq;

    ret = rbt_search(query->word_freqs, &parent, &word);
    /* It must exist. */
    ut_a(ret == 0);
    word_freq = rbt_value(fts_word_freq_t, parent.last);
                                      /* ★ 取该词的 idf 与 doc_freqs 树 */

    ret = rbt_search(word_freq->doc_freqs, &parent, &ranking->doc_id);
    /* It must exist. */
    ut_a(ret == 0);
    doc_freq = rbt_value(fts_doc_freq_t, parent.last);
                                      /* ★ tf = doc_freq->freq */

    weight = (double)doc_freq->freq * word_freq->idf;
                                      /* ① weight = tf × idf —— tf 是裸词频，无饱和 */

    ranking->rank += (fts_rank_t)(weight * word_freq->idf);
                                      /* ② 再乘一次 idf ⇒ 每 term 贡献 tf × idf² */
  }
}
```
> ★★ **整个函数没有任何文档长度项，也没有 clamp**。BM25 的两个参数在这里连影子都没有。

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

**★ tf 是怎么数出来的**（`fts_query_filter_doc_ids` `fts0que.cc:2955`，★ ilist 解码现场）：

```c
  /* Decode the ilist and add the doc ids to the query doc_id set. */
  while (decoded < len) {
    ulint freq = 0;
    ...
    ulint pos = fts_decode_vlc(&ptr);       /* doc_id 以 vlc 增量编码 */
    ...
    doc_id += pos;                          /* 累加 delta 得绝对 doc_id */

    if (calc_doc_count) {
      word_freq->doc_count++;
    }

    /* Unpack the positions within the document. */
    while (*ptr) {                          /* 遇 0x00 表示该 doc 的位置列表结束 */
      last_pos += fts_decode_vlc(&ptr);

      if (query->collect_positions) {
        ib_vector_push(match->positions, &last_pos);
      }

      ++freq;                               /* ★★ tf 就在这里数：每个 position +1 */
    }

    doc_freq = fts_query_add_doc_freq(query, doc_freqs, doc_id);

    /* Avoid duplicating frequency tally. */
    if (doc_freq->freq == 0) {
      doc_freq->freq = freq;                /* ★ tf 落盘到 fts_doc_freq_t::freq */
    }

    ++ptr;                                  /* 跳过 0x00 终止符 */
    decoded = ptr - (byte *)data;
```
> ★ tf = ilist 里该 doc 的 **position 个数**，即原始出现次数。没有 log、没有饱和。

**★ 两个关键数据结构**（`fts0que.cc:277`，★ "缺了两块"的硬证据）：

```c
/** For storing the frequency of a word/term in a document */
struct fts_doc_freq_t {
  doc_id_t doc_id; /*!< Document id */
  ulint freq;      /*!< Frequency of a word in a document */
};
                    /* ★★ 这就是 tf 的载体 —— 结构体里没有任何文档长度信息 */

/** To determine the word frequency per document. */
struct fts_word_freq_t {
  fts_string_t word;   /*!< Word for which we need the freq */
  ib_rbt_t *doc_freqs; /*!< RB Tree for storing per document word frequencies */
  uint64_t doc_count;  /*!< Total number of documents that contain this word */
  double idf;          /*!< Inverse document frequency */
};                      /* ★★ 整个结构体 = {df, idf}，没有 avgdl、没有 dl */
```
> ★★ 补充第三块差异：除了"缺两块"，InnoDB 还**多乘了一次 idf**（`idf²`）。
> ★ 另一条硬证据：`fts_query_t::total_words`（`fts0que.cc:157`）**声明后全仓库从未使用** —— 它本可作为长度归一化的基础，但没人用。

**★ BM25 逐项对照表**（每条都有源码落点）：

| BM25 项 | BM25 公式 | InnoDB 8.0.39 | 源码证据 |
|---|---|---|---|
| IDF | `ln(1+(N-df+0.5)/(df+0.5))` | `log10(N/df)`，**底数 10、无 +0.5 平滑** | `fts0que.cc:3237` |
| IDF 的幂 | 一次方 | **二次方**（连续乘两次同一 double，非 `pow`） | `fts0que.cc:3284,3286` |
| **① 长度归一化** | `1-b+b·dl/avgdl` | **完全缺失** | 无 `avgdl`/`dl`；`fts0que.cc:283-294` |
| **② tf 饱和** | `tf·(k1+1)/(tf+k1·(...))` | **完全缺失**，tf 是裸线性值 | `fts0que.cc:3026,3043,3284` |
| k1 / b 参数 | 可调 | **不存在** | 全 innobase grep 无命中 |
| 结果量纲 | 无界正分 | 无界，**不 clamp 到 [0,1]** | clamp 只在布尔算子路径 |

**★★ 一个必须纠正的说法**：常见以为"InnoDB rank 被 clamp 到 [0,1]"，**这是错的**。`fts_query_calculate_ranking` 里只有两个 `ut_ad`（断言），没有任何 clamp。clamp 只发生在**布尔算子**路径：

```c
/* fts0que.cc:754-761 —— 布尔算子的 ±1 调整后才 clamp，且范围是 [-1, 1] */
    ranking->rank += downgrade ? RANK_DOWNGRADE : RANK_UPGRADE;

    /* Allow at most 2 adjustment by RANK_DOWNGRADE (-0.5)
    and RANK_UPGRADE (0.5) */
    if (ranking->rank >= 1.0F) {
      ranking->rank = 1.0F;
    } else if (ranking->rank <= -1.0F) {
      ranking->rank = -1.0F;
    }
```
> ⚠️ **注释滞后于代码**：注释还写 `RANK_DOWNGRADE (-0.5)` / `RANK_UPGRADE (0.5)`，但 `fts0que.cc:56-57` 的实际值是 **±1.0F**。
> ★ 最终 `rank = Σ tf·idf²` 可以**远大于 1**；`fts_rank_t` 的注释声称"0..1 inclusive"是过时的。

### 11.3 布尔算子的调整与唯一的 clamp

`fts_query_change_ranking`（`fts0que.cc:733`）—— `~ / < / >` 只在这里生效：

```c
/* fts0que.cc:56-57 */
#define RANK_DOWNGRADE (-1.0F)
#define RANK_UPGRADE (1.0F)
...
ranking->rank += downgrade ? RANK_DOWNGRADE : RANK_UPGRADE;
/* Allow at most 2 adjustment by RANK_DOWNGRADE (-0.5)
   and RANK_UPGRADE (0.5) */                          /* ← 注释是陈旧的，实际是 ±1.0F */
if (ranking->rank >= 1.0F)      ranking->rank = 1.0F;
else if (ranking->rank <= -1.0F) ranking->rank = -1.0F;
```

**唯一的归一化就是 `[-1.0, +1.0]` 这个 clamp**，且它只作用于布尔算子产生的预置分（发生在打分之前）。**最终 rank 不做任何归一化**，实际值可以远大于 1——`fts0fts.h:303` 注释里写的 "Rank is between 0..1" 早已不成立（`fts_query_calculate_ranking` 里的 `ut_ad(ranking->rank <= 1.0)` 只是 debug 断言）。

> 又一处"注释比代码老"：上面那两行注释写的是 `-0.5` / `0.5`，而常量实际是 **±1.0F**（8.0 早期版本的残留）。

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
- 是 **lazy** 的，触发点共 6 处（逐个核实）：

| 触发点 | 位置 |
|---|---|
| `fts_init_doc_id` | `fts0fts.cc:4935`（未设 `DICT_TF2_FTS_ADD_DOC_ID` 时） |
| `fts_get_next_doc_id` | `:2772`（`first_doc_id == FTS_NULL_DOC_ID` 时转调 `fts_init_doc_id`，间接） |
| `fts_add_doc_by_id` | `:3587-3589` |
| `fts_add_doc_from_tuple` | `:3524`（ALTER 内部重放的旁路） |
| `ha_innobase::ft_init_ext` | `ha_innodb.cc:11008-11012` |
| `init_fts_doc_id_for_ref` | `row0mysql.cc:1918`（外键引用表） |

- 幂等：由 `fts_status |= ADDED_TABLE_SYNCED`（`:6281`）与 `first_doc_id != FTS_NULL_DOC_ID`（`:4920`）双重保护
- **启动时没有全量 FTS 修复流程** —— `srv0start.cc` 里只做 `fts_optimize_init()`（`:2529`）

### 12.4 DDL

| 操作 | 路径 |
|------|------|
| **CREATE / ADD FULLTEXT** | `handler0alter.cc:5003` `fts_create_index_tables()`（`fts0fts.cc:2331`）建 6 张 INDEX 表 → `ddl::Builder`（`ddl/ddl0builder.cc:720`）`start_parse_threads()` → `ddl::FTS`（`ddl0fts.cc`）并行分词 → 归并 → `scan_finished()`（`ddl0fts.cc:1682`）`fts_sync_table` + `fts_update_next_doc_id` |
| DROP TABLE | `row_drop_ancillary_fts_tables`（`row0mysql.cc:3668`）→ `fts_drop_tables`（`fts0fts.cc:1643`）= 5 张公共表 + 每个 FTS 索引 6 张分片表 |
| TRUNCATE | 8.0 是 drop + create，辅助表整体删除再重建（`ha_innodb.cc:14560`）。新 table_id ⇒ 新的 aux 表名，**doc_id 从 0 重新开始** |
| RENAME | `fts_rename_aux_tables`（`fts0fts.cc:1385`） |
| DROP INDEX | `fts_drop_index`（`fts0fts.cc:707`） |

> **`row0ftsort.cc` / `fts0ftsort` 在 8.0.39 已完全不存在**（全仓库 0 命中）。5.7 时代建 FTS 索引走它，8.0 起全部改由 `ddl/ddl0fts.cc` 的 `ddl::FTS` 负责——这是 8.0 把 DDL 统一到新 DDL 框架（`ddl/` 目录）的一部分。读旧资料时注意这个文件名已成历史。

### 12.5 已知限制（完整清单）

> 本章是 FTS 局限的**唯一详写处**，按"限制落在哪一层"分组。语法层的另有 §2.3，此处只给指针。

**A. 对象模型与列约束**

| 限制 | 报错 / 位置 |
|---|---|
| 列类型只限 `CHAR` / `VARCHAR` / `TEXT(BLOB)`；数值、JSON、ENUM/SET、空间类型全部被拒 | `ER_BAD_FT_COLUMN`（`sql_table.cc:4952-4958`） |
| **同一索引的各列必须同一字符集且同一排序规则**（比较的是 `CHARSET_INFO*` 指针，所以 `_general_ci` 与 `_0900_ai_ci` 也算不同） | 同上（`sql_table.cc:4957`） |
| 禁止 `binary` 字符集、禁止 `mbminlen > 1`（**ucs2 / utf16 / utf32 全部不支持**） | 同上 |
| **不支持前缀索引，且是静默忽略**——`FULLTEXT(col(10))` 不报错，前缀被丢弃、按整列索引 | `sql_table.cc:5264-5272`（注释 "fulltext indexes ignores prefixes"） |
| 不能建在虚拟生成列 / 函数索引上 | `ER_FULLTEXT_FUNCTIONAL_INDEX`（`sql_table.cc:4898`） |
| **可以为 NULL**（`ER_NULL_COLUMN_IN_INDEX` 检查只对非 FULLTEXT 索引生效） | `sql_table.cc:5111-5130` |
| 词长上限 **84 字符 / 336 字节**；短语与邻近搜索词数上限 **128** | `HA_FT_MAXBYTELEN=336`（`ft_global.h:42`）、`MAX_PROXIMITY_ITEM=128`（`fts0que.cc:60`） |
| 外键可以与 FTS **共存**（常被误认为不行），但 FTS 索引不能充当外键的支撑索引 | `sql_table.cc:5510/5702/6169/6243` 四处跳过 `HA_FULLTEXT` |

**B. 表类型**

| 限制 | 报错 / 位置 |
|---|---|
| **分区表不支持 FULLTEXT** | `ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING`（`handler0alter.cc:10117`） |
| **临时表不支持** | `ER_INNODB_NO_FT_TEMP_TABLE`（`ha_innodb.cc:13174`、`dict0dd.cc:1582`） |

**C. DDL**

| 限制 | 报错 / 位置 |
|---|---|
| **ADD / DROP FULLTEXT 是 INPLACE，但 `LOCK=SHARED`——会阻塞 DML 写**（不是 Online DDL） | `handler0alter.cc:1332-1354` 返回 `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE`，理由码 `ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FTS` |
| **不支持 `ALGORITHM=INSTANT`** | `INNOBASE_INSTANT_ALLOWED` 不含 `ADD_INDEX`（`handler0alter.cc:148-156`） |
| 一条 ALTER 只能 ADD 一个 FTS 索引 | `ER_INNODB_FT_LIMIT`（`handler0alter.cc:4479`） |
| **已有 FTS 索引的表不能再走 inplace rebuild** | `ER_INNODB_FT_LIMIT`（`handler0alter.cc:1317`，理由是重建会牵动 aux 表） |
| `FTS_DOC_ID_INDEX` 不能降序、名字必须精确 | `ha_innodb.cc:13187` |
| 多个变量是**只读**：`innodb_ft_min/max_token_size`、`ngram_token_size`——改了要重建索引 | `ha_innodb.cc:22512/22518`、`plugin_ngram.cc:250` |
| **辅助表全部登记在 8.0 数据字典里**：每表 5 张公共表 + **每个 FTS 索引** 6 张 INDEX 表。N 个索引 ≈ 5+6N 张表 ⇒ 建表/删表要逐张拿 exclusive MDL，DD 膨胀、DDL 变慢 | `dict0dd.cc:6463`、`fts0fts.cc:1663`（`fts_lock_common_tables` 逐张 acquire） |

> `ER_INNODB_FT_LIMIT` 的语义常被误读：它不是"一张表只能有一个 FTS 索引"，而是**"一条 DDL 只能 ADD 一个"** + **"已有 FTS 的表不能再 inplace rebuild"**。一张表可以有多组 FULLTEXT 索引，分多次 ADD 即可。

**D. 事务可见性 ★**

| 限制 | 说明 |
|---|---|
| **未提交的 DML 对 `MATCH` 不可见** | token 直到 `trx_commit_low → fts_commit → fts_add_doc_by_id` 才写入 `fts_cache`（`trx0trx.cc:2211`）；而 `fts_query` 用的是**另一个后台事务**（`fts0que.cc:3636`）、只读 cache + 磁盘，`fts0que.cc` 里 grep `fts_trx` **0 命中**——它完全不看未提交的 op。<br>⇒ **在一个事务里 INSERT 后立即 `MATCH` 查不到刚插入的行**（autocommit 下每条语句都提交，所以看起来"可见"） |

**E. 复制 ★**

| 限制 | 说明 |
|---|---|
| **辅助表的修改不记 binlog** | 内部 query graph 不过 server 层（`storage/innobase/fts/` grep `binlog` 0 命中）。从库的 FTS 索引是**重放行级 DML 后自己分词重建**的 |
| **主从的 stopword 表 / `ngram_token_size` / parser 插件配置若不同，`MATCH` 结果会不同** | 两端各自分词（`fts0fts.cc:220`），变量是 GLOBAL/SESSION 级（`ha_innodb.cc:1088/22101/22512`） |
| 澄清：`FTS_DOC_ID` **不随行复制也不需要** | 隐藏 doc_id 由本地 `fts_create_doc_id` 生成；aux 表根本不复制，两端 doc_id 是否相同不影响排名（排名只由 tf/idf 决定） |

**F. 空间与运维**

| 限制 | 说明 |
|---|---|
| 删除不回收空间（只写墓碑） | 后台自动回收门槛 1000 万个已删文档 + 5 分钟；否则要 `innodb_optimize_fulltext_only=ON` + `OPTIMIZE TABLE` |
| OPTIMIZE 全局单线程 | `fts_optimize_init` 只允许一个线程 |
| 结果集内存上限 | 超 `innodb_ft_result_cache_limit`（默认 2GB）报 `HA_ERR_FTS_EXCEED_RESULT_CACHE_LIMIT`（"FTS query exceeds result cache limit"），检查点 `fts0que.cc:1385/3071/3397/3443` |

**★ 几条最容易被误传的限制——逐条给源码证据**：

**① 临时表确实不支持**（`dict0dd.cc:1578` 与 `ha_innodb.cc:13169` 两处独立拦截）：

```cpp
  /* Check if there are any FTS indexes defined on this table. */
  for (uint i = 0; i < form->s->keys; i++) {
    const KEY *key = &form->key_info[i];

    if ((key->flags & HA_FULLTEXT) && is_temporary) {
      /* We don't support FTS indexes in temporary tables. */
      my_error(ER_INNODB_NO_FT_TEMP_TABLE, MYF(0));
      return true;
    }
  }
```

**② 分区表：能力位声明了，但 DDL 路径硬拦**（★ 两处矛盾值得点出）：

```cpp
/* storage/innobase/handler/ha_innopart.h:55 —— 声明了 HA_CAN_FULLTEXT */
```
```cpp
/* handler0alter.cc:10121 —— ADD_INDEX 时直接拒绝 */
      if (key->flags & HA_FULLTEXT) {
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING);
        return HA_ALTER_INPLACE_NOT_SUPPORTED;
      }
```
> ★ 即"引擎说支持、DDL 路径禁止"——不要只看能力位就下结论。

**③ 列类型四连禁**（`sql_table.cc:4951`）：

```cpp
  if (key->type == KEYTYPE_FULLTEXT) {
    if ((sql_field->sql_type != MYSQL_TYPE_STRING &&
         sql_field->sql_type != MYSQL_TYPE_VARCHAR &&
         !is_blob(sql_field->sql_type)) ||
        sql_field->charset == &my_charset_bin ||
        sql_field->charset->mbminlen > 1 ||  // ucs2 doesn't work yet
        (*ft_key_charset && sql_field->charset != *ft_key_charset)) {
      my_error(ER_BAD_FT_COLUMN, MYF(0), column->get_field_name());
      return true;
    }
    *ft_key_charset = sql_field->charset;
```
> ★ 四连禁：非 CHAR/VARCHAR/TEXT、binary、`mbminlen > 1`（ucs2/utf16/utf32 全不支持）、同一索引内字符集不一致。

**④ ngram 模式下 `innodb_ft_min_token_size` 被显式忽略**（★ 中文场景必知）：

```cpp
/* fts0fts.cc:4492 —— fts_check_token */
  if (!is_ngram) {
    if (token->f_n_char < fts_min_token_size ||
        token->f_n_char > fts_max_token_size ||
        (stopwords != nullptr && rbt_search(stopwords, &parent, token) == 0)) {
      return (false);
    } else {
      return (true);
    }
  }

  /* Check token for ngram. */
  /* We ignore fts_min_token_size when ngram */
  ut_ad(token->f_n_char > 0 && token->f_n_char <= fts_max_token_size);
```
> ★★ 所以中文搜不到单字时**改 `innodb_ft_min_token_size` 完全没用**，要改的是 `ngram_token_size`（默认 2）。

**⑤ ADD FULLTEXT 为什么要锁表**（`handler0alter.cc:1306`，★ 官方注释直说）：

```cpp
  } else if ((ha_alter_info->handler_flags & Alter_inplace_info::ADD_INDEX)) {
    /* Building a full-text index requires a lock.
    We could do without a lock if the table already contains
    an FTS_DOC_ID column, but in that case we would have
    to apply the modification log to the full-text indexes. */
    for (uint i = 0; i < ha_alter_info->index_add_count; i++) {
      const KEY *key = ...;
      if (key->flags & HA_FULLTEXT) {
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FTS);
        online = false;   /* ★ 退化为 SHARED_LOCK */
        break;
      }
    }
  }
```

**⑥ 一条 ALTER 只能加一个**（`handler0alter.cc:4479`）：

```cpp
  if (num_fts_index > 1) {
    my_error(ER_INNODB_FT_LIMIT, MYF(0));
    goto error_handled;
  }
```

**⑦ FTS_DOC_ID 与 FTS_DOC_ID_INDEX 不可删改**（`handler0alter.cc:1275`）：

```cpp
  if (m_prebuilt->table->fts && innobase_fulltext_exist(altered_table)) {
    /* FULLTEXT indexes are supposed to remain. */
    /* Disallow DROP INDEX FTS_DOC_ID_INDEX */
    for (uint i = 0; i < ha_alter_info->index_drop_count; i++) {
      if (!my_strcasecmp(system_charset_info,
                         ha_alter_info->index_drop_buffer[i]->name,
                         FTS_DOC_ID_INDEX_NAME)) {
        ha_alter_info->unsupported_reason = innobase_get_err_msg(
            ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_CHANGE_FTS);
        return HA_ALTER_INPLACE_NOT_SUPPORTED;
      }
    }
```

**⑧ 用户自建 FTS_DOC_ID 的形态约束**（`ha_innodb.cc:14865`，★ 两处）：

```cpp
    if (fts_doc_id_index) {
      switch (fts_doc_id_index->type()) {
        case dd::Index::IT_UNIQUE:
          ut_ad(fts_doc_id_index->algorithm() == dd::Index::IA_BTREE);
          if (dd_is_only_column(fts_doc_id_index, fts_doc_id)) {
            break;                    /* ★ 必须单列 UNIQUE BTREE */
          }
          [[fallthrough]];
        case dd::Index::IT_MULTIPLE:
        case dd::Index::IT_FULLTEXT:
        case dd::Index::IT_SPATIAL:
          my_error(ER_INNODB_FT_WRONG_DOCID_INDEX, MYF(0), ...);
          ...
      }
    }

    if (fts_doc_id) {
      if (fts_doc_id->type() != dd::enum_column_types::LONGLONG ||
          fts_doc_id->is_nullable() ||
          fts_doc_id->name() != FTS_DOC_ID_COL_NAME) {
        my_error(ER_INNODB_FT_WRONG_DOCID_COLUMN, MYF(0), ...);
        ...
      }
    } else {
      /* Add hidden FTS_DOC_ID column */
      dd::Column *col = dd_table->add_column();
      col->set_hidden(dd::Column::enum_hidden_type::HT_HIDDEN_SE);
      col->set_name(FTS_DOC_ID_COL_NAME);
      col->set_type(dd::enum_column_types::LONGLONG);
      col->set_nullable(false);
      col->set_unsigned(true);
    }
```
> ★ 用户列必须是 `BIGINT UNSIGNED NOT NULL` 且名字就叫 `FTS_DOC_ID`；否则 InnoDB 自动加一个 `HT_HIDDEN_SE` 隐藏列（**注意是 SE 级隐藏，与 GIPK 的 USER 级不同**）。

**⑨ ROLLUP 列不支持**（`sql_resolver.cc:371`）：

```cpp
  if (fulltext_uses_rollup_column(this)) {
    my_error(ER_FULLTEXT_WITH_ROLLUP, MYF(0));
    return true;
  }
```

**⑩ 物化子查询/派生表不能建**（`item_func.cc:7494`）：`ER_NO_FT_MATERIALIZED_SUBQUERY`。

**★ 相关错误码全表**（`share/messages_to_clients.txt`）：

| 错误码名 | 行 | 文案 |
|---|---|---|
| `ER_FT_MATCHING_KEY_NOT_FOUND` | 4157 | Can't find FULLTEXT index matching the column list |
| `ER_TABLE_CANT_HANDLE_FT` | 4488 | The used table type doesn't support FULLTEXT indexes |
| `ER_BAD_FT_COLUMN` | 5075 | Column '%-.192s' cannot be part of FULLTEXT index |
| `ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING` | 6707 | FULLTEXT index is not supported for partitioned tables. |
| `ER_TABLE_HAS_NO_FT` | 6734 | The table does not have FULLTEXT index to support this query |
| `ER_INNODB_FT_LIMIT` | 6828 | InnoDB presently supports one FULLTEXT index creation at a time |
| `ER_INNODB_NO_FT_TEMP_TABLE` | 6831 | Cannot create FULLTEXT index on temporary InnoDB table |
| `ER_INNODB_FT_WRONG_DOCID_COLUMN` | 6834 | Column '%-.192s' is of wrong type for an InnoDB FULLTEXT index |
| `ER_INNODB_FT_WRONG_DOCID_INDEX` | 6837 | Index '%-.192s' is of wrong type for an InnoDB FULLTEXT index |
| `ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FTS` | 7021 | Fulltext index creation requires a lock |
| `ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_CHANGE_FTS` | 7018 | Cannot drop or rename FTS_DOC_ID |
| `ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_HIDDEN_FTS` | 7015 | Cannot replace hidden FTS_DOC_ID with a user-visible one |
| `ER_NO_FT_MATERIALIZED_SUBQUERY` | 7214 | Cannot create FULLTEXT index on materialized subquery |
| `ER_BINLOG_UNSAFE_FULLTEXT_PLUGIN` | 7178 | Statement is unsafe because it uses a fulltext parser plugin ... |
| `ER_FULLTEXT_FUNCTIONAL_INDEX` | 8723 | Fulltext functional index is not supported. |
| `ER_FULLTEXT_WITH_ROLLUP` | 9693 | MATCH does not support ROLLUP columns. |

> ⚠️ `ER_INNODB_NO_FT_USES_PARSER` 已废弃：现名 **`OBSOLETE_ER_INNODB_NO_FT_USES_PARSER`**（`:7069`），代码中 **0 处引用** ⇒ **8.0.39 的 InnoDB 是支持 `WITH PARSER` 的**（`is_ngram` 由 parser 名判定，`ha_innodb.cc:7596`）。"InnoDB 不支持 WITH PARSER"是过时说法。

**G. 能力缺失**

| 缺失 | 证据 |
|---|---|
| **无高亮 / 摘要（snippet）** | 词位置只用于短语匹配（`fts0que.cc:3054`），无对外 API；`storage/innobase` grep `snippet\|highlight` 0 命中 |
| **无自定义打分插件** | parser 插件接口只有 `init/deinit/parse`（`fts0plugin.h:43-58`），**没有 rank 回调**；打分硬编码在 `fts0que.cc:3405/3431` |
| 无 facet / 聚合 / 拼写纠错 / 自动补全 | 见 Misc「FTS 没有实现哪些算法」 |
| 分数不可解释 | 只暴露最终 rank，拿不到 tf、文档长度 ⇒ 也无法在 SQL 层补救长度归一化 |

---

## 十三、两层的结合点

本章是本篇的价值所在——前面按层讲了两侧，这里看**两层握手的那几个接口**。

### 13.1 三个接口 + 一组协议

| 结合点 | 方向 | 载体 |
|---|---|---|
| **`ft_init_ext_with_hints`（查询）** | SQL → 引擎 | SQL 层把"搜索串 + 索引号 + hints"交给引擎，引擎返回 `FT_INFO *` |
| **`ft_init` / `ft_read`（取行）** | SQL → 引擎 | 迭代器驱动，引擎按 rank 顺序吐行（内部回表） |
| **`_ft_vft` / `_ft_vft_ext`（取值）** | 引擎 → SQL | 引擎把 vtable 挂在 `FT_INFO` 上，SQL 层通过它取 rank / doc_id / 命中数 |
| **hints（意图）** | SQL → 引擎 | `Ft_hints`：要不要排序、rank 阈值、limit 截断 |

**`FT_INFO` 的"INTERCAL style"设计**（`include/ft_global.h:72`，注释原文就写着 "INTERCAL style :-)"）：

```c
struct FT_INFO { struct _ft_vft *please; };          /* 基础：5 个函数指针 */
struct FT_INFO_EXT {                                  /* 扩展：前面 8 字节就是 FT_INFO */
  struct _ft_vft *please;
  struct _ft_vft_ext *could_you;                      /* 再 4 个函数指针 */
};
```

`FT_INFO_EXT` 的前 8 字节与 `FT_INFO` 布局相同，所以可以安全向下转型（SQL 层就是这么干的：`((FT_INFO_EXT *)ft_handler)->could_you->count_matches(...)`）。InnoDB 的 `NEW_FT_INFO`（`ha_innodb.h:691`）正是这个布局的扩展，多带 `row_prebuilt_t *` 和 `fts_result_t *`。

InnoDB 只实现了基础 5 槽中的 3 个（`ha_innodb.cc:607`）：

```c
const struct _ft_vft ft_vft_result = {nullptr,                    /* read_next */
                                      innobase_fts_find_ranking,
                                      innobase_fts_close_ranking,
                                      innobase_fts_retrieve_ranking,
                                      nullptr};                    /* reinit_search */
```

`read_next` / `reinit_search` 为 nullptr——因为 InnoDB 不需要"引擎内部推进"，取行由 SQL 层的迭代器驱动 `ft_read` 完成。这是 InnoDB 与 MyISAM 在 FT 接口用法上的分野（MyISAM 用 `read_next` 自己推进）。

### 13.2 doc_id 是两层之间唯一的"行标识"

- SQL 层：`Item_func_match::fix_fields` 把 `table->fts_doc_id_field` 加入读集（`item_func.cc:7625`）——**这是为了让引擎能拿到 doc_id 去回表**；若引擎没有 DOC_ID 列（MyISAM），则退化为把 MATCH 的每个列都加入读集，并 `covering_keys.clear_all()` 禁掉覆盖索引。
- 引擎层：查询结果里只有 doc_id，`ft_read` 用它构造 `(doc_id)` 元组，切到 `FTS_DOC_ID_INDEX` 回聚簇索引取整行。

**doc_id 因此是两层之间唯一的"行标识"**，也是为什么 8.0 要自动加那个隐藏列（第四章）。

### 13.3 打分的职责是"跨界"的

引擎算分（它需要 ilist 里的 tf 与 INDEX 表的 doc_count），SQL 层取值（`val_real()`）。这个跨界带来两个后果：

1. **filesort 必须退回 rowid 模式**（§2.8）——因为 rank 不是从 `record[0]` 读的，而是"问"handler 要的。
2. **hypergraph 下 FTS 表不能参与 hash join**（§2.4）——同理，handler 必须真实定位在行上。

### 13.4 事务边界的不对称

这是 FTS 最"不干净"的地方，值得单独点出：

```
用户事务 COMMIT
  └─ trx_commit_low
       ├─ [同事务] 用户行的 redo/undo        ← 与用户事务原子
       └─ fts_commit → fts_commit_table
            └─ trx_allocate_for_background()  ← 另起一个后台事务
                 ├─ 分词、写 fts_cache（内存）
                 ├─ INSERT INTO DELETED（删）
                 └─ fts_sql_commit()          ← 立即提交，不等用户事务
```

后果：用户事务随后若失败回滚，**FTS 侧已提交的墓碑不会回滚**；崩溃发生在两者之间时，可能重复插入 DELETED 导致 `DB_DUPLICATE_KEY`，源码（`trx0trx.cc:2213-2224`）用 `FTS-FIXME` 注释明确写着"temporarily tolerate"。

### 13.5 引擎差异：InnoDB vs MyISAM

> 按 [`feat/README.md`](README.md) 的规则，多引擎实现同一特性时不逐引擎开篇，此处只做对比。

| 维度 | InnoDB | MyISAM |
|---|---|---|
| 索引载体 | **11 张辅助表**（普通 InnoDB 表，走事务/redo） | 存在 `.MYI` 索引文件里，无事务 |
| 索引维护时机 | **延迟**：commit 时进内存 cache，后台 sync 落盘 | **同步**：行更新时 `_mi_ft_update()` 直接改（`ft_update.cc:187`） |
| 删除 | 墓碑表 + OPTIMIZE 重写 | 即时 |
| 最小词长变量 | **`innodb_ft_min_token_size`**（默认 3） | **`ft_min_word_len`**（默认 4） |
| 布尔语法 | 固定（`fts0tokenize.h:43` 静态常量，`ft_boolean_syntax` 无效） | `ft_boolean_syntax` 可调 |
| 扩展 API | 支持（`HA_CAN_FULLTEXT_EXT`）⇒ 有 hints、limit 下推、覆盖索引 | 不支持 ⇒ 无 hints；`test_if_ft_index_order` 的 `ordered_result()` 直接返回 false |
| 非索引列布尔搜索 | 不支持（MATCH 必须匹配某个 FULLTEXT 索引） | **支持**（`allows_search_on_non_indexed_columns`）⇒ `key = NO_SUCH_KEY`，用 `concat_ws(' ', cols...)` 拼串再 `find_relevance` |
| 中文 | 需 ngram / MeCab parser | 同（parser 是 server 层插件接口） |
| 并发 | 行级锁 | 表级锁 |

> 注意 `HA_CAN_FULLTEXT_EXT` 是分水岭：SQL 层多处（hints 下推、免排序、覆盖索引、`can_skip_ranking`）都以它为前提，MyISAM 全都走不到。

**★ 能力位的直接 diff**（★ 最硬的证据）：

```cpp
/* storage/innobase/handler/ha_innodb.cc:2973 —— InnoDB */
      m_int_table_flags(
          HA_NULL_IN_KEY | HA_CAN_INDEX_BLOBS | HA_CAN_SQL_HANDLER |
          HA_PRIMARY_KEY_REQUIRED_FOR_POSITION | HA_PRIMARY_KEY_IN_READ_INDEX |
          HA_BINLOG_ROW_CAPABLE | HA_CAN_GEOMETRY | HA_PARTIAL_COLUMN_READ |
          HA_TABLE_SCAN_ON_INDEX | HA_CAN_FULLTEXT | HA_CAN_FULLTEXT_EXT |
          HA_CAN_FULLTEXT_HINTS | HA_CAN_EXPORT | HA_CAN_RTREEKEYS |
          ...);
                    /* ★★ 三个 FULLTEXT 能力位全有 */
```
```cpp
/* storage/myisam/ha_myisam.cc:625 —— MyISAM */
      int_table_flags(
          HA_NULL_IN_KEY | HA_CAN_FULLTEXT | HA_CAN_SQL_HANDLER |
          HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE | HA_DUPLICATE_POS |
          HA_CAN_INDEX_BLOBS | HA_AUTO_PART_KEY | HA_FILE_BASED |
          HA_CAN_GEOMETRY | HA_NO_TRANSACTIONS | HA_CAN_BIT_FIELD |
          HA_CAN_RTREEKEYS | HA_COUNT_ROWS_INSTANT | HA_STATS_RECORDS_IS_EXACT |
          HA_CAN_REPAIR | HA_GENERATED_COLUMNS | HA_ATTACHABLE_TRX_COMPATIBLE |
          HA_SUPPORTS_DEFAULT_EXPRESSION),
                    /* ★★ 只有 HA_CAN_FULLTEXT：无 EXT、无 HINTS，且 HA_NO_TRANSACTIONS */
```
```cpp
/* sql/handler.h:305 / :416 / :441 —— ⚠️ 三者都不在 include/my_base.h */
#define HA_CAN_FULLTEXT (1 << 21)
#define HA_CAN_FULLTEXT_EXT (1LL << 39)
#define HA_CAN_FULLTEXT_HINTS (1LL << 43)
```
```cpp
/* include/my_base.h:106 —— my_base.h 里只有这两个 */
  HA_KEY_ALG_FULLTEXT = 4 /* FULLTEXT. */
/* include/my_base.h:497 */
#define HA_FULLTEXT (1 << 7)
```

**★ SQL 层按能力位分流**（`sql/item_func.h:3487`，★ 三个判断都以 EXT 为前提）：

```cpp
  bool ordered_result() {
    assert(!master);
    if (hints->get_flags() & FT_SORTED) return true;

    if ((table_ref->table->file->ha_table_flags() & HA_CAN_FULLTEXT_EXT) == 0)
      return false;        /* ★★ MyISAM → false → 优化器必须自己 filesort */

    assert(ft_handler);
    return ((FT_INFO_EXT *)ft_handler)->could_you->get_flags() &
           FTS_ORDERED_RESULT;
  }

  bool docid_in_result() {
    if ((table_ref->table->file->ha_table_flags() & HA_CAN_FULLTEXT_EXT) == 0)
      return false;
    return ((FT_INFO_EXT *)ft_handler)->could_you->get_flags() &
           FTS_DOCID_IN_RESULT;
  }
```

**★ 非索引列搜索：注释直接点名了两家的差异**（`sql/item_func.h:3651`）：

```cpp
  bool allows_search_on_non_indexed_columns(const TABLE *tr) {
    // Only Boolean search may support non_indexed columns
    if (!(flags & FT_BOOL)) return false;

    // Assume that if extended fulltext API is not supported,
    // non-indexed columns are allowed.  This will be true for MyISAM.
    if ((tr->file->ha_table_flags() & HA_CAN_FULLTEXT_EXT) == 0) return true;

    return false;          /* ★★ InnoDB → false */
  }
```
> ★ 上游注释（`item_func.h:3638`）也承认这是"hack"：*"The current hack is to assume that search on non-indexed columns are supported for engines that does not support the extended fulltext API (e.g., MyISAM), while it is not supported for other engines (e.g., InnoDB)"*。

**★ MyISAM 的 50% 阈值从哪来**（★ InnoDB 没有，这是两家的实质差异）：

```cpp
/* storage/myisam/ftdefs.h:88 —— GWS_IN_USE 用的是 GWS_PROB */
#define GWS_PROB                                                        \
  ((aio->info->state->records > doc_cnt)                                \
       ? log(((double)(aio->info->state->records - doc_cnt)) / doc_cnt) \
       : 0)
                    /* ★ df > N/2 时 (N-df)/df < 1 ⇒ log 为负 */
```
```cpp
/* storage/myisam/ft_nlq_search.cc:163 —— 负权重直接归零 */
    gweight = word->weight * GWS_IN_USE;
    if (gweight < 0 || doc_cnt > 2000000) gweight = 0;
```
> ★★ 这就是 MyISAM"50% 阈值"的**全部实现**：不是一个显式规则，而是 `GWS_PROB` 对数的副作用。**InnoDB 完全没有对应代码**（`fts0que.cc` 里 idf 恒 ≥ 0）。
> ★ 布尔模式两家都无此阈值 —— MyISAM 布尔用静态权重表 `wghts`/`nwghts`（`ft_boolean_search.cc:176`，clamp 到 ±5），与 df/N 无关。

**★ MyISAM 的索引就是 `.MYI` 里的 key，无辅助表**：

```cpp
/* storage/myisam/ftdefs.h:124 */
uint _ft_make_key(MI_INFO *, uint, uchar *, FT_WORD *, my_off_t);
uchar ft_get_word(const CHARSET_INFO *, uchar **, uchar *, FT_WORD *,
                  MYSQL_FTPARSER_BOOLEAN_INFO *);
FT_INFO *ft_init_nlq_search(MI_INFO *, uint, uchar *, uint, uint, uchar *);
FT_INFO *ft_init_boolean_search(MI_INFO *, uint, uchar *, uint,
                                const CHARSET_INFO *);
```
> ★ MyISAM 也有 `ft_nlq_get_docid` 挂在 `_ft_vft` 上，但**不属于 `FT_INFO_EXT` 扩展 API**。

**★★ 历史继承的硬证据**（★ MyISAM → InnoDB 的接口复用）：

```cpp
/* include/ft_global.h:27 —— ft_global.h 是 MyISAM 时代的产物 */
/* Written by Sergei A. Golubchik, who has a shared copyright to this code */
```
```cpp
/* storage/innobase/include/fts0fts.h:42 —— InnoDB 直接 include 它 */
#include "ft_global.h"
```
```cpp
/* storage/innobase/include/fts0fts.h:91 —— 并逐字镜像常量 */
/** Defines for FTS query mode, they have the same values as
those defined in mysql file ft_global.h */
#define FTS_NL 0
#define FTS_BOOL 1
...
```
> ★★ 结论：`ft_global.h` 里 `_ft_vft` 这套"函数指针虚表"（含那个 `INTERCAL style :-)` 的梗）是 **MyISAM FTS 定下的 SE 接口契约**，InnoDB FTS **没有另起炉灶，而是实现同一套接口**。

```cpp
/* include/ft_global.h:42 —— 接口本体 */
#define HA_FT_MAXBYTELEN 336
#define HA_FT_MAXCHARLEN (HA_FT_MAXBYTELEN / 4)
#define DEFAULT_FTB_SYNTAX "+ -><()~*:\"\"&|"

struct FT_INFO;
struct _ft_vft {
  int (*read_next)(FT_INFO *, char *);
  float (*find_relevance)(FT_INFO *, uchar *, uint);
  void (*close_search)(FT_INFO *);
  float (*get_relevance)(FT_INFO *);
  void (*reinit_search)(FT_INFO *);
};

/* Flags for extended FT API */
#define FTS_ORDERED_RESULT (1LL << 1)
#define FTS_DOCID_IN_RESULT (1LL << 2)
#define FTS_DOC_ID_COL_NAME "FTS_DOC_ID"
#define FTS_NGRAM_PARSER_NAME "ngram"

struct FT_INFO {
  struct _ft_vft *please; /* INTERCAL style :-) */
};
```

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

**★★ 真名核查（★ 三条容易写错的）**：

1. **`innodb_ft_*` 不在 `sql/sys_vars.cc`** —— 它们定义在 **`storage/innobase/handler/ha_innodb.cc`**，声明名是 `ft_xxx`，`innodb_` 前缀由**插件自动加**。
2. **`ngram_token_size` 不在 `sys_vars.cc` 也不在 `ha_innodb.cc`** —— 它是 **ngram 插件**的 sysvar（`plugin/fulltext/ngram_parser/plugin_ngram.cc:250`）。
3. **`ft_min_word_len` 在 8.0.39 仍然存在**（`sys_vars.cc:2290`，默认 4，READ_ONLY）—— "8.0 已删除"是错的。

**★ InnoDB 侧变量的注册原文**（`ha_innodb.cc:22481` 起）：

```cpp
static MYSQL_SYSVAR_BOOL(
    ft_enable_diag_print, fts_enable_diag_print, PLUGIN_VAR_OPCMDARG,
    "Whether to enable additional FTS diagnostic printout ", nullptr, nullptr,
    false);

static MYSQL_SYSVAR_STR(ft_aux_table, fts_internal_tbl_name,
                        PLUGIN_VAR_NOCMDARG | PLUGIN_VAR_MEMALLOC,
                        "FTS internal auxiliary table to be checked",
                        innodb_internal_table_validate, nullptr, nullptr);

static MYSQL_SYSVAR_ULONG(ft_cache_size, fts_max_cache_size,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "InnoDB Fulltext search cache size in bytes", nullptr,
                          nullptr, 8000000, 1600000, 80000000, 0);
                          /* ★ 默认 8,000,000；范围 [1600000, 80000000]；只读 */

static MYSQL_SYSVAR_ULONG(
    ft_total_cache_size, fts_max_total_cache_size,
    PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
    "Total memory allocated for InnoDB Fulltext Search cache", nullptr, nullptr,
    640000000, 32000000, 1600000000, 0);

static MYSQL_SYSVAR_ULONG(
    ft_result_cache_limit, fts_result_cache_limit, PLUGIN_VAR_RQCMDARG,
    "InnoDB Fulltext search query result cache limit in bytes", nullptr,
    nullptr, 2000000000L, 1000000L, 4294967295UL, 0);

static MYSQL_SYSVAR_ULONG(
    ft_min_token_size, fts_min_token_size,
    PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
    "InnoDB Fulltext search minimum token size in characters", nullptr, nullptr,
    3, 0, 16, 0);                       /* ★ 默认 3，范围 [0,16]，只读 */

static MYSQL_SYSVAR_ULONG(
    ft_max_token_size, fts_max_token_size,
    PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
    "InnoDB Fulltext search maximum token size in characters", nullptr, nullptr,
    FTS_MAX_WORD_LEN_IN_CHAR, 10, FTS_MAX_WORD_LEN_IN_CHAR, 0);

static MYSQL_SYSVAR_ULONG(ft_num_word_optimize, fts_num_word_optimize,
                          PLUGIN_VAR_OPCMDARG,
                          "InnoDB Fulltext search number of words to optimize "
                          "for each optimize table call ",
                          nullptr, nullptr, 2000, 1000, 10000, 0);

static MYSQL_SYSVAR_ULONG(ft_sort_pll_degree, ddl::fts_parser_threads,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "InnoDB Fulltext search parallel sort degree, will "
                          "round up to nearest power of 2 number",
                          nullptr, nullptr, 2, 1, 16, 0);

static MYSQL_SYSVAR_BOOL(optimize_fulltext_only, innodb_optimize_fulltext_only,
                         PLUGIN_VAR_NOCMDARG,
                         "Only optimize the Fulltext index of the table",
                         nullptr, nullptr, false);
```
> ★ `ft_sort_pll_degree` 绑的是 **`ddl::fts_parser_threads`**（建索引时的分词线程数），不是 OPTIMIZE 线程数。

**★ 两个会话级变量是 `THDVAR`**（`ha_innodb.cc:1088`，★ 作用域是 SESSION）：

```cpp
static MYSQL_THDVAR_BOOL(ft_enable_stopword, PLUGIN_VAR_OPCMDARG,
                         "Create FTS index with stopword.", nullptr, nullptr,
                         /* default */ true);

static MYSQL_THDVAR_STR(
    ft_user_stopword_table, PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_MEMALLOC,
    "User supplied stopword table name, effective in the session level.",
    innodb_stopword_table_validate, nullptr, nullptr);
```

**★ 三个停用词变量的汇合点**（`ha_innodb.cc:12893`）：

```cpp
bool innobase_fts_load_stopword(
    dict_table_t *table, /*!< in: Table has the FTS */
    trx_t *trx,          /*!< in: transaction */
    THD *thd)            /*!< in: current thread */
{
  ut_ad(!dict_sys_mutex_own());

  return (fts_load_stopword(table, trx, innobase_server_stopword_table,
                            THDVAR(thd, ft_user_stopword_table),
                            THDVAR(thd, ft_enable_stopword), false));
}
```
> ★ 会话级 `innodb_ft_user_stopword_table` **优先于**全局 `innodb_ft_server_stopword_table`；两者都写在 CONFIG 表持久化，重启可复原。

**★ `ngram_token_size` 的注册**（`plugin/fulltext/ngram_parser/plugin_ngram.cc:246`）：

```cpp
static struct st_mysql_ftparser ngram_parser_descriptor = {
    MYSQL_FTPARSER_INTERFACE_VERSION, ngram_parser_parse, nullptr, nullptr};

static MYSQL_SYSVAR_INT(
    token_size, ngram_token_size, PLUGIN_VAR_READONLY,
    "InnoDB ngram full text plugin parser token size in characters", nullptr,
    nullptr, 2, 1, 10, 0);   /* ★ 默认 2，范围 [1,10]，只读 */

static SYS_VAR *ngram_system_variables[] = {MYSQL_SYSVAR(token_size), nullptr};

mysql_declare_plugin(ngram_parser){
    MYSQL_FTPARSER_PLUGIN,
    &ngram_parser_descriptor,
    "ngram",                  /* ★ 插件名 → 前缀 ngram_ */
    ...
```
> ⚠️ **MeCab 没有 token size 变量**，它只有 1 个 sysvar `mecab_rc_file`（`plugin_mecab.cc:346`）。

**ngram 插件**：`ngram_token_size` 默认 **2**，范围 1..10，只读（`plugin_ngram.cc:250`）。

### 状态变量

**8.0.39 中没有任何 `Innodb_ft_*` 状态变量**（`export_var_t` 与 `innodb_status_variables[]` 中均无 FTS 字段）。

FTS 的可观测性靠 **INFORMATION_SCHEMA 表**（都必须先设 `innodb_ft_aux_table`）：

| I_S 表 | 需要 `innodb_ft_aux_table` | 内容 |
|--------|:---:|------|
| `INNODB_FT_INDEX_CACHE` | ✅ | 内存 cache 中尚未 sync 的倒排条目 |
| `INNODB_FT_INDEX_TABLE` | ✅ | 已落盘的倒排索引内容（解 ilist 后逐条展开） |
| `INNODB_FT_DELETED` | ✅ | 已删除但尚未清除的 doc_id |
| `INNODB_FT_BEING_DELETED` | ✅ | 本轮 OPTIMIZE 正在处理的 doc_id |
| `INNODB_FT_CONFIG` | ✅ | 见下 |
| `INNODB_FT_DEFAULT_STOPWORD` | ❌ | 直接遍历 `fts_default_stopword[]`，与具体表无关 |

（需要 aux table 的 5 张都先 `check_global_access(thd, PROCESS_ACL)`，非 PROCESS 权限返回空结果，如 `i_s.cc:2304`）

> **★ `INNODB_FT_CONFIG` 只输出 4 个 key**（`fts_config_key[]`，`i_s.cc:3243`）：
> `optimize_checkpoint_limit`、`synced_doc_id`、`stopword_table_name`、`use_stopword`。
> 建表时写进去的 `cache_size_in_mb` / `deleted_doc_count` / `table_state` **虽然存在，但 I_S 查不到**（另有 `FTS_TOTAL_WORD_COUNT` 在 `:3326` 有段特殊处理却不在数组里，实际也看不到）。
> 另外 **`INNODB_FT_INSERTED` 在 8.0 已彻底移除**（全仓库 0 命中；`fts0fts.cc:109` 的注释还提到它，是陈旧的）。

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

### FTS 没有实现哪些算法

> 按知识库"社区没有 X 这类澄清"的规矩：给证据、说清替代品、说清谁有。

对 `storage/innobase/fts/`、`include/fts0*`、`ddl/ddl0fts.cc`、`plugin/fulltext/` 做精确检索，**以下全部 0 命中**：

**检索模型**

| 算法 | 替代品（社区版实际用的） |
|---|---|
| **BM25 / Okapi**、任何带长度归一化的模型 | `tf × idf²`（§11） |
| **语言模型（LM）/ DFR / LTR** | 无 |
| **向量检索 / embedding / ANN**（HNSW、IVF、PQ） | 无 |

**文本处理**

| 算法 | 替代品 |
|---|---|
| **词干提取**（Porter / Snowball / Krovetz） | 无——`run` 与 `running` 是两个独立词 |
| **同义词 / 词林 / thesaurus** | 无 |
| **编辑距离 / fuzzy / Levenshtein / 拼写纠错** | 只有尾部 `%` 前缀匹配，靠红黑树前缀扫描（`fts0que.cc:930`） |

**数据结构**

| 算法 | 替代品 |
|---|---|
| **Trie / 前缀树 / 自动补全** | 红黑树 + `innobase_fts_text_cmp_prefix` 双向扫描 |
| **跳表（skip list）** | **posting list 只能从头顺序 VLC 解码，无法随机访问中间 doc_id** |
| **Roaring bitmap** | 朴素 char bitmap（初始 4B，2 倍扩容） |
| **Bloom filter** | 无（停用词直接红黑树查） |

**★★ 一个必须纠正的说法：InnoDB **是**支持短语查询与邻近（proximity）查询的**。

常见以为"InnoDB FTS 存了位置但不支持短语查询"，**这是错的**。位置信息不仅存了，而且有完整的短语/邻近检索实现：

| 能力 | 源码证据 |
|---|---|
| 短语/邻近主入口 | `fts_phrase_or_proximity_search()`（`fts0que.cc:353`、`:4031`） |
| 位置提取 | `fts_proximity_get_positions()`（`fts0que.cc:364`） |
| `@N` 距离判定 | `fts_proximity_is_word_in_range()`（`:1541`）、`fts_query_is_in_proximity_range()`（`:2277`） |
| AST 上的距离属性 | `fts_ast_text_set_distance()`（`fts0ast.cc:381` 注释 "Set the proximity attribute of a text node"） |
| **语法 `@N`** | `fts0pars.y:151` `text '@' FTS_NUMB { fts_ast_text_set_distance($1, fts_ast_string_to_ul($3, 10)); }`；`:167` 前缀形式 |
| 位置写入 | `fts_add_token()`（`fts0fts.cc:4585` 注释 "Add the token and its start position to the token's list of positions"） |
| 词数上限 | `MAX_PROXIMITY_ITEM = 128`（`fts0que.cc:60`） |

```yacc
# fts0pars.y:151 —— InnoDB 布尔语法里的邻近距离
        | text '@' FTS_NUMB {
                fts_ast_text_set_distance($1, fts_ast_string_to_ul($3, 10));
                fts_ast_string_free($3);
        }
```
> ★★ 所以 `AGAINST('"database tuning" @2' IN BOOLEAN MODE)` 这类查询在 InnoDB 里**是支持的**。
> ⚠️ 但注意：`&` 和 `|` 虽然在 `ft_boolean_syntax` 里，InnoDB 的 bison 语法**没有对应算子** —— 它们只是被 server 层接受。

**★ 逐条 grep 实证的"没有"清单**（★ 每条都真的 grep 过）：

| IR 特性 | 检索结果 |
|---|---|
| stemming（词干提取） | `storage/innobase` grep `stemming\|stemmer` → **0 命中**；`sql/` 也无 FTS 相关命中 |
| lemmatization（词形还原） | grep `lemmatiz` → **0 命中** |
| synonym（同义词） | `storage/innobase` grep `synonym` → **0 命中** |
| highlight / snippet（高亮） | grep `highlight\|snippet` → **0 命中**；没有 `ft_highlight()` 类函数 |
| facet（分面） | grep `facet` → **0 命中** |
| fuzzy（编辑距离模糊匹配） | grep `fuzzy` → **0 命中**；只有尾部 `%` 前缀匹配 |
| spell-correction（拼写纠错） | grep `spell` → **0 命中** |
| BM25 | 全 innobase grep `bm25` → **0 命中**；且 `k1` / `avgdl` 无命中 |
| 字段加权（field boost） | `fts_query_calculate_ranking()` 中无任何列权重项 |
| 自定义 scoring 插件 | `ft_vft_result` 只有 4 个槽（`ha_innodb.cc:607`），无 scoring hook |

**★ 有的（别漏）**：

| 特性 | 证据 |
|---|---|
| 短语查询 phrase | `fts_phrase_or_proximity_search()` |
| 邻近查询 proximity `@N` | 见上表 |
| 查询扩展 query expansion | `fts_expand_query()`（`fts0que.cc:3904`）+ `ft_query_expansion_limit`（默认 20） |
| 尾部通配符 `*` | `fts_query_get_token()`（`fts0que.cc:2693`，转成 `xxx%`） |
| ngram / MeCab 分词 | `plugin/fulltext/{ngram,mecab}_parser/` |

**★ 盲查询扩展的代价**（`fts0que.cc:3904`，★ 源码注释自白）：

```c
/** This function implements a simple "blind" query expansion search:
 words in documents found in the first search pass will be used as
 search arguments to search the document again, thus "expand"
 the search result set. */
...
    /* Fetch the documents with the doc_id from the
    result of first search pass. Since we do not
    store document-to-word mapping, we need to
    fetch the original document and parse them.
    Future optimization could be done here if we
    support some forms of document-to-word mapping */
    fts_doc_fetch_by_doc_id(nullptr, ranking->doc_id, index,
                            FTS_FETCH_DOC_BY_ID_EQUAL,
                            fts_query_expansion_fetch_doc, &result_doc);
```
> ★★ 注释直说：**没有 doc→word 正排索引**，所以 QE 必须把第一趟命中的每个文档整篇读回来重新分词 —— 这是 QE 慢的根本原因。
> ★ 二轮检索的新词**全部按 `FTS_NONE`（并集）**加入，同样参与 `Σ tf·idf²`。
| **Aho-Corasick**（多模式串匹配；KMP / Boyer-Moore 也没有） | 朴素逐 token 顺序匹配 |
| **MinHash / SimHash / LSH** | 无 |

**压缩编码**

| 算法 | 替代品 |
|---|---|
| **PForDelta / FOR / SIMD 批量解码** | 逐字节 7-bit VLC（`fts0vlc.ic:114`） |
| **前缀压缩 / front coding** | word 完整存储 |

**其它**

| 算法 | 替代品 |
|---|---|
| **k 路归并 / 优先队列 / 败者树**（外部排序） | `N_WAY_MERGE = 2` 的平衡 2 路外部归并 |
| **LRU / LFU 淘汰**（FTS cache） | 超阈值全量 flush 后清空，无逐项淘汰 |

**三处"看起来有、其实是死代码"**（读 FTS 源码最容易踩的坑）：

| 符号 | 真相 |
|---|---|
| `fts_query_lcs`（`fts0que.cc:427`）——教科书式 O(m·n) LCS 动态规划 | 整段被 **`fts0que.cc:398 #if 0` … `:482 #endif`** 关掉，**无调用点**，运行时等效不存在 |
| `fts_optimize_add_threshold` / `delete_threshold` / `fts_optimize_need_sync` | `fts0opt.cc:2744 #if 0` … `:2792 #endif` |
| `fts_query_total_docs_containing_term`（`fts0que.cc:2078`） | 无调用者；`df` 实际只取 INDEX 表的 `doc_count` 列 |

**最该记住的一条**：posting list **没有跳表、没有分块**。单个高频词的 ilist 很长时，即便只要前 N 条结果，也要把整个 ilist **从头顺序 VLC 解码一遍**——唯一的加速是 `fts0que.cc:1148-1163` 的 node 级 `[lower_doc_id, upper_doc_id]` 区间剪枝，粒度是整个 node 而非 block。这是 InnoDB FTS 在大数据集上查询慢的**结构性原因**。

**整体定位：能用、够用——以及为什么"正经搜索场景上 ES"是技术判断而非生态判断。**

综合"已实现"（「理论基础 → 算法与数据结构」的 7 类盘点）与"未实现"（本节清单），这套实现的定位可以概括为：**分词、倒排、布尔、短语、查询扩展、崩溃恢复一应俱全，但压缩编码（只有 VLC+delta）、posting list 加速（无跳表/分块）、打分模型（无长度归一化）三处都停在最朴素的方案上**——每一处都"能用"，没有一处是现代搜索引擎的水准。

⇒ 业界普遍建议"正经搜索场景上 ES"的根因是**技术性的，不是生态或易用性**：这三个短板在大数据量下表现为"查询慢 + 排序差"，而且**不可通过调参修复**——要修就得重写存储格式，而这又被「理论基础①」的根本设计（辅助表必须是普通 InnoDB 表）锁死了。

反过来说，它的优势同样真实：**辅助表就是普通 InnoDB 表 ⇒ 事务、崩溃恢复、备份、复制与主表同一生命周期**——不需要双写、没有同步延迟。选型就按这个定位取舍：数据量不大、要与事务强一致、不想引入外部组件 ⇒ InnoDB FTS；数据量大、要求排序质量与查询性能 ⇒ 外部搜索引擎（与 Lucene / PostgreSQL 的逐项对比见「理论基础 → 类似实现对比」）。

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
A：不一样。ES/Lucene 用 **BM25**（有 k1/b 参数、有文档长度归一化、tf 饱和）；InnoDB 用 **tf × idf²**（底数 log10、无归一化、tf 是原始词频）。所以同样的语料，两者的排序结果和分数绝对量都不同，`MATCH` 的分数只能用于**相对比较**，不能当绝对阈值用。逐项对比与数值例子见 [§11.0](#110-先理解-bm25innodb-缺了哪两块)。

**Q：能改成 BM25 吗？**
A：社区版不能——全仓库搜 `bm25` / `Okapi` 零命中，没有插件点。最省事的缓解办法是在应用层给分数补一个长度惩罚因子（如 `score / log(1 + len/avg_len)`），因为"长文本霸榜"是缺长度归一化最直接的症状。

**Q：FTS 辅助表能直接 SELECT 吗？**
A：不能。它们是 `DICT_TF2_AUX` 标记的隐藏表。想看内容只能通过 I_S 的 `INNODB_FT_*` 表（且要先设 `innodb_ft_aux_table`）。

**Q：InnoDB 支持短语查询吗？**
A：**支持**（这是常见误解的反面）。`fts_phrase_or_proximity_search()` + `@N` 语法都有完整实现，见 [§FTS 没有实现哪些算法](#fts-没有实现哪些算法) 的纠正段。不支持的是 stemming / 同义词 / 高亮 / 模糊匹配。

**Q：InnoDB 支持 `WITH PARSER` 吗？**
A：**支持**。`ER_INNODB_NO_FT_USES_PARSER` 已改名为 `OBSOLETE_ER_INNODB_NO_FT_USES_PARSER` 且代码中 0 处引用；`is_ngram` 由 parser 名判定（`ha_innodb.cc:7596`）。

**Q：MATCH 的列顺序要和索引一致吗？**
A：**不用**。`fix_index()` 只统计"MATCH 的列有多少落在该索引的列集合中"，比较的是**数量**而非顺序（`item_func.cc:7694-7726`）。

### ★★★ 真名勘误（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `add_ft_key()`（单数） | **不存在**，真名 **`add_ft_keys()`**（复数），`static`，在 `sql/sql_optimizer.cc:7822` |
| `Item_func_match::get_mm_tree()` | **不存在**（0 命中）。FT **不走 range optimizer**，由 `add_ft_keys()` 直接 push `Key_use` |
| `fulltext_cond()` | **不存在** |
| `FT_SELECT` 结构体 | **不存在**，8.0 用 `JOIN_TAB::set_ft_func()` + `JT_FT` |
| `FT_EXPANSION` | **不存在**，真名 **`FT_EXPAND` = 4**，在 `include/ft_global.h:109`（不是 `sql/sql_lex.h`） |
| `FT_OPERATOR_*` 枚举 | **不存在**，InnoDB 侧是 `fts_ast_oper_t`，SQL 层是 `ft_boolean_syntax` 字符串 |
| `fts_ast_type` | 真名 **`fts_ast_type_t`**，**7 个取值**（OPER/NUMB/TERM/TEXT/PARSER_PHRASE_LIST/LIST/SUBEXP_LIST） |
| `fts_vlc_encode` / `fts_encode_ulint` | **都不存在**。真名 **`fts_encode_int`** / **`fts_decode_vlc`** / **`fts_get_encoded_len`** |
| `fts_encode_ilist` / `fts_decode_ilist` | **都不存在**（0 命中）。编码内联在 `fts_cache_node_add_positions()`，解码内联在各消费点 |
| `fts_trx` 在 `trx0fts.cc` | **该文件不存在**。`fts_trx_*` 全部在 `fts0fts.cc` |
| DELETE 写墓碑用 `fts_add_deleted_doc_id` / `fts_delete_doc_id` | **都不存在**，真名是 **`fts_delete()`**（`static`，`fts0fts.cc:3048`） |
| 后台线程唤醒用 `fts_optimize_wakeup` | **不存在**。靠 **`ib_wqueue_timedwait(wq, 5s)`** 轮询 + 消息（`FTS_MSG_*`） |
| `fts_builtin_wordsep` / `fts_default_tokenizer` | **都不存在**。内置 parser 是 **`fts_default_parser`**（`st_mysql_ftparser` 实例，复用 MyISAM 的 `mysql_parse`） |
| `FTS_STOPWORD_TABLE_NAME` 是一张表名 | 它是 **CONFIG 表里的一个 key** `"stopword_table_name"`（配 `FTS_USE_STOPWORD` = `"use_stopword"`） |
| `fts_get_synced_doc_id()` | **不存在**。水位线就是字段 **`fts_cache_t::synced_doc_id`**（`fts0types.h:193`），读写函数是 `fts_cmp_set_sync_doc_id()`（`static`） |
| `fts_init_tables()` / `FTS_Init_tables()` | **不存在**。入口是 **`fts_init_index()`**；恢复回调 `fts_init_recover_doc()` / `fts_init_get_doc_id()` |
| 崩溃恢复在启动线程扫描重建 | **错**。是**懒**的：首次 FTS 访问时由 `ha_innodb.cc:11008` 检查 `ADDED_TABLE_SYNCED` 才触发 |
| 状态变量 `Innodb_ft_inserted` 等 | **8.0.39 已彻底没有**（`mysqld.cc` grep `Innodb_ft_` → 0 命中）。只剩 I_S 表 |
| `innodb_ft_*` 定义在 `sql/sys_vars.cc` | **错**。在 **`storage/innobase/handler/ha_innodb.cc`**，声明名 `ft_xxx`，前缀由插件自动加 |
| `ngram_token_size` 在 `sys_vars.cc` / `ha_innodb.cc` | **都不在**。在 **`plugin/fulltext/ngram_parser/plugin_ngram.cc:250`** |
| `ft_min_word_len` 在 8.0 被删 | **还在**（`sys_vars.cc:2290`，默认 4，READ_ONLY），只服务 MyISAM |
| InnoDB 不支持短语/邻近查询 | **错**，完整实现了（`fts_phrase_or_proximity_search()` + `@N`） |
| InnoDB 不支持 `WITH PARSER` | **错**，错误码已 `OBSOLETE_`，0 引用 |
| MATCH 列顺序必须与索引一致 | **错**，`fix_index()` 只比数量不比顺序 |
| 打分函数叫 `fts_rank_doc` | **不存在**，真名 **`fts_query_calculate_ranking()`** |
| InnoDB rank 被 clamp 到 [0,1] | **错**。打分函数无任何 clamp；clamp 只在布尔算子路径（范围 `[-1,1]`） |
| idf 用 `pow(log(...), 2)` | **不是 `pow`**，是连续乘两次同一个 `double` |
| InnoDB 有 50% 阈值 | **没有**。那是 **MyISAM** 的 `GWS_PROB` 副作用（`log((N-df)/df)`，df>N/2 时归零） |
| 单 term 时"跳过 AST 解析" | **错**，AST 照样解析。快路径真名 **`FTS_OPT_RANKING`**，跳过的是 `doc_ids` 树 + 命中词位图 |
| `fts0pars.yy` | 真名 **`fts0pars.y`**（另有生成物 `fts0pars.cc`、词法 `fts0blex.l` / `fts0tlex.l`） |
| `HA_CAN_FULLTEXT` 在 `include/my_base.h` | **不在**，三个 FULLTEXT 能力位都在 **`sql/handler.h`**（`:305/:416/:441`） |
| MyISAM 也有 `HA_CAN_FULLTEXT_EXT` | **没有**。MyISAM 只有 `HA_CAN_FULLTEXT`，无 EXT、无 HINTS |
| 辅助表是 6 张 | **6 张 INDEX 分片 + 5 张 COMMON**（`FTS_NUM_AUX_INDEX=6`、`FTS_NUM_AUX_COMMON=5`） |
| InnoDB FTS 有 WL 编号 | 整个 `storage/innobase/fts` 目录 grep `WL#` **只有 1 条**（`fts0fts.cc:5479`，WL#6608，还在 `#if 0` 里）。**没有**引入 FTS 的 WL 编号 |

> ★ 一个官方未清理的开发残留：`fts0que.cc:756` 注释写 `RANK_DOWNGRADE (-0.5)` / `RANK_UPGRADE (0.5)`，但实际值（`fts0que.cc:56-57`）是 **±1.0F** —— 注释滞后于代码。

---

## 参考

**论文**

- Salton, G., Wong, A., Yang, C. S. *A Vector Space Model for Automatic Indexing*. CACM 1975. —— 向量空间模型与 TF-IDF 的出处；InnoDB 的 `log10(N/df)` 与 `tf × idf²` 是它的简化实现（理论溯源见「理论基础 → 理论溯源」）。
- Robertson, S., Zaragoza, H. *The Probabilistic Relevance Framework: BM25 and Beyond*. Foundations and Trends in IR, 2009. —— **用来对照说明 InnoDB 缺什么**：BM25 的 `k1` 饱和与 `b` 长度归一化，`fts_query_calculate_ranking` 里一个都没有，这是"长文档占优、rank 不能当绝对阈值"的根因。
- Rocchio, J. J. *Relevance Feedback in Information Retrieval*. 1971. —— 查询扩展（`WITH QUERY EXPANSION`）的思想源头：用首轮命中文档的词反馈进查询。
- Robertson, S. E., Spärck Jones, K. *Relevance Weighting of Search Terms*. JASIS 1976. —— 概率排序原理，IDF 的 `log((N-df+0.5)/(df+0.5))` 形式出于此；BM25 是它的工程实现。
- Ponte, J. M., Croft, W. B. *A Language Modeling Approach to Information Retrieval*. SIGIR 1998. —— 语言模型检索（查询似然）的开山之作，BM25 之外最大的一支（算法家族见 §11.0）。
- Lv, Y., Zhai, C. *When Documents Are Very Long, BM25 Fails!* SIGIR 2011. —— 提出 BM25+，修正 BM25 对超长文档的过度惩罚；读它有助于理解 `b` 参数的作用边界。

**官方文档**

- *MySQL 8.0 Reference Manual → Full-Text Search Functions*（`MATCH ... AGAINST` 的三种模式与布尔操作符语义）
- *MySQL 8.0 Reference Manual → InnoDB Full-Text Indexes*（辅助表体系、`FTS_DOC_ID`、`innodb_ft_*` 变量、`INNODB_FT_*` I_S 表）
- *MySQL 8.0 Reference Manual → ngram Full-Text Parser* / *MeCab Full-Text Parser Plugin*（CJK 分词）
- *MySQL 8.0 Reference Manual → Full-Text Restrictions*

**内核月报 / 技术文章**

- 阿里数据库内核月报（mysql.taobao.org/monthly）InnoDB 全文索引系列。注意：其函数名与行号多基于 5.7，**与本版 8.0.39 对不上**（如旧的 `row0ftsort.cc` 建索引路径在 8.0 已被 `ddl/ddl0fts.cc` 取代），读时须回源码核实。

**相关文档**

- 上层（谁驱动 FTS 查询）：执行器与 filesort 见 [`../server/query/runtime/01_filesort.md`](../server/query/runtime/01_filesort.md)（§1.3 addon vs rowid，本篇 §2.8 是它的一个强制场景）
- 接口层：handler/handlerton 分层与行定位见 [`../server/handler.md`](../server/handler.md)
- InnoDB 侧通用机制：B-tree 页结构 [`../innodb/physical/page_structure.md`](../innodb/physical/page_structure.md)、大对象 [`../innodb/physical/lob.md`](../innodb/physical/lob.md)（ilist 是 BLOB）、在线 DDL [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md)
- 同类跨层特性：分区表 [`partitioning.md`](partitioning.md)（与 FTS 一样同时改 SQL 层与引擎层，且**分区表不支持 FULLTEXT**）
