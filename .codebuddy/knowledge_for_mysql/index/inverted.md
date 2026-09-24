# 倒排索引：全文检索的存储结构与编码

> 基于 MySQL 8.0.39 源码。本篇讲 **倒排索引作为索引数据结构的完整实现**：ilist 的字节级编码（VLC + delta）、6 档辅助表分档策略、删除的旁路表示与 optimize 合并、word 的组织与查询展开、cache 与辅助表的边界。
>
> **边界**：全文检索的全链路（`MATCH AGAINST` 语法 → 优化器 → 打分 → 崩溃恢复）见 [`../feat/fts.md`](../feat/fts.md)；11 张辅助表的清单与 `FTS_DOC_ID` 在类型谱系里的位置见 [`types.md`](types.md)。本篇聚焦**倒排数据本身的组织与编码**——它是"索引存储结构"视角的权威出处。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [ilist 的字节级布局：VLC + delta 编码](#ilist-的字节级布局vlc--delta-编码)
    - [6 档辅助表：按范围 vs CJK 哈希](#6-档辅助表按范围-vs-cjk-哈希)
  - 读取路径
    - [word 的组织与查询展开](#word-的组织与查询展开)
  - 写路径与维护
    - [写入：cache 里的 node 追加](#写入cache-里的-node-追加)
    - [删除的旁路表示与 optimize 合并](#删除的旁路表示与-optimize-合并)
    - [cache 与辅助表的边界](#cache-与辅助表的边界)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

全文索引的倒排结构 = **word → (doc_id, positions) 列表**。与 B-tree/R-tree 完全不同：词到文档的映射是"一对多、无序增长"的，物理上由 **6 张 INDEX 辅助表**（每张是独立 InnoDB 表，内部还是 B-tree）+ 一个**内存 cache** 承载：

```
INDEX 辅助表（每张）：主键 (word, first_doc_id) 聚簇
  word VARCHAR | first_doc_id | last_doc_id | doc_count | ilist VARBLOB
```

**ilist 是核心**：一个 word 的若干文档（一个 node）的倒排列表，用 VLC 变长编码 + delta 增量压成字节串。

### 用途

- `MATCH...AGAINST` 查询时把 word 展开成 doc_id 集合（打分需要 position 统计词频 tf）；
- 倒排结构决定查询路径：定位 word → 展开所有 node 的 ilist → 过滤已删 doc → 排名。

### 版本演进

- 5.6/5.7 有 `fts0vlc.cc` 与 `fts_vlc_encode` API；**8.0.39 里已内联为 `include/fts0vlc.ic` 的函数**（`fts_encode_int` / `fts_decode_vlc`）——凭 5.x 记忆写函数名必错。

---

## 理论基础

### 设计思想与权衡

**1. 为什么分 6 张表（而不是一张大表）**：把首字符相近的词散到 6 棵独立 B-tree，每棵更小、cache 局部性更好；同步（sync）时可以分表并行刷。代价是查询多一层"选表"逻辑。

**2. 为什么 doc_id/position 用 delta 编码**：同一 word 的文档按 doc_id 升序追加，相邻 doc_id 差值小；一个文档内 position 递增——delta 值大多落在 1~2 字节的 VLC 区间，显著压缩。

**3. 为什么 ilist 里没有删除墓碑**：ilist 是**只追加、不可变**的字节流（合并时整体重写）。若原地打删除标记，会破坏 delta 链（删一个 doc_id，后续所有 delta 都要改）。所以删除记在**旁路集合**（DELETED 表 + cache 向量），查询时过滤，optimize 时才物理清除。

**4. node 是追加单元**：同一个 word 在磁盘上是**多行**（多 node），每行覆盖一段不重叠的 doc_id 区间。cache 已 sync 的 node 不再追加（不可变），新文档开新 node——这使"内存 cache → 落盘"变成纯追加，无需改写已刷数据。

### 理论溯源

- 倒排索引（inverted index）是信息检索的经典结构；SQLite FTS、Elasticsearch 等同源。
- VLC 变长编码与大端 7-bit 分组：与 protobuf varint 同类但**方向相反**（MySQL 是大端 + **终止位**，protobuf 是小端 + **继续位**）。

---

## 核心实现

### 主线与基础构件

#### ilist 的字节级布局：VLC + delta 编码

重复单元：

```
[ delta(doc_id) : VLC ] [ delta(pos1) : VLC ] [ delta(pos2) : VLC ] ... [ 0x00 结束字节 ]
```

- **doc_id 增量**：当前 doc_id − 该 node 上一个文档的 `last_doc_id`（node 首文档即 `first_doc_id` 本身；解码侧 `fts0que.cc:2986` 断言 `ut_a(pos == node->first_doc_id)` 当 `doc_id == 0`）；
- **position 增量**：`pos − last_pos`，`last_pos` **每个文档重置为 0**；
- **0x00 终止字节**：标志该文档 position 列表结束。它可行是因为 VLC 编码的任何字节都不可能是 0x00（见下）。

VLC 编码（`include/fts0vlc.ic:64-110`）：

```cpp
static inline ulint fts_encode_int(ulint val, byte *buf) {
  ulint len;
  if (val <= 127) {
    *buf = (byte)val; len = 1;
  } else if (val <= 16383) {
    *buf++ = (byte)(val >> 7); *buf = (byte)(val & 0x7F); len = 2;
  } else if (val <= 2097151) { ... len = 3; }
  ...
  /* High-bit on means "last byte in the encoded integer". */
  *buf |= 0x80;
  return (len);
}
```

关键设计：

- 每 byte 只用**低 7 位**，按**大端 7-bit 组**从高到低输出（与 protobuf varint 相反）；
- **终止位**：最后一个字节的最高位**置 1**，前面的字节最高位为 0（"还有后续"）；
- 长度分界（`fts_get_encoded_len`，1/2/3/4/5 字节）：127、16383、2097151、268435455——最多 5 字节覆盖 32 位值。

解码（`fts_decode_vlc`，114-135 行）：

```cpp
static inline ulint fts_decode_vlc(byte **ptr) {
  ulint val = 0;
  for (;;) {
    byte b = **ptr;
    ++*ptr;
    val |= (b & 0x7F);
    if (b & 0x80) break;      // 终止位
    val <<= 7;
  }
  return (val);
}
```

写端组装（`fts_cache_node_add_positions`，`fts0fts.cc:1053-1161`）：

```cpp
doc_id_delta = (ulint)(doc_id - node->last_doc_id);
enc_len = fts_get_encoded_len(doc_id_delta);
last_pos = 0;
for (i = 0; i < ib_vector_size(positions); i++) {
  ulint pos = *(static_cast<ulint *>(ib_vector_get(positions, i)));
  enc_len += fts_get_encoded_len(pos - last_pos);
  last_pos = pos;
}
enc_len++;                                   // 0x00 结束字节
...
ptr += fts_encode_int(doc_id_delta, ptr);
for (...positions...) { ptr += fts_encode_int(pos - last_pos, ptr); last_pos = pos; }
*ptr++ = 0;
if (node->first_doc_id == FTS_NULL_DOC_ID) node->first_doc_id = doc_id;
node->last_doc_id = doc_id;
++node->doc_count;
```

读端展开（`fts_query_filter_doc_ids`，`fts0que.cc:2958-3077`）：

```cpp
while (decoded < len) {
  ulint pos = fts_decode_vlc(&ptr);
  if (doc_id == 0) ut_a(pos == node->first_doc_id);   // node 首文档校验
  doc_id += pos;
  while (*ptr) {                       // ★ 直接用 0x00 判 position 段结束
    last_pos += fts_decode_vlc(&ptr);
    ++freq;
  }
  ++ptr;                               // 跳过结束字节
  ...
}
ut_a(doc_id == node->last_doc_id);     // node 尾校验
```

`while (*ptr)` 之所以安全：VLC 流的中间字节高 7 位若全零则该值必已结束（带 0x80）——所以 0x00 只能是哨兵。

#### 6 档辅助表：按范围 vs CJK 哈希

```cpp
// fts0fts.cc:151-158
const fts_index_selector_t fts_index_selector[] = {
    {9, "index_1"},  {65, "index_2"}, {70, "index_3"}, {75, "index_4"},
    {80, "index_5"}, {85, "index_6"}, {0, nullptr}};
```

选择逻辑（`include/fts0types.ic:178-189`）：

```cpp
static inline ulint fts_select_index(const CHARSET_INFO *cs, const byte *str, ulint len) {
  if (fts_is_charset_cjk(cs)) {
    return fts_select_index_by_hash(cs, str, len);
  }
  return fts_select_index_by_range(cs, str, len);
}
```

- **非 CJK：按范围**——对**整词**（不是首字符）做 `innobase_strnxfrm(cs, str, len)` 得排序权重，在 selector 数组里找上界。边界 9/65/70/75/80/85 对应 collation 值：0-8（数字/标点）→ index_1，65('A')-69 → index_2，70('F')-74、75('K')-79、80('P')-84、85('U')-… → 其余——**意图是把 26 个字母近似均匀切 5 段 + 1 段符号**（E/J/O/T/Y 之后切，每段约 5 个字母），让 6 张表数据均衡；
- **CJK 字符集**（gb2312/gbk/big5/gb18030/ujis/sjis/cp932/eucjpms/euckr，`fts_is_charset_cjk`）：排序权重不便按范围切，改**哈希**——`cs->coll->hash_sort(首字符)` 取 `nr1 % FTS_NUM_AUX_INDEX`（=6）。

---

### 读取路径

#### word 的组织与查询展开

INDEX 表主键 `(word, first_doc_id)`：**同一 word 的多个 node 按区间不重叠、`first_doc_id` 升序**排布在相邻记录。word 按该 FT 索引列的 charset/collation 存取（写入前 token 已按 collation casedn 归一化，binary collation 则大小写敏感）。

查询内嵌 SQL（`fts0que.cc:1987-2007`）：

```sql
SELECT doc_count, ilist
  FROM $index_table_name
 WHERE word LIKE :word
   AND first_doc_id <= :min_doc_id
   AND last_doc_id  >= :max_doc_id
 ORDER BY first_doc_id;
```

- doc_id 范围条件用于**跳过区间外的 node**（对应 `fts_query_read_node` 的 skip 逻辑）；
- 行回调展开 ilist（上文读端），node 边界由 `first_doc_id` 校验衔接；
- 过滤已删 doc（见下节）后进打分结构（`fts_word_freq_t::doc_freqs`，tf × idf² 见 [`../feat/fts.md`](../feat/fts.md)）。

#### 从 ilist 到打分：tf × idf² 的数据流

**ilist 的 position 计数就是 tf**（`fts_query_filter_doc_ids`，`fts0que.cc:2958-3077`）：

```cpp
while (decoded < len) {
  ulint freq = 0;
  ulint pos = fts_decode_vlc(&ptr);      // doc_id delta
  doc_id += pos;
  ...
  while (*ptr) {                          // 0x00 哨兵结束 position 段
    last_pos += fts_decode_vlc(&ptr);
    ++freq;                               // ← tf：每个 position +1
  }
  ...
  doc_freq = fts_query_add_doc_freq(query, doc_freqs, doc_id);
  /* Avoid duplicating frequency tally. */
  if (doc_freq->freq == 0) doc_freq->freq = freq;   // 跨 node 不累加
  ++ptr;
  ...
}
```

两个关键语义：

- **tf = 该文档在 ilist 里的 position 个数**——与字节级编码（上节）直接对应；
- **同一 word 的多个 node 命中同一文档时 freq 不累加**（`freq == 0` 才赋值）——因为各 node 的 doc_id 区间天然不重叠（`first_doc_id`/`last_doc_id` 校验）；**跨 node 累加的是 `doc_count`**（`fts_query_read_node` 对每个 node 行的 DOC_COUNT 列累加，`fts0que.cc:3128`；缓存/通配路径走 `calc_doc_count=true` 的 ilist 直数）。

**打分公式（确认 tf × idf²，无文档长度归一化）**：

```cpp
// fts_query_calculate_idf（fts0que.cc:3214）
word_freq->idf = log10(total_docs / (double)word_freq->doc_count);
// fts_query_calculate_ranking（fts0que.cc:3250）
weight = (double)doc_freq->freq * word_freq->idf;
ranking->rank += (fts_rank_t)(weight * word_freq->idf);   // Σ tf × idf²
```

- `idf` 每词每查询**现算不缓存**；`total_docs = dict_table_get_n_rows()`（统计值，可能过期）；
- 单 term 优化路径（`FTS_OPT_RANKING`，AST 为单个无操作符 TERM）：先 `ranking.rank = freq`、算出 idf 后 `rank = rank × idf × idf`（3405-3418）；
- **optimize 后 doc_count 变小 → idf 变大**（删除文档在合并时被跳过、`++dst_node->doc_count` 写回 DOC_COUNT 列）。

**汇入排序结构**：`fts_query_add_ranking`（3290）把 `fts_ranking_t{doc_id, rank, words 位图}` 按 doc_id 汇入 `rankings_by_id`；`ha_innobase::ft_read` 首次取行时 `fts_query_sort_result_on_rank`（3849）构建 `rankings_by_rank`（rank 降序 + doc_id，比较器 `fts_query_compare_rank`）；逐行沿 rank 树前进经 `fts_doc_id_index` 回表。SQL 层取单文档 rank 走 `innobase_fts_retrieve_ranking` → **`fts_retrieve_ranking`**（3317，按 doc_id 查 rank；注意**不存在** `fts_query_retrieve_ranking` 这个函数名）。

**boolean 模式的一个易错点**：boolean 的 rank 初值是操作符调整（`RANK_UPGRADE(1.0F)`/`RANK_DOWNGRADE(-1.0F)`），但 `fts_query_prepare_result` 对**所有**非 `FTS_OPT_RANKING` 模式（含 boolean）都再叠加 Σ tf·idf²——**boolean 结果同样按合并后 rank 降序输出**（SQL 层不消费该 rank 值，但返回行的物理顺序由它决定），并非严格按 doc_id 序。

---

### 写路径与维护

#### 写入：cache 里的 node 追加

commit 时 `fts_add` → `fts_add_doc_by_id`：用 doc_id 查 `FTS_DOC_ID` 索引定位行、读回原文、分词（`fts_doc_t.tokens` 红黑树）→ `fts_cache_add_doc`（`fts0fts.cc:1164-1230`）：对每个 token 找到 cache 中该 word 的**最后一个 node**；若 `synced || ilist_size > FTS_ILIST_MAX_SIZE || doc_id < last_doc_id` 则**新开 node**，否则在现 node 追加编码。

sync 落盘（`fts_sync_write_words`，`fts0fts.cc:4071-4179`）：

```cpp
for (rbt_node = rbt_first(index_cache->words); rbt_node; rbt_node = rbt_next(...)) {
  word = rbt_value(fts_tokenizer_word_t, rbt_node);
  selected = fts_select_index(index_cache->charset, word->text.f_str, word->text.f_len);
  fts_table.suffix = fts_get_suffix(selected);
  for (i = 0; i < ib_vector_size(word->nodes); ++i) {
    fts_node_t *fts_node = ...;
    if (fts_node->synced) continue;
    fts_node->synced = true;
    error = fts_write_node(trx, &index_cache->ins_graph[selected], &fts_table, &word->text, fts_node);
  }
}
```

`fts_write_node` 最终执行 `INSERT INTO $index_table VALUES (:token, :first_doc_id, :last_doc_id, :doc_count, :ilist)`——**sync 不改写已有行，只追加新 (word, first_doc_id) 行**；node 标记 `synced`，回滚时 `fts_sync_index_reset` 置回。

#### 删除的旁路表示与 optimize 合并

**ilist 里没有墓碑**（不可变字节流，原地打标会破坏 delta 链）。删除信息在三处旁路：

1. **commit**：`fts_delete`（`fts0fts.cc:3048`）把 doc_id `INSERT INTO $deleted` 表，并 `++cache->deleted`；`fts_modify` = `fts_delete` + `fts_add`；
2. **内存 cache**：`fts_cache_t::deleted_doc_ids`（`fts_update_t` 向量）；
3. **查询时**：`fts_query` 初始化把三处删除集合并成**排序向量**（`fts0que.cc:3684-3709`）——先 fetch `deleted` 表、再 `deleted_cache` 表、再 append cache 向量，然后 `ib_vector_sort(query.deleted->doc_ids, fts_update_doc_id_cmp)`——之后对展开出的每个 doc_id **二分查找**过滤。

物理清除（后台 optimize 线程，`fts0opt.cc`）：把 DELETED/DELETED_CACHE 移入 BEING_DELETED，然后**逐词合并 ilist**——核心 `fts_optimize_node`（`fts0opt.cc:1139-1225`）：

```cpp
while (copied < src_node->ilist_size && dst_node->ilist_size < FTS_ILIST_MAX_SIZE) {
  delta = fts_decode_vlc(&enc->src_ilist_ptr);
  ...
  doc_id += delta;

  if (del_doc_id > 0 && doc_id == del_doc_id) {
    ++*del_pos;
    while (*enc->src_ilist_ptr) fts_decode_vlc(&enc->src_ilist_ptr);  // 跳过该文档 position 段
    ++enc->src_ilist_ptr;
  } else {
    if (del_doc_id > 0 && doc_id > del_doc_id) { del_doc_id = 0; ++*del_pos; delta = 0; goto test_again; }
    fts_optimize_encode_node(dst_node, doc_id, enc);   // ★ 重新 delta 编码拼入目标 node
    ++dst_node->doc_count;
  }
  copied = enc->src_ilist_ptr - src_node->ilist;
}
```

- **顺序扫描源 ilist**，命中已排序删除向量的文档**整体跳过其 position 段**，其余文档**重新 delta 编码**拼入目标 node（node 填满 `FTS_ILIST_MAX_SIZE` 就换下一个）；
- 删除向量定位用 `fts_optimize_deleted_pos`（1229-1262 行）按 word 的 `[first_doc_id, last_doc_id]` 区间二分找起点；
- 删除集没变化（`changed == false`）的 node 直接复用不重写。

#### cache 与辅助表的边界

```cpp
struct fts_cache_t {
  rw_lock_t lock;                    // 保护全部 cache
  ib_mutex_t deleted_lock;           // 保护 deleted_doc_ids
  ib_vector_t *deleted_doc_ids;
  ib_vector_t *indexes;              // 每个 FT 索引一个 fts_index_cache_t
  ulint total_size;                  // 所有 node ilist 字节数，超限触发 SYNC
  doc_id_t synced_doc_id;            // 已写入 CONFIG 表的 doc id
  ulint deleted;                     // 自上次 optimize 以来删除数
  ...
};
```

边界与合并规则：

- **cache 中 ilist 与落盘 ilist 编码完全相同**（同是 `fts_cache_node_add_positions` 的产物，落盘只是把 `fts_node_t` 字段插进表）；落盘后 node 打 `synced=true` 不再追加，**新文档开新 node**；
- 同一个 word 在磁盘上因此是**多行**，行间 doc_id 区间不重叠；查询端 `ORDER BY first_doc_id` 顺序展开全部行拼接解码；
- **sync 触发条件**：`cache->total_size` 超过 `innodb_ft_cache_size` / `innodb_ft_total_cache_size`；sync = "cache → INDEX 表追加 + deleted → DELETED_CACHE + CONFIG 表更新 `synced_doc_id`"；
- 只有后台 optimize 才做**新旧 ilist 的合并/删除过滤/行重写**；平时 cache 与磁盘互不覆盖，查询时取并集。

---

## 相关的系统变量/状态变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `innodb_ft_cache_size` | 32M | 每 FT 索引的 cache 上限（触发 sync） |
| `innodb_ft_total_cache_size` | 640M | 整表所有 FT 索引的 cache 总上限 |
| `innodb_ft_enable_stopword` / `innodb_ft_server_stopword_table` | ON / 无 | 停用词过滤（发生在分词入 cache 前） |
| `FTS_ILIST_MAX_SIZE` | 源码常量 | 单 node ilist 上限，超过则 node 轮转 |
| `FTS_NUM_AUX_INDEX` | 6 | INDEX 辅助表数（CJK 哈希的模数同源） |
| `FTS_MAX_WORD_LEN` | 84 | word 列宽度 |

---

## Misc

### 易混淆点

- **8.0 没有 `fts0vlc.cc`/`fts_vlc_encode`**——VLC 已内联进 `include/fts0vlc.ic`（`fts_encode_int`/`fts_decode_vlc`），凭 5.x 记忆写函数名必错。
- **VLC 与 protobuf varint 方向相反**：MySQL 大端 7-bit 分组 + **终止位**（最后字节高位 1）；protobuf 小端 + 继续位。
- **doc_id 是 delta、position 也是 delta**，但 `last_pos` **每个文档重置为 0**；node 首文档的 delta 就是 `first_doc_id` 本身。
- **分档对非 CJK 用"整词"排序权重**，不是首字符（查询路径里的单字节传参是 5.6 遗留）。
- **ilist 不可变**：删除在旁路三处（DELETED 表 / DELETED_CACHE 表 / cache 向量），查询过滤、optimize 才物理清除并**重新 delta 编码**。
- **sync 只追加不重写**；optimize 才重写行——两层维护的分工。

### 一句话总结

倒排索引的存储 = "不可变 delta 编码的 ilist + 旁路删除集合 + 追加式 node 轮转"三者叠加：写路径永远是"追加新 node 或追加新文档"，删除永远不碰 ilist 字节，直到后台 optimize 用"顺序扫描 + 重编码"把墓碑变沉默——整个设计围绕**避免改写已刷盘数据**这一约束展开。

---

## 参考

**论文**
- Zobel, J. & Moffat, A. *Inverted Files for Text Search Engines*. ACM Computing Surveys, 2006.（倒排结构的分类学：doc-id 编码、delta/gap 压缩、按 block 分档）

**官方文档**
- *MySQL 8.0 Reference Manual → 17.6.2.4 InnoDB Full-Text Indexes*（辅助表结构与 optimize）
- *MySQL 8.0 Reference Manual → 15.6.2.4 Fine-Tuning InnoDB Full-Text Search*（cache 变量）

**相关文档**
- 全文检索全链路（语法/优化器/打分/崩溃恢复）：[`../feat/fts.md`](../feat/fts.md)
- 倒排在索引类型谱系中的位置：[`types.md`](types.md)
- FTS 辅助表清单与 `FTS_DOC_ID`：[`types.md`](types.md) 全文索引一节
- 另一条非 B-tree 结构（R-tree）：[`rtree.md`](rtree.md)
- B-tree 存储结构（INDEX 辅助表内部也是 B-tree）：[`btr.md`](btr.md)
