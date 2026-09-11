# 12 分区裁剪（Partition Pruning）

> 分区裁剪决定"一个 WHERE 条件只需要扫哪些分区"，把全表扫描缩小到几个分区。核心洞察：**它被归约为 range 分析问题**——复用 `get_mm_tree` 把 WHERE 转成 SEL_TREE，再遍历 SEL_ARG 区间树映射成分区号。
>
> **边界**：本篇讲**裁剪算法本身**（优化器视角）。分区表的完整链路（元数据模型、DD 持久化、DML 路由、分区级 DDL、InnoDB `ha_innopart` 实现）见 [`../../../feat/partitioning.md`](../../../feat/partitioning.md)。

## 目录

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

**从记录算分区 id**（`set_up_partition_func_pointers`，`sql_partition.cc:1226` 装配函数指针）：
- RANGE：在有序边界数组二分（`get_partition_id_range`）
- LIST：在有序 `list_array` 二分（`get_partition_id_list`）
- HASH：`hash_value % num_parts`（`get_part_id_hash`）
- KEY：`handler::calculate_key_hash_value(fields) % num_parts`（`get_part_id_key`）

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
