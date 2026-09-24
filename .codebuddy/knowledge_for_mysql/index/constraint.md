# 索引与约束：唯一性检查与外键

> 基于 MySQL 8.0.39 源码。本篇讲 **索引承载的约束语义**：`PRIMARY`/`UNIQUE` 的唯一性是怎么被检查出来的，外键为什么必须有索引、检查与级联怎么发生。
>
> **边界**：索引类型与存储结构见 [`types.md`](types.md)；B-tree 的插入与分裂见 [`btr.md`](btr.md)；行锁/间隙锁的完整语义见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)；change buffer 的缓存规则见 [`ibuf.md`](ibuf.md)。本篇只讲"约束是怎么靠索引实现的"。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [`n_uniq`：唯一性检查的粒度](#n_uniq唯一性检查的粒度)
  - 唯一性：重复检测链路
    - [聚簇索引：`row_ins_duplicate_error_in_clust`](#聚簇索引row_ins_duplicate_error_in_clust)
    - [二级唯一索引：`row_ins_scan_sec_index_for_duplicate`](#二级唯一索引row_ins_scan_sec_index_for_duplicate)
    - [NULL 语义：为什么唯一索引允许多个 NULL](#null-语义为什么唯一索引允许多个-null)
    - [在线 DDL 期间的唯一性放宽](#在线-ddl-期间的唯一性放宽)
  - 外键：依赖索引的约束
    - [`dict_foreign_t`：只有两个索引指针](#dict_foreign_t只有两个索引指针)
    - [为什么必须有索引 + "自动创建"的真实实现](#为什么必须有索引--自动创建的真实实现)
    - [外键检查链路与加锁](#外键检查链路与加锁)
    - [级联：类型位标志、实现与深度限制](#级联类型位标志实现与深度限制)
    - [外键的已知限制](#外键的已知限制)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引在 MySQL 里不只是加速结构，它还**承载约束**：

1. **PRIMARY / UNIQUE** —— 约束的**物化**：唯一性不是 B-tree 的固有属性，而是 row 层用"有序性 + 可枚举性"外挂上去的语义；
2. **FOREIGN KEY** —— **依赖索引的约束**：`dict_foreign_t` 自身不存数据，只存两个索引指针；所有外键操作都在这两个索引上做定位。

### 用途

- 保证实体完整性（主键/唯一键不重复）；
- 保证引用完整性（子表外键值必须在父表存在；父表被引用的行不能随意删改，或按规则级联）；
- 顺带提供这些约束检查所需的**访问路径**（没有索引就无法高效检查）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | `innodb_locks_unsafe_for_binlog` 影响 RC 下的锁与半一致性读 |
| 8.0 | 该变量移除；`unique_checks` / `foreign_key_checks` 仍是会话级开关（默认 on），并**写入 binlog**（从库回放时恢复） |
| 8.0 | 外键支撑索引由 **SQL 解析器**自动生成（`parse_tree_nodes.cc`），InnoDB 侧明确"不隐式建索引" |

---

## 理论基础

### 设计思想与权衡

**1. B-tree 本身没有唯一性概念。** `btr_cur_optimistic_insert`/`pessimistic_insert` 只是把一个 entry 放进有序容器。唯一性是 **row 层**用三个原语"外挂"的：

- `n_uniq`（`dict_index_t::n_uniq`）——唯一性只需比较前 N 个字段；
- `up_match`/`low_match`（`PAGE_CUR_LE` 定位的副产品）——快速判断"候选重复项是否就在游标附近"；
- `cmp_dtuple_rec`——落到真实记录做精确比较。

**2. 必须在插入前检查，不能在插入后。** 因为 `row_ins_clust_index_entry_low` 里紧随重复检测之后就是 `row_ins_must_modify_rec()`：一旦成立就把 INSERT 转成对已有记录的 UPDATE / delete-unmark。若插入前不拦住，插入已存在的 PK 会**静默覆盖或复活旧记录**，而不是报错。

**3. 约束检查必须"看见真实数据"，于是与 change buffer 逻辑互斥。** change buffer 的本质是"我不读页，把操作缓存下来"；唯一性检查必须读页。这条互斥在 `ibuf_should_try` 的 `ignore_sec_unique` 参数上被显式表达（见 [`ibuf.md`](ibuf.md)）。

**4. 外键把"约束"完全绑在索引上。** 子表插入校验、父表删改校验、CASCADE/SET NULL 定位子表行、`ALTER` 判定索引能否删除——全部是在 `foreign_index` / `referenced_index` 上做 `PAGE_CUR_GE` 定位。没有索引，外键在物理上无法实现。

### 理论溯源

- Codd 的关系完整性约束（实体完整性 / 引用完整性）——索引是这些约束的物理实现手段。
- Gray & Reuter，*Transaction Processing*（1993）：约束检查与隔离级别的关系（为什么必须先加锁再判定，否则并发下会漏判）。
- SQL 标准的 `MATCH SIMPLE` 语义（外键列含 NULL 则不检查）——`row_ins_check_foreign_constraint` 的实现依据。

### 算法与数据结构

- 唯一性检查 = B-tree 定位（O(log n)）+ 沿叶子链扫描同键记录（O(k)，k = 同键记录数，受 MVCC 版本数影响）；
- 外键检查 = 一次 `PAGE_CUR_GE` 定位 + 一次比较；
- 关键结构：`dict_index_t::n_uniq`、`dict_foreign_t`（两个 `dict_index_t*`）、`btr_cur_t::up_match/low_match`。

---

## 核心实现

### 主线与基础构件

#### `n_uniq`：唯一性检查的粒度

```cpp
// storage/innobase/dict/dict0dict.cc:3241-3245（二级索引构建时）
if (dict_index_is_unique(index)) {
  new_index->n_uniq = index->n_fields;      // 唯一：只算用户声明的键字段
} else {
  new_index->n_uniq = new_index->n_def;     // 非唯一：键列 + 追加的主键列
}
```

`n_uniq` 决定：
- 比较器只比前 `n_uniq` 个字段（`dtuple_set_n_fields_cmp(entry, n_unique)`）；
- node pointer 只带 `n_uniq` 个字段 + 4 字节页号；
- 重复判定的门槛（`low_match/up_match >= n_uniq`）。

另外两个相关标志：

```cpp
unsigned allow_duplicates : 1;   // 即使是 unique 索引也允许重复
unsigned nulls_equal : 1;        // SQL NULL == SQL NULL
```

---

### 唯一性：重复检测链路

#### 聚簇索引：`row_ins_duplicate_error_in_clust`

触发（`row_ins_clust_index_entry_low`，`row0ins.cc:2464-2497`）：

```cpp
if (!index->allow_duplicates && n_uniq &&
    (cursor->up_match >= n_uniq || cursor->low_match >= n_uniq)) {
  ...
  err = row_ins_duplicate_error_in_clust(flags, cursor, entry, thr, &mtr);
}
```

其中 `n_uniq = dict_index_is_unique(index) ? index->n_uniq : 0`。**定位用的是 `PAGE_CUR_LE`**——唯一能同时给出有意义的 `low_match`（游标所在记录，即最后一条 ≤ entry 的）和 `up_match`（后继记录）的模式。

核心代码（`row0ins.cc:2131-2257`，节选）：

```cpp
n_unique = dict_index_get_n_unique(cursor->index);

if (cursor->low_match >= n_unique) {                    // ① 下侧候选
  rec = btr_cur_get_rec(cursor);
  if (!page_rec_is_infimum(rec)) {
    offsets = rec_get_offsets(rec, cursor->index, offsets, ...);
    if (flags & BTR_NO_LOCKING_FLAG) {
      err = DB_SUCCESS;                                 // 在线 rebuild 日志回放：不加锁
    } else {
      err = row_ins_set_rec_lock(row_allow_duplicates(thr) ? LOCK_X : LOCK_S,
                                 LOCK_REC_NOT_GAP, btr_cur_get_block(cursor),
                                 rec, cursor->index, offsets, thr);
    }
    switch (err) {
      case DB_SUCCESS_LOCKED_REC: case DB_SUCCESS: break;
      default: goto func_exit;                          // DB_LOCK_WAIT 等直接返回
    }
    if (row_ins_dupl_error_with_rec(rec, entry, cursor->index, offsets)) {
    duplicate:
      trx->error_index = cursor->index;
      err = DB_DUPLICATE_KEY;
      goto func_exit;
    }
  }
}

if (cursor->up_match >= n_unique) {                     // ② 上侧候选（后继记录）
  rec = page_rec_get_next(btr_cur_get_rec(cursor));
  if (!page_rec_is_supremum(rec)) {
    ... 同样加锁 + row_ins_dupl_error_with_rec ...
  }
  ut_error;                                             // up_match>=n_uniq 时后继必是用户记录
}
```

**为什么两个方向都要判**：`PAGE_CUR_LE` 只保证一侧（≤），真正与 entry 相等的记录可能是后继那条；唯一键的有序性保证"若重复项存在，必然与 entry 在 B-tree 序上相邻"。

**为什么用 `>=` 而不是 `==`**：源码注释指出"B-tree 上层 node pointer 可能比叶子层真实用户记录匹配更多字段"，所以 `low_match >= n_uniq` 只表示**可能**重复——必须落到叶子层取真实记录再做一次 `row_ins_dupl_error_with_rec()` 精确比较。

**锁的选择**（`row0ins.cc:2182-2185` 注释）：先加锁是为了**逻辑日志重放时拿到相同的 duplicate 错误**。类型 `LOCK_REC_NOT_GAP`（只锁记录不锁间隙，因为重复判定只关心记录本身）；模式由 `row_allow_duplicates(thr)` 决定：

```cpp
// row0mysql.h:924
bool allow_duplicates() { return (replace || on_duplicate_key_update); }
```

即 `REPLACE` / `INSERT ... ON DUPLICATE KEY UPDATE` / `LOAD DATA ... REPLACE` 取 `LOCK_X`（接下来要改这条记录），普通 INSERT 取 `LOCK_S`。

**真正的"重复"判定**（`row_ins_dupl_error_with_rec`，`row0ins.cc:1822-1861`）：

```cpp
n_unique = dict_index_get_n_unique(index);
entry->compare(rec, index, offsets, &matched_fields);
if (matched_fields < n_unique) return false;
if (!index->is_clustered() && !index->nulls_equal) {
  for (i = 0; i < n_unique; i++) {
    if (dfield_is_null(dtuple_get_nth_field(entry, i))) return false;   // NULL 不算重复
  }
}
return rec_get_deleted_flag(rec, rec_offs_comp(offsets)) == 0;          // ★ delete-mark 不算重复
```

#### 二级唯一索引：`row_ins_scan_sec_index_for_duplicate`

> **名称更正**：8.0.39 里**不存在** `row_ins_duplicate_error_in_sec`；二级唯一的检测由 `row_ins_scan_sec_index_for_duplicate` + `row_ins_dupl_error_with_rec`（与聚簇共用）完成，且**扫描发生在真正插入之前**。

**为什么必须扫描而不是"看一眼"**：二级索引记录布局是 `(用户键字段…, 追加的主键字段…)`，同一用户键值下可存在**多条**记录（不同主键、不同 MVCC 版本、delete-mark 状态不同）。所以必须从第一条 ≥ entry 的记录开始沿叶子链扫，直到遇到第一条严格大于 entry 的为止。

核心代码（`row0ins.cc:1872-2054`，节选）：

```cpp
if (!index->nulls_equal) {                                  // ① NULL 短路
  for (ulint i = 0; i < n_unique; i++) {
    if (UNIV_SQL_NULL == dfield_get_len(dtuple_get_nth_field(entry, i))) return DB_SUCCESS;
  }
}
n_fields_cmp = dtuple_get_n_fields_cmp(entry);
dtuple_set_n_fields_cmp(entry, n_unique);                    // ② 只比唯一键字段，不比追加 PK
pcur.open(index, 0, entry, PAGE_CUR_GE, BTR_SEARCH_LEAF, mtr, ...);   // ③ 第一条 ≥ entry

do {
  const rec_t *rec = pcur.get_rec();
  ...
  const bool is_next = !is_supremum && (cmp_dtuple_rec(entry, rec, index, offsets) < 0);
  if (flags & BTR_NO_LOCKING_FLAG) {
    /* 不加锁 */
  } else if (allow_duplicates) {
    err = row_ins_set_rec_lock(LOCK_X, lock_type, block, rec, index, offsets, thr);
  } else {
    if (skip_gap_locks) { ... lock_type = LOCK_REC_NOT_GAP; }
    else if (is_supremum)   lock_type = LOCK_ORDINARY;
    else if (is_next)       lock_type = LOCK_GAP;            // 首个更大的记录：只需锁它前面的间隙
    else                    lock_type = LOCK_ORDINARY;       // 相等记录：next-key
    err = row_ins_set_rec_lock(LOCK_S, lock_type, block, rec, index, offsets, thr);
  }
  ...
  if (!is_next && !index->allow_duplicates) {
    if (row_ins_dupl_error_with_rec(rec, entry, index, offsets)) {
      err = DB_DUPLICATE_KEY;
      thr_get_trx(thr)->error_index = index;                 // ④ SQL 层据此报 "for key 'xxx'"
      goto end_scan;
    }
  } else {
    goto end_scan;                                           // 已扫过所有相等记录
  }
} while (pcur.move_to_next(mtr));
```

要点：

1. **`dtuple_set_n_fields_cmp(entry, n_unique)`** 是必需的——否则两条用户键相同但主键不同的记录会被误判为"不相等"；
2. **delete-mark 的记录也要扫、也要加锁**（它所在的位置仍是他人可能插入同键的间隙，且删它的事务可能回滚），只是最后判定时被 `rec_get_deleted_flag(...) == 0` 排除；
3. `trx->error_index` 设置后，SQL 层通过 `trx_get_error_index()` 取出，拼出 "Duplicate entry 'x' for key 'uk_name'"。

**加锁前必须先做隐式锁转换**。唯一性检查走 `row_ins_set_rec_lock` → `lock_clust/sec_rec_read_check_and_lock`（`lock0lock.cc:5700/5749`），它们内部第一行就是：

```cpp
lock_rec_convert_impl_to_expl(block, rec, index, offsets);
```

**原因**：一条刚被未提交事务插入的记录只有**隐式锁**（聚簇靠 `DB_TRX_ID`，二级靠页 max trx id）。锁系统只认显式 lock struct；不先物化成显式 `LOCK_X|LOCK_REC_NOT_GAP` 就看不到冲突，**两个事务可以并发插入相同唯一键而互相看不见**。

**与 change buffer 的互斥（为什么必须先读页）**：`ibuf_should_try(index, ignore_sec_unique)` 里的 `(ignore_sec_unique || !dict_index_is_unique(index))`。链路是：

```
trx->check_unique_secondary（默认 true）
  → 不加 BTR_IGNORE_SEC_UNIQUE → btr_op = BTR_INSERT_OP → ignore_sec_unique = 0
  → 唯一索引上 ibuf_should_try = false → 正常读页 → 才有 up_match/low_match 与扫描
```

`unique_checks=0` 时反过来：`btr_op = BTR_INSERT_IGNORE_UNIQUE_OP` → `ignore_sec_unique=1` → 允许 ibuf → 页不在 buffer pool 时 `ibuf_insert()` 成功、`cursor->flag = BTR_CUR_INSERT_TO_IBUF` → `row0ins.cc:2913` 直接 `goto func_exit`，**整个重复检测被跳过**。

> 精确结论：`unique_checks=0` 不是"无条件跳过检查"，而是**允许 change-buffer 化从而在页不在内存时绕过检查**；页已在 buffer pool 时检查仍会执行。这就是"`unique_checks=0` 可能插入重复键"的源码根因。

绑定关系（`ha_innodb.cc:2764-2777`）：

```cpp
trx->check_foreigns = !thd_test_options(thd, OPTION_NO_FOREIGN_KEY_CHECKS);
trx->check_unique_secondary = !thd_test_options(thd, OPTION_RELAXED_UNIQUE_CHECKS);
```

#### NULL 语义：为什么唯一索引允许多个 NULL

三层事实串起来：

1. **物理比较层**：`cmp_data`（`rem0cmp.cc:397-411`）认为 `NULL == NULL`（返回 0），且 NULL 最小——所以多条 NULL 记录**有序相邻**，扫描能找到它们；
2. **row 层主动排除**：`!index->nulls_equal`（普通 InnoDB 表恒为 true，只有内部临时表设 `HA_NULL_ARE_EQUAL` 才为 false）在两处短路——`row_ins_scan_sec_index_for_duplicate` 开头和 `row_ins_dupl_error_with_rec` 里；
3. **语义依据**：SQL 标准 `UNIQUE` 基于 `NOT (a = b)`，而 `NULL = NULL` 求值为 `UNKNOWN`，两个 NULL **不满足"相等"**，故不构成唯一性违反。

主键（聚簇）不允许 NULL 是因为 InnoDB 层主键列强制 `DATA_NOT_NULL`，根本走不到二级索引的 NULL 分支。

#### 在线 DDL 期间的唯一性放宽

在线建唯一索引时，并发 DML 写入的重复键**不能让 DML 失败**（否则用户 INSERT 会因为一个还没建完的索引而报错）。8.0.39 的做法（`row0ins.cc:2950-2963`）：

```cpp
case DB_DUPLICATE_KEY:
  if (!index->is_committed()) {
    dict_set_corrupted(index);
    /* Do not return any error to the caller. The duplicate will be reported
       by ALTER TABLE or CREATE UNIQUE INDEX. */
    err = DB_SUCCESS;
  }
```

即 **DML 照常成功、索引被标记 corrupted**，由 DDL 线程在收尾时报错。

> **事实纠正**：`dict_index_t::allow_duplicates` 在 8.0.39 **不是**给在线建索引用的——它只用于**内部临时表**的 `disable_indexes()`/`enable_indexes()`（`ha_innodb.cc:18050-18100`），并有断言 `ut_ad(!index->allow_duplicates || index->table->is_intrinsic())`。

在线表重建（rebuild）时走另一条路：`row_ins_duplicate_error_in_clust_online` + `row_ins_duplicate_online`，比较 `n_uniq + 2` 个字段（多比 `DB_TRX_ID`、`DB_ROLL_PTR`）来区分"完全相同的记录"（`DB_SUCCESS_LOCKED_REC`，跳过插入）与"主键冲突"（`DB_DUPLICATE_KEY`）。

---

### 外键：依赖索引的约束

#### `dict_foreign_t`：只有两个索引指针

```cpp
// storage/innobase/include/dict0mem.h:1691-1726（节选）
struct dict_foreign_t {
  char *id;
  unsigned n_fields : 10;        // 约束列数（索引可含更多字段，但前 n_fields 个必须正好是约束列）
  unsigned type : 6;             // ON DELETE/UPDATE 的动作位标志
  dict_table_t *foreign_table;   // 子表
  const char **foreign_col_names;
  dict_table_t *referenced_table; // 父表
  const char **referenced_col_names;
  dict_index_t *foreign_index;    // ★ 子表支撑索引
  dict_index_t *referenced_index; // ★ 父表被引用索引
  dict_vcol_set *v_cols;
};
```

源码注释的一句话是关键：**"we require that both tables contain explicitly defined indexes for the constraint: InnoDB does not generate new indexes implicitly"** —— InnoDB 自己不隐式建索引。

#### 为什么必须有索引 + "自动创建"的真实实现

**强制规则的三处源码**：

1. SQL 层创建/ALTER 时校验子表支撑键与父表键（`sql/sql_table.cc:6727-6739`、`:6391-6396`，错误 `ER_FK_NO_INDEX_CHILD` / `ER_FK_NO_INDEX_PARENT`）；
2. InnoDB inplace ALTER（`handler0alter.cc:2010-2030`、`:2094-2110`）；
3. 字典缓存层 `dict_foreign_add_to_cache`（`dict0dict.cc:3484-3551`）——找不到合格索引就 `DB_CANNOT_ADD_CONSTRAINT`。

**"自动创建"发生在 SQL 解析器里**（`sql/parse_tree_nodes.cc:1863-1913`）：

```cpp
Key_spec *foreign_key = new (pc->mem_root) Foreign_key_spec(...);
pc->alter_info->key_list.push_back(foreign_key);
/* 无条件再 push 一个普通索引作为支撑键 */
Key_spec *key = new (pc->mem_root) Key_spec(thd->mem_root, KEYTYPE_MULTIPLE, key_name,
                                            &default_key_create_info, true /*generated*/, true, cols);
pc->alter_info->key_list.push_back(key);
```

若你已手工建了符合前缀要求的索引，`count_keys()` 里的 `foreign_key_prefix()` 会把生成的那个标记为 `redundant` 并忽略（`sql/sql_table.cc:4782-4821`）——这就是为什么"已有合适索引时看不到多出来的索引"。**父表侧不自动创建**，必须有已存在的索引。

**索引合格性的判定**（`dict_foreign_qualify_index`，`dict0dict.cc:4766-4830`）——四条硬规则全部落在这一函数里：

```cpp
bool dict_foreign_qualify_index(const dict_table_t *table, const char **col_names,
                                const char **columns, ulint n_cols,
                                const dict_index_t *index,
                                const dict_index_t *types_idx,
                                bool check_charsets, ulint check_null) {
  if (dict_index_get_n_fields(index) < n_cols) return false;      // ① 字段数不足

  for (ulint i = 0; i < n_cols; i++) {
    dict_field_t *field = index->get_field(i);
    ulint col_no = dict_col_get_no(field->col);

    if (field->prefix_len != 0) return false;                     // ② 拒绝列前缀索引

    if (check_null && (field->col->prtype & DATA_NOT_NULL))       // ③ SET NULL 要求列可为 NULL
      return false;

    const char *col_name = col_names ? col_names[col_no] : ...;
    if (0 != innobase_strcasecmp(columns[i], col_name))           // ④ 严格最左前缀、顺序一致
      return false;

    if (types_idx && !cmp_cols_are_equal(index->get_col(i), types_idx->get_col(i),
                                         check_charsets))         // ⑤ 父子列类型必须匹配
      return false;
  }
  return true;
}
```

- ① 索引字段数 ≥ 约束列数（**多余字段忽略**，所以 `KEY(a,b,c)` 可以支撑 `FOREIGN KEY (a,b)`）；
- ② **不接受列前缀索引**（`KEY(a(5))` 不合格）；
- ③ `check_null` 只在找**子表支撑索引**时传入（值 = `foreign->type & (ON_DELETE_SET_NULL | ON_UPDATE_SET_NULL)`）——**有 `SET NULL` 动作时子表外键列必须可为 NULL**，否则 SET NULL 无法执行；
- ④ 第 i 个索引字段必须正好是第 i 个外键列（顺序严格一致）；
- ⑤ `types_idx` 是对端索引（找父表索引时传子表索引），用来逐列比对类型。

查找函数 `dict_foreign_find_index`（`dict0dict.cc:3358-3399）还会跳过：`DICT_FTS`、空间索引、`to_be_dropped`、以及 `ONLINE_INDEX_ABORTED / ABORTED_DROPPED` 的未提交索引。

**父表为什么必须是唯一索引**：`dict_foreign_find_index` 本身**不检查** `DICT_UNIQUE`；真正的要求来自语义——外键的"引用"必须指向**唯一确定**的一行，否则"父行是否存在"不确定，父行删除时也无从判断"是否还有其它父行提供该值"。代码上的体现是 `row_ins_check_foreign_constraint` 里一旦 `cmp == 0` 就立刻 `err = DB_SUCCESS; goto end_scan;`（命中一条即认为约束成立）。SQL 层也优先挑唯一键（`sql/sql_table.cc:6350` 注释 "Prefer unique key if possible"）。

#### 外键检查链路与加锁

**函数名确认**：8.0.39 是 `row_ins_check_foreign_constraint(bool check_ref, ...)`（`row0ins.cc:1369`）+ 包装器 `row_ins_check_foreign_constraints`（`row0ins.cc:1757`）；**不存在** `row_ins_check_foreign_constraint_by_rec`。

**两个方向的调用点**：

```cpp
// 子表 INSERT/UPDATE（check_ref = true）—— row0ins.cc:3089 / 3189
if (!index->table->foreign_set.empty()) {
  err = row_ins_check_foreign_constraints(index->table, index, entry, thr);
}
// 且只有 foreign->foreign_index == index 时才检查（row0ins.cc:1783）

// 父表 DELETE/UPDATE 被引用列（check_ref = false）—— row0upd.cc:232
if (foreign->referenced_index == index &&
    (node->is_delete || row_upd_changes_first_fields_binary(entry, index, node->update,
                                                            foreign->n_fields))) {
  err = row_ins_check_foreign_constraint(false, foreign, table, entry, thr);
}
```

**三个快速短路**：

```cpp
if (dict_sys_t::is_dd_table_id(table->id)) return DB_SUCCESS;   // DD 表不检查
if (trx->check_foreigns == false) goto exit_func;               // foreign_key_checks=0
for (ulint i = 0; i < foreign->n_fields; i++) {
  if (dfield_is_null(dtuple_get_nth_field(entry, i))) goto exit_func;   // ★ 任一 FK 列为 NULL 则不检查
}
```

最后一条是 `MATCH SIMPLE` 语义（与 Oracle 兼容）。

**核心定位与比较**：

```cpp
if (check_ref) {
  check_table = foreign->referenced_table;
  check_index = foreign->referenced_index;      // 用父表索引定位
} else {
  check_table = foreign->foreign_table;
  check_index = foreign->foreign_index;         // 用子表支撑索引定位
}
...
if (check_table != table) lock_table(0, check_table, LOCK_IS, thr);   // 父表加 IS
dtuple_set_n_fields_cmp(entry, foreign->n_fields);                     // 只比前 n_fields 列
pcur.open(check_index, 0, entry, PAGE_CUR_GE, BTR_SEARCH_LEAF, &mtr, ...);
```

**加锁（源码确认）**：

| 情形 | 模式 | 类型 |
|---|---|---|
| 父行命中且未 delete-marked | `LOCK_S` | **`LOCK_REC_NOT_GAP`**（"Lock only a record because we can allow inserts into gaps"） |
| 命中但已 delete-marked | `LOCK_S` | `LOCK_REC_NOT_GAP` 或 `LOCK_ORDINARY` |
| 未命中（首个更大的记录） | `LOCK_S` | `LOCK_GAP`（`skip_gap_lock` 时不加） |
| supremum | `LOCK_S` | `LOCK_ORDINARY` |

错误码：`DB_NO_REFERENCED_ROW`（子表插入找不到父行）、`DB_ROW_IS_REFERENCED`（父行被引用且无 ON 动作）、`DB_FOREIGN_DUPLICATE_KEY`（级联导致别的表唯一键冲突，由 `DB_DUPLICATE_KEY` 映射而来，避免误导用户）。对应 handler 层 `HA_ERR_NO_REFERENCED_ROW` / `HA_ERR_ROW_IS_REFERENCED` / `HA_ERR_CANNOT_ADD_FOREIGN`。

#### 级联：类型位标志、实现与深度限制

**`type` 是位标志，不是枚举**（`dict0mem.h:1829-1844`）：

```cpp
constexpr uint32_t DICT_FOREIGN_ON_DELETE_CASCADE    = 1;
constexpr uint32_t DICT_FOREIGN_ON_DELETE_SET_NULL   = 2;
constexpr uint32_t DICT_FOREIGN_ON_UPDATE_CASCADE    = 4;
constexpr uint32_t DICT_FOREIGN_ON_UPDATE_SET_NULL   = 8;
constexpr uint32_t DICT_FOREIGN_ON_DELETE_NO_ACTION  = 16;
constexpr uint32_t DICT_FOREIGN_ON_UPDATE_NO_ACTION  = 32;
```

注意：**`RESTRICT` = `type == 0`**（注释："RESTRICT just means no flag"）；`SET DEFAULT` 在 8.0.39 没有对应常量（解析器层拒绝）。

**真正的级联入口不是 `row_ins_cascade_*`**（那三个只是辅助函数：`row_ins_cascade_ancestor_updates_table` 防环、`row_ins_cascade_n_ancestors` 深度、`row_ins_cascade_calc_update_vec` 构造 update vector），而是：

```
row_ins_check_foreign_constraint(false, ...)            [父表 delete/update]
  └─ row_ins_foreign_check_on_constraint()   row0ins.cc:957
       ├─ row_ins_cascade_ancestor_updates_table()      防循环级联 UPDATE
       ├─ row_ins_cascade_n_ancestors()                 深度检查
       ├─ row_ins_cascade_calc_update_vec()             ON UPDATE CASCADE 构造向量
       ├─ lock_table(LOCK_IX) + lock_clust_rec_read_check_and_lock_alt(LOCK_X, LOCK_REC_NOT_GAP)
       └─ row_update_cascade_for_mysql()     row0mysql.cc:2604
            └─ row_upd_step(thr) ──递归──> 子表 update/delete ──> 再次触发子表 referenced_set 检查
```

**级联如何用索引定位子表记录**：传入的 `pcur` 已定位在 `foreign->foreign_index` 上；若支撑索引不是聚簇索引，则用记录里的主键引用回查聚簇（`row_build_row_ref` + `open_no_init(clust_index, ref, PAGE_CUR_LE, ...)`）。子表行加 **`LOCK_X` + `LOCK_REC_NOT_GAP`**。

**级联的执行入口**（`row_update_cascade_for_mysql`，`row0mysql.cc:2604-2636`）：

```cpp
dberr_t row_update_cascade_for_mysql(que_thr_t *thr, upd_node_t *node,
                                     dict_table_t *table) {
  /* Increment fk_cascade_depth to record the recursive call depth on
     a single update/delete that affects multiple tables chained together
     with foreign key relations. */
  thr->fk_cascade_depth++;
  if (thr->fk_cascade_depth > FK_MAX_CASCADE_DEL) {
    return (DB_FOREIGN_EXCEED_MAX_CASCADE);
  }
run_again:
  thr->run_node = node;
  thr->prev_node = node;
  TABLE *temp = thr->prebuilt->m_mysql_table;
  thr->prebuilt->m_mysql_table = nullptr;
  row_upd_step(thr);                      // ★ 递归入口：子表 update/delete
  thr->prebuilt->m_mysql_table = temp;
  /* The recursive call for cascading update/delete happens in above
     row_upd_step(), reset the counter once we come out of the recursive call,
     so it does not accumulate for different row deletes */
  thr->fk_cascade_depth = 0;
  ...
}
```

注意 `thr->fk_cascade_depth = 0` 的时机：**只在从递归返回后才归零**，所以深度是"当前嵌套层数"而非"累计次数"。

**SET NULL 的 update vector**（`row0ins.cc:1160-1190`）直接按索引前 `n_fields` 个字段取列并置 NULL——再次印证外键列必须是支撑索引的最左前缀：

```cpp
if (node->is_delete ? (foreign->type & DICT_FOREIGN_ON_DELETE_SET_NULL)
                    : (foreign->type & DICT_FOREIGN_ON_UPDATE_SET_NULL)) {
  update = cascade->update;
  update->n_fields = foreign->n_fields;
  for (i = 0; i < foreign->n_fields; i++) {
    upd_field_t *ufield = &update->fields[i];
    ulint col_no = index->get_col_no(i);                       // ★ 按索引第 i 个字段取列
    ufield->field_no = dict_table_get_nth_col_pos(table, col_no);
    dfield_set_null(&ufield->new_val);
  }
}
```

**子表行的加锁**（`row0ins.cc:1126-1141`）：

```cpp
err = lock_table(0, table, LOCK_IX, thr);
if (err == DB_SUCCESS) {
  /* Here it suffices to use a LOCK_REC_NOT_GAP type lock;
     we already have a normal shared lock on the appropriate gap if the
     search criterion was not unique */
  err = lock_clust_rec_read_check_and_lock_alt(clust_block, clust_rec, clust_index,
                                               LOCK_X, LOCK_REC_NOT_GAP, thr);
}
```

即子表行是 **`LOCK_X` + `LOCK_REC_NOT_GAP`**（与父表行的 `LOCK_S`+`LOCK_REC_NOT_GAP` 对称）。

**循环级联的禁止**（`row0ins.cc:1040-1062`）：DELETE 允许环，**UPDATE 不允许**——`row_ins_cascade_ancestor_updates_table(cascade, table)` 为真时直接 `DB_ROW_IS_REFERENCED`，注释说明"cyclic cascaded updating ... can lead to an infinite cycle"。

**深度限制**：**`FK_MAX_CASCADE_DEL = 15`**（`dict0mem.h:318`，注意不是 `FK_CASCADE_MAX_DEPTH`）。三重防护：

1. 查询图祖先层数 `row_ins_cascade_n_ancestors(cascade) >= FK_MAX_CASCADE_DEL` → `DB_FOREIGN_EXCEED_MAX_CASCADE`；
2. 线程级 `thr->fk_cascade_depth`（每次从 `row_update_cascade_for_mysql` 返回时归零）；
3. **禁止循环级联 UPDATE**（DELETE 允许环，UPDATE 不允许）→ `DB_ROW_IS_REFERENCED`。

#### 外键的已知限制

| 限制 | 源码位置 |
|---|---|
| **不支持跨引擎**（父表 handlerton 必须与子表相同） | `sql/sql_table.cc:6813-6817`（`ER_FK_CANNOT_OPEN_PARENT`） |
| **不支持虚拟列上的外键** | `sql/sql_table.cc:6650-6653`（`ER_FK_CANNOT_USE_VIRTUAL_COLUMN`） |
| **有 CASCADE 的 FK 不能建在存储生成列的基列上** | `ha_innodb.cc:13710-13754`（`DB_NO_FK_ON_S_BASE_COL`） |
| **分区表不支持外键** | `ha_innodb.cc:4487-4491`（`HA_CANNOT_PARTITION_FK`）+ 四处拦截（`ER_FOREIGN_KEY_ON_PARTITIONED`） |
| **不能 DROP 被 FK 依赖的索引**（除非有等价替代） | `handler0alter.cc:5177-5255`（`innobase_check_foreign_key_index`） |
| **不支持前缀索引 / 函数索引 / 空间 / 全文索引作外键索引** | `dict_foreign_qualify_index`（prefix_len≠0 不合格） |

索引重建后需重新绑定（`dict0dict.cc:4442-4475`，用 `dict_foreign_find_index` 找替代索引）。

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `unique_checks` | **ON** | Global/Session（HINT_UPDATEABLE，写入 binlog） | 关闭 → `trx->check_unique_secondary = false` → 允许唯一二级索引的插入 change-buffer 化（页不在内存时绕过重复检测） |
| `foreign_key_checks` | **ON** | Global/Session（写入 binlog） | 关闭 → `trx->check_foreigns = false` → 完全跳过外键检查 |
| `innodb_change_buffering` | all | Global | 与 `unique_checks` 联动影响唯一索引插入能否缓存 |

> 两者都是 `REVERSE(...)` 定义的位变量（`sql/sys_vars.cc:5598/5604`），并通过 `OPTIONS_WRITTEN_TO_BIN_LOG` 记录到 binlog，从库回放时按位恢复（`sql/log_event.cc:9677-9686`）。`LOAD DATA` **不会**自动关闭 `unique_checks`。

---

## Misc

### 约束类型总表

| 约束 | 依赖的索引 | 检查函数 | 关键比较 | 加的锁 | 失败错误码 | 开关（默认） |
|---|---|---|---|---|---|---|
| 主键（聚簇唯一） | 聚簇索引 | `row_ins_duplicate_error_in_clust`（`row0ins.cc:2140`） | `low/up_match >= n_uniq` + `row_ins_dupl_error_with_rec` | 候选记录 `LOCK_S`(或 `LOCK_X` for REPLACE) + `LOCK_REC_NOT_GAP` | `DB_DUPLICATE_KEY` | **无开关** |
| 二级唯一 | 唯一二级索引 | `row_ins_scan_sec_index_for_duplicate`（`row0ins.cc:1876`） | `cmp_dtuple_rec`（只比前 `n_uniq` 字段），扫到首个更大记录为止 | 相等记录 `LOCK_S\|LOCK_ORDINARY`；首个更大记录 `LOCK_S\|LOCK_GAP` | `DB_DUPLICATE_KEY` | `unique_checks`（ON） |
| 二级唯一（在线 DDL） | 同上（`is_committed()==false`） | 同上 + `row0ins.cc:2950` | 同上 | 同上 | 索引标 corrupted，DML 成功，由 ALTER 报错 | — |
| 聚簇（在线 rebuild） | 聚簇索引 | `row_ins_duplicate_error_in_clust_online`（`row0ins.cc:2101`） | 比 `n_uniq + 2` 个字段 | 无（`BTR_NO_LOCKING_FLAG`） | `DB_DUPLICATE_KEY` / `DB_SUCCESS_LOCKED_REC` | — |
| 外键：子表 INSERT/UPDATE | 子表 `foreign_index` 触发 + 父表 `referenced_index` 定位 | `row_ins_check_foreign_constraint(true,...)`（`row0ins.cc:1369`） | `PAGE_CUR_GE` + `cmp_dtuple_rec`（前 `n_fields`） | 父表 `LOCK_IS` + 父行 `LOCK_S\|LOCK_REC_NOT_GAP` | `DB_NO_REFERENCED_ROW` | `foreign_key_checks`（ON） |
| 外键：父表 DELETE/UPDATE | 父表 `referenced_index` 触发 + 子表 `foreign_index` 定位 | `row_ins_check_foreign_constraint(false,...)`（调用于 `row0upd.cc:256`） | 同上 | 子表行 `LOCK_S`；级联时 `LOCK_X\|LOCK_REC_NOT_GAP` | `DB_ROW_IS_REFERENCED` / `DB_FOREIGN_DUPLICATE_KEY` | `foreign_key_checks`（ON） |

### 易混淆点

- **`allow_duplicates` 不是给在线 DDL 用的**——它只服务于内部临时表的 `disable_indexes()`；在线建索引靠"标 corrupted"来放宽。
- **RESTRICT = `type == 0`**（没有位标志），`NO ACTION` 是独立的位（16/32）。
- **`unique_checks=0` 不等于"跳过检查"**——只是允许 change-buffer 化，页已在 buffer pool 时检查照做。
- **外键列为 NULL 时不检查**（`MATCH SIMPLE`），这与"唯一索引允许多个 NULL"是两条独立的 NULL 语义。
- **外键索引不接受前缀索引**，且必须是严格最左前缀、顺序一致。
- **`row_ins_scan_sec_index_for_duplicate` 是插入前扫描**，不是插入后检查（源码里的 NOTE 是历史注释）。

### 一句话总结

B-tree 只提供"有序"和"可枚举"，唯一性语义是 row 层用 `n_uniq` + `up_match/low_match` + `cmp_dtuple_rec` 外挂上去的，代价是**插入前必须读页并加锁**（因此与 change buffer 互斥）；外键则把约束完全绑在两个索引指针上，所有检查、级联、`ALTER` 判定都在这两个索引上做 `PAGE_CUR_GE` 定位——没有索引，二者都无法实现。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 15.6.2.1 Clustered and Secondary Indexes*
- *MySQL 8.0 Reference Manual → 13.1.20.5 FOREIGN KEY Constraints*（`MATCH`、级联动作、限制）
- *MySQL 8.0 Reference Manual → 5.1.8 Server System Variables*（`unique_checks` / `foreign_key_checks`）

**论文**
- Codd, E. *A Relational Model of Data for Large Shared Data Banks*. CACM, 1970.（完整性约束的原始定义）
- Gray, J. & Reuter, A. *Transaction Processing: Concepts and Techniques*. 1993.（约束检查与并发控制的交互）

**相关文档**
- 索引类型与存储结构：[`types.md`](types.md)
- 唯一索引与 change buffer 的互斥规则：[`ibuf.md`](ibuf.md)
- 行锁、间隙锁与隐式锁转换：[`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- 二级索引可见性与回表：[`../innodb/mvcc.md`](../innodb/mvcc.md)
- B-tree 插入与分裂：[`btr.md`](btr.md)
