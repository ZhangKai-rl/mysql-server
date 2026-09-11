# 10 计划改进（Plan Refinement）

> 物理优化**之后**的收尾：Ordering index 选择（避免排序）、条件最终化、访问方法 setup、引擎下推、临时表与排序。
>
> 分工：**ICP 的 SQL 层决策**在本篇，**ICP 的 handler 接口与 InnoDB 实现**在 [05 篇 4 节](logical/05_logical_predicate.md)。

## 目录

- [零、执行顺序与三条依赖链](#零执行顺序与三条依赖链)
- [一、Ordering index 选择（避免排序）](#一ordering-index-选择避免排序)
- [二、条件最终化与访问方法 setup](#二条件最终化与访问方法-setup)
- [三、push_to_engines：两层下推](#三push_to_engines两层下推)
- [四、setup_join_buffering](#四setup_join_buffering)
- [五、临时表与排序 make_tmp_tables_info](#五临时表与排序-make_tmp_tables_info)

---

## 零、执行顺序与三条依赖链

| # | 行号 | 调用 | 阶段 |
|---|---|---|---|
| 0 | `:696` | `make_join_plan()` | ③（前置） |
| 1 | `:736-768` | `substitute_for_best_equal_field` | ① |
| 2 | `:770` | `init_ref_access()` | |
| 3 | **`:781`** | **`make_join_query_block`**（条件最终化 + 二次 range 重估） | ③ |
| 4 | **`:796`** | **`optimize_distinct_group_order`** | ① |
| 5 | `:894-906` | `setup_join_buffering` | ③ |
| 6 | `:975-1007` | `need_tmp_before_win` 判定 | |
| 7 | `:1011` | `alloc_qep` | |
| 8 | **`:1015`** | **`test_skip_sort`**（Ordering index） | ④ |
| 9 | **`:1017`** | **`finalize_table_conditions`** | ④ |
| 10 | **`:1020`** | **`make_join_readinfo`**（keyread + ICP） | ④ |
| 11 | **`:1023`** | **`make_tmp_tables_info`** | ④ |
| 12 | `:1037` | `create_access_paths` | 代码生成 |
| 13 | **`:1064`** | **`push_to_engines`** | ④ |
| 14 | `:1067` | `set_plan_state(PLAN_READY)` | 冻结 |

### 三条关键依赖链

1. **条件归属 → ORDER/GROUP 化简 → join buffer → 有序索引**
   `make_join_query_block` → `optimize_distinct_group_order` → `setup_join_buffering` → `test_skip_sort`。
   **join buffer 与有序索引互斥**（见 1.3），所以 buffering 决策必须早于 test_skip_sort。
2. **alloc_qep → 定索引 → 裁条件 → keyread/ICP**
   `alloc_qep` → `test_skip_sort` → `finalize_table_conditions` → `make_join_readinfo`。
   **ICP 必须在条件裁剪之后**，否则下推的可能是马上要被删掉的冗余谓词。
3. **临时表/Filesort → AccessPath → 引擎改写**
   `make_tmp_tables_info` → `create_access_paths` → `push_to_engines`。中间不能插入 Iterator 创建。

---

## 一、Ordering index 选择（避免排序）

核心问题：**用某个索引的天然顺序 vs filesort 排序，哪个便宜？**

### 1.1 test_if_order_by_key（`sql_optimizer.cc:1795`）

判定"索引 idx 的天然顺序能否满足 ORDER/GROUP 列表"。

> ⚠️ **返回值语义纠正常见误解**：返回的是**扫描方向**（`+1` 正向可用 / `-1` 需反向可用 / `0` 不可用），**不是"用了几个 keypart"**。后者通过出参 `*used_key_parts` 返回，且**可能大于** `user_defined_key_parts`（PK 后缀扩展时）。

**逐段算法**：

**① 常量 keypart 跳过**——这就是 `WHERE a=1 ORDER BY a,b` 中 a 能被跳过的原因：

```cpp
key_part_map const_key_parts = table->const_key_parts[idx];
for (; order; order = order->next, const_key_parts >>= 1) {
  ...
  for (; const_key_parts & 1 && key_part < key_part_end &&
         (order_src->is_const_optimized() || key_part->field != field);
       const_key_parts >>= 1) {
    key_part++;
  }
```

- `const_key_parts` 是**位掩码**（`table->const_key_parts[idx]`），第 i 位为 1 表示 keypart i 在 WHERE 中已判定为常量。由 `update_const_equal_items`（`sql_executor.cc:466`）填充。
- 前面的 `JOIN::remove_const()`（`:10175`）已把 ORDER BY 中的常量项删掉并标记 `const_optimized=true`，所以进入本函数时 ORDER 链表通常只剩 `b`。
- **本质**：`a` 在结果集中只有单一取值，索引里 `a` 这一层的排序无意义，可直接"下沉"到 `b` 层——`(a,b)` 索引在 `a=const` 的横截面上就是按 `b` 有序的。

**② 排序项必须是 Item_field**：`real_item()->type() != Item::FIELD_ITEM` → 返回 0。`ORDER BY a+1`、`ORDER BY f(a)` 都失败。

**③ 聚簇主键后缀扩展**（`use_index_extensions`，默认 **on**）：

```cpp
if (key_part == key_part_end) {   // 二级索引 keypart 用完，ORDER 还有剩余
  if (!on_pk_suffix &&
      (table->file->ha_table_flags() & HA_PRIMARY_KEY_IN_READ_INDEX) &&  // InnoDB 设此位
      table->s->primary_key != MAX_KEY && table->s->primary_key != idx) {
    on_pk_suffix = true;
    key_part = table->key_info[table->s->primary_key].key_part;   // 继续用 PK 匹配
```

InnoDB 二级索引叶子隐含 PK 列作为后缀，所以 `INDEX(a)` 实际等价于 `(a, pk1, ...)`——`ORDER BY a, id`（id 为 PK）即使 a 非唯一也能靠扩展部分保证确定性顺序。

**④ 字段匹配与方向判定**：

```cpp
if (key_part->field != field || !field->part_of_sortkey.is_set(idx)) return 0;
if (order->direction != ORDER_NOT_RELEVANT) {
  const enum_order keypart_order =
      (key_part->key_part_flag & HA_REVERSE_SORT) ? ORDER_DESC : ORDER_ASC;
  int cur_scan_dir = (order->direction == keypart_order) ? 1 : -1;
  if (reverse && cur_scan_dir != reverse) return 0;    // ★ 必须同向
  reverse = cur_scan_dir;
}
```

- `part_of_sortkey` 只给**完整 keypart 且引擎支持 `HA_READ_ORDER`** 的列置位（`table.cc:764`）——前缀索引列、R-Tree 不进，故不能用于免排序。
- **一致性检查**：所有排序项必须同向。`ORDER BY a ASC, b DESC` 在单一方向上无法满足（除非用 8.0 的降序索引 `INDEX(a ASC, b DESC)`，此时 `HA_REVERSE_SORT` 让 `b DESC` 反而对应 `cur_scan_dir = +1`）。
- **混合升降序 + 反向 → 禁用 range**：`if (mixed_order && reverse < 0) *skip_quick = true;`——多 range 反向扫描需要重排 range 列表，混合索引无法简单重排。

**⑤ 反向能力校验**：`index_flags(idx, n-1, true) & HA_READ_PREV`。PK 扩展场景要求二级索引和 PK **都**支持。

### 1.2 test_if_cheaper_ordering（`sql_select.cc:5081`）—— 核心

**基准**：`read_time = tab->position()->read_cost`（**当前已选访问方法的代价**）。
`fanout` = 本表之后所有表 `rows_fetched × filter_effect` 之积（本表 1 行在最终结果里膨胀成多少行）。

**select_limit 的三重缩放**（完整语义）：

**① group 分支**——GROUP BY 时输出结果 1 行 = 1 个 group，而产出 1 个 group 平均要读 `rec_per_key(group 列)` 条索引项：

```cpp
if (select_limit > table_records / rec_per_key) select_limit = table_records;
else select_limit = (ha_rows)(select_limit * rec_per_key);
```
上限保护：若 `L > 组数（table_records/rec_per_key）` 说明要全表。

**② fanout 分支**——本表 1 行 → 后面表膨胀 fanout 行，所以拿到最终结果前 L 行只需读 `L/fanout` 行：

```cpp
if (fanout == 0) select_limit = HA_POS_ERROR;      // 除零 → 'infinite'
else if (fanout >= 0) select_limit = max(select_limit / fanout, 1.0);
```
`fanout < 0` 表示"未知"（段 1 的 break），**不缩放**（保守）。

**③ refkey_rows_estimate 分支**——假设排序索引与当前 ref/range 条件不相关：

```cpp
if (select_limit > refkey_rows_estimate) select_limit = table_records;
else select_limit = select_limit * table_records / refkey_rows_estimate;
```
沿排序索引走时，平均每 `table_records / refkey_rows_estimate` 条记录才有 1 条满足当前 ref 条件。

> **实现细节**：`select_limit` 是**按值传入的形参**，循环内就地改写且**不重置**——第 2 个及以后的候选索引，缩放起点沿用上次结果。这只是用于代价比较，真正返回的 `best_select_limit` 只在 `best_key` 更新时记录。

**索引顺序扫描代价公式**（`:5243`）：

```cpp
rec_per_key = keyinfo->records_per_key(keyinfo->user_defined_key_parts - 1);
const double index_scan_time =
    select_limit / rec_per_key *
    min<double>(table->file->page_read_cost(nr, rec_per_key),
                table->file->table_scan_cost().total_cost());
```

**推导**：
1. `rec_per_key` 是**整个索引**的——一个完整 key value 平均对应 `rec_per_key` 行
2. 这 `rec_per_key` 行在索引里**连续存放**（同 key 值内按 rowid/PK 有序），回表视作"一次连续小簇"
3. 扫 `select_limit` 条索引项 ⇒ `select_limit / rec_per_key` 个"回表簇访问"
4. 每簇最坏读 `rec_per_key` 个不同 page，但**整簇代价不可能超过全表扫一次** → 取 `min(...)`

**切换决策**（`:5256`）：

```cpp
if (((cur_access_method == JT_ALL || cur_access_method == JT_INDEX_SCAN) &&
     (is_covering || group || table->force_index_order)) ||
    index_scan_time < read_time) {
```

- **无条件切换**（不看代价）：当前是全表扫/索引扫，且候选是覆盖索引 / GROUP BY / `FORCE INDEX FOR ORDER BY`。理由：反正是全扫，换成能提供顺序的（覆盖）索引纯赚（省掉 filesort）
- **比代价**：其余情况要求 `index_scan_time < read_time`

> **比较基准是什么**：`read_time` 是**读**的代价，不含 filesort。filesort 代价在计划生成阶段已通过 `consider_plan()`（`sql_planner.cc:2496`）计入 `join->best_read`——**且只在"排序表 ≠ 第一个非 const 表"时才加**（若排序表就是首表，`sort_cost` 保持 0）。所以这里的语义是"用有序索引读 vs 用当前方法读 + 已记账的 filesort"。

**平局打破**（`:5274-5288`）：

| 情况 | 规则 |
|---|---|
| `select_limit <= min(quick_records, best_records)`（都够用） | 选 **keypart 更少**（key 更短）的 |
| 否则 | 选 **`quick_records` 更小**（选择性更好）的 |
| 附加 | **正扫优于反扫**（`direction > best_key_direction`），即使 key 更长 |

### 1.3 test_if_skip_sort_order（`sql_optimizer.cc:2193`）—— 编排

**流程**：

1. **早退**：`JT_EQ_REF/JT_CONST/JT_SYSTEM`（最多 1 行）→ 直接 true
2. **usable_keys** = 所有排序字段 `part_of_sortkey` 位图的**交集**；`JT_REF_OR_NULL` / `JT_FT` 直接拒绝
3. **当前索引不可用 → 找 subkey**：`test_if_subkey`（`:2072`）找以原 ref_key 的 keyparts 为**前缀**、且顺序匹配、key_length 最小的索引。ref 场景只置 `set_up_ref_access_to_key`（推迟改），**range 场景必须重跑 `test_quick_select`** 重建 QUICK
4. **代价搜索**：`prefer_ordering_index`（默认 **on**）或 FORCE INDEX 时调 `test_if_cheaper_ordering`
5. **多表反向保护**（见下）
6. **改写访问方法**

**多表 join 的反向保护**（`:2477-2492`）——为什么 filesort + join cache 通常更快：

```cpp
if (!is_force_index && (select_limit >= table_records) &&
    (tab->type() == JT_ALL && join->primary_tables > join->const_tables + 1) &&
    ((unsigned)best_key != table->s->primary_key ||
     !table->file->primary_key_is_clustered())) {
  can_skip_sorting = false;
```

四个条件同时成立才放弃有序索引：① 无 FORCE INDEX；② **没有能提前终止的 LIMIT**（`select_limit >= table_records`）；③ **多表 join 且首表是全表扫**；④ 选出的索引不是聚簇 PK。

**理由**：走有序索引意味着首表必须按索引顺序出 row；但内层表若用 join cache（BNL），cache 要求先批量攒够外层行再一次探测内层——**两者互斥**。而 filesort 只在最后排一次，内层全程可用 join cache 批量探测，IO 上通常远省。聚簇 PK 例外：按聚簇 PK 读 = 天然顺序 IO，不比全表扫差。

**改写访问方法的具体动作**（`:2591-2717`）：

| 场景 | 动作 |
|---|---|
| 转索引扫描 | `ref().key=-1`、`set_index(best_key)`、`set_type(JT_INDEX_SCAN)`、`set_range_scan(nullptr)`、`ha_index_or_rnd_end()` |
| 转 range | `set_type(calc_join_type(range_scan))`、`use_quick=QS_RANGE`；loose index scan 还要置 `precomputed_group_by=true`（故后面必须重跑 `count_field_types`） |
| ref 换 key | `create_ref_for_key()` **从零重建** `ref()`（两个索引的 search tuple 可能不同）；可能再经 `can_switch_from_ref_to_range` 转 range |
| 覆盖性回退 | 新索引非覆盖 → `set_keyread(false)`（撤销 "Using index"） |
| Temptable 临时表 | 不支持 `index_first()/index_last()` → 不建 index scan |

**反向三态**：`make_reverse()`（range）/ `reversed_access = true`（ref、index scan）。**反向时故意把 `changed_key` 设成当前 key**——后面 `push_index_cond`（`sql_select.cc:2928`）会因 `reversed_access` 直接 return，即**反向访问不做 ICP**（组合有性能回退）。

**rows_fetched 修正**（`:2727`）：成功且是 index scan 且有 LIMIT 时，`rows_fetched` 改为 `select_limit`（EXPLAIN rows 更真实）。

### 1.4 test_skip_sort（`sql_optimizer.cc:1642`）

**为什么只对第一个非 const 表判断**：

1. 代码硬约束：`JOIN_TAB *const tab = best_ref[const_tables];` + `assert(tab->idx() == const_tables)`
2. **算法原因**：嵌套循环 join 的**输出行序由最外层决定**。只有第一个非 const 表按索引顺序出 row，整个 join 输出才可能有序。第 2 个及以后的表即使用有序索引，其顺序也会被外层表的循环"打散"
3. 额外约束：即使首表有序，后续任何表用 join cache 仍会打乱行序——所以 `JOIN::optimize:907-915` 在 buffering 后清 `simple_order/simple_group`

**GROUP BY 分支 vs ORDER BY 分支**：

| | GROUP BY | ORDER BY |
|---|---|---|
| 可跳过性 | `simple_group && !select_distinct` | `simple_order \|\| skip_sort_order`，且 `!m_windows_sort` |
| 外层开关 | `SELECT_BIG_RESULT`/`with_json_agg`（否则没必要强行避免物化） | 无 |
| limit | `need_tmp_before_win ? HA_POS_ERROR : m_select_limit` | `m_select_limit` |
| Key_map | `keys_in_use_for_group_by` | `keys_in_use_for_order_by` |
| 失败后果 | 强制临时表但**不排序**（`simple_order=simple_group=false`） | 走 filesort |

> `simple_group`/`simple_order` 来自 `JOIN::remove_const()`（`:10175`）：排序/分组只引用首个非 const 表的字段，且该表**不是 LEFT JOIN 内表**（否则 NULL 补行破坏顺序）。

`m_ordered_index_usage` 枚举（`sql_optimizer.h:395`）：`ORDERED_INDEX_VOID` / `ORDERED_INDEX_GROUP_BY` / `ORDERED_INDEX_ORDER_BY`。消费者是 `make_tmp_tables_info:4759`（决定是否加 filesort）。

---

## 二、条件最终化与访问方法 setup

### 2.1 finalize_table_conditions（`sql_optimizer.cc:9177`）

两步算法：

**① `reduce_cond_for_table`（`:9042`）——删掉被 ref 保证为真的谓词**

`test_if_ref`（`:8987`）的判定链：

```
1. 访问类型不是 JT_REF_OR_NULL（它语义是 x=y OR x IS NULL，比 x=y 弱）
2. part_of_refkey()：在 tab->ref() 的 search tuple 中找到与 field 对应的 item
3. ref_item->eq(right_item, true)：谓词右侧与 ref 查找键是同一个表达式
4. ref_lookup_subsumes_comparison()：确认 ref 查找语义蕴含该比较（含 NULL 处理）
   → 判定冗余 ⇒ 整个 EQ 谓词被移除
```

**典型收益**：`t1 JOIN t2 ON t2.a=t1.a WHERE t2.a = 5`，t2 用 ref 访问 `a=t1.a`，条件 `t2.a = <ref key>` 在取到行后必然为真 → 删掉，省一次每行求值。

**AND/OR/TRIG_COND 的处理**：AND 逐参数递归（恒真则 remove）；OR 任一参数恒真则整体恒真；**TRIG_COND(FOUND_MATCH)** 把 inner tables 并入 `null_extended`（这些表可能被 NULL 扩展，其上的谓词**不能**被消除）。

**② `cache_const_expr`**——用 `Item::compile()` 的 analyzer+transformer 双遍机制，把常量子表达式包进 `Item_cache`，只在首次求值时算一次。同时作用于每个表条件和 HAVING。

### 2.2 make_join_readinfo（`sql_select.cc:3282`）

按 `QEP_TAB::type()` 分派：

| 类型 | 动作 |
|---|---|
| `JT_EQ_REF/REF/REF_OR_NULL/SYSTEM/CONST` | 覆盖索引（含聚簇 PK）→ `set_keyread(true)`；**否则** `push_index_cond` |
| `JT_ALL` | `set_status_no_index_used()`；EXPLAIN 展示修正（见下） |
| `JT_INDEX_SCAN` | EXPLAIN 展示修正 |
| `JT_RANGE` | 先试 `set_keyread(true)`；**若没进覆盖读则再试** `push_index_cond` |
| `JT_INDEX_MERGE` | 不做（used_index 为 MAX_KEY） |
| `JT_FT` | `fts_index_access` 成功 → `set_keyread(true)` |

**覆盖索引优先于 ICP 的原因**：覆盖读时已经不回表，ICP 省无可省。

**EXPLAIN 展示修正**（`JT_ALL`/`JT_INDEX_SCAN`/range）：计划阶段的 `rows_fetched` 是"扣掉常量条件后要读的行数"，对用户不直观；这里改回**真实扫描行数** `stats.records`，并把常量条件的过滤效果**搬回** `filter_effect`：

```cpp
tab->position()->filter_effect *= (float)(rows_w_const_cond / tab->position()->rows_fetched);
```
保证 `rows × filtered` 不变。

**覆盖索引 `set_keyread(true)`**：

```cpp
// sql/table.cc:6126
if (flag && !key_read) {
  key_read = true;
  if (is_created()) file->ha_extra(HA_EXTRA_KEYREAD);   // 只读索引不回表
}
```

判定依据 `TABLE::covering_keys`：初值 `s->keys_for_keyread`（要求 `index_flags & HA_KEYREAD_ONLY`），随后**随列被读取而收窄**——`mark_column_used()` 每次把字段加入 read_set 时就 `covering_keys.intersect(field->part_of_key)`。**一旦引用了不在某索引里的列，该索引就不再"覆盖"**。

**EXPLAIN "Using index" 输出**（`opt_explain.cc:1571`）：
- `JT_INDEX_SCAN/JT_CONST` 用 `covering_keys.is_set(tab->index())`
- 其它用 `table->key_read` 或 `tab->keyread_optim()`（后者在 `set_plan_state()` 里快照，**必须在 `make_tmp_tables_info` 之后**，因为它可能给首表加 sort）

### 2.3 push_index_cond（`sql_select.cc:2920`）—— SQL 层决策

**前置 early-out**：`reversed_access`（反向访问）、InnoDB 内部临时表、索引含虚拟生成列。

**other_tbls_ok**（`:2941`）：

```cpp
const bool other_tbls_ok =
    !((type() == JT_ALL || type() == JT_INDEX_SCAN || type() == JT_RANGE ||
       type() == JT_INDEX_MERGE) &&
      join_tab->use_join_cache() == JOIN_CACHE::ALG_BNL);
```

**扫描类 + BNL 时，下推条件不能引用其它非 const 表的字段**——BNL 会攒多行，条件中的"其它表字段"在攒行期间可能变化。ref/eq_ref（BKA）下 `other_tbls_ok` 为 true。

**8 项准入**：① 有 condition ② 引擎 `HA_DO_INDEX_COND_PUSHDOWN` ③ 开关+hint ④ 非多表 UPDATE/DELETE ⑤ `!has_guarded_conds()` ⑥ 非 JT_CONST/JT_SYSTEM ⑦ 非聚簇主键。

**下推与残留处理**：

```cpp
Item *idx_cond = make_cond_for_index(condition(), tbl, keyno, other_tbls_ok);
if ((idx_cond->used_tables() & table_ref->map()) == 0) return;   // 不含本表字段 → 推了没用
...
idx_remainder_cond = tbl->file->idx_cond_push(keyno, idx_cond);   // 返回残留
if (idx_remainder_cond != idx_cond) ref().disable_cache = true;    // 禁用 eq_ref lookup cache
set_condition(idx_remainder_cond);                                 // 残留留在 server
```

> handler 接口、`uses_index_fields_only` 四条限制、InnoDB 实现、EXPLAIN 输出见 **[05 篇 4 节](logical/05_logical_predicate.md)**。

---

## 三、push_to_engines：两层下推

**这是最容易混淆的一对概念**：

| 维度 | **handlerton 级**（整查询 offload） | **handler 级**（经典 ICP） |
|---|---|---|
| 函数 | `JOIN::push_to_engines()`（`sql_optimizer.cc:1143`） | `handler::idx_cond_push()`（`handler.h:5958`） |
| 粒度 | **整棵 AccessPath 树**（join/filter/aggregate） | **单表单索引上的一个布尔条件** |
| 目标 | 卸载到引擎（NDB pushed join、secondary engine offload） | 引擎内提前过滤索引记录，减少回表 |
| 输入 | `AccessPath *root_path` + `JOIN *` | `keyno` + `Item *idx_cond`（限索引列） |
| 时机 | `JOIN::optimize:1064`（`create_access_paths` 之后、Iterator 创建之前） | `make_join_readinfo`（`:1020`）内，**更早** |
| 副作用 | **会改写 AccessPath 树** | 只影响该 handler 的 `pushed_idx_cond` |
| 数量 | 每查询最多 1 个 handlerton（`break`） | 每个可用表各一次 |
| 失败处理 | 返回 true → optimize 失败 | 返回残留条件，server 保留求值（永远安全） |

```cpp
// sql/sql_optimizer.cc:1143
for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
  const handlerton *hton = tl->table->file->hton_supporting_engine_pushdown();
  if (hton != nullptr) {
    if (unlikely(hton->push_to_engine(thd, m_root_access_path, this))) return true;
    break;   // Assume that at most a single handlerton per query support pushdown
  }
}
```

**关键约束**（`:1049-1063` 注释）：push_to_engines **允许修改 AccessPath 本身**（移除已下推的 FILTER、改 JOIN 算法、改聚合表达式），所以**必须在创建 Iterator 之前**完成。

**谁实现**：本仓库中只有 **NDB**（`ha_ndbcluster.cc:12664` 注册 `ndbcluster_push_to_engine`）。（社区版无 HeatWave 实现。）

---

## 四、setup_join_buffering

> 本篇讲**优化器的 buffer 决策**（能否用、用 BNL 还是 BKA、外连接/semijoin 约束）。**执行器侧的缓冲机制**（BNL 改写 hash join、BKA + DS-MRR、hash join 的 build/probe/spill）详见 [`../../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)。

`sql_optimizer.cc:3451`。**扫描类 → BNL，ref 类 → BKA**（基于 MRR）。

**参数**：`bnl` 默认 **on**，`bka` 默认 **off**（不在 `OPTIMIZER_SWITCH_DEFAULT` 里）；`join_buffer_size` 默认 **256KB**。

**外连接/semijoin 约束**（节选）：

| 约束 | 说明 |
|---|---|
| 首表永不 buffer（`tableno == const_tables`） | 没有可缓存的外层行 |
| 外连接内表：nest 的 `first_inner` 不 buffer → 本表也不 | match flag 只能挂在第一个内表的 cache 上 |
| `SJ_OPT_FIRST_MATCH` | 仅当 semijoin 只含一个内表才允许 |
| `SJ_OPT_LOOSE_SCAN` / 被物化的第一个表 | 禁止 |
| `DUPS_WEEDOUT` / `NONE` | 允许 |
| LATERAL 派生表依赖区间内的表 | 禁止 |

**BKA 三个否决条件**：MRR 不可用（`rows == HA_POS_ERROR`）/ 用了 MRR 默认实现（不是真 MRR）/ 引擎声明 `HA_MRR_NO_ASSOCIATION`。

**`revise_cache_usage`**（`:3326`）：本表不用 buffer 时，要**向前回溯**关掉依赖它的表的 buffer（外连接场景关掉整个 nest 之前的表）。

> **Hash join 不在本函数**：8.0.39 的 hash join 只在 **hypergraph 优化器**路径生成；传统优化器只有 BNL/BKA。

---

## 五、临时表与排序 make_tmp_tables_info

`sql_select.cc:4354`。七个阶段：

| 阶段 | 行号 | 关键决策 |
|---|---|---|
| 0 前置 | 4354-4416 | `m_windowing_steps`；**loose index scan 特判**：`precomputed_group_by = !is_agg_loose_index_scan(...)`（LIS 已完成分组和 MIN/MAX 预计算） |
| 1 第一张临时表 | 4422-4550 | `tmp_group` 只在 `!simple_group` 时传——即"是否把 GROUP BY 做进临时表的唯一键" |
| 2 HAVING/聚合 | 4502-4550 | HAVING 可下推为临时表条件（**DISTINCT 存在时不能**）；临时表已分组 → `order = group_list` |
| 3 第二张临时表 | 4558-4640 | **触发条件**：GROUP BY 与 ORDER BY 顺序不一致（`!test_if_subpart`）、或有 DISTINCT/window/ROLLUP → "先排序再分组" |
| 4 DISTINCT | 4641-4654 | `needs_duplicate_removal = true` |
| 5 filesort | 4712-4789 | `m_ordered_index_usage` 与所需 clause **不同**才加 filesort |
| 6 window | 4791-4897 | 每个 window 一张临时表 + `REF_SLICE_WIN_wn` slice |
| 7 收尾 | 4899-4928 | `refresh_base_slice()`、`unplug_join_tabs()` |

**filesort limit 优化**（`:4780`）：

```cpp
sort_tab->filesort->limit =
    (has_group_by || (primary_tables > curr_tmp_table + 1) || calc_found_rows)
        ? m_select_limit
        : query_expression()->select_limit_cnt;
```

只有"无 GROUP BY + 单表 + 非 calc_found_rows"时才把 `select_limit_cnt` 下压给 filesort → 允许 `Bounded_queue`（`ORDER BY b DESC LIMIT 1` 只需维护 1 个元素的堆）。

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → ORDER BY Optimization*（`test_if_skip_sort_order` 的行为）
- *MySQL 8.0 Reference Manual → Index Condition Pushdown Optimization*
- *MySQL 8.0 Reference Manual → Block Nested-Loop and Batched Key Access Joins*、`Hash Join Optimization`
- *MySQL 8.0 Reference Manual → Internal Temporary Table Use in MySQL*

**内核月报**
- **2021/06《Order By 优化逻辑代码分析》** —— `test_skip_sort` / `test_if_order_by_key` 的索引免排序判定 + `filesort` 执行流程

