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
  - [一、元数据：建表 → DD 持久化 → 打开与对象模型](#一元数据建表--dd-持久化--打开与对象模型)
  - [二、运行期路由：行 → 分区、写、改删、读、裁剪](#二运行期路由行--分区写改删读裁剪)
  - [三、分区级 DDL：结构变更 / TRUNCATE / EXCHANGE / 维护](#三分区级-ddl结构变更--truncate--exchange--维护)
  - [四、分区 DDL 的原子性与崩溃恢复](#四分区-ddl-的原子性与崩溃恢复)
  - [五、统计、代价估算与优化器反馈](#五统计代价估算与优化器反馈)
  - [六、事务、锁、2PC 与死锁](#六事务锁2pc-与死锁)
  - [七、复制与并行](#七复制与并行)
  - [八、能力边界与限制](#八能力边界与限制)
- [可观测性](#可观测性)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
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

### 一、元数据：建表 → DD 持久化 → 打开与对象模型

#### 1.1 建表：语法 → `partition_info` → 校验 → DD → 引擎

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

**★★ `partition_info` 类的真实成员**（⚠️ 在 **`sql/partition_info.h:209`**，不在 `sql/sql_partition.h`——后者只有自由函数声明）：

```cpp
class partition_info {
 public:
  List<partition_element> partitions;       /* ★ 主分区链表 */
  List<partition_element> temp_partitions;  /* ★ ALTER 期间的临时链表 */

  List<char> part_field_list;               /* KEY / COLUMNS 的列名列表 */
  List<char> subpart_field_list;

  /*
    If there is no subpartitioning, use only this func to get partition ids.
    If there is subpartitioning use this to get the partition_id which will
    consider the subpartition as well. ...
  */
  get_part_id_func get_partition_id;        /* ★★ 主入口：UPDATE 调两次、写/删各一次 */
  get_part_id_func get_part_partition_id;   /* 只算主分区维，忽略子分区 */
  get_subpart_id_func get_subpartition_id;  /* 只算子分区维（签名无 func_value） */
  get_part_id_func get_part_partition_id_charset;   /* ★ 字符集转换包装层 */
  get_subpart_id_func get_subpartition_id_charset;

  Field **part_field_array;
  Field **subpart_field_array;
  Field **part_charset_field_array;
  Field **subpart_field_array;
  Field **full_part_field_array;            /* 去重后的全部分区字段 */
  MY_BITMAP full_part_field_set;            /* 供 UPDATE 时扩 read_set */

  uchar **part_field_buffers;               /* strnxfrm 归一化缓冲 */
  uchar **subpart_field_buffers;
  uchar **restore_part_field_ptrs;
  uchar **restore_subpart_field_ptrs;

  Item *part_expr;
  Item *subpart_expr;
  Item *item_list;

  /*
    Bitmaps of partitions used by the current query.
    * read_partitions  - partitions to be used for reading.
    * lock_partitions  - partitions that must be locked (read or write).
    Usually read_partitions is the same set as lock_partitions, but
    in case of UPDATE the WHERE clause can limit the read_partitions set,
    but not necessarily the lock_partitions set.
    ...
  */
  MY_BITMAP read_partitions;                /* ★★ 类型 MY_BITMAP（C 结构），不是 Bitmap<> 模板 */
  MY_BITMAP lock_partitions;
  bool bitmaps_are_initialized;

  union {
    longlong *range_int_array;              /* RANGE：每个分区的 LESS THAN 上界 */
    LIST_PART_ENTRY *list_array;            /* LIST：排序后的 (值 → 分区号) 条目 */
    part_column_list_val *range_col_array;  /* RANGE COLUMNS */
    part_column_list_val *list_col_array;   /* LIST COLUMNS */
  };

  get_partitions_in_range_iter get_part_iter_for_interval;     /* 区间分析（裁剪用） */
  get_partitions_in_range_iter get_subpart_iter_for_interval;

  longlong err_value;
  char *part_func_string;                   /* 分区表达式文本 */
  char *subpart_func_string;
  uint num_columns;

  TABLE *table;
  Key_map all_fields_in_PF, all_fields_in_PPF, all_fields_in_SPF;
  Key_map some_fields_in_PF;

  handlerton *default_engine_type;
  partition_type part_type;                 /* ★ 只有 4 个值，见下 */
  partition_type subpart_type;
  uint num_parts;
  uint num_subparts;
  uint num_list_values;
  uint num_part_fields;
  uint num_subpart_fields;
  uint num_full_part_fields;
  uint has_null_part_id;
  uint16 linear_hash_mask;                  /* LINEAR KEY/HASH 用 */
  enum_key_algorithm key_algorithm;

  bool use_default_partitions;
  bool use_default_num_partitions;
  bool use_default_subpartitions;
  bool use_default_num_subpartitions;
  bool default_partitions_setup;
  bool defined_max_value;
  bool list_of_part_fields;                 // KEY or COLUMNS PARTITIONING
  bool list_of_subpart_fields;
  bool linear_hash_ind;                     // LINEAR HASH/KEY
  bool fixed;
  bool is_auto_partitioned;
  bool has_null_value;
  bool column_list;                         // COLUMNS PARTITIONING, 5.5+
  bool is_pruning_completed;
};
```

> ★★ **`get_partition_id` 是 5 个函数指针成员，不是虚函数**。注释原文说明调用频率：*"This is to speed up execution of get_partition_id which is executed once every record to be written and deleted and twice for updates."*
> ⚠️ **`read_partitions` / `lock_partitions` 类型是 `MY_BITMAP`（`my_bitmap.h` 的 C 结构）**，不是 `Bitmap<>` 模板。
> ⚠️ **`TABLE_SHARE` 里叫 `m_part_info`，`TABLE` 里才叫 `part_info`**——两个不同的对象，`TABLE` 那份是每个实例重新 parse 出来的。

**★ 四个枚举的真名与全部值**：

```cpp
/* sql/partition_element.h:33 —— ★ enum class，只有 4 个值！没有 KEY */
enum class partition_type { NONE = 0, RANGE, HASH, LIST };

/* ★★ KEY 不是独立枚举值：KEY = partition_type::HASH + list_of_part_fields==true
       LINEAR 靠 linear_hash_ind；算法版本靠 enum_key_algorithm */

/* sql/partition_element.h:35 —— ★ 真名 partition_state，不是 enum_part_state（10 个值） */
enum partition_state {
  PART_NORMAL = 0, PART_IS_DROPPED = 1, PART_TO_BE_DROPPED = 2,
  PART_TO_BE_ADDED = 3, PART_TO_BE_REORGED = 4, PART_REORGED_DROPPED = 5,
  PART_CHANGED = 6, PART_IS_CHANGED = 7, PART_IS_ADDED = 8, PART_ADMIN = 9
};

/* sql/partition_info.h:174 */
enum class enum_key_algorithm {
  KEY_ALGORITHM_NONE = 0,
  KEY_ALGORITHM_51 = 1,   /* Use 5.1 hashing (numeric fields as binary) */
  KEY_ALGORITHM_55 = 2    /* Use 5.5 hashing (numeric like latin1 bytes) */
};
```

> ★★ **server 侧 4 个值 vs DD 侧 12 个值**（`sql/dd/types/table.h:91`）是两套枚举，靠 `fill_dd_partition_from_create_info()` / `fill_partitioning_from_dd()` 的 switch 互转：
> ```cpp
> enum enum_partition_type { PT_NONE=0, PT_HASH, PT_KEY_51, PT_KEY_55,
>     PT_LINEAR_HASH, PT_LINEAR_KEY_51, PT_LINEAR_KEY_55, PT_RANGE, PT_LIST,
>     PT_RANGE_COLUMNS, PT_LIST_COLUMNS, PT_AUTO, PT_AUTO_LINEAR };
> ```

**★ `partition_element` 完整定义**（单个分区描述，含二级嵌套）：

```cpp
class partition_element {
 public:
  List<partition_element> subpartitions;    /* ★ 第二层嵌套 */
  List<part_elem_value> list_val_list;      // LIST 的值 / COLUMNS 元组数组
  ha_rows part_max_rows;
  ha_rows part_min_rows;
  longlong range_value;                     /* RANGE 的 LESS THAN 值 */
  const char *partition_name;
  const char *tablespace_name;
  char *part_comment;
  const char *data_file_name;
  const char *index_file_name;
  handlerton *engine_type;
  enum partition_state part_state;          /* ★ ALTER 期间的生命周期状态 */
  uint16 nodegroup_id;
  bool has_null_value;                      /* LIST 分区含 NULL */
  bool signed_flag;                         /* Range value signed */
  bool max_value;                           /* MAXVALUE range */
};
```

```cpp
/* COLUMNS 分区的元组值（part_column_list_val） */
typedef struct p_column_list_val {
  union column_value_union {
    const uchar *field_image;  /* 打开表后：字段镜像 */
    const char *value_str;     /* 从 dd::Partition_value 读出：C 串 */
  } column_value;
  Item *item_expression;
  partition_info *part_info;
  uint partition_id;
  bool max_value;              /* MAXVALUE（仅 RANGE COLUMNS） */
  bool null_value;             /* NULL（仅 LIST COLUMNS） */
  char fixed;                  /* ★ 1 = add_column_list_values 部分 fix；2 = 完整 fix */
} part_column_list_val;

typedef struct p_elem_val {
  longlong value;
  uint added_items;
  bool null_value;
  bool unsigned_flag;
  part_column_list_val *col_val_array;
} part_elem_value;
```

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

**★ 校验主函数的真名与关键点**（⚠️ 在 **`sql/partition_info.cc:1413`**，不在 `sql_partition.cc`；第 5 参真名 `add_or_reorg_part`）：

```cpp
bool partition_info::check_partition_info(THD *thd, handlerton **eng_type,
                                          handler *file, HA_CREATE_INFO *info,
                                          bool add_or_reorg_part) {
  ...
  if (!add_or_reorg_part) {
    if (!list_of_part_fields) {
      err = part_expr->walk(&Item::check_partition_func_processor,
                            enum_walk::POSTFIX, nullptr);
    }
    if (!err && is_sub_partitioned() && !list_of_subpart_fields) {
      err = subpart_expr->walk(&Item::check_partition_func_processor,
                               enum_walk::POSTFIX, nullptr);
    }
    if (err) { my_error(ER_PARTITION_FUNCTION_IS_NOT_ALLOWED, MYF(0)); goto end; }
    if (thd->lex->sql_command == SQLCOM_CREATE_TABLE && fix_parser_data(thd))
      goto end;
  }
  /* ★ 只有 RANGE/LIST 可子分区 */
  if (unlikely(is_sub_partitioned() &&
               (!(part_type == partition_type::RANGE ||
                  part_type == partition_type::LIST)))) {
    my_error(ER_SUBPARTITION_ERROR, MYF(0));
    goto end;
  }
  if (!(tot_partitions = get_tot_partitions())) {
    my_error(ER_PARTITION_NOT_DEFINED_ERROR, MYF(0), "partitions");
    goto end;
  }
  if (unlikely(tot_partitions > MAX_PARTITIONS)) {    /* ★ 唯一上限检查点 */
    my_error(ER_TOO_MANY_PARTITIONS_ERROR, MYF(0));
    goto end;
  }
  ... 引擎混用检查 ER_MIX_HANDLER_ERROR、分区名合法性 ...
  if (add_or_reorg_part) {
    if (unlikely(
            (part_type == partition_type::RANGE && check_range_constants(thd)) ||
            (part_type == partition_type::LIST && check_list_constants(thd))))
      goto end;
  }
  result = false;
end:
  return result;
}
```

> ★★ **`MAX_PARTITIONS` = 8192**（`sql/sql_const.h:118`，`constexpr const int MAX_PARTITIONS{8192};`）——⚠️ 不在 `my_base.h`，也不是 1024（那是 5.x 的值）。
> ⚠️ **`get_tot_partitions()` = `num_parts * (is_sub_partitioned() ? num_subparts : 1)`**——**叶子分区总数**，所以上限卡的是叶子数而非主分区数。

**★ "unique key 必须含所有分区列"不在这个函数里**（★ 这是最容易找错的一处）：

```cpp
/* sql/sql_partition.cc:1011 / 1040 */
static bool check_primary_key(TABLE *table) {
  if (primary_key < MAX_KEY) {
    set_indicator_in_key_fields(table->key_info + primary_key);
    check_fields_in_PF(table->part_info->full_part_field_array, &all_fields, &some_fields);
    clear_indicator_in_key_fields(table->key_info + primary_key);
    if (unlikely(!all_fields)) {
      my_error(ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF, MYF(0), "PRIMARY KEY");
      result = true;
    }
  }
  return result;
}

static bool check_unique_keys(TABLE *table) {
  for (i = 0; i < keys; i++) {
    if (table->key_info[i].flags & HA_NOSAME) { ... 同上 ... }
  }
}
```

```cpp
/* 调用点（fix_partition_func 尾部，sql/sql_partition.cc:1556） */
    if (unlikely(!part_info->column_list &&
                 part_info->part_expr->result_type() != INT_RESULT)) {
      /* ★ 分区函数返回值必须是 INT_RESULT（COLUMNS 分区除外） */
      part_info->report_part_expr_error(false);
      goto end;
    }
  ...
  if (unlikely(create_full_part_field_array(thd, table, part_info))) goto end;
  if (unlikely(check_primary_key(table))) goto end;
  if (unlikely((!(table->s->db_type()->partition_flags &&
                  (table->s->db_type()->partition_flags() &
                   HA_CAN_PARTITION_UNIQUE))) &&
               check_unique_keys(table)))     /* ★★ 引擎无 HA_CAN_PARTITION_UNIQUE 时才强制 */
    goto end;
  if (unlikely(set_up_partition_bitmaps(part_info))) goto end;   /* ★ "无条件全开"的入口 */
  ...
  set_up_partition_func_pointers(part_info);
  set_up_range_analysis_info(part_info);
```

> ★★ **`HA_CAN_PARTITION_UNIQUE` 这个引擎标志才是关键**——只有引擎**不**具备"跨分区唯一"能力时，才强制 unique key 全覆盖分区列。所以这条规则不是绝对的，是"引擎能力不足时的补偿"。

**★ `check_partition_func_processor` 的 Doxygen 讲清了"什么是允许的分区函数"**：

```cpp
    @details
    check_partition_func_processor is used to check if a partition function
    uses an allowed function. An allowed function will always ensure that
    X=Y guarantees that also part_function(X)=part_function(Y) where X is
    a set of partition fields and so is Y. The problems comes mainly from
    character sets where two equal strings can be quite unequal. E.g. the
    german character for double s is equal to 2 s.

    The default is that an item is not allowed in a partition function. ...
    If a new Item class is defined and it inherits from a class that is
    allowed in a partition function then it is very important to consider
    whether this should be inherited to the new class.
  virtual bool check_partition_func_processor(uchar *) { return true; }
```

> ★★ 判据是 **`X=Y` 必须蕴含 `part_function(X)=part_function(Y)`**——这个不变式一旦破坏，同一行在不同时刻可能算出不同分区。字符集是主要风险源（德语 `ß` == `ss`）。**默认是"不允许"**，靠继承白名单放开。

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

#### 1.2 元数据持久化：DD → 文本 → 重新解析

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

**★ 写入侧真名**（⚠️ **不是** `dd_write_partition_info()`，是 **`fill_dd_partition_from_create_info()`**，`sql/dd/dd_table.cc:1422`）：

```cpp
static bool fill_dd_partition_from_create_info(
    THD *thd, dd::Table *tab_obj, const HA_CREATE_INFO *create_info,
    const List<Create_field> &create_fields, partition_info *part_info) {
  if (part_info) {
    switch (part_info->part_type) {
      case partition_type::RANGE:
        if (part_info->column_list) tab_obj->set_partition_type(dd::Table::PT_RANGE_COLUMNS);
        else                        tab_obj->set_partition_type(dd::Table::PT_RANGE);
        break;
      case partition_type::LIST:  ... PT_LIST_COLUMNS / PT_LIST ...
      case partition_type::HASH:
        if (part_info->list_of_part_fields) {        /* ★ KEY */
          if (part_info->linear_hash_ind)
            tab_obj->set_partition_type(part_info->key_algorithm == KEY_ALGORITHM_51
                                        ? PT_LINEAR_KEY_51 : PT_LINEAR_KEY_55);
          else
            tab_obj->set_partition_type(part_info->key_algorithm == KEY_ALGORITHM_51
                                        ? PT_KEY_51 : PT_KEY_55);
        } else {
          tab_obj->set_partition_type(part_info->linear_hash_ind
                                      ? PT_LINEAR_HASH : PT_HASH);
        }
        break;
    }
    if (part_info->is_auto_partitioned) { ... PT_AUTO / PT_AUTO_LINEAR ... }

    /* ★ 分区表达式：用 part_expr->print() 生成文本（关掉 ANSI_QUOTES 等模式影响） */
    Sql_mode_parse_guard parse_guard(thd);
    part_info->part_expr->print(thd, &tmp,
        enum_query_type(QT_TO_SYSTEM_CHARSET | QT_NO_DB | QT_NO_TABLE));
    if (tmp.numchars() > PARTITION_EXPR_CHAR_LEN) {
      my_error(ER_PART_EXPR_TOO_LONG, MYF(0));
      return true;
    }
    tab_obj->set_partition_expression(expr);
    tab_obj->set_partition_expression_utf8(expr_utf8);
    ...
    /* ★ 每个分区一个 dd::Partition 对象 */
    while ((part_elem = part_it++)) {
      if (part_elem->part_state == PART_TO_BE_DROPPED ||
          part_elem->part_state == PART_REORGED_DROPPED) {
        continue;                          /* ★ 待删除的分区不进新定义 */
      }
      dd::Partition *part_obj = tab_obj->add_partition();
      part_obj->set_name(part_elem->partition_name);
      part_obj->set_number(part_num);
      set_partition_options(part_elem, &part_obj->options());

      if (part_info->part_type == partition_type::RANGE) {
        if (part_info->column_list) { ... add_part_col_vals(...) ... }
        else {
          dd::Partition_value *val_obj = part_obj->add_value();
          if (part_elem->max_value) val_obj->set_max_value(true);
          else val_obj->set_value_utf8(
              part_elem->signed_flag
                ? dd::Properties::to_str(part_elem->range_value)
                : dd::Properties::to_str((ulonglong)part_elem->range_value));
        }
      } else if (part_info->part_type == partition_type::LIST) {
        if (part_elem->has_null_value) {   /* ★ LIST 的 NULL 是一个 value_null 条目 */
          dd::Partition_value *val_obj = part_obj->add_value();
          val_obj->set_value_null(true);
          val_obj->set_list_num(list_index++);
        }
        ... 逐个 add_value() ...
      } else {
        assert(part_info->part_type == partition_type::HASH);   /* KEY/HASH 无值 */
      }

      if (!part_info->is_sub_partitioned()) {
        /* ★ 无子分区时：每个 (分区, 索引) 建一条 dd::Partition_index */
        for (dd::Index *idx : *tab_obj->indexes()) part_obj->add_index(idx);
      } else {
        while ((sub_elem = sub_it++)) {
          dd::Partition *sub_obj = part_obj->add_subpartition();
          ...
          for (dd::Index *idx : *tab_obj->indexes()) sub_obj->add_index(idx);
        }
      }
      part_num++;
    }
  }
}
```

> ★★ **DD 里既有结构化列也有文本**：类型/表达式/值/索引都是**结构化列**（`dd::Partition`、`dd::Partition_index`、`dd::Partition_value` 各自一行）；文本只用于 `partition_expression` 和 `TABLE_SHARE::partition_info_str`。⚠️ 常见说法"DD 把分区信息存成一整段文本"是错的。
> ⚠️ **`sql/dd/impl/partition_impl.cc` 不存在**（真名 `sql/dd/impl/types/partition_impl.cc`）；`sql/dd/dd_table_share.cc` 也不存在（真名 `sql/dd_table_share.cc`，sql 根目录）。

**★ 读取侧：`fill_partitioning_from_dd()` 的枚举逆变换**（`sql/dd_table_share.cc:1906`）：

```cpp
static bool fill_partitioning_from_dd(THD *thd, TABLE_SHARE *share,
                                      const dd::Table *tab_obj) {
  if (tab_obj->partition_type() == dd::Table::PT_NONE) return false;
  /* ★ 二级引擎（如 HeatWave/RAPID）不参与分区元数据 */
  if (share->is_secondary_engine()) return false;

  part_info = new (&share->mem_root) partition_info;
  ...
  switch (tab_obj->partition_type()) {
    case dd::Table::PT_RANGE_COLUMNS:
      part_info->column_list = true;
      part_info->list_of_part_fields = true;
      [[fallthrough]];                     /* ★★ 靠 fallthrough 累积标志 */
    case dd::Table::PT_RANGE:
      part_info->part_type = partition_type::RANGE;
      break;
    case dd::Table::PT_LIST_COLUMNS: ... [[fallthrough]] ...
    case dd::Table::PT_LIST:   part_info->part_type = partition_type::LIST; break;
    case dd::Table::PT_LINEAR_HASH:
      part_info->linear_hash_ind = true;
      [[fallthrough]];
    case dd::Table::PT_HASH:   part_info->part_type = partition_type::HASH; break;
    case dd::Table::PT_LINEAR_KEY_55:
      part_info->linear_hash_ind = true;
      [[fallthrough]];
    case dd::Table::PT_KEY_55:
      part_info->key_algorithm = enum_key_algorithm::KEY_ALGORITHM_55;
      part_info->list_of_part_fields = true;
      part_info->part_type = partition_type::HASH;   /* ★★ KEY 落成 HASH + 标志位 */
      break;
    case dd::Table::PT_AUTO: ... is_auto_partitioned = true; share->auto_partitioned = true; ...
  }
  ...
  /* ★ 遍历 dd::Partition（已按 level + number 排序，有 assert 校验） */
  for (const dd::Partition *part_obj : tab_obj->partitions()) {
    curr_part_elem = new (&share->mem_root) partition_element;
    setup_partition_from_dd(thd, &share->mem_root, part_info,
                            curr_part_elem, part_obj, false);
    part_info->partitions.push_back(curr_part_elem, &share->mem_root);
    for (const dd::Partition *sub_part_obj : part_obj->subpartitions()) {
      ... setup_partition_from_dd(..., true) ...
      curr_part_elem->subpartitions.push_back(curr_sub_part_elem, &share->mem_root);
    }
  }
  part_info->num_parts = part_info->partitions.elements;
  part_info->num_subparts = part_info->partitions[0]->subpartitions.elements;

  /* ★★ 反向生成整段可解析的 PARTITION BY 文本，存进 TABLE_SHARE */
  buf = generate_partition_syntax(part_info, &buf_len, true, true, false, nullptr);
  share->partition_info_str = strmake_root(&share->mem_root, buf, buf_len);
  share->partition_info_str_len = buf_len;
  share->m_part_info = part_info;
  return false;
}
```

> ★★ **"DD → 文本 → 重新解析"这个绕路的理由**（源码注释原文）：*"[sub]part_expr still needs to be parsed from [sub]part_func_string for each TABLE instance to use the correct mem_root etc. To be as compatible with the .frm way to open a table as possible we currently generate the full partitioning clause which will be parsed for each new TABLE instance."*——**每个 TABLE 实例都要重新 parse 一次**，因为 Item 树必须挂在自己的 mem_root 上。
> ★ `generate_partition_syntax()` 用 **临时文件**（`mysql_tmpfile("psy")`）拼串，最后 `get_file_content()` 读回。

**★ `mysql_unpack_partition()` 复用 yacc 语法**（`sql/sql_partition.cc:3869`）：

```cpp
  thd->variables.character_set_client = system_charset_info;
  Sql_mode_parse_guard parse_guard(thd);
  Partition_expr_parser_state parser_state;
  if (parser_state.init(thd, part_buf, part_info_len)) goto end;
  if (init_lex_with_single_table(thd, table, &lex)) goto end;
  ...
  if (parse_sql(thd, &parser_state, nullptr) ||
      parser_state.result->fix_parser_data(thd)) {   /* ★ 复用同一个 fix_parser_data() */
    ...
  }
  part_info = parser_state.result;
  table->part_info = part_info;
  part_info->table = table;
  part_handler = table->file->get_partition_handler();
  part_handler->set_part_info(part_info, true);
```

> ★ 触发点在 `open_table_from_share()`（`sql/table.cc:3016`）：`if (share->partition_info_str_len && outparam->file && unpack_partition_info(...))`。**分区表每次打开表都要 parse 一遍分区子句**——这是分区多时 DDL/DML 开销的一个来源。

**★ "无条件全开"其实是两层**：

```cpp
/* ① server 层（sql/sql_partition.cc:1127） */
static bool set_up_partition_bitmaps(partition_info *part_info) {
  if (part_info->init_partition_bitmap(&part_info->read_partitions, &part_info->table->mem_root)) return true;
  if (part_info->init_partition_bitmap(&part_info->lock_partitions, &part_info->table->mem_root)) return true;
  part_info->bitmaps_are_initialized = true;
  part_info->set_partition_bitmaps(nullptr);   /* ★ 传 nullptr = 无 PARTITION() 子句 → 全开 */
  return false;
}

/* sql/partition_info.cc:262 */
bool partition_info::set_partition_bitmaps(Table_ref *table_list) {
  if (table_list && table_list->partition_names && table_list->partition_names->elements) {
    if (set_read_partitions(table_list->partition_names)) return true;  /* PARTITION(p0,p1) */
  } else {
    bitmap_set_all(&read_partitions);          /* ★★ 所有 bit 置 1 */
  }
  bitmap_copy(&lock_partitions, &read_partitions);   /* ★ 初态 lock == read */
  return false;
}
```

```cpp
/* ② InnoDB 层（ha_innopart::open → open_table_parts）遍历 dd::Table::leaf_partitions() 逐个全开 */
```

> ★★ 所以"全开"= **server 侧位图全置 1** + **InnoDB 侧逐个打开所有 leaf 分区的 `dict_table_t`**。每个分区在 InnoDB 里是**各自独立的 `dict_table_t`**（`set_partition()` 里 `m_prebuilt->table = m_part_share->get_table_part(part_id)`）。

也就是说：**打开一张分区表，MySQL 会重新解析一遍 `CREATE TABLE` 的分区子句**。这个往返让 DD 表不需要知道 `partition_info` 的内部结构，代价是每次 share 重建要跑一遍 yacc（结果缓存在 `TABLE_SHARE`）。

#### 1.3 打开表：对象模型与"无条件全开"

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

### 二、运行期路由：行 → 分区、写、改删、读、裁剪

#### 2.1 分区上下文切换：`set_partition` / `update_partition`

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

#### 2.2 行 → 分区：`get_partition_id` 函数指针族

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

**★ 装配现场 `set_up_partition_func_pointers()`**（`sql/sql_partition.cc:1209`，★ 揭示 KEY/LINEAR 如何编码）：

```cpp
static void set_up_partition_func_pointers(partition_info *part_info) {
  if (part_info->is_sub_partitioned()) {
    part_info->get_partition_id = get_partition_id_with_sub;   /* ★ 恒定 */
    if (part_info->part_type == partition_type::RANGE) { ... get_part_partition_id = range[_col] ... }
    else /* LIST */                                        { ... get_part_partition_id = list[_col] ... }
    /* ★★ 子分区维只能是 HASH/KEY，绝不可能是 RANGE/LIST */
    part_info->get_subpartition_id =
        part_info->list_of_subpart_fields
          ? (part_info->linear_hash_ind ? get_partition_id_linear_key_sub
                                        : get_partition_id_key_sub)
          : (part_info->linear_hash_ind ? get_partition_id_linear_hash_sub
                                        : get_partition_id_hash_sub);
  } else {
    part_info->get_part_partition_id = nullptr;
    part_info->get_subpartition_id = nullptr;
    if (part_info->part_type == partition_type::RANGE)      { ... }
    else if (part_info->part_type == partition_type::LIST)  { ... }
    else /* HASH */ {
      part_info->get_partition_id =
          part_info->list_of_part_fields            /* ★★ KEY 的判据：list_of_part_fields */
            ? (part_info->linear_hash_ind ? get_partition_id_linear_key_nosub
                                          : get_partition_id_key_nosub)
            : (part_info->linear_hash_ind ? get_partition_id_linear_hash_nosub
                                          : get_partition_id_hash_nosub);
    }
  }
  /* ★ 字符集字段：包一层"归一化 + 回调 + 还原" */
  if (part_info->part_charset_field_array) {
    part_info->get_part_partition_id_charset = part_info->get_part_partition_id;
    part_info->get_part_partition_id = get_part_id_charset_func_part;
  }
  if (part_info->subpart_charset_field_array) {
    part_info->get_subpartition_id_charset = part_info->get_subpartition_id;
    part_info->get_subpartition_id = get_part_id_charset_func_subpart;
  }
}
```

> ⚠️ **不存在 `get_partition_id_range_sub` / `get_partition_id_list_sub`**——RANGE/LIST 只出现在 `get_part_partition_id`（主分区维）；子分区维恒为 HASH/KEY。全 16 个真名见下表：

| 真名 | 行 | 语义 |
|---|---|---|
| `get_partition_id_range` / `_range_col` | 3059 / 3026 | RANGE（非 COLUMNS 的**不是 static**，供裁剪/ALTER 复用） |
| `get_partition_id_list` / `_list_col` | 2841 / 2813 | LIST |
| `get_partition_id_hash_nosub` / `_key_nosub` / `_linear_hash_nosub` / `_linear_key_nosub` | 3229 / 3242 / 3235 / 3250 | 无子分区四变体 |
| `get_partition_id_with_sub` | 3258 | 有子分区的**总入口** |
| `get_partition_id_hash_sub` / `_key_sub` / `_linear_hash_sub` / `_linear_key_sub` | 3302 / 3316 / 3309 / 3325 | 子分区四变体 |
| `get_part_id_charset_func_part` / `_subpart` | 2783 / 2799 | 字符集包装 |

**★★ `get_partition_id_range()` 全文**（★ NULL 与无符号两个细节最易错）：

```cpp
int get_partition_id_range(partition_info *part_info, uint32 *part_id,
                           longlong *func_value) {
  longlong *range_array = part_info->range_int_array;
  uint max_partition = part_info->num_parts - 1;
  uint min_part_id = 0, max_part_id = max_partition, loc_part_id;
  longlong part_func_value;
  int error = part_val_int(part_info->part_expr, &part_func_value);
  bool unsigned_flag = part_info->part_expr->unsigned_flag;

  if (error) return HA_ERR_NO_PARTITION_FOUND;

  if (part_info->part_expr->null_value) {
    *part_id = 0;      /* ★★ RANGE：NULL 永远落在第一个分区（视为小于任何值） */
    return 0;
  }
  *func_value = part_func_value;
  if (unsigned_flag)
    part_func_value -= 0x8000000000000000ULL;   /* ★ 无符号做"符号位翻转"变回可比有序区间 */

  while (max_part_id > min_part_id) {           /* ★ 二分 */
    loc_part_id = (max_part_id + min_part_id) / 2;      /* ★ 下中位数 */
    if (range_array[loc_part_id] <= part_func_value)
      min_part_id = loc_part_id + 1;   /* 值 >= 上界 → 右半区 */
    else
      max_part_id = loc_part_id;       /* 值 <  上界 → 左半区（含 loc 本身，所以不 -1） */
  }
  loc_part_id = max_part_id;
  *part_id = (uint32)loc_part_id;
  if (loc_part_id == max_partition &&                 /* ★ 只有最后一个分区才判溢出 */
      part_func_value >= range_array[loc_part_id] &&
      !part_info->defined_max_value)                  /* ★ 声明了 MAXVALUE 则永不报错 */
    return HA_ERR_NO_PARTITION_FOUND;
  return 0;
}
```

> ★★ **`_col` 版本的中位数取法不同**：`(max + min + 1) >> 1`（上中位数）且 `max_part_id = loc_part_id - 1`——因为 `cmp_rec_and_tuple(...) >= 0` 用的是 `>` 语义。**两个版本不能互换**。
> ★ `LIST` 的 NULL 与 RANGE 相反——**必须显式写了 `VALUES IN (NULL)` 才有 `has_null_part_id`**，否则直接 `HA_ERR_NO_PARTITION_FOUND`。

**★ KEY 分区的 collation 哈希：三层"偷梁换柱"**：

```cpp
/* ① 包装层（sql/sql_partition.cc:2783）：不算哈希，只换字段指针 */
static int get_part_id_charset_func_part(partition_info *part_info, uint32 *part_id,
                                         longlong *func_value) {
  copy_to_part_field_buffers(part_info->part_charset_field_array,
                             part_info->part_field_buffers,
                             part_info->restore_part_field_ptrs);
  res = part_info->get_part_partition_id_charset(part_info, part_id, func_value);
  restore_part_field_pointers(part_info->part_charset_field_array,
                              part_info->restore_part_field_ptrs);
  return res;
}

/* ② 归一化：my_strnxfrm 按字段 collation 生成排序键 */
      if (field->type() == MYSQL_TYPE_VARCHAR) {
        uint len_bytes = field->get_length_bytes();
        my_strnxfrm(cs, field_buf + len_bytes, max_len, field->data_ptr(), data_len);
        if (len_bytes == 1) *field_buf = (uchar)data_len;
        else                int2store(field_buf, data_len);
      } else {
        my_strnxfrm(cs, field_buf, max_len, field->field_ptr(), max_len);
      }
      field->set_field_ptr(field_buf);   /* ★★ 字段指针改指向归一化缓冲 */

/* ③ 真正算哈希（Partition_helper::ph_calculate_key_hash_value, partition_handler.cc:801） */
uint32 Partition_helper::ph_calculate_key_hash_value(Field **field_array) {
  ulong nr1 = 1, nr2 = 4;
  bool use_51_hash = (*field_array)->table->part_info->key_algorithm ==
                     enum_key_algorithm::KEY_ALGORITHM_51;
  do {
    Field *field = *field_array;
    if (use_51_hash) {
      switch (field->real_type()) {
        case MYSQL_TYPE_TINY: case MYSQL_TYPE_SHORT: case MYSQL_TYPE_LONG: ... {
          /* Force this to my_hash_sort_bin, which was used in 5.1! */
          my_charset_bin.coll->hash_sort(&my_charset_bin, field->field_ptr(),
                                         len, &tmp1, &tmp2);
          continue;
        }
        case MYSQL_TYPE_ENUM: case MYSQL_TYPE_SET: {
          /* ENUM/SET uses my_hash_sort_simple in 5.1 (i.e. my_charset_latin1) */
          my_charset_latin1.coll->hash_sort(&my_charset_latin1, field->field_ptr(),
                                            len, &tmp1, &tmp2);
          continue;
        }
        ...
      }
    }
    field->hash(&nr1, &nr2);   /* ★★ 8.0 默认（KEY_ALGORITHM_55）：走字段自己的 collation */
  } while (*(++field_array));
  return (uint32)nr1;
}
```

> ★★ **"按 collation 计算哈希"的落点是 `field->hash()`**（内部走 `field->charset()->coll->hash_sort`）。而 5.1 兼容模式下**数值/时间/ENUM/SET 强制用 binary/latin1 哈希**——这就是 `ALGORITHM=1` 与 `ALGORITHM=2` 分区结果不同的根源，也是升级必须谨慎的原因。
> ★ `copy_to_part_field_buffers` 只处理 VARCHAR / 非 binary collation，且 **NULL 字段跳过**（保持原指针）。

#### 2.3 分区裁剪：把 WHERE 变成分区集合

> **分工（两边不重复）**：裁剪**算法**的第一主语是优化器（它住在 `sql/range_optimizer/`，复用 `get_mm_tree`），只在 [`07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md) 展开——假索引构造、`SEL_ARG` 遍历、端点二分、子分区笛卡尔积。
>
> 本篇只留**分区表使用者视角**：两个时点、能否裁的三条件、结果落在哪两份位图、EXPLAIN 怎么看、失效清单。

一句话原理：裁剪器把分区列（+ 子分区列）拼成一个**虚拟索引**喂给 range 优化器的 `get_mm_tree`，区间代数（AND/OR/IN 归约、求交求并、min/max key 打包）全部免费复用——**这个复用既是裁剪能力的来源，也是所有失效场景的来源**。

算法细节——假索引的构造、`SEL_ARG` 树的递归遍历、各分区类型的端点二分、子分区的笛卡尔积——属于优化器视角，见 12 篇。下面只讲**分区表使用者需要知道的三件事**：什么时候裁、什么条件下能裁、裁完落在哪。

#### 两个时点

`prune_partitions(thd, table, query_block, cond)` 没有 `read_partitions` 参数，结果直接写回 `table->part_info->read_partitions`。它跑两次：

1. **prepare 期**：未加锁，子查询/存储程序还不能求值，用来确定 `lock_partitions`；若全部分区被裁掉且非外连接内表 → `set_empty_query()`，语句根本不执行。
2. **optimize 期**：已加锁，产出最终的 `read_partitions`。

`is_pruning_completed` 决定还需不需要第三次（条件非常量且在 prepare 期时保留 `false`）。**执行期没有 per-row 裁剪**——优化器不会在扫每一行时重算分区集合；唯一的逐行路径是 INSERT 的 `can_prune_insert`。

> **别和另一条执行期路径混淆**：`get_partition_set()` 会在索引等值查找时按当前 ref 键值定位分区。它属于**执行期**、但粒度是 per-lookup 而非 per-row，且**不走 `prune_partitions`**，另有一整套失效模式（绑定标记残留会跨语句污染、`FLUSH TABLES` 可自愈）。按第一主语归 [`../index/partitioning.md`](../index/partitioning.md)「判例二」。

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

**★ 裁剪结果如何落回两份位图**（`sql/range_optimizer/partition_pruning.cc:423`，⚠️ 真名不在 `opt_range.cc`——8.0 已把 range optimizer 拆到 `sql/range_optimizer/`）：

```cpp
bitmap_intersect(&read_partitions, &lock_partitions);   // 必须是锁集的子集
if (!locked && !partition_key_modified(table, table->write_set))
  bitmap_copy(&lock_partitions, &read_partitions);      // 反向收缩锁集
if (bitmap_is_clear_all(&read_partitions)) table->all_partitions_pruned_away = true;
```

> ★★ **裁剪不只写 `read_partitions`，还会反向收缩 `lock_partitions`**——条件为"尚未锁表 **且** 未修改分区键"。所以 `SELECT` 能同时缩小锁范围，而 `UPDATE`/`INSERT` 因为要写，锁集不能缩。

**★ 裁剪入口与兜底**（⚠️ `mark_all_partitions_as_used()` 不在 `sql_partition.cc`，在 `partition_pruning.cc:1038`，且**函数体只有 1 行**）：

```cpp
static void mark_all_partitions_as_used(partition_info *part_info) {
  bitmap_set_all(&(part_info->read_partitions));   /* ★ 就这一行 */
}
```

> ⚠️ **裁剪入口真名**：不是 `Partition_pruning_module` 类（不存在），而是自由函数 `prune_partitions()`（`sql/range_optimizer/partition_pruning.cc:251`）+ 纯 C 结构体 `PART_PRUNE_PARAM`。

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

#### 2.4 写路径：`Partition_helper`

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

**★ `ph_write_row()` 全文**（`sql/partitioning/partition_handler.cc:452`）：

```cpp
int Partition_helper::ph_write_row(uchar *buf) {
  uint32 part_id;
  int error;
  longlong func_value;
  bool have_auto_increment = m_table->next_number_field && buf == m_table->record[0];
  THD *thd = get_thd();
  sql_mode_t saved_sql_mode = thd->variables.sql_mode;
  ...
  if (have_auto_increment) {
    error = m_handler->update_auto_increment();    /* ① 先定 auto_inc —— 否则算出的分区可能不对 */
    if (error) return error;
    if (m_table->next_number_field->val_int() == 0) {
      m_table->autoinc_field_has_explicit_non_null_value = true;
      thd->variables.sql_mode |= MODE_NO_AUTO_VALUE_ON_ZERO;   /* 禁止二次生成 */
    }
  }

  error = m_part_info->get_partition_id(m_part_info, &part_id, &func_value);  /* ② 算分区号 */
  if (unlikely(error)) {
    m_part_info->err_value = func_value;   /* 供 my_error(ER_NO_PARTITION_FOR_GIVEN_VALUE) 报出 */
    goto exit;
  }
  if (!m_part_info->is_partition_locked(part_id)) {   /* ③ 一致性闸门 */
    error = HA_ERR_NOT_IN_LOCK_PARTITIONS;
    goto exit;
  }
  m_last_part = part_id;                              /* ④ 上下文切换（server 侧记录） */
  error = write_row_in_part(part_id, buf);            /* ⑤ 下发 SE（InnoDB 内部再 set_partition） */
  if (have_auto_increment && !m_table->s->next_number_keypart) {
    set_auto_increment_if_higher();                   /* ⑥ 回写共享 next_auto_inc_val */
  }
exit:
  thd->variables.sql_mode = saved_sql_mode;
  ...
  return error;
}
```

> ★★ 第 ③ 步 `is_partition_locked(part_id)` 常被忽略——**目标分区必须已在 `lock_partitions` 里**，否则 `HA_ERR_NOT_IN_LOCK_PARTITIONS`。这是裁剪缩小锁集后写操作会失败的机制。

**★★ `ph_update_row()`：跨分区移动与一致性校验**（★ 最值得逐字读的一个函数）：

```cpp
int Partition_helper::ph_update_row(const uchar *old_data, uchar *new_data) {
  uint32 new_part_id, old_part_id;
  longlong func_value;
  m_err_rec = nullptr;

  assert(bitmap_is_subset(&m_part_info->full_part_field_set, m_table->read_set));
  /* ① 同时算出 old / new 两个分区号（内部把字段指针临时指向 old_data 再还原） */
  if ((error = get_parts_for_update(old_data, new_data, m_table->record[0],
                                    m_part_info, &old_part_id, &new_part_id,
                                    &func_value)))
    return error;
  if (!bitmap_is_set(&(m_part_info->lock_partitions), new_part_id)) {
    error = HA_ERR_NOT_IN_LOCK_PARTITIONS;
    return error;
  }

  /*
    The protocol for updating a row is:
    1) position the handler (cursor) on the row to be updated,
       either through the last read row (rnd or index) or by rnd_pos.
    2) call update_row with both old and new full records as arguments.

    This means that m_last_part should already be set to actual partition
    where the row was read from. And if that is not the same as the
    calculated part_id we found a misplaced row, we return an error to
    notify the user that something is broken in the row distribution
    between partitions! Since we don't check all rows on read, we return an
    error instead of correcting m_last_part, to make the user aware of the
    problem!
    ...
  */
  if (old_part_id != m_last_part) {          /* ②★ 一致性校验：算出的 old 分区 ≠ 游标所在分区 */
    m_err_rec = old_data;
    return HA_ERR_ROW_IN_WRONG_PARTITION;    /* ★★ 行放错了分区 —— 报错而非自动纠正 */
  }

  m_last_part = new_part_id;
  if (new_part_id == old_part_id) {          /* ③ 同分区：原地 update */
    error = update_row_in_part(new_part_id, old_data, new_data);
  } else {
    /* ★★ 跨分区移动 = 先 insert 到新分区，再 delete 旧分区 */
    Field *saved_next_number_field = m_table->next_number_field;
    m_table->next_number_field = nullptr;    /* 禁止移动过程中生成 auto_inc */
    error = write_row_in_part(new_part_id, new_data);
    m_table->next_number_field = saved_next_number_field;
    if (!error) {
      error = delete_row_in_part(old_part_id, old_data);
    }
  }
  if (m_table->found_next_number_field && ... ) set_auto_increment_if_higher();
  return error;
}
```

> ★★★ **为什么报错而不自动纠正**：源码注释说得很直白——读路径并不逐行校验分区归属，若这里默默改 `m_last_part` 把行删掉，用户永远不会知道数据分布已经坏了。**"报错而非纠正"是一个刻意的可观测性设计**。
> ★ 配套：`get_parts_for_update()`（`sql/sql_partition.cc:315`）用 `set_field_ptr()` 临时把分区字段指向 `old_data`，算完再还原到 `record[0]`——所以 **`get_partition_id` 被调用了两次**（一次 old 一次 new）。

**★ `ph_delete_row()` 同构**（判据同样是 `part_id != m_last_part` → `HA_ERR_ROW_IN_WRONG_PARTITION`）。

**★ 引擎侧真正的上下文切换 `ha_innopart::set_partition()`**（⚠️ **`Partition_handler` / `Partition_helper` 里都没有 `set_partition`**）：

```cpp
void ha_innopart::set_partition(uint part_id) {
  if (part_id >= m_tot_parts) { ut_d(ut_error); ut_o(return); }
  if (m_pcur_parts != nullptr) {
    m_prebuilt->pcur = &m_pcur_parts[m_pcur_map[part_id]];       /* ① 切二级索引游标 */
  }
  if (m_clust_pcur_parts != nullptr) {
    m_prebuilt->clust_pcur = &m_clust_pcur_parts[m_pcur_map[part_id]];  /* ② 切聚簇游标 */
  }
  const auto &part{m_parts[part_id]};
  m_prebuilt->ins_node = part.m_ins_node;      /* ③ 恢复该分区的 insert/update 节点 */
  m_prebuilt->upd_node = part.m_upd_node;
  /* For unordered scan and table scan, use blob_heap from first
     partition as we need exactly one blob. */
  m_prebuilt->blob_heap = m_parts[m_ordered ? part_id : 0].m_blob_heap;   /* ④ ★ */
  m_prebuilt->trx_id = part.m_trx_id;
  m_prebuilt->row_read_type = part.m_row_read_type;
  m_prebuilt->new_rec_lock = part.m_new_rec_lock;
  m_prebuilt->sql_stat_start = m_sql_stat_start_parts.test(part_id);
  m_prebuilt->table = m_part_share->get_table_part(part_id);   /* ⑤★ 切 dict_table_t */
  m_prebuilt->index = innopart_get_index(part_id, active_index);  /* ⑥ 切 dict_index_t */
}
```

> ★★ **`m_prebuilt->table = get_table_part(part_id)` 是理解 InnoDB 原生分区的关键**——每个分区在 InnoDB 里是**各自独立的 `dict_table_t`**（有自己的 space_id / 索引树），切换分区就是把 `m_prebuilt` 的游标、节点、table/index 指针整体换一套。
> ★ ④ 的注释解释了 blob_heap 的规则：**有序扫用本分区的，无序/全表扫统一用第 0 个**（因为只需要一份 blob）。

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

#### 2.5 改 / 删路径：跨分区移动与一致性校验

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

#### 2.6 读路径：串行扫 + 优先队列归并

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

**★ 全表扫真身：串行，不是归并**（`ph_rnd_next()`，`partition_handler.cc:1535`）：

```cpp
int Partition_helper::ph_rnd_next(uchar *buf) {
  int result = HA_ERR_END_OF_FILE;
  uint part_id = m_part_spec.start_part;

  if (NO_CURRENT_PART_ID == part_id) goto end;   /* 裁剪后无分区 → 空结果 */
  assert(m_scan_value == 1);

  while (true) {
    result = rnd_next_in_part(part_id, buf);
    if (!result) {
      m_last_part = part_id;             /* ★ 记住分区，供 position/update/delete 校验 */
      m_part_spec.start_part = part_id;
      return 0;
    }
    if (result == HA_ERR_RECORD_DELETED) continue;   // Probably MyISAM
    if (result != HA_ERR_END_OF_FILE) goto end_dont_reset_start_part;

    /* End current partition */
    if ((result = rnd_end_in_part(part_id, true))) break;      /* ① 当前分区 EOF → 收尾 */

    /* Shift to next partition */
    part_id = m_part_info->get_next_used_partition(part_id);   /* ② ★ 位图里下一个被读分区 */
    if (part_id >= m_tot_parts) { result = HA_ERR_END_OF_FILE; break; }
    m_last_part = part_id;
    m_part_spec.start_part = part_id;
    if ((result = rnd_init_in_part(part_id, true))) break;     /* ③ 初始化下一个继续扫 */
  }
end:
  m_part_spec.start_part = NO_CURRENT_PART_ID;
end_dont_reset_start_part:
  return result;
}
```

> ★★ **关键纠正：全表扫不走优先队列**——它是"一个分区扫完 → `rnd_end` → 切下一个 → `rnd_init`"的**串行**过程。优先队列**只用于有序索引扫**。
> ★ `ph_rnd_init()` 里 `scan == true` 时只 `rnd_init_in_part` 第一个分区；`scan == false`（`rnd_pos` 场景）才把所有被读分区都初始化。

**★ `ref` 的编码含分区号**（`ph_position()`，★ 这是"分区表 rowid 更长"的原因）：

```cpp
  if (m_ref_usage != REF_NOT_USED && m_ordered_scan_ongoing) {
    memcpy(m_handler->ref, m_queue->top(), m_handler->ref_length);  /* 有序归并：ref 已在 PQ 里 */
  } else {
    int2store(m_handler->ref, m_last_part);                          /* ★★ ref = [2B 分区号][rowid] */
    position_in_last_part(m_handler->ref + PARTITION_BYTES_IN_POS, record);
  }
```

> ★ `PARTITION_BYTES_IN_POS = 2`（`partition_handler.h:61`）——分区表的 `ref_length` 比普通表多 2 字节。

**★★ 优先队列的真名与结构**（⚠️ **不是 `QUEUE`，是 `Priority_queue`**）：

```cpp
/* sql/partitioning/partition_handler.h:391 */
class Partition_helper {
  typedef Priority_queue<uchar *, std::vector<uchar *>, Key_rec_less> Prio_queue;
  ...
  uchar *m_ordered_rec_buffer;   /* ★ 每被读分区一块：[前缀][rec] */
  Prio_queue *m_queue;
  uint m_rec_offset;             /* ★ 2（有聚簇PK）或 ref_length（需按 rowid 二级排序） */
  uint m_top_entry;
  KEY *m_curr_key_info[3];       /* [0]=当前索引 [1]=聚簇PK（若有） [2]=NULL */
  enum enum_using_ref { REF_NOT_USED, REF_STORED_IN_PQ, REF_USED_FOR_SORT } m_ref_usage;
  MY_BITMAP m_key_not_found_partitions;   /* ★ KEY_NOT_FOUND 的分区要补进队列 */
  bool m_key_not_found;
};

/* 比较器（:356） */
struct Key_rec_less {
  bool operator()(uchar *first, uchar *second) {
    const int cmpval = (*m_fun)(m_keys, first + m_rec_offset, second + m_rec_offset);
    /* ★ 跳过分区号/ref 前缀，只比较记录体 */
    return m_max_at_top ? cmpval < 0 : cmpval > 0;
    /* ★ m_max_at_top=false → 最小堆（正序）；=true → 最大堆（倒序 prev） */
  }
};
```

**★ 建队：`init_record_priority_queue()`**（`partition_handler.cc:1656`）：

```cpp
int Partition_helper::init_record_priority_queue() {
  uint used_parts = m_part_info->num_partitions_used();
  if (!m_queue) {
    m_queue = new (std::nothrow) Prio_queue(Key_rec_less(m_curr_key_info));
    if (!m_queue) return HA_ERR_OUT_OF_MEM;
  }
  if (!m_ordered_rec_buffer) {
    if (m_pkey_is_clustered && m_table->s->primary_key != MAX_KEY) {
      m_rec_offset = PARTITION_BYTES_IN_POS;    /* = 2：前缀只放分区号 */
      m_ref_usage = REF_NOT_USED;               /* 聚簇 PK 已能做二级排序，无需 rowid */
    } else {
      m_rec_offset = m_handler->ref_length;     /* 无聚簇 PK：前缀放完整 ref */
      m_ref_usage = REF_STORED_IN_PQ;
    }
    alloc_len = used_parts * (m_rec_offset + m_rec_length);
    alloc_len += m_table->s->max_key_length;    /* ★ 尾部多一块当临时 key 缓冲 */
    m_ordered_rec_buffer = static_cast<uchar *>(
        my_malloc(key_memory_partition_sort_buffer, alloc_len, MYF(MY_WME)));
    ...
    char *ptr = (char *)m_ordered_rec_buffer;
    for (i = m_part_info->get_first_used_partition(); i < MY_BIT_NONE;
         i = m_part_info->get_next_used_partition(i)) {
      int2store(ptr, i);                        /* ★★ 每块缓冲头部 2 字节写分区号 */
      ptr += m_rec_offset + m_rec_length;
    }
    m_start_key.key = (const uchar *)ptr;
    m_queue->m_rec_offset = m_rec_offset;
    if (m_queue->reserve(used_parts)) return HA_ERR_OUT_OF_MEM;
  }
  return init_record_priority_queue_for_parts(used_parts);   /* 留给 SE（InnoDB 分配自己的资源） */
}
```

> ★★ **`m_rec_offset` 的两种取值是理解"是否需要 rowid 二级排序"的关键**：有聚簇 PK 时 PK 本身就能区分等值键，只需 2 字节分区号；无聚簇 PK 时要按 rowid 排，前缀得放完整 `ref`。

**★ 填充与推进**（★ `HA_ERR_KEY_NOT_FOUND` 这个细节最容易漏）：

```cpp
/* handle_ordered_index_scan（填充，:2431）★ 节选 */
    switch (m_index_scan_type) {
      case PARTITION_INDEX_READ:  error = index_read_map_in_part(i, read_buf, ...); break;
      case PARTITION_INDEX_FIRST: error = index_first_in_part(i, read_buf); break;
      case PARTITION_INDEX_LAST:  error = index_last_in_part(i, read_buf); break;
      case PARTITION_READ_RANGE:  error = read_range_first_in_part(...); break;
    }
    /* ★ ICP 下推时必须读到 record[0]，之后再 memcpy 回缓冲 */
    if (m_handler->pushed_idx_cond) memcpy(rec_buf_ptr, read_buf, m_rec_length);
    if (!error) {
      found = true;
      if (m_ref_usage != REF_NOT_USED) {
        m_last_part = i;
        position_in_last_part(part_rec_buf_ptr + PARTITION_BYTES_IN_POS, rec_buf_ptr);
      }
      parts.push_back(part_rec_buf_ptr);        /* ★ 每个分区读到的首行入候选 */
    } else if (error != HA_ERR_KEY_NOT_FOUND && error != HA_ERR_END_OF_FILE) {
      return error;
    } else if (error == HA_ERR_KEY_NOT_FOUND) {
      bitmap_set_bit(&m_key_not_found_partitions, i);   /* ★★ 游标已定位，后续 next 要带上 */
      m_key_not_found = true;
      saved_error = error;
    }
  }
  if (found) {
    m_queue->m_max_at_top = m_reverse_order;
    m_queue->m_keys = m_curr_key_info;
    m_queue->assign(parts);          /* ★★ 一次性建堆 O(n)，而非逐个 push 的 O(n log n) */
    return_top_record(buf);
    return 0;
  }
```

```cpp
/* handle_ordered_next（推进，:2660）★ 节选 */
  error = index_next_in_part(part_id, read_buf);   /* ★ 只从"刚返回行的那个分区"取下一行 */
  if (error) {
    if (error == HA_ERR_END_OF_FILE) {
      if (!m_queue->empty()) m_queue->pop();       /* 该分区扫完 → 出堆 */
      if (m_queue->empty()) { m_top_entry = NO_CURRENT_PART_ID; }
      else { return_top_record(buf); error = 0; }
    }
    return error;
  }
  ...
  m_queue->update_top();      /* ★★ 原地 sift-down，只调整堆顶 O(log n) */
  return_top_record(buf);
```

> ★★★ **`HA_ERR_KEY_NOT_FOUND` 的分区不能丢**：`index_read_map` 返回这个错误时游标**已经定位**在"下一个大于 key 的位置"，只是当前没有精确匹配。所以这些分区要记进 `m_key_not_found_partitions`，在后续 `index_next` 时由 `handle_ordered_index_scan_key_not_found()` **补进队列**——否则会漏行。
> ★ `return_top_record()` 里 `copy_cached_row()` 是虚函数——InnoDB 的实现**只拷贝 read_set 标记的列**，避免污染未读列。

**★ 归并的必要性判定**（`partition_scan_set_up`，`:2219`）：

```cpp
  if (m_part_spec.start_part > m_part_spec.end_part) return HA_ERR_END_OF_FILE;
  if (m_part_spec.start_part == m_part_spec.end_part) {
    m_ordered_scan_ongoing = false;   /* ★ 单分区 → 无需归并 */
  } else {
    uint start_part = m_part_info->get_first_used_partition();
    if (start_part == MY_BIT_NONE) return HA_ERR_END_OF_FILE;
    if (start_part > m_part_spec.start_part) m_part_spec.start_part = start_part;
    /* ★ 与 read_partitions（静态裁剪结果）取交集起点 */
    m_ordered_scan_ongoing = m_ordered;   /* ★ 多分区 + 要求有序 → 开归并 */
  }
```

> ★ 所以**归并只在"多分区 + 要求有序"时开启**。这也是为什么 `ORDER BY pk LIMIT 10` 在分区表上可能比单表慢——多了 k 路堆维护的常数开销。

### 三、分区级 DDL：结构变更 / TRUNCATE / EXCHANGE / 维护

8.0 **已不存在** `mysql_change_partitions()` / `alter_table_partition()`（5.x 的名字，全仓 0 命中）。分区管理分两条路。

#### 3.1 结构变更：ADD / DROP / COALESCE / REORGANIZE / REBUILD

**① 结构变更**走 `Sql_cmd_alter_table` → `mysql_alter_table` → `prep_alter_part_table`：

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

#### 3.2 分区级维护语句的入口

**② 分区级维护**走各自的 `Sql_cmd_*`：

| 语句 | 入口 |
|---|---|
| TRUNCATE PARTITION | `Sql_cmd_alter_table_truncate_partition::execute` → `Partition_handler::truncate_partition` |
| ANALYZE / CHECK / OPTIMIZE / REPAIR PARTITION | `sql_partition_admin.cc` 各自的 `execute`，委派对应 `Sql_cmd_*`，设 `ALTER_ADMIN_PARTITION` flag |
| EXCHANGE PARTITION | `Sql_cmd_alter_table_exchange_partition::execute` → `exchange_partition` |
| DISCARD / IMPORT PARTITION TABLESPACE | `Sql_cmd_discard_import_tablespace::execute` → `mysql_discard_or_import_tablespace` |

**★ 六个 `Sql_cmd` 的完整类声明**（`sql/sql_partition_admin.h:34-131`）：

```cpp
class Sql_cmd_alter_table_exchange_partition : public Sql_cmd_common_alter_table {
 public:
  bool execute(THD *thd) override;
 private:
  bool exchange_partition(THD *thd, Table_ref *, Alter_info *);
};

class Sql_cmd_alter_table_analyze_partition final : public Sql_cmd_analyze_table {
 public:
  bool execute(THD *thd) override;
  /* Override SQLCOM_ANALYZE, since it is an ALTER command */
  enum_sql_command sql_command_code() const override { return SQLCOM_ALTER_TABLE; }
};
/* CHECK / OPTIMIZE / REPAIR 三个同构：分别继承 Sql_cmd_check_table / _optimize_table / _repair_table */
class Sql_cmd_alter_table_truncate_partition final : public Sql_cmd_ddl_table {
 public:
  bool execute(THD *thd) override;
  enum_sql_command sql_command_code() const override { return SQLCOM_ALTER_TABLE; }
};
```

> ★★ **ANALYZE / CHECK / OPTIMIZE / REPAIR PARTITION 的 `execute()` 只有三行**——直接复用非分区版 admin 命令，靠 `ALTER_ADMIN_PARTITION` 标记做分区裁剪：
> ```cpp
> bool Sql_cmd_alter_table_analyze_partition::execute(THD *thd) {
>   assert(m_alter_info->flags & Alter_info::ALTER_ADMIN_PARTITION);
>   return Sql_cmd_analyze_table::execute(thd);
> }
> ```
> ⚠️ **不存在 `Sql_cmd_alter_table_add/drop/reorganize_partition`**——ADD/DROP/REORGANIZE 统一走 `mysql_alter_table()` + INPLACE 分区 API。**只有 6 个独立 Sql_cmd**。

**★ EXCHANGE PARTITION 的权限与校验**：

```cpp
  ulong priv_needed = ALTER_ACL | DROP_ACL | INSERT_ACL | CREATE_ACL;   /* ★ 四项权限 */
```
```cpp
static bool check_exchange_partition(TABLE *table, TABLE *part_table) {
  if (!part_table->part_info) { my_error(ER_PARTITION_MGMT_ON_NONPARTITIONED, MYF(0)); return true; }
  if (table->part_info) { my_error(ER_PARTITION_EXCHANGE_PART_TABLE, MYF(0), ...); return true; }
  if (!(part_table->file->ht->partition_flags() & HA_CAN_EXCHANGE_PARTITION)) { ... }
  if (table->file->ht != part_table->part_info->default_engine_type) {
    my_error(ER_MIX_HANDLER_ERROR, MYF(0)); return true;      /* ★ 引擎必须相同 */
  }
  if (table->s->tmp_table != NO_TMP_TABLE) { ... }            /* ★ 临时表不行 */
  if (table->s->foreign_keys || table->s->foreign_key_parents) { ... }  /* ★ 不能有外键 */
}
```

> ★ `compare_table_with_partition()` 逐个比对 `ROW_FORMAT` / `AVG_ROW_LENGTH` / `TABLE OPTION` / `CHARACTER SET` / 分区选项，任一不同报 `ER_PARTITION_EXCHANGE_DIFFERENT_OPTION` 或 `ER_TABLES_DIFFERENT_METADATA`。

**★★ EXCHANGE 的 MDL：注释与代码不一致**（★ 这是源码里的一处已知矛盾）：

```cpp
/* 注释（sql_partition_admin.cc:299）写着：
     4) Upgrade to exclusive mdl for both tables
   但代码里根本没调 upgrade_shared_lock！ */
  table_list->mdl_request.set_type(MDL_SHARED_NO_WRITE);   /* ★ 只请求禁写锁 */
  ...
  if (wait_while_table_is_used(thd, swap_table, HA_EXTRA_PREPARE_FOR_RENAME) ||
      wait_while_table_is_used(thd, part_table, HA_EXTRA_PREPARE_FOR_RENAME))
    return true;                                           /* ★★ 靠"驱赶"而非"升级" */
```

> ★★ 实际机制是：**拿 `MDL_SHARED_NO_WRITE`（禁写但允许读）** → 校验 → `wait_while_table_is_used()` 等所有使用者退出 → `close_all_tables_for_name()` 强制关闭 → 再动 DD。**不是升级到 X 锁**。源码注释里那句 "Upgrade to exclusive mdl" 是遗留描述。

**★ TRUNCATE PARTITION 直接要 X 锁**（与 EXCHANGE 不同）：

```cpp
bool Sql_cmd_alter_table_truncate_partition::execute(THD *thd) {
  first_table->set_lock({TL_WRITE, THR_DEFAULT});
  first_table->mdl_request.set_type(MDL_EXCLUSIVE);   /* ★★ 直接 X，不走 SU→X 升级 */
  if (check_one_table_access(thd, DROP_ACL, first_table)) return true;   /* ★ 需 DROP 权限 */
  ...
  /* ★ 只把待 TRUNCATE 的分区置入 read_partitions 位图，SE 靠位图判断范围 */
  first_table->partition_names = &m_alter_info->partition_names;
  if (first_table->table->part_info->set_partition_bitmaps(first_table)) return true;
```

**★ SQL 层的两道断言**（`Partition_handler::truncate_partition` / `exchange_partition`，★ 用断言区分锁强度要求）：

```cpp
int Partition_handler::truncate_partition(dd::Table *table_def) {
  handler *file = get_handler();
  assert(file->table_share->tmp_table != NO_TMP_TABLE ||
         file->m_lock_type == F_WRLCK);          /* ★★ truncate 必须已持有 F_WRLCK */
  file->mark_trx_read_write();
  return truncate_partition_low(table_def);
}

int Partition_handler::exchange_partition(uint part_id, dd::Table *part_table_def,
                                          dd::Table *swap_table_def) {
  handler *file = get_handler();
  assert(file->table_share->tmp_table != NO_TMP_TABLE ||
         file->m_lock_type != F_UNLCK);          /* ★ exchange 只要求"不是未加锁" */
  file->mark_trx_read_write();
  return exchange_partition_low(part_id, part_table_def, swap_table_def);
}
```

**★ TRUNCATE PARTITION 的 AUTOINC 语义**（★ ALL 与部分 TRUNCATE 不同）：

```cpp
int ha_innopart::truncate_partition_low(dd::Table *dd_table) {
  bool truncate_all = (m_part_info->num_partitions_used() == m_tot_parts);
  ...
  for (const auto dd_part : *dd_table->leaf_partitions()) {
    ...
    innobase_truncate<dd::Partition> truncator(thd, norm_name, table, dd_part,
                                               !truncate_all, truncate_all);
    ...
    if (!m_part_info->is_partition_used(part_num++)) continue;   /* ★ 裁剪：不在位图里就跳过 */
    error = truncator.exec();                                    /* ★ rename → delete → create */
  }
  if (table->found_next_number_field) {
    dd_set_autoinc(dd_table->se_private_data(),
                   (truncate_all ? 0 : autoinc + 1));   /* ★★ ALL 归 0；部分保留 max+1 */
  }
}
```

> ⚠️ **整表 TRUNCATE 走的是另一条路**：`ha_innopart::create()` 开头判断 `thd_sql_command(thd) == SQLCOM_TRUNCATE` → 转 `truncate_impl()`。所以 `TRUNCATE TABLE` 与 `TRUNCATE PARTITION ALL` 是**两个不同函数**。

**★ rename → delete → create 三步本体**（`ha_innodb.cc:14559`）：

```cpp
template <typename Table> int innobase_truncate<Table>::truncate() {
  if (m_file_per_table) {
    error = rename_tablespace();     /* ① 把 xxx.ibd 改名成临时名（崩溃可由 DDL log 回滚） */
  }
  ...
  error = innobase_basic_ddl::delete_impl(m_thd, m_name, m_dd_table, nullptr);  /* ② delete/drop */
  ...
  m_trx->in_truncate = true;
  error = innobase_basic_ddl::create_impl(m_thd, m_name, m_form, &m_create_info,
                                          m_dd_table, m_file_per_table, false, true,
                                          m_flags, m_flags2,
                                          inherit_metadata ? &m_dd_table->table() : nullptr);
  m_trx->in_truncate = false;        /* ③ 重建 .ibd */
  ...
}
```

> ★ 临时名由 `dict_mem_create_temporary_tablename()` 生成（形如 `db/table#P#p0#TMP`）；`fil_rename_tablespace()` 内部会写 `RENAME_SPACE_LOG`。
> ★ `innobase_truncate<dd::Partition>::load_fk()` **直接 return 0**——分区表不支持外键。

#### 3.3 TRUNCATE PARTITION：rename → delete → create

物理实现：

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

#### 3.4 EXCHANGE PARTITION：三次 rename + 元数据互换

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

#### 3.5 引擎侧的维护实现

分区级维护在引擎侧：

| 操作 | 实现 |
|---|---|
| OPTIMIZE | 直接返回 `HA_ADMIN_TRY_ALTER`，让 SQL 层改写成 `ALTER TABLE ... ENGINE=InnoDB` 重建 |
| ANALYZE | `info_low(flag, is_analyze=true)`：逐分区 `update_table_stats`（持久统计 `DICT_STATS_RECALC_PERSISTENT`，否则 `DICT_STATS_RECALC_TRANSIENT`） |
| CHECK | 逐分区 `ha_innobase::check()` + `check_misplaced_rows(i, false)`（仅 `T_MEDIUM`/`T_EXTEND`） |
| REPAIR | `check_misplaced_rows(i, true)`——**只把放错分区的行搬走，不修索引** |
| REBUILD | 走 inplace alter 的 `alter_parts` 路径 |
| DISCARD / IMPORT | `discard_or_import_tablespace`：按 `read_partitions` 逐分区调用，再 `set_dd_discard_attribute` 逐分区改 DD 的 discarded 状态与 tablespace state，并回写每个 `dd_index` 的 `DD_INDEX_ROOT` |

`check_misplaced_rows` 是个补丁式机制：扫描分区，把"按分区函数本不该在这里"的行搬到正确分区（REPAIR）或仅报告（CHECK）。这类行通常来自分区定义被 ALTER 改动后未重分布的历史遗留。

#### 3.6 MDL 与锁升级

- `prep_alter_part_table` 断言至少持有 `MDL_INTENTION_EXCLUSIVE`；
- TRUNCATE PARTITION 取 `MDL_EXCLUSIVE`，`LOCK TABLES` 模式下 `upgrade_shared_lock`，并用 `set_partition_bitmaps` 限定只处理目标分区（避免全表 `external_lock`）；
- DISCARD/IMPORT 同样 `MDL_EXCLUSIVE`；
- 引擎声明 `HA_TRUNCATE_PARTITION_PRECLOSE` 时（InnoDB 就是），SQL 层会先关闭表并构造临时 `TABLE`/`TABLE_SHARE` 再调用。

### 四、分区 DDL 的原子性与崩溃恢复

分区 DDL 涉及**多个物理文件**（N 个分区的表空间）与**两份元数据**（server 的 `dd::Partition` 与 InnoDB 的 `dict_table_t`），一次 TRUNCATE/DROP 中间被打断会留下半截状态。8.0 用两条机制兜住：**DDL log**（记录物理操作，可补做/回退）+ **原子 DDL 的 dd 事务**（元数据要么全改要么全不改）。

#### 4.1 各操作写了哪些 DDL log

| 操作 | 日志条目 | 写入点 |
|---|---|---|
| TRUNCATE PARTITION | `RENAME_SPACE_LOG` + `DROP_LOG` + `DELETE_SPACE_LOG` | rename 走 `innobase_truncate::rename_tablespace` → `fil_rename_tablespace`；删除走 `delete_impl` → `row_drop_table_for_mysql` |
| DROP PARTITION | `DROP_LOG` + `DELETE_SPACE_LOG`；**重名冲突时先** `RENAME_TABLE_LOG` + `RENAME_SPACE_LOG` | `alter_part_drop::try_commit` |
| ADD PARTITION | 需要改名时 `RENAME_TABLE_LOG` + `RENAME_SPACE_LOG`；新建表空间失败由 `DELETE_SPACE_LOG` 补偿 | `alter_part_add::try_commit` |

关键代码：

```cpp
// storage/innobase/row/row0mysql.cc  (row_drop_table_for_mysql)
  if (!is_temp) {
    log_ddl->write_drop_log(trx, table_id);          // DROP_LOG
  }
  ...
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);
```

```cpp
// storage/innobase/fil/fil0fil.cc  (Fil_shard::space_rename)
      /* Rename ddl log is for rollback, so we exchange
      old file name with new file name. */
      dberr_t err = log_ddl->write_rename_space_log(space_id, new_file_name, old_file_name);
```

注意最后这段注释——**rename 日志的 old/new 是故意互换的**，因为它记录的是"如何回滚这次改名"，不是"如何重做"。

#### 4.2 提交后：`post_ddl` 补做物理动作

DDL 事务提交后，server 回调 `handlerton->post_ddl`（InnoDB 侧 `innobase_post_ddl`）→ `Log_DDL::post_ddl` → `replay_by_thread_id` → 逐条 `replay()` → 成功后 `delete_by_ids()` 删除日志行。

```cpp
// storage/innobase/handler/ha_innodb.cc  (innobase_post_ddl)
  if (log_ddl != nullptr) {
    dberr_t err = log_ddl->post_ddl(thd);
```

也就是说：**物理动作（删文件、改名）是在元数据提交之后才真正做的**。DDL 返回成功时元数据已持久化，物理残留由 post_ddl 清理。

#### 4.3 崩溃恢复：`recover` → `replay_all`

启动时 `handlerton->post_recover`（`innobase_post_recover`，条件 `srv_force_recovery < SRV_FORCE_NO_TRX_UNDO`）→ `log_ddl->recover()` → `replay_all()`：

```cpp
// storage/innobase/log/log0ddl.cc  (Log_DDL::recover)
  thread_local_ddl_log_replay = true;
  s_in_recovery = true;
  dberr_t err = replay_all();
```

`replay_all` 按日志 id **升序**回放（保证"先改名后删除"这类顺序依赖成立），全部成功后才 `delete_by_ids`。单条按 `Log_Type` 分派到 `replay_delete_space_log` / `replay_rename_space_log` / `replay_drop_log` 等。

#### 4.4 未提交就崩溃：靠 dd 事务回滚

DDL log 行本身是用**DDL 自己的 trx** 写的（`trx_start_if_not_started(trx, true)` + `DDL_Log_Table`）。因此事务未提交就崩溃时，这些日志行随 trx 回滚一起消失——**不会被 replay**，物理残留由 trx 回滚清理。已提交但物理动作没做完的，才由 post_ddl 或下次启动的 replay 补做。

#### 4.5 ALGORITHM 判定：什么时候必须重建表

```cpp
// sql/sql_table.cc  (mysql_alter_table)
  if ((thd->variables.old_alter_table && ...) ||
      is_inplace_alter_impossible(table, create_info, alter_info) ||
      (partition_changed &&
       !(table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION) &&
       !new_part_info)) {
```

三个条件任一成立就走 COPY/rebuild。其中：

- `HA_USE_AUTO_PARTITION` 就在这个表达式里（NDB 这类"自动分区"引擎可以原地改；普通引擎不行）；
- `HA_INPLACE_CHANGE_PARTITION` **不出现在这个表达式**——它通过 `*new_part_info != nullptr` 间接生效：`prep_alter_part_table` 里若引擎声明支持，就把新 `part_info` 交出去（`*new_part_info = tab_part_info`）并 `table->invalidate_dict()` 强制重开表。

**是否需要搬迁数据**由 `alter_parts::need_copy` 决定：

```cpp
// storage/innobase/handler/handler0alter.cc  (alter_parts::need_copy)
    /* Basically, only DROP PARTITION, ADD PARTITION for RANGE/LIST
    partitions don't require copying data between partitions */
    if (ha_alter_info->handler_flags & Alter_inplace_info::ADD_PARTITION) {
      switch (ha_alter_info->modified_part_info->part_type) {
        case partition_type::RANGE:
        case partition_type::LIST:  return (false);
        default: break; } }
    return (!(ha_alter_info->handler_flags & (Alter_inplace_info::DROP_PARTITION)));
```

即：**只有 DROP PARTITION 与 RANGE/LIST 的 ADD PARTITION 不搬数据**；REORGANIZE、COALESCE、HASH/KEY 的 ADD PARTITION 都必须走 `copy_partitions`——因为取模基数或值区间变了，几乎所有行都要重新映射。

#### 4.6 各语句的 MDL

| 语句 | MDL | 位置 |
|---|---|---|
| TRUNCATE PARTITION | `MDL_EXCLUSIVE`；LOCK TABLES 模式下 `upgrade_shared_lock`，结束后降级 | `sql_partition_admin.cc` 的 `Sql_cmd_alter_table_truncate_partition::execute` |
| EXCHANGE PARTITION | 先 `MDL_SHARED_NO_WRITE` 校验，通过后 `lock_tables()` 取写锁 | `exchange_partition` |
| 结构变更 ALTER | 解析期 `MDL_SHARED_UPGRADABLE` → `mysql_alter_table` 内升级（inplace 升 X / SHARED 场景升 `MDL_SHARED_NO_WRITE` / prepare 后允许并发时降回）→ 提交前再升 | `sql_table.cc` |
| DISCARD / IMPORT | `MDL_EXCLUSIVE` | `mysql_discard_or_import_tablespace` |

引擎声明 `HA_TRUNCATE_PARTITION_PRECLOSE` 时（InnoDB 就是），SQL 层会走一段特殊流程：`close_all_tables_for_name` 关掉所有 TABLE → `init_tmp_table_share` + `open_table_from_share`（`db_stat=0`、dd::Table 传 nullptr）**造一个哑 TABLE**，把分区位图拷过去再调 `truncate_partition`。原因是 truncate 要重建表空间，不能让已打开的 handler 还持有旧分区对象。

### 五、统计、代价估算与优化器反馈

#### 5.1 统计：逐分区采集，向优化器"求和 + 取最大分区"

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

#### 5.2 代价估算：分区表上的 `records_in_range` 与 bad case

**估算方式**：对每个**未被裁剪掉的分区**各做一次 `btr_estimate_n_rows_in_range` 再求和：

```cpp
// storage/innobase/handler/ha_innopart.cc  (ha_innopart::records_in_range)
  part_id = m_part_info->get_first_used_partition();
  set_partition(part_id);
  ...
  n_rows = btr_estimate_n_rows_in_range(index, range_start, mode1, range_end, mode2);
  for (part_id = m_part_info->get_next_used_partition(part_id);
       part_id < m_tot_parts;
       part_id = m_part_info->get_next_used_partition(part_id)) {
    index = m_part_share->get_index(part_id, keynr);
    int64_t n = btr_estimate_n_rows_in_range(index, range_start, mode1, range_end, mode2);
    n_rows += n;
  }
  ...
  if (n_rows == 0) n_rows = 1;          // ★ 至少为 1
```

`scan_time` 与 `estimate_rows_upper_bound` 同样是逐分区累加（各分区 `stat_n_leaf_pages` 换算后相加）。

**`eq_range_index_dive_limit` 在分区表上的行为**：这个阈值判定的对象是**等值 range 的个数**（`eq_ranges_exceeds_limit`），与分区数无关。所以：

- 未超阈值 → **每个 range × 每个未剪枝分区**各一次 dive：1000 个分区 + 10 个 IN 值 = 10 000 次 B 树下潜；
- 超过阈值 → 一次都不 dive，改用 `records_per_key`（见下）估算。

**`rec_per_key` 只取最大分区**（这是分区表代价模型最粗糙的一处）：

```cpp
// ha_innopart.cc  (HA_STATUS_CONST 分支)
  if (ib_table->stat_n_rows > max_rows) { max_rows = ib_table->stat_n_rows; biggest_partition = i; }
  ...
  dict_index_t *index = innopart_get_index(biggest_partition, i);        // ★ 只用最大分区的索引
  const rec_per_key_t rec_per_key = innodb_rec_per_key(index, j, max_rows);
```

对照非分区表用的是全表 `stat_n_rows` 作分母。分区表用"最大分区的行数"作分母，后果如下：

| bad case | 代码依据 | 后果 |
|---|---|---|
| **空分区多 → 行数被抬高** | `n_rows == 0 → 1`；`info_low` 里无 `HA_STATUS_TIME` 时 `n_rows++` | 8192 分区只有 3 个有数据，全表 `stats.records` 仍 ≥ 1；`mean_rec_length = data_file_length / records` 被高估 → `scan_time` 偏大 |
| **`rec_per_key` 被低估 P 倍 → 错选索引** | 分母只取 `max_rows ≈ rows_total / P`；而若索引前缀含分区键，每个分区几乎保有全部取值（`n_diff_part ≈ n_diff_total`） | 该索引看起来区分度极好；走"超 dive 阈值"路径时 rows 被低估 P 倍 → 优化器倾向选它，join 顺序也可能选错 |
| **裁剪后不重算** | `info_low` 的 TODO 注释：*"If this is called after pruning, then we could also update the statistics according to the non-pruned partitions"* | 裁剪到 1 个分区后，`records_per_key` 仍是"最大分区"的值，不随裁剪收敛 |
| **dive 结果被系统性削顶** | `btr0cur.cc`：`n_rows > table_n_rows / 2 → n_rows = table_n_rows / 2` | 分区越碎，单分区 `table_n_rows` 越小，削顶越早发生 → 求和后总量低估，可能反过来让优化器放弃 range 改走全表扫描 |

**一句话结论**：分区越多，`records_in_range`（受削顶影响偏低）与 `scan_time`（线性叠加偏高）的**不对称越严重**，代价模型失真越大。这是"分区表执行计划突然变差"最常见的根因。

#### 5.3 持久统计在磁盘上长什么样

**一个反直觉但很重要的结论**：`mysql.innodb_table_stats` 里，**一行 = 一个分区**，`table_name` 是带分区后缀的完整分区名（`t#p#p0`）。

原因是这两张统计表的 key 直接取自 `table->name.m_name`，而 InnoDB 里每个分区就是一张独立的 `dict_table_t`：

```cpp
// storage/innobase/dict/dict0stats.cc  (dict_stats_save)
  table = dict_stats_snapshot_create(table_orig);
  dict_fs2utf8(table->name.m_name, db_utf8mb3, sizeof(db_utf8mb3),
               table_utf8mb3, sizeof(table_utf8mb3));
  pars_info_add_str_literal(pinfo, "database_name", db_utf8mb3);
  pars_info_add_str_literal(pinfo, "table_name",    table_utf8mb3);
  ... "DELETE FROM \"mysql/innodb_table_stats\" WHERE database_name=:database_name
       AND table_name=:table_name;  INSERT INTO ... VALUES(...)"
```

两张表的结构（由 InnoDB 在 DD 初始化时注册，见 `ha_innodb.cc`）：

| 表 | 主键 | 分区粒度 |
|---|---|---|
| `mysql.innodb_table_stats` | `(database_name, table_name)` | 每行一个分区，`table_name = t#p#pN` |
| `mysql.innodb_index_stats` | `(database_name, table_name, index_name, stat_name)` | 每行一个（分区 × 索引 × stat） |

于是 P 个分区的表在 `innodb_table_stats` 里有 P 行，在 `innodb_index_stats` 里有 P × 索引数 × stat 数行。

**`ANALYZE PARTITION` 的完整链路**：

```
Sql_cmd_analyze_table::execute → mysql_admin_table(..., &handler::ha_analyze, ...)
  → handler::ha_analyze → ha_innobase::analyze()
       → info_low(HA_STATUS_TIME|HA_STATUS_CONST|HA_STATUS_VARIABLE, is_analyze=true)
          → (虚派发) ha_innopart::info_low
             → set_altered_partitions()                按 ALTER_ADMIN_PARTITION 标定 used 分区
             → for (i = get_first_used_partition(); ...)
                  update_table_stats(ib_table, is_analyze)
                    → dict_stats_update(table, opt)
                        → DICT_STATS_RECALC_PERSISTENT → dict_stats_update_persistent + dict_stats_save
```

`opt` 的选择（`update_table_stats`）：持久统计开启且是 ANALYZE → `DICT_STATS_RECALC_PERSISTENT`（重算并写盘）；持久统计开启但不是 ANALYZE（如 `SHOW INDEX`）→ `DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY`；未开持久统计 → `DICT_STATS_RECALC_TRANSIENT`（只在内存，不落盘）。

**采样参数是"每分区"语义，不是"整表"**：

```cpp
// storage/innobase/dict/dict0stats.cc
#define N_SAMPLE_PAGES(index)                                    \
  static_cast<uint64_t>((index)->table->stats_sample_pages != 0   \
                            ? (index)->table->stats_sample_pages  \
                            : srv_stats_persistent_sample_pages)
```

`index->table` 就是该**分区**的 `dict_table_t`。而且注释明确："For each n-column prefix (for n=1..n_uniq) N_SAMPLE_PAGES(index) will be sampled"——即每个索引的每个 n-prefix 各采样 N 页。所以 `ANALYZE TABLE t` 在 P 个分区上的总采样量 ≈ **P × 索引数 × Σn_uniq × N** 页，不是整表 N 页。

**后台自动重算也是按分区**：`dict_stats_recalc_pool_add` 入池的是 `table->id`，而每个分区有独立的 `se_private_id`（分区表自身的 `dd::Table::se_private_id()` 是 INVALID），所以池中最多 P 条；10% 变更阈值（`row_update_statistics_if_needed`）也是**按分区各自计数**；后台线程一次只处理一个条目，对该分区单独 `dict_stats_update` + `dict_stats_save`。

**易混淆**：`mysql.table_stats` / `mysql.index_stats`（DD 层，给 I_S 做缓存）与 `mysql.innodb_table_stats` 不是一回事——前者整表一行，且 `can_persist_I_S_dynamic_statistics()` 对分区（`partition_name != nullptr`）**直接返回 false**，所以分区的动态统计从不落进 `mysql.table_stats`。

### 六、事务、锁、2PC 与死锁

#### 6.1 事务与 2PC

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

#### 6.2 锁粒度与死锁

**锁粒度就是 `dict_table_t`**，每个分区一份（`Ha_innopart_share::m_table_parts[]`）。分区表在 InnoDB 里没有"表级聚合对象"，所以"分区级表锁"就是该分区 `dict_table_t` 的锁。

两个容易误解的实现细节：

**（a）`external_lock` 只对分区 0 真正加锁**：

```cpp
// storage/innobase/handler/ha_innopart.cc  (ha_innopart::external_lock)
  m_prebuilt->table = m_part_share->get_table_part(0);
  error = ha_innobase::external_lock(thd, lock_type);     // ← 只有分区 0
  for (uint i = 0; i < m_tot_parts; i++) {
    dict_table_t *table = m_part_share->get_table_part(i);
    switch (table->quiesce) {                             // ← 其余分区只处理 quiesce
      case QUIESCE_START:  ... row_quiesce_table_start(table, m_prebuilt->trx);
      case QUIESCE_COMPLETE: ... row_quiesce_table_complete(table, m_prebuilt->trx);
```

也就是说：`LOCK TABLES` + 非自动提交场景下的 InnoDB 表锁（`row_lock_table`）只覆盖**分区 0**。

**（b）`lock_partitions` 位图根本不传给 InnoDB**。在 `storage/innobase` 全目录里搜 `lock_partitions`，唯一命中是错误码常量 `HA_ERR_NOT_IN_LOCK_PARTITIONS`——InnoDB 完全不知道"锁了哪些分区"这个概念，它只看"当前 `m_prebuilt->table` 指向哪个分区"。

**死锁形态**：锁管理器对分区无感知（`lock0lock.cc` 里搜 `partition` / `part_id` 零命中），等待图按 `lock_t`（表锁用 `dict_table_t`、行锁用 page/rec）工作。加锁顺序是"分区号升序"这一天然顺序（遍历一律走 `get_first_used_partition` / `get_next_used_partition`），但**没有专门的按分区号排序加锁函数**。

唯一可能形成跨分区环的形态是**跨分区行迁移的固定顺序**："先插新分区、再删旧分区"。T1 把行从 P0 迁到 P3（先持 P3 插入锁，再等 P0 删除锁），T2 反向从 P3 迁到 P0，两者即成环。

#### 6.3 `FLUSH TABLE ... FOR EXPORT`：分区表的 quiesce

`quiesce` 不是一种锁，而是**"把内存状态追平到磁盘"**——让 `.ibd` 文件处于可直接拷贝的一致状态。它是 `FLUSH TABLE t FOR EXPORT` 的实现，服务于"拷贝文件 → 另一实例 `IMPORT TABLESPACE`"这条运维路径。

**为什么需要它**：一张正在被写入的表，磁盘映像是不完整的——脏页还在 buffer pool、change buffer 里的二级索引变更还没合并、purge 线程还在跑。要让 `.ibd` 可以直接拷走，这三样都得追平。`row_quiesce_table_start` 正是在做这件事：

```cpp
// storage/innobase/row/row0quiesce.cc
void row_quiesce_table_start(dict_table_t *table, trx_t *trx) {
  ...
  if (trx_purge_state() != PURGE_STATE_DISABLED) {
    trx_purge_stop();                                    // ① 停 purge
  }
  for (ulint count = 0;
       ibuf_merge_space(table->space) != 0 && !trx_is_interrupted(trx);
       ++count) { ... }                                   // ② 合并 change buffer（循环到合并完）

  if (!trx_is_interrupted(trx)) {
    ...
    buf_LRU_flush_or_remove_pages(table->space, BUF_REMOVE_FLUSH_WRITE, trx);  // ③ 刷脏页
    ...
    row_quiesce_write_cfg(table, trx->mysql_thd);          // ④ 写 .cfg（表空间元数据）
    row_quiesce_write_cfp(table, trx->mysql_thd);          //    写 .cfp（加密信息）
  }
  row_quiesce_set_state(table, QUIESCE_COMPLETE, trx);     // ⑤ 状态推进
}
```

`.cfg` 记录表空间的元数据（页大小、索引 root page 等），供 `IMPORT TABLESPACE` 校验用。收尾时 `row_quiesce_table_complete` 会做四件事：等状态变为 `QUIESCE_COMPLETE`（被 kill 时每秒轮询等待，**不是直接返回**）→ 删掉 `.cfg` / `.cfp`（注释说明：不删的话用户 drop 数据库时删不掉目录）→ `trx_purge_run()` 恢复 purge → 状态回 `QUIESCE_NONE`。

状态机是 `QUIESCE_NONE → QUIESCE_START → QUIESCE_COMPLETE`，由 `row_quiesce_set_state` 推进。

**分区表上必须逐分区做**：quiesce 的作用单位是 `dict_table_t`（更准确地说是 `table->space`），而每个分区是独立的 `dict_table_t` + 独立 space。所以：

| 阶段 | 分区 0 | 其余分区 |
|---|---|---|
| `store_lock` | 基类 `ha_innobase::store_lock` 置 `QUIESCE_START` | `ha_innopart::store_lock` 循环 `i=1..` 逐个 `row_quiesce_set_state(table, QUIESCE_START, trx)` |
| `external_lock` | 基类加锁（见 6.2） | `ha_innopart::external_lock` 循环按 `table->quiesce` 分支：`QUIESCE_START` → `row_quiesce_table_start`；`QUIESCE_COMPLETE` → `row_quiesce_table_complete` |

注意这个不对称很有意思：**加锁只需一次**（MDL 已经拦住并发，`row_lock_table` 只锁分区 0），**刷盘必须 N 次**（每个分区有自己的 space 要 flush）。这也解释了为什么分区数上千时 `FLUSH TABLE ... FOR EXPORT` 会明显变慢——它要串行刷 N 个表空间。

### 七、复制与并行

#### 7.1 复制：binlog 里的分区信息

分区表的事务在 binlog 里会额外带上两个分区号，定义在 `Rows_event::Extra_row_info`（`libbinlogevents/include/rows_event.h`，type_code `PART=1`，每个值 2 字节）：

- `partition_id`：**目标分区**，在 `Rows_log_event` 构造时写入（`sql/log_event.cc` 的 `get_rpl_part_id`）。
- `source_partition_id`：**源分区**，只在 `THD::binlog_update_row` 里写：

```cpp
// sql/binlog.cc  (THD::binlog_update_row)
partition_info *part_info = table->part_info;
uint32 source_part_id = binary_log::Rows_event::Extra_row_info::UNDEFINED;
if (part_info) {
  uint32 new_part_id = ...UNDEFINED;
  longlong func_value = 0;
  get_parts_for_update(before_record, after_record, table->record[0],
                       part_info, &source_part_id, &new_part_id, &func_value);
}
...
if (part_info) {
  ev->m_extra_row_info.set_source_partition_id(source_part_id);
}
```

注意 `get_parts_for_update` **无条件计算 old/new 两个分区号**，所以写入条件是"表是分区表"，**不是"发生了跨分区移动"**——同分区 UPDATE 同样会写。

**从库完全不使用这两个值**：全仓只有 `mysqlbinlog` 的 `Rows_log_event::print_verbose_one_row` 会读它（打印 `### Extra row info for partitioning: source_partition: N target_partition: M`），`rpl_applier.cc` / `rpl_record.cc` / `rpl_rli_pdb.cc` 中均无引用。

一个容易被忽略的副作用：pending Rows event 合并时用 `compare_extra_row_info` **同时比较** `part_id` 与 `source_part_id`，两者任一不同就必须 flush 当前 event 另起一个。因此**分区表上的批量 DML 会产生更多、更小的 Rows event**。

#### 7.2 MTS：社区版没有"按分区并行"

分区信息**完全不参与**从库并行复制的依赖计算：

- DATABASE 模式按 db 名分桶，LOGICAL_CLOCK 模式按 `last_committed` / `sequence_number`（值来自 commit-order 或 writeset）；
- writeset 只做主键/唯一键的 64 位哈希历史比对，**分区号从未进入 writeset**（`sql/rpl_write_set_handler.cc` 中 `partition` 命中 0 次）；
- `rpl_mta_submode.cc` 里的 "partition"（`mts_assigned_partitions`、`usage_partition`）指的是 **db 分桶**，不是表分区。

`replica_preserve_commit_order` 与分区表没有任何耦合。实际影响反而是负面的：分区表不提供额外并行维度，跨分区 UPDATE 在从库被 `ph_update_row` 拆成 delete+insert，全部落在同一个 worker 内串行执行。

#### 7.3 并行：分区是 InnoDB 内部并行的天然分片

**（a）精确 COUNT(\*) 的并行**——`records()` 把每个分区当作一条独立 scan 塞进同一个 `Parallel_reader`：

```cpp
// storage/innobase/handler/ha_innopart.cc  (ha_innopart::records)
  size_t n_threads = thd_parallel_read_threads(m_prebuilt->trx->mysql_thd);
  n_threads = Parallel_reader::available_threads(n_threads, false);
  if (n_threads > 1 && trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
      m_prebuilt->select_lock_type == LOCK_NONE && ...) {
    ...
    for (auto i = first_used_partition; i < m_tot_parts;
         i = m_part_info->get_next_used_partition(i)) {
      set_partition(i);
      indexes.push_back(m_prebuilt->table->first_index());   // ★ 只收聚簇索引
    }
    row_mysql_parallel_select_count_star(trx, indexes, n_threads, &n_rows);
```

条件不满足就串行回退（逐分区 `ha_innobase::records()` 累加）。并发度 = `innodb_parallel_read_threads` 与全局配额 `Parallel_reader::available_threads` 的较小值（上限 `MAX_THREADS` 256）。分片是两阶段：根层静态切 + worker 空闲时动态再切（`Ctx::split()`）。

**（b）并行建索引：分区间串行，分区内并行**——这是两个不同层级的并行，容易混为一谈：

```
for (uint i = 0; i < m_tot_parts; ++i) {          ← 分区之间：串行
    set_partition(i);
    inplace_alter_table_impl<dd::Partition>(...);  ← 每个分区一个 ddl::Context
        └─ Loader::build_all()
             ├─ Parallel_cursor::scan()   ← 分区内部：并行扫聚簇索引（innodb_parallel_read_threads）
             └─ Loader::load()            ← 分区内部：多线程归并排序+装载（innodb_ddl_threads）
}
```

所以 ALTER 分区表的墙钟时间 ≈ Σ(各分区建索引时间)，受两类"分区内"并行加速，但**不受分区数并行加速**——没有任何代码把多个分区的索引合并进同一个 `Parallel_reader`（对比 `records()` 是一次喂进去的）。

**（c）两个限制**：并行扫描只支持聚簇索引（`row0pread.h` 注释明写 `Secondary index scans are not supported currently`）；`parallel_scan_init` 这条 handler API 在 8.0.39 的 SQL 层**没有任何调用点**（只有虚声明），属于已接线但无调用方的接口。

### 八、能力边界与限制

#### 8.1 能力标志与限制

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

#### 8.2 `REMOVE PARTITIONING`：一定退化为 COPY

把分区表变回普通表（`ALTER TABLE t REMOVE PARTITIONING`）**不是元数据操作，数据必须搬迁**。

SQL 层的关键决策（`prep_alter_part_table`）只有三行：

```cpp
// sql/sql_partition.cc
      if (alter_info->flags & Alter_info::ALTER_REMOVE_PARTITIONING) {
        if (!(create_info->used_fields & HA_CREATE_USED_ENGINE)) {
          create_info->db_type = tab_part_info->default_engine_type;
        }
        thd->work_part_info = nullptr;   /* 新表不再有分区定义 */
        *partition_changed = true;
      }
```

`partition_changed = true` 且 InnoDB 的 `partition_flags()` **不含** `HA_USE_AUTO_PARTITION`，也没有 `new_part_info`，于是 `mysql_alter_table` 直接判定走 COPY——**显式指定 `ALGORITHM=INPLACE` / `INSTANT` 会报 `ER_ALTER_OPERATION_NOT_SUPPORTED`**。

物理过程：建一个**新的无分区表**（普通 `ha_innobase`，单个 .ibd）→ `copy_data_between_tables` 逐行从旧分区表读出再插入 → 旧表改名备份 → 旧分区的 N 个 .ibd 由 `ha_innopart::delete_table` 逐个 `delete_impl` 删除。

注意最后一步：**不是"把 N 个文件合并成一个"**，而是 SQL 层把行重新写一遍。所以 REMOVE PARTITIONING 的代价 ≈ 全表重建。

#### 8.3 `REBUILD PARTITION`：inplace，逐分区重建

与 REMOVE PARTITIONING 相反，REBUILD **是 inplace**（不是 COPY），且**只重建被点名的分区**：

- SQL 层只把列出的分区标成 `PART_CHANGED`（`set_part_state`），其余保持 `PART_NORMAL`；
- 引擎侧 `check_if_supported_inplace_alter` 判定：`need_copy()` 为真 → 返回 `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE`（主阶段需 S 锁，但不重建整表）；
- 三阶段：prepare 为每个 `PART_CHANGED` 分区建一个临时 `dict_table_t` → inplace 阶段 `copy_partitions`（只扫这些分区，`write_row_in_new_part`）→ commit 阶段旧分区改临时名、新分区 rename 归位、旧表 drop。

`PART_NORMAL` 的分区从头到尾不动。

#### 8.4 分区表上的 ALGORITHM 支持矩阵

`ha_innopart::check_if_supported_inplace_alter` 的分支顺序（这是判定优先级，不是随便列的）：

| 顺序 | 条件 | 返回 |
|---|---|---|
| ① | ADD/DROP FOREIGN KEY | `NOT_SUPPORTED`（`ER_FOREIGN_KEY_ON_PARTITIONED`） |
| ② | ADD 全文索引 | `NOT_SUPPORTED` |
| ③ | 改变 KEY 分区字段顺序（`!same_key_column_order()`） | `NOT_SUPPORTED` |
| ④ | 增删主键且分区/子分区是 `PARTITION BY KEY()`（空字段列表） | `NOT_SUPPORTED` |
| ⑤ | 分区操作且 `need_copy()`（REBUILD / COALESCE / REORGANIZE / HASH·KEY 的 ADD PARTITION） | `SHARED_LOCK_AFTER_PREPARE` |
| ⑥ | 分区操作且 `!need_copy()`（DROP PARTITION、RANGE/LIST 的 ADD PARTITION） | `NO_LOCK_AFTER_PREPARE` |
| ⑦ | instant 类但字段数超 `REC_MAX_N_USER_FIELDS` | `NOT_SUPPORTED` |
| ⑧ | instant 条件全满足 | `HA_ALTER_INPLACE_INSTANT` |
| ⑨ | 显式 `ALGORITHM=INSTANT` 但行数/行版本/行长度超限 | `HA_ALTER_ERROR` |
| ⑩ | 其余全部交给基类（在分区 0 上执行） | 基类判定 |

INSTANT 的额外条件是**所有分区都必须支持**：

```cpp
// handler0alter.cc  (innopart_support_instant)
  for (uint32_t i = 0; i < num_parts; ++i) {
    type = innobase_support_instant(ha_alter_info, part_share->get_table_part(i), ...);
    if (type == Instant_Type::INSTANT_IMPOSSIBLE) return (type);
  }
```

任一分区不支持 → 整体退化为 INPLACE。

#### 8.5 `CHECK` / `REPAIR PARTITION` 与"错放行"

- **`check`**：对每个 used 分区做 `ha_innobase::check`，只有 `T_MEDIUM` / `T_EXTEND` 才额外调 `Partition_helper::check_misplaced_rows(i, false)` 检查"行是否放错分区"。
- **`repair`**：非 MEDIUM/EXTEND 直接返回 OK；且**不修索引**——注释明写 `TODO: Implement and use ha_innobase::repair()`，实际只做一件事：把放错分区的行搬走。
- **`optimize`**：直接返回 `HA_ADMIN_TRY_ALTER`，让 SQL 层改写成 `ALTER TABLE ... ENGINE=InnoDB`。

`check_misplaced_rows(part_id, repair)` 的实现要点：

1. repair 模式读整行，check 模式只读分区字段（省 IO）；
2. 判定方式是**对读出的每一行重新求分区 id** 再与目标分区比较：`get_partition_id(...) != read_part_id` 即为错放行；
3. repair 模式：先 `write_row_in_part(correct_part_id)` 插入正确分区，再 `delete_row_in_part(read_part_id)` 从错误分区删除——**同一事务**，重复键按 `ignore` 处理；
4. 这类行通常来自"分区定义被 ALTER 改动后数据未重分布"的历史遗留。

## 可观测性

### 系统变量与状态变量

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

```cpp
/* ★ 常量的真实定义点（逐个核查过） */
/* sql/sql_const.h:118 —— ⚠️ 不在 my_base.h，也不是 1024（那是 5.x 的值） */
constexpr const int MAX_PARTITIONS{8192};

/* sql/partition_info.h:48 */
#define NOT_A_PARTITION_ID UINT_MAX32

/* sql/partitioning/partition_handler.h:61,65 */
#define PARTITION_BYTES_IN_POS 2          /* ★ ref 前缀里分区号占 2 字节 */
static const uint NO_CURRENT_PART_ID = UINT_MAX32;

/* sql/partition_info.cc:543 —— 默认分区名缓存大小 */
#define MAX_PART_NAME_SIZE 8
```

> ★★ `NOT_A_PARTITION_ID` 与 `NO_CURRENT_PART_ID` **值相同但语义不同**：前者表示"查不到这个分区名"（`get_part_elem` 失败），后者表示"当前没有活动分区"（`m_last_part` / `m_part_spec.start_part` 的初值）。
> ⚠️ **`NO_PARTITION` 常量不存在**。

---

### 观测对象 → 手段 速查

| 想看什么 | 手段 | 源码依据 |
|---|---|---|
| **裁剪是否生效** | `EXPLAIN` 的 `partitions` 列 / `EXPLAIN FORMAT=JSON` 的 `"partitions"` 数组 | `make_used_partitions_str` 遍历 `read_partitions` 生成。⚠️ 8.0 已移除 `EXPLAIN PARTITIONS` 语法，`PARTITIONS` 仅剩非保留关键字 |
| **逐分区的裁剪决策** | `SET debug='+d,info'` 后看 error log | `partition_pruning.cc` 的 `DBUG_PRINT("Mark partition %u as used")`、`dbug_print_segment_range` 打印区间 |
| **裁剪的内存开销** | PFS `memory/partition/%`，键 `Partition::prune_exec` | `MEM_ROOT alloc(key_memory_partitions_prune_exec, ...)` |
| **分区统计** | `I_S.PARTITIONS` | 动态值走 `get_dynamic_partition_info` → `ha_innopart::info_low` |
| **binlog 里的分区信息** | `mysqlbinlog -v`，输出 `### Extra row info for partitioning: source_partition: N target_partition: M` | `Rows_log_event::print_verbose_one_row`——**唯一**的读取点 |
| **统计刷新粒度** | `HA_STATUS_*` 标志：`HA_STATUS_NO_LOCK` 置位即"I_S 读取"，不带即"优化器读取" | `ha_innopart::info_low` 按位分支 |
| **是否强制重算统计** | `innodb_stats_on_metadata=ON` 时每次 I_S/SHOW 都触发各分区 `dict_stats_update` | `ha_innopart.cc` 的 `update_table_stats` |
| **分区相关的 status 变量** | **不存在** | 全仓无分区 `SHOW_VAR` 条目；`Partition::...` 都是 PFS 内存键名，不是 status 变量 |

#### 为什么查 `I_S.PARTITIONS` 在分区多时极慢

四条叠加原因（都有源码依据）：

1. **每行 9 次 `INTERNAL_*()` 调用**——TABLE_ROWS / AVG_ROW_LENGTH / DATA_LENGTH / MAX_DATA_LENGTH / INDEX_LENGTH / DATA_FREE / UPDATE_TIME / CHECK_TIME / CHECKSUM 各自求值。
2. **统计缓存是单槽的，key 含分区名**——`form_key()` = `db.table[.partition]`，注释明说 "We store just a single table statistics in this cache"。于是**每换一个分区就整槽失效**，每一行都要重新 `open_tables_for_query()`（含 MDL、DD 查找、打开**全部**分区）。
3. **取 1 个分区的统计，代价是 O(总分区数)**——`get_dynamic_partition_info_low` 把位图临时清成 `{part_id}` 后调 `info()`，而 `ha_innopart::info_low` 对 `HA_STATUS_VARIABLE` 会遍历**所有**分区累加。
4. **CHECKSUM 列**触发 `checksum_in_part(part_id)`，对 InnoDB 意味着真实扫描该分区。

唯一缓解是持久统计未过期时直接读 `mysql.table_stats`；把 `information_schema_stats_expiry` 设为 0 会**放大**上述开销。

#### 典型排查路径

| 症状 | 排查顺序 |
|---|---|
| **裁剪没生效** | ① `EXPLAIN FORMAT=JSON` 看 `partitions` 数组是否含全部分区 ② 按 §2.3 的三条件自查（表达式单调？类型可反解？条件落在第 1 个分区列？） ③ `SET debug='+d,info'` 看逐分区的裁剪日志 |
| **分区多导致 DDL / 打开表变慢** | `open` 无条件打开全部 leaf 分区 + 逐分区 `dict_stats_init`；分区数是硬成本，没有开关能绕过 |
| **`I_S.PARTITIONS` 很慢** | 见上文四条根因；缓解是靠持久统计（不要设 `information_schema_stats_expiry=0`），且避免查 `CHECKSUM` 列 |
| **跨分区 UPDATE 慢 / 出现死锁** | ① 是否改了分区键（→ 锁集=全分区 + 物理上 delete+insert） ② 死锁日志里看两条事务的分区迁移方向是否相反（§6.2） |
| **执行计划突然变差** | `rec_per_key` 只取最大分区（被低估 P 倍）+ dive 结果被 `table_n_rows/2` 削顶（§5.2）；分区越碎失真越大 |
| **`EXCHANGE PARTITION` 失败** | 依次查：任一方有 instant column、指定了 INDEX DIRECTORY、结构不一致（ROW_FORMAT 等）、被交换表带外键 |
| **统计不准** | `ANALYZE PARTITION pN` 只重算指定分区；注意采样页数是**每分区**语义（§5.3），不是整表共享 |

## ★ 本机制里的工程实现技法

### 一、高级语法技巧 / C++ 特性

| 技法 | 落点 | 怎么用 |
|---|---|---|
| **函数指针派发** | `partition_info::get_partition_id`、`get_part_iter_for_interval` | 建表时在 `fix_partition_func` 里按 RANGE/LIST/HASH/KEY/COLUMNS 把 14 种实现之一挂上，运行期零分支调用 |
| **模板特化（显式实例化）** | `dd_open_table<dd::Table>` / `<dd::Partition>`；`innobase_truncate<dd::Table>` / `<dd::Partition>` | 同一套代码服务"整表"与"分区"两种 DD 实体，是"一分区一 dict_table_t"模型能复用的关键 |
| **同一模板混用** | `rename_impl<dd::Table>` 与 `rename_impl<dd::Partition>` 交替调用（EXCHANGE 的三次 rename） | 普通表与分区在同一套 rename 逻辑下互换文件 |
| **mixin 多重继承** | `class ha_innopart : public ha_innobase, public Partition_helper, public Partition_handler` | 存储能力 / 通用分区行为 / 分区 handler 接口三块正交拼装 |
| **placement new（MEM_ROOT）** | `new (mem_root) ha_innopart(hton, table)` | handler 对象分配在 TABLE_SHARE 的 MEM_ROOT 上，随表生命周期一次性释放 |
| **const_cast 绕过只读接口** | `const_cast<dd::Partition *>(dd_part)`、`const_cast<dd::Column *>` | DD 接口返回 const 指针，写入路径（改 `se_private_data`、加列）用它拿回可写视图 |
| **"手工 RAII"的线程配额** | `Parallel_reader::release_threads(n_threads)` | 配额用 `fetch_add` 领用后，**每个错误分支都要显式归还**——没有 RAII 包装，是最易漏的一类写法 |
| **带上限的 MEM_ROOT** | `MEM_ROOT alloc(key_memory_partitions_prune_exec, thd->variables.range_alloc_block_size)` + `set_max_capacity(range_optimizer_max_mem_size)` + `set_error_for_capacity_exceeded(true)` | 裁剪器把内存上限变成"可失败的优雅降级"：超限不是 OOM，而是抛错被上层捕获后 `goto all_used` |
| **`ut::unique_ptr<T[]>` 管理分区数组** | `m_parts = ut::make_unique<saved_prebuilt_t[]>(m_tot_parts)` | 分区数运行期才定，用智能指针数组替代裸 new[] |
| **`dynamic_cast` 下行转换** | `ha_innopart::clone` 里 `dynamic_cast<ha_innopart *>(handler::clone(name, mem_root))` | 基类工厂返回 `handler*`，派生类 clone 需要拿回自己类型 |
| **`virtual` + `override` 的三层派发** | `ha_innobase::info_low` 是虚函数 → `ha_innopart::info_low` override；`analyze()` 只调 `info_low` | 加一层间接，让"统计采集"自动走分区版，基类代码零改动 |

### 二、经典算法的实现落地（不限于并发）

| 算法 | 落点 | 说明 |
|---|---|---|
| **k 路有序归并** | `Prio_queue`（`Priority_queue<uchar*, vector, Key_rec_less>`）；`assign(parts)` / `top()` / `update_top()` | 有序跨分区扫描的骨架；`m_max_at_top` 翻转比较器，一份堆代码同时支持正序与逆序 |
| **排序数组 + 二分** | `list_array`（LIST）、`range_int_array`（RANGE）；`std::sort` 构建一次，查询 O(log n) | 构建期排序 + 相邻去重检查，查询期纯二分——比 hash 省内存且天然支持"区间端点"查询 |
| **位图集合代数** | `bitmap_intersect` / `bitmap_copy` / `bitmap_set_all` 于 `read_partitions` / `lock_partitions` | 分区集合是 0..8191 的稠密整数集，交/并/子集判定 O(n/64) |
| **线性哈希** | `get_part_id_from_linear_hash(hash_value, mask, num_parts)` | LINEAR HASH/KEY 用 mask 递推避免取模 |
| **swap-with-temp** | EXCHANGE PARTITION 的三次 rename | 两个目标名互相占用，必须借临时名做三跳 |
| **两阶段分片 + work stealing** | `Parallel_reader`：`add_scan` 时根层静态切，worker 空闲时 `Ctx::split()` 动态再切 | 见 `row0pread.h` 的设计注释（"5 个子树 4 个线程"的例子） |
| **区间端点映射（单调反解）** | `Item::val_int_endpoint` + `get_monotonicity_info` | 把 `F(x) OP const` 变成 `x OP' F⁻¹(const)`——这是裁剪能作用于 RANGE/LIST 的**唯一**理论支点 |
| **带端点语义的二分变体** | `get_list_array_idx_for_endpoint`（LIST）、`get_partition_id_cols_range_for_endpoint`（COLUMNS） | 二分的比较结果还要结合 `left_endpoint ^ include_endpoint` 决定是否 ±1，是 lower_bound/upper_bound 的手写变体 |
| **批量缓冲的同质判定** | `compare_extra_row_info`（binlog pending Rows event） | 判断新行能否并入当前 event：分区号不同就必须 flush 另起一个——"缓冲 + 同质性校验"是批量写的通用套路 |

### 三、复杂体系与设计模式的代码结构

| 模式 | 落点 | 说明 |
|---|---|---|
| **模板方法** | `Partition_helper` 的 `write_row_in_part` / `update_row_in_part` / `rnd_init_in_part` 等纯虚；驱动方 `ph_write_row` / `ph_update_row` | 基类固定"算分区号 → 锁集校验 → autoinc → 调 `*_in_part`"的骨架，落盘交给引擎 |
| **策略** | `PARTITION_ITERATOR::get_next` 函数指针，RANGE/LIST/NULL-only 各有实现 | 分区枚举策略运行期装配，调用方只写 `iter.get_next(&iter)` |
| **三段式 RAII（DDL 事务）** | `innobase_truncate`：`prepare()` → `truncate()` → `cleanup()`，析构兜底 | 分区 TRUNCATE 逐分区各走一遍三段式；`cleanup` 无论成败都收尾 |
| **引用计数共享** | `Ha_innopart_share::m_ref_count` + `increment_ref_counts` / `close_table_parts` | 多个 handler（含 clone）共享同一份"每分区 dict_table_t 数组 + 索引映射"，最后一个关闭者才真正释放 |
| **上下文保存/恢复** | `saved_prebuilt_t m_parts[]`；`set_partition()` 拷回、`update_partition()` 拷入 | 一份 `row_prebuilt_t` 被 N 个分区复用，切换 = 一次结构体拷贝（类比寄存器 spilling） |
| **编排器 / 生产者-消费者** | `Parallel_reader` → `Scan_ctx` → `Ctx` 三层 + `enqueue/dequeue` + worker 主循环 | Scan 描述（索引+trx）与执行上下文（游标+线程）分离，才能"多分区共用一池线程 + 动态再切分" |
| **Builder + Loader** | 建索引：`Loader::prepare`（每索引一个 `Builder`）→ `scan_and_build_indexes` → `load()`（`Task_queue(m_n_threads)`） | 扫 → 落临时文件 → 多线程归并排序 → 批量装载 |
| **状态机驱动的工厂分派** | `partition_element::part_state`（`PART_NORMAL` / `PART_CHANGED` / `PART_TO_BE_ADDED` / `PART_TO_BE_DROPPED`）→ `alter_parts` 按状态生成 `alter_part_normal` / `alter_part_change` / `alter_part_add` / `alter_part_drop` | 一条 ALTER 里不同分区有不同命运：用"状态枚举 + 工厂"把 N 个分区的异构处理统一成一次遍历，每个 `alter_part_*` 自己实现 `prepare` / `try_commit` / `rollback` |
| **三段式 + 显式状态机** | quiesce：`QUIESCE_NONE → QUIESCE_START → QUIESCE_COMPLETE`（`row_quiesce_set_state`） | 三段式 RAII 之外再叠一层状态机，用于跨函数（`store_lock` 置 START → `external_lock` 执行 → `complete` 收尾）的协作 |
| **引用计数 + 条件释放** | `Ha_innopart_share::m_ref_count`（见三段式那段）之外：`dict_table_t::acquire()` 在 `increment_ref_counts` 里逐分区调用 | "shared 对象 + 分区对象"两级引用计数，任一层归零才真正释放 |

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

### FAQ

| 问题 | 简答 | 详见 |
|---|---|---|
| **分区数上限为什么是 8192？** | `MAX_PARTITIONS` 常量。真正的约束不是这个数本身，而是打开表要装载全部 N 个 `dict_table_t` 并逐个 `dict_stats_init`——8192 是"元数据开销尚可接受"的妥协点，不是性能甜点 | §1.3 |
| **为什么分区表不支持外键？** | `innobase_partition_flags()` 返回 `HA_CANNOT_PARTITION_FK`。根因是分区在引擎层是 N 张独立表、又没有全局索引，无法跨分区保证引用完整性 | §8.1 |
| **为什么唯一键必须包含分区列？** | 每分区一套**本地索引**，唯一性只能在分区内保证。若唯一键不含分区列，同一个键值可能落在两个不同分区而无人发现 | 概述 |
| **HASH 分区为什么基本裁不动？** | `% num_parts` 不可反解。只能等值点正向算，或逐值枚举（值域 ≤ `MAX_RANGE_TO_WALK`=32），所以实际只有 `IS NULL` 能稳定裁 | §2.3 |
| **分区表的 auto_increment 会有空洞吗？** | 计数器是**全表共享**的 `Partition_share::next_auto_inc_val`，不是每分区一份。空洞来自预留区间/回滚/REPLACE，与分区无关 | §2.4 |
| **`TRUNCATE PARTITION` 和 `DELETE` 差在哪？** | 前者是 rename→delete→create 重建表空间：无 undo、不进 purge、空间立即释放；后者逐行删，undo + purge 压力大且不释放空间 | §3.3 |
| **`EXCHANGE PARTITION` 会搬数据吗？** | 不搬。三次 rename 换文件 + 交换 `se_private_data`；只有 `WITH VALIDATION` 才逐行扫描校验 | §3.4 |
| **UPDATE 改了分区键会怎样？** | 锁集退化为**全部分区**；物理上是先插新分区再删旧分区；binlog 仍记 `Update_rows` + `source_partition_id` | §2.5 |
| **为什么分区多了反而变慢？** | 四条叠加：打开表无条件全开、`rec_per_key` 只取最大分区（低估 P 倍）、`I_S.PARTITIONS` 逐分区开表、代价模型因削顶而不对称 | §1.3 / §5.2 / 可观测性 |
| **`REMOVE PARTITIONING` 是元数据操作吗？** | 不是。强制 COPY：建一张无分区新表 → 逐行重写 → 旧分区 .ibd 逐个删除。显式 `ALGORITHM=INPLACE/INSTANT` 会报错 | §8.2 |
| **`.cfg` 是什么、`FLUSH TABLE FOR EXPORT` 为什么要它？** | 表空间元数据文件（列定义 + 索引 root page no 等）。`.ibd` 里没有这些信息，IMPORT 时要靠它校验；分区表**每个分区一份** | §6.3 |
| **同一条查询前后两次结果不一样？** | 分区定位失败是**静默**的：空分区集被伪造成 `HA_ERR_END_OF_FILE`（"查无此行"），不报错。若 `FLUSH TABLES` 后恢复，多半是执行期定位的状态残留；若 `Handler_read_key` 为 0 但表里有数据，基本可确认 | §2.3 / [`../index/partitioning.md`](../index/partitioning.md) |
| **分区表上二级索引查找为什么会牵扯到主键列？** | InnoDB 二级索引**物理上隐含 PK 列**（index extension），server 层 `KEY::key_part[]` 尾部也有这些隐藏列（`actual_key_parts > user_defined_key_parts`）。若隐藏列同时是分区键，就会被当作"已绑定"的分区字段 | [`../index/partitioning.md`](../index/partitioning.md) |

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `class partition_info` 在 `sql/sql_partition.h` | 在 **`sql/partition_info.h:209`**；`sql_partition.h` 只有自由函数声明 |
| `enum partition_type` 在 `partition_info.h`、有 `KEY` 值 | 在 **`sql/partition_element.h:33`**，是 `enum class`，**只有 4 个值** `NONE/RANGE/HASH/LIST`；**KEY = `HASH` + `list_of_part_fields==true`** |
| `enum_part_state` | **不存在** → `enum partition_state`（10 个值，`PART_NORMAL`..`PART_ADMIN`） |
| `check_partition_info()` 在 `sql/sql_partition.cc` | 在 **`sql/partition_info.cc:1413`**；第 5 参真名 `add_or_reorg_part` |
| `MAX_PARTITIONS` 在 `my_base.h`、值 1024 | 在 **`sql/sql_const.h:118`**，值 **8192** |
| `fill_partition_info_from_dd()` / `dd_write_partition_info()` | **均不存在** → 写 **`fill_dd_partition_from_create_info()`**；读 **`fill_partitioning_from_dd()`** + **`setup_partition_from_dd()`** |
| DD 把分区信息存成一整段文本 | **部分错**。类型/表达式/值/索引都是**结构化列**；只有 `partition_expression` 与 `TABLE_SHARE::partition_info_str` 是文本 |
| `sql/dd/impl/partition_impl.cc` | **不存在** → `sql/dd/impl/types/partition_impl.cc` |
| `sql/dd/dd_table_share.cc` | **不存在** → `sql/dd_table_share.cc`（sql 根目录） |
| `get_partition_id` 是虚函数 | 是 **5 个函数指针成员**；真实函数名是 `get_partition_id_with_sub` / `get_part_partition_id` / `get_subpartition_id` 三族 |
| 有 `get_partition_id_range_sub` / `_list_sub` | **不存在**。RANGE/LIST 只出现在 `get_part_partition_id`（主分区维）；子分区维恒为 HASH/KEY |
| `get_used_partitions()` | **不存在** → `set_used_partition()`；查询侧 `is_partition_used()` / `num_partitions_used()` |
| `mark_all_partitions_as_used()` 在 `sql_partition.cc` / `opt_range.cc` | 是 **static**，在 **`sql/range_optimizer/partition_pruning.cc:1038`**，**函数体只有 1 行** |
| 裁剪在 `sql/opt_range.cc`、有 `Partition_pruning_module` 类 | **`sql/opt_range.cc` 整个文件不存在**（8.0 拆到 `sql/range_optimizer/`）；**无 `Partition_pruning_module` 类**（是自由函数 `prune_partitions()` + `PART_PRUNE_PARAM`） |
| `read_partitions` 是 `Bitmap<>` 模板 | 类型是 **`MY_BITMAP`**（C 结构） |
| `TABLE_SHARE::part_info` | `TABLE_SHARE` 里是 **`m_part_info`**；`TABLE` 里才叫 **`part_info`** |
| `NO_PARTITION` 常量 | **不存在** → `NOT_A_PARTITION_ID` 与 `NO_CURRENT_PART_ID`（值同为 `UINT_MAX32`，语义不同） |
| `sql/partitioning/partition_helper.{h,cc}` | **两个文件都不存在**。`Partition_helper` 定义与实现全在 **`partition_handler.{h,cc}`**（该目录只有这 2 个文件） |
| `sql/sql_partition_exec.cc` | **不存在** |
| `ha_partition`（老的 .par 分区引擎）还在 | **已删除**。`sql/handler.h:4092` 注释明写 "copied from storage/partition/ha_partition.h when support for non-native partitioning was **removed**" |
| `Partition_helper::ph_get_auto_increment()` | **不存在** → **`get_auto_increment_first_field()`**；另有 `set_auto_increment_if_higher()` / `ph_release_auto_increment()` |
| `ph_start_bulk_insert()` / `ph_end_bulk_insert()` | **不存在**。bulk insert 走普通 `ph_write_row()` 逐行路由 |
| `ph_prepare_for_new_value()` | **不存在**。一致性校验的真实载体是 **`get_parts_for_update()`** / **`get_part_for_delete()`** |
| `ph_index_init()` | **不存在** → **`ph_index_init_setup(uint, bool)`** |
| `Partition_handler::set_partition()` | `Partition_handler`/`Partition_helper` 里**都没有** → 引擎侧 `ha_innopart::set_partition(uint)` |
| 优先队列用 `QUEUE` | **不是**。是 **`Priority_queue<uchar*, std::vector<uchar*>, Key_rec_less>`**，成员名 **`m_queue`**；缓冲 **`m_ordered_rec_buffer`**；比较器 **`Key_rec_less`** |
| 全表扫用优先队列归并 | **错**。全表扫是**串行**（EOF → `rnd_end` → 下一分区 → `rnd_init`）；归并**只用于有序索引扫** |
| `ha_innopart::truncate()` | **不存在** → **`truncate_impl()`**（整表）+ **`truncate_partition_low()`**（TRUNCATE PARTITION） |
| `ha_innopart::rename_partitions()` / `change_partitions()` | **不存在** → `prepare_inplace_alter_partition()` / `inplace_alter_partition()` / `commit_inplace_alter_partition()` |
| `ha_innopart::delete_all_rows()` 有分区实现 | **没有** → 一行转发 `handler::delete_all_rows()` |
| `ha_innopart::check_and_repair()` | **不存在** → `check()` / `repair()` |
| `ha_innopart::export_tablespace()` | **不存在**。`FLUSH TABLE ... FOR EXPORT` 走 **quiesce**（`store_lock()` 置 `QUIESCE_START` + `external_lock()` 里 `row_quiesce_table_start()`） |
| `ha_innopart::info()` | **不存在**（`handler::info()` 非虚）→ **`info_low(uint flag, bool is_analyze)`** |
| `storage/innobase/partitioning/part0part.cc` | **不存在**。无 `partitioning/` 目录；全在 `handler/ha_innopart.{h,cc}` + `handler0alter.cc` |
| DDL log 有 `write_add_partition_log` / `write_drop_partition_log` | **不存在**。8.0 只有 7 类：`FREE_TREE` / `DELETE_SPACE` / `RENAME_SPACE` / `DROP` / `RENAME_TABLE` / `REMOVE_CACHE` / `ALTER_ENCRYPT(_UNENCRYPT)`，全在 **`storage/innobase/log/log0ddl.cc`** |
| EXCHANGE PARTITION 会 upgrade 到 `MDL_EXCLUSIVE` | **代码里没有 `upgrade_shared_lock`**。只 `set_type(MDL_SHARED_NO_WRITE)`，靠 `wait_while_table_is_used()` + `close_all_tables_for_name()` 驱赶。**注释与代码不一致**（注释仍写 "Upgrade to exclusive mdl"） |
| 有 `Sql_cmd_alter_table_add/drop/reorganize_partition` | **不存在** → 统一走 `mysql_alter_table()` + INPLACE 分区 API；只有 **6 个**独立 Sql_cmd（EXCHANGE / TRUNCATE / ANALYZE / CHECK / OPTIMIZE / REPAIR） |
| TRUNCATE PARTITION = 删行 | **不是** → rename 表空间 → `delete_impl`(drop) → `create_impl`，逐分区执行 |
| `REMOVE PARTITIONING` 可以 INPLACE | **一定退化 COPY**；显式 `ALGORITHM=INPLACE/INSTANT` 报 `ER_ALTER_OPERATION_NOT_SUPPORTED` |
| "unique key 必须含所有分区列"是绝对的 | **不是**。只有引擎**不**具备 `HA_CAN_PARTITION_UNIQUE` 时才强制（见 `fix_partition_func` 尾部的条件判断） |

**★ 三个"结构性事实"**

1. **"每个分区在 InnoDB 里是各自独立的 `dict_table_t`"——这是理解原生分区一切行为的根**。`ha_innopart::set_partition()` 的核心一行是 `m_prebuilt->table = m_part_share->get_table_part(part_id)`；切换分区=整体换掉 `pcur`/`clust_pcur`/`ins_node`/`upd_node`/`table`/`index` 六项。由此推出：分区表没有全局索引（所以 unique key 必须含分区列、不支持外键）、`ref` 要多 2 字节放分区号（`PARTITION_BYTES_IN_POS`）、`I_S.PARTITIONS` 每个分区都要单独采集统计。

2. **`Partition_helper` 是"模板方法"mixin，`*_in_part()` 是留给 SE 的插槽——server 侧只做三件事**：算分区号、查锁集、管 autoinc。所以它才能被 `ha_innopart` 多重继承复用，而 `m_last_part` 这个"游标当前所在分区"的变量成为跨分区一致性校验的支点（`old_part_id != m_last_part` → `HA_ERR_ROW_IN_WRONG_PARTITION`）。**这个校验刻意"报错而非纠正"**——注释说得很直白：读路径不逐行校验，若默默纠正，用户永远不会知道数据分布坏了。

3. **"DD → 文本 → 重新解析"这个绕路不是历史包袱，而是 Item 生命周期的必然**。" 注释原文：*"[sub]part_expr still needs to be parsed from [sub]part_func_string for each TABLE instance to use the correct mem_root"*。所以 `generate_partition_syntax()` 反向生成整段 `PARTITION BY` 子句存进 `TABLE_SHARE::partition_info_str`，**每个 TABLE 实例打开时都要再 parse 一遍**（`mysql_unpack_partition` 复用 yacc 的 `partition_clause` 语法），然后重新跑 `fix_parser_data()` + `fix_partition_func()` 绑定函数指针。这是分区多时打开表开销的一个来源，也是"改了分区定义必须重开表"的原因。

> 注：`sql/partition_info.h`、`sql/sql_partition.cc`、`storage/innobase/handler/ha_innopart.cc` 等文件中的部分 `// 中文` 注释为本工作区人工批注，非 Oracle 上游原文。

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
- 原子 DDL 与 DDL log（分区 TRUNCATE / DROP 的崩溃安全来源）：[`../feat/ddl/ddl.md`](../feat/ddl/ddl.md)
- InnoDB 内部并行扫描：[`../innodb/parallel_scan.md`](../innodb/parallel_scan.md)
