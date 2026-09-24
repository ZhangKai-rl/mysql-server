# 索引访问优化：回表、覆盖索引、ICP、MRR

> 基于 MySQL 8.0.39 源码。本篇讲 **MySQL 怎么用索引取数据，以及为减少"回表"做了哪些优化**。
>
> **边界**：B-tree 的搜索与结构操作见 [`btr.md`](btr.md)；索引类型与存储结构见 [`types.md`](types.md)；AHI / change buffer 这类"引擎内透明加速"见 [`ahi.md`](ahi.md)、[`ibuf.md`](ibuf.md)（本篇第 0 节明确区分两类优化）；优化器的访问路径选择与代价模型见 [`../server/query/07_optimize/`](../server/query/07_optimize/)；取行主链见 [`../innodb/row_search.md`](../innodb/row_search.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [三层分工与 `row_prebuilt_t` 模板体系](#三层分工与-row_prebuilt_t-模板体系)
  - 读路径基础
    - [回表：二级索引 → 聚簇索引](#回表二级索引--聚簇索引)
  - 读路径优化（执行计划可见）
    - [覆盖索引（index-only scan）](#覆盖索引index-only-scan)
    - [ICP（Index Condition Pushdown）](#icpindex-condition-pushdown)
    - [MRR 与 DS-MRR](#mrr-与-ds-mrr)
  - 优化器层的特殊路径
    - [Skip Scan 与 Loose Index Scan](#skip-scan-与-loose-index-scan)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

二级索引记录只含"键列 + 主键列"。取不在索引里的列必须**回表**（按主键回聚簇索引取整行）。回表是二级索引访问最大的随机 IO 来源，本篇的优化——覆盖索引、ICP、MRR——本质上都在**消灭或缓解回表**：

| 优化 | 手段 |
|---|---|
| 覆盖索引 | 让查询根本不需要回表（列全在索引里） |
| ICP | 在回表**之前**就用索引列过滤掉不匹配的行 |
| MRR | 无法避免回表时，把 N 次随机 IO 变成近似顺序 IO |

### 用途

- 减少回表次数与随机 IO；
- 减少 InnoDB 记录 → MySQL 行格式的转换次数；
- 提前终止扫描（ICP 的 `ICP_OUT_OF_RANGE`、end_range 检查）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | ICP 引入；MRR 的 DS-MRR 实现 |
| 8.0 | Skip Scan（`skip_scan`，默认 on）；**降序索引**使反向扫描不再是唯一手段 |
| 8.0.39 现状 | ICP/MRR/skip_scan 均在 `optimizer_switch` 里，默认全 on |

---

## 理论基础

### 设计思想与权衡

**1. 两类"索引优化"必须分开看**（这是知识库分类的关键判据）：

| | **引擎内透明加速** | **执行计划可见的访问优化** |
|---|---|---|
| 成员 | AHI、change buffer | 覆盖索引、ICP、MRR、Skip Scan |
| 决策者 | 引擎自己（统计驱动） | server 优化器（代价驱动） |
| 执行计划 | 不可见 | 可见（`Using index condition` / `Using MRR` / `Using index`） |
| 开关 | `innodb_adaptive_hash_index` / `innodb_change_buffering` | `optimizer_switch` |
| 引擎感知 | 引擎专属 | 引擎只看到多次 `index_read`/`rnd_pos` |

**2. ICP 是"跨层反向调用"的罕见设计。** 优化器把一棵 **`Item` 条件树**交给引擎，引擎在遍历索引记录时**反向调用 SQL 层的 `Item::val_int()`** 求值（`innobase_index_cond`）。这打破了"引擎不依赖 server"的通常分层，换来的是"过滤发生在回表之前"。

**3. MRR 用"打乱输出顺序"换"IO 顺序性"。** 代价是**不能保序**——因此有 `ORDER BY`/`GROUP BY`/反向扫描时 DS-MRR 直接被排除（`HA_MRR_SORTED` 时退回默认实现）。

**4. 覆盖索引对二级索引是"弱化"的。** 二级索引记录没有 `DB_TRX_ID`，覆盖扫描只能做页级粗筛（`lock_sec_rec_cons_read_sees`），**不能回溯旧版本**——不确定时仍得回表。聚簇索引上的覆盖则能精确判定。

### 理论溯源

- ICP/MRR 属于经典的 **query processing 优化**：条件下推（predicate pushdown）与 **disk-sweep / batched key access**（Graefe, *Query Evaluation Techniques for Large Databases*, 1993 中对批处理与 IO 顺序性的讨论）。
- Skip Scan 是 **loose index scan** 家族（先枚举 distinct 前缀，再对每个前缀做范围扫描）。

### 算法与数据结构

- `row_prebuilt_t` 的列模板（`mysql_row_templ_t`）决定"取哪些列、从哪个索引取、ICP 用哪些列"——是所有访问优化的**公共基础设施**；
- `handler` 层提供 `idx_cond_push` / `multi_range_read_init` / `HA_EXTRA_KEYREAD` 三个接口；
- 引擎侧的三个判据：`need_to_access_clustered`（是否回表）、`read_just_key`（server 说这是覆盖）、`idx_cond`（是否有下推条件）。

---

## 核心实现

### 主线与基础构件

#### 三层分工与 `row_prebuilt_t` 模板体系

```
server 优化器：访问路径决策 / 代价估算 / 条件抽取
    QEP_TAB::push_index_cond()（sql/sql_select.cc:2920）
    DsMrr_impl::choose_mrr_impl()（sql/handler.cc:6966）
    TABLE::covering_keys（sql/table.cc:5445）
        ↓ handler API：idx_cond_push / multi_range_read_init / HA_EXTRA_KEYREAD / set_end_range
handler 层：默认 MRR、DS-MRR、range 边界比较、ICP 结果枚举
        ↓ row_prebuilt_t（模板 + idx_cond + clust_ref + clust_pcur）
InnoDB 引擎：row_search_mvcc()（row0sel.cc:4437）
    row_search_idx_cond_check()（row0sel.cc:3782）
    Row_sel_get_clust_rec_for_mysql::operator()（row0sel.cc:3136）
```

引擎侧判据在 `build_template()`（`ha_innodb.cc:8377-8675`）里算出：

```cpp
m_prebuilt->need_to_access_clustered = (index == clust_index);      // 初值
...
// build_template_field 内：
if (!index->is_clustered() && templ->rec_field_no == ULINT_UNDEFINED) {
  prebuilt->need_to_access_clustered = true;      // 模板里有一列在当前索引找不到 → 必须回表
}
if (dict_index_is_spatial(index)) {
  prebuilt->need_to_access_clustered = true;      // 空间索引强制回表
}
```

**判据一句话**：只要模板中有任何一列在 `prebuilt->index` 里找不到，就回表。

模板项的关键字段：

```cpp
struct mysql_row_templ_t {
  ulint col_no;                // 列号
  ulint rec_field_no;          // 在当前索引记录里的字段号（二级索引可能 = ULINT_UNDEFINED）
  ulint clust_rec_field_no;    // 在聚簇索引记录里的字段号
  ulint icp_rec_field_no;      // ICP 求值用的字段号（仅 ICP 列有定义）
  ...
};
```

模板类型：`ROW_MYSQL_WHOLE_ROW`（整行）/ `ROW_MYSQL_REC_FIELDS`（只取需要的列——覆盖索引的模式）。

#### handler 能力位体系：`index_flags`

优化器"能不能按序读 / 能不能只读索引列 / 能不能反向扫"全部由 `ha_innobase::index_flags()` 声明（`ha_innodb.cc:6364-6398`）：

```cpp
ulong ha_innobase::index_flags(uint key, uint, bool) const {
  if (table_share->key_info[key].algorithm == HA_KEY_ALG_FULLTEXT) {
    return (0);                              // 全文：不走 index_read/index_next 路径
  }
  ulong flags = HA_READ_NEXT | HA_READ_PREV | HA_READ_ORDER | HA_READ_RANGE |
                HA_KEYREAD_ONLY | HA_DO_INDEX_COND_PUSHDOWN;

  /* For spatial index, we don't support descending scan and ICP so far. */
  if (table_share->key_info[key].flags & HA_SPATIAL) {
    flags = HA_READ_NEXT | HA_READ_ORDER | HA_READ_RANGE | HA_KEYREAD_ONLY |
            HA_KEY_SCAN_NOT_ROR;
    return (flags);
  }
  /* For dd tables mysql.*, we disable ICP for them,
     it's for avoiding recursively access same page. */
  if (dbname && strstr(dbname, dict_sys_t::s_dd_space_name) != nullptr &&
      strlen(dbname) == 5) {
    flags = HA_READ_NEXT | HA_READ_PREV | HA_READ_ORDER | HA_READ_RANGE |
            HA_KEYREAD_ONLY;
  }
  /* Multi-valued keys don't support ordered retrieval, neither they're
     suitable for keyread only retrieval. */
  if (table_share->key_info[key].flags & HA_MULTI_VALUED_KEY) {
    flags &= ~(HA_READ_ORDER | HA_KEYREAD_ONLY);
  }
  return (flags);
}
```

逐项含义与"为什么关"：

| 标志 | InnoDB | 含义 | 关闭的场景 |
|---|---|---|---|
| `HA_READ_NEXT` | 全开（非 FTS） | 支持 `index_next` | 从不关（server 不检查，必须支持） |
| `HA_READ_PREV` | 开；**SPATIAL 关** | 反向扫描 `index_prev` | R-tree 无全序，不能反向 |
| `HA_READ_ORDER` | 开；**多值关** | 能按索引序产出（`part_of_sortkey` 用） | 多值一行多条索引项，顺序非索引序 |
| `HA_READ_RANGE` | 全开（非 FTS） | 支持范围优化 | — |
| `HA_KEYREAD_ONLY` | 开；**多值关** | 支持覆盖索引 | 多值索引项只是数组一个值，不能当完整列返回 |
| `HA_KEY_SCAN_NOT_ROR` | **仅 SPATIAL** | 扫描不按 rowid 序 → 不能 ROR | R-tree 多路下潜 |
| `HA_DO_INDEX_COND_PUSHDOWN` | 开；**SPATIAL / `mysql.*` 关** | 支持 ICP | 空间比较不可推；DD 表怕递归访问同页 |

**"覆盖索引"在引擎侧的真正落点不是 `HA_KEYREAD_ONLY`**——InnoDB 没有 `key_read` 成员（全仓零命中）。实际机制是 `build_template()` 里按需判断（`ha_innodb.cc:8319-8326`）：

```cpp
if (!index->is_clustered() && templ->rec_field_no == ULINT_UNDEFINED) {
  prebuilt->need_to_access_clustered = true;   // 需要的列不在二级索引里
}
if (dict_index_is_spatial(index)) prebuilt->need_to_access_clustered = true;
```

`row_search_mvcc` 里**不进 `requires_clust_rec` 分支 = 不读聚簇 = 覆盖索引**（`row0sel.cc:5477`）。

**handler 层其余关键接口**（详见各节）：`records_in_range`（索引下潜估算，0 结果 +1 防优化器误判空集）、`info()` 的 `HA_STATUS_CONST`（`records_per_key` 经 `key->supports_records_per_key()`，FTS/SPATIAL 跳过）、`index_read`（`key_ptr==nullptr` 定位首/尾；`HA_READ_KEY_EXACT` → `ROW_SEL_EXACT`）。

**KEY / KEY_PART 层的标志**（`my_base.h:521-574`）与引擎侧对应：`HA_NOSAME`→唯一、`HA_FULLTEXT`/`HA_SPATIAL`→`index_flags` 分派、`HA_REVERSE_SORT`（key_part 层）→ `dict_field_t::is_ascending`、`HA_PART_KEY_SEG`→前缀段、`HA_MULTI_VALUED_KEY`（索引级，`dd_table_share.cc:1263` 从 `field->is_array()` 设置）vs `HA_MULTI_VALUED_KEY_SUPPORT`（handlerton 级能力声明，`ha_innodb.cc:2981`）。

---

### 读路径基础

#### 回表：二级索引 → 聚簇索引

触发（`row_search_mvcc`，`row0sel.cc:5477`）：

```cpp
if (index != clust_index && prebuilt->need_to_access_clustered) {
requires_clust_rec:
  mtr_has_extra_clust_latch = true;
  err = row_sel_get_clust_rec_for_mysql(prebuilt, index, rec, thr, &clust_rec, &offsets,
                                        &heap, need_vrow ? &vrow : nullptr, &mtr, ...);
```

注意 `requires_clust_rec:` 这个标签**也被 ICP 分支 `goto` 过来**（二级索引记录不可见时必须回表取 undo）。

完整链路（`Row_sel_get_clust_rec_for_mysql::operator()`，`row0sel.cc:3136-3399`）：

**第 1 步：从二级索引记录构造主键元组**

```cpp
row_build_row_ref_in_tuple(prebuilt->clust_ref, rec, sec_index, *offsets);
```

`row_build_row_ref_in_tuple`（`row0row.cc:755`）用 `dict_index_get_nth_field_pos()` 做"聚簇第 i 个唯一列 → 二级索引第几个字段"的映射，浅拷贝指针；前缀列要按 `dtype_get_at_most_n_mbchars` 截断长度，否则类型不匹配会定位失败。

**第 2 步：用独立的 `clust_pcur` 以 `PAGE_CUR_LE` 打开聚簇索引**

```cpp
prebuilt->clust_pcur->open_no_init(clust_index, prebuilt->clust_ref, PAGE_CUR_LE,
                                   BTR_SEARCH_LEAF, 0, mtr, UT_LOCATION_HERE);
```

用独立 pcur 避免破坏扫描二级索引的游标位置；`PAGE_CUR_LE` 兼容"记录已被删除 / 页面分裂"的边界。

**第 3 步：定位失败不算错误**（purge 竞态），返回 `clust_rec == nullptr`，调用方跳过该行。

**第 4 步：可见性再判断 + 版本回溯**

```cpp
if (trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
    !lock_clust_rec_cons_read_sees(clust_rec, clust_index, *offsets, trx_get_read_view(trx))) {
  if (clust_rec != cached_clust_rec) {
    err = row_sel_build_prev_vers_for_mysql(trx->read_view, clust_index, prebuilt, clust_rec,
                                            offsets, offset_heap, &old_vers, vrow, mtr, lob_undo);
    cached_clust_rec = clust_rec;
    cached_old_vers = old_vers;
  } else {
    old_vers = cached_old_vers;                 // ★ 记忆化：省掉整条 undo 链遍历
    *offsets = rec_get_offsets(old_vers, clust_index, ...);   // offsets 可能不同，需重算
  }
  if (old_vers == nullptr) goto err_exit;       // 该行在 read view 里不存在
  clust_rec = old_vers;
}
```

`cached_clust_rec/cached_old_vers` 是 `row_search_mvcc()` 的**栈上局部变量**（functor 对象），因此记忆化的作用域是"一次 `row_search_mvcc` 调用内的多条记录"——典型受益场景是半一致读重试、同一主键被多次命中。

**第 5 步：一致性复核**

```cpp
if (clust_rec && (old_vers || ... || rec_get_deleted_flag(rec, ...))) {
  err = row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec, clust_index, thr, rec_equal);
  if (!rec_equal) clust_rec = nullptr;
}
```

用回表得到的聚簇记录反推它"本应该"对应的二级索引项，与 `rec` 比较；不等则丢弃——这是**二级索引不存 trx_id** 必做的校验。

**回表的代价**：一次 B-tree 下降 + 一次随机页访问（MRR 要解决的正是它）+ 额外 latch（`mtr_has_extra_clust_latch`）+ 可能的版本回溯 + 复核 + ICP 场景下的**二次格式转换**（见下）。

---

### 读路径优化（执行计划可见）

#### 覆盖索引（index-only scan）

**两层判定，两套机制：**

**(a) server 优化器**——`covering_keys` = 所有被读列所属索引集合的**交集**：

```cpp
// sql/table.cc:5439-5449（mark_columns）
case MARK_COLUMNS_READ: {
  Key_map part_of_key = field->part_of_key;
  part_of_key.merge(field->part_of_prefixkey);
  covering_keys.intersect(part_of_key);        // ★ 每读一列就收窄一次
  ...
}
```

命中则 `table->set_keyread(true)` → `HA_EXTRA_KEYREAD` → `prebuilt->read_just_key = 1`（`ha_innodb.cc:18382`）。

**(b) 引擎侧**——`build_template_needs_field()`（`ha_innodb.cc:8165-8219`）：

```cpp
if (!index_contains) {
  if (read_just_key) {
    return (nullptr);      // ★ server 说这是覆盖 → 不在索引里的列直接丢弃，不进模板 → 不回表
  }
} else if (fetch_all_in_key) {
  return (field);
}
if (bitmap_is_set(table->read_set, i) || bitmap_is_set(table->write_set, i)) {
  return (field);
}
```

**覆盖索引与 ICP 在 server 侧是互斥的**（`sql_select.cc:3326-3331`）：先试 `set_keyread(true)`，失败才 `push_index_cond()`。

**执行侧分支**：

```cpp
if (index != clust_index && prebuilt->need_to_access_clustered) {
  ... 回表 ...
} else {
  result_rec = rec;        // ← 覆盖：直接用索引记录
}
```

**二级索引覆盖 vs 聚簇覆盖的差异**：

| | 二级索引覆盖 | 聚簇索引覆盖 |
|---|---|---|
| 可见性判断 | `lock_sec_rec_cons_read_sees()`（页级粗筛），**不能回溯旧版本**，不确定时仍要回表 | `lock_clust_rec_cons_read_sees()` 可精确判定并可回溯 |
| ICP | 支持 | server 侧默认排除（`push_index_cond` 条件 6：`!(keyno == primary_key && primary_key_is_clustered())`） |
| 列值权威性 | 可能是前缀、大小写可能不准（源码注释 Bug #56680） | 权威 |

**失效条件**：访问索引外列（含 `SELECT *`）、前缀索引需完整值、`LOCK_X`（UPDATE/DELETE 当前读强制 whole_row → 一定回表）、空间索引（强制回表）。

**EXPLAIN**：`Using index`（`opt_explain.cc:1611`，JSON `using_index`）。

#### ICP（Index Condition Pushdown）

**完整链路三层：**

**(1) server 决策**：`QEP_TAB::push_index_cond()`（`sql/sql_select.cc:2920-3074`）。准入条件（源码注释 0-7）：

| # | 条件 |
|---|---|
| 0 | 表有 select condition |
| 1 | 引擎 `index_flags` 带 `HA_DO_INDEX_COND_PUSHDOWN` |
| 2 | `optimizer_switch=index_condition_pushdown=on` 且无 `NO_ICP` hint |
| 3 | 不是多表 UPDATE / DELETE |
| 4 | 不在带 guarded conditions 的子查询里 |
| 5 | 不是 `JT_CONST`/`JT_SYSTEM` |
| 6 | **不是聚簇主键**（`!(keyno == primary_key && primary_key_is_clustered())`——聚簇上 ICP 收益低） |
| 7 | 索引不含虚拟生成列 |

另：`reversed_access`（反向访问）与 InnoDB intrinsic 临时表也被排除。

**条件抽取**（`make_cond_for_index()` + `uses_index_fields_only()`）：
- AND 可**部分**下推，OR 必须**全部**可推；
- 能下推：字段属于该索引且非 GEOMETRY/BLOB、常量、普通函数（递归参数）、`REF_ITEM`；
- 不能下推：含存储程序、子查询、`TRIG_COND_FUNC`（外连接 NULL-complemented）、`DD_INTERNAL_FUNC`、GEOMETRY/BLOB 字段。

**(2) handler 接口**：`handler::idx_cond_push(keyno, idx_cond)` 返回"我不求值的那部分"。InnoDB 实现（`ha_innodb.cc:23824`）返回 `nullptr`——**全盘接受**：

```cpp
Item *ha_innobase::idx_cond_push(uint keyno, Item *idx_cond) {
  pushed_idx_cond = idx_cond;
  pushed_idx_cond_keyno = keyno;
  in_range_check_pushed_down = true;
  return nullptr;      // "We will evaluate the condition entirely"
}
```

ICP 结果枚举（`include/my_icp.h:35-49`）：`ICP_NO_MATCH` / `ICP_MATCH` / `ICP_OUT_OF_RANGE`。

**(3) 引擎执行**：`row_search_idx_cond_check()`（`row0sel.cc:3782-3862`）：

```cpp
if (!prebuilt->idx_cond) {
  return (ICP_MATCH);                        // ① 没下推条件：零开销短路
}
MONITOR_INC(MONITOR_ICP_ATTEMPTS);
for (i = 0; i < prebuilt->idx_cond_n_cols; i++) {                    // ② 只转换 ICP 需要的列
  const mysql_row_templ_t *templ = &prebuilt->mysql_template[i];
  if (templ->is_virtual) continue;
  if (!row_sel_store_mysql_field(mysql_rec, prebuilt, rec, prebuilt->index, prebuilt->index,
                                 offsets, templ->icp_rec_field_no, templ, ..., prebuilt->blob_heap)) {
    return (ICP_NO_MATCH);                   // 转换失败：保守丢弃
  }
}
result = innobase_index_cond(prebuilt->m_mysql_handler);             // ③ 反向调用 SQL 层 Item 求值
switch (result) {
  case ICP_MATCH:
    if (!prebuilt->need_to_access_clustered || prebuilt->index->is_clustered()) {
      row_sel_store_mysql_rec(...);          // ④ 不回表 → 此刻把剩余列也转换掉
    }
    return (result);
  case ICP_OUT_OF_RANGE:
    row_sel_get_record_buffer(prebuilt)->set_out_of_range(true);     // ⑤ 通知预取缓冲终止
    return (result);
}
```

求值回调（`ha_innodb.cc:23507-23522`）：

```cpp
ICP_RESULT innobase_index_cond(ha_innobase *h) {
  if (h->end_range && h->compare_key_icp(h->end_range) > 0) {
    return ICP_OUT_OF_RANGE;                 // ★ OUT_OF_RANGE 的真正来源是 end_range 越界
  }
  return h->pushed_idx_cond->val_int() ? ICP_MATCH : ICP_NO_MATCH;
}
```

**三个调用点**（在 `row_search_mvcc` 里）：

| 位置 | 场景 | 处理 |
|---|---|---|
| `row0sel.cc:4716` | AHI shortcut 快路径 | NO_MATCH/OUT_OF_RANGE → `shortcut_mismatch` |
| `row0sel.cc:5401` | 二级索引记录**对 read view 不可见**时 | **先用 ICP 过滤**：不匹配 → 直接跳过，**省掉"为做可见性判断而回表"**；匹配 → `requires_clust_rec` |
| `row0sel.cc:5462` | 主扫描循环（删除标记检查后、回表前） | 不匹配 → `try_unlock` + `next_rec`；越界 → `DB_RECORD_NOT_FOUND` |

**ICP 与 MVCC 的顺序**：定位记录 → 可见性判断（聚簇可回溯到 `old_vers`）→ 删除标记检查 → **ICP** → 回表 → 二次转换。

**副作用**：ICP 改变了格式转换时机，因此 `row0sel.cc:5642` 的预取缓存只在 `!prebuilt->idx_cond` 时启用；且 ICP 求值用的是二级索引里的值（可能是前缀/大小写不准），回表后必须用聚簇记录**重新全量转换覆盖**（`row0sel.cc:5563`）。

**引擎内的额外限制**（`ha_innodb.cc:8524-8542` 注释）：聚簇索引上 ICP **只能用于 PRIMARY KEY 列**，因为其它列可能 off-page，ICP 前不会拉取外部列。空间索引不支持 ICP（`index_flags` 无 `HA_DO_INDEX_COND_PUSHDOWN`）。

**`end_range` 与三个比较函数**（ICP 越界判定的基础设施）：

```cpp
// sql/handler.h:4490-4500
key_range *end_range;        // ★ 指针：当前生效的 range 上界；nullptr = 无上界/无进行中的 range 扫描
key_range save_end_range;    // 值：end_range 指向的存储体
```

| 函数 | 位置 | 用途 |
|---|---|---|
| `handler::compare_key(key_range*)` | `handler.cc:7479` | `read_range_first/next` 用；`in_range_check_pushed_down` 为真时**直接返回 0**（引擎自己判） |
| `handler::compare_key_icp(const key_range*)` | `handler.cc:7514` | **ICP 专用**；处理降序扫描（`RANGE_SCAN_DESC` 取反），被 `innobase_index_cond()` 调用 |
| `handler::compare_key_in_buffer(const uchar*)` | `handler.cc:7549` | 比较**非 `record[0]` 缓冲区**里的行（预取 buffer），被 `row_search_end_range_check()` 调用 |

`set_end_range()`（`handler.cc:7436`）里还有一句关键副作用：

```cpp
if (m_record_buffer != nullptr) {
  m_record_buffer->set_out_of_range(false);
  in_range_check_pushed_down = true;     // ★ 有预取缓冲时，把上界检查下推给引擎
}
```

引擎侧对应 `prebuilt->m_end_range`（bool：填充预取缓存时是否已越界）与 `prebuilt->m_stop_tuple`（dtuple，由 `end_range->key` 转换而来，`row0sel.cc:10264`）。`row_search_end_range_check()`（`row0sel.cc:3880`）判越界后给 `Record_buffer` 打标记，避免继续填充无用行。

**EXPLAIN**：`Using index condition`（判据就是 `handler::pushed_idx_cond != nullptr` 且 keyno 匹配，`opt_explain.cc:999`）。

#### MRR 与 DS-MRR

**核心设计：两个 handler**。`DsMrr_impl` 克隆出一个 `h2` 专门扫描二级索引，原来的 `h` 转成 RND 按 rowid 取全行——**索引扫描与按 rowid 取行解耦**。

**`dsmrr_fill_buffer()` 是"随机 IO 变顺序 IO"的全部秘密**（`sql/handler.cc:6767-6828`）：

```cpp
table->key_read = true;                                   // 只读索引列
while ((rowids_buf_cur < rowids_buf_end) &&
       !(res = h2->handler::multi_range_read_next(&range_info))) {
  h2->position(table->record[0]);                         // 取当前记录对应的 rowid（InnoDB = 主键打包）
  memcpy(rowids_buf_cur, h2->ref, h2->ref_length);
  rowids_buf_cur += h2->ref_length;
  if (is_mrr_assoc) { ... 附带 range_id ... }
}
table->key_read = table_keyread_save;

qsort(rowids_buf, (rowids_buf_cur - rowids_buf) / elem_size, elem_size,
      [](const void *a, const void *b) {
        return current_handler->cmp_ref(...);             // ★ 按 rowid 排序
      });
```

之后 `dsmrr_next()` 按排序顺序 `h->ha_rnd_pos()` 取行——聚簇索引上的访问序列单调递增，物理上近似顺序扫。

**init 时的关键动作**（`dsmrr_init`，`sql/handler.cc:6550-6725`）：
- `mrr=on` 且**不要求有序输出**才用 DS-MRR，否则退回默认实现；
- 克隆 `h2` → 打开索引扫描 + `HA_EXTRA_KEYREAD` → **把 ICP 条件"转移"到 h2**（`h2->idx_cond_push(...)`）——这是 MRR 与 ICP 能叠加的关键；
- `h` 转 RND。

**代价与开关**（`choose_mrr_impl`，`sql/handler.cc:6966-7044`）。硬性排除 DS-MRR：

1. `mrr=off` 且无 `MRR()`/`BKA()` hint；
2. `HA_MRR_INDEX_ONLY`（覆盖索引，已不回表）或 `HA_MRR_SORTED`（DS-MRR 不能保序）；
3. 索引就是聚簇主键（本来就是主键序）；
4. 前缀索引（`key_uses_partial_cols()`，rowid 不完整）；
5. 临时表。

`mrr_cost_based` 的三条启发式：表大小 > buffer pool（无信息时按 100MB）、预计读取 > 50 行、代价不高于默认实现。

**InnoDB 侧**：`ha_innobase::multi_range_read_*` 直接复用 server 的通用 `DsMrr_impl`（`ha_innodb.cc:23470-23500`），`ha_rnd_pos()` 落到 `row_search_mvcc(buf, PAGE_CUR_WITHIN, ...)`。

**代价模型**（`get_disk_sweep_mrr_cost`，`handler.cc:7063-7126`）——决定"排序 + 扫掠"值不值：

```cpp
const uint elem_size   = h->ref_length + sizeof(void *) * !(flags & HA_MRR_NO_ASSOCIATION);
const ha_rows max_buff_entries = *buffer_size / elem_size;      // 缓冲能装多少 rowid
n_full_steps   = (uint)floor(rows2double(rows) / max_buff_entries);   // 需要几轮满缓冲
rows_in_last_step = rows % max_buff_entries;

if (n_full_steps) {
  get_sort_and_sweep_cost(table, max_buff_entries, cost);       // 一轮 = qsort + 顺序读
  cost->multiply(n_full_steps);
} else {
  /* 只有不满一轮：按 1.2 倍 + 至少 100 条调整缓冲，避免为少量行分配过大缓冲 */
  const ha_rows keys_in_buffer = max<ha_rows>(1.2 * rows_in_last_step, 100);
  *buffer_size = min<ulong>(*buffer_size, keys_in_buffer * elem_size);
}
get_sort_and_sweep_cost(table, rows_in_last_step, &last_step_cost);
(*cost) += last_step_cost;
cost->add_mem(*buffer_size);
(*cost) += h->index_scan_cost(keynr, 1, rows);                  // 索引扫描本身的代价
cost->add_cpu(table->cost_model()->row_evaluate_cost(rows));     // 行求值 CPU
```

即：**代价 = 若干轮（qsort + sweep read） + 索引扫描 + CPU 行求值**。注意它**不**计入"少做的随机 IO 省下的代价"之外的收益——所以 `mrr_cost_based=ON` 时只有大表才会选它。

**EXPLAIN**：`Using MRR`（判据：`INDEX_RANGE_SCAN` 且最终 flags 不含 `HA_MRR_USE_DEFAULT_IMPL`）。

---

### 优化器层的特殊路径

#### Skip Scan 与 Loose Index Scan

**Skip Scan** 解决"复合索引前缀无等值条件"：把前缀的 **distinct 取值**枚举出来，对每个取值做一次 range 扫描。

适用条件（`index_skip_scan_plan.cc:117-163` 注释）：复合索引 `I = <A_1..A_k, B_1..B_m, C>`、单表、无 GROUP BY/DISTINCT、**查询只引用索引里的列（必须配合覆盖索引）**、`A` 上是等值常量、`C` 上有 range 条件、合取式。

执行核心（`IndexSkipScanIterator::Read()`）：

```cpp
// ① "skip" 动作：跳到下一个不同的前缀
result = index_next_different(false, table()->file, index_info->key_part,
                              table()->record[0], distinct_prefix,
                              distinct_prefix_len, distinct_prefix_key_parts);
// ② 对每个 distinct prefix 拼出 start/end key 走标准 range 扫描
memcpy(min_search_key, distinct_prefix, distinct_prefix_len);
memcpy(min_search_key + distinct_prefix_len, min_range_key, range_key_len);
...
result = table()->file->ha_read_range_first(&start_key, &end_key, ..., true);
```

**引擎完全无感**——它只看到多次 `ha_read_range_first/next`。

**Loose Index Scan（group-by）**：`get_best_group_min_max()`，用于 `GROUP BY`/`MIN/MAX` 只扫每组首尾，EXPLAIN 显示 `Using index for group-by`（可带 `scanning`）。与 Skip Scan 同属 `is_loose_index_scan()` 家族，但**没有独立开关**（`optimizer_switch` 里的 `loosescan` 是 semi-join 策略，不是这个）。

---

## 相关的系统变量/状态变量

| 变量（optimizer_switch 项） | 默认值 | 说明 |
|---|---|---|
| `index_condition_pushdown` | **on** | ICP 总开关（`1ULL<<5`）；hint `NO_ICP`/`ICP` |
| `mrr` | **on** | MRR 总开关；hint `MRR()`/`NO_MRR`、`BKA()` |
| `mrr_cost_based` | **on** | 是否按代价决定用不用 DS-MRR（关掉=强制启用） |
| `skip_scan` | **on** | Skip Scan（`OPTIMIZER_SKIP_SCAN = 1ULL<<20`）；hint `SKIP_SCAN()` |
| `use_invisible_indexes` | off | 调试用，让优化器看见不可见索引 |

InnoDB 侧可观测：`MONITOR_ICP_ATTEMPTS` / `MONITOR_ICP_MATCH` / `MONITOR_ICP_NO_MATCH` / `MONITOR_ICP_OUT_OF_RANGE`（通过 `innodb_monitor_enable` 打开）。

---

## Misc

### 优化机制总表

| 机制 | 目的 | 决策层 | 核心函数 | EXPLAIN | 开关 |
|---|---|---|---|---|---|
| 回表 | 取完整行 | 引擎（`need_to_access_clustered`） | `Row_sel_get_clust_rec_for_mysql::operator()` | **不可见** | 无 |
| 回表记忆化 | 省重复版本回溯 | 引擎 | `cached_clust_rec`/`cached_old_vers` | 不可见 | 无 |
| 覆盖索引 | 消除回表 | 优化器 + 引擎双向 | `TABLE::covering_keys`、`build_template_needs_field` | `Using index` | 受 `no_keyread` 影响 |
| ICP | 回表前过滤 | 优化器决策 / 引擎求值 | `push_index_cond`、`row_search_idx_cond_check` | `Using index condition` | `index_condition_pushdown` |
| MRR/DS-MRR | 随机 IO → 顺序 IO | 优化器（代价） | `DsMrr_impl::dsmrr_fill_buffer`（qsort by rowid） | `Using MRR` | `mrr`、`mrr_cost_based` |
| Skip Scan | 前缀无等值仍用索引 | 优化器 | `get_best_skip_scan`、`index_next_different` | `Using index for skip scan` | `skip_scan` |
| Loose Index Scan | GROUP BY 只扫组边界 | 优化器 | `get_best_group_min_max` | `Using index for group-by` | 无 |

### 易混淆点

- **ICP 的 `ICP_OUT_OF_RANGE` 来自 `end_range` 越界**，不是下推条件本身判出"超出范围"。
- **覆盖索引与 ICP 在 server 侧互斥**（先试 keyread，失败才 push ICP）——所以 EXPLAIN 里不会同时出现 `Using index` 和 `Using index condition`。
- **ICP 在聚簇索引上被 server 默认排除**，只有二级索引（和二级索引覆盖扫描）才有。
- **DS-MRR 不能保序**，一有 `HA_MRR_SORTED`（ORDER BY/GROUP BY/反向扫描）就退回默认实现。
- **MRR 与 ICP 能叠加**：`dsmrr_init` 会把 ICP 条件转移到克隆的 `h2` 上。
- `end_range`（指针，可空）与 `save_end_range`（值，存储体）是两回事；InnoDB 侧对应 `prebuilt->m_end_range` 与 `m_stop_tuple`。

### 一句话总结

回表是二级索引访问的固有成本；覆盖索引**消灭**它，ICP 在**回表前**过滤掉不需要回表的行，MRR 把无法避免的回表**重排成顺序 IO**；三者都由 server 优化器决策、在执行计划里可见——这与 AHI/change buffer 那种引擎内透明加速是完全不同的两类优化。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 10.2.1.6 Index Condition Pushdown Optimization*
- *MySQL 8.0 Reference Manual → 10.2.1.11 Multi-Range Read Optimization*
- *MySQL 8.0 Reference Manual → 10.2.1.16 ORDER BY Optimization*（覆盖索引与排序消除）
- *MySQL 8.0 Reference Manual → 10.2.1.17 GROUP BY Optimization*（loose index scan）
- *MySQL 8.0 Reference Manual → 8.9.3 Optimizer Hints*（ICP/MRR/SKIP_SCAN/NO_ICP）

**论文**
- Graefe, G. *Query Evaluation Techniques for Large Databases*. ACM Computing Surveys, 1993.（批处理、IO 顺序性、Disk-Sweep 的理论背景）

**相关文档**
- B-tree 搜索与结构：[`btr.md`](btr.md)
- 索引类型与存储结构：[`types.md`](types.md)
- 索引承载的约束：[`constraint.md`](constraint.md)
- 引擎内透明加速：AHI [`ahi.md`](ahi.md)、change buffer [`ibuf.md`](ibuf.md)
- 优化器访问路径与代价：[`../server/query/07_optimize/`](../server/query/07_optimize/)
- 取行主链 `row_search_mvcc`：[`../innodb/row_search.md`](../innodb/row_search.md)
