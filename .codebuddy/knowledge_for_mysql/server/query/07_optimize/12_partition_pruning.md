# 12 分区裁剪（Partition Pruning）

> 分区裁剪决定"一个 WHERE 条件只需要扫哪些分区"，把全表扫描缩小到几个分区。核心洞察：**它被归约为 range 分析问题**——复用 `get_mm_tree` 把 WHERE 转成 SEL_TREE，再遍历 SEL_ARG 区间树映射成分区号。
>
> **边界**：本篇讲**裁剪算法本身**（优化器视角）。分区表的完整链路（元数据模型、DD 持久化、DML 路由、分区级 DDL、InnoDB `ha_innopart` 实现）见 [`../../../feat/partitioning.md`](../../../feat/partitioning.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、位置与本质](#一位置与本质)
- [二、partition_info 数据结构](#二partition_info-数据结构)
- [三、prune_partitions 主流程](#三prune_partitions-主流程)
- [四、find_used_partitions：遍历 SEL_ARG 树](#四find_used_partitions遍历-sel_arg-树)
- [五、各分区类型的裁剪逻辑](#五各分区类型的裁剪逻辑)
- [六、与 SEL_TREE 的复用关系](#六与-sel_tree-的复用关系)
- [七、结果使用与 EXPLAIN](#七结果使用与-explain)
- [八、server 层分区机制：打开 / DML 分发 / 接口](#八server-层分区机制打开--dml-分发--接口)
- [九、失效场景](#九失效场景)

---

## 设计思想与理论基础

### 分区裁剪解决什么问题

分区表把一张逻辑表拆成 N 个物理分区。若查询条件能确定"只有某几个分区可能含匹配行"，就可**跳过其余分区**——这是分区表最主要的查询期收益（从扫 N 个分区降到 k 个，k ≪ N）。

### 核心设计：归约为 range 分析（本篇最重要的思想）

朴素做法：为裁剪专门写一套"分析 WHERE → 确定分区"的逻辑。

MySQL 的做法是**不写**——把分区字段**伪装成一个虚拟索引**，然后复用 range 优化器：

```
create_partition_index_description()   // 分区字段 → 虚拟 index
get_mm_tree()                          // 复用 range 分析：WHERE → SEL_TREE
find_used_partitions()                 // 遍历 SEL_ARG 树 → 分区号
```

> 源码头部注释（`partition_pruning.cc:57`）把这称为把裁剪**归约为 range 分析问题**。

**为什么这是好设计**：

1. **零重复实现**——range 优化器已有的区间分析、AND/OR 处理、类型比较全部可用
2. **语义一致**——`WHERE part_col = 5` 在裁剪里和在 range 优化里得到相同解释
3. **不要求索引真实存在**（`using_real_indexes=false`）

**代价**：裁剪能力直接受制于"分区键能否被 range 表达"。这解释了不同分区类型的裁剪能力差异（本文第五章）——HASH/KEY 基本只能裁 `IS NULL`，因为哈希分区键不是区间可表达的。

### 静态裁剪 vs 动态裁剪：为什么裁两次

| | 时机 | 条件类型 | 用途 |
|---|---|---|---|
| **静态** | prepare 期（表未锁） | 只含常量 | **锁裁剪**（少锁分区） |
| **动态** | optimize 期（表已锁） | 可含非常量 | **读裁剪**（少读分区） |

两阶段的存在理由是分区表特有的：**分区裁剪同时影响锁和读两个层面**。prepare 期表还没锁，但锁的粒度必须在锁表前确定，所以先用常量条件裁一轮；optimize 期条件更全（含已求值的子查询、函数），再裁一轮用于读。

> 对照：PostgreSQL 的分区裁剪分 **plan-time** 与 **run-time**（执行期才裁，用于参数未知场景）；MySQL 8.0 只有编译期裁剪，**没有 runtime partition pruning**。

### 失效的本质

裁剪只在优化器能**静态理解**条件时生效。一旦条件含无法静态求值的部分（存储函数、非确定性表达式、未展开的子查询），就退化为全分区扫描。本文第九章列具体场景。

---

## 一、位置与本质

### 1.1 为什么放在 range_optimizer 目录下

文件头部大注释（`partition_pruning.cc:57`）直接回答——**分区裁剪被归约为 range 分析问题**：

```
prune_partitions() {
  call create_partition_index_description();  // 把分区字段伪装成一个虚拟 index
  call get_mm_tree();                         // 复用 range 分析模块，把 WHERE 转成 SEL_TREE
  call find_used_partitions();                // 遍历 SEL_ARG 树，映射成分区号
}
```

它复用的 range 优化机制：`get_mm_tree`（WHERE→SEL_TREE）、`SEL_TREE/SEL_ROOT/SEL_ARG`（区间图）、`RANGE_OPT_PARAM`（参数结构）。**分区裁剪不自己分析 Item 树，而是让 `get_mm_tree` 生成 SEL_TREE**，只是把"分区字段"伪装成一个虚拟 index（`keys=1, using_real_indexes=false`），不需要索引真实存在。

### 1.2 架构背景：8.0 已移除 ha_partition

MySQL 8.0 的旧 `ha_partition`（non-native partitioning）**已被移除**。现在的机制：
- 分区裁剪算法在 `sql/range_optimizer/partition_pruning.cc`（纯优化器层，不依赖引擎）
- 引擎侧由 `Partition_helper`（`partitioning/partition_handler.h:390`）基类提供，InnoDB 的 `ha_innopart` 继承它，通过 `ph_rnd_init` 遍历 `read_partitions` 位图

### 1.3 四个调用点

**⚠️ 勘误：`prune_partitions()` 不在 `sql_select.cc`，那里一次都没调用过**。全仓只有 **4 处**：

| 阶段 | 位置 | 函数 | cond 来源 |
|---|---|---|---|
| SELECT **prepare 期** | `sql/sql_resolver.cc:812` | `Query_block::prepare()` 尾部 | `tbl->join_cond() ?: m_where_cond` |
| SELECT **optimize 期** | `sql/sql_optimizer.cc:2825` | `JOIN::prune_table_partitions()` | join nest 最外层 cond |
| UPDATE | `sql/sql_update.cc:475` | 单表 UPDATE optimize | `conds` |
| DELETE | `sql/sql_delete.cc:391` | `Sql_cmd_delete::delete_from_single_table()` | `conds` |

> ⚠️ **`sql_derived.cc` / `sql_union.cc` 里没有调用点**；**INSERT 完全不走 `prune_partitions()`**（走独立的逐行通道，见 §8.3）。

```cpp
// ① prepare 期（sql_resolver.cc:802）—— ★ 用于"锁裁剪"
  if (partitioned_table_count && prune) {   // ★ prune 为 false 时（如 CREATE VIEW）整段跳过
    for (Table_ref *tbl = leaf_tables; tbl; tbl = tbl->next_leaf) {
      /* This will only prune constant conditions, which will be used for
         lock pruning. */
      if (prune_partitions(thd, tbl->table, this,
                           tbl->join_cond() ? tbl->join_cond() : m_where_cond))
        return true;
      if (tbl->table->all_partitions_pruned_away &&
          !tbl->is_inner_table_of_outer_join())
        set_empty_query();       // ★ 整表被剪光 ⇒ 整查询置空
    }
  }
```

```cpp
// ② optimize 期（sql_optimizer.cc:2793）—— ★ 用于"读裁剪"
bool JOIN::prune_table_partitions() {
  assert(query_block->partitioned_table_count);
  for (Table_ref *tbl = query_block->leaf_tables; tbl; tbl = tbl->next_leaf) {
    // Predicates for pruning of this table must be placed in the outer-most
    // join nest (Predicates in other join nests, or in the WHERE clause,
    // would have caused an outer join to be converted to an inner join ...)
    Item *cond = where_cond;
    const table_map tbl_map = tbl->map();
    for (Table_ref *nest = tbl; nest != nullptr; nest = nest->embedding) {
      if (nest->join_cond_optim() != nullptr &&
          Overlaps(tbl_map, nest->join_cond_optim()->used_tables())) {
        cond = nest->join_cond_optim();
        if (nest->is_aj_nest()) break;   // ★ anti-join 的合成 nest 要跳过
      }
    }
    if (prune_partitions(thd, tbl->table, query_block, cond)) return true;
  }
  return false;
}
```

```cpp
// ③ UPDATE（sql_update.cc:470）—— ★ 锁表后二次裁剪
  /* Also try a second time after locking, to prune when subqueries and
     stored programs can be evaluated. */
  if (table->part_info && !no_rows) {
    if (prune_partitions(thd, table, query_block, conds)) return true;
    if (table->all_partitions_pruned_away) {
      no_rows = true;
      if (thd->lex->is_explain()) {
        Modification_plan plan(thd, MT_UPDATE, table,
                               "No matching rows after partition pruning", true, 0);
        ...
```

> ★ `sql_update.cc` 另有两处注释强调**顺序依赖**：`table->triggers->mark_fields(TRG_EVENT_UPDATE)` 必须**在 prune 之前**做完，因为 `prune_partitions()` 要用 `table->write_set` 判断 lock 能否收缩。

```cpp
// ④ DELETE（sql_delete.cc:389）—— 同 UPDATE
  /* Prune a second time to be able to prune on subqueries in WHERE clause. */
  if (table->part_info && !no_rows) {
    if (prune_partitions(thd, table, query_block, conds)) return true;
    if (table->all_partitions_pruned_away) { ... "No matching rows after partition pruning" ... }
```

| 时机 | 调用链 | 裁剪内容 |
|------|--------|----------|
| SELECT prepare 期 | `apply_local_transforms`（`sql_resolver.cc:802`） | 只裁**常量条件**（表未锁），用于**锁裁剪** |
| SELECT optimize 期 | `JOIN::prune_table_partitions`（`sql_optimizer.cc:502/2801`） | 裁**非静态条件**（表已锁），用于**读裁剪** |
| UPDATE 单表快路径 | `update_single_table`（`sql_update.cc:474`） | 锁表后二次裁剪 |
| DELETE 单表快路径 | `delete_from_single_table`（`sql_delete.cc:389`） | 同上 |
| INSERT | `can_prune_insert`（`partition_info.cc:311`） | 按 VALUES 值算分区（不走 WHERE 分析） |

---

## 二、partition_info 数据结构

`partition_info`（`partition_info.h:209`）关键成员：

```cpp
List<partition_element> partitions;        // 分区列表
get_part_id_func get_partition_id;         // 求叶子分区 id（含子分区）
get_part_id_func get_part_partition_id;    // 只求分区 id
get_subpart_id_func get_subpartition_id;   // 只求子分区 id
Field **part_field_array;                  // 分区表达式字段数组
Item *part_expr;                           // 分区表达式 Item 树
MY_BITMAP read_partitions;                 // 读分区位图（裁剪结果）
MY_BITMAP lock_partitions;                 // 锁分区位图
union {                                    // 各分区类型的边界值数组
  longlong *range_int_array;               // RANGE 整数边界
  LIST_PART_ENTRY *list_array;             // LIST 整数列表值（有序）
  part_column_list_val *range_col_array;   // RANGE COLUMNS
  part_column_list_val *list_col_array;    // LIST COLUMNS
};
get_partitions_in_range_iter get_part_iter_for_interval;  // 区间映射函数
partition_type part_type;                  // RANGE/LIST/HASH/KEY
uint num_parts; uint num_subparts;
```

**位图语义**：`read_partitions` = 本次查询读的分区；`lock_partitions` = 必须加锁的分区（通常相同，但 UPDATE 修改分区键字段时 lock 可能比 read 多）。

`partition_element`（每个分区，`partition_element.h:108`）：`range_value`（RANGE 的 LESS THAN 边界）、`list_val_list`（LIST 的 VALUES IN 列表）、`subpartitions`（子分区）。

**⚠️ 三个命名勘误**：

| 常见说法 | 8.0.39 真相 |
|---|---|
| `sql/partitioning/partition_info.h` | **不存在 `sql/partitioning/partition_info.h`**——真身 `sql/partition_info.h`（旧版目录已拆） |
| `partition_info::used_partitions` | **不是成员**。成员只有 `read_partitions` / `lock_partitions` 两个；`used_partitions` 是 `sql_insert.cc:1443` 的**局部变量** + `can_prune_insert()` 的出参 |
| 标志 `bitmaps_ready` | 真名 **`bitmaps_are_initialized`** |

> ★ 类里的第三个 `MY_BITMAP` 成员是 **`full_part_field_set`**——但它是**字段位图**（分区表达式用到哪些列），不是分区位图。

**★ 位图的位数 = 叶子级分区数**（`partition_info::init_partition_bitmap`，`partition_info.cc:2861`）：

```cpp
bool partition_info::init_partition_bitmap(MY_BITMAP *bitmap, MEM_ROOT *mem_root) {
  uint32 *bitmap_buf;
  uint bitmap_bits = num_subparts ? (num_subparts * num_parts) : num_parts;
  //   ★★ 有子分区时是乘积，不是 num_parts
  uint bitmap_bytes = bitmap_buffer_size(bitmap_bits);
  if (!(bitmap_buf = (uint32 *)mem_root->Alloc(bitmap_bytes))) {
    mem_alloc_error(bitmap_bytes);
    return true;
  }
  bitmap_init(bitmap, bitmap_buf, bitmap_bits);
  return false;
}
```

**★ 两个 bitmap 的真正分配点**（`set_up_partition_bitmaps()`，static，`sql_partition.cc:1140`）：

```cpp
static bool set_up_partition_bitmaps(partition_info *part_info) {
  assert(!part_info->bitmaps_are_initialized);
  if (part_info->init_partition_bitmap(&part_info->read_partitions,
                                       &part_info->table->mem_root))
    return true;
  if (part_info->init_partition_bitmap(&part_info->lock_partitions,
                                       &part_info->table->mem_root))
    return true;
  part_info->bitmaps_are_initialized = true;      // ★ 唯一置 true 的地方
  part_info->set_partition_bitmaps(nullptr);      // ★ 立刻全置 1
  return false;
}
```

```cpp
bool partition_info::set_partition_bitmaps(Table_ref *table_list) {
  assert(bitmaps_are_initialized);
  is_pruning_completed = false;      // ★★ 每次重设 bitmap 都清空"裁剪已完成"，允许 optimize 期二次裁剪
  ...
  if (table_list && table_list->partition_names &&
      table_list->partition_names->elements) {
    if (table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION) {
      my_error(ER_PARTITION_CLAUSE_ON_NONPARTITIONED, MYF(0));   // ★ NDB 禁止 PARTITION() 子句
      return true;
    }
    if (set_read_partitions(table_list->partition_names)) return true;
  } else {
    bitmap_set_all(&read_partitions);   // ★ 没有 PARTITION(pX)：先全 1，等 prune 来清
  }
  bitmap_copy(&lock_partitions, &read_partitions);   // ★ 初始 lock == read
  return false;
}
```

> ⚠️ **`open_partitioning()` 不初始化这两个 bitmap**——它只 `bitmap_init(&m_key_not_found_partitions, ...)`（handler 内部状态）。`read/lock_partitions` 由 `set_up_partition_bitmaps()`（在 `fix_partition_func` 内）分配。

**从记录算分区 id**（`set_up_partition_func_pointers`，`sql_partition.cc:1226` 装配函数指针）。真实的赋值表（★ **有子分区时 `get_partition_id` 一律是 `get_partition_id_with_sub`**）：

```cpp
static void set_up_partition_func_pointers(partition_info *part_info) {
  if (part_info->is_sub_partitioned()) {
    part_info->get_partition_id = get_partition_id_with_sub;   // ★★ 入口统一
    if (part_info->part_type == partition_type::RANGE) {
      part_info->get_part_partition_id = part_info->column_list
          ? get_partition_id_range_col : get_partition_id_range;
      part_info->get_subpartition_id = part_info->list_of_subpart_fields
          ? (part_info->linear_hash_ind ? get_partition_id_linear_key_sub
                                        : get_partition_id_key_sub)
          : (part_info->linear_hash_ind ? get_partition_id_linear_hash_sub
                                        : get_partition_id_hash_sub);
    } else /* LIST */ { ... }
  } else {
    part_info->get_part_partition_id = nullptr;   // ★ 无子分区时这两个是 NULL
    part_info->get_subpartition_id = nullptr;
    if (part_info->part_type == partition_type::RANGE)
      part_info->get_partition_id = part_info->column_list
          ? get_partition_id_range_col : get_partition_id_range;
    else if (part_info->part_type == partition_type::LIST)
      part_info->get_partition_id = part_info->column_list
          ? get_partition_id_list_col : get_partition_id_list;
    else /* HASH */ {
      if (part_info->list_of_part_fields)         // ★ KEY 分区
        part_info->get_partition_id = part_info->linear_hash_ind
            ? get_partition_id_linear_key_nosub : get_partition_id_key_nosub;
      else
        part_info->get_partition_id = part_info->linear_hash_ind
            ? get_partition_id_linear_hash_nosub : get_partition_id_hash_nosub;
    }
  }
  // ★ 字符串字段需要 strnxfrm 归一化，多包一层
  if (part_info->part_charset_field_array) {
    part_info->get_part_partition_id_charset = part_info->get_part_partition_id;
    part_info->get_partition_id = get_part_id_charset_func_part;
  }
```

> ⚠️ **真名勘误**：不存在 `get_part_id_range` / `get_part_id_list` / `get_part_id_columns_range` / `get_part_id_columns_list`——真名是 **`get_partition_id_range()`** / **`get_partition_id_list()`** / **`get_partition_id_range_col()`** / **`get_partition_id_list_col()`**。

**HASH 与 KEY 的底层算子**：

```cpp
static int get_part_id_hash(uint num_parts, Item *part_expr, uint32 *part_id,
                            longlong *func_value) {
  if (part_val_int(part_expr, func_value)) return HA_ERR_NO_PARTITION_FOUND;
  longlong int_hash_id = *func_value % num_parts;
  *part_id = int_hash_id < 0 ? (uint32)-int_hash_id : (uint32)int_hash_id;  // ★ 负数取绝对值
  return false;
}

inline static uint32 get_part_id_key(handler *file, Field **field_array,
                                     uint num_parts, longlong *func_value) {
  *func_value = file->calculate_key_hash_value(field_array);   // ★★ KEY 把哈希交给存储引擎
  return (uint32)(*func_value % num_parts);
}
```

```cpp
static uint32 get_part_id_from_linear_hash(longlong hash_value, uint mask,
                                           uint num_parts) {
  uint32 part_id = (uint32)(hash_value & mask);
  if (part_id >= num_parts) {                    // ★ 越界 → mask 折半再取
    uint new_mask = ((mask + 1) >> 1) - 1;
    part_id = (uint32)(hash_value & new_mask);
  }
  return part_id;
}
void set_linear_hash_mask(partition_info *part_info, uint num_parts) {
  uint mask;
  for (mask = 1; mask < num_parts; mask <<= 1);  // ★ mask = 2^n - 1 且 2^n >= num_parts
  part_info->linear_hash_mask = mask - 1;
}
```

**★ RANGE 与 LIST 的 NULL 落点不同**——这是最容易记混的一条：

```cpp
// RANGE：NULL 恒落第 0 个分区（sql_partition.cc:3059）
int get_partition_id_range(partition_info *part_info, uint32 *part_id, longlong *func_value) {
  ...
  if (part_info->part_expr->null_value) {
    *part_id = 0;                       // ★★ RANGE 硬编码 0
    return 0;
  }

// LIST：NULL 落在 has_null_part_id（建表时记住的那个分区）
static int get_partition_id_list(partition_info *part_info, uint32 *part_id, ...) {
  if (part_info->part_expr->null_value) {
    if (part_info->has_null_value) {
      *part_id = part_info->has_null_part_id;   // ★★ LIST 用 has_null_part_id
      return 0;
    }
    goto notfound;
  }
```

> `has_null_part_id` 的**唯一赋值点**是 `partition_info.cc:1283`（`check_list_constants()` 遍历分区定义时记下哪个分区含 NULL）。

**★ `PARTITION_ITERATOR` 是 struct 不是枚举**（`partition_info.h:91`）——裁剪结果的载体：

```cpp
struct PARTITION_ITERATOR {
  partition_iter_func get_next;         // ★ 唯一的"取下一个分区"入口
  bool ret_null_part, ret_null_part_orig;  // ★ 是否额外吐出"装 NULL 的分区"
  struct st_part_num_range {             // ★ 分区号空间游标（via_mapping 用）
    uint32 start; uint32 cur; uint32 end;
  };
  struct st_field_value_range {          // ★ 字段值空间游标（via_walking 用）
    ulonglong start; ulonglong cur; ulonglong end;
  };
  union {
    struct st_part_num_range part_nums;
    struct st_field_value_range field_vals;
  };
  partition_info *part_info;
};
#define NOT_A_PARTITION_ID UINT_MAX32
```

```cpp
static inline void init_single_partition_iterator(uint32 part_id, PARTITION_ITERATOR *part_iter) {
  part_iter->part_nums.start = part_iter->part_nums.cur = part_id;
  part_iter->part_nums.end = part_id + 1;                  // ★ 单元素集合
  part_iter->get_next = get_next_partition_id_range;
}
static inline void init_all_partitions_iterator(partition_info *part_info, PARTITION_ITERATOR *part_iter) {
  part_iter->part_nums.start = part_iter->part_nums.cur = 0;
  part_iter->part_nums.end = part_info->num_parts;         // ★ 放弃裁剪的兜底
  part_iter->get_next = get_next_partition_id_range;
}
```

---

## 三、prune_partitions 主流程

`partition_pruning.cc:251`，核心步骤：

```cpp
// (1) prepare 期已裁剪完成则直接返回
if (part_info && part_info->is_pruning_completed) return false;
// (2) 无 WHERE → 全部标记
if (!pprune_cond) { mark_all_partitions_as_used(part_info); return false; }
// (3) 构造"分区索引描述"：把分区字段伪装成 index
if (create_partition_index_description(&prune_param)) { mark_all_partitions_as_used(...); return false; }

// (4) 关键：锁表前只能用常量条件（const_only）
const bool const_only = !thd->lex->is_query_tables_locked();
const table_map prev_tables = const_only ? 0 : INNER_TABLE_BIT;
const table_map read_tables = const_only ? 0 : INNER_TABLE_BIT;

range_par->keys = 1;                 // 只有一个"伪索引"
range_par->using_real_indexes = false;

// (5) 复用 range 优化器：把 WHERE 转成 SEL_TREE
tree = get_mm_tree(thd, range_par, prev_tables, read_tables, current_table,
                   /*remove_jump_scans=*/false, pprune_cond);
if (!tree) goto all_used;
if (tree->type == SEL_TREE::IMPOSSIBLE) { is_pruning_completed = true; goto end; }  // 全裁
if (tree->type != SEL_TREE::KEY) goto all_used;

// (6) 按 SEL_TREE 形态分派
if (tree->merges.is_empty())
  find_used_partitions(thd, &prune_param, tree->keys[0]);            // 单区间
else if (tree->merges.elements == 1)
  find_used_partitions_imerge(thd, &prune_param, tree->merges.head());  // OR
else
  find_used_partitions_imerge_list(thd, &prune_param, tree->merges);    // AND of OR

// (7) 收尾
bitmap_intersect(&read_partitions, &lock_partitions);   // 尊重显式 PARTITION(pX)
if (!is_query_tables_locked() && !partition_key_modified(table, write_set))
  bitmap_copy(&lock_partitions, &read_partitions);      // 同步锁分区
if (bitmap_is_clear_all(&read_partitions))
  table->all_partitions_pruned_away = true;             // 全裁掉
```

**关键点**：

1. **const_only**：`!is_query_tables_locked()` 决定。锁表前 `prev_tables/read_tables=0`，`get_mm_tree` 遇到引用其他表的条件只能返回 MAYBE_KEY；锁表后才允许 `INNER_TABLE_BIT`。
2. **两轮裁剪**：prepare 期裁常量（锁裁剪），optimize 期裁非静态（读裁剪），`is_pruning_completed` 避免重复。
3. **三种 SEL_TREE 形态**：`merges` 空（单区间）/ 单 imerge（OR）/ 多 imerge（AND of OR），对应 `find_used_partitions` / `_imerge` / `_imerge_list`。

**★ 完整函数体**（`partition_pruning.cc:251`，含原注释的骨架）：

```cpp
bool prune_partitions(THD *thd, TABLE *table, Query_block *query_block,
                      Item *pprune_cond) {
  partition_info *part_info = table->part_info;

  /*
    If the prepare stage already have completed pruning successfully,
    it is no use of running prune_partitions() again on the same condition.
  */
  if (part_info && part_info->is_pruning_completed) return false;   // ★ 二次裁剪短路

  table->all_partitions_pruned_away = false;
  if (!part_info) return false; /* not a partitioned table */

  if (table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION &&
      part_info->is_auto_partitioned)
    return false; /* Should not prune auto partitioned table */       // ★ NDB

  if (!pprune_cond) { mark_all_partitions_as_used(part_info); return false; }

  if (bitmap_is_clear_all(&part_info->read_partitions)) {
    table->all_partitions_pruned_away = true;                        // ★ 全裁完
    return false;
  }

  PART_PRUNE_PARAM prune_param;
  MEM_ROOT alloc(key_memory_partitions_prune_exec,
                 thd->variables.range_alloc_block_size);
  RANGE_OPT_PARAM *range_par = &prune_param.range_param;
  ...
  alloc.set_max_capacity(thd->variables.range_optimizer_max_mem_size);
  alloc.set_error_for_capacity_exceeded(true);

  if (create_partition_index_description(&prune_param)) {
    mark_all_partitions_as_used(part_info);                          // ★ 造不出伪索引 → 全选
    thd->pop_internal_handler();
    return false;
  }

  dbug_tmp_use_all_columns(table, old_sets, table->read_set, table->write_set);
  //  ★★ 临时把 read_set/write_set 全打开，保证分区表达式可求值
  const bool const_only = !thd->lex->is_query_tables_locked();
  range_par->keys = 1;  // one index
  range_par->using_real_indexes = false;                             // ★ 明确不是真实索引
  bitmap_clear_all(&part_info->read_partitions);                     // ★ 清空，随后逐位点亮

  SEL_TREE *tree = get_mm_tree(thd, range_par, prev_tables, read_tables,
                               current_table,
                               /*remove_jump_scans=*/false, pprune_cond);
  //                             ★★ 注意 false：jump scan 对裁剪有用，对普通 range 优化无用
  if (!tree) goto all_used;
  if (tree->type == SEL_TREE::IMPOSSIBLE) {
    part_info->is_pruning_completed = true;
    goto end;
  }
  if (tree->type != SEL_TREE::KEY) goto all_used;
  ...
  /*
    Decide if the current pruning attempt is the final one.
    During the prepare phase, before locking, subqueries and stored programs
    are not evaluated. So we need to run prune_partitions() a second time in
    the optimize phase ...
  */
  if (pprune_cond->const_item() || !pprune_cond->const_for_execution() ||
      thd->lex->is_query_tables_locked())
    part_info->is_pruning_completed = true;      // ★★ 静态 vs 动态的真正判据
  goto end;

all_used:
  mark_all_partitions_as_used(prune_param.part_info);
end:
  thd->pop_internal_handler();
  dbug_tmp_restore_column_maps(table->read_set, table->write_set, old_sets);
  if (thd->is_error()) return true;
  bitmap_intersect(&prune_param.part_info->read_partitions,
                   &prune_param.part_info->lock_partitions);   // ★ read ⊆ lock
  if (!thd->lex->is_query_tables_locked() &&
      !partition_key_modified(table, table->write_set)) {      // ★ 未锁表且没改分区键
    bitmap_copy(&prune_param.part_info->lock_partitions,
                &prune_param.part_info->read_partitions);      // ★ lock 也跟着收缩
  }
  if (bitmap_is_clear_all(&(prune_param.part_info->read_partitions)))
    table->all_partitions_pruned_away = true;
  return false;
}
```

```cpp
static void mark_all_partitions_as_used(partition_info *part_info) {
  bitmap_copy(&(part_info->read_partitions), &(part_info->lock_partitions));
}   // ★ 注意：是 copy lock→read，不是"全置 1"
```

```cpp
bool partition_key_modified(TABLE *table, const MY_BITMAP *fields) {
  partition_info *part_info = table->part_info;
  if (!part_info) return false;
  if (table->s->db_type()->partition_flags &&
      (table->s->db_type()->partition_flags() & HA_CAN_UPDATE_PARTITION_KEY))
    return false;                          // ★ 引擎声明能改分区键 → 放行
  for (Field **fld = part_info->full_part_field_array; *fld; fld++)
    if (bitmap_is_set(fields, (*fld)->field_index())) return true;
  return false;
}
```

**★ `create_partition_index_description()`：分区字段 → 虚拟 KEY_PART 数组**（`partition_pruning.cc:1074`）：

```cpp
  used_part_fields = fields_ok_for_partition_index(part_info->part_field_array)
                         ? part_info->num_part_fields : 0;   // ★ 一旦拒绝整层归 0
  used_subpart_fields =
      fields_ok_for_partition_index(part_info->subpart_field_array)
          ? part_info->num_subpart_fields : 0;
  uint total_parts = used_part_fields + used_subpart_fields;
  ...
  if (!total_parts || !(key_part = (KEY_PART *)alloc->Alloc(sizeof(KEY_PART) * total_parts)) || ...)
    return true;                 // ★ total_parts == 0 → 直接放弃整个裁剪

  for (uint part = 0; part < total_parts; part++, key_part++) {
    key_part->key = 0;                                            // ★ 固定为 0：虚拟索引号
    key_part->part = part;
    key_part->length = (uint16)(*field)->key_length();
    key_part->store_length = (uint16)get_partition_field_store_length(*field);
    key_part->field = (*field);
    key_part->image_type = Field::itRAW;                          // ★ 统一按 RAW 字节像
    key_part->flag = 0;   /* We set keypart flag to 0 here as the only
                             HA_PART_KEY_SEG is checked in the RangeAnalysisModule. */
    ...
    if (!*(++field)) { field = part_info->subpart_field_array; in_subpart_fields = true; }
  }
```

**★ 伪索引只拒绝两种类型**（`fields_ok_for_partition_index`，`:1042`）：

```cpp
/*
    We can't process GEOMETRY fields - for these fields singlepoint intervals
    can't be generated, and non-singlepoint are "special" kinds of intervals
    to which our processing logic can't be applied.

    It is not known if we could process ENUM fields, so they are disabled to be
    on the safe side.
*/
static bool fields_ok_for_partition_index(Field **pfield) {
  if (!pfield) return false;
  for (; (*pfield); pfield++) {
    enum_field_types ftype = (*pfield)->real_type();
    if (ftype == MYSQL_TYPE_ENUM || ftype == MYSQL_TYPE_GEOMETRY) return false;
    //  ★★ 只拒 ENUM 与 GEOMETRY（用 real_type() 剥掉 ENUM 的整型外衣）
  }
  return true;
}
```

> ⚠️ **勘误**：BIT / SET 不是在这里被拒的——它们在**更早的建表阶段**就被 `check_part_field()`（`sql_partition.cc:1939`）整体禁止进分区表达式，错误码 `ER_FIELD_TYPE_NOT_ALLOWED_AS_PARTITION_FIELD`。所以三层拦截是：**建表 `check_part_field`（拒 BIT/SET/ENUM/GEOMETRY/BLOB）→ 伪索引（只再拒 ENUM/GEOMETRY）→ `set_up_range_analysis_info`（非整数型拿不到 walking）**。

**★ `PART_PRUNE_PARAM` 裁剪上下文**（`partition_pruning.cc:147`）——注意结果以**迭代器**承载而非直接写 bitmap：

```cpp
struct PART_PRUNE_PARAM {
  RANGE_OPT_PARAM range_param;
  partition_info *part_info;
  get_part_id_func get_top_partition_id_func;   // ★ 有子分区=get_part_partition_id，否则=get_partition_id
  mark_full_part_func mark_full_partition_used; // ★ 经函数指针分派（mark_partition_as_used 不存在）
  KEY_PART *key;                                 // ★ 虚拟索引描述
  uint part_fields; uint subpart_fields;
  int last_part_partno; int last_subpart_partno;
  bool *is_part_keypart; bool *is_subpart_keypart;
  bool ignore_part_fields;
  SEL_ARG **arg_stack; SEL_ARG **arg_stack_end;  // ★ 递归栈
  uint cur_part_fields; uint cur_subpart_fields;
  PARTITION_ITERATOR part_iter;                  // ★ 当前"已确定的分区集合"
  MY_BITMAP subparts_bitmap;
  uchar min_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];
  uchar max_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];
  uchar *cur_min_key; uchar *cur_max_key;
  uint cur_min_flag, cur_max_flag;
};
```

---

## 四、find_used_partitions：遍历 SEL_ARG 树

`partition_pruning.cc:713`，**递归遍历 SEL_ARG 树**。关键：SEL_ARG 的 `next_key_part` 是 AND（纵向深入），`left/right`（红黑树）是 OR（横向遍历同 key part 的多个区间）。

```cpp
static int find_used_partitions(THD *thd, PART_PRUNE_PARAM *ppar, SEL_ARG *key_tree) {
  // (1) 先遍历左子树（同 key part 的其他区间，OR）
  if (key_tree->left != null_element)
    left_res = find_used_partitions(thd, ppar, key_tree->left);

  // (2) 入栈：记录当前 key part 属于分区键还是子分区键
  ppar->cur_part_fields += ppar->is_part_keypart[key_tree->part];
  ppar->cur_subpart_fields += ppar->is_subpart_keypart[key_tree->part];
  *(ppar->arg_stack_end++) = key_tree;

  // (3) 等值单点：所有分区字段都是 field=const → 直接算分区 id
  if (key_tree->is_singlepoint() &&
      key_tree_part == ppar->last_part_partno &&
      ppar->cur_part_fields == ppar->part_fields &&
      ppar->get_part_iter_for_interval == nullptr) {
    store_selargs_to_rec(ppar, ppar->arg_stack, ppar->part_fields);  // 常量写进 record
    ppar->get_top_partition_id_func(ppar->part_info, &part_id, &func_value);  // 算分区
    init_single_partition_iterator(part_id, &ppar->part_iter);
  }
  // (4) 区间映射：RANGE/LIST 且支持 interval analysis
  else if (ppar->part_info->get_part_iter_for_interval && key_tree->part <= last_part_partno) {
    key_tree->store_min_value(...); key_tree->store_max_value(...);
    res = ppar->part_info->get_part_iter_for_interval(...);  // 调区间映射函数
  }
  // (5) 非单点且处理到子分区层 → 放弃（整个分区标记使用）

  // (6) 沿 next_key_part 深入（AND）
  if (key_tree->next_key_part)
    res = find_used_partitions(thd, ppar, key_tree->next_key_part);

  // (7) 出栈 + 遍历右子树
  ppar->arg_stack_end--;
  if (key_tree->right != null_element)
    right_res = find_used_partitions(thd, ppar, key_tree->right);
  return (left_res || right_res || res);
}
```

**两条路径**：

1. **等值单点**：凑齐所有分区字段的等值约束（`cur_part_fields == part_fields`），直接调 `get_partition_id` 精确算出分区——**HASH/KEY 分区的等值条件就走这条路**。
2. **区间映射**：`get_part_iter_for_interval != nullptr`（RANGE/LIST 且分区函数单调），把 SEL_ARG 的 min/max 边界拷入缓冲，调区间映射函数得到命中分区迭代器。

**标记函数**（`mark_full_partition_used`，:491）：位图是**叶子级**的，有子分区时分区 `part_id` 的叶子范围是 `[part_id*num_subparts, (part_id+1)*num_subparts)`。

---

## 五、各分区类型的裁剪逻辑

### 5.1 RANGE 分区 —— 二分查找

`get_partition_id_range`（`sql_partition.cc:3059`）：`range_int_array[i]` 是第 i 个分区的**上界**（LESS THAN 值），二分找第一个 `range_array[i] > part_func_value` 的下标。

区间映射 `get_partition_id_range_for_endpoint`（:3152）找"与给定区间有非空交集的分区子数组边界"，处理开闭区间（`left_endpoint` / `include_endpoint` 的边界语义）。

### 5.2 LIST 分区 —— 二分查找

`get_partition_id_list`（:2841）：`list_array` 按 `list_value` 有序（`fix_partition_func` 里排序），二分找相等的 list_value，命中返回 `partition_id`。NULL 值单独处理（`has_null_part_id`）。

### 5.3 HASH/KEY 分区 —— 等值可裁，范围难裁

> ⚠️ **纠正常见误解**：HASH/KEY 分区的**等值条件**（`hash_key = const`）可以精确裁剪到 1 个分区（走单点路径）。真正"全裁或全不裁"的是**范围条件**，因为 hash 结果均匀散布、无单调性。

`get_part_id_hash`（:2555）：`int_hash_id = part_func_value % num_parts`。

`set_up_range_analysis_info`（:5358）决定区间分析能力：

```cpp
switch (part_info->part_type) {
  case RANGE: case LIST:
    if (!column_list) {
      if (part_expr->get_monotonicity_info() != NON_MONOTONIC)
        get_part_iter_for_interval = get_part_iter_for_interval_via_mapping;  // 二分映射
    }
  default:;  // HASH/KEY 落到这里
}
// HASH/KEY：仅单整型字段时用 via_walking（逐个值枚举）
if (num_part_fields == 1 && field->type() 是整数类型)
  get_part_iter_for_interval = get_part_iter_for_interval_via_walking;
```

`get_part_iter_for_interval_via_walking`（:5890）末尾的阈值判断是"全裁或全不裁"的根源：

```cpp
ulonglong n_values = b - a;
if ((n_values > 2 * total_parts) && n_values > MAX_RANGE_TO_WALK) return -1;  // 枚举量超阈值
```

返回 `-1` 意味着"无法推理出部分分区 → 全部标记使用"。所以 HASH/KEY 范围条件：**小范围**整数区间可枚举出命中分区（部分裁剪），**大范围/非整数**直接全不裁。

**★ `MAX_RANGE_TO_WALK = 32`**（`sql_partition.cc:5849`，普通 `#define` 非 constexpr）：

```cpp
/* See get_part_iter_for_interval_via_walking for definition of what this is */
#define MAX_RANGE_TO_WALK 32
```

> ★★ 两个条件**同时成立**才放弃：`n_values > 2 × 分区数` **且** `n_values > 32`。所以 HASH 4 分区下 `x BETWEEN 1 AND 20`（n=20）：`20 > 8` 成立但 `20 > 32` 不成立 → **仍然 walking**；`BETWEEN 1 AND 100` 才放弃。

**⚠️ 重大勘误：8.0.39 里只剩 3 个区间映射函数**（`partition_info.h:135` 的官方注释原文）：

```cpp
  @note
    There are currently three functions of this type:
     - get_part_iter_for_interval_via_walking
     - get_part_iter_for_interval_cols_via_map
     - get_part_iter_for_interval_via_mapping
```

> 旧文里的 `_hash_via_map` / `_key_via_map` / `_linear_hash_via_map` / `_linear_key_via_map` / `_hash_via_array` / `_list_via_map` / `_cols_list_via_map` **全部不存在**（5.7 已删除并合并进 `via_walking`）。
> ⚠️ 后缀是 **`_via_walking`**（带 `-ing`），不是 `_via_walk`；`get_next_partition_via_walk` 的真名也是 **`get_next_partition_via_walking`**。

**★★ 三条路径总览**（dispatch 的唯一实现是 `set_up_range_analysis_info()`，**不存在 `get_partition_iter_for_interval()`**）：

| 分区类型 | 条件 | 选中的函数 | 复杂度 |
|---|---|---|---|
| RANGE(expr) / LIST(expr) | `part_expr` 单调 | `..._via_mapping` | **O(log n)** 二分 × 2 端 |
| RANGE/LIST COLUMNS | 总是 | `..._cols_via_map` | **O(log n)** 二分 × 2 端 |
| HASH / KEY / LINEAR HASH / LINEAR KEY | 单整数型字段 | `..._via_walking` | **O(区间宽度)**，超限放弃 |
| 非单调 RANGE/LIST(expr) | 单整数型字段 | `..._via_walking` | 同上 |
| 其它（字符串/DECIMAL/BIT/ENUM/SET/多字段） | — | **`nullptr`** | **完全不裁**（只走单点路径） |

```cpp
static void set_up_range_analysis_info(partition_info *part_info) {
  /* Set the catch-all default */
  part_info->get_part_iter_for_interval = nullptr;   // ★ nullptr = 该表完全不支持区间裁剪
  part_info->get_subpart_iter_for_interval = nullptr;

  switch (part_info->part_type) {
    case partition_type::RANGE:
    case partition_type::LIST:                       // ★ 只有 RANGE/LIST 能进 mapping
      if (!part_info->column_list) {
        if (part_info->part_expr->get_monotonicity_info() != NON_MONOTONIC) {
          part_info->get_part_iter_for_interval =
              get_part_iter_for_interval_via_mapping;
          goto setup_subparts;                       // ★ goto：选中 (1) 后不再试 (3)
        }
      } else {
        part_info->get_part_iter_for_interval =
            get_part_iter_for_interval_cols_via_map;
        goto setup_subparts;
      }
    default:;                                        // ★ HASH/KEY（part_type==HASH）落到这里
  }

  if (part_info->num_part_fields == 1) {             // ★ 硬性前提：只用到 1 个字段
    Field *field = part_info->part_field_array[0];
    switch (field->type()) {                         // ★ 且必须是整数型
      case MYSQL_TYPE_TINY: case MYSQL_TYPE_SHORT: case MYSQL_TYPE_INT24:
      case MYSQL_TYPE_LONG: case MYSQL_TYPE_LONGLONG:
        part_info->get_part_iter_for_interval =
            get_part_iter_for_interval_via_walking;
        break;
      default:;                                      // ★ 其它类型保持 nullptr
    }
  }
setup_subparts:
  if (part_info->num_subpart_fields == 1) {
    Field *field = part_info->subpart_field_array[0];
    switch (field->type()) {
      case MYSQL_TYPE_TINY: case MYSQL_TYPE_SHORT:
      case MYSQL_TYPE_LONG:                          // ⚠️ 子分区这里漏了 INT24（8.0.39 现状）
      case MYSQL_TYPE_LONGLONG:
        part_info->get_subpart_iter_for_interval =
            get_part_iter_for_interval_via_walking;  // ★ 子分区只有 walking 一种
        break;
      default:;
    }
  }
}
```

> ★ **子分区永远只有 `via_walking` 一种实现**，`via_mapping` 里有硬断言 `assert(!is_subpart)`。

**★★ "HASH/KEY 只能裁 IS NULL" 的源码依据**——`via_walking` 里唯一的成功分支就是这个：

```cpp
  /* Handle the "t.field IS NULL" interval, it is a special case */
  if (field->is_nullable() && !(flags & (NO_MIN_RANGE | NO_MAX_RANGE)) &&
      *min_value && *max_value) {          // ★ 两端都为 NULL 标记字节 == "IS NULL"
    /*
      We don't have a part_iter->get_next() function that would find which
      partition "t.field IS NULL" belongs to, so find partition that contains
      NULL right here, and return an iterator over singleton set.
    */
    uint32 part_id;
    field->set_null();                     // ★ 把字段置 NULL，直接调分区函数求值
    ...
    if (!res) {
      init_single_partition_iterator(part_id, part_iter);   // ★★ 单元素迭代器
      return 1;
    }
    return 0; /* No partitions match */
  }

  if ((field->is_nullable() &&
       ((!(flags & NO_MIN_RANGE) && *min_value) ||   // NULL <? X
        (!(flags & NO_MAX_RANGE) && *max_value))) || // X <? NULL
      (flags & (NO_MIN_RANGE | NO_MAX_RANGE)))       // ±inf 在任一端
  {
    return -1; /* Can't handle this interval, have to use all partitions */
  }
```

> ★★ 除 `IS NULL` 外，区间分支要么返回 `0`（无分区）要么返回 `-1`（放弃）。**这就是"HASH/KEY 基本只能裁 `IS NULL`"的完整证据**。

**walking 的枚举成本**（`get_next_partition_via_walking`，`:6067`）——每个值都要真实调一次分区函数：

```cpp
static uint32 get_next_partition_via_walking(PARTITION_ITERATOR *part_iter) {
  uint32 part_id;
  Field *field = part_iter->part_info->part_field_array[0];
  while (part_iter->field_vals.cur != part_iter->field_vals.end) {
    longlong dummy;
    field->store(part_iter->field_vals.cur++, field->is_flag_set(UNSIGNED_FLAG));
    if ((part_iter->part_info->is_sub_partitioned() &&
         !part_iter->part_info->get_part_partition_id(part_iter->part_info,
                                                      &part_id, &dummy)) ||
        !part_iter->part_info->get_partition_id(part_iter->part_info, &part_id,
                                                &dummy))
      return part_id;              // ★ 每值一次 HASH 取模 / KEY 哈希
  }
  part_iter->field_vals.cur = part_iter->field_vals.start;  // ★ 自动复位，迭代器可重用
  return NOT_A_PARTITION_ID;
}
```

> ★★ **为什么 HASH/KEY 不能像 RANGE 那样二分**：`hash(v) % n` **不保序**，`v` 空间的连续区间在分区号空间里没有任何结构。唯一办法就是逐个值代入算一遍再去重。

对比 **LIST 的真 O(1)**（`get_next_partition_id_list`，`:6057`）——这才是 "via_map" 的含义：

```cpp
  } else {
    partition_info *part_info = part_iter->part_info;
    uint32 num_part = part_iter->part_nums.cur++;
    if (part_info->column_list) {
      uint num_columns = part_info->part_field_list.elements;
      return part_info->list_col_array[num_part * num_columns].partition_id;
    }
    return part_info->list_array[num_part].partition_id;
    //  ★ 纯数组下标 O(1)；O(log n) 的成本已在端点二分时付过
  }
```

**★ `enum monotonicity_info`（真名，非 `MONOTONICITY_*`）**——`sql/item.h:582`，5 个值：

```cpp
typedef enum monotonicity_info {
  NON_MONOTONIC,                          /* none of the below holds */
  MONOTONIC_INCREASING,                   /* F() is unary and (x < y) => (F(x) <= F(y)) */
  MONOTONIC_INCREASING_NOT_NULL,          /* But only for valid/real x and y */
  MONOTONIC_STRICT_INCREASING,            /* F() is unary and (x < y) => (F(x) <  F(y)) */
  MONOTONIC_STRICT_INCREASING_NOT_NULL    /* But only for valid/real x and y */
} enum_monotonicity_info;
```

**⚠️ 全库只有 6 处 override**（这是分区函数能否用 `via_mapping` 的全部可能性）：

| 位置 | 返回 |
|---|---|
| `Item`（虚基类默认，`item.h:1698`） | `NON_MONOTONIC` |
| **`Item_field`**（`item.h:4337`） | `MONOTONIC_STRICT_INCREASING` ★ `PARTITION BY RANGE(int_col)` 能二分全靠这行 |
| `Item_func_to_days` | DATE→`STRICT_INCREASING_NOT_NULL`；DATETIME→`INCREASING_NOT_NULL` |
| `Item_func_to_seconds` | DATE/DATETIME→`STRICT_INCREASING_NOT_NULL` |
| `Item_func_year` | DATE/DATETIME→`MONOTONIC_INCREASING` |
| `Item_func_unix_timestamp` | TIMESTAMP→`MONOTONIC_INCREASING` |

```cpp
  virtual enum_monotonicity_info get_monotonicity_info() const {
    return NON_MONOTONIC;      // ★ 默认；Item_int / Item_func_plus 等全部继承这个
  }
  virtual longlong val_int_endpoint(bool left_endp, bool *incl_endp) {
    assert(0);                 // ★ 只有单调函数才允许调用
    return 0;
  }
```

> ★★ **实战结论**：`PARTITION BY HASH(col+1)` 里的 `col+1` 是 `Item_func_plus`，**没有 override** → `NON_MONOTONIC` → 只能走 walking。`Item_int` 也没有 override。

**非严格单调要把 `<` 降级为 `<=`**（`Item_func_year::val_int_endpoint` 的注释原文）：

```
      col < '2007-01-01 00:00:00'  -> YEAR(col) <  2007     ← 年边界，保 "<"
      col < '2007-09-15 23:00:00'  -> YEAR(col) <= 2007     ← 非年边界，降级为 "<="
```

```cpp
  if (!left_endp && ltime.day == 1 && ltime.month == 1 &&
      !(ltime.hour || ltime.minute || ltime.second || ltime.second_part))
    ; /* do nothing */
  else
    *incl_endp = true;         // ★★ 非年边界 → "<" 必须降级为 "<="
  return ltime.year;
```

**单调但可返 NULL → 必须额外扫 NULL 分区**（`via_mapping`，`:5756`）：

```cpp
    monotonic = part_info->part_expr->get_monotonicity_info();
    if (monotonic == MONOTONIC_INCREASING_NOT_NULL ||
        monotonic == MONOTONIC_STRICT_INCREASING_NOT_NULL) {
      /* col is NOT NULL, but F(col) can return NULL, add NULL partition */
      part_iter->ret_null_part = part_iter->ret_null_part_orig = true;
      check_zero_dates = true;
    }
```

> ★ 典型场景：`TO_DAYS('2000-00-00')` 返回 NULL（但 `'2000-00-00'` 本身可与 `'2000-01-01'` 比较）。源码注释逐字写了这个例子。

### 5.4 子分区的递归裁剪

子分区不是单独函数，**内嵌在 `find_used_partitions` 的同一棵 SEL_ARG 树遍历里**。`create_partition_index_description`（:1096）把分区字段和子分区字段**拼接成一个伪复合索引**，`key_tree->part` 编号 0..`last_part_partno` 是分区字段，之后是子分区字段。遍历到分区层确定分区集，继续沿 `next_key_part` 深入确定子分区集，再对每个分区 × 命中子分区标记叶子位。

---

## 六、与 SEL_TREE 的复用关系

**分区裁剪确实调用 `get_mm_tree`**（`partition_pruning.cc:325`），复用其产出的 SEL_TREE：

```
WHERE 条件
  → get_mm_tree()              ← 复用 range 分析（range_analysis.cc:845）
  → SEL_TREE（分区键上的区间图）
  → find_used_partitions()     ← 自己遍历 SEL_ARG 树（不调 check_quick_select）
  → read_partitions 位图
```

区别：range 优化器用 `check_quick_select`/`get_ranges_from_tree` 生成 AccessPath；分区裁剪用 `find_used_partitions` 把区间映射成分区号。两者共用 `SEL_ARG` 的边界导出函数 `store_min_value`/`store_max_value`。

---

## 七、结果使用与 EXPLAIN

### 7.1 位图如何传给 handler

`set_partition_bitmaps`（`partition_info.cc:262`）：显式 `PARTITION(pX)` 子句只设命名分区，否则 `bitmap_set_all`。

`Partition_helper::ph_rnd_init`（`partition_handler.cc:1427`）遍历位图：

```cpp
part_id = m_part_info->get_first_used_partition();   // 第一个命中分区
if (MY_BIT_NONE == part_id) { error = 0; goto err1; }  // 全裁 → 空扫描
for (i = part_id; i < MY_BIT_NONE; i = get_next_used_partition(i))
  rnd_init_in_part(i, scan);                          // 逐个分区初始化扫描
```

### 7.2 EXPLAIN 的 partitions 列

`make_used_partitions_str`（`sql_partition.cc:5277`）遍历 `read_partitions` 位图，把命中分区/子分区名拼成字符串（子分区格式 `分区名_子分区名`），由 `opt_explain.cc:933` 输出。

### 7.3 prune_table_partitions vs prune_partitions

`JOIN::prune_table_partitions`（`sql_optimizer.cc:2801`）是 optimize 阶段的**调度器**：遍历叶子表，从 join nest 层次挑出最外层作用于该表的谓词，再调 `prune_partitions`。真正算法全在 `prune_partitions`。

---

## 八、server 层分区机制：打开 / DML 分发 / 接口

裁剪（前七节）只是分区表的"读路径优化"。本节点出位图之后的下游：**表如何打开分区、DML 如何分发、server 与 `ha_innopart` 的接口**。

### 8.1 sql/partitioning/ 目录清单

| 文件 | 职责 |
|------|------|
| `partition_handler.h/.cc` | server 写的分区引擎适配层：`Partition_helper`（DML 分发/扫描/有序归并/auto_inc 缓存）+ `Partition_handler`（引擎必须实现的抽象接口）+ `Partition_share`（跨实例共享分区名 hash、auto_inc mutex） |
| `partition_info.h/.cc` | `partition_info`：分区元数据、id 计算函数指针、read/lock 位图、`can_prune_insert` |
| `partition_element.h` | 每分区描述：`partition_name`/`range_value`/`list_val_list`/`subpartitions`/`part_state` |
| `sql_partition.h/.cc` | 分区 DDL 反解/装配：`mysql_unpack_partition`、`fix_partition_func`、id 计算函数族、`get_parts_for_update` |
| `sql_partition_admin.cc` | EXCHANGE/TRUNCATE/REORGANIZE PARTITION 等管理命令 |
| `range_optimizer/partition_pruning.cc` | 裁剪本身（前七节） |
| `ha_innopart.cc/.h`（storage/innobase） | InnoDB 分区 handler：继承 `ha_innobase` + `Partition_helper` |

> ⚠️ `partition_base.cc` 在 8.0 已不存在；`get_partition_row_count` 是 ha_partition 时代接口已移除，统计走 `get_dynamic_partition_info`（`partition_handler.h:209`）。

### 8.2 partition_info 关键成员与表打开流程

**关键成员**（`partition_info.h`）：

- `get_partition_id`（h:232）/ `get_part_partition_id`（h:242）/ `get_subpartition_id`（h:251）——**函数指针**，按类型在 `set_up_partition_func_pointers`（`sql_partition.cc:1226-1279`）装配（RANGE→`get_partition_id_range`、LIST→`get_partition_id_list`、HASH/KEY→hash 函数）
- `full_part_field_set`（h:278）——要求 UPDATE 时 read_set 必须含全部分区字段（跨分区移动需要）
- `read_partitions`/`lock_partitions` 位图（h:317-318）——裁剪的输出
- `TABLE::part_info`（`table.h:1874`）挂 TABLE 上

**打开流程（5 步）**：

1. `TABLE_SHARE` 从 DD 取序列化串 `partition_info_str`（`dd_table_share.cc:2143-2147`）
2. `open_table_from_share`（`table.cc:2926`）→ `get_new_handler(..., partitioned)` → `innobase_create_handler`（`ha_innodb.cc:1764-1775`）：partitioned=true 时 `new ha_innopart`（**双继承 `ha_innobase` + `Partition_helper(this)`**）
3. `unpack_partition_info`（`table.cc:3017`）：把 DD 元数据"反编译"成分区语法串**重解析**（`table.cc:2700-2725` 注释），`mysql_unpack_partition` 重建 partition_element 列表，`set_part_info(part_info, true)`（early）
4. `fix_partition_func`（`sql_partition.cc:1470`）：`set_up_field_array` → `set_up_partition_key_maps`（算 `all_fields_in_PF`，供裁剪/点查快速判断）→ `set_up_partition_func_pointers` → `part_handler->set_part_info(part_info, false)`
5. `ha_innopart::open`（`ha_innopart.cc:781`）：`open_table_parts` 打开每个分区的 dict_table_t（:829）、`populate_partition_name_hash`（:851）、`init_auto_inc_mutex`（:855）、`open_partitioning`（:873，断言 `m_part_info == m_table->part_info`）

### 8.3 DML 分发（含 UPDATE 跨分区移动）

**⚠️ 勘误：UPDATE 跨分区移动是"先 insert 后 delete"，不是"先 delete 再 insert"**——这样即使失败也不会丢行。而且它发生在 **handler 层 `Partition_helper::ph_update_row()`**，不是 server 层：

```cpp
int Partition_helper::ph_update_row(const uchar *old_data, uchar *new_data) {
  assert(bitmap_is_subset(&m_part_info->full_part_field_set, m_table->read_set));
  if ((error = get_parts_for_update(old_data, new_data, m_table->record[0],
                                    m_part_info, &old_part_id, &new_part_id,
                                    &func_value)))
    return error;
  if (!bitmap_is_set(&(m_part_info->lock_partitions), new_part_id)) {
    return HA_ERR_NOT_IN_LOCK_PARTITIONS;      // ★ 新分区必须已加锁
  }
  if (old_part_id != m_last_part) {
    m_err_rec = old_data;
    return HA_ERR_ROW_IN_WRONG_PARTITION;      // ★ 行放错分区 → 报错，不自动纠正
  }
  m_last_part = new_part_id;
  if (new_part_id == old_part_id) {
    error = update_row_in_part(new_part_id, old_data, new_data);   // 同分区：原地更新
  } else {
    Field *saved_next_number_field = m_table->next_number_field;
    /* Don't allow generation of auto_increment value for update. ... */
    m_table->next_number_field = nullptr;      // ★★ 禁止移动时重新生成 auto_inc
    error = write_row_in_part(new_part_id, new_data);    // ★★ ① 先 INSERT 新分区
    m_table->next_number_field = saved_next_number_field;
    if (!error) {
      error = delete_row_in_part(old_part_id, old_data); // ★★ ② 再 DELETE 旧分区
    }
  }
  ...
}
```

**★ 三种 DML 的权限校验对比**（`partition_handler.cc`）——这是 `read_partitions` 与 `lock_partitions` 分工的直接体现：

| 操作 | 校验 `lock_partitions` | 校验 `read_partitions` |
|---|---|---|
| `ph_write_row` | ✅ `is_partition_locked(part_id)` | ❌ |
| `ph_update_row` | ✅ 新分区必须已锁 | ❌ |
| `ph_delete_row` | ✅ | ✅ 额外 `assert(is_partition_used())` |

```cpp
int Partition_helper::ph_write_row(uchar *buf) {
  ...
  error = m_part_info->get_partition_id(m_part_info, &part_id, &func_value);
  if (!m_part_info->is_partition_locked(part_id)) {     // ★ INSERT 用 lock 校验
    error = HA_ERR_NOT_IN_LOCK_PARTITIONS;
    goto exit;
  }
  m_last_part = part_id;
  error = write_row_in_part(part_id, buf);
```

> ★ 这正是 `partition_info.h:299` 注释所说：*"in case of UPDATE the WHERE clause can limit the read_partitions set, but not necessarily the lock_partitions set"*。

**★ INSERT 的独立裁剪通道**（`sql_insert.cc:1416`，**不调 `prune_partitions()`**）：

```cpp
  MY_BITMAP used_partitions;         // ★★ 这就是"第三个 bitmap"的真相：局部变量
  bool prune_needs_default_values = false;
  if (insert_table->part_info->can_prune_insert(
          thd, duplicates, update, update_field_list, insert_field_list,
          value_count == 0, &can_prune_partitions,
          &prune_needs_default_values, &used_partitions))
    return true;

  if (can_prune_partitions != partition_info::PRUNE_NO) {
    /* PRUNE_DEFAULTS means the partitioning fields are only set to DEFAULT
       values, so we only need to check the first INSERT value, since all the
       rest will be in the same partition. */
    ...
    while (its != insert_many_values.end()) {
      if (can_prune_partitions == partition_info::PRUNE_YES) {
        insert_table->part_info->set_used_partition(
            thd, insert_field_list, *values, info,
            prune_needs_default_values, &used_partitions);   // ★ 逐行算 part_id
        if (!(counter % num_partitions)) {
          if (bitmap_is_set_all(&used_partitions))
            can_prune_partitions = partition_info::PRUNE_NO;  // ★ 都用上了 ⇒ 放弃裁剪
        }
      }
    }
  }
  if (can_prune_partitions != partition_info::PRUNE_NO) {
    bitmap_intersect(&insert_table->part_info->read_partitions, &used_partitions);
    bitmap_intersect(&insert_table->part_info->lock_partitions, &used_partitions);
    //  ★★ INSERT 裁剪的落地：read &= used，lock &= used
  }
```

```cpp
  enum enum_can_prune { PRUNE_NO = 0, PRUNE_DEFAULTS, PRUNE_YES };
```

**★ `can_prune_insert()` 的全部拒绝条件**（`partition_info.cc:311`）——这是 INSERT 裁剪为何常常不生效的答案：

```cpp
  if (table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION)
    return false;                                    // ① NDB 自动分区

  if (table->triggers) {
    Trigger_chain *trigger_chain =
        table->triggers->get_triggers(TRG_EVENT_INSERT, TRG_ACTION_BEFORE);
    if (trigger_chain &&
        trigger_chain->has_updated_trigger_fields(&full_part_field_set))
      return false;                                  // ② BEFORE INSERT 触发器改分区列
  }

  if (table->vfield) { /* 生成列在分区表达式里 */ return false; }        // ③
  if (table->gen_def_fields_ptr) { /* 生成默认表达式 */ return false; }  // ④

  if (table->found_next_number_field) {
    if (bitmap_is_set(&full_part_field_set,
                      table->found_next_number_field->field_index()))
      return false;                                  // ⑤ AUTO_INCREMENT 在分区表达式里
  }

  if (duplic == DUP_UPDATE) {
    if (update.function_defaults_apply_on_columns(&full_part_field_set))
      return false;
    if (is_fields_in_part_expr(update_fields)) return false;   // ⑥ ON DUPLICATE 改分区列
    /* ⑦ BEFORE UPDATE 触发器同理 */
  }
```

> ⚠️ **真名勘误**：是 **`is_fields_in_part_expr()`**（复数 fields），不是 `is_field_in_part_expr()`；另有 `is_full_part_expr_in_fields()`。

**★ `read_set` 与分区字段的关系**（`Partition_helper::set_partition_read_set()`）：

```cpp
  if (m_handler->get_lock_type() == F_WRLCK) {
    if (bitmap_is_overlapping(&m_part_info->full_part_field_set,
                              m_table->write_set)) {
      bitmap_set_all(m_table->read_set);   // ★★ 改了分区键 ⇒ read_set 全开（要整行重写）
    } else {
      bitmap_union(m_table->read_set, &m_part_info->full_part_field_set);
      //  ★ 至少把分区表达式字段并入 read_set，否则算不出 part_id
    }
```

`ha_innopart` 的 `write_row/update_row/delete_row` 是薄壳，转调 `Partition_helper`（`ha_innopart.h:1018/1049/1053`）：

- **INSERT**（`ph_write_row`，`partition_handler.cc:452`）：`get_partition_id` 按记录值算 part_id（:503）→ 检查 `is_partition_locked` 否则 `HA_ERR_NOT_IN_LOCK_PARTITIONS`（:513-516）→ 记 `m_last_part` → `write_row_in_part(part_id)`
- **UPDATE**（`ph_update_row`，`:557`）：`get_parts_for_update`（`sql_partition.cc:315`：把 old/new 缓冲交替挂到 `full_part_field_array` 分别算 part_id）→ 校验 new 在 lock_partitions → **校验 `old_part_id == m_last_part`**（:594，不符报 `HA_ERR_ROW_IN_WRONG_PARTITION`，防行放错分区）→ **跨分区移动 = insert+delete**（:600-623）：先 `write_row_in_part(new)` 再 `delete_row_in_part(old)`；期间临时清空 `next_number_field` 防二次生成 auto_inc
- **DELETE**（`ph_delete_row`，`:643`）：`get_part_for_delete` → 校验 lock + `m_last_part` → `delete_row_in_part`
- **引擎侧落点**：`ha_innopart::write_row_in_part`（`ha_innopart.cc:1418`）先 `set_partition(part_id)` 把 `m_prebuilt` 切到该分区的 dict_table_t，再调 `ha_innobase::write_row`——**单表代码无感地在指定分区上运行**
- INSERT 批量：`can_prune_insert`（`sql_insert.cc:1445`）算 `used_partitions`，`bitmap_intersect(&lock_partitions, &used_partitions)`（`:1520`）
- UPDATE 改分区键时 lock 比 read 宽：WHERE 只裁 read_partitions，但**所有可能的目标分区都要锁**（`partition_info.h:296-301`、`partition_pruning.cc:413-417`）

### 8.4 裁剪结果的消费（与前七节的衔接）

裁剪输出 `read_partitions` 位图。消费链是 **位图 → `m_part_spec` → `*_in_part` 引擎调用**：

| 扫描类型 | 消费方式 |
|---------|---------|
| 全表扫（`ph_rnd_init`，`partition_handler.cc:1427`） | `get_first_used_partition` 取第一个置位（:1438）；`MY_BIT_NONE` → 空结果；`ph_rnd_next` 按 `get_next_used_partition` 逐个切换分区 |
| 精确点查（`ph_index_read_idx_map`，`:2011`） | `HA_READ_KEY_EXACT` 时 `get_partition_set`（`sql_partition.cc:3668`）：若 index 的 `all_fields_in_PF` 且整 key 已给，`get_full_part_id_from_key` **直接算出唯一分区**（:3701），再 `prune_partition_set`（:3615）与 read_partitions 求交收缩成 `m_part_spec`；逐分区 `index_read_idx_map_in_part` |
| 范围扫描（`partition_scan_set_up`，`:2219`） | 有 key 起点 → `get_partition_set`，否则 0..m_tot_parts-1；单分区关闭归并，多分区有序时用优先队列 `m_ordered_rec_buffer` 跨分区归并（`:2261`） |
| 空集短路 | `all_partitions_pruned_away`（`partition_pruning.cc:281`）→ UPDATE 快路径直接 `no_rows`（`sql_update.cc:477-489`） |

**两次裁剪的时机差异**：prepare 期（`sql_resolver.cc:802`）裁常量 → 只缩 lock_partitions；optimize 期 `JOIN::prune_table_partitions`（`sql_optimizer.cc:2801-2830`）裁含子查询/存储函数的条件 → 缩 read_partitions；单表 UPDATE/DELETE 锁表后二次 `prune_partitions`（`sql_update.cc:474-476`）。

### 8.5 server 层与 ha_innopart 的接口边界

- **`handler::get_partition_handler`**（`handler.h:6934`，默认 nullptr）：server 唯一入口。`fix_partition_func` 末尾 `table->file->get_partition_handler()->set_part_info(...)` 把 `partition_info` 交给引擎；`ha_innopart` 把 this cast 为 `Partition_handler` 返回（`ha_innopart.h:563-565`）
- **`Partition_handler`**（`partition_handler.h:194-354`）：引擎实现的纯虚接口——`set_part_info`、`get_dynamic_partition_info`（分区统计）、`truncate_partition/exchange_partition`（DDL）、`alter_flags`（`HA_INPLACE_CHANGE_PARTITION`）、`get_handler`（回取 handler）
- **`Partition_helper`**（`:390`）：server 写给引擎的通用实现，引擎只需实现 `*_in_part` 系列（`ha_innopart.h:764-917`）
- **`ha_innopart::set_partition(part_id)`**（`ha_innopart.h:715`）：把活跃分区指针切到 `m_parts[part_id]`，使 `ha_innobase` 单表代码无感运行
- **反向回调**：引擎侧 `ph_*` 通过 `m_part_info->get_partition_id` 函数指针回调 server 侧分区计算逻辑
- **auto_inc 跨分区唯一**：`Partition_share`（`partition_handler.h:104`）持 `next_auto_inc_val` + mutex，`open_partitioning` 时挂接

## 九、失效场景

| 场景 | 原因 |
|------|------|
| 分区键字段是 GEOMETRY/ENUM | `fields_ok_for_partition_index`（:1065）拒绝，该字段不进伪索引 |
| RANGE/LIST 分区键上有非单调函数 | `get_monotonicity_info()==NON_MONOTONIC`，`get_part_iter_for_interval=nullptr` |
| HASH/KEY 大范围条件 | 枚举量超 `MAX_RANGE_TO_WALK` → 返回 -1 → 全不裁 |
| 参数化/未求值子查询 | prepare 期 `const_only=true` 不裁，optimize 期锁表后再试 |
| 无 WHERE | `mark_all_partitions_as_used` |
| NDB 自动分区表 | `HA_USE_AUTO_PARTITION` 直接跳过 |
| INSERT 的触发器/生成列改分区列 | `can_prune_insert` 返回 `PRUNE_NO` |

---


## 参考

> 注：分区裁剪**没有专门的内核月报**（阿里内核月报只有《分区表基本类型》(2017/11)、《一致性哈希算法应用》(2022/06) 等"分区表最佳实践"，不涉及裁剪算法本身）。本文完全基于源码 `partition_pruning.cc` 深挖。

**官方文档**
- *MySQL 8.0 Reference Manual → Partitioning → Partition Pruning*

**相关文档**
- SEL_TREE/SEL_ARG 区间森林见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
- 调用点（apply_local_transforms）见 [`../06_resolver_prepare.md`](../06_resolver_prepare.md)
- DML 的分区裁剪短路见 [`../10_dml.md`](../10_dml.md)
