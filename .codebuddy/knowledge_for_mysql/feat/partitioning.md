<!-- 单篇体例见 ../_template.md。
     本篇是「跨层端到端特性」：同时改 SQL 层（语法 / 元数据 / 优化器 / MDL）与引擎层（存储 / 分区 DDL）。
     按特性归位在 feat/，不按层拆成 server/ + innodb/ 两篇——理由见「Misc → 为什么本篇在 feat/」。
     裁剪算法的优化器视角另见 ../server/query/07_optimize/12_partition_pruning.md。 -->

# 分区表深度解析

> 基于 MySQL 8.0.39 源码，完整覆盖分区表的**两层实现**：SQL 层的 `partition_info` 元数据模型、DD 持久化与"DD→文本→重解析"往返、分区裁剪、`Partition_helper` 的 DML 路由、分区级 DDL 与 MDL；以及 InnoDB 侧 `ha_innopart` 的类结构、分区上下文切换、per-partition `dict_table_t` 模型、TRUNCATE / EXCHANGE / ADD-DROP 的物理实现、分区统计与能力边界。
>
> **边界**：本篇讲"一张分区表从生到死"的整条链路，SQL 层与引擎层在同一篇里闭环叙述。裁剪器作为 range 优化器的"客户"，其 `SEL_ARG` 遍历的优化器视角见 [`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md)；handler / handlerton 分界面见 [`../server/handler.md`](../server/handler.md)。

## 目录

- [概述](#概述)
  - [先从一个真实的建表语句开始](#先从一个真实的建表语句开始)
  - [分区子句的 EBNF](#分区子句的-ebnf)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

分区表是**一个逻辑表 + N 个物理分区**：

- **SQL 层**保留一份完整的表定义（`TABLE_SHARE` / `partition_info`），持有"规则"：分区类型、分区表达式、分区定义链表、读/锁位图，以及"这一行属于哪个分区"的计算函数。
- **引擎层**（8.0 只有 InnoDB）把每个分区实现为**一个独立的 `dict_table_t`**——独立表空间、独立的一整套 `dict_index_t`，表名编码为 `db/tbl#P#p0`。

两层通过 `handler` 接口缝合：一个 `ha_innopart` 实例管全部 N 个分区。

### 先从一个真实的建表语句开始

下面这条语句把"分区表该怎么写"的绝大多数约束都踩到了——多列 RANGE COLUMNS、HASH 子分区、分区级表空间、MAXVALUE、以及那条最容易踩的"唯一键必须含全部分区列"：

```sql
-- 前置：分区级表空间需先存在（8.0 per-partition tablespace）
CREATE TABLESPACE ts_ord_cn ADD DATAFILE 'ord_cn.ibd' ENGINE=InnoDB;

CREATE TABLE orders (
  id         BIGINT       NOT NULL AUTO_INCREMENT,
  region     VARCHAR(16)  NOT NULL,
  created_at DATETIME     NOT NULL,
  amount     DECIMAL(10,2) NOT NULL,
  -- ★ 唯一键必须包含"全部分区列 + 全部子分区列"，否则
  --   ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF
  PRIMARY KEY (id, region, created_at),
  KEY idx_created (created_at)
) ENGINE=InnoDB
PARTITION BY RANGE COLUMNS (region, created_at)          -- ① 多列 RANGE
SUBPARTITION BY HASH (id) SUBPARTITIONS 4 (              -- ② 子分区 4 个
  PARTITION p_cn_2024 VALUES LESS THAN ('CN', '2025-01-01') (   -- ③ 显式命名子分区
    SUBPARTITION p_cn_2024_s0 TABLESPACE ts_ord_cn,      -- ④ 分区级选项
    SUBPARTITION p_cn_2024_s1 ENGINE=InnoDB COMMENT 'CN 2024',
    SUBPARTITION p_cn_2024_s2,
    SUBPARTITION p_cn_2024_s3
  ),
  PARTITION p_cn_2025 VALUES LESS THAN ('CN', '2026-01-01'),  -- 子分区走默认名 p_cn_2025sp0..3
  PARTITION p_us      VALUES LESS THAN ('US', MAXVALUE),      -- ⑤ 多列中的 MAXVALUE
  PARTITION pmax      VALUES LESS THAN (MAXVALUE, MAXVALUE)
);
```

六处要点：

| # | 写法 | 约束来源 |
|---|---|---|
| ① | `RANGE COLUMNS(region, created_at)` | 分区值按**元组**比较，不是标量；分区列可以是非整型 |
| ② | `SUBPARTITION BY HASH(id) SUBPARTITIONS 4` | 只有 RANGE/LIST 能被子分区，子分区只能是 HASH/KEY |
| ③ | 显式命名子分区 | 未命名的分区由 `create_default_subpartition_names` 生成 `pXspY`；**允许部分分区显式、部分默认** |
| ④ | `TABLESPACE` / `ENGINE` / `COMMENT` | 8.0 支持分区级（含子分区级）选项，`dd::Partition::tablespace_id` 落 `mysql.table_partitions` |
| ⑤ | `('US', MAXVALUE)` | COLUMNS 分区里 MAXVALUE 可以出现在任一列，语义是"该列取上界" |
| — | `PRIMARY KEY (id, region, created_at)` | 分区列 + 子分区列必须出现在每个唯一键里；`AUTO_INCREMENT` 还必须是**索引首列**（断言 `next_number_keypart == 0`） |

它在内存里是一棵两层链表，全局分区号按 `part * num_subparts + subpart` 编码，共 4 × 4 = 16 个 leaf 分区：

```
partition_info{ part_type=RANGE, subpart_type=HASH, num_parts=4, num_subparts=4,
                num_columns=2, part_field_list=(region, created_at), subpart_expr=hash(id) }
  └─ partitions: [p_cn_2024, p_cn_2025, p_us, pmax]        ← List<partition_element>
        └─ p_cn_2024.subpartitions: [s0, s1, s2, s3]       ← 同类型第二层嵌套
        └─ p_cn_2025.subpartitions: （空 → 由默认值补齐）
```

#### 分区子句的 EBNF

从 `sql_yacc.yy` 的分区语法段提炼（`partition_clause` 起、`part_option` 止）：

```ebnf
partition_clause ::=
    "PARTITION" "BY" part_type_def [num_parts] [sub_part] [part_defs]

part_type_def ::=
    ["LINEAR"] "KEY" ["ALGORITHM" "=" ("1"|"2")] "(" [name_list] ")"
  | ["LINEAR"] "HASH" "(" expr ")"
  | "RANGE" "(" expr ")"
  | "RANGE" "COLUMNS" "(" name_list ")"
  | "LIST"  "(" expr ")"
  | "LIST"  "COLUMNS" "(" name_list ")"

num_parts  ::= "PARTITIONS" ulong_num                      (* PARTITIONS 0 → ER_NO_PARTS_ERROR *)
sub_part   ::= "SUBPARTITION" "BY" ["LINEAR"] ("HASH" "(" expr ")" | "KEY" ["ALGORITHM" "=" ("1"|"2")] "(" name_list ")")
               ["SUBPARTITIONS" ulong_num]
part_defs  ::= "(" part_definition { "," part_definition } ")"

part_definition ::=
    "PARTITION" ident [part_values] [part_options] [sub_partition]

part_values ::=                                            (* 空 = HASH/KEY 分区 *)
    "VALUES" "LESS" "THAN" ( "MAXVALUE" | "(" value_item_list ")" )   (* RANGE *)
  | "VALUES" "IN" ( "(" value_item_list ")"                            (* LIST 单值 *)
                  | "(" value_tuple { "," value_tuple } ")" )          (* LIST 多值 *)
value_item  ::= "MAXVALUE" | expr
value_tuple ::= "(" value_item_list ")"

sub_partition      ::= "(" sub_part_definition { "," sub_part_definition } ")"
sub_part_definition::= "SUBPARTITION" ident [part_options]

part_options ::= part_option { part_option }
part_option  ::= "TABLESPACE" ["="] ident
               | ["STORAGE"] "ENGINE" ["="] ident
               | "NODEGROUP" ["="] ulong_num            (* NDB 遗留 *)
               | "MAX_ROWS" ["="] ulonglong_num
               | "MIN_ROWS" ["="] ulonglong_num
               | "DATA"  "DIRECTORY" ["="] string
               | "INDEX" "DIRECTORY" ["="] string
               | "COMMENT" ["="] string
```

三点值得注意：

- `VALUES LESS THAN MAXVALUE` 不带括号，而 `VALUES LESS THAN (expr)` / COLUMNS 的多列形式必须带括号——这是语法层面的差异，不是风格问题。
- `part_values` 为空即 HASH/KEY 分区（分区值由哈希算出，无需声明），解析时 `part_type` 默认填 `HASH`。
- `KEY` 分区在实现上被归为 `part_type=HASH` + `list_of_part_fields=true`（`parse_tree_partitions.cc`），即"用字段列表做 HASH"。

### 用途

分区表存在的三个真实价值（按重要性）：

1. **分区裁剪**：`WHERE` 只命中少数分区时，扫描集从 N 降到 1~2，这是分区唯一"免费"的查询收益。
2. **分区级运维**：`DROP PARTITION` / `TRUNCATE PARTITION` / `EXCHANGE PARTITION` 是 O(1)~O(分区大小) 的操作，用来替代 `DELETE FROM t WHERE date < ...`（后者产生大量 undo + purge 压力 + 不释放空间）。这是"按月清理历史数据"的标准做法。
3. **管理大表**：单个分区可单独 `ANALYZE` / `CHECK` / `OPTIMIZE` / `REPAIR`，失败范围被限制在一个分区内。

### 两个反直觉的切入点

**其一：为什么我的分区裁剪没生效**

```sql
CREATE TABLE t (
  id INT NOT NULL,
  created DATETIME NOT NULL,
  PRIMARY KEY (id, created)
) PARTITION BY RANGE (TO_DAYS(created)) (
  PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
  PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
  PARTITION pmax   VALUES LESS THAN MAXVALUE
);

EXPLAIN SELECT * FROM t WHERE created >= '2024-02-01' AND created < '2024-03-01'\G
-- partitions: p202402               ← 生效：TO_DAYS 是单调函数，可反解

EXPLAIN SELECT * FROM t WHERE DATE(created) = '2024-02-15'\G
-- partitions: p202401,p202402,pmax   ← 失效：DATE() 不是已知的单调函数

EXPLAIN SELECT * FROM t WHERE id = 10\G
-- partitions: p202401,p202402,pmax   ← 失效：条件不在分区列上
```

第三行是最常被误解的场景：**分区表没有全局索引**。每个分区各自一套完整的 B+ 树（含主键），`id = 10` 在每个分区都可能是"存在的"，因此必须扫全部分区。这直接带来那条著名限制——**唯一键（含主键）必须包含全部分区列**（`sql_partition.cc` 的 `check_primary_key` / `check_unique_keys`，报错 `ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`）。

**其二：为什么 8192 个分区会让一切变慢**

```sql
-- 行数相同的两张表
SELECT COUNT(*) FROM t_no_part;   -- 快
SELECT COUNT(*) FROM t_8192_part; -- 慢得多，information_schema 查询也变慢
```

根因不在扫描本身，而在**对象模型的膨胀**：每个分区是一个完整的 `dict_table_t`，有自己的 space 对象、自己的 `dict_index_t` 全套、自己的统计项、自己在 dict cache 里的条目。打开这张表要在 dict cache 里装载 8192 个表对象并逐个 `dict_stats_init`。这也是分区数上限定在 8192、且"分区数上千后 DDL/打开表性能急剧下降"的原因。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.1 | 引入分区：RANGE / LIST / HASH / KEY + 子分区 |
| 5.5 | RANGE COLUMNS / LIST COLUMNS，分区列不再局限于整数表达式 |
| 5.6 | EXCHANGE PARTITION；显式分区选择 `SELECT ... FROM t PARTITION (p0)`；分区数上限提高到 8192 |
| 5.7 | **native partitioning**：InnoDB 自带 `ha_innopart`，取代通用分区层 `ha_partition` |
| 8.0 | 彻底移除 non-native partitioning（`storage/partition/` 删除，`handler.h` 仅留一句"本注释拷贝自 `ha_partition.h`"的墓志铭）；分区 DDL 纳入原子 DDL；引入 per-partition tablespace |
| 8.0.29+ | instant ADD/DROP COLUMN 支持分区表（要求所有分区都支持） |

---

## 理论基础

### 设计思想与权衡

**为什么分区规则放在 SQL 层，而物理存储下推引擎？**

分区要做三件事：解析分区表达式、在优化期推导"哪些分区可能被访问"、在运行期算"这一行属于哪个分区"。前两件强依赖 SQL 层的 `Item` 表达式体系与优化器，第三件依赖引擎写入。于是形成天然分工：**SQL 层持规则，引擎层持物理**。

代价是两层之间必须反复同步元信息。8.0 的做法是把分区定义**序列化成 SQL 文本**存进 DD，打开表时**重新解析**这段文本重建 `partition_info`。这是明显的性能与复杂度妥协——换来的是 DD 表结构不必知道 `partition_info` 的内部布局（`sql/dd/dd_table.cc` 里至今留着 `TODO-PARTITION: move into partitioning service, WL#4827`）。

**为什么每个分区是一个完整的 `dict_table_t`？**

5.x 的 `ha_partition` 是"server 层一个表对象 + 引擎层一个表对象，分区只是 server 层的路由"。Native 化之后反过来：**分区身份下沉到引擎的物理对象**。

收益：

- 复用 InnoDB 关于"表"的一切既有机制——表空间管理、B+ 树、事务与行锁、change buffer、统计、崩溃恢复，**零分区特判**；
- 分区级运维天然可做：`DROP PARTITION` = 删一个表空间，`TRUNCATE PARTITION` = 重建一个表空间，`EXCHANGE` = 交换两个表空间的文件；
- 分区级粒度自然获得：行锁、表锁、buffer pool 冷热、统计都按分区独立。

代价（权衡的另一半）：

| 代价 | 具体后果 |
|---|---|
| 元数据线性膨胀 | N 个 `dict_table_t` + N×索引数 个 `dict_index_t`，dict cache 压力与分区数成正比；`open` 要装载全部分区 |
| **无全局索引** | 每个分区一套本地索引 → 唯一键必须包含分区列；非分区列的点查要扫全部分区 |
| 统计碎片化 | 每分区独立统计；向优化器汇报时只能"行数求和、区分度取最大分区" |
| 打开表无按需加载 | `open` 无条件打开全部 leaf 分区，裁剪只影响后续遍历哪些分区 |

**为什么"一个 handler 管全部分区"，而不是每分区一个 handler？**

5.x 的通用分区层 `ha_partition` 是"每分区一个 handler 实例"的模型。8.0 的 `ha_innopart` 改成**一个 handler + 一份 `row_prebuilt_t` + 一个 per-partition 状态数组**，靠 `set_partition()` / `update_partition()` 在分区间切换上下文。

收益：

- server 层 `TABLE` 与 `handler` 严格一对一，上层 SQL 不必感知"表内有多个游标"；
- 事务、2PC、表锁、auto_increment 计数器都只有一份（`N` 个分区共享 1 个 `trx_t`、1 个 `Ha_trx_info`），注册粒度是 `(THD, handlerton)`；
- 内存与 fd 开销不随分区数线性膨胀。

代价落在引擎内部：每次跨分区都要把 `ins_node / upd_node / trx_id / row_read_type / new_rec_lock` 等上下文在 `m_prebuilt` 与 `m_parts[part_id]` 之间搬运一次，且必须为每个分区预分配持久化游标。

**为什么裁剪复用 range 优化器，而不是自己写区间代数？**

`partition_pruning.cc` 的头注释把这个问题讲得很直白（译其要点）：

> 对于 `WHERE` 中的 `t1.a`，我们需要 ① 求出 `t1.a` 上的区间列表，② 对每个区间求与它有交集的分区。第 ① 步正是 range analysis（`get_mm_tree`）能做的事——**只要给它一份索引描述（KEY_PART 数组）**。而"这类索引是否真的存在并不重要，range analysis 只用描述"。

于是裁剪器构造一个**假索引**：把分区列（+ 子分区列）拼成一个 KEY，令 `using_real_indexes = false`，直接调 `get_mm_tree`。AND / OR / IN / `<` / `<=>` 的归约、区间求交求并、min/max key 打包，全部免费复用。

这个复用的代价（即裁剪失效的全部来源）：

| 简化假设 | 失效场景 / 退化阈值 |
|---|---|
| 谓词能被 `get_mm_tree` 消化 | 非 `SEL_TREE::KEY` 结果、字段是 ENUM/GEOMETRY（`fields_ok_for_partition_index` 直接返回 false）→ `goto all_used` 全分区 |
| 分区表达式**单调** | 非单调（`Item::get_monotonicity_info()` 默认返回 `NON_MONOTONIC`）→ 只能裁"等值点"，范围条件全分区 |
| 区间数量可控 | `range_optimizer_max_mem_size` 超限、递归过深 → 放弃裁剪 |
| 只有第一个分区列带条件才裁剪 | 注释明写"更一般的情况留给 WL#4065"，条件落在第 2 个分区列 → `res = -1` 全分区 |
| HASH/KEY 不可反解 | 只能逐值枚举，且 `n_values > 32` 且 `> 2*分区数` 时直接放弃 |

**为什么需要 `read_partitions` 和 `lock_partitions` 两份位图？**

这是最容易被忽略、也最能说明"分区 UPDATE 不是普通 UPDATE"的设计点。

- `read_partitions`：**要扫哪些分区**，可以被 WHERE 裁掉。
- `lock_partitions`：**要锁哪些分区**，只能在"分区键不会被修改"时才跟着收缩。

原因：`UPDATE t SET part_key = ... WHERE cond` 中，新行可能落到一个**未被读到的分区**。若锁集跟着读集收缩，写到未加锁的分区就产生一致性问题。源码处理：

```cpp
// range_optimizer/partition_pruning.cc
if (!thd->lex->is_query_tables_locked() &&
    !partition_key_modified(table, table->write_set)) {
  bitmap_copy(&prune_param.part_info->lock_partitions,
              &prune_param.part_info->read_partitions);
}
```

只有"分区键没被写"时才反向收缩锁集；写入时若目标分区不在锁集，`Partition_helper` 直接返回 `HA_ERR_NOT_IN_LOCK_PARTITIONS`。

**代价**：任何修改分区键的 UPDATE，锁集 = 全部分区（退化为全表锁粒度）。

**为什么没有动态（per-row）裁剪？**

8.0 只在两个时点裁剪：prepare 期（未加锁，确定锁集）与 optimize 期（已加锁，确定读集）。**执行期没有 per-row 重算**。prepare 期子查询/存储程序无法求值，所以需要第二次裁剪来补（`is_pruning_completed` 标记决定是否还有下一次）。唯一"逐行"的路径是 INSERT——每一行都可能落在不同分区，`can_prune_insert` / `set_used_partition` 在 `sql_insert.cc` 里按行算 `used_partitions`。

**为什么 TRUNCATE PARTITION 是"rename → drop → create"？**

逐行删（`DELETE FROM`）有三个不可接受的代价：写大量 undo、产生大量 purge 工作、不释放 .ibd 空间。而 `TRUNCATE PARTITION` 的实现是：

```
rename_tablespace()   旧 .ibd 物理重命名为临时名
  → delete_impl()     删除旧分区
  → create_impl()     新建空分区
```

三步之间的崩溃安全由 **DDL log** 保证（`row0mysql.cc` 里 `write_drop_log` / `write_delete_space_log`，`dict0dict.cc` 的 `write_rename_table_log`，`fil0fil.cc` 的 `write_rename_space_log`；崩溃恢复走 `Log_DDL::recover` / `replay_all`）。源码甚至留了故障注入点（`ib_truncate_crash_after_rename` 之类）测这三个中间态。

**权衡**：换来 O(1) 的空间回收与无 undo 开销，代价是 DDL log 这套"为 DDL 做的 WAL"，以及 truncate 期间必须独占 MDL。

**为什么 EXCHANGE PARTITION 是"三次 rename"？**

要把分区 P 与普通表 S 的数据互换，最小代价是交换两个表空间的文件，而不是搬数据。三个文件、两个目标，只能借助一个临时名做三跳：

```
1. rename S    → temp   （S 让出名字）
2. rename P    → S      （分区数据变成 S 的表空间）
3. rename temp → P      （原 S 数据变成分区的表空间）
```

随后还要交换每个索引的 `se_private_data`（index root page no 等物理位置）与 `tablespace_id`，并调整 DATA DIRECTORY。

**为什么不是两次 rename**：两个目标名互相占用，直接 `rename A→B; rename B→A` 第二步会撞上已存在的 B。这是教科书式的 swap-with-temp。

**为什么 ADD / DROP PARTITION 有时要全表重建？**

- RANGE / LIST 在**末尾** ADD、DROP 一个分区：行的映射关系不变，只需新建 / 删除对应分区的表空间；
- HASH / KEY 改分区数（COALESCE / ADD PARTITIONS）、REORGANIZE PARTITION：**取模基数或值区间变了，几乎所有行都可能属于另一个分区**，必须逐行重算并搬迁（`copy_partitions`）。

因此引擎用 `alter_flags()` 声明能力（`HA_PARTITION_FUNCTION_SUPPORTED | HA_INPLACE_CHANGE_PARTITION`），由 `alter_parts::prepare` 判定本次是否需拷贝，进而决定 `check_if_supported_inplace_alter` 返回 `HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE`（不用拷，允许并发 DML）还是 `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE`（要拷，主阶段需 S 锁）。

### 理论溯源

| 理论 / 经典算法 | 源码落点 |
|---|---|
| **区间代数 + 红黑树森林**（range 优化器 `SEL_ROOT` / `SEL_ARG`） | 裁剪器直接复用 `get_mm_tree` 产出的 `SEL_ARG` 图；`find_used_partitions` 对红黑树做纵向递归、沿 `next_key_part` 做横向递归 |
| **单调函数反解**（`F(x) op c` ⇔ `x op' F⁻¹(c)`） | `Item::val_int_endpoint` + `get_monotonicity_info`；RANGE/LIST 的 interval mapper 据此把端点映成分区号 |
| **k 路有序归并**（Knuth《TAOCP》5.4.1 外部排序的归并思想） | 跨多分区的有序索引扫描用优先队列 `handle_ordered_index_scan`，比较器 `Key_rec_less` |
| **模板方法模式**（Template Method） | `Partition_helper` 把"跨分区遍历"写成算法骨架，把"单个分区内的动作"留成 `*_in_part()` 纯虚函数交给 `ha_innopart` 实现 |
| **Mixin / 多重继承** | `ha_innopart : ha_innobase + Partition_helper + Partition_handler`——把"存储能力""通用分区算法""分区 API"三件事正交组合 |
| **迭代器 / 策略模式** | `PARTITION_ITERATOR::get_next` 是函数指针，RANGE / LIST / NULL-only 各有实现 |
| **RAII + 模板特化** | `innobase_truncate<dd::Partition>`：`prepare()` / `exec()` / `cleanup()`；同一套代码通过模板参数同时服务"整表"和"分区"两种粒度 |
| **Save/Restore 上下文**（与协程上下文切换同构） | `m_parts[]` 保存每个分区的"执行现场"，切换时恢复（粒度是字段拷贝） |
| **WAL for DDL**（DDL log） | 分区 TRUNCATE / DROP / rename 的崩溃安全靠 `Log_DDL` 记录物理操作，恢复时 replay |
| **Swap with temporary** | EXCHANGE PARTITION 的三次 rename |

### 算法与数据结构

| 结构 | 为什么选它 | 复杂度 |
|---|---|---|
| `SEL_ARG` 红黑树森林 | 复用 range 优化器的区间代数，支持 AND/OR/IN 自动归约 | 构造 O(n log n)，遍历 O(区间数 × 列数) |
| `range_int_array` / `list_array`（**排序数组**） | 分区定义数量固定且不大，排序数组 + 二分比 hash 省内存、无碰撞，且天然支持"区间端点"查询 | 构建 O(n log n)，定位 O(log n) |
| 假索引 `KEY_PART[]` | 让 range 优化器以为在优化一个真实索引 | — |
| `MY_BITMAP` 位图 | 分区集合是 0..8191 的稠密小整数集，交/并是 O(n/64) | O(n/64) |
| 优先队列（有序扫描） | 分区数 = 路数，k 路归并是外部排序的标准手法 | O(log k) / 行 |
| `LIST_PART_ENTRY{list_value, partition_id}` | LIST 分区的"值 → 分区"映射；注意一个值区间可能映出**重复**的 partition_id（头文件注释明确说明） | — |
| `Ha_innopart_share`（TABLE_SHARE 级共享） | 一张表的所有 `TABLE` 实例共享分区对象数组与索引映射；`m_ref_count` 归零才真正释放 | 已打开时 O(1) |
| `dict_table_t **m_table_parts` | 每分区一个表对象，名字 `db/tbl#P#p0[#SP#sp0]` | O(N) 内存 |
| `dict_index_t **m_index_mapping` | `(partition, mysql keynr) → dict_index_t`，布局 `part * mysql_num_index + idx` | O(N × 索引数) |
| `saved_prebuilt_t m_parts[]` | 只保存必要字段（不做整份 prebuilt 深拷贝） | O(N)，切换 O(1) |

### 他库对比与演进动机

| 维度 | MySQL 8.0 | PostgreSQL（声明式分区） | Oracle |
|---|---|---|---|
| 分区是不是"真正的表" | 引擎层每个分区是独立 `dict_table_t`，但 SQL 层不可单独引用 | 每个分区就是一张表，可 `ATTACH/DETACH`、可单独查询 | 分区是 segment，不可单独查询 |
| 全局索引 | **无**（所有索引都是 per-partition 本地索引）→ 唯一键必须含分区列，非分区列点查要扫全分区 | 支持全局索引 | 支持全局/本地索引 |
| 运行时裁剪 | 无（prepare + optimize 两期静态裁剪） | 支持 run-time pruning（含参数化执行计划） | 支持，含动态裁剪 |
| partition-wise join / aggregate | 无 | 支持 | 支持 |
| 自动建分区 | 无（手动 ADD 或脚本） | 无原生 interval 分区 | 有 INTERVAL 分区 |
| 分区键与外键 | **互斥**（`HA_CANNOT_PARTITION_FK`） | 支持 | 支持 |
| 分区交换 | `EXCHANGE PARTITION`（三次 rename + 结构完全一致校验，`WITHOUT VALIDATION` 可跳过逐行扫描） | `ATTACH/DETACH PARTITION`（元数据操作，靠约束校验，无需逐行扫描） | `EXCHANGE PARTITION`（元数据 + 可选校验） |

**演进动机的主线**：MySQL 分区从"server 层通用分区层（`ha_partition`，一表一分区 handler）"演进到"引擎 native 分区"，核心动机是消除 `ha_partition` 带来的额外 handler 层（每分区一次虚调用 + 事务/锁/统计的重复管理）。代价是：所有分区 DDL 必须由每个引擎各自实现，所以 8.0 直接用"不支持 native partitioning 就报错"的方式清场（`ER_CHECK_NOT_IMPLEMENTED, "native partitioning"`）。

**如果将来有多个引擎实现同一特性怎么办**：本篇给出的答案是——**不为每个引擎单独开篇**，而是一篇里用"引擎差异"小节对比（例如分区表目前的引擎差异只有 InnoDB 与 NDB 两家：NDB 走 `HA_USE_AUTO_PARTITION` 的自动分区路径，且保留 `nodegroup_id`；未声明 native 分区能力的引擎直接被拒绝）。分裂成 N 篇会同时违反"第一主语原则"和"模块闭环"。

---

## 核心实现

### 主链路

```
【建表】Sql_cmd_create_table::execute
   → mysql_prepare_create_table
       → partition_info::check_partition_info   合法性
       → fix_partition_func                     绑定 get_partition_id 函数指针
   → fill_dd_table_from_create_info → fill_dd_partition_from_create_info   写 DD
   → ha_innopart::create                       逐分区建表空间

【打开表】open_table_def
   → fill_partitioning_from_dd                 DD → partition_info
   → generate_partition_syntax                 分区定义 → SQL 文本（存 share）
   → unpack_partition_info → mysql_unpack_partition   重新解析文本 → TABLE::part_info
   → fix_partition_func
   → ha_innopart::open                         打开全部 N 个分区

【查询】JOIN::optimize
   → JOIN::prune_table_partitions → prune_partitions     裁剪 → read_partitions 位图
   → make_join_plan                                      之后才选访问路径
   → 执行期：TableScanIterator → handler::ha_rnd_next
       → Partition_helper::ph_rnd_next        按 read_partitions 逐分区串行扫
       → ha_innopart::rnd_next_in_part        set_partition → ha_innobase

【写入】write_record → handler::ha_write_row
   → Partition_helper::ph_write_row
       → get_partition_id(...)                运行期算分区号
       → write_row_in_part → set_partition → ha_innobase::write_row
```

注意调用顺序：**裁剪在 `make_join_plan` 之前**（`optimize_cond` → `prune_table_partitions` → `make_join_plan`），这样后续每表的访问路径选择才能基于裁剪后的行数估计。

### 3.1 建表：语法 → `partition_info` → 校验 → DD → 引擎

> 以概述那条 `orders` 表为例，看它如何从文本变成内存对象，再变成 DD 行与 N 个 .ibd。

#### 第一步：SQL 片段 → `partition_info` 字段

| SQL 片段 | 落到哪里 |
|---|---|
| `PARTITION BY RANGE COLUMNS (region, created_at)` | `part_type=RANGE`、`column_list=true`、`num_columns=2`、`part_field_list=(region, created_at)` |
| `SUBPARTITION BY HASH (id) SUBPARTITIONS 4` | `subpart_type=HASH`、`subpart_expr=hash(id)`、`num_subparts=4` |
| 4 个 `PARTITION p_xxx VALUES LESS THAN (...)` | `partitions` 链表 4 个 `partition_element`；元组值进 `list_val_list`，含 `MAXVALUE` 时 `max_value` 置位 |
| `p_cn_2024` 的 4 个 `SUBPARTITION` | `p_cn_2024.subpartitions` 链表（第二层嵌套） |
| `p_cn_2025` 未写子分区 | `subpartitions` 为空 → `set_up_default_subpartitions` 补出 `p_cn_2025sp0..3` |
| `TABLESPACE ts_ord_cn` / `COMMENT '...'` | `partition_element::tablespace_name` / `part_comment` |
| 没写 `PARTITIONS n` | `use_default_num_partitions=true`，`num_parts` 由分区定义个数推出 |

这步之后才轮到 `fix_partition_func`：把元组排序去重、构建 `range_int_array`（RANGE）或 `list_array`（LIST，排序后二分），并绑定运行期的 `get_partition_id` 函数指针。

#### 第二步：语法层归约

`PARTITION BY` 在语法层被归约为 `PT_partition`，它内部**内嵌一个 `partition_info` 成员**，contextualize 时把分区类型、分区定义逐个填进去，最后挂到 `LEX::part_info`：

```cpp
// parse_tree_nodes.cc
bool PT_alter_table_partition_by::contextualize(Table_ddl_parse_context *pc) {
  if (super::contextualize(pc) || m_partition->contextualize(pc)) return true;
  pc->thd->lex->part_info = &m_partition->part_info;
  return false;
}
```

分区类型落盘的关键映射（`parse_tree_partitions.cc`）：

- `KEY(...)` → `part_type = HASH` + `list_of_part_fields = true`（KEY 在实现上是"用字段列表做 HASH"）
- `HASH(expr)` → `part_type = HASH` + `column_list = false`
- `RANGE(expr)` / `RANGE COLUMNS(...)` → `part_type = RANGE`
- `LIST(expr)` / `LIST COLUMNS(...)` → `part_type = LIST`

子分区在 `PT_subpartition::contextualize` 中复制进 `partition_element::subpartitions`。

#### 内存结构

**`partition_info`** 是表级描述，**`partition_element`** 是单个分区描述；二者是父子链表关系，且 `partition_element::subpartitions` 是**同类型的第二层嵌套**：

```cpp
// sql/partition_info.h（节选）
class partition_info {
  List<partition_element> partitions;       // 顶层分区链表
  List<partition_element> temp_partitions;  // ALTER 期间的临时分区
  List<char> part_field_list;               // KEY / COLUMNS 的字段名列表
  Item *part_expr;                          // HASH/RANGE(expr)/LIST(expr) 的分区表达式
  Item *subpart_expr;
  MY_BITMAP read_partitions;                // 要读的分区集合
  MY_BITMAP lock_partitions;                // 要锁的分区集合
  uint num_parts, num_subparts, num_columns;
  partition_type part_type, subpart_type;
  bool use_default_partitions, is_auto_partitioned;
  get_part_id_func get_partition_id;        // ★ 运行期算分区号的函数指针
  ...
};

// sql/partition_element.h（节选）
class partition_element {
  List<partition_element> subpartitions;    // 子分区（同类型嵌套）
  List<part_elem_value> list_val_list;      // LIST 的值 / RANGE 单值 / COLUMNS 数组
  const char *partition_name;
  const char *tablespace_name;
  longlong range_value;                     // RANGE 的 LESS THAN 值
  enum partition_state part_state;          // PART_NORMAL / PART_TO_BE_ADDED / ...
  bool max_value, has_null_value, signed_flag;
};
```

两个易踩的点：

1. **8.0 的 `partition_element` 没有 `id` 字段**——`id` 只存在于 DD 侧 `dd::Partition::id()`。分区顺序由 `dd::Partition::number()` 决定。
2. **`partition_info` 有三份**：`LEX::part_info`（解析期）、`TABLE_SHARE::m_part_info`（share 级，DD 填充）、`TABLE::part_info`（每个 `TABLE` 一份，**重新解析** share 里的分区语法文本得到）。

#### 校验清单

| 检查项 | 函数 | 报错 |
|---|---|---|
| 分区表达式不允许的函数/子查询 | `check_partition_info`（`Item::check_partition_func_processor`） | `ER_PARTITION_FUNCTION_IS_NOT_ALLOWED` |
| 只有 RANGE/LIST 可子分区 | `check_partition_info` | `ER_SUBPARTITION_ERROR` |
| 总分区数上限（`MAX_PARTITIONS`=8192） | `check_partition_info` | `ER_TOO_MANY_PARTITIONS_ERROR` |
| 分区列重复 / 分区名重复 | `check_partition_info` + `find_duplicate_name` | `ER_SAME_NAME_PARTITION_FIELD` / `ER_SAME_NAME_PARTITION` |
| 分区名合法性/长度 | `check_partition_info` | `ER_WRONG_PARTITION_NAME` |
| 引擎混用 | `check_partition_info` | `ER_MIX_HANDLER_ERROR` |
| KEY ALGORITHM、COLUMNS 列数、NULL in VALUES LESS THAN | `fix_parser_data` | `ER_PARTITION_COLUMN_LIST_ERROR` / `ER_NULL_IN_VALUES_LESS_THAN` |
| 分区表达式必须返回 INT；RANGE/LIST 常量递增/不重叠 | `fix_partition_func`（`check_range_constants` / `check_list_constants`） | `report_part_expr_error` |
| **主键/唯一键必须含全部分区列** | `check_primary_key` / `check_unique_keys` | `ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF` |
| 分区表达式过长 | `fill_dd_partition_from_create_info` | `ER_PART_EXPR_TOO_LONG` |
| 分区字段数量上限（`MAX_REF_PARTS`） | `parse_tree_partitions.cc` | `ER_TOO_MANY_PARTITION_FUNC_FIELDS_ERROR` |

关于"分区名字典序"：8.0 **没有**字典序强制要求。实际约束是默认名按 `p0..pN` 生成、`find_duplicate_name` 用 `collation_unordered_set` 保证分区名+子分区名全局唯一、DD 层以 `UNIQUE KEY(table_id, name)` 兜底，打开时的顺序由 `dd::Partition::number()` 决定（`fill_partitioning_from_dd` 有 assert 校验有序）。

#### 引擎侧 `ha_innopart::create`

```cpp
// ha_innopart::create（节选）
if (thd_sql_command(thd) == SQLCOM_TRUNCATE)
  return (truncate_impl(name, form, table_def));

if (is_shared_tablespace(create_info->tablespace)) {
  my_printf_error(ER_ILLEGAL_HA_CREATE_OPTION, PARTITION_IN_SHARED_TABLESPACE, MYF(0));
  return HA_ERR_INTERNAL_ERROR;
}
/* Not allowed to create temporary partitioned tables. */
...
for (const auto dd_part : *table_def->leaf_partitions()) {
  dict_name::build_partition(dd_part, partition);
  dict_name::build_table("", saved_table_name, partition, false, false, part_table);
  info.prepare_create_table(part_table.c_str());
  info.create_table(&dd_part->table(), nullptr);
  info.create_table_update_global_dd<dd::Partition>(const_cast<dd::Partition *>(dd_part));
  info.create_table_update_dict();
  info.detach();
  ++created;
}
```

- **一个 `create_table_info_t` 复用 N 次**，每次把表名换成分区名（`prepare_create_table` → `create_table` → `detach` 循环）；
- 分区**只能放 file-per-table 表空间**，共享表空间直接报错；
- 不支持临时分区表（`ER_PARTITION_NO_TEMPORARY`）；
- `TRUNCATE TABLE`（非分区级）复用同一入口，走 `truncate_impl` 逐分区处理。

### 3.2 元数据持久化：DD → 文本 → 重新解析

分区定义分散在四张 DD 表里：

| 表 | 关键列 |
|---|---|
| `mysql.tables` | `partition_type` / `partition_expression` / `partition_expression_utf8` / `subpartition_type` / `default_partitioning` |
| `mysql.table_partitions` | `table_id` / `parent_partition_id` / `number` / `name` / `engine` / `description_utf8` / `se_private_id` / `tablespace_id` |
| `mysql.index_partitions` | `partition_id` / `index_id` / `se_private_data`（每分区每索引一条，保存 index root page 等） |
| `mysql.table_partition_values` | `partition_id` / `list_num` / `column_num` / `value_utf8` / `max_value` |

写入：`fill_dd_partition_from_create_info`（`sql/dd/dd_table.cc`，注意是**单数** `partition`）。它把 `partition_info` 翻译成 `dd::Partition` + `dd::Partition_index` + `dd::Partition_value`：RANGE 的 `MAXVALUE` 落在 `set_max_value`，LIST 逐个 `add_value`，COLUMNS 走 `add_part_col_vals`，最后为**每个分区每个索引**建一条 `dd::Partition_index`。

读取（8.0 最有意思的一段设计）：

```
open_table_def
  → fill_partitioning_from_dd        // dd::Partition → partition_element 链表
  → generate_partition_syntax        // partition_element 链表 → "PARTITION BY ..." 文本
  → share->partition_info_str = ...
  → unpack_partition_info            // 打开 TABLE 时
       → mysql_unpack_partition      // 用 GRAMMAR_SELECTOR_PART 复用 yacc 的 partition_clause 语法
       → fix_partition_func          // 重新绑定 get_partition_id、重建 range_int_array/list_array
```

也就是说：**打开一张分区表，MySQL 会重新解析一遍 `CREATE TABLE` 的分区子句**。这个往返让 DD 表不需要知道 `partition_info` 的内部结构，代价是每次 share 重建要跑一遍 yacc（结果缓存在 `TABLE_SHARE`）。

### 3.3 打开表：对象模型与"无条件全开"

```
ha_innopart::open
  →（share 已存在？只 m_ref_count++，对每个 m_table_parts[i]->acquire()）
  → Ha_innopart_share::open_table_parts
       → for (dd_part : dd_table->leaf_partitions())
             dict_name::build_partition(dd_part, partition)     "db/tbl#P#pN"
             open_one_table_part(...)                            dict cache miss → dd_open_table<dd::Partition>
  → set_table_parts_and_indexes()        构造 m_index_mapping，逐索引校验列一致性
  → open_partitioning(m_part_share)      SQL 层 Partition_helper 初始化
  → 对所有分区 dict_stats_init()
  → 建 m_prebuilt（仅一份，基于分区 0）、分配 m_parts[] / m_pcur_parts[]
```

完整对象模型：

```
server 层
  TABLE + 1 个 ha_innopart                     ← 一个 handler 管全部分区
     ├── Ha_innopart_share（挂在 TABLE_SHARE 上，跨 TABLE 共享）
     │      ├── dict_table_t *m_table_parts[N]        db/tbl#P#p0 ... #P#pN
     │      ├── dict_index_t *m_index_mapping[N × idx_count]
     │      └── m_ref_count                           归零才释放 dict_table_t
     ├── row_prebuilt_t *m_prebuilt（只有一份，基于分区 0 建）
     ├── saved_prebuilt_t m_parts[N]           ← 每分区上下文快照
     └── m_pcur_parts[N] / m_clust_pcur_parts[N]   ← 每分区持久化游标

DD 层
  1 个 dd::Table + N 个 dd::Partition（leaf_partitions()）+ N × idx 个 dd::Partition_index
```

`dict_table_t` **本身没有分区字段**——分区身份完全由表名编码决定，判定函数是 `dict_table_is_partition()`（检查名字里有没有 `#P#`）。DD↔InnoDB 一致性靠断言保证：`ut_ad(dict_table_is_partition(table) == dd_table_is_partitioned(*dd_table))`。

**没有"只打开被裁剪到的分区"**：循环遍历的是 `leaf_partitions()` 全集，`lock_partitions` 位图根本不传到引擎（它只在 SQL 层做写入合法性检查）。按需打开在社区版不存在——全库搜 `m_open_and_lock_partition` 是 0 命中。

### 3.4 分区上下文切换：`set_partition` / `update_partition`

这是 `ha_innopart` 最核心、也最能体现"一个 handler 管全部分区"的地方。

```cpp
void ha_innopart::set_partition(uint part_id) {
  if (part_id >= m_tot_parts) { ut_d(ut_error); ut_o(return ); }

  if (m_pcur_parts != nullptr) {
    m_prebuilt->pcur = &m_pcur_parts[m_pcur_map[part_id]];
  }
  if (m_clust_pcur_parts != nullptr) {
    m_prebuilt->clust_pcur = &m_clust_pcur_parts[m_pcur_map[part_id]];
  }

  const auto &part{m_parts[part_id]};
  m_prebuilt->ins_node = part.m_ins_node;
  m_prebuilt->upd_node = part.m_upd_node;
  /* For unordered scan and table scan, use blob_heap from first
  partition as we need exactly one blob. */
  m_prebuilt->blob_heap = m_parts[m_ordered ? part_id : 0].m_blob_heap;
  m_prebuilt->trx_id        = part.m_trx_id;
  m_prebuilt->row_read_type = part.m_row_read_type;
  m_prebuilt->new_rec_lock  = part.m_new_rec_lock;

  m_prebuilt->sql_stat_start = m_sql_stat_start_parts.test(part_id);
  m_prebuilt->table = m_part_share->get_table_part(part_id);
  m_prebuilt->index = innopart_get_index(part_id, active_index);
}
```

**逐段解释**：

1. **越界即 `ut_error`**：分区号越界是内部逻辑错误，debug 版直接断言失败。
2. **换游标指针**：`pcur`（二级索引游标）与 `clust_pcur`（聚簇索引游标）必须换成该分区自己的游标——否则会把上一个分区的游标位置带过来，读到的页属于错分区。
3. **恢复执行现场**：`ins_node` / `upd_node` 是 row insert/update 的执行节点（含已构造的 dtuple 与 heap），`trx_id` / `row_read_type` / `new_rec_lock` 是该分区上次的锁与读类型。
4. **`blob_heap` 的特殊分支**：有序扫描（多路归并）时堆里同时有 N 个候选行，每分区要用自己的 blob heap；无序扫描 / 全表扫描同一时刻只有一行，统一用**分区 0** 的 blob heap，省内存也避免重复分配。
5. **`sql_stat_start` 按位维护**："语句开始"标志，指示下次访问该分区时要不要重新做语句级初始化（如重新评估半一致性读）。
6. **最后两行是切换的实质**：换掉 `table`（`dict_table_t`）与 `index`（`dict_index_t`）——后续所有 `ha_innobase` 的代码都以为自己在一张普通表上工作。

`update_partition()` 是反向拷贝（`m_prebuilt` → `m_parts[part_id]`），每个分区用完时调用，成对出现。

### 3.5 行 → 分区：`get_partition_id` 函数指针族

`get_partition_id` 是**函数指针**，在 `fix_partition_func` → `set_up_partition_func_pointers` 时绑定到 14 种实现之一（源码注释："每写/删一行执行一次，更新执行两次"）：

| 类型 | 函数 | 算法要点 |
|---|---|---|
| RANGE | `get_partition_id_range` | `part_val_int` 求值；NULL → 分区 0；无符号值统一减 `0x8000...`；**二分** `range_int_array`；越界且无 MAXVALUE → `HA_ERR_NO_PARTITION_FOUND` |
| RANGE COLUMNS | `get_partition_id_range_col` | `cmp_rec_and_tuple` 元组比较 + 二分 |
| LIST | `get_partition_id_list` | 二分 `list_array`；NULL 走 `has_null_part_id` |
| LIST COLUMNS | `get_partition_id_list_col` | 二分 `list_col_array` |
| HASH | `get_part_id_hash` | `*func_value % num_parts`，负值取绝对值 |
| LINEAR HASH | `get_part_id_linear_hash` | `get_part_id_from_linear_hash`（位运算，避免取模） |
| KEY | `get_part_id_key` | `calculate_key_hash_value(field_array) % num_parts`（引擎提供哈希，`ha_innopart` 里是 `ph_calculate_key_hash_value`） |
| 有子分区 | `get_partition_id_with_sub` | `part * num_subparts + subpart` |

找不到分区时：`m_part_info->err_value = func_value` → `print_partition_error` → `my_error(ER_NO_PARTITION_FOR_GIVEN_VALUE)`。

### 3.6 分区裁剪：把 WHERE 变成分区集合

> **算法只留一处**：`get_mm_tree` 假索引的构造、`SEL_ARG` 树的递归遍历、各分区类型的端点二分，全部见 [`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md)。裁剪器住在 `sql/range_optimizer/` 下，第一主语是优化器，所以算法细节归它。
>
> 本节只讲**使用者视角**：两个时点、什么条件下能裁、结果落在哪、怎么确认。

一句话原理：裁剪器把分区列（+ 子分区列）拼成一个**虚拟索引**喂给 range 优化器的 `get_mm_tree`，区间代数（AND/OR/IN 归约、求交求并、min/max key 打包）全部免费复用——**这个复用既是裁剪能力的来源，也是所有失效场景的来源**。

#### 两个时点

`prune_partitions(thd, table, query_block, cond)` 没有 `read_partitions` 参数，结果直接写回 `table->part_info->read_partitions`。它跑两次：

1. **prepare 期**：未加锁，子查询/存储程序还不能求值，用来确定 `lock_partitions`；若全部分区被裁掉且非外连接内表 → `set_empty_query()`，语句根本不执行。
2. **optimize 期**：已加锁，产出最终的 `read_partitions`。

`is_pruning_completed` 决定还需不需要第三次（条件非常量且在 prepare 期时保留 `false`）。**执行期没有 per-row 裁剪**——唯一的逐行路径是 INSERT 的 `can_prune_insert`。

#### 能不能裁，取决于三件事

| 条件 | 影响 |
|---|---|
| **分区表达式是否单调** | 单调（如 `TO_DAYS`）才能把 `F(x) op c` 反解为 `x op' F⁻¹(c)`（`Item::val_int_endpoint`），RANGE/LIST 据此在排序数组上二分；非单调只能裁等值点 |
| **分区类型能否反解** | RANGE/LIST：二分 `range_int_array` / `list_array`，O(log n)；**HASH/KEY：`% num_parts` 不可反解**，只能等值点正向算，或逐值枚举（值域 ≤ `MAX_RANGE_TO_WALK`(32) 且 ≤ 2×分区数），因此基本只有 `IS NULL` 能稳定裁 |
| **条件落在哪个分区列** | 只有**第 1 个**分区列带条件才裁；落在第 2 个及以后直接放弃（源码注释指向 WL#4065） |

两点补充：

- **NULL 要单独处理**：`PARTITION_ITERATOR::ret_null_part` 标志——`TO_DAYS('0000-00-00')` 这类求值为 NULL 的行落在最低分区，区间左端是 `NULL <= X` 时必须额外把分区 0 加回来。
- **子分区是两级笛卡尔积**：先求一级分区集合，再对每个一级分区求子分区集合，按 `part * num_subparts + subpart` 写回全局分区号。

#### 结果落在哪：两份位图

```cpp
bitmap_intersect(&read_partitions, &lock_partitions);   // 必须是锁集的子集
if (!locked && !partition_key_modified(table, table->write_set))
  bitmap_copy(&lock_partitions, &read_partitions);      // 反向收缩锁集
if (bitmap_is_clear_all(&read_partitions)) table->all_partitions_pruned_away = true;
```

为什么必须两份、以及"改分区键的 UPDATE 锁全分区"的代价，见「理论基础」里关于双位图的权衡。

#### 怎么确认裁剪生效

1. `EXPLAIN` 的 `partitions` 列 / `EXPLAIN FORMAT=JSON` 的 `"partitions"` 数组，由 `make_used_partitions_str` 遍历 `read_partitions` 生成（子分区写成 `p_sp`）。**注意它的注释明写：UPDATE 只显示"读到的分区"，不含写入/锁定分区**。
2. DBUG trace（`dbug_print_segment_range`、`"Mark partition %u as used"`）。
3. **optimizer trace 没有分区裁剪专属节点**——`partition_pruning.cc` 全文件不含 `Opt_trace_object`，这是验证裁剪的盲区。

#### 失效清单（统一兜底 `mark_all_partitions_as_used`）

```cpp
bitmap_intersect(&read_partitions, &lock_partitions);   // 必须是锁集的子集
if (!locked && !partition_key_modified(table, table->write_set))
  bitmap_copy(&lock_partitions, &read_partitions);      // 反向收缩锁集
if (bitmap_is_clear_all(&read_partitions)) table->all_partitions_pruned_away = true;
```

EXPLAIN 的 `partitions` 列由 `Explain_table_base::explain_partitions` → `make_used_partitions_str` 生成，遍历 `read_partitions` 收集分区名（子分区拼成 `p_sp`）。**注意它的注释明写：UPDATE 只显示"读到的分区"，不含写入/锁定分区**——这解释了为什么 `EXPLAIN UPDATE` 的 partitions 列经常看起来"少了"。

观察裁剪是否生效的手段：

1. `EXPLAIN` 的 `partitions` 列 / `EXPLAIN FORMAT=JSON` 的 `"partitions"` 数组；
2. DBUG trace（`dbug_print_segment_range`、`mark_full_partition_used_*` 里的 `"Mark partition %u as used"`）；
3. **optimizer trace 没有分区裁剪专属节点**——`partition_pruning.cc` 全文件不含任何 `Opt_trace_object`，这是验证裁剪的一个盲区。

#### 失效清单（统一兜底 `mark_all_partitions_as_used`）

| 场景 | 结果 |
|---|---|
| 分区列上有函数 / 无法生成 SEL_ARG（非 `SEL_TREE::KEY`） | 全分区 |
| 分区字段是 ENUM / GEOMETRY（`fields_ok_for_partition_index` 返回 false） | 全分区 |
| 分区表达式非单调 | 仅等值可裁，范围全分区 |
| OR 条件（任一子树 `res == -1`） | 全分区 |
| JOIN 条件 / 跨表依赖（产生 `SEL_ROOT::MAYBE_KEY`） | 多数无法裁剪 |
| IN 子查询 / 非常量（prepare 期谓词被丢弃） | 靠 optimize 期第二次裁剪补救 |
| 字符集/collation 不一致 | 退化为 `_via_walking` 或全分区 |
| 类型不匹配 / 转换失败 | 全分区 |
| 开区间 ±inf / NULL 边界（walking 路径） | 全分区 |
| 只有第 2 个及以后的分区列有条件 | 全分区 |
| 递归过深 / `range_optimizer_max_mem_size` 超限 | 全分区 |

### 3.7 写路径：`Partition_helper`

#### 三类角色的分工

| 角色 | 定位 |
|---|---|
| `Partition_handler` | **接口类**（纯虚），引擎向 server 暴露分区管理 API：`truncate_partition`、`exchange_partition`、`set_part_info`。由 `handler::get_partition_handler()` 返回 |
| `Partition_helper` | **可复用实现 mixin**（不是 handler 子类），实现跨分区扫描/写/改/删的通用骨架，把单分区动作留成 `*_in_part()` 纯虚 |
| `ha_innopart` | 多重继承 `ha_innobase` + `Partition_helper` + `Partition_handler` |

```cpp
// storage/innobase/handler/ha_innopart.h
class ha_innopart : public ha_innobase,
                    public Partition_helper,
                    public Partition_handler {
```

建 handler 时就按是否分区分流：`innobase_create_handler` 中 `partitioned ? new ha_innopart(...) : new ha_innobase(...)`。server 层**一个分区表只有一个 handler 实例**。

#### INSERT：分区号在运行期才算

```
write_record → handler::ha_write_row → ha_innopart::write_row
  → Partition_helper::ph_write_row
      → update_auto_increment()               ★ 先生成 auto_inc 值
      → get_partition_id(&part_id, &func_value)  ★ 再算分区号
      → is_partition_locked(part_id)?         否则 HA_ERR_NOT_IN_LOCK_PARTITIONS
      → write_row_in_part(part_id, buf)
```

顺序很重要——**先生成 auto_increment 值，再算分区号**。因为分区号依赖行内容，若先算分区再改 auto_inc 值，行就可能落错分区。紧接着还有一道保险：

```cpp
if (m_table->next_number_field->val_int() == 0) {
  m_table->autoinc_field_has_explicit_non_null_value = true;
  thd->variables.sql_mode |= MODE_NO_AUTO_VALUE_ON_ZERO;
}
```

注释解释得清楚：若分区 handler 再改 auto_inc 值，行就不匹配分区了，所以临时开启 `NO_AUTO_VALUE_ON_ZERO` 禁止二次生成。

引擎侧 `ha_innopart::write_row_in_part`：`set_partition(part_id)` → 清 `table->next_number_field`（防二次生成）→ `ha_innobase::write_row` → `update_partition(part_id)`。

#### auto_increment

计数器不是每分区一份，而是**全表共享** `Partition_share::next_auto_inc_val`：

```cpp
// Partition_helper::get_auto_increment_first_field（节选）
lock_auto_increment();
*first_value = m_part_share->next_auto_inc_val;
m_part_share->next_auto_inc_val += nb_desired_values * increment;
```

语句级复制（`binlog_format != ROW`）时整语句持锁，保证从库能推出连续值。另一个硬限制：

```cpp
assert(m_table->s->next_number_keypart == 0);
```

即**分区表的 auto_increment 必须是索引首列**，server 层报 `ER_WRONG_AUTO_KEY`。所谓"分区表 auto_increment 有空洞"并不准确——计数器是表级的，空洞来自常规原因（预留区间、回滚、REPLACE）。

### 3.8 改 / 删路径：跨分区移动与一致性校验

```cpp
// Partition_helper::ph_update_row（节选）
if ((error = get_parts_for_update(old_data, new_data, m_table->record[0],
                                  m_part_info, &old_part_id, &new_part_id, &func_value)))
  return error;
if (!m_part_info->is_partition_locked(new_part_id))
  return HA_ERR_NOT_IN_LOCK_PARTITIONS;
if (old_part_id != m_last_part) { m_err_rec = old_data; return HA_ERR_ROW_IN_WRONG_PARTITION; }

m_last_part = new_part_id;
if (new_part_id == old_part_id) {
  error = update_row_in_part(new_part_id, old_data, new_data);
} else {
  m_table->next_number_field = nullptr;              // 禁止移动时生成 auto_inc
  error = write_row_in_part(new_part_id, new_data);  // ★ 先插入新分区
  if (!error) error = delete_row_in_part(old_part_id, old_data);  // ★ 再删旧分区
}
```

- `get_parts_for_update` 通过 `set_field_ptr` 把分区字段临时指向 `old_data`，算完切回 —— 所以 **`get_partition_id` 在更新时被调用两次**。
- 物理上是 delete+insert，但 **binlog 仍记录为 `Update_rows_log_event`**，只是额外写入 `source_partition_id`（`Extra_row_info`），供从库做分区过滤。这个设计避免从库执行 delete+insert 两次索引维护。
- `HA_ERR_ROW_IN_WRONG_PARTITION` 是"游标所在分区 ≠ 算出的分区"的一致性校验失败信号，由 `print_partition_error` 转成 `ER_ROW_IN_WRONG_PARTITION` 并写 error log。
- 没有独立的"撤销已插入行"逻辑，一致性依赖语句/事务回滚。

删除同理，**重算分区号不是为了定位**（行来源缓存在 `m_last_part`），而是校验：

```cpp
if ((error = get_part_for_delete(buf, m_table->record[0], m_part_info, &part_id))) return error;
if (!m_part_info->is_partition_locked(part_id)) return HA_ERR_NOT_IN_LOCK_PARTITIONS;
if (part_id != m_last_part) { m_err_rec = buf; return HA_ERR_ROW_IN_WRONG_PARTITION; }
m_last_part = part_id;
error = delete_row_in_part(part_id, buf);
```

源码此处留了一句 TODO："把 InnoDB 里的 assert 改成 error，把这里的 error 改成 assert，并删掉 `get_part_for_delete()`"。

两个状态变量要分清：

- `m_last_part`：**当前行**所在分区，读成功时刷新；
- `m_part_spec`（`part_id_range{start_part, end_part}`）：**扫描区间**，由 `ph_rnd_init` / `partition_scan_set_up` 设置。

server 层还做了额外限制：`used_key_is_modified |= num_partitions_used() > 1 && partition_key_modified(...)`，即**可能跨分区移动时禁用"立即更新"快路径**——否则同一行可能被扫到两次。

### 3.9 读路径：串行扫 + 优先队列归并

全表扫描按 `read_partitions` 顺序**逐个分区串行**：当前分区 `HA_ERR_END_OF_FILE` 就 `rnd_end_in_part` → `get_next_used_partition` → `rnd_init_in_part`。

索引扫描分两种，判定在 `partition_scan_set_up`：

```cpp
if (start_part == end_part) {
  m_ordered_scan_ongoing = false;    // 单分区，无需排序
} else {
  m_ordered_scan_ongoing = m_ordered;  // m_ordered 来自 index_init(keynr, sorted)
}
```

- **无需有序**：`handle_unordered_scan_next_partition`，跳到下一个被标记的分区继续读；
- **需要有序**（如 `ORDER BY pk LIMIT 10`）：`handle_ordered_index_scan` 对所有用到的分区各 `index_read_map_in_part` 一次，把首行推进**优先队列**，之后每次从堆顶取行、从对应分区补一行：

```cpp
m_queue->m_max_at_top = m_reverse_order;
m_queue->m_keys = m_curr_key_info;
m_queue->assign(parts);
return_top_record(buf);
```

比较器是 `Key_rec_less`（用 `key_rec_cmp`）。聚簇索引时 `m_curr_key_info[1]` 作为次级排序键。**这意味着全局有序不依赖分区定义顺序，也不依赖 RANGE/HASH 类型**——代价是 O(log k) 堆操作与 k 份游标。

补充：8.0 **没有**分区间并行扫描（社区版无 SQL 层并行查询），只有上述多路归并；分区表会关掉 prefetch buffer（不感知分区），ICP 时直接读进 `record[0]`。

### 3.10 分区级 DDL 与 MDL

8.0 **已不存在** `mysql_change_partitions()` / `alter_table_partition()`（5.x 的名字，全仓 0 命中）。分区管理分两条路。

**① 结构变更（ADD / DROP / COALESCE / REORGANIZE / REBUILD PARTITION）**：

```
Sql_cmd_alter_table::execute → mysql_alter_table → prep_alter_part_table
  → 识别 ALTER_.*_PARTITION flags
  → table->file->get_partition_handler()
  → tab_part_info = table->part_info->get_full_clone(thd)
  → part_handler->alter_flags(...) & HA_INPLACE_CHANGE_PARTITION
  → 分派到 ADD / DROP / REBUILD / COALESCE / REORGANIZE 分支
  → *partition_changed = true; thd->work_part_info = tab_part_info
```

是否 in-place 由 `is_inplace_alter_impossible` 判定：只有引擎支持 `HA_USE_AUTO_PARTITION`，或 `prep_alter_part_table` 产出了 `new_part_info`（引擎声明 `HA_INPLACE_CHANGE_PARTITION`）时才 in-place，否则一律 rebuild。

引擎侧三阶段钩子（`handler0alter.cc`）：

```
ha_innopart::prepare_inplace_alter_partition   → 建 alter_parts，prepare 新旧分区
  → ha_innopart::inplace_alter_partition       → 需要时 copy_partitions(&deleted)
  → ha_innopart::commit_inplace_alter_partition
       → alter_parts::try_commit
            → 先提交"待删除分区"（清理数据文件）
            → 再提交"新增分区"
       → dd_copy_table + dd_part_adjust_table_id
```

底层动作：ADD → `innobase_basic_ddl::create_impl<dd::Partition>`；DROP → `delete_impl<dd::Partition>`（冲突时先 rename 到临时名再删）；回滚走 `alter_parts::rollback()`。"先删后建"的顺序是有意的：避免中间态同时持有两份空间。

**② 分区级维护**：

| 语句 | 入口 |
|---|---|
| TRUNCATE PARTITION | `Sql_cmd_alter_table_truncate_partition::execute` → `Partition_handler::truncate_partition` |
| ANALYZE / CHECK / OPTIMIZE / REPAIR PARTITION | `sql_partition_admin.cc` 各自的 `execute`，委派对应 `Sql_cmd_*`，设 `ALTER_ADMIN_PARTITION` flag |
| EXCHANGE PARTITION | `Sql_cmd_alter_table_exchange_partition::execute` → `exchange_partition` |
| DISCARD / IMPORT PARTITION TABLESPACE | `Sql_cmd_discard_import_tablespace::execute` → `mysql_discard_or_import_tablespace` |

**TRUNCATE PARTITION 的物理实现**：

```
Partition_handler::truncate_partition
  → ha_innopart::truncate_partition_low
      → for (dd_part : dd_table->leaf_partitions())
            innobase_truncate<dd::Partition> truncator(...)
            if (!m_part_info->is_partition_used(part_num++)) continue;   ★ 跳过未使用分区
            truncator.exec()
                → prepare()
                → truncate()
                     → rename_tablespace()                 旧 .ibd 重命名为临时名
                     → innobase_basic_ddl::delete_impl()   删除旧分区
                     → innobase_basic_ddl::create_impl()   新建空分区
                → cleanup()
                → load_fk()                                ★ 分区 truncate 跳过
```

`is_partition_used()` 过滤正是"TRUNCATE PARTITION p0 不影响 p1"的实现。本版本**没有** `truncate_by_reassign` 那种重新分配式实现，就是 drop+create。

**EXCHANGE PARTITION**：

引擎侧前置校验（`handler0alter.cc`）：

```cpp
if (dd_table_has_instant_cols(*part_table) || dd_table_has_instant_cols(*swap_table))
  → ER_PARTITION_EXCHANGE_DIFFERENT_OPTION "INSTANT COLUMN(s)"
if (dd_part->options().exists(index_file_name_key) || ...)
  → "INDEX DIRECTORY"
```

即：**分区与待交换表只要有一方带 instant column 就不能交换**（行格式多了 row version 字节，物理行不兼容）。

三次 rename：

```cpp
/* 1. Rename the swap table to the intermediate file
   2. Rename the partition to the swap table file
   3. Rename the intermediate file of swap table to the partition file */
innobase_basic_ddl::rename_impl<dd::Table>(thd, swap_name, temp_name, ...);
innobase_basic_ddl::rename_impl<dd::Partition>(thd, part_name, swap_name, ...);
innobase_basic_ddl::rename_impl<dd::Table>(thd, temp_name, part_name, ...);
```

随后交换每个索引的 `se_private_data` 与 `tablespace_id`，DATA DIRECTORY 走 `exchange_partition_adjust_datadir`。

**结构一致性与 `WITH VALIDATION` 在 SQL 层做，引擎只搬文件**：

| 检查 | 函数 | 说明 |
|---|---|---|
| 引擎能力 / 同引擎 / 非临时表 / 无外键 | `check_exchange_partition` | — |
| 结构比对（ROW_FORMAT 等不同 → `ER_PARTITION_EXCHANGE_DIFFERENT_OPTION`） | `compare_table_with_partition` | 永远做 |
| **`WITH VALIDATION` 逐行扫描**：确认每行确实属于该分区 | `verify_data_with_partition` | `WITHOUT VALIDATION` 时跳过 |

所以 `WITHOUT VALIDATION` 只是跳过全分区扫描，并不是"不检查结构"。

**分区级维护在引擎侧**：

| 操作 | 实现 |
|---|---|
| OPTIMIZE | 直接返回 `HA_ADMIN_TRY_ALTER`，让 SQL 层改写成 `ALTER TABLE ... ENGINE=InnoDB` 重建 |
| ANALYZE | `info_low(flag, is_analyze=true)`：逐分区 `update_table_stats`（持久统计 `DICT_STATS_RECALC_PERSISTENT`，否则 `DICT_STATS_RECALC_TRANSIENT`） |
| CHECK | 逐分区 `ha_innobase::check()` + `check_misplaced_rows(i, false)`（仅 `T_MEDIUM`/`T_EXTEND`） |
| REPAIR | `check_misplaced_rows(i, true)`——**只把放错分区的行搬走，不修索引** |
| REBUILD | 走 inplace alter 的 `alter_parts` 路径 |
| DISCARD / IMPORT | `discard_or_import_tablespace`：按 `read_partitions` 逐分区调用，再 `set_dd_discard_attribute` 逐分区改 DD 的 discarded 状态与 tablespace state，并回写每个 `dd_index` 的 `DD_INDEX_ROOT` |

`check_misplaced_rows` 是个补丁式机制：扫描分区，把"按分区函数本不该在这里"的行搬到正确分区（REPAIR）或仅报告（CHECK）。这类行通常来自分区定义被 ALTER 改动后未重分布的历史遗留。

**MDL 与锁**：

- `prep_alter_part_table` 断言至少持有 `MDL_INTENTION_EXCLUSIVE`；
- TRUNCATE PARTITION 取 `MDL_EXCLUSIVE`，`LOCK TABLES` 模式下 `upgrade_shared_lock`，并用 `set_partition_bitmaps` 限定只处理目标分区（避免全表 `external_lock`）；
- DISCARD/IMPORT 同样 `MDL_EXCLUSIVE`；
- 引擎声明 `HA_TRUNCATE_PARTITION_PRECLOSE` 时（InnoDB 就是），SQL 层会先关闭表并构造临时 `TABLE`/`TABLE_SHARE` 再调用。

### 3.11 统计：逐分区采集，向优化器"求和 + 取最大分区"

```cpp
/* Currently we track statistics for all partitions, but for
the secondary indexes we only use the biggest partition. */
for (uint part_id = 0; part_id < m_tot_parts; part_id++) {
  dict_stats_init(m_part_share->get_table_part(part_id));
}
```

- **行数类按"使用的分区"求和**：`records_in_range()` 对每个用到的分区调 `btr_estimate_n_rows_in_range` 并累加；任一分区不可用则整体返回 `HA_POS_ERROR`；最终结果至少为 1（防优化器误判空集）。`estimate_rows_upper_bound`、`scan_time`、`stat_clustered_index_size` 同理求和。
- **`rec_per_key`（索引区分度）只取最大分区**：`info_low` 先找 `biggest_partition`，所有 `rec_per_key` 都用该分区的索引。源码 TODO 明写"只对所有分区分析 PK，二级索引只分析最大分区"。
- `get_dynamic_partition_info()`（`Partition_helper::get_dynamic_partition_info_low`）：临时把 `read_partitions` 位图清成只剩 `part_id`，再 `info(HA_STATUS_TIME|VARIABLE|VARIABLE_EXTRA|NO_LOCK)`——这就是 `I_S.PARTITIONS` 里单分区行数的来源。

**这个简化假设的后果**：若分区间数据分布严重不均（例如 pmax 塞了 90% 数据，或大量空的历史分区），优化器拿到的 `rec_per_key` 与实际扫描代价偏差很大，可能错选索引。这是"分区表执行计划突然变差"的常见根因。

全表扫描的并行：`records()` 对每个使用分区 `set_partition(i)` 后加入 `Parallel_reader` 的 scan 列表——**分区是 InnoDB 内部并行扫描的天然分片单位**（详见 [`../innodb/parallel_scan.md`](../innodb/parallel_scan.md)）。

### 3.12 事务、锁与 2PC

**N 个分区共享 1 个 `trx_t`、1 个 `Ha_trx_info`、1 次 2PC 注册**：

```cpp
void innobase_register_trx(handlerton *hton, THD *thd, trx_t *trx) {
  trans_register_ha(thd, false, hton, &trx_id);
  if (!trx_is_registered_for_2pc(trx) &&
      thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
    trans_register_ha(thd, true, hton, &trx_id);
  }
  trx_register_for_2pc(trx);
}
```

`Ha_trx_info` 挂在 `THD` 的 `m_ha_list` 上，粒度是 `(THD, handlerton)`，**与分区数无关**。

锁粒度：

- 行锁 / 表锁的粒度是 `dict_table_t`——**每分区一张表**，所以天然是"分区级表锁"；落在哪个分区由 `set_partition()` 切换的 `m_prebuilt->table` 决定；
- `lock_partitions` 位图**不传给 InnoDB**，只在 SQL 层做写入合法性检查（写不在锁集的分区 → `HA_ERR_NOT_IN_LOCK_PARTITIONS`），InnoDB 侧只是把这个错误列为可忽略。

`external_lock` 的两个细节：全部分区被裁掉时直接返回（省一次锁）；`FLUSH TABLE ... FOR EXPORT` 要对所有分区设 quiesce 状态。

### 3.13 能力标志与限制

被禁用的能力（构造时从 `ha_innobase` 的标志里清掉）：

```cpp
/* HA_DUPLICATE_POS and HA_READ_BEFORE_WRITE_REMOVAL is not set from ha_innobase,
but cannot yet be supported in ha_innopart. Full text and geometry is not yet supported. */
const handler::Table_flags HA_INNOPART_DISABLED_TABLE_FLAGS =
    (HA_CAN_FULLTEXT | HA_CAN_FULLTEXT_EXT | HA_CAN_GEOMETRY |
     HA_DUPLICATE_POS | HA_READ_BEFORE_WRITE_REMOVAL);
```

- `table_flags()` = `ha_innobase::table_flags() | HA_CAN_REPAIR`（唯一新增）
- `index_flags()` 不覆写，沿用 `ha_innobase`
- 全文索引 API（`ft_init` / `ft_init_ext` / `ft_read`）直接 `ut_d(ut_error)`
- `disable_indexes` / `enable_indexes` 返回 `HA_ERR_WRONG_COMMAND`

handlerton 级能力：

```cpp
static uint innobase_partition_flags() {
  return (HA_CAN_EXCHANGE_PARTITION | HA_CANNOT_PARTITION_FK |
          HA_TRUNCATE_PARTITION_PRECLOSE);
}
```

Online DDL：多数情况回落到基类 `ha_innobase::check_if_supported_inplace_alter`；分区特有不支持项：ADD/DROP FOREIGN KEY、ADD FULLTEXT INDEX、改变 KEY 分区字段顺序、对 `PARTITION BY KEY()` 的表增删主键。

INSTANT ADD/DROP COLUMN（8.0.29+）：

```cpp
Instant_Type instant_type = innopart_support_instant(
    ha_alter_info, m_tot_parts, m_part_share, this->table, altered_table);
...
return HA_ALTER_INPLACE_INSTANT;
```

约束是**所有分区都必须支持**才可行：

```cpp
for (uint32_t i = 0; i < num_parts; ++i) {
  type = innobase_support_instant(ha_alter_info, part_share->get_table_part(i), ...);
  if (type == Instant_Type::INSTANT_IMPOSSIBLE) return (type);
}
```

（回想 EXCHANGE 那条限制：只要有一方有 instant column 就不能交换——两边是同一件事的正反面：instant 改变了物理行格式，而 EXCHANGE 不搬数据只换文件。）

**限制总表**：

| 限制 | 依据 |
|---|---|
| 分区表不能有外键（也不能被外键引用） | `HA_CANNOT_PARTITION_FK`；`sql_table.cc` 四处拦截报 `ER_FOREIGN_KEY_ON_PARTITIONED` |
| 分区表不支持全文索引 | `HA_INNOPART_DISABLED_TABLE_FLAGS` 去掉 `HA_CAN_FULLTEXT`；`ft_init` 等直接 `ut_error` |
| 分区不能放共享表空间 | `ha_innopart::create` 报 `PARTITION_IN_SHARED_TABLESPACE` |
| 不支持临时分区表 | `ha_innopart::create` 报 `ER_PARTITION_NO_TEMPORARY` |
| 只支持 native 分区引擎 | 非 native 分区引擎报 `ER_CHECK_NOT_IMPLEMENTED, "native partitioning"` |
| 只有 RANGE/LIST 可子分区 | `check_partition_info` 报 `ER_SUBPARTITION_ERROR` |

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 对分区的影响 |
|--------|--------|--------|--------------|
| `range_optimizer_max_mem_size` | 8 MB | Global/Session | 裁剪器构造 SEL_ARG 时用它给 MEM_ROOT 设容量上限；超限直接放弃裁剪 → 全分区扫描 |
| `range_alloc_block_size` | 4 KB | Global/Session | 裁剪专用 MEM_ROOT 的初始块大小（PSI key `key_memory_partitions_prune_exec`） |
| `innodb_file_per_table` | ON | Global | 分区表**强制** file-per-table；分区放共享表空间直接报 `PARTITION_IN_SHARED_TABLESPACE` |
| `innodb_stats_persistent` | ON | Global | 决定分区 ANALYZE 走 `DICT_STATS_RECALC_PERSISTENT` 还是 `DICT_STATS_RECALC_TRANSIENT` |

### 常量

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_PARTITIONS` | 8192 | 单表分区总数上限（`sql/sql_const.h`），超限报 `ER_TOO_MANY_PARTITIONS_ERROR` |
| `MAX_RANGE_TO_WALK` | 32 | HASH/KEY 裁剪时"逐值枚举"的值域上限，超过即放弃 |
| `MAX_REF_PARTS` | — | 分区字段数量上限，报 `ER_TOO_MANY_PARTITION_FUNC_FIELDS_ERROR` |

---

## Misc

### 为什么本篇在 `feat/` 而不是 `server/` 或 `innodb/`

本知识库的主轴是按层组织（`server/` `innodb/`），但**跨层端到端特性**不适用"第一主语按层归位"：分区表同时改 SQL 层（语法、元数据、优化器、MDL）与引擎层（存储、分区 DDL），两半是同一对象的定义与实现，拆成两篇会双双断头。

判据有三条（满足任一即进 `feat/`）：

1. **同时改动多个层的同一特性**：分区、全文索引、GIS 这类"语法 → 优化器 → 引擎存储"全链路特性；
2. **可能被多个引擎实现**：若按引擎拆篇，N 个引擎就是 N 篇，且每篇都要复述一遍 server 层——正确做法是**一篇 + "引擎差异"小节**（分区表目前只有 InnoDB 与 NDB 两家：NDB 走 `HA_USE_AUTO_PARTITION` 的自动分区路径并保留 `nodegroup_id`，未声明 native 分区能力的引擎直接被拒绝）；
3. **读者需要一次读完闭环**：分区表的 DML 路径横跨 `ph_write_row`（SQL 层算分区号）与 `set_partition`（引擎换 `dict_table_t`），中间只隔一层虚调用。

不属于此类的仍按层归位：纯引擎内部机制（buffer pool、redo）进 `innodb/`，纯 SQL 层机制（MDL、优化器）进 `server/`，两层接口本身进 `server/handler.md`。

#### 为什么本篇用"生命周期"而不是"server: / innodb:" 分节

`feat/` 的篇内组织有两种形态（见 [`README.md`](README.md)）：两层职责清晰可分时按层分节，两层动作交织时按生命周期。分区表属于后者：一次写入是"SQL 层算分区号 → 引擎换 `dict_table_t` → 写行"的紧耦合序列，强行按层分节会把同一次写入拆到两节里，读者要在两节之间来回对照。因此本篇的每个环节都在节内闭环（SQL 层做什么 → 引擎层做什么），而不是分成两大块。

对照：SELECT 适合按层分节（解析优化 vs 扫描取行是清晰的两段接力）。

### 容易混淆的三份 `partition_info`

| 位置 | 生命周期 | 生成方式 |
|---|---|---|
| `LEX::part_info` | 语句解析期 | yacc 归约 `PT_partition` 时直接填充 |
| `TABLE_SHARE::m_part_info` | share 级，所有 TABLE 共享 | `fill_partitioning_from_dd` 从 DD 重建 |
| `TABLE::part_info` | 每个 `TABLE` 实例 | `unpack_partition_info` **重新解析** `share->partition_info_str` |

### `read_partitions` vs `lock_partitions`

| | 谁能改 | 作用 |
|---|---|---|
| `read_partitions` | 裁剪器、显式 `PARTITION (pX)` 选择 | 决定扫哪些分区；写入时校验 |
| `lock_partitions` | 裁剪器（仅在分区键不被修改时）、`set_partition_bitmaps` | 决定 `external_lock` 锁哪些分区；写入目标必须在此集合内 |

恒有 `read_partitions ⊆ lock_partitions`。

### 三个"X 分区"不是一回事

| 名称 | 含义 |
|---|---|
| 分区裁剪（pruning） | 优化期推导"不必访问哪些分区"，写 `read_partitions` |
| 分区选择（`FROM t PARTITION (p0)`） | 显式指定分区集合，直接设位图，不走裁剪 |
| `all_partitions_pruned_away` | 裁剪结果为空 → prepare 期可把整个查询短路掉 |

### 术语对照

| 术语 | 含义 |
|---|---|
| **leaf partition** | 叶子分区 = 最终承载数据的分区。有子分区时 leaf 是子分区，否则就是一级分区。`leaf_partitions()` 是引擎遍历分区的统一入口 |
| **per-partition tablespace** | 8.0 每个分区可指定自己的 tablespace（`dd::Partition::tablespace_id`），`I_S.PARTITIONS` 的 `TABLESPACE_NAME` 列由此而来 |
| `se_private_id` / `se_private_data` | DD 里给引擎用的私有字段：前者通常是 space id，后者是 index root page no 等物理位置。EXCHANGE 时要交换的就是这些 |
| `saved_prebuilt_t` | `ha_innopart` 为每个分区保存的 `row_prebuilt_t` 关键字段快照 |

### 常见误解

| 说法 | 事实 |
|---|---|
| "分区表 auto_increment 每分区独立" | 错，全表共享 `Partition_share::next_auto_inc_val` |
| "分区表的索引是全局的" | 错，每分区一套本地索引 → 唯一键必须含分区列 |
| "分区裁剪能减少打开表开销" | 错，`open` 无条件打开全部 leaf 分区 |
| "TRUNCATE PARTITION 是快速 DELETE" | 错，是 drop+create 表空间，无 undo、立即释放空间 |
| "UPDATE 跨分区 binlog 记成 delete+insert" | 错，仍是 `Update_rows` + `source_partition_id` |

### 与 `innodb/parallel_scan.md` 的关系

分区是 InnoDB 内部并行扫描（`Parallel_reader`）的天然分片单位：全表扫描时每个分区可作为一路 scan 加入并行读。这是**引擎内部**并行，社区版没有 SQL 层并行查询——详见 [`../innodb/parallel_scan.md`](../innodb/parallel_scan.md)。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Partitioning*：https://dev.mysql.com/doc/refman/8.0/en/partitioning.html
- *MySQL 8.0 Reference Manual → Restrictions and Limitations on Partitioning*：https://dev.mysql.com/doc/refman/8.0/en/partitioning-limitations.html （唯一键必须包含分区列、外键互斥等限制）
- *MySQL 8.0 Reference Manual → Partition Pruning*：https://dev.mysql.com/doc/refman/8.0/en/partitioning-pruning.html
- *MySQL 8.0 Reference Manual → Exchanging Partitions and Subpartitions with Tables*：https://dev.mysql.com/doc/refman/8.0/en/partitioning-management-exchange.html

**WorkLog（源码注释中实际引用的两个）**
- WL#4827：把 `partition_info` / `partition_element` 移进 partitioning service —— 至今未做，`sql/dd/dd_table.cc` 仍留 `TODO-PARTITION`
- WL#4065：分区裁剪的进一步场景（多分区列条件）—— `partition_pruning.cc` 注释里作为"当前不支持"的去向

**内核月报**
- 数据库内核月报（阿里云/PolarDB 内核团队）：http://mysql.taobao.org/monthly/ —— 分区相关文章多基于 5.6/5.7 的 `ha_partition` 时代，与 8.0 的 `ha_innopart` 结构差异极大（如 5.x 的 `PartitionPruning` / `PART_PRUNE_PARAM` 类层次在本版已重构为自由函数 + 假索引），函数名与行号需回源码核实

**相关文档**
- 裁剪算法的优化器视角：[`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md)
- handler / handlerton 分界面：[`../server/handler.md`](../server/handler.md)
- 原子 DDL 与 DDL log（分区 TRUNCATE / DROP 的崩溃安全来源）：[`../innodb/ddl.md`](../innodb/ddl.md)
- InnoDB 内部并行扫描：[`../innodb/parallel_scan.md`](../innodb/parallel_scan.md)
