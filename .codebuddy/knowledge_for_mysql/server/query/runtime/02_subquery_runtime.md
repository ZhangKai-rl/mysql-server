# 02 子查询：运行期执行策略（算法级）

> 本篇覆盖：子查询在 prepare 阶段被改写后，**运行期各种形态分别怎么执行**——物化、IN2EXISTS、semi-join 五种策略、标量、EXISTS。

## 目录

- [先有个整体印象](#先有个整体印象)
- [一、Subquery_strategy 与运行期落地](#一subquery_strategy-与运行期落地)
- [二、物化执行：subselect_hash_sj_engine](#二物化执行subselect_hash_sj_engine)
- [三、IN2EXISTS 执行：Item_in_optimizer](#三in2exists-执行item_in_optimizer)
- [四、semi-join 五种策略的运行期](#四semi-join-五种策略的运行期)
- [五、标量子查询](#五标量子查询)
- [六、EXISTS 子查询](#六exists-子查询)
- [七、子查询缓存](#七子查询缓存)
- [八、深潜补充：物化引擎的建表 / 迭代器 / trig_cond](#八深潜补充物化引擎的建表--迭代器--trig_cond)

---

## 先有个整体印象

子查询在 prepare 阶段（06 篇）被改写成几种形态，运行期对应不同执行路径：

```
IN (SELECT ...)
 ├─ 扁平化成 semi-join ──▶ JOIN 执行器 5 种策略（FirstMatch/LooseScan/Weedout/Materialize）
 └─ 未扁平化
     ├─ IN2EXISTS ──▶ 外层每行重跑内层 EXISTS + LIMIT 1
     └─ 物化 ──────▶ 首次物化去重临时表，之后逐行 ref 查找
标量子查询 ──▶ 每次 exec()，结果存 Item_cache
EXISTS ──────▶ 一行即停短路
```

**关键区分两套"物化"**（别混淆）：semi-join 的 `MaterializeAccessPath` 物化的是**扁平化后的 nest**；`subselect_hash_sj_engine` 物化的是**未扁平化的 IN 子查询**。

---

## 一、Subquery_strategy 与运行期落地

`Subquery_strategy`（`item_subselect.h:398-418`）枚举与运行期对应：

| 策略 | 运行期怎么执行 |
|------|---------------|
| `SUBQ_EXISTS` | `Item_exists_subselect::val_bool()` → `exec()` → 内层重跑 |
| `SUBQ_MATERIALIZATION` | `subselect_hash_sj_engine::exec()`：首次物化 + ref 查找 |
| `SEMIJOIN` | 已扁平化，走 JOIN 执行器 5 种策略（第四节） |
| `DERIVED_TABLE` | 变成带 `MaterializeIterator` 的派生表 |
| `DELETED` | 表达式树已移除（恒假等） |

`CANDIDATE_FOR_IN2EXISTS_OR_MAT` 是"双重候选"：优化器比较 IN2EXISTS 与物化代价二选一，通过 `finalize_exists_transform()` / `finalize_materialization_transform()` 定死。

---

## 二、物化执行：subselect_hash_sj_engine

### 2.1 临时表建在哪

`subselect_hash_sj_engine::setup`（`item_subselect.cc:3248`）用 `Query_result_union::create_result_table` 建**去重临时表**（`is_union_distinct=true`，因为 IN 语义里重复值不影响结果），并建唯一/哈希索引。然后为每列构造 `store_key`（把外层左表达式值 encode 成查找 key），并生成后过滤条件 `tmp_table.col == left_expr`（处理截断、哈希碰撞假阳性）。

### 2.2 exec 的惰性物化 + ref 查找

`subselect_hash_sj_engine::exec`（`item_subselect.cc:3531`）：

```cpp
if (!is_materialized) {                        // 1. 首次：惰性物化
  bool error = m_iterator->Init();             //   MaterializeIterator 物化整个内层
  is_materialized = true;
  table->file->info(HA_STATUS_VARIABLE);
  has_zero_rows = (stats.records == 0);        // 2. 0 行短路
}

if (has_zero_rows) { item->value = false; return false; }

if (item->left_expr->element_index(0)->null_value) {  // 3. 左 NULL 短路
  item->value = true; return false;            //   结果 UNKNOWN，由上层翻译
}

bool found;
ExecuteExistsQuery(thd, item->unit, m_iterator.get(), &found);  // 4. ref 查找，只读一行
item->value = found;

if (!found && mat_table_has_nulls != NEX_IRRELEVANT_OR_FALSE) {
  // 5. 未命中但内层可能有 NULL：查一次 NULL 行，结果 UNKNOWN
  ...
}
```

算法要点：

1. **惰性物化**：首次求值才 `Init()` 触发 `MaterializeIterator`，之后 `is_materialized` 保证只物化一次。
2. **0 行短路**：物化表空则 IN 恒 FALSE。
3. **左 NULL 短路**：`oe IS NULL` 时 IN 是 UNKNOWN。
4. **ref 查找**：`ExecuteExistsQuery`（`item_subselect.cc:3086`）只读一行（`RefAccessPath + Filter`），命中即 true。
5. **NULL 补查**：未命中但内层含 NULL 时，IN 结果应为 UNKNOWN 而非 FALSE。

---

## 三、IN2EXISTS 执行：Item_in_optimizer

改写（`single_value_in_to_exists_transformer`，`item_subselect.cc:1907`）把 `oe IN (SELECT ie ...)` 变成 `EXISTS(SELECT 1 ... WHERE oe = ie ...)`，注入的等值谓词把外层值"推进"内层。

运行期核心 `Item_in_optimizer::val_int`（`item_cmpfunc.cc:2338`）：

```cpp
longlong Item_in_optimizer::val_int() {
  Item_in_subselect *subqpred = down_cast<Item_in_subselect *>(args[0]);

  cache->store(subqpred->left_expr);   // 1. 求外层左表达式，存入缓存
  cache->cache_value();

  if (cache->null_value) {             // 2. 外层含 NULL
    if (subqpred->abort_on_null) {     //   顶层裸 IN：直接 NULL
      null_value = true;
    } else {                           //   NOT IN 等：关掉 NULL 列的 trig_cond 再跑
      ...
      (void)subqpred->val_bool_naked();
      ...
    }
    return subqpred->translate(null_value, false);
  }

  bool result = subqpred->val_bool_naked();   // 3. 正常路径：执行内层 EXISTS
  null_value = subqpred->null_value;
  return subqpred->translate(null_value, result);
}
```

算法要点：

1. 外层左值存 `cache`（`Item_cache`），用于判断 NULL 和触发 trig_cond。
2. 外层 NULL 时：`abort_on_null`（裸 IN）直接 UNKNOWN；否则（NOT IN）用 `set_cond_guard_var(i,false)` 关掉左列为 NULL 的比较谓词再跑内层。
3. 正常路径 `val_bool_naked()`（`item_subselect.cc:1647`）→ `exec()` → `subquery->exec()` → `unit->execute()` **重跑内层**。这就是"外层每行 + 内层 EXISTS 检查"。
4. `finalize_exists_transform` 加的 `LIMIT 1`（`:383`）保证找到一行即停。

---

## 四、semi-join 五种策略的运行期

### 4.1 FirstMatch → `NestedLoopIterator`（`JoinType::SEMI`）

核心在 `NestedLoopIterator::Read`（`composite_iterators.cc:466`）状态机：

```cpp
if (m_join_type == JoinType::SEMI) {   // 找到一行内层匹配后
  m_state = NEEDS_OUTER_ROW;           // 不继续扫内层，直接回外层
} else {
  m_state = READING_INNER_ROWS;
}
return 0;
```

每个外层行只输出一次（第一个匹配），实现半连接的"去重 + 短路"。`ConnectJoins` 里 `join_type = substructure == SEMIJOIN ? JoinType::SEMI : JoinType::OUTER`（`sql_executor.cc:2549`）。

### 4.2 LooseScan → `RemoveDuplicatesOnIndexIterator`

依赖"内层按索引有序输出"，去重只需比较**相邻行**的 key：

```cpp
// composite_iterators.cc:2098
if (!m_first_row && key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0) {
  continue;   // 相同 key，跳过
}
m_first_row = false;
key_copy(m_key_buf, m_table->record[0], m_key, m_key_len);
return 0;
```

多表 LooseScan 用 `NestedLoopSemiJoinWithDuplicateRemovalIterator`（`composite_iterators.cc:2145`）：外层按 key 去重 + 内层只读一个匹配。

### 4.3 Duplicate Weedout → `WeedoutIterator`

思路：**把半连接当普通内连接执行，但按外层表 rowid 去重**（普通连接会让同一外层行匹配多个内层行产生多份输出）。

```cpp
// composite_iterators.cc:2008
for (SJ_TMP_TABLE_TAB *tab = m_sj->tabs; ...) {
  table->file->position(table->record[0]);   // 收集参与去重的表的 rowid
}
ret = do_sj_dups_weedout(thd(), m_sj);       // 用临时表判断 rowid 组合是否出现过
```

`do_sj_dups_weedout`（`sql_executor.cc:3386`）把多个表的 rowid 拼成一行，`check_unique_constraint` + `ha_write_row` 去重。若"confluent"（全字段去重）退化为 `LIMIT 1`。

### 4.4 Materialize lookup / scan → `MaterializeAccessPath`

规划期选 lookup 还是 scan（`sql_planner.cc:2187`）：内层不依赖外层前面的表且可 lookup → `SJ_OPT_MATERIALIZE_LOOKUP`，否则 `SJ_OPT_MATERIALIZE_SCAN`。

运行期（`sql_executor.cc:1763`）：把半连接内层当独立"虚拟 join"递归建树 → 对可空列加 `IS NOT NULL`（**因为 handler 层把 NULL=NULL 当相等，IN 语义要求 NULL≠NULL**）→ 用 `NewMaterializeAccessPath` 物化。lookup 模式临时表带唯一/哈希索引，外层逐行 ref 查找；scan 模式全表扫描后连接过滤。

---

## 五、标量子查询

`Item_singlerow_subselect::val_int/val_str`（`item_subselect.cc:1238`）：

```cpp
longlong Item_singlerow_subselect::val_int() {
  if (!no_rows && !exec(current_thd) && !value->null_value) {
    null_value = false;
    return value->val_int();    // 从 Item_cache 读结果
  } else {
    reset();
    return error_int();
  }
}
```

- 每次 `val_*` 都调 `exec()`；结果由 `Query_result_scalar_subquery::send_data` 写入 `Item_cache value`（`store` + `cache_value` 快照）。
- 多行 → `send_data` 抛 `ER_SUBQUERY_NO_1_ROW`。
- `no_rows`（内层空结果）时直接返回 NULL/错误值。
- 所以"缓存"是**本次执行结果快照**（`Item_cache`），不是跨外层行的结果缓存。

---

## 六、EXISTS 子查询

`Item_exists_subselect::val_bool`（`item_subselect.cc:1608`）：

```cpp
bool Item_exists_subselect::val_bool() {
  if (exec(current_thd)) { reset(); return false; }
  assert(!null_value);                    // EXISTS 永不返回 NULL
  return translate(null_value, value);
}
```

短路机制在 `Query_result_exists_subquery::send_data`（`:1319`）：收到第一行就把 `value=true`。配合 `LIMIT 1`，内层找到一行即停。

---

## 七、子查询缓存

> ⚠️ MySQL 8.0 **没有** `Subquery_cache` 类（那是 MariaDB 的特性）。8.0 的"缓存"是三处机制：

1. **左值缓存 `left_expr_cache`**（`Item_in_subselect`，仅 `SUBQ_MATERIALIZATION`）：`update_item_cache_if_changed`（`:771`）比较当前值与缓存值，未变则复用上次结果，跳过 ref 查找。
2. **`result_for_null_param`**（`Item_in_optimizer`）：不相关 IN 子查询外层全 NULL 时，缓存 `NULL IN (...)` 结果。
3. **物化临时表跨行复用**：`is_materialized` 保证临时表只物化一次，之后逐行 ref。

---


## 八、深潜补充：物化引擎的建表 / 迭代器 / trig_cond

### 8.1 setup() 建表细节（`item_subselect.cc:3283-3386`）

- **`key_length` 分两种**（`:3283-3289`）：有 `hash_field` 时 `key_length = ALIGN_SIZE(reclength)`；否则 `ALIGN_SIZE(key_length) * 2`（后者是因为要同时存"原始值"和"用于比较的值"，防截断误判）
- **访问类型**（`:3310`）：`type = (flags & HA_NOSAME) ? JT_EQ_REF : JT_REF`——唯一索引走 EQ_REF，否则 REF
- **store_key 的选择**（`:3317-3386`）：哈希列走 `ref.keypart_hash = &hash` + `store_key_hash_item`；普通列走 `store_key`；**可空列的 key buffer 首位留 NULL 字节**（这是后面 8.2 后过滤和 NULL 补查能工作的前提）

### 8.2 后过滤为什么必须（`:3338-3370`）

物化后还要逐列造 `Item_func_eq` 加进 `Item_cond_and` 做**后过滤**，两个原因：

1. **截断假阳性**：如 VARCHAR(10) 的列，索引里按前缀比较，`'abcdefghij'` 和 `'abcdefghijXYZ'` 可能判等
2. **哈希碰撞**：`hash_field` 模式下不同值的 hash 可能相同

为了给这些 `Item_func_eq` 做 `fix_fields`，代码会造一个临时 `Table_ref("<materialized_subquery>")`（`:3344-3350`）。

### 8.3 create_iterators（`:3418-3482`）—— 三个反直觉设计

1. **永不用 `EQRefIterator` / `AlternativeIterator`**（`:3428-3438` 注释）：物化表是临时表，语义上不会有"NULL 导致 ref 非法"的情况
2. **`JT_EQ_REF` 且有 cond/having 时补 `LIMIT 1`**（`:3443`）——唯一键定位，多读无意义
3. **把物化表伪装成 derived table** 调 `GetAccessPathForDerivedTable(rematerialize=false, invalidators=nullptr)`（`:3463-3471`）——**这正是"子查询物化不参与 lateral 失效"的源码根据**（对比真正的 derived table 会传 invalidators）
4. 末尾 `unit->clear_root_access_path()`（`:3481`）

### 8.4 cleanup 与重执行（`:3495-3511`）

每次执行后：`is_materialized = false`、`close_tmp_table/free_tmp_table`、`unit->reset_executed()`。**即 Prepared Statement 重执行会重新物化**——物化不是跨语句持久的。

### 8.5 NULL 补查的真实实现（`:3619-3630`）

未命中但内层可能有 NULL 时：`*ref.null_ref_key = 1`（把 key 设成 NULL）后走 `safe_index_read`（`:3513`，即 `ha_index_read_map(..., HA_READ_KEY_EXACT)`）。结果用**三态枚举**缓存（`item_subselect.h:886-894`：`NEX_UNKNOWN`/`NEX_IRRELEVANT_OR_FALSE`/`NEX_TRUE`）。

`has_zero_rows` 判定（`:3561-3575`）：`HA_STATS_RECORDS_IS_EXACT` 时直接用 `stats.records`，否则**真起一个 `TableScanIterator` 读一行**才知道是否为空。

### 8.6 trig_cond 机制（IN2EXISTS 的开关核心）

- `pushed_cond_guards` 分配：单值 `:1856-1861`、行值 `:2159-2165`
- **求值核心**（`item_cmpfunc.cc:7123`）：`return *trig_var ? args[0]->val_int() : 1;`——守卫变量为真才求值，否则**当作真**（`1`），即"这个条件暂时不算数"
- 类型 `OUTER_FIELD_IS_NOT_NULL`（`item_cmpfunc.h:840`）；`Item_is_not_null_test::val_int`（`item_cmpfunc.cc:6184-6194`）副作用 `owner->was_null |= 1` 再返回 0

### 8.7 `subselect_indexsubquery_engine` 从哪来

`JOIN::replace_index_subquery`（`sql_optimizer.cc:1388`）：当子查询可转成 `JT_EQ_REF`/`JT_REF`（或 `JT_REF_OR_NULL` 且 having 源自 in2exists）时，在 `:1446-1452` 创建该引擎。它的 `exec`（`item_subselect.cc:3134-3143`）直接复用 `unit->root_iterator()`——**不走重新执行，而是走优化好的迭代器树**。

> ⚠️ 本版**没有** `subselect_rowid_merge_engine`（MariaDB 的特性，别在 MySQL 里找）。

## 参考

**论文**
- **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)** —— 子查询去关联化与执行策略的理论基础

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizing Subqueries with Materialization*
- *MySQL 8.0 Reference Manual → Optimizing IN/=ANY Subqueries*

**内核月报**
- **2021/06《Semi-join 优化与执行逻辑》**

**两侧分工**
- 优化器侧（策略选择与改写）见 [`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md)、[`../07_optimize/logical/02_subquery.md`](../07_optimize/logical/02_subquery.md)
