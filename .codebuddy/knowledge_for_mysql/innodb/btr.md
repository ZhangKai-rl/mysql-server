# InnoDB B-tree：结构修改与游标操作（btr 模块）

> 基于 MySQL 8.0.39 源码。剖析 InnoDB 索引 B-tree 的全套结构操作：游标搜索（`btr0cur`）、页面分裂/合并/根页管理（`btr0btr`）、持久游标（`btr0pcur`）、自适应哈希索引 AHI（`btr0sea`）、排序批量构建（`btr0load`）。
>
> **边界**：本篇讲 **B-tree 结构与索引记录**这一层；SQL 层如何用游标逐行取数据（`row_search_mvcc` 主链、行缓冲转换）见 [`row_search.md`](row_search.md)；页物理格式与页内目录槽二分定位见 [`physical/page_structure.md`](physical/page_structure.md)；记录物理格式与 offsets 解析见 [`physical/record.md`](physical/record.md)；latch 体系与行锁见 [`lock.md`](lock.md)；mtr（mini-transaction）与 redo 见 [`redo_log.md`](redo_log.md)；undo 与 purge 见 [`undo_log.md`](undo_log.md)；LOB 外存字段在 btr 中的交互见 [`physical/lob.md`](physical/lob.md)；online DDL 与并行索引构建上下文见 [`ddl.md`](ddl.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [游标体系：`btr_cur_t` 与 `btr_pcur_t`](#游标体系btr_cur_t-与-btr_pcur_t)
  - [搜索：`btr_cur_search_to_nth_level`（逐行解析）](#搜索btr_cur_search_to_nth_level逐行解析)
  - [页内二分：`page_cur_search_with_match`](#页内二分page_cur_search_with_matchpage0curcc-逐行解析)
  - [插入与分裂](#插入与分裂)
  - [中间页与 node pointer](#中间页与-node-pointer构建解析最小记录标记)
  - [删除与合并（含 `btr_compress` / `btr_lift_page_up` 逐行）](#删除与合并)
  - [更新：三条路径](#更新三条路径)
  - [自适应哈希索引（AHI）](#自适应哈希索引ahi)
  - [批量构建：`Btree_load`](#批量构建btree_load)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

btr（B-TRee）是 InnoDB 中负责**索引 B-tree 一切结构操作**的代码层，由 5 个文件构成，职责分明：

| 文件 | 职责 |
|------|------|
| `btr0cur.cc` | 树游标与 DML 核心：搜索下钻、乐观/悲观插入、删除、更新、叶子锁（约 5000 行，最核心） |
| `btr0btr.cc` | 结构修改（SMO）：页面分裂、合并、重整、根页抬高、页面分配/释放、树统计 |
| `btr0pcur.cc` | 持久游标：跨 mtr 保存/恢复位置、跨页推进 |
| `btr0sea.cc` | 自适应哈希索引（AHI）：统计、构建、探测、失效维护 |
| `btr0load.cc` | 排序批量构建（`Btree_load`）：DDL 建索引的"自底向上"打包路径 |

上游是 `row` 层（`row0sel`/`row0ins`/`row0upd`/`row0purge`/`row0umod`/`row0uins`），它们把 SQL 操作翻译成"在某个索引上对某个 entry 做某件事"；下游是 `page` 层（页内记录链表与目录槽）、`mtr`（原子性与 redo）、`lock`（行锁）、`fsp`（页分配）。btr 层把这些零件组装成一颗**并发安全、可崩溃恢复的 B+ 树**。

### 用途

InnoDB 是索引组织表（IOT）：聚簇索引的叶子就是表数据本身，二级索引的叶子是（索引键 → 主键）映射。因此 btr 层解决的不只是"索引查找"，而是**整个引擎的数据组织问题**：每一条 SQL 写入（INSERT/UPDATE/DELETE）最终都落到 btr 层对某个 B-tree 的插入、删除、更新与分裂/合并上；每一条读取（点查/范围扫描）都落到 btr 层的搜索与游标推进上。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | WL#7277 快速索引创建：新增 `btr0bulk.cc` 排序批量构建，建索引不再逐条插入 |
| 5.7 | 新增全局变量 `innodb_fill_factor`（10–100，默认 100）控制 bulk 页的预留空间；分裂填充因子可经变量调整 |
| 8.0 | `btr0bulk.cc` 重构为 `btr0load.cc` 的 `Btree_load` 类，并接入并行 DDL 框架 `ddl0loader`/`ddl0builder`；B-tree SMO 的树锁从 X 演进为 **SX + 搜索路径预测裁剪**；分裂填充逻辑回归硬编码 50%，可调变量移除；LOB 重构（`lob::BtrContext`，旧 `btr_store_big_rec_extern_fields` 删除）；`node_ptr_optimistic_delete` 拆为 `btr_node_ptr_delete` + `btr_insert_on_non_leaf_level`；instant ADD COLUMN 的物化判断进入更新路径 |
| 8.0.30 | AHI 大改：全局单一 `search_latch` 拆为按 (space_id, index_id) 哈希的 **sharded latch**，新增只读变量 `innodb_adaptive_hash_index_parts`（1–512，默认 8）；`btr_search_info_t` 改名 `btr_search_t`；`hash_table_t` 以 0 个内部同步对象创建（内容完全由 part latch 保护） |
| 8.0.27 | redo 新增 `MLOG_LIST_END_DELETE_8027` / `MLOG_COMP_LIST_END_DELETE_8027` 新格式（恢复端兼容新旧） |

> **为什么**演进（X→SX 的动机、为什么保留普通 B-tree 而非 B-link、为什么 bulk 与单条插入采用截然不同的策略）见「理论基础」。

---

## 理论基础

### 设计思想与权衡

#### 1. 索引组织表（IOT）：数据就是聚簇 B+ 树

InnoDB 没有独立的堆表。聚簇索引的**叶子记录 = 完整行数据**（含 `trx_id`/`roll_ptr` 隐藏列），非叶层只存"子页最小 key + 4 字节子页号"（node pointer）。二级索引的叶子存"索引键 + 主键值"，定位后回表。

**权衡**：优点是主键查询一次下钻即得全行、范围扫描天然聚簇有序、无需 rowid 间接层；代价是二级索引"胖"（要带主键值）、主键更新昂贵（delete-mark + 重插）、无主键表还要额外分配 6 字节 hidden row id 维持聚簇性。这是 InnoDB 一切 B-tree 设计的起点——分裂、合并、latch 策略都围绕"叶子是数据本身"展开。

#### 2. 普通 B-tree 而非 B-link 树：SMO 靠锁而非兄弟指针

文件头注释（btr0btr.cc）明确写明："For each page there is exactly one node pointer stored: thus our tree is an ordinary B-tree, not a B-link tree."

B-link 树（Lehman & Yao, 1981）的核心思想是：每个节点额外保存指向**右兄弟**的指针和"high key"，使 SMO 期间搜索能"从右兄弟再找一次"而不阻塞读者。InnoDB 选择**普通 B-tree**：

- 兄弟链（`FIL_PAGE_PREV/NEXT`）只服务于**顺序扫描**与分裂/合并挂接，**不是搜索路由**——搜索永远从 root 沿 node pointer 下钻；
- 后果是 SMO（分裂/合并）必须**串行化**：8.0 用 `index->lock` 的 SX 锁排斥其他 SMO，分裂时还要 X 锁**左、中、右三个叶子兄弟页**（因为 prev/next 指针会被改写）。这与 B-link 树"SMO 不阻塞读"的理念相反，换来的是实现简单、node pointer 语义单一（每页恰一条、无 high key 维护）。

**代价落到具体后果**：热点页分裂瞬间，所有要写该页或途经该页的并发操作全部被 SX/X 锁挡住——这是 InnoDB 高并发插入下锁等待的主要来源之一（`innodb_row_lock_current_waits` 背后常是树锁而非行锁）。

#### 3. 乐观/悲观双路径：先小锁试探，失败再升级

所有 DML 都有两条路径：

- **乐观路径**（`BTR_MODIFY_LEAF`）：index 加 S 锁（与并发读/写共存），只 X 锁目标叶子页；假设"页内放得下、不用动树结构"。绝大多数 OLTP 操作走这条路，代价只是一次页内搜索 + 一次页锁。
- **悲观路径**（`BTR_MODIFY_TREE`）：index 加 SX（默认），路径页按预测 X 锁；假设"要分裂/合并/根抬高"。**SMO 一旦开始不可逆**——分裂中间态无法回滚（"the operation of this function must always succeed, we cannot reverse it"），所以悲观操作**先预留表空间 extent**（`fsp_reserve_free_extents`，插入预留 `tree_height/16 + 3` 个，删除预留 `tree_height/32 + 1` 个），保证分裂中途不会因空间耗尽而失败。

乐观路径失败的触发条件本身就是设计精华：插入时"重整后最大可插空间 < 记录大小"、或"聚簇页接近满且是顺序插入模式（主动预留更新空间）"；删除时"删完数据量低于 merge 阈值（可能要合并）"；更新时"新记录放不下 / 页面太空 / 含 extern 字段"。**乐观失败的成本**：mtr 提交、释放锁、按 `BTR_MODIFY_TREE` 重新搜索——一次失败的乐观尝试约等于一次完整的额外下钻。

#### 4. 分裂策略：50% 填充与顺序插入感知的权衡

页分裂有两个目标：a) 分裂后两半都能装下各自记录（正确性）；b) 为未来插入留空间（性能）。8.0.39 的规则：

- 默认取**记录数中点**（`page_get_middle_rec`）对半分；
- **顺序插入感知**：若 `PAGE_LAST_INSERT` 显示上次插入就在本次插入点之前（持续右追加，自增主键场景），`btr_page_get_split_rec_to_right` 直接把"几乎所有记录"分走（新页只装新 tuple），给顺序流留满整页；倒序同理（`to_left`）。理由：50/50 分裂对单调插入是灾难——每次分裂只写一半页，页利用率 ~50%，分裂频率翻倍；
- 二次迭代（压缩页分裂后仍放不下）用 `btr_page_get_split_rec` 按**字节空间 50%** 分割（累计数据量过半即分裂点）。

**代价**：顺序插入优化的副作用是——页几乎全空时就开始"预留"分裂（乐观插入提前失败），以及分裂后旧页利用率极低（可能 < 50%）。而 bulk load 100% 打包的页，构建完成后**任意一条插入立即触发分裂**（bitmap free bits 被清零，声明"此页已满"，连 change buffer 都无法缓冲）。三种填充策略并存，是"写放大"与"空间浪费"之间摇摆的历史结果。

#### 5. merge_threshold：合并不是对称的

分裂是"马上发生"（放不下就必须做），合并是**懒**的：删除后若页数据量 < `merge_threshold`（默认 50，持久化在 `SYS_INDEXES.MERGE_THRESHOLD`，即 `dict_index_t::merge_threshold`），下次悲观删除才顺手 `btr_compress`。合并不对称的深层原因：

- 删除往往只是 delete-mark，物理删除由 purge 后台做——purge 不急，可以慢慢合并；
- 合并需要 X 锁左右兄弟页并搬移全部记录，成本高且收益不确定（可能马上又插入）；
- 分裂必须即时是因为它影响**可用性**（写不进去），合并只影响**空间与扫描性能**（页变空、树变宽）。

**代价**：大量删除后树可能长期处于"半空"状态（每页 < 50% 使用率），范围扫描要读更多页；`OPTIMIZE TABLE` 重建索引才能收缩。

#### 6. AHI：为点查引入的"有偏记忆"，纯启发式

> **★ AHI 的完整剖析见专篇 [`ahi.md`](ahi.md)**——哈希键的**前缀长度自适应算法**、查询路径 8 道门禁、建/删/改的 nowait 哲学、**AHI 是"允许有瑕疵、惰性修补"的结构**、★ 分片按 (space_id, index_id) 路由故**单索引热点仍争用同一把 latch**、关闭 AHI 可能 600 秒后 crash。
>
> 本节只保留它在 B-tree 搜索路径中的位置。

AHI 解决的问题：B+ 树点查需要 `height` 次 page fetch + 每页目录槽二分；如果 buffer pool 够大，页都在内存里，那树结构本身成了唯一开销。AHI 把"叶子页上的具体记录"按**键前缀 fold** 直接哈希，一次哈希 + 一次叶子页 latch 定位。

设计的精髓是**只做稳赚的事，一旦有代价立即放弃**：

- 统计全程**无锁脏读**（"NOT protected by any semaphore, to save CPU time"），允许不一致，持锁后再复检；
- 构建门槛双条件：索引级连续 100 次潜在命中（`BTR_SEARCH_BUILD_LIMIT`）+ 页级 `n_hash_helps > n_recs / 16`；
- 所有 nowait 加锁路径竞争即放弃（"The AHI is supposed to be heuristic for speed-up. When adding a block to index, waiting here for the latch would defy the purpose"）；
- `last_hash_succ` 门控：上次哈希失败就不尝试下一次；
- 前缀长度/方向（`left_side`）由真实搜索的 `up_match/low_match` **自适应学习**。

**代价与退化场景**：写密集负载下维护成本（每次 insert/delete/split 都要同步哈希）超过收益；顺序扫描每次探测都 miss，白付 fold 开销；超大 working set 下记录指针随页淘汰而快速失效。官方手册因此建议"只读/点查为主且 buffer pool 足够大时才开"。8.0.30 的 sharded latch 把写锁竞争摊薄到 1–512 个分区，才让默认开启在 OLTP 下普遍可接受。

#### 7. 8.0 的 SMO 锁演进：X → SX + 路径预测裁剪

5.7 里 `BTR_MODIFY_TREE` 对整棵树加 **X 锁**——任何一次分裂都排他整棵树，读到该索引的任何并发操作全部阻塞。8.0 的双重优化（详见「搜索」一节）：

1. **SX 树锁**：SX 与 S 兼容（普通读/乐观写继续放行）、与 SX/X 互斥（SMO 之间仍串行）；
2. **路径预测裁剪**：`btr_cur_will_modify_tree` 在爬树时**预估**这条路径是否真会触发 SMO——不会就提前释放上层页锁（普通页只 pin 不 X 锁）；`btr_cur_need_opposite_intention` 发现"删除意图却落在页首/页尾（会在父层引发 node pointer 变更）"时，释放全部锁、意图升级为 `BTR_INTENTION_BOTH` 从 root 重搜。

这是对 B-link 树思想的"廉价回敬"：不引入兄弟路由，而是用**更细的锁 + 更聪明的预测**让多数 SMO 不再阻塞全树。

### 理论溯源

- **B-tree**（Bayer & McCreight, 1972）与 **B+ tree**（所有记录在叶子、非叶只存路由）：IOT 与 node pointer 结构直接对应。聚簇索引叶子带全行、二级索引叶子带（键, 主键）是 B+ 树的数据库变体。
- **B-link tree**（Lehman & Yao, 1981）：见上——InnoDB 明确否决（文件头注释 "not a B-link tree"），但保留了兄弟链用于扫描。这个"否决"就是设计的另一半。
- **Latch coupling（crabbing）**：搜索下钻时先锁子页再放父页（`tree_savepoints`/`mtr_release_block_at_savepoint`），来自 B-tree 并发控制经典技术；InnoDB 的变体是"乐观修改只 X 锁叶子、路径页 S 锁即放"，以及 8.0 的"MODIFY_TREE 下路径页先 pin 后按预测升级 X"。
- **ARIES 生理日志（physiological logging）**：`MLOG_LIST_END_COPY`/`MLOG_LIST_END_DELETE`（列表级搬移只记 2 字节起始偏移，恢复端 `page_parse_delete_rec_list` 按 index 定义重建）、`MLOG_PAGE_REORGANIZE`（重整不记内容、恢复端按 index 重排）都是"物理页 + 逻辑结构"混合的生理日志，来自 ARIES 的生理 redo 思想——用"结构可推导"换日志体积。
- **顺序插入感知分裂**：利用 `PAGE_LAST_INSERT` 识别追加模式，是 B-tree 领域 "insert pattern detection" 的工程实践；在无 fill factor 变量的社区版里，它就是唯一的"为未来留空间"机制。
- **自适应哈希**：把 hash 当 B-tree 的**二级索引**（对索引键做哈希再定位叶子页），借鉴了内存数据库 hash 索引思想，但以"可丢弃"为前提——AHI 是纯缓存，丢了一致性无损，所以一切维护都能 nowait 放弃。

### 算法与数据结构

| 结构/算法 | 说明与复杂度 |
|-----------|-------------|
| 页内定位 | 目录槽二分 + 槽内线性扫（`page_cur_search_with_match`）；约 O(log₂(slots) + slot_size) |
| 树搜索 | 自 root 逐层下钻，O(tree_height) 次页访问；`BTR_MAX_LEVELS=100`、`BTR_MAX_NODE_LEVEL=45`（ASAN 栈限制下调） |
| 插入 | 乐观 O(1) 定位 + 页内插入；分裂 O(页记录数)（搬移半页 + 递归父层） |
| node pointer | "子页首条记录 key + 4 字节页号"（`dict_index_build_node_ptr`，尾字段 `REC_NODE_PTR_SIZE=4`）；最左子页的 node pointer 打 `REC_INFO_MIN_REC_FLAG` |
| 批量构建 | 自底向上：叶子满页→向上插 node pointer，逐层生长；页目录槽在 `finish()` 时 O(n) 均分（每 4 条一槽），**零分裂、零搜索、零 redo** |

### 他库对比与演进动机

| 维度 | InnoDB | PostgreSQL (nbtree) | 其他 |
|------|--------|--------------------|----|
| 并发 SMO | 普通 B-tree + SX 树锁 + 兄弟页 X 锁 | Lehman-Yao B-link（右链 + high key，读不阻塞 SMO） | Oracle：index branch block + ITL（undo 驱动并发），本质也是 B*-tree 变体 |
| 页分裂 | 顺序插入感知 + 50% | 90/10 rightmost split（快速分裂，社区 11+） | SQLite（WAL 下 btree）：单写者，无并发 SMO 问题 |
| 页回收 | delete-mark + purge + merge_threshold 懒合并 | page deletion + vacuum 回收 | — |
| 批量建索引 | 排序 + 100% 打包 + no-redo（5.6 起） | 排序 + deduplicate（13+ 的索引去重） | — |
| 点查加速 | AHI（自适应、可丢弃） | 无（靠 os 页缓存 + 目录缓存） | — |

**演进动机**：B-tree 结构本身 5.6 之后趋于稳定，8.0 的演进集中在**并发**（X→SX、AHI 分片）与**DDL 性能**（并行构建）。AHI 分片的直接动因是 8.0.30 release notes 记载的："Enabling the adaptive hash index on a high-concurrency instance caused temporary AHI search latch contention"——单一 search_latch 成为高并发点查的瓶颈，拆成 8 个分区后写锁竞争下降一个量级。

---

## 核心实现

### 主链路

先把四条 DML 主链路摆出来，后续各节展开细节（函数名均为 8.0.39 实际符号）：

```
【读取/定位】 row_search_mvcc (row0sel.cc)
  → btr_pcur_t::open_no_init（定位）/ open_at_side（全扫起点）
    → btr_cur_search_to_nth_level
      → [AHI 捷径] btr_search_guess_on_hash（命中则直接返回）
      → mtr_s_lock(index->lock) → buf_page_get_gen 逐层下钻
        → page_cur_search_with_match(_bytes)  页内目录槽二分
      → 到达叶子：释放 index S 锁与上层页锁（latch coupling 收尾）
      → btr_search_info_update（AHI 构建决策）

【插入】 row_ins_clust_index_entry_low / row_ins_sec_index_entry (row0ins.cc)
  → btr_cur_search_to_nth_level(PAGE_CUR_LE, BTR_MODIFY_LEAF|BTR_INSERT)
  → btr_cur_optimistic_insert                      // 页内放得下
  └─ DB_FAIL → mtr 重启，BTR_MODIFY_TREE|BTR_INSERT 重搜
      → btr_cur_pessimistic_insert
        → fsp_reserve_free_extents(tree_height/16+3)   // SMO 不可逆，先留空间
        → 游标在根页 ? btr_root_raise_and_insert : btr_page_split_and_insert
            → btr_attach_half_pages（先插父层 node pointer，可能递归分裂）
            → page_move_rec_list_end/start（搬半页记录，列表级 redo）
            → btr_insert_on_non_leaf_level（递归到父层）

【删除】 purge/undo 物理删除 (row0purge.cc / row0uins.cc)
  → 搜索(BTR_MODIFY_LEAF|BTR_DELETE 或 BTR_MODIFY_TREE|BTR_LATCH_FOR_DELETE)
  → btr_cur_optimistic_delete（无 extern 字段且删完不低于 merge 阈值）
  └─ false → btr_cur_pessimistic_delete
      ├─ extern 字段 → lob::BtrContext::free_externally_stored_fields（多 mtr）
      ├─ 页只剩 1 条记录且非根 → btr_discard_page（整页丢弃）
      ├─ 删除非叶层最左 node pointer → btr_node_ptr_delete + btr_insert_on_non_leaf_level
      └─ btr_cur_compress_if_useful → btr_compress（合并 / btr_lift_page_up 降高）

【更新】 row_upd_clust_rec / row_upd_sec_index_entry (row0upd.cc)
  → 字段大小不变 → btr_cur_update_in_place（原地覆写）
  → 否则 → btr_cur_optimistic_update（先删后插同页）
  └─ DB_OVERFLOW/UNDERFLOW/ZIP_OVERFLOW → mtr 重启，BTR_MODIFY_TREE 重搜
      → btr_cur_pessimistic_update（先删旧、同页插不上再悲观插入，可能分裂）
```

### 游标体系：`btr_cur_t` 与 `btr_pcur_t`

#### `btr_cur_t`：mtr 生命周期内的树游标

```cpp
struct btr_cur_t {
  dict_index_t *index{nullptr};      // 所在索引
  page_cur_t page_cur;               // 页内游标：(block, rec) 二元组
  purge_node_t *purge_node{nullptr}; // BTR_DELETE（purge）专用
  buf_block_t *left_block{nullptr};  // SEARCH_PREV/MODIFY_PREV 时已锁的左兄弟

  que_thr_t *thr{nullptr};           // ibuf 操作需要的事务上下文

  btr_cur_method flag{BTR_CUR_UNSET}; // 本次搜索方式：HASH/BINARY/INSERT_TO_IBUF...
  ulint tree_height{0};              // 悲观操作用：预留 extent 的输入
  ulint up_match{0}, up_bytes{0};    // 与右邻用户记录匹配的字段数/字节数
  ulint low_match{0}, low_bytes{0};  // 与左邻匹配（仅 PAGE_CUR_LE 有意义）
  struct {                           // AHI 搜索状态
    btr_search_prefix_info_t prefix_info{};
    uint64_t ahi_hash_value{0};
  } ahi;

  btr_path_t *path_arr{nullptr};     // BTR_ESTIMATE（行数估算）路径数组
  rtr_info_t *rtr_info{nullptr};     // R-tree 搜索状态
  bool m_own_rtr_info = true;
  Page_fetch m_fetch_mode{Page_fetch::NORMAL};  // SCAN 时不污染 LRU
};
```

关键语义：

- **生命周期 = 页被 latch 期间**。`page_cur.rec` 指向的页由 mtr 锁住，mtr 提交后记录指针即悬空。需要跨 mtr 存活的定位一律用 `btr_pcur_t`。
- `up_match/low_match` 是**搜索的副产品**：`PAGE_CUR_LE` 后游标停在"≤ key 的最后一条"，`low_match` 是它与 key 的匹配字段数——`row_ins_duplicate_error_in_clust` 靠 `up_match >= n_uniq` 判断主键重复、靠 low/up 组合定位重复记录。头文件明确警告：跨叶子页边界时这些值可能偏大（邻页记录没真正比较过）。
- `up_bytes/low_bytes` 只有叶子层 + AHI 开启时由 `page_cur_search_with_match_bytes` 填充，搜索结束即无定义——它们喂给 AHI 的前缀长度学习。
- `tree_height` 只在悲观搜索时有效（`root_height + 1`），是 `fsp_reserve_free_extents` 预留页数的输入。

#### `btr_pcur_t`：跨 mtr 的持久游标

```cpp
struct btr_pcur_t {
  btr_cur_t m_btr_cur;               // 组合（值包含）普通游标

  ulint m_latch_mode{0};             // 当前持有的 latch 模式；BTR_NO_LATCHES=已 detach
  bool m_old_stored{false};
  rec_t *m_old_rec{nullptr};         // 位置快照：记录排序前缀的私有副本
  ulint m_old_n_fields{0};           // old_rec 字段数（= n_unique_in_tree）
  btr_pcur_pos_t m_rel_pos{BTR_PCUR_UNSET};  // ON/BEFORE/AFTER/空树两端
  buf::Block_hint m_block_when_stored;       // 保存位置时的 block 指针 hint
  uint64_t m_modify_clock{0};        // 保存位置时的页修改时钟
  pcur_pos_t m_pos_state{BTR_PCUR_NOT_POSITIONED};
  page_cur_mode_t m_search_mode{PAGE_CUR_UNSUPP};
  byte *m_old_rec_buf{nullptr};      // old_rec 的动态缓冲
  size_t m_buf_size{0};
  ulint m_read_level{0};
  import_ctx_t *import_ctx{nullptr}; // 仅 IMPORT TABLESPACE 使用
};
```

**保存**（`store_position`）逐行解析：

```cpp
void btr_pcur_t::store_position(mtr_t *mtr) {
  ut_ad(m_pos_state == BTR_PCUR_IS_POSITIONED);   // 只能保存"已定位"的游标
  ut_ad(m_latch_mode != BTR_NO_LATCHES);

  auto block = get_block();
  auto index = get_btr_cur()->index;
  auto rec = page_cur_get_rec(get_page_cur());    // 当前游标指向的记录
  auto page = page_align(rec);
  auto offs = page_offset(rec);

  if (page_is_empty(page)) {
    // 空索引树：不存 modify_clock（恢复时总是重搜），只记"树首/树尾"
    m_old_stored = true;
    if (page_rec_is_supremum_low(offs)) m_rel_pos = BTR_PCUR_AFTER_LAST_IN_TREE;
    else                                m_rel_pos = BTR_PCUR_BEFORE_FIRST_IN_TREE;
    return;
  }

  // 把"哨兵记录"归一化到相邻的用户记录 + 相对位置：
  if (page_rec_is_supremum_low(offs)) {      // 游标在 supremum（页尾之后）
    rec = page_rec_get_prev(rec);            // 退到本页最后一条用户记录
    m_rel_pos = BTR_PCUR_AFTER;              // 记为"在那条之后"
  } else if (page_rec_is_infimum_low(offs)) {// 游标在 infimum（页首之前）
    rec = page_rec_get_next(rec);            // 进到本页第一条用户记录
    m_rel_pos = BTR_PCUR_BEFORE;             // 记为"在那条之前"
  } else {
    m_rel_pos = BTR_PCUR_ON;                 // 正停在一条用户记录上
  }

  m_old_stored = true;
  // ★ 拷贝记录"排序前缀"（前 n_unique_in_tree 个字段）到私有缓冲区——这是悲观重搜的 key
  m_old_rec = dict_index_copy_rec_order_prefix(index, rec, &m_old_n_fields,
                                               &m_old_rec_buf, &m_buf_size);
  m_block_when_stored.store(block);          // block 指针 hint（乐观恢复直接取用）
  m_modify_clock = block->get_modify_clock(...);  // 页修改时钟快照
}
```

逐行解释：

- **位置快照的三要素**：`m_old_rec`（记录排序前缀副本，供悲观重搜当 key）、`m_rel_pos`（ON/BEFORE/AFTER，供恢复后微调）、`m_block_when_stored` + `m_modify_clock`（供乐观恢复验证）。
- **哨兵归一化**：游标不会停在 infimum/supremum 上保存——它们不是用户记录，无法做"排序前缀"。所以 supremum 上的位置退到前一条用户记录记 `AFTER`，infimum 上则进到下一条记 `BEFORE`。这样快照永远锚定在**一条真实用户记录**的相对位置。
- `m_modify_clock` 是页的"修改时钟"：任何可能使记录指针失效的操作（删除、重整、分裂）都会递增它。乐观恢复就靠"clock 没变 ⇒ 页内布局没变 ⇒ 记录指针仍有效"。

**恢复**（`restore_position`）逐行解析：

```cpp
bool btr_pcur_t::restore_position(ulint latch_mode, mtr_t *mtr, ut::Location location) {
  ut_ad(mtr->is_active());
  ut_ad(m_old_stored);      // 必须先 store_position
  ut_ad(is_positioned());

  auto index = get_btr_cur()->index;

  // 空树两端：不尝试乐观恢复，直接定位到索引端点
  if (m_rel_pos == BTR_PCUR_AFTER_LAST_IN_TREE || m_rel_pos == BTR_PCUR_BEFORE_FIRST_IN_TREE) {
    btr_cur_open_at_index_side(m_rel_pos == BTR_PCUR_BEFORE_FIRST_IN_TREE,
                               index, latch_mode, get_btr_cur(), m_read_level, ..., mtr);
    m_latch_mode = BTR_LATCH_MODE_WITHOUT_INTENTION(latch_mode);
    m_pos_state = BTR_PCUR_IS_POSITIONED;
    m_block_when_stored.clear();
    return false;
  }

  ut_a(m_old_rec != nullptr);
  ut_a(m_old_n_fields > 0);

  // ── 乐观恢复 ──
  if ((latch_mode == BTR_SEARCH_LEAF || latch_mode == BTR_MODIFY_LEAF ||
       latch_mode == BTR_SEARCH_PREV || latch_mode == BTR_MODIFY_PREV) &&
      !m_btr_cur.index->table->is_intrinsic()) {
    if (m_block_when_stored.run_with_hint([&](buf_block_t *hint) {
          return hint != nullptr &&
                 btr_cur_optimistic_latch_leaves(hint, m_modify_clock, &latch_mode,
                                                 &m_btr_cur, ..., mtr);
        })) {
      // block 还在、clock 未变 → 直接加锁，记录指针复用，零搜索
      m_pos_state = BTR_PCUR_IS_POSITIONED;
      m_latch_mode = latch_mode;
      if (m_rel_pos == BTR_PCUR_ON) {
        // 调试下用 cmp_rec_rec 校验 old_rec 与当前记录完全一致，返回 true（精确恢复）
        return true;
      }
      // BEFORE/AFTER：回到同一物理记录，但相对位置需按方向微调
      if (is_on_user_rec()) m_pos_state = BTR_PCUR_IS_POSITIONED_OPTIMISTIC;
      return false;
    }
  }

  // ── 悲观恢复 ──
  auto heap = mem_heap_create(256, ...);
  tuple = dict_index_build_data_tuple(index, m_old_rec, m_old_n_fields, heap);  // 用前缀重建 key

  switch (m_rel_pos) {                       // rel_pos → 搜索模式
    case BTR_PCUR_ON:    mode = PAGE_CUR_LE; break;
    case BTR_PCUR_AFTER: mode = PAGE_CUR_G;  break;
    case BTR_PCUR_BEFORE:mode = PAGE_CUR_L;  break;
    default: ut_error;
  }
  open_no_init(index, tuple, mode, latch_mode, 0, mtr, location);   // 从 root 重搜

  if (m_rel_pos == BTR_PCUR_ON && is_on_user_rec() &&
      !cmp_dtuple_rec(tuple, get_rec(), index, ...)) {
    // 重搜后停在同一排序键的记录上：刷新 clock 快照（可能落在新页），返回 true
    auto block = get_block();
    m_block_when_stored.store(block);
    m_modify_clock = block->get_modify_clock(...);
    m_old_stored = true;
    return true;
  }

  // 记录已被删/改，或落到了别的页：用 store_position 重新压快照，返回 false
  store_position(mtr);
  return false;
}
```

逐行解释：

- **返回值语义**（`true` = "恢复在排序前缀完全相同的用户记录上"）：调用方（`row_search_mvcc` 的 `need_to_process`）据此决定是否要重新处理当前记录。
- **乐观恢复的代价是零**：`run_with_hint` 取出保存的 block 指针，`btr_cur_optimistic_latch_leaves` 先检查 `modify_clock` 没变、状态仍是 `BUF_BLOCK_FILE_PAGE`，再 `buf_page_optimistic_get` 直接加锁——不碰 B-tree。`BTR_PCUR_ON` 且校验通过时精确恢复。
- **悲观恢复天然容忍一切结构变化**：`PAGE_CUR_LE` 重搜在"记录已被删除"时落在 ≤ 旧 key 的最后一条（最近的合法位置）；页分裂/合并/淘汰完全无需特判（重搜走当前树）。`rel_pos` 决定用哪个模式：`ON→LE`（找原记录）、`AFTER→G`（找原记录之后第一条）、`BEFORE→L`（找原记录之前最后一条）。
- **只有 `BTR_SEARCH/MODIFY_LEAF(_PREV)` 才走乐观**：SMO 的 `BTR_MODIFY_TREE` 等模式直接重搜——因为那些场景本来就要完整下钻加路径锁。

**跨页推进的不对称**（`move_to_next_page` vs `move_backward_from_page`）：

- **向前**（`move_to_next_page`）在**同一 mtr** 内完成：读 `FIL_PAGE_NEXT` 得右页号 → `btr_block_get` 锁右页 → 释放当前页 → 置 `before_first`。先锁右再放左，符合左→右锁序，无需重开 mtr。
- **向后**（`move_backward_from_page`）必须 **`mtr_commit` + `mtr_start`**：按左→右锁序**不能**先持右页锁再去锁左页（会与"从左往右扫描"的线程死锁）。所以先 `store_position` 留快照 → 提交 mtr 释放当前页锁 → 新 mtr 里 `restore_position(BTR_SEARCH_PREV/MODIFY_PREV)`，乐观路径 `btr_cur_optimistic_latch_leaves` 会**先锁左邻居再锁当前页**（源码注释 "latch order, latch prev page first"），恢复后释放多余的左页锁。这是 InnoDB 死锁预防的经典细节：**加锁顺序永远是自顶向下、自左向右**，向后跨页宁可多一次 mtr 提交也要维持它。

### 搜索：`btr_cur_search_to_nth_level`（逐行解析）

这是 InnoDB 被调用最频繁、最核心的函数（btr0cur.cc 629~1863，约 1200 行）。点查、范围扫描、插入、删除、更新的定位都从它开始。下面按**执行顺序**逐段贴出源码并逐行解释；DEBUG 断言、R-tree 专用分支、编译开关分支会省略并标注，主链路一字不漏。

#### 签名与路径追踪数组（629~716）

```cpp
void btr_cur_search_to_nth_level(
    dict_index_t *index,     // 目标索引
    ulint level,             // 目标层级（0=叶子；SMO 中找父页会传 >0）
    const dtuple_t *tuple,   // 搜索键。n_fields_cmp 必须裁到不比较 node_ptr 的页号字段
    page_cur_mode_t mode,    // 叶子层定位模式 PAGE_CUR_L/LE/G/GE；插入一律 PAGE_CUR_LE
    ulint latch_mode,        // btr_latch_mode OR 一个操作标志（BTR_INSERT/DELETE_MARK/DELETE/ESTIMATE）
    btr_cur_t *cursor,       // in/out：搜索完成后定位到目标记录前驱的游标
    ulint has_search_latch,  // 调用者已持 AHI S 锁？RW_S_LATCH 或 0
    const char *file, ulint line,
    mtr_t *mtr)              // 持有所有 latch 的 mini-transaction
{
  page_t *page = nullptr;
  buf_block_t *block;
  ulint height;            // 当前所在层级（root 层=树高，0=叶子）；ULINT_UNDEFINED=还未到 root
  ulint up_match, up_bytes, low_match, low_bytes;   // 与右/左邻记录的匹配字段数、字节数
  ulint savepoint;         // index->lock 压入 mtr 时的 memo 栈水位
  ulint rw_latch;          // 传给 buf_page_get_gen 的页锁类型
  page_cur_mode_t page_mode;
  page_cur_mode_t search_mode = PAGE_CUR_UNSUPP;
  Page_fetch fetch;        // 页面获取策略（NORMAL / IF_IN_POOL / IF_IN_POOL_OR_WATCH / SCAN）
  ulint node_ptr_max_size = UNIV_PAGE_SIZE / 2;  // 非叶层 node pointer 的尺寸上限
  page_cur_t *page_cursor;
  btr_op_t btr_op;         // BTR_NO_OP / INSERT_OP / DELETE_OP / DELMARK_OP ...
  ulint root_height = 0;
  ulint upper_rw_latch, root_leaf_rw_latch;   // 非叶页统一锁类型 / root 兼叶时的锁类型
  btr_intention_t lock_intention;             // BTR_INTENTION_INSERT / DELETE / BOTH
  bool modify_external;                       // BTR_MODIFY_EXTERNAL：要动 LOB 外存页

  buf_block_t *tree_blocks[BTR_MAX_LEVELS];   // 搜索路径上每层页面（BTR_MAX_LEVELS=100）
  ulint tree_savepoints[BTR_MAX_LEVELS];      // 每层页面在 mtr memo 栈中的水位
  ulint n_blocks = 0;    // 已入栈的路径页面数
  ulint n_releases = 0;  // 已释放的路径页面数（< n_blocks 的前缀已释放 latch）

  bool detected_same_key_root = false;   // 非唯一索引 + 键命中页首/页尾 → 不能放父页锁
  bool retrying_for_search_prev = false; // BTR_SEARCH_PREV/MODIFY_PREV 的回溯重试标志
  ulint leftmost_from_level = 0;         // 从哪一层起需要锁定 prev 兄弟页
  buf_block_t **prev_tree_blocks = nullptr;    // 回溯时 prev 兄弟页路径
  ulint *prev_tree_savepoints = nullptr;
  ulint prev_n_blocks = 0, prev_n_releases = 0;
```

逐行解释：

- `tree_blocks[]`/`tree_savepoints[]` 是**搜索路径的栈**：每下钻一层就把该层页面 push 进去，`savepoints[i]` 记录"页面 i 压入 mtr 时 memo 栈的大小"——这是 `mtr_release_block_at_savepoint` 能"精确回滚到某一页"的依据（latch coupling 的物理基础）。
- `n_blocks` 是"已 push 数量"，`n_releases` 是"已释放数量"，两者之间的页面仍持有 latch。这个双游标设计让"走到哪放到哪"成为可能：一旦判定上层页不会再被 SMO 触碰，就把 `n_releases` 推进、释放那批页。
- `node_ptr_max_size` 初值 `UNIV_PAGE_SIZE/2`，只有当 `root_leaf_rw_latch == RW_X_LATCH`（即要做悲观修改）时才换成 `dict_index_node_ptr_max_size(index)`——它喂给 `btr_cur_will_modify_tree` 判断"这层还能塞下几条 node pointer"。

#### 操作标志解析与 AHI 捷径（792~889）

```cpp
  switch (UNIV_EXPECT(latch_mode & (BTR_INSERT | BTR_DELETE | BTR_DELETE_MARK), 0)) {
    case 0:                  btr_op = BTR_NO_OP;    break;
    case BTR_INSERT:         btr_op = (latch_mode & BTR_IGNORE_SEC_UNIQUE)
                                          ? BTR_INSERT_IGNORE_UNIQUE_OP : BTR_INSERT_OP; break;
    case BTR_DELETE:         btr_op = BTR_DELETE_OP;  ut_a(cursor->purge_node); break;
    case BTR_DELETE_MARK:    btr_op = BTR_DELMARK_OP; break;
    default: ut_error;  // 三个操作标志互斥，只能有其一
  }
  // 以下四类索引都不能走 change buffer（ibuf 树/聚簇/临时表/spatial）
  ut_ad(btr_op == BTR_NO_OP || !dict_index_is_ibuf(index));
  ut_ad(btr_op == BTR_NO_OP || !index->is_clustered());
  ut_ad(btr_op == BTR_NO_OP || !index->table->is_temporary());
  ut_ad(btr_op == BTR_NO_OP || !dict_index_is_spatial(index));

  auto estimate = latch_mode & BTR_ESTIMATE;   // 优化器行数估算
  lock_intention = btr_cur_get_and_clear_intention(&latch_mode);  // 剥出 LATCH_FOR_INSERT/DELETE
  modify_external = latch_mode & BTR_MODIFY_EXTERNAL;
  latch_mode = BTR_LATCH_MODE_WITHOUT_FLAGS(latch_mode);   // 现在 latch_mode 只剩纯 btr_latch_mode

  cursor->flag = BTR_CUR_BINARY;
  cursor->index = index;

  // AHI 捷径门槛（8 个条件同时成立才尝试）：
  if (rw_lock_get_writer(btr_get_search_latch(index)) == RW_LOCK_NOT_LOCKED &&  // AHI 未被独占
      latch_mode <= BTR_MODIFY_LEAF && index->search_info->last_hash_succ &&   // 非 SMO + 上次哈希成功
      !index->disable_ahi && !estimate && !dict_index_is_spatial(index) &&
      UNIV_LIKELY(btr_search_enabled) && !modify_external &&
      btr_search_guess_on_hash(tuple, mode, latch_mode, cursor, has_search_latch, mtr)) {
    btr_cur_n_sea++;   // AHI 命中，直接返回，连 root 都不碰
    return;
  }
  btr_cur_n_non_sea++;

  if (has_search_latch) {
    rw_lock_s_unlock(btr_get_search_latch(index));  // 放弃 AHI 走 B-tree：先还 AHI 锁，遵守加锁序
  }
  savepoint = mtr_set_savepoint(mtr);   // 记录 index->lock 压栈前的水位
```

逐行解释：

- `btr_op` 决定**叶子页不在 buffer pool 时能否走 change buffer**：只有二级索引的 INSERT/DELETE_MARK/DELETE 才允许（聚簇/ibuf/临时/spatial 全被断言排除）。`BTR_IGNORE_SEC_UNIQUE` 是唯一性检查的豁免——事务明确不做二级唯一性检查时，即使唯一索引也允许缓冲插入。
- **AHI 捷径是"白嫖"路径**：8 个条件缺一不可，其中 `last_hash_succ` 是上一次哈希是否成功的记忆——连续失败就永远走 B-tree，避免每次都白付 fold + S 锁开销。命中后 `btr_cur_n_sea++` 直接返回，这是 `SHOW ENGINE INNODB STATUS` 里 "hash searches/s" 的来源。
- `savepoint = mtr_set_savepoint(mtr)` 在**加 index 锁之前**记录水位——到叶子后要按这个水位把 index S 锁精确释放掉，不误伤叶子页锁。

#### index->lock 获取与 lock intention（892~968）

```cpp
  switch (latch_mode) {
    case BTR_MODIFY_TREE: /* SMO */
      if (lock_intention == BTR_INTENTION_DELETE &&
          trx_sys->rseg_history_len.load() > BTR_CUR_FINE_HISTORY_LENGTH &&   // 100000
          buf_get_n_pending_read_ios()) {
        mtr_x_lock(dict_index_get_lock(index), mtr, ...);   // purge 高压：给 IO 让路，X 锁
      } else if (dict_index_is_spatial(index) && lock_intention <= BTR_INTENTION_BOTH) {
        mtr_x_lock(dict_index_get_lock(index), mtr, ...);   // spatial 删除可能向上锁父页，X 锁
      } else {
        mtr_sx_lock(dict_index_get_lock(index), mtr, ...);  // ★ 默认：SX（与 S 兼容，放行并发读）
      }
      upper_rw_latch = RW_X_LATCH;   // SMO 时非叶页统一用 X（但 MODIFY_TREE 下先只 pin，见后）
      break;
    case BTR_CONT_MODIFY_TREE:
    case BTR_CONT_SEARCH_TREE:
      /* Do nothing：假定 mtr 里已有 index 的 X/SX */
      upper_rw_latch = RW_NO_LATCH;
      break;
    default: /* BTR_SEARCH_LEAF / BTR_MODIFY_LEAF / BTR_SEARCH_TREE ... */
      if (!srv_read_only_mode) {
        if (s_latch_by_caller) {
          /* BTR_ALREADY_S_LATCHED：调用者已持 S/SX，跳过 */
        } else if (!modify_external) {
          mtr_s_lock(dict_index_get_lock(index), mtr, ...);   // 普通读/乐观写：index 加 S
        } else {
          mtr_sx_lock(dict_index_get_lock(index), mtr, ...);  // 要动 LOB 外存页：SX 排斥其他 SX
        }
        upper_rw_latch = RW_S_LATCH;
      } else {
        upper_rw_latch = RW_NO_LATCH;   // 只读模式：不加锁
      }
  }
  root_leaf_rw_latch = btr_cur_latch_for_root_leaf(latch_mode);
```

逐行解释：

- **`index->lock` 是整棵树的 rw-lock**，三种拿法直接决定并发度：普通读/乐观写拿 **S**（彼此兼容，只与 SMO 的 SX/X 互斥）；SMO 默认拿 **SX**（与 S 兼容、与 SX/X 互斥）——这是 8.0 相对 5.7 的最大改动，5.7 的 SMO 拿 X 会排他整棵树。
- 两个 X 例外都是"让路"语义：purge 在 history list > 100000 且有 pending IO 时 X（把 IO 带宽优先给 purge）；spatial 的删除意图 X（R-tree 会向上锁父页，SX 不够）。
- `BTR_MODIFY_EXTERNAL`（LOB 操作）拿 SX 而非 S，因为要分配/释放外存页，必须排斥其他 SMO 与同类操作。
- `upper_rw_latch` 是"非叶页的统一锁类型"：普通路径 S、SMO 路径 X。但注意 `BTR_MODIFY_TREE` 下它实际被 `RW_NO_LATCH` 覆盖（见 search_loop），X 只在到达目标层后回补。

#### 搜索模式换算：非叶层 node pointer 的下界语义（977~1031）

```cpp
  height = ULINT_UNDEFINED;   // 尚未到 root

  // leaf 的 mode 与非叶层的 page_mode 不同：node_ptr 的 key 是子页"下界"
  switch (mode) {
    case PAGE_CUR_GE: page_mode = PAGE_CUR_L;  break;   // leaf 找 >=X → 非叶找 <X 的最大 node_ptr
    case PAGE_CUR_G:  page_mode = PAGE_CUR_LE; break;   // leaf 找 >X  → 非叶找 <=X 的最大 node_ptr
    default:          page_mode = mode;        break;   // L/LE 原样
  }
```

逐行解释：这是 B-tree 搜索最经典的坑。node pointer `(key, child_page_no)` 指向 `[key, 下一个 node_ptr 的 key)` 区间，key 是**下界**。搜索 15（`PAGE_CUR_GE`）时，若非叶层也用 GE 会命中 node_ptr=20（指向 `[20,30)`），漏掉 15；换成 `PAGE_CUR_L`（严格小于 15）命中 node_ptr=10，正确定位到 `[10,20)`。到叶子层再恢复原始 `mode`（`page_mode = mode`）。

#### search_loop：页面获取与 change buffer（1039~1153）

```cpp
search_loop:
  fetch = cursor->m_fetch_mode;
  rw_latch = RW_NO_LATCH;
  rtree_parent_modified = false;

  // 决定当前页面的锁类型
  if (height != 0) {                          // 非叶子层
    if ((latch_mode != BTR_MODIFY_TREE || height == level) &&
        !retrying_for_search_prev) {
      // 非 SMO（或已到目标层）：普通页需要显式加锁
      if (modify_external && height == ULINT_UNDEFINED && upper_rw_latch == RW_S_LATCH)
        rw_latch = RW_SX_LATCH;               // LOB 操作 root 页要 SX（fseg 操作）
      else
        rw_latch = upper_rw_latch;            // 普通 S；SMO 下为 X
    }
    // else: MODIFY_TREE 且未到目标层 → rw_latch 保持 RW_NO_LATCH（只 pin 不加锁！）
  } else if (latch_mode <= BTR_MODIFY_LEAF) {
    rw_latch = latch_mode;                    // 叶子层：SEARCH_LEAF→S，MODIFY_LEAF→X
    if (btr_op != BTR_NO_OP && ibuf_should_try(index, btr_op != BTR_INSERT_OP)) {
      // 二级索引 DML + 页不在 pool 时可缓冲 → 降级 fetch
      fetch = btr_op == BTR_DELETE_OP ? Page_fetch::IF_IN_POOL_OR_WATCH
                                      : Page_fetch::IF_IN_POOL;
    }
  }

retry_page_get:
  tree_savepoints[n_blocks] = mtr_set_savepoint(mtr);   // 记录本页压栈水位
  block = buf_page_get_gen(
      page_id, page_size, rw_latch,
      (height == ULINT_UNDEFINED ? index->search_info->root_guess : nullptr),  // root 用缓存 hint
      fetch, {file, line}, mtr);
  tree_blocks[n_blocks] = block;

  if (block == nullptr) {
    // 页不在 buffer pool：且 fetch 允许缓冲 → 走 change buffer 直接返回
    switch (btr_op) {
      case BTR_INSERT_OP:
      case BTR_INSERT_IGNORE_UNIQUE_OP:
        if (ibuf_insert(IBUF_OP_INSERT, tuple, index, page_id, page_size, cursor->thr)) {
          cursor->flag = BTR_CUR_INSERT_TO_IBUF;  goto func_exit;
        }
        break;
      case BTR_DELMARK_OP:
        if (ibuf_insert(IBUF_OP_DELETE_MARK, ...)) {
          cursor->flag = BTR_CUR_DEL_MARK_IBUF;  goto func_exit;
        }
        break;
      case BTR_DELETE_OP:   // purge 物理删除二级索引记录
        if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
          cursor->flag = BTR_CUR_DELETE_REF;   // 记录还被活跃版本引用，不能 purge
        } else if (ibuf_insert(IBUF_OP_DELETE, ...)) {
          cursor->flag = BTR_CUR_DELETE_IBUF;  // 删除已缓冲
        } else {
          buf_pool_watch_unset(page_id);       // 缓冲失败，取消 watch，落到下面重读
          break;
        }
        buf_pool_watch_unset(page_id);
        goto func_exit;
      default:
        ut_error;
    }
    fetch = cursor->m_fetch_mode;   // 缓冲失败 → 恢复普通 fetch 重新拿页
    goto retry_page_get;
  }
```

逐行解释：

- **非叶层的锁策略是理解 8.0 并发的钥匙**：普通路径（`latch_mode != BTR_MODIFY_TREE`）非叶页用 `upper_rw_latch`（S/X）加锁；而 `BTR_MODIFY_TREE` 且**还没到目标层**时，`rw_latch` 保持 `RW_NO_LATCH`——即**只 pin（bufferfix）不加锁**。这是可行的，因为 `index->lock` 的 SX 已经排他所有其他 SMO，普通读者只加 S 读非叶页且到叶即放，不会与这个"准备分裂"的线程争写。路径页真正要 X 锁时，是到达目标层后统一升级（见下文）。
- `root_guess` 是 `index->search_info` 缓存的 root block 指针，传给 `buf_page_get_gen` 作 page hash 查找 hint，省一次 hash 计算。
- **change buffer 集成在搜索里**：当叶子页不在 pool 且 `ibuf_should_try` 判定可以缓冲，`fetch` 降级为 `IF_IN_POOL`（不在池直接返回 null）或 `IF_IN_POOL_OR_WATCH`（purge 删除，同时在 page hash 挂 watch）。返回 null 后按 `btr_op` 分派：插入/删除标记直接 `ibuf_insert` 缓冲；purge 删除先 `row_purge_poss_sec` 确认"记录还能不能 purge"（还要看是否有活跃读视图引用它），能删才缓冲。**任何一条路径成功都 `goto func_exit`，完全不读页**——这就是 change buffer 减少随机 IO 的机制。
- `BTR_CUR_DELETE_REF` 是 purge 的"白旗"：记录还不能删，本轮放弃，游标标记后返回。

#### 到达 root / 到达叶子（1210~1322）

```cpp
  page = buf_block_get_frame(block);

  if (height == ULINT_UNDEFINED && page_is_leaf(page) &&
      rw_latch != RW_NO_LATCH && rw_latch != root_leaf_rw_latch) {
    // 树只有一层（root 即 leaf）：刚才按"非叶"拿了 S/SX，但叶子需要 S/X → 重来
    mtr_release_block_at_savepoint(mtr, tree_savepoints[n_blocks], tree_blocks[n_blocks]);
    upper_rw_latch = root_leaf_rw_latch;
    goto search_loop;
  }

  if (UNIV_UNLIKELY(height == ULINT_UNDEFINED)) {
    // 第一次到达 root
    height = btr_page_get_level(page);   // 读 PAGE_LEVEL = 树高
    root_height = height;
    cursor->tree_height = root_height + 1;   // 悲观操作据此预留 extent
    index->search_info->root_guess = block;  // 缓存 root block
  }

  if (height == 0) {
    if (rw_latch == RW_NO_LATCH) {
      // MODIFY_TREE 路径到这里才补叶子锁（可能一次锁 left/target/right 三页）
      latch_leaves = btr_cur_latch_leaves(block, page_id, page_size, latch_mode, cursor, mtr);
    }
    switch (latch_mode) {
      case BTR_MODIFY_TREE:
      case BTR_CONT_MODIFY_TREE:
      case BTR_CONT_SEARCH_TREE:
        break;   // SMO：保留 index 锁和路径页锁
      default:
        if (!s_latch_by_caller && !srv_read_only_mode && !modify_external) {
          mtr_release_s_latch_at_savepoint(mtr, savepoint, dict_index_get_lock(index));
          // ★ latch coupling 收尾：立即释放 index S 锁
        }
        if (retrying_for_search_prev) { /* 先释放 prev 兄弟页路径 */ }
        for (; n_releases < n_blocks; n_releases++) {
          if (n_releases == 0 && modify_external) continue;  // LOB 操作保留 root 页锁（fseg）
          mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases], tree_blocks[n_releases]);
        }
        // ★ 释放全部上层页锁，只留叶子
    }
    page_mode = mode;   // 恢复叶子层原始搜索模式
  }
```

逐行解释：

- **root 即叶的特判**：树高 0 时，第一次取 root 按"非叶"逻辑可能拿了 S（普通路径）或没拿锁（SMO），但作为叶子它需要与 `latch_mode` 匹配的 S/X。发现锁型不符就释放重取——`root_leaf_rw_latch = btr_cur_latch_for_root_leaf(latch_mode)` 就是为此准备的。
- **`cursor->tree_height = root_height + 1`**：这就是悲观插入 `fsp_reserve_free_extents(tree_height/16+3)` 的输入。树越高，分裂可能递归的层数越多，预留的 extent 越多。
- **latch coupling 收尾**（普通路径）：到叶子后，index S 锁按 `savepoint` 水位释放，所有上层页按 `tree_savepoints[]` 逐个释放——**搜索结束时只持有叶子页锁**。这是"边下边放"（latch coupling / crabbing）的完整落地。
- `modify_external` 保留 root 页锁：LOB 操作还要在 root 的 fseg header 上分配/释放外存页，root 锁不能放。

#### 页内定位分派（1390~1411）

```cpp
  if (height == 0 && btr_search_enabled && !dict_index_is_spatial(index)) {
    // 叶子层 + AHI 开启：记录字节级匹配（喂给 AHI 前缀学习）
    page_cur_search_with_match_bytes(block, index, tuple, page_mode, &up_match,
                                     &up_bytes, &low_match, &low_bytes, page_cursor);
  } else {
    // 非叶层 / 非 AHI 路径：标准字段级二分
    up_bytes = low_bytes = 0;
    page_cur_search_with_match(block, index, tuple, page_mode, &up_match,
                               &low_match, page_cursor,
                               need_path ? cursor->rtr_info : nullptr);
  }
```

逐行解释：两个函数只有"是否记录字节级匹配"的区别。`page_cur_search_with_match` 只回填 `up_match/low_match`（匹配的**完整字段数**）；`page_cur_search_with_match_bytes` 额外回填 `up_bytes/low_bytes`（首个部分匹配字段内匹配的**字节数**）——后者只为 AHI 服务（AHI 需要知道"用几个完整字段 + 下一字段几个字节"做哈希前缀）。`up_bytes/low_bytes` 搜索结束后即无定义，不能给上层用。

#### 下钻、意图升级重搜、SMO 预测（1441~1752）

```cpp
  if (level != height) {
    // 还没到目标层，继续向下
    const rec_t *node_ptr;
    ut_ad(height > 0);
    height--;   // 先递减：height 现在表示"将要进入的层"

    node_ptr = page_cur_get_rec(page_cursor);   // 页内定位命中的那条 node_ptr 记录
    offsets = rec_get_offsets(node_ptr, index, offsets, ULINT_UNDEFINED, ..., &heap);

    // ★ 意图与落点矛盾：删除/插入会波及父层，必须升级意图重搜
    if (latch_mode == BTR_MODIFY_TREE &&
        btr_cur_need_opposite_intention(page, lock_intention, node_ptr)) {
    need_opposite_intention:
      // 释放 root block + 全部路径页
      if (n_releases > 0)
        mtr_release_block_at_savepoint(mtr, tree_savepoints[0], tree_blocks[0]);
      for (; n_releases <= n_blocks; n_releases++)
        mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases], tree_blocks[n_releases]);
      lock_intention = BTR_INTENTION_BOTH;   // 升级：按最坏情况保留所有父页锁
      page_id.reset(space, dict_index_get_page(index));
      up_match = low_match = 0;
      height = ULINT_UNDEFINED;   // 从 root 重新下钻
      n_blocks = n_releases = 0;
      goto search_loop;
    }

    // ★ 非唯一索引 + 键命中页首/页尾：并发同键可能选到别的页，不能放父页锁
    if (!detected_same_key_root && lock_intention == BTR_INTENTION_BOTH &&
        !dict_index_is_unique(index) && latch_mode == BTR_MODIFY_TREE &&
        (up_match >= rec_offs_n_fields(offsets) - 1 ||
         low_match >= rec_offs_n_fields(offsets) - 1)) {
      // 比较 node_ptr 与页首/页尾记录，若 key 相同 → detected_same_key_root = true
      ...
    }

    // ★ SMO 预测裁剪：不会触发 SMO 就提前释放上层页锁
    if (!detected_same_key_root && latch_mode == BTR_MODIFY_TREE &&
        !btr_cur_will_modify_tree(index, page, lock_intention, node_ptr,
                                  node_ptr_max_size, page_size, mtr) &&
        !rtree_parent_modified) {
      for (; n_releases < n_blocks; n_releases++) {
        if (n_releases == 0) continue;   // root 保留 pin（作下次 root_guess 用）
        mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases], tree_blocks[n_releases]);
      }
    }

    // ★ 到达目标层：保留的路径页从"只 pin"升级为 X（root 升级 SX）
    if (height == level && latch_mode == BTR_MODIFY_TREE) {
      if (n_releases > 0)
        mtr_block_sx_latch_at_savepoint(mtr, tree_savepoints[0], tree_blocks[0]);
      for (ulint i = n_releases; i <= n_blocks; i++)
        mtr_block_x_latch_at_savepoint(mtr, tree_savepoints[i], tree_blocks[i]);
    }

    // BTR_SEARCH_PREV/MODIFY_PREV：node_ptr 是页内最左记录且有 prev 页 → 回溯
    if ((latch_mode == BTR_SEARCH_PREV || latch_mode == BTR_MODIFY_PREV) &&
        !retrying_for_search_prev) {
      if (btr_page_get_prev(page, mtr) != FIL_NULL && page_rec_is_first(node_ptr, page)) {
        if (leftmost_from_level == 0) leftmost_from_level = height + 1;
      } else {
        leftmost_from_level = 0;
      }
      if (height == 0 && leftmost_from_level > 0) {
        // 到叶子但发现前驱叶子在别的父页子树里：释放自 leftmost_from_level 的路径，回溯重搜
        retrying_for_search_prev = true;
        ...  // 分配 prev_tree_blocks，释放 n_blocks 后半段，height=leftmost_from_level，重放匹配
        goto search_loop;
      }
    }

    // 下钻到子页
    page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets));
    n_blocks++;
    goto search_loop;
  }
```

逐行解释：

- **`height--` 的时机**：先递减再取 node_ptr，因为 `height` 现在表示"下一层"的层号；node_ptr 的 `child_page_no` 就是下一层的页。
- **意图升级重搜**（`btr_cur_need_opposite_intention`）是 8.0 并发正确性的关键。它回答"只打算插入/删除、却发现落点会在父层引发额外 node pointer 改动"怎么办：释放全部锁、意图升级 `BOTH`、从 root 重来。`BTR_INTENTION_BOTH` 下 `btr_cur_will_modify_tree` 用最坏情况（`max_nodes_deleted = 2^(level-1)`）评估，路径页全部保留——用"多锁几个父页"换"不用再重搜一次"。
- **`detected_same_key_root`** 处理一个微妙竞态：非唯一索引里，两个并发线程搜相同键，可能被页内二分导向**不同的叶子页**（键跨越页边界时）。若此时放掉父页锁，两者都去改各自的叶子，可能死锁（都要回锁对方经过的父页）。所以键与页首/页尾记录匹配时强制保留父页锁。
- **SMO 预测裁剪**（`btr_cur_will_modify_tree`）：这是"MODIFY_TREE 下非叶页只 pin 不加锁"能成立的另一半——如果预测**不会**分裂/合并，就提前释放所有上层页锁（root 保留 pin）；如果**会**，则这些 pin 的页在到达目标层后被 `mtr_block_x_latch_at_savepoint` 升级为 X（root 升 SX，因为 root 带 fseg header）。锁的"延迟获取 + 按需升级"让绝大多数只改叶子不分裂的操作，全程不 X 锁任何非叶页。
- **PREV 回溯**：`BTR_SEARCH_PREV/MODIFY_PREV` 要同时锁叶子及其左兄弟。当某层 node_ptr 是页内最左记录且该页有 prev 页时，左兄弟叶子挂在**另一个父页**的子树里，不在当前路径上——于是记 `leftmost_from_level`、释放自该层的路径页、`retrying_for_search_prev=true` 重来；第二遍在每层额外锁 prev 兄弟页（左→右顺序），避免死锁。

#### 收尾：结果写回与 AHI 统计（1774~1863）

```cpp
  if (level != 0) {
    // 搜到非叶层（SMO 中找父页）：补子页锁
    if (upper_rw_latch == RW_NO_LATCH) {
      buf_block_t *child_block;
      if (latch_mode == BTR_CONT_MODIFY_TREE)
        child_block = btr_block_get(page_id, page_size, RW_X_LATCH, ..., mtr);
      else
        child_block = btr_block_get(page_id, page_size, RW_SX_LATCH, ..., mtr);
      ...
    }
    if (page_mode <= PAGE_CUR_LE) {
      cursor->low_match = low_match;
      cursor->up_match = up_match;
    }
  } else {
    // 叶子层：写回匹配信息 + AHI 构建决策
    cursor->low_match = low_match;
    cursor->low_bytes = low_bytes;
    cursor->up_match = up_match;
    cursor->up_bytes = up_bytes;
    if (btr_search_enabled && !index->disable_ahi) {
      btr_search_info_update(cursor);   // 累计 hash_analysis，达阈值才走慢路径
    }
  }
```

逐行解释：

- `level != 0` 分支服务于 SMO 中"找父页/祖父页"：`BTR_CONT_MODIFY_TREE` 给子页补 X、`BTR_CONT_SEARCH_TREE` 补 SX——因为发起 SMO 的 mtr 已持有 index X/SX，这里只需补子页锁。
- 叶子层把 `up_match/low_match`（以及 AHI 用的字节级 `up_bytes/low_bytes`）写回游标——上层（`row_ins_duplicate_error_in_clust` 判重复键、`row_search_mvcc` 判命中）全靠这些值。
- `btr_search_info_update` 是 AHI 的入口节流：内部 `hash_analysis` 每搜索 +1，**每 17 次**才真正进入统计与构建决策（见「AHI」节）。

#### 页内二分：`page_cur_search_with_match`（page0cur.cc 逐行解析）

这是搜索在**单个页面内**的定位算法，两级查找：

```cpp
void page_cur_search_with_match(const buf_block_t *block, const dict_index_t *index,
                                const dtuple_t *tuple, page_cur_mode_t mode,
                                ulint *iup_matched_fields, ulint *ilow_matched_fields,
                                page_cur_t *cursor, rtr_info_t *rtr_info) {
  ulint up, low, mid;
  const rec_t *up_rec, *low_rec, *mid_rec;
  ulint up_matched_fields, low_matched_fields, cur_matched_fields;
  int cmp = 0;
  ...
  up_matched_fields = *iup_matched_fields;   // 从上层下钻带来的公共前缀（跨层复用！）
  low_matched_fields = *ilow_matched_fields;

  // 第一级：对 page directory 槽二分，直到上下槽距离为 1
  low = 0;
  up = page_dir_get_n_slots(page) - 1;

  while (up - low > 1) {
    mid = (low + up) / 2;
    slot = page_dir_get_nth_slot(page, mid);
    mid_rec = page_dir_slot_get_rec(slot);    // 槽主记录（每个槽"拥有"若干条记录的代表）

    cur_matched_fields = std::min(low_matched_fields, up_matched_fields);  // ★ 公共前缀起点
    cmp = tuple->compare(mid_rec, index, offsets, &cur_matched_fields);

    if (cmp > 0) {                  // tuple > mid_rec：目标在右半
    low_slot_match:
      low = mid;
      low_matched_fields = cur_matched_fields;
    } else if (cmp) {               // tuple < mid_rec：目标在左半
    up_slot_match:
      up = mid;
      up_matched_fields = cur_matched_fields;
    } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE) {
      goto low_slot_match;          // 相等 + 找下界模式 → 归左（继续向右找更大的等值）
    } else {
      goto up_slot_match;           // 相等 + 找上界模式 → 归右
    }
  }

  slot = page_dir_get_nth_slot(page, low);
  low_rec = page_dir_slot_get_rec(slot);
  slot = page_dir_get_nth_slot(page, up);
  up_rec = page_dir_slot_get_rec(slot);

  // 第二级：在 up_rec 槽"拥有"的记录里线性查找，直到 low_rec 与 up_rec 相邻
  while (page_rec_get_next_const(low_rec) != up_rec) {
    mid_rec = page_rec_get_next_const(low_rec);

    cur_matched_fields = std::min(low_matched_fields, up_matched_fields);
    cmp = tuple->compare(mid_rec, index, offsets, &cur_matched_fields);

    if (cmp > 0) { low_rec = mid_rec; low_matched_fields = cur_matched_fields; }
    else if (cmp) { up_rec = mid_rec; up_matched_fields = cur_matched_fields; }
    else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE) {
      // 相等且找下界：若匹配字段数为 0 说明撞上 REC_INFO_MIN_REC_FLAG 的最小记录
      if (!cmp && !cur_matched_fields) {
        cur_matched_fields = dtuple_get_n_fields_cmp(tuple);  // 视为全匹配，保证能落位
      }
      low_rec = mid_rec;
      low_matched_fields = cur_matched_fields;
    } else {
      up_rec = mid_rec;
      up_matched_fields = cur_matched_fields;
    }
  }

  // 定位：mode <= PAGE_CUR_GE 定位到 up_rec（右界）；否则 low_rec（左界）
  if (mode <= PAGE_CUR_GE) {
    page_cur_position(up_rec, block, cursor);
  } else {
    page_cur_position(low_rec, block, cursor);
  }
  *iup_matched_fields = up_matched_fields;   // 匹配数回传给上层，跨层继续复用
  *ilow_matched_fields = low_matched_fields;
}
```

逐行解释：

- **两级查找的结构原因**：页目录槽是**稀疏索引**（每个槽"拥有"最多 `PAGE_DIR_SLOT_MAX_N_OWNED` 条记录），先在槽数组上二分（O(log slots)），再在目标槽内线性扫（最多 8 条）。槽数远小于记录数，所以比纯线性快，又比维护完整有序数组省空间。
- **`cur_matched_fields = min(low, up)` 是跨层公共前缀复用**：从上一次比较（可能是父层下钻）带进来的 `up_matched_fields/low_matched_fields`，是"已知 tuple 与上界/下界记录的公共前缀长度"。用它做比较起点，跳过了已经确定的相等前缀字段——这是 B-tree 搜索省比较次数的核心技巧。
- **相等时的方向**（`cmp == 0`）：`PAGE_CUR_G/LE` 是"找下界"（找最后一个 ≤/第一个 ≥ 的位置要向右），归左继续；`PAGE_CUR_L/GE` 是"找上界"，归右。这决定了等值键重复时游标停在等值组的哪一端。
- **`!cmp && !cur_matched_fields` 的特殊处理**：当 `mode` 是 G/LE 且比较出"相等但匹配字段数为 0"，意味着撞上了带 `REC_INFO_MIN_REC_FLAG` 的**预定义最小记录**（最左子页的 node pointer 的 key 被强制为全最小值，比较器返回相等但字段不匹配）。此时强制把 `cur_matched_fields` 设为全字段数，确保能正常落位。
- **最终定位**：`mode <= PAGE_CUR_GE`（L/LE/G/GE 的枚举序）定位到右界 `up_rec`，否则定位到左界 `low_rec`。插入用 `PAGE_CUR_LE`，游标最终停在"≤ key 的最后一条"（插入点的前驱），插入发生在游标之后。
- `page_cur_search_with_match_bytes` 与之同构，只多了 `up_bytes/low_bytes` 的回填（首个部分匹配字段内的字节匹配数），专供 AHI。






### 插入与分裂

#### 乐观插入：`btr_cur_optimistic_insert`

先做**空间预估快速失败**，再锁、undo、插入：

```cpp
rec_size = rec_get_converted_size(index, entry);
if (page_zip_rec_needs_ext(rec_size, page_is_comp(page),
                           dtuple_get_n_fields(entry), page_size)) {
  big_rec_vec = dtuple_convert_big_rec(index, nullptr, entry);   // 摘出超长字段
  ...
}
// 压缩页：插入后 data_size + rec_size >= 最优填充 → 直接放弃乐观
if (leaf && page_size.is_compressed() &&
    (page_get_data_size(page) + rec_size >=
     dict_index_zip_pad_optimal_page_size(index))) goto fail;

ulint max_size = page_get_max_insert_size_after_reorganize(page, 1);
if (page_has_garbage(page)) {
  if ((max_size < rec_size || max_size < BTR_CUR_PAGE_REORGANIZE_LIMIT) &&
      page_get_n_recs(page) > 1 && page_get_max_insert_size(page, 1) < rec_size)
    goto fail;
} else if (max_size < rec_size) goto fail;

// 聚簇索引顺序插入：主动失败转分裂，给未来更新留空间
if (leaf && !page_size.is_compressed() && index->is_clustered() &&
    page_get_n_recs(page) >= 2 &&
    dict_index_get_space_reserve() + rec_size > max_size &&
    (btr_page_get_split_rec_to_right(cursor, &dummy) ||
     btr_page_get_split_rec_to_left(cursor, &dummy))) goto fail;

page_cursor = btr_cur_get_page_cur(cursor);
err = btr_cur_ins_lock_and_undo(flags, cursor, entry, thr, mtr, &inherit);
//   ↑ 行锁（lock_rec_insert_check_and_lock：隐式锁转换 + 插入意向锁 + 唯一性冲突检测）
//   ↑ undo：仅聚簇索引写 TRX_UNDO_INSERT_OP，roll_ptr 回填 entry 的 DATA_ROLL_PTR
*rec = page_cur_tuple_insert(page_cursor, entry, index, offsets, heap, mtr);
if (*rec == nullptr && !page_size.is_compressed()) {
  if (!btr_page_reorganize(page_cursor, index, mtr)) goto fail;   // 重整后再试一次
  *rec = page_cur_tuple_insert(...);
}
```

要点：

- **大记录**：`page_zip_rec_needs_ext` 判定需要外存时 `dtuple_convert_big_rec` 把超长字段抽出（行内只留 20 字节引用前缀），`big_rec_vec` 由 row 层在 mtr 提交前调 `lob::btr_store_big_rec_extern_fields` 写 LOB 页。
- **`BTR_CUR_PAGE_REORGANIZE_LIMIT = UNIV_PAGE_SIZE / 32`**：重整能换回的空间下限——有垃圾但重整后仍不够就直接放弃（不浪费一次重整）。
- **聚簇顺序插入的"预留"**：`dict_index_get_space_reserve()` ≈ 1/16 页。若插入后剩余空间不足 1/16 页且检测到顺序插入模式，**主动失败转分裂**——这解释了为什么自增主键的表叶子页利用率始终达不到 100%：更新（尤其变长字段变长）需要页内余地。
- **插入失败后的重整**：`page_cur_tuple_insert` 失败 → `btr_page_reorganize` 重排页回收碎片 → 再插一次。
- **二级索引不写 undo**：undo 只在聚簇索引上记录（二级索引靠聚簇索引的 undo 重放）。这是 InnoDB 的重要设计——二级索引变更的"真相"在聚簇索引，二级索引只是投影。
- 收尾：AHI 更新（`btr_search_update_hash_node_on_insert` O(1) 快路径或整页重建）、`lock_update_insert` 把 infimum 暂存的 gap 锁迁移到新记录、非压缩二级索引页在独立 mtr 递减 change buffer bitmap free bits。

#### 悲观插入与页分裂：`btr_cur_pessimistic_insert` → `btr_page_split_and_insert`

悲观插入先 `btr_cur_ins_lock_and_undo`（undo 在分裂前写完——undo 是逻辑日志，与页怎么拆无关），再预留 extent，然后分派：

```cpp
ulint n_extents = cursor->tree_height / 16 + 3;
fsp_reserve_free_extents(&n_reserved, index->space, n_extents, FSP_NORMAL, mtr);
...
if (dict_index_get_page(index) == btr_cur_get_block(cursor)->page.id.page_no())
  *rec = btr_root_raise_and_insert(flags, cursor, offsets, heap, entry, mtr);
else
  *rec = btr_page_split_and_insert(flags, cursor, offsets, heap, entry, mtr);
```

`btr_page_split_and_insert` 的完整算法（关键步骤）：

**第 0 步：优先插右兄弟**（8.0 优化，`btr_insert_into_right_sibling`）——游标在页尾且存在右兄弟时，把 tuple 直接插到右兄弟页头，并重建父层 node pointer（删旧的、插新的）。单调递增插入下这替代了反复 50/50 分裂。

`btr_page_split_and_insert` 完整源码逐行解析（btr0btr.cc 2279~2656，spatial 分支省略）：

```cpp
rec_t *btr_page_split_and_insert(flags, cursor, offsets, heap, tuple, mtr) {
  buf_block_t *block; page_t *page; page_zip_des_t *page_zip;
  page_no_t page_no, hint_page_no; byte direction;
  buf_block_t *new_block; page_t *new_page; page_zip_des_t *new_page_zip;
  rec_t *split_rec; buf_block_t *left_block, *right_block, *insert_block;
  page_cur_t *page_cursor; rec_t *first_rec; byte *buf = nullptr;
  rec_t *move_limit; bool insert_will_fit, insert_left;
  ulint n_iterations = 0; rec_t *rec; ulint n_uniq; dict_index_t *index;

  index = cursor->index;
  if (!*heap) *heap = mem_heap_create(1024, ...);
  n_uniq = dict_index_get_n_unique_in_tree(cursor->index);

func_start:
  mem_heap_empty(*heap); *offsets = nullptr;
  block = btr_cur_get_block(cursor);
  page = buf_block_get_frame(block);
  page_zip = buf_block_get_page_zip(block);
  ut_ad(mtr_is_block_fix(mtr, block, MTR_MEMO_PAGE_X_FIX, ...));  // 游标页必须 X-latch
  ut_ad(!page_is_empty(page));

  /* 第 0 步：优先插右兄弟（顺序插入优化，8.0 新增） */
  rec = btr_insert_into_right_sibling(flags, cursor, offsets, *heap, tuple, mtr);
  if (rec != nullptr) return rec;   // 直接插进右兄弟成功，不分裂

  page_no = block->page.id.page_no();

  /* 第 1 步：决定 split 点与分裂方向 */
  insert_left = false;
  if (n_iterations > 0) {                       // 压缩页二次分裂
    direction = FSP_UP; hint_page_no = page_no + 1;
    split_rec = btr_page_get_split_rec(cursor, tuple);   // 按字节空间 50% 分割
    if (split_rec == nullptr)
      insert_left = btr_page_tuple_smaller(cursor, tuple, offsets, n_uniq, heap);
  } else if (btr_page_get_split_rec_to_right(cursor, &split_rec)) {   // 顺序右追加
    direction = FSP_UP; hint_page_no = page_no + 1;
  } else if (btr_page_get_split_rec_to_left(cursor, &split_rec)) {    // 倒序左追加
    direction = FSP_DOWN; hint_page_no = page_no - 1;
  } else {                                      // 默认：按记录数中点
    direction = FSP_UP; hint_page_no = page_no + 1;
    if (page_get_n_recs(page) > 1) split_rec = page_get_middle_rec(page);
    else if (btr_page_tuple_smaller(cursor, tuple, offsets, n_uniq, heap))
      split_rec = page_rec_get_next(page_get_infimum_rec(page));
    else split_rec = nullptr;
  }

  /* 第 2 步：分配新页 */
  new_block = btr_page_alloc(cursor->index, hint_page_no, direction,
                             btr_page_get_level(page), mtr, mtr);
  if (!new_block) return nullptr;
  new_page = buf_block_get_frame(new_block);
  new_page_zip = buf_block_get_page_zip(new_block);
  btr_page_create(new_block, new_page_zip, cursor->index, btr_page_get_level(page), mtr);

  /* 第 3 步：确定上半页首记录 first_rec 与搬移起点 move_limit */
  if (split_rec) {
    first_rec = move_limit = split_rec;   // 上半页首记录 = split_rec
    *offsets = rec_get_offsets(split_rec, cursor->index, *offsets, n_uniq, ..., heap);
    insert_left = cmp_dtuple_rec(tuple, split_rec, cursor->index, *offsets) < 0;  // tuple 落在哪半
    if (!insert_left && new_page_zip && n_iterations > 0) { split_rec = nullptr; goto insert_empty; }
  } else if (insert_left) {
    ut_a(n_iterations > 0);
    first_rec = page_rec_get_next(page_get_infimum_rec(page));
    move_limit = page_rec_get_next(btr_cur_get_rec(cursor));
  } else {
insert_empty:
    // split_rec==nullptr：tuple 自己做上半页首记录 → 临时物化它（不落页）
    buf = ut::new_arr_withkey<byte>(...rec_get_converted_size(cursor->index, tuple)...);
    first_rec = rec_convert_dtuple_to_rec(buf, cursor->index, tuple);
    move_limit = page_rec_get_next(btr_cur_get_rec(cursor));
  }

  /* 第 4 步：先改树结构（插父层 node pointer），后搬记录 */
  btr_attach_half_pages(flags, cursor->index, block, first_rec, new_block, direction, mtr);

  /* 若叶子层 + 非压缩 + 插入半页装得下 → 提前释放 index 树锁 */
  if (split_rec)
    insert_will_fit = !new_page_zip && btr_page_insert_fits(cursor, split_rec, offsets, tuple, heap);
  else
    insert_will_fit = !new_page_zip && btr_page_insert_fits(cursor, nullptr, offsets, tuple, heap);
  if (!srv_read_only_mode && !cursor->index->table->is_intrinsic() &&
      insert_will_fit && page_is_leaf(page) && !dict_index_is_online_ddl(cursor->index)) {
    mtr->memo_release(dict_index_get_lock(cursor->index), MTR_MEMO_X_LOCK | MTR_MEMO_SX_LOCK);
    // 注意：root block latch 不能放（fseg header 在其上且已改）
  }

  /* 第 5 步：搬移上半页记录 */
  if (direction == FSP_DOWN) {
    if (!page_move_rec_list_start(new_block, block, move_limit, cursor->index, mtr)) {
      // 压缩页 zip 日志失败回退：整页字节拷贝后逐页删除
      page_zip_copy_recs(new_page_zip, new_page, page_zip, page, cursor->index, mtr);
      page_delete_rec_list_end(move_limit - page + new_page, new_block, ...);
      lock_move_rec_list_start(new_block, block, move_limit, new_page + PAGE_NEW_INFIMUM);
      btr_search_update_hash_on_move(new_block, block, cursor->index);
      page_delete_rec_list_start(move_limit, block, cursor->index, mtr);
    }
    left_block = new_block; right_block = block;
    lock_update_split_left(right_block, left_block);   // gap 锁继承
  } else {
    if (!page_move_rec_list_end(new_block, block, move_limit, cursor->index, mtr)) {
      page_zip_copy_recs(...); page_delete_rec_list_start(...);
      lock_move_rec_list_end(new_block, block, move_limit);
      btr_search_update_hash_on_move(new_block, block, cursor->index);
      page_delete_rec_list_end(move_limit, block, cursor->index, ...);
    }
    left_block = block; right_block = new_block;
    lock_update_split_right(right_block, left_block);
  }

  /* 第 6-8 步：定位插入半页并插入；失败重整/再分裂 */
  insert_block = insert_left ? left_block : right_block;
  page_cursor = btr_cur_get_page_cur(cursor);
  page_cur_search(insert_block, cursor->index, tuple, page_cursor);
  rec = page_cur_tuple_insert(page_cursor, tuple, cursor->index, offsets, heap, mtr);
  if (rec != nullptr) goto func_exit;

  // 插入失败：重整半页再试
  if (page_cur_get_page_zip(page_cursor) || !btr_page_reorganize(page_cursor, cursor->index, mtr))
    goto insert_failed;
  rec = page_cur_tuple_insert(page_cursor, tuple, cursor->index, offsets, heap, mtr);
  if (rec == nullptr) {
insert_failed:
    if (!cursor->index->is_clustered() && !cursor->index->table->is_temporary()) {
      ibuf_reset_free_bits(new_block); ibuf_reset_free_bits(block);   // 安全重置 free bits
    }
    n_iterations++;
    ut_ad(n_iterations < 2 || buf_block_get_page_zip(insert_block));
    ut_ad(!insert_will_fit);
    goto func_start;   // 重新选 split 点再分裂一轮
  }

func_exit:
  if (!cursor->index->is_clustered() && !cursor->index->table->is_temporary() && page_is_leaf(page))
    ibuf_update_free_bits_for_two_pages_low(left_block, right_block, mtr);
  MONITOR_INC(MONITOR_INDEX_SPLIT);
  return rec;
}
```

**split 点选择的三个函数**（顺序追加/倒序追加/字节 50%）：

```cpp
// 倒序追加检测：上次插入恰在本次插入点之后 → 收敛点在页中段
bool btr_page_get_split_rec_to_left(btr_cur_t *cursor, rec_t **split_rec) {
  page = btr_cur_get_page(cursor);
  insert_point = btr_cur_get_rec(cursor);
  if (page_header_get_ptr(page, PAGE_LAST_INSERT) == page_rec_get_next(insert_point)) {
    infimum = page_get_infimum_rec(page);
    // 收敛点在页中段：把 insert_point 之前那条也带上半页（避免反复搬动收敛点前的小记录）
    if (infimum != insert_point && page_rec_get_next(infimum) != insert_point)
      *split_rec = insert_point;
    else
      *split_rec = page_rec_get_next(insert_point);
    return true;
  }
  return false;
}

// 顺序追加检测（右收敛）
bool btr_page_get_split_rec_to_right(btr_cur_t *cursor, rec_t **split_rec) {
  page = btr_cur_get_page(cursor);
  insert_point = btr_cur_get_rec(cursor);
  // eager 启发式：本次插入紧跟上次插入 → 判断为顺序插入模式
  if (page_header_get_ptr(page, PAGE_LAST_INSERT) == insert_point) {
    next_rec = page_rec_get_next(insert_point);
    if (page_rec_is_supremum(next_rec)) { split_at_new: *split_rec = nullptr; }  // 插在页尾：新页只装 tuple
    else {
      next_next_rec = page_rec_get_next(next_rec);
      if (page_rec_is_supremum(next_next_rec)) goto split_at_new;
      *split_rec = next_next_rec;   // 插入点后 ≥2 条：只留 1 条，其余全分走
    }
    return true;
  }
  return false;
}

// 字节空间 50% 分割（压缩页二次分裂用）：从左累加直到 ≥ total_space/2
static rec_t *btr_page_get_split_rec(btr_cur_t *cursor, const dtuple_t *tuple) {
  insert_size = rec_get_converted_size(cursor->index, tuple);
  free_space = page_get_free_space_of_empty(page_is_comp(page));
  if (page_zip) free_space = min(free_space, page_zip_empty_size(...));  // 压缩页取更小者
  total_data = page_get_data_size(page) + insert_size;
  total_n_recs = page_get_n_recs(page) + 1;
  total_space = total_data + page_dir_calc_reserved_space(total_n_recs);

  n = 0; incl_data = 0;
  ins_rec = btr_cur_get_rec(cursor); rec = page_get_infimum_rec(page);
  do {
    // 依次"包含"下一条记录：先 tuple，再插入点之后，再之前
    if (rec == ins_rec) rec = nullptr;          // 包含 tuple
    else if (rec == nullptr) rec = page_rec_get_next(ins_rec);
    else rec = page_rec_get_next(rec);
    if (rec == nullptr) incl_data += insert_size;
    else incl_data += rec_offs_size(rec_get_offsets(rec, ...));
    n++;
  } while (incl_data + page_dir_calc_reserved_space(n) < total_space / 2);

  // 左半若装得进空页，右半首记录即 split_rec；否则 split_rec = 最后包含的那条
  if (incl_data + page_dir_calc_reserved_space(n) <= free_space) {
    next_rec = (rec == nullptr) ? page_rec_get_next(ins_rec) : page_rec_get_next(rec);
    if (!page_rec_is_supremum(next_rec)) rec = next_rec;
  }
  return rec;
}
```

逐行解释：

- **`PAGE_LAST_INSERT` 是"上次插入位置"的页头字段**，`btr_page_get_split_rec_to_right/left` 用它识别插入模式：`LAST_INSERT == insert_point` 说明上次就插在紧邻位置（顺序追加）；`LAST_INSERT == next(insert_point)` 说明倒序。识别出顺序追加就把**几乎所有记录分走**（`split_rec = next_next_rec`，只留 1 条），给顺序流让满整页——这是自增主键场景避免"50/50 分裂导致页利用率 50%"的关键。留 1 条的原因源码注释明写：让 AHI 能靠页上这一条做右搜索位置的校验。
- **`btr_page_get_split_rec` 按字节 50%**（`total_space/2`），不是记录数 50%：从 infimum 起逐条累加记录大小（含目录槽预留），直到累计过半。它只用于压缩页二次分裂（`n_iterations > 0`），因为压缩页的空间按字节算，记录数中点可能导致某半页压缩失败。
- **`first_rec` 三种来源**：正常是 `split_rec`（上半页首记录，作为父层 node pointer 的 key）；`insert_empty` 分支（`split_rec == nullptr`）时 tuple 自己做上半页首记录，所以临时 `rec_convert_dtuple_to_rec` 物化它——**只用来取 key 前缀构建 node pointer，不真正落页**。
- **搬移的列表级日志**：`page_move_rec_list_end = page_copy_rec_list_end`（写 `MLOG_LIST_END_COPY`/`MLOG_COMP_LIST_END_COPY`）+ `page_delete_rec_list_end`（写 `MLOG_LIST_END_DELETE`，日志体仅 2 字节起始偏移）。整批搬移只两条日志，恢复端 `page_parse_delete_rec_list` 按偏移重放——ARIES 生理日志在页搬移上的应用。
- **压缩页回退路径**：`page_move_rec_list_*` 返回 false（zip 日志装不下）时，退化为 `page_zip_copy_recs` 整页字节拷贝 + `page_delete_rec_list_start/end` 逐页删除——删除总会成功，这是压缩表分裂的"保底"。
- **`n_iterations` 保证终止**：普通页断言 `< 2`（`insert_will_fit` 为 false 时 index 锁没放，第二轮必然成功）；只有压缩页可能多轮（`insert_will_fit = !new_page_zip && ...`）。

**`btr_attach_half_pages`：先改结构后搬数据的落点**（btr0btr.cc 1988~2126）：

```cpp
static void btr_attach_half_pages(flags, dict_index_t *index, buf_block_t *block,
                                  const rec_t *split_rec, buf_block_t *new_block,
                                  ulint direction, mtr_t *mtr) {
  page_t *page = buf_block_get_frame(block);
  page_t *lower_page, *upper_page; page_no_t lower_page_no, upper_page_no;
  page_zip_des_t *lower_page_zip, *upper_page_zip;
  dtuple_t *node_ptr_upper; buf_block_t *prev_block = nullptr, *next_block = nullptr;

  heap = mem_heap_create(1024, ...);

  // 按方向决定哪个是 lower（下半页）、哪个是 upper（上半页）
  if (direction == FSP_DOWN) {
    lower_page = buf_block_get_frame(new_block);      // 向左分裂：新页是下半页
    lower_page_no = new_block->page.id.page_no();
    upper_page = buf_block_get_frame(block);          // 原页是上半页
    upper_page_no = block->page.id.page_no();
    // ★ 先找父页，把指向原页的 node pointer 的 child page no 改成新页（lower）
    offsets = btr_page_get_father_block(nullptr, heap, index, block, mtr, &cursor);
    btr_node_ptr_set_child_page_no(btr_cur_get_rec(&cursor), btr_cur_get_page_zip(&cursor),
                                   offsets, lower_page_no, mtr);
  } else {   // FSP_UP：向右分裂
    lower_page = buf_block_get_frame(block);          // 原页是下半页
    lower_page_no = block->page.id.page_no();
    upper_page = buf_block_get_frame(new_block);      // 新页是上半页
    upper_page_no = new_block->page.id.page_no();
    // FSP_UP 不改父页旧 node pointer：它仍指向原页=lower，天然正确
  }

  prev_page_no = btr_page_get_prev(page, mtr);
  next_page_no = btr_page_get_next(page, mtr);

  // 为一致性，先锁住要改 next/prev 的邻居页
  if (prev_page_no != FIL_NULL && direction == FSP_DOWN)
    prev_block = btr_block_get(page_id_t(space, prev_page_no), ..., RW_X_LATCH, ..., mtr);
  if (next_page_no != FIL_NULL && direction != FSP_DOWN)
    next_block = btr_block_get(page_id_t(space, next_page_no), ..., RW_X_LATCH, ..., mtr);

  level = btr_page_get_level(buf_block_get_frame(block));

  // ★ 构建上半页的 node pointer（key = split_rec 的前缀 + child page no）
  node_ptr_upper = dict_index_build_node_ptr(index, split_rec, upper_page_no, heap, level);
  // ★ 插入父层（可能触发父层递归分裂！）
  btr_insert_on_non_leaf_level(flags, index, level + 1, node_ptr_upper, ..., mtr);

  // 修正本层双向链：prev → lower → upper → next
  if (prev_block) btr_page_set_next(prev_page, ..., lower_page_no, mtr);
  if (next_block) btr_page_set_prev(next_page, ..., upper_page_no, mtr);
  if (direction == FSP_DOWN) btr_page_set_prev(lower_page, ..., prev_page_no, mtr);
  btr_page_set_next(lower_page, ..., upper_page_no, mtr);
  btr_page_set_prev(upper_page, ..., lower_page_no, mtr);
  if (direction != FSP_DOWN) btr_page_set_next(upper_page, ..., next_page_no, mtr);
}
```

逐行解释：

- **`direction` 决定 node pointer 怎么改**：FSP_UP（向右分裂）时，父页旧 node pointer 指向原页，而原页成为 lower，**key 不变、页号不变，天然正确**——只需为 new_block 新建一条 node pointer；FSP_DOWN（向左分裂）时，原页成为 upper（key 变了），父页旧 node pointer 的 key 和页号都要改——先把 child page no 改成 new_block（`btr_node_ptr_set_child_page_no`），随后插入的上半页 node pointer 才是"原页的新 key"。
- **先插父层 node pointer，再改兄弟链**：`btr_insert_on_non_leaf_level` 是唯一会**递归向上分裂**的地方——父页满了它又会调 `btr_page_split_and_insert`，一路可能分裂到根。把它放在搬记录之前，是因为一旦父层分裂失败（空间不足）整个操作必须能继续（SMO 不可逆），而搬记录只是叶子内部物理移动，顺序无所谓。
- 所有 `btr_page_set_next/prev` 对压缩页走 `page_zip_write_header`，普通页走 `mlog_write_ulint`——兄弟链是持久化在 `FIL_PAGE_PREV/NEXT` 里的。

**`btr_insert_on_non_leaf_level`：非叶层插入的统一入口**（btr0btr.cc 1926~1978）：

```cpp
void btr_insert_on_non_leaf_level(flags, dict_index_t *index, ulint level,
                                  dtuple_t *tuple, ut::Location location, mtr_t *mtr) {
  btr_cur_t cursor; rec_t *rec; mem_heap_t *heap = nullptr; ...

  ut_ad(level > 0);   // 只用于非叶层（level ≥ 1）

  // 用 BTR_CONT_MODIFY_TREE 搜索（假定 mtr 已持 index X/SX，只补子页锁）
  btr_cur_search_to_nth_level(index, level, tuple, PAGE_CUR_LE,
                              BTR_CONT_MODIFY_TREE, &cursor, 0, ..., mtr);

  // 先乐观插入（带 NO_LOCKING|KEEP_SYS|NO_UNDO——node pointer 不是用户数据，不加锁不写 undo）
  err = btr_cur_optimistic_insert(
      flags | BTR_NO_LOCKING_FLAG | BTR_KEEP_SYS_FLAG | BTR_NO_UNDO_LOG_FLAG,
      &cursor, &offsets, &heap, tuple, &rec, &dummy_big_rec, nullptr, mtr);

  if (err == DB_FAIL) {
    // 父页满了 → 悲观插入（可能再次分裂父页，递归向上）
    err = btr_cur_pessimistic_insert(
        flags | BTR_NO_LOCKING_FLAG | BTR_KEEP_SYS_FLAG | BTR_NO_UNDO_LOG_FLAG,
        &cursor, &offsets, &heap, tuple, &rec, &dummy_big_rec, nullptr, mtr);
    ut_a(err == DB_SUCCESS);
  }
}
```

逐行解释：这是**非叶层 node pointer 插入的唯一入口**，分裂、根抬高、删最左 node pointer 后重插，最终都走它。三个 flag 的含义：`BTR_NO_LOCKING_FLAG`（node pointer 不参与行锁）、`BTR_KEEP_SYS_FLAG`（不更新记录的 trx_id/roll_ptr——node pointer 不是用户记录）、`BTR_NO_UNDO_LOG_FLAG`（不写 undo——node pointer 由 redo 保证，事务回滚不回溯它）。乐观失败转悲观，就形成了**分裂的递归链**：`split → attach_half_pages → insert_on_non_leaf_level → pessimistic_insert → split(父页) → ...`。

#### 中间页与 node pointer：构建、解析、最小记录标记

非叶层（中间页）只存 **node pointer**——一条记录 = "子页首条记录 key 的前缀 + 4 字节 child page no"。它既是"路由"（搜索时据此下钻），也是"下界标记"（指向 `[key, 下一 node_ptr 的 key)` 区间）。node pointer 不是用户数据：不加行锁、不写 undo、不更新 trx_id（对应 `btr_insert_on_non_leaf_level` 的三个 flag）。

**构建 `dict_index_build_node_ptr`**（dict0dict.cc 3649~3709）：

```cpp
dtuple_t *dict_index_build_node_ptr(const dict_index_t *index, const rec_t *rec,
                                    page_no_t page_no, mem_heap_t *heap, ulint level) {
  dtuple_t *tuple; dfield_t *field; byte *buf; ulint n_unique;

  if (dict_index_is_ibuf(index)) {
    // ibuf 树：叶层取整条记录，非叶层去掉最后一个字段（child page no）
    n_unique = rec_get_n_fields_old_raw(rec);
    if (level > 0) n_unique--;
  } else {
    n_unique = dict_index_get_n_unique_in_tree_nonleaf(index);  // 非叶层只用 unique 字段数
  }

  tuple = dtuple_create(heap, n_unique + 1);   // n_unique 个 key 字段 + 1 个 child 页号字段

  // ★ 关键：n_fields_cmp 设为 n_unique，搜索时"不比较最后一个页号字段"
  dtuple_set_n_fields_cmp(tuple, n_unique);

  dict_index_copy_types(tuple, index, n_unique);   // 拷贝 key 字段类型

  buf = mem_heap_alloc(heap, 4);
  mach_write_to_4(buf, page_no);                   // child page no 写进 4 字节

  field = dtuple_get_nth_field(tuple, n_unique);
  dfield_set_data(field, buf, 4);
  dtype_set(dfield_get_type(field), DATA_SYS_CHILD, DATA_NOT_NULL, 4);  // 系统列类型

  rec_copy_prefix_to_dtuple(tuple, rec, index, n_unique, heap);   // 拷贝 rec 前 n_unique 字段作 key
  dtuple_set_info_bits(tuple, dtuple_get_info_bits(tuple) | REC_STATUS_NODE_PTR);  // 标记 NODE_PTR

  return tuple;
}
```

逐行解释：

- **node pointer 的字段布局**：前 `n_unique` 个字段是"子页最小 key 的前缀"（`dict_index_get_n_unique_in_tree_nonleaf`，非叶层只需 unique 字段，因为非叶层不存完整记录），最后一个字段是 `DATA_SYS_CHILD` 类型的 4 字节 child page no。
- **`n_fields_cmp = n_unique` 是搜索正确性的保证**：上层搜索时 `tuple->compare` 只比较前 `n_unique` 个字段，**绝不比较页号字段**——因为不同 node pointer 可能 key 完全相同（同一键跨多页），若比较页号会导致"等值搜索"定位错乱。这也是 `btr_cur_search_to_nth_level` 注释里"n_fields_cmp must be set so that it cannot get compared to the node ptr page number field"的含义。
- `REC_STATUS_NODE_PTR` 是 info bits，让 `rec_get_node_ptr_flag` 能识别"这是 node pointer 不是普通记录"（`btr_node_ptr_get_child_page_no` 的断言靠它）。

**解析/改写 child page no**（btr0btr.ic）：

```cpp
// 读：child 地址在最后一个字段
static inline page_no_t btr_node_ptr_get_child_page_no(const rec_t *rec, const ulint *offsets) {
  ut_ad(!rec_offs_comp(offsets) || rec_get_node_ptr_flag(rec));
  field = rec_get_nth_field(nullptr, rec, offsets, rec_offs_n_fields(offsets) - 1, &len);
  ut_ad(len == 4);
  page_no = mach_read_from_4(field);
  ut_ad(page_no > 1);
  return page_no;
}
```

搜索下钻（`page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets))`）就靠它取子页号。`btr_node_ptr_set_child_page_no` 是它的写方向（FSP_DOWN 分裂时把父页 node pointer 的页号改成新 lower 页），内部 `mlog_write_ulint(field, page_no, MLOG_4BYTES, mtr)` 写 redo。

**最小记录标记 `btr_set_min_rec_mark`**（btr0btr.cc 2758~2774）：

```cpp
void btr_set_min_rec_mark(rec_t *rec, mtr_t *mtr) {
  ulint info_bits;
  if (page_rec_is_comp(rec)) {
    info_bits = rec_get_info_bits(rec, true);
    rec_set_info_bits_new(rec, info_bits | REC_INFO_MIN_REC_FLAG);
    btr_set_min_rec_mark_log(rec, MLOG_COMP_REC_MIN_MARK, mtr);  // redo：2 字节记录偏移
  } else {
    info_bits = rec_get_info_bits(rec, false);
    rec_set_info_bits_old(rec, info_bits | REC_INFO_MIN_REC_FLAG);
    btr_set_min_rec_mark_log(rec, MLOG_REC_MIN_MARK, mtr);
  }
}
```

逐行解释：

- **最左子页没有下界**，它的 node pointer 的 key 无法用"首记录前缀"表示（子页里任意 key 都可能是最小），所以打 `REC_INFO_MIN_REC_FLAG`，把 key 语义上定义为"预定义最小值"。搜索比较器遇到它返回"相等但字段不匹配"（`cur_matched_fields == 0`），`page_cur_search_with_match` 里那个 `if (!cmp && !cur_matched_fields)` 特判就是为此兜底。
- 根页抬高、删最左 node pointer、分裂出最左子页时都要打/重打这个标记；redo 是 `MLOG_COMP_REC_MIN_MARK`/`MLOG_REC_MIN_MARK`（日志体仅 2 字节记录偏移，恢复端 `btr_parse_set_min_rec_mark`）。

#### 根页抬高：`btr_root_raise_and_insert`

树加高的算法很巧妙——**不直接分裂根页，而是先整体搬家再让子页做普通分裂**：

1. 分配一个**与旧根同级**的新页，`page_copy_rec_list_end` 把根页全部记录搬过去（`lock_update_root_raise` 迁移 infimum 上的锁信息，AHI 经 `btr_search_update_hash_on_move` 更新）；
2. 以新页首记录为 key 构建 node pointer，打 **`REC_INFO_MIN_REC_FLAG`**（最左子页无下界，其 node pointer 必须是预定义最小记录）；
3. `btr_page_empty` 把根页重建为 **level+1** 的空页（**保留 fseg header**——根页承担两个 segment 头，不能抹掉）；
4. 根页插入这唯一一条 node pointer（`ut_a(node_ptr_rec)` 必成——新根只有 1 条）；
5. 游标重定位到新页，**递归调用 `btr_page_split_and_insert`** 分裂新页，`btr_attach_half_pages` 会向新根插入第二条 node pointer。

为什么绕一圈？若直接分裂旧根（它同时是叶子），会在"根页分裂"里产生两个子页——而"先搬到子页、再做 50/50 普通分裂"把根分裂**规约**为已有代码路径，且 raise 中途根页恰好只有一条 node pointer（新页），避免"先分裂再抬高"可能触发根页再次分裂的递归。`btr_create` 里还有一条正确性断言：根页必须能容纳 2 条 `BTR_PAGE_MAX_REC_SIZE`（`UNIV_PAGE_SIZE/2 - 200`）记录——否则"分裂后两半各需容纳 1 条最大记录"的前提不成立。

### 删除与合并

#### 乐观删除：`btr_cur_optimistic_delete`

核心判断 `btr_cur_can_delete_without_compress`：

```cpp
if ((page_get_data_size(page) - rec_size <
     BTR_CUR_PAGE_COMPRESS_LIMIT(cursor->index)) ||       // 删完低于 merge 阈值
    (该层只剩一页) || (page_get_n_recs(page) < 2))         // 可能需合并
  return false;                                            // → 走悲观路径
return !rec_offs_any_extern(offsets);                      // extern 字段只能悲观删
```

通过则直接 `lock_update_delete` + `btr_search_update_hash_on_delete` + `page_cur_delete_rec`（游标停在被删记录的后继）。乐观删除**不持有 index 锁**（只 X 锁叶子页）——它只能做"纯页内"的删除。

#### 悲观删除：`btr_cur_pessimistic_delete`

```cpp
if (rec_offs_any_extern(offsets)) {
  lob::BtrContext btr_ctx(mtr, pcur, index, rec, offsets, block);
  btr_ctx.free_externally_stored_fields(trx_id, undo_no, rollback, rec_type, node);
  // ↑ LOB 页多、不能一个 mtr 装下：内部 commit/重启 mtr，游标必须通过 pcur 重定位
  if (pcur != nullptr) { cursor = pcur->get_btr_cur(); ... }
}

if (!has_reserved_extents && !rollback) {
  ulint n_extents = cursor->tree_height / 32 + 1;
  fsp_reserve_free_extents(&n_reserved, index->space, n_extents,
                           FSP_CLEANING, mtr);            // 合并/丢弃也需要空间保险
}

if (page_get_n_recs(page) < 2 &&
    dict_index_get_page(index) != block->page.id.page_no()) {
  btr_discard_page(cursor, mtr);                          // 只剩 1 条且非根：整页丢弃
  ret = true; goto return_after_reservations;
}

if (flags == 0) lock_update_delete(block, rec);

if (level > 0 && UNIV_UNLIKELY(rec == page_rec_get_next(page_get_infimum_rec(page)))) {
  rec_t *next_rec = page_rec_get_next(rec);
  if (btr_page_get_prev(page, mtr) == FIL_NULL)
    btr_set_min_rec_mark(next_rec, mtr);                  // 新左端升级为最小记录
  else {
    // 普通：删父页旧 node pointer + 以 next_rec 为 key 插新 node pointer
    btr_node_ptr_delete(index, block, mtr);
    dtuple_t *node_ptr = dict_index_build_node_ptr(index, next_rec,
                                                   block->page.id.page_no(),
                                                   heap, level);
    btr_insert_on_non_leaf_level(flags, index, level + 1, node_ptr, ..., mtr);
  }
}

btr_search_update_hash_on_delete(cursor);
page_cur_delete_rec(btr_cur_get_page_cur(cursor), index, offsets, mtr);

return_after_reservations:
  ret = btr_cur_compress_if_useful(cursor, false, mtr);   // 顺手压缩
  if (!srv_read_only_mode && page_is_leaf(page) && !dict_index_is_online_ddl(index))
    mtr_memo_release(mtr, dict_index_get_lock(index), MTR_MEMO_X_LOCK | MTR_MEMO_SX_LOCK);
  // ↑ 主动提前还 index 锁（不必等 mtr commit）
```

要点：

- **删非叶层最左 node pointer 是"改路由"事件**：该子页的下界变了，父页里代表它的 node pointer 必须换成新首记录做 key。最左页（无 prev）则只需给新首记录打最小记录标记（`btr_set_min_rec_mark`，redo `MLOG_COMP_REC_MIN_MARK`），因为父 node pointer 的 key 就是预定义最小值，不用改。5.7 的 `node_ptr_optimistic_delete` 在 8.0 拆成了 `btr_node_ptr_delete` + `btr_insert_on_non_leaf_level` 两步——后者统一处理"父页放不下还要分裂"的递归。
- **单记录页直接丢弃**（`btr_discard_page`）：比"删完再合并"更彻底——摘链、删父 node pointer、整页 free。
- **merge 的触发**（`btr_cur_compress_if_useful` → `btr_cur_compress_recommendation`）：页数据量 < `merge_threshold`（默认 50）% 页容量，**或该层只剩一页**（且非根），就推荐压缩。`btr_compress` 优先与**左兄弟**合并（`btr_can_merge_with_page` 检查对方装不装得下、必要时先重整对方）；左兄弟不行才右合并——右合并要把本页的父 node pointer 的 child page no 改成右兄弟、再删右兄弟的父 node pointer（可能引发父层递归合并）。
- 本层只剩一页时走 `btr_lift_page_up` **降树高**：把父页 `btr_page_empty` 清空重建为低一层、子页全部记录拷上、释放子页、所有祖先页 `PAGE_LEVEL` 递减 1。8.0 新增 `lift_father_up` 分支：被 lift 的是叶子且其父页也不是根时，**必须先 lift 父页**——因为 `btr_page_free` 按 level 选 segment，页被 lift 后 level 变了，直接释放会选错段。

#### 页合并 `btr_compress` 与降高 `btr_lift_page_up`（逐行解析）

合并与降高是 SMO 的"收缩"方向，由悲观删除末尾的 `btr_cur_compress_if_useful` 触发（条件是页数据量 < `merge_threshold`% 或该层只剩一页且非根）。

**`btr_compress` 完整源码逐行解析**（btr0btr.cc 2971~，spatial 分支省略）：

```cpp
bool btr_compress(btr_cur_t *cursor, bool adjust, mtr_t *mtr) {
  dict_index_t *index; space_id_t space;
  page_no_t left_page_no, right_page_no;
  buf_block_t *merge_block; page_t *merge_page = nullptr;
  bool is_left; buf_block_t *block; page_t *page;
  btr_cur_t father_cursor; mem_heap_t *heap; ulint *offsets; ulint nth_rec = 0;

  block = btr_cur_get_block(cursor);
  page = btr_cur_get_page(cursor);
  index = cursor->index;
  ut_ad(mtr_is_block_fix(mtr, block, MTR_MEMO_PAGE_X_FIX, index->table));  // 本页 X-latch
  space = dict_index_get_space(index);

  left_page_no = btr_page_get_prev(page, mtr);
  right_page_no = btr_page_get_next(page, mtr);

  heap = mem_heap_create(100, ...);
  // ★ 找父页（定位到指向本页的 node pointer）
  offsets = btr_page_get_father_block(nullptr, heap, index, block, mtr, &father_cursor);

  if (adjust) {
    nth_rec = page_rec_get_n_recs_before(btr_cur_get_rec(cursor));  // 记录删除位置（合并后调整游标）
  }

  // ★ 本层只剩一页：不是合并，是"降高"
  if (left_page_no == FIL_NULL && right_page_no == FIL_NULL) {
    merge_block = btr_lift_page_up(index, block, mtr);
    goto func_exit;
  }

  // 优先左合并，不行再右合并
  is_left = btr_can_merge_with_page(cursor, left_page_no, &merge_block, mtr);
retry:
  if (!is_left && !btr_can_merge_with_page(cursor, right_page_no, &merge_block, mtr)) {
    goto err_exit;   // 左右都装不下：放弃合并（保留半空页）
  }
  merge_page = buf_block_get_frame(merge_block);

  if (is_left) {
    // ── 左合并：本页记录拷到左兄弟尾部 ──
    rec_t *orig_pred = page_copy_rec_list_start(
        merge_block, block, page_get_supremum_rec(page), index, mtr);   // MLOG_LIST_START_COPY
    if (!orig_pred) goto err_exit;
    btr_search_drop_page_hash_index(block);         // 摘 AHI
    btr_level_list_remove(space, page_size, page, index, mtr);   // 摘兄弟链（prev→next）
    btr_node_ptr_delete(index, block, mtr);         // 删父页里指向本页的 node pointer
    lock_update_merge_left(merge_block, orig_pred, block);       // 锁继承给左兄弟
    if (adjust) nth_rec += page_rec_get_n_recs_before(orig_pred);
  } else {
    // ── 右合并：本页记录拷到右兄弟头部 ──
    rec_t *orig_succ = page_copy_rec_list_end(
        merge_block, block, page_get_infimum_rec(page), cursor->index, mtr);  // MLOG_LIST_END_COPY
    btr_search_drop_page_hash_index(block);
    btr_level_list_remove(space, page_size, page, index, mtr);
    // ★ 右合并的父层处理与左合并不同：
    //   把"指向本页"的 node pointer 的 child page no 改成右兄弟（右兄弟顶替本页位置），
    //   再删掉右兄弟原来的 node pointer
    btr_node_ptr_set_child_page_no(btr_cur_get_rec(&father_cursor), ..., offsets,
                                   right_page_no, mtr);
    compressed = btr_cur_pessimistic_delete(&err, true, &cursor2, BTR_CREATE_FLAG,
                                            false, 0, 0, 0, mtr, nullptr, nullptr);
    // ↑ cursor2 指向右兄弟的父 node pointer，删之（可能引发父层递归合并）
    lock_update_merge_right(merge_block, orig_succ, block);
  }
  ...
}
```

逐行解释：

- **合并是"整页搬移 + 释放空页"**：`page_copy_rec_list_start/end` 把本页全部记录拷到兄弟页（各写一条 `MLOG_LIST_START_COPY`/`MLOG_LIST_END_COPY`），然后 `btr_level_list_remove` 摘链、`btr_node_ptr_delete` 删父 node pointer——**本页不用逐条删除**，直接整页 `btr_page_free`（redo 随页作废）。
- **左右合并的父层差异**：左合并本页在右、左兄弟在左，删掉"指向本页"的 node pointer 即可（左兄弟的 node pointer 不变，因为左兄弟还是那个 key）；右合并本页在左、右兄弟在右，本页的父 node pointer **不能删**（删了右兄弟就没父指针了），而是把它的 child page no 改成右兄弟、再删右兄弟原 node pointer。
- **锁继承**（`lock_update_merge_left/right`）：被合并页上还挂着的 record lock（gap/next-key）要转移到兄弟页的对应记录上，否则并发事务的锁会随页释放而丢失。
- `btr_can_merge_with_page` 判定兄弟页能否装下本页全部数据：`page_get_max_insert_size_after_reorganize` 不够就失败，压缩页还要保证合并后不超 `dict_index_zip_pad_optimal_page_size`，必要时先 `btr_page_reorganize_block` 重整兄弟页腾空间。

**`btr_lift_page_up`：降树高**（btr0btr.cc 2804~2969）：

```cpp
static buf_block_t *btr_lift_page_up(dict_index_t *index, buf_block_t *block, mtr_t *mtr) {
  buf_block_t *father_block; page_t *father_page; ulint page_level;
  page_t *page = buf_block_get_frame(block);
  buf_block_t *blocks[BTR_MAX_LEVELS]; ulint n_blocks, i;
  bool lift_father_up; buf_block_t *block_orig = block;

  ut_ad(btr_page_get_prev(page, mtr) == FIL_NULL);   // 本层唯一页（无左右兄弟）
  ut_ad(btr_page_get_next(page, mtr) == FIL_NULL);

  page_level = btr_page_get_level(page);
  root_page_no = dict_index_get_page(index);

  {
    // ★ 沿父链一直找到根，把全部祖先页存进 blocks[]（lift 开始后树不一致，不能再搜索）
    btr_cur_t cursor; ...
    offsets = btr_page_get_father_block(offsets, heap, index, block, mtr, &cursor);
    father_block = btr_cur_get_block(&cursor);
    father_page = buf_block_get_frame(father_block);
    n_blocks = 0;
    for (b = father_block; b->page.id.page_no() != root_page_no;) {
      blocks[n_blocks++] = b = btr_cur_get_block(&cursor);  // 继续往上
      offsets = btr_page_get_father_block(offsets, heap, index, b, mtr, &cursor);
    }

    // ★ 8.0 新增：被 lift 的是叶子且父页不是根 → 先 lift 父页
    lift_father_up = (n_blocks && page_level == 0);
    if (lift_father_up) {
      // 原因：btr_page_free 按 level 选 segment（level==0 叶子段 / !=0 非叶段），
      // 叶子 lift 进父页后 level 变了，直接释放会选错段
      block = father_block;
      page = buf_block_get_frame(block);
      page_level = btr_page_get_level(page);
      father_block = blocks[0];
      father_page = buf_block_get_frame(father_block);
    }
  }

  btr_search_drop_page_hash_index(block);

  // ★ 清空父页、把它降一级，再把子页全部记录拷上
  btr_page_empty(father_block, father_page_zip, index, page_level, mtr);
  page_level++;
  if (!page_copy_rec_list_end(father_block, block, page_get_infimum_rec(page), index, mtr)) {
    // 压缩页回退：整页字节拷贝 + 锁/AHI 迁移
    page_zip_copy_recs(father_page_zip, father_page, page_zip, page, index, mtr);
    lock_move_rec_list_end(father_block, block, page_get_infimum_rec(page));
    btr_search_update_hash_on_move(father_block, block, index);
  }
  lock_update_copy_and_discard(father_block, block);   // 行锁从子页迁到父页

  // ★ 向上：所有祖先页的 PAGE_LEVEL 递减 1
  for (i = lift_father_up ? 1 : 0; i < n_blocks; i++, page_level++) {
    btr_page_set_level(buf_block_get_frame(blocks[i]), ..., page_level, mtr);
  }

  btr_page_free(index, block, mtr);   // 释放被 lift 掉的空页
  if (!index->is_clustered() && !index->table->is_temporary()) ibuf_reset_free_bits(father_block);
  return (lift_father_up ? block_orig : father_block);
}
```

逐行解释：

- **降高的本质是"父页顶替子页"**：父页 `btr_page_empty` 清空重建为 `page_level`（低一层），子页全部记录 `page_copy_rec_list_end` 拷进父页，释放子页，所有祖先页 `PAGE_LEVEL` 递减。树高减 1。
- **先存祖先页再动手**：`blocks[]` 在 lift 前把整条祖先链都搜出来，因为 `btr_page_empty` 后树暂时不一致，无法再搜。`btr_page_get_father_block` 每次向上爬一层。
- **`lift_father_up` 分支（8.0 新增）**：被 lift 的是叶子（`page_level == 0`）且父页不是根时，必须先 lift 父页。原因是 `btr_page_free` 按 level 选 segment：叶子页被 lift 进父页后 level 从 0 变了，若直接释放会按错 segment 释放（源码注释明写此坑）。
- `lock_update_copy_and_discard`：子页上的行锁随记录搬到父页（父页现在成了叶子）。

#### 页重整：`btr_page_reorganize`

整页重排的逻辑：`mtr_set_log_mode(mtr, MTR_LOG_NONE)` 关日志 → 页内容拷到临时块 → `page_create` 重建空页 → `page_copy_rec_list_end_no_locks` 把记录紧凑拷回（不拷锁位、不写 redo）→ 成功后只写**一条 `MLOG_PAGE_REORGANIZE`**（压缩页 `MLOG_ZIP_PAGE_REORGANIZE` + 压缩级别字节）。恢复端 `btr_parse_page_reorganize` 按 index 定义重排即可——**重整不改变数据内容，redo 只需说"这页重整过"**。开销之外还有两处必须小心：先 `btr_search_drop_page_hash_index` 摘 AHI；行锁 bitmap 经 `lock_move_reorganize_page` 迁移。调用时机：乐观插入放不下时、分裂后插入失败时、合并目标页放不下时。

### 更新：三条路径

#### 原地更新 `btr_cur_update_in_place`

前提 `row_upd_changes_field_size_or_external() == false`（所有字段大小不变、无 extern）：锁检查 + undo（`btr_cur_upd_lock_and_undo`，聚簇写 `TRX_UNDO_MODIFY_OP`，二级只做锁检查不写 undo）→ `row_upd_rec_in_place` 覆写字节 → 写 `MLOG_REC_UPDATE_IN_PLACE`（1 字节 flags + sys 字段 + 2 字节页内 offset + 变更字段差异）。若改动了排序字段，先 `btr_search_update_hash_on_delete` 摘 AHI 条目。**压缩页**要先 `btr_cur_update_alloc_zip` 预留 redo 修改空间，失败返回 `DB_ZIP_OVERFLOW`。

#### 乐观更新 `btr_cur_optimistic_update`

大小变了但**同页放得下**时，做 **delete + insert**（这是最反直觉处：乐观更新不是"改"，是"删了再插"）：

```cpp
if (!row_upd_changes_field_size_or_external(index, *offsets, update))
  return btr_cur_update_in_place(...);       // 大小不变 → 原地

if (rec_offs_any_extern(*offsets) || 新值含 extern) goto any_extern;  // → 悲观

// 边界检查：新记录 ≥ REC_MAX_DATA_SIZE / ≥ 空页一半 / 删完低于 merge 阈值 → 失败
...
err = btr_cur_upd_lock_and_undo(...);        // undo 先写
lock_rec_store_on_page_infimum(block, rec);  // ★ 显式锁暂存到 infimum
btr_search_update_hash_on_delete(cursor);
page_cur_delete_rec(page_cursor, index, *offsets, mtr);
page_cur_move_to_prev(page_cursor);
row_upd_index_entry_sys_field(new_entry, index, DATA_ROLL_PTR, roll_ptr);
row_upd_index_entry_sys_field(new_entry, index, DATA_TRX_ID, trx_id);
rec = btr_cur_insert_if_possible(cursor, new_entry, offsets, heap, mtr);
ut_a(rec);
lock_rec_restore_from_page_infimum(block, rec, block);  // ★ 锁归还
```

**`infimum` 充当锁的临时载体**是这里最精巧的设计：删除记录会连带删除挂在它上面的锁结构，所以先 `lock_rec_store_on_page_infimum` 把显式锁挪到 infimum 上，插入新记录后再 `lock_rec_restore_from_page_infimum` 归还——删除瞬间锁不丢失，对并发事务而言锁的"连续占有"从未中断。乐观更新还检查"更新后页太满"（放不下→OVERFLOW）与"太空"（低于 merge 阈值→**DB_UNDERFLOW**，意思是该合并了，转悲观）。

#### 悲观更新 `btr_cur_pessimistic_update`

顺序是 **先删旧记录 → 先试同页插入 → 放不下才悲观插入（可能分裂）**——不是"先悲观插新再悲观删旧"。原因：

- undo 在删旧记录前已写好（`btr_cur_upd_lock_and_undo`），删与插共享同一条 `TRX_UNDO_MODIFY_OP`；
- 游标在删除后 `move_to_prev` 到前驱，插入点就是旧记录位置，**先删后插能复用旧记录释放的空间**，避免不必要的分裂；
- 锁同样经 infimum 中转；若最终走了分裂（新记录落到别的页），`btr_cur_pess_upd_restore_supremum` 修正分裂中 supremum 继承的 gap 锁；
- 悲观插入用 `BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG | BTR_KEEP_SYS_FLAG`：undo/锁检查已在上面完成，绝不重复做；二级索引还要补 `page_update_max_trx_id`（供 purge 可见性判断）。

### 自适应哈希索引（AHI）

#### 结构：分片哈希系统（8.0.30+）

```cpp
class btr_search_sys_t {
 public:
  class search_part_t {
   public:
    alignas(ut::INNODB_CACHE_LINE_SIZE) rw_lock_t latch;      // 保护本分区
    alignas(ut::INNODB_CACHE_LINE_SIZE) hash_table_t *hash_table;
    std::atomic<buf_block_t *> free_block_for_heap;
  };
  ut::unique_ptr_aligned<search_part_t[]> parts;               // 1..512 个分区
};
```

- 分区数 = `btr_ahi_parts`（默认 8，`innodb_adaptive_hash_index_parts`，READONLY）；分区选择键 = `ut::hash_uint64_pair(space_id, index_id) % parts`（`fast_modulo_t` 加速）。
- 每个分区的 `hash_table_t` 以 **0 个内部同步对象**创建——整张子表由该分区自己的 rw-lock 保护，这是与 8.0.30 前"per-cell 锁"的本质区别。
- latch 与 hash_table 各自占独立 cache line，消除伪共享；`free_block_for_heap` 让 hash 节点分配不穿透锁。

索引级统计在 `dict_index_t::search_info`（类型 `btr_search_t`，8.0.30 前叫 `btr_search_info_t`）：

```cpp
struct btr_search_t {
  std::atomic<size_t> ref_count;        // 已建哈希的页数（摘除时递减，disable 要等清零）
  buf_block_t *root_guess;              // root block 缓存（搜索 hint）
  std::atomic<uint64_t> hash_analysis;  // 节流计数：< 17 直接返回（BTR_SEARCH_HASH_ANALYSIS）
  bool last_hash_succ;                  // 上次哈希是否成功（探测门控）
  std::atomic<uint64_t> n_hash_potential;   // 连续"潜在命中"计数
  std::atomic<btr_search_prefix_info_t> prefix_info;  // 推荐前缀 {n_fields, n_bytes, left_side}
};
```

块级状态在 `buf_block_t::ahi`：`index`（该页为谁建了哈希）、`prefix_info`（建时用的前缀）、`recommended_prefix_info`；`n_hash_helps` 计数页级命中。

#### 构建：双门槛 + 全 nowait

每次叶子搜索结束后 `btr_search_info_update`：`hash_analysis` 每搜索自增，**每 17 次**才进慢路径（省 CPU 的节流）。慢路径 `btr_search_info_update_hash` 验证"本次搜索若用推荐前缀做哈希是否会成功"：成功则 `n_hash_potential++`；失败则推荐失效，用 `up_match/low_match`/`up_bytes/low_bytes` **重新学习前缀**（取匹配字段/字节数，`cmp > 0` 表示上界更近 → `left_side=true` 缓存每组最左记录，反之 false）。

构建门槛（`btr_search_update_block_hash_info`）：

```cpp
if (info->n_hash_potential >= BTR_SEARCH_BUILD_LIMIT &&         // 100：索引级连续潜在命中
    block->n_hash_helps > page_get_n_recs(block->frame) / BTR_SEARCH_PAGE_BUILD_LIMIT)  // 页级 > n_recs/16
  return true;   // 才为这一页建哈希
```

`btr_search_build_page_hash_index`：从 infimum 顺序扫到 supremum，对每条记录 `rec_hash` 折叠前 `n_fields` 个字段 + 下一字段前 `n_bytes` 字节（种子 = `btr_search_hash_index_id` 的 (space, index_id) 哈希）；**相等前缀组只存一条**（left_side 存最左/否则最右），`ha_insert_for_hash` 入表。哈希表里存的是 **rec 指针**，页通过 `buf_block_from_ahi(rec)` 从地址反推。首次构建用 `btr_search_x_lock_nowait`——竞争立即放弃（"waiting here for the latch would defy the purpose"）；只有分裂/移动后的**强制重建**（`update=true`，经 `btr_search_update_hash_on_move`）才阻塞等锁，保证旧指针一定被改写。

#### 探测：`btr_search_guess_on_hash`

```cpp
cursor->flag = BTR_CUR_HASH_NOT_ATTEMPTED;
if (info->n_hash_potential == 0) return false;         // 无推荐前缀
hash_value = dtuple_hash(tuple, prefix_info.n_fields, prefix_info.n_bytes,
                         btr_hash_seed_for_record(index));
if (!has_search_latch && !btr_search_s_lock_nowait(index, ...)) return false;
rec = ha_search_and_get_data(btr_get_search_table(index), hash_value);
if (rec == nullptr) { cursor->flag = BTR_CUR_HASH_FAIL; return false; }
block = buf_block_from_ahi(rec);
if (!buf_page_get_known_nowait(latch_mode, block, Cache_hint::MAKE_YOUNG, ..., mtr))
  return false;                                        // 页锁拿不到：放弃，不等待
// 验证猜测：空间 id / index id 比对 + btr_search_check_guess 邻居比较
if (index->space != block->page.id.space() ||
    index->id != btr_page_get_index_id(block->frame) ||
    !btr_search_check_guess(cursor, has_search_latch, tuple, mode, mtr)) {
  btr_leaf_page_release(block, latch_mode, mtr); return false;
}
info->n_hash_potential++;
info->last_hash_succ = true;
cursor->flag = BTR_CUR_HASH;
return true;
```

命中后必须 `btr_search_check_guess` 验证：哈希只保证"这个 fold 的组内一条记录"，还要按搜索模式与相邻记录比较（GE 要比较 prev、LE 要比较 next），且校验页归属（防哈希冲突引错索引的页）。**miss 的代价**：一次 fold + 一次 nowait S 锁 + 一次链扫 + 一次 nowait 页锁，然后回退全树下钻——所以入口处用 `last_hash_succ` 门控（上次失败这次直接不试）。所有 `btr_cur_n_sea++`/`btr_cur_n_non_sea++` 计数就是 `SHOW ENGINE INNODB STATUS` 里的 "hash searches/s"。

#### 失效维护：insert/delete/move 三面同步

- **insert**：`btr_search_update_hash_node_on_insert`（本次就是哈希命中的快路径：把节点改指新记录，O(1)）或 `btr_search_update_hash_on_insert`（维护"每组一条"不变量，按 left_side 决定插新记录还是邻居）；两者先 `btr_search_check_free_space_in_heap` **预留** hash 节点空闲块——防止持 AHI 锁时触发 LRU 淘汰、淘汰又要摘 AHI 造成死锁环（`buf_block_alloc → buf_LRU_free_page → btr_search_drop_page_hash_index`）。
- **delete**：`btr_search_update_hash_on_delete`（阻塞 X 锁，因为残留死指针是错误，不能放弃）。
- **move/split**：`btr_search_update_hash_on_move` 新页可用同前缀重建则 `update=true` 强制重建，否则摘掉旧块全部条目。
- **页被释放/淘汰**：`btr_search_drop_page_hash_when_freed`（fsp 释放页时）与 `buf_LRU_free_page` 中的 `btr_search_drop_page_hash_index(block, true)`。
- **disable 协议**：`innodb_adaptive_hash_index` 动态关闭时 `btr_search_disable` 对 `dict_sys->table_LRU/table_non_LRU` 逐表 `btr_search_await_no_reference` 等 `ref_count` 清零（10ms 轮询，600 秒未清零则报错自杀）。

### 批量构建：`Btree_load`

DDL 建索引（ADD INDEX / 重建聚簇）不走单条插入，走 **排序批量构建**——输入记录流已全局有序，因此可以做三件单条插入做不到的事：**零搜索**（永远链尾追加）、**零分裂**（普通页永不分裂）、**零 redo**（`MTR_LOG_NO_REDO`）。

#### 页面构建器 `Page_load`：页尾追加 + finish 一次成型

```cpp
// 插入（热路径）：堆顶顺序分配，永远插在链尾，debug 版只断言有序
page_header_set_ptr(m_page, nullptr, PAGE_HEAP_TOP, m_heap_top + rec_size);
auto insert_rec = rec_copy(m_heap_top, rec, offsets);     // 直接拷到 heap top
rec_t *next_rec = page_rec_get_next(m_cur_rec);
page_rec_set_next(insert_rec, next_rec);                  // 链表挂到 m_cur_rec 之后
page_rec_set_next(m_cur_rec, insert_rec);
m_heap_top += rec_size; m_rec_no += 1; m_cur_rec = insert_rec;
```

页目录不在插入时维护——`finish()` 收尾时 O(n) 均分：每 4 条一个槽（`RECORDS_PER_SLOT = (PAGE_DIR_SLOT_MAX_N_OWNED + 1) / 2`），余数挂 supremum 槽，最后一次性写 `PAGE_HEAP_TOP/PAGE_N_RECS/PAGE_LAST_INSERT/PAGE_DIRECTION` 等页头。对比单条插入的"每条记录目录槽二分 + 可能槽分裂 + 写 redo"，这是数量级的简化。

**填充策略**：`innodb_fill_factor`（10–100，默认 100）决定每页预留 `UNIV_PAGE_SIZE * (100 - fill_factor) / 100` 字节；fill_factor=100 时聚簇索引仍保留 `dict_index_get_space_reserve()`（1/16 页，兼容 5.6 行为，为行内更新留余地），二级索引零预留（100% 打包）。空间判定 `is_space_available` 还保证**每页至少 2 条记录**（防树高膨胀）。

#### 自底向上的层级生长：`Btree_load::insert` + `page_commit`

```cpp
dberr_t Btree_load::insert(dtuple_t *tuple, size_t level) noexcept {
  if (level + 1 > m_page_loaders.size()) {     // 该层还没有页：新建（树高+1）
    m_page_loaders.push_back(new Page_load(...));
    m_root_level = level;
    is_left_most = true;
  }
  ...
  if (is_left_most && level > 0 && page_loader->get_rec_no() == 0)
    dtuple_set_info_bits(tuple, info_bits | REC_INFO_MIN_REC_FLAG);  // 最左 node pointer
  ...
  err = prepare_space(page_loader, level, rec_size);   // 页满 → finish + 新页 + page_commit
  ...
}
```

`m_page_loaders[level]` 永远指向该层**当前装填中的最右页**；叶子页满时 `page_commit` 把本页收尾、向 level+1 插一个 node pointer（key = 页首记录，`dict_index_build_node_ptr`）——上层页又满就再向上，树**自底向上自然生长**，根层永远最后创建。mtr 策略：每页一个 mtr（NO_REDO），非叶页每插一个 node pointer 就 `release()`（commit 但 block 保持 buffer-fixed + 记 `m_modify_clock`），下次 `latch()` 乐观重拿；叶子页跨多条记录持续持有 X latch（热路径）。

**redo 与持久化**：页数据零 redo，但**页面分配**走单独的 `alloc_mtr`（正常日志模式）——"页被投入使用"这件事由分配日志保证（`MLOG_INIT_FILE_PAGE2` 的语义），页内容靠**刷盘**持久化：每切一个叶子页 `log_free_check`（NO_REDO 页必须落盘才能推进 LSN 水位），`Flush_observer` 跟踪全部 NO_REDO 脏页，DDL 收尾 `Context::cleanup → observer->flush()` 全部刷盘后才允许 redo 记录的操作触碰该索引。崩溃恢复的正确性来自：要么页被刷盘（内容完整），要么分配日志不完整（整页作废重来）——这就是"bulk 页不写 redo"能成立的边界。

**收尾**：`finalize_page_loads` 逐层提交每层最后一页；`load_root_page` 把顶层最后一页**整页拷到真正的 root 页**（root 页号在 `ddl::create_index` 时已预分配），`btr_page_free_low` 释放临时页。调用链：`ha_innobase::inplace_alter_table_impl → ddl::Context::build → ddl::Loader::build_all → Builder::btree_build（Merge_cursor 归并排序输入）→ Btree_load::build`；重建聚簇且键序不变时走 `Builder::insert_direct`（内存 `Key_sort_buffer_cursor`，不落临时文件）。R-tree 和 FTS 不走 btr0load（前者扫描期逐条 `RTree_inserter`，后者 `fts_sort_and_build`）。

### 空闲页与树统计

- `btr_page_free` → `btr_page_free_low`：按 `level == 0` 选叶子段/非叶段，`fseg_free_page` 还回 fseg（碎片页直接还表空间，整 extent 全空则整 extent 归还）；`buf_block_modify_clock_inc` 使 AHI 与乐观恢复失效；页保持 buffer-fixed 直到 mtr 提交。
- **ibuf 树例外**：change buffer 树的空闲页不还 fseg，挂根页的 `PAGE_BTR_IBUF_FREE_LIST` 链表（`btr_page_free_for_ibuf` / `btr_page_alloc_for_ibuf`）——ibuf 页频繁分配释放，自留缓存池。
- 整树释放 `btr_free_if_exists`：`btr_free_but_not_root` 用 `fseg_free_step` **每个 mtr 只释放一小步**（防超大索引把单个 mtr 撑爆），最后 `btr_free_root_invalidate` 把根页 `PAGE_INDEX_ID` 置 0（`BTR_FREED_INDEX_ID`），防 index_id 复用误判。
- 统计 `btr_get_size`（`BTR_N_LEAF_PAGES`/`BTR_TOTAL_SIZE`）直接读两个 fseg 的保留页数；高度 `btr_height_get` 读根页 `PAGE_LEVEL`。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_adaptive_hash_index` | ON | Global | AHI 总开关，可动态切换（`btr_search_enable`/`btr_search_disable`）；关闭时逐表等 `ref_count` 清零后清空哈希 |
| `innodb_adaptive_hash_index_parts` | 8 | Global, READONLY | AHI 分区数（1–512），8.0.30 引入，对应 `btr_ahi_parts` |
| `innodb_fill_factor` | 100 | Global | bulk 构建每页填充百分比（10–100）；100 时聚簇仍保留约 1/16 页给更新，二级索引全满 |

> merge 阈值本身**没有社区变量**——它固化在 `SYS_INDEXES.MERGE_THRESHOLD`（默认 `DICT_INDEX_MERGE_THRESHOLD_DEFAULT = 50`），`dict_index_set_merge_threshold` 写 DD。调试构建才有 `innodb_merge_threshold_set_debug`。5.7 时代的 `BTR_CUR_PAGE_FILL_FACTOR`（分裂填充变量）在 8.0 已移除，分裂按字节空间 50% 硬编码（`btr_page_get_split_rec` 的 `total_space / 2`）。

### 状态变量

| 变量名 | 说明 |
|--------|------|
| `btr_cur_n_sea` / `btr_cur_n_non_sea` | AHI 命中/未命中搜索计数，`SHOW ENGINE INNODB STATUS` 输出为 "hash searches/s, non-hash searches/s" |
| `MONITOR_INDEX_SPLIT` | 页分裂次数 |
| `MONITOR_ADAPTIVE_HASH_PAGE_ADDED/PAGE_REMOVED/ROW_ADDED/ROW_REMOVED/ROW_UPDATED` | AHI 页/行级别维护计数 |
| `MONITOR_ADAPTIVE_HASH_SEARCH` / `MONITOR_ADAPTIVE_HASH_SEARCH_BTREE` | AHI 命中搜索 / 回退 B-tree 搜索 |

---

## Misc

### 三组"乐观/悲观"别混淆

| 语境 | 乐观 | 悲观 |
|------|------|------|
| **DML 路径**（本篇） | `BTR_MODIFY_LEAF`：index S 锁 + 只 X 锁叶子，假设不动树 | `BTR_MODIFY_TREE`：index SX 锁 + 路径页 X，允许分裂/合并 |
| **pcur 位置恢复** | `modify_clock` 未变 → `buf_page_optimistic_get` 直接复用记录指针 | clock 变了/页淘汰 → 用 old_rec 前缀从 root 重搜 |
| **压缩页 reorg** | `page_zip` 就地微调 | 整页 `page_zip_compress` 重压缩 |

### "此 X 非彼 X"

- `btr_cur_t`（mtr 内短命游标）≠ `btr_pcur_t`（跨 mtr，组合了前者）≠ `page_cur_t`（页内 (block, rec) 二元组，三者层层包含）。
- **node pointer**（非叶层记录：子页首 key + 4 字节页号）≠ **index record**（旧资料的叫法）≠ 二级索引里的"主键指针"。node pointer 的 key 是**下界**，这正是非叶层搜索要把 `GE→L`、`G→LE` 转换的原因。
- `BTR_CUR_PAGE_COMPRESS_LIMIT`（merge 阈值线，`UNIV_PAGE_SIZE * merge_threshold / 100`）与 `BTR_CUR_PAGE_REORGANIZE_LIMIT`（重整收益线，`UNIV_PAGE_SIZE / 32`）是两个不同用途的阈值。
- 目录槽（page directory slot，页内稀疏索引）与 AHI 哈希表（内存中 (fold → rec) 映射）都是"索引的索引"，但一个持久、一个纯缓存。
- `MLOG_LIST_END_COPY/DELETE`（列表级搬移日志，日志体 2 字节）≠ `MLOG_REC_INSERT`（单条插入差异日志）≠ `MLOG_PAGE_REORGANIZE`（重整"通知型"日志）。

### 8.0.39 中已不存在的旧符号（外部资料常见，勿照抄）

| 旧符号 | 8.0.39 实际 |
|--------|------------|
| `BTR_CUR_PAGE_FILL_FACTOR` / `btr_cur_page_fill_factor` | 已删，分裂固定 50% |
| `btr_page_merge` / `btr_page_merge_low` | `btr_compress` |
| `btr_free_page` / `btr_free_but_not_delete` | `btr_page_free` / `btr_free_but_not_root` |
| `node_ptr_optimistic_delete` | `btr_node_ptr_delete` + `btr_insert_on_non_leaf_level` |
| `btr_cur_latch_for_insert` / `btr_cur_save_path_position` | `btr_cur_latch_for_root_leaf` / 已删（路径保留由搜索内部完成） |
| `btr_search_info_t` / `btr_search_guess_on_hash_func` | `btr_search_t` / `btr_search_guess_on_hash` |
| `btr_load_index` / `btr_load_add_block` / `BTR_BULK_INSERT` | `Btree_load::build` / `Page_load` 体系 / `MTR_LOG_NO_REDO` + `ddl::fill_factor` |
| `btr_store_big_rec_extern_fields`（btr0cur 静态版）/ `MLOG_INDEX_EXTERN` | `lob::btr_store_big_rec_extern_fields` / 普通页日志（LOB 重构） |

---

## 参考

**论文**
- Bayer, R. & McCreight, E. *Organization and Maintenance of Large Ordered Indexes*. Acta Informatica, 1972.（B-tree 起源；InnoDB 的 node pointer/分裂合并直接对应）
- Lehman, P. & Yao, S. *Efficient Locking for Concurrent Operations on B-Trees*. TODS, 1981.（B-link tree；InnoDB 明确否决此路线，见 btr0btr.cc 文件头 "not a B-link tree"）
- Mohan, C. et al. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. TODS, 1992.（physiological redo；列表级搬移日志与重整日志的理论来源）
- Graefe, G. *A Survey of B-Tree Locking Techniques*. TODS, 2010.（latch coupling、兄弟页锁的经典梳理）

**官方文档**
- *MySQL 8.0 Reference Manual → 17.5.3 Adaptive Hash Index*（AHI 适用场景与关闭建议）
- *MySQL 8.0.30 Release Notes*（AHI search latch 竞争问题与 `innodb_adaptive_hash_index_parts` 引入）
- *MySQL 5.7 Reference Manual → innodb_fill_factor*（bulk load 填充变量沿革）
- WorkLog: [WL#7277](https://dev.mysql.com/worklog/task/?id=7277) InnoDB fast index creation（btr0bulk 排序批量构建）

**内核月报 / 技术文章**
- [Innodb 中的 Btree 实现 (一) · 引言 & insert 篇](http://mysql.taobao.org/monthly/2022/12/03/)（月报 2022/12，阿里 RDS 内核组）
- [Innodb 中的 Btree 实现 (二) · select 篇](http://mysql.taobao.org/monthly/2023/07/03/)（月报 2023/07）
- [MySQL · 内核特性 · InnoDB btree latch 优化历程](http://mysql.taobao.org/monthly/2020/06/01/)（月报 2020/06，X→SX 演进动机）
- [MySQL · 源码分析 · btr_cur_search_to_nth_level 函数分析](http://mysql.taobao.org/monthly/2021/07/02/)（月报 2021/07）
- [MySQL · 引擎特性 · InnoDB Adaptive Hash Index 介绍](http://mysql.taobao.org/monthly/2017/01/01/)（月报 2017/01）

> 注意：月报文章基于 8.0.13 及更早版本，其函数名/行号与 8.0.39 有出入（如 `btr_cur_latch_for_insert`、`btr_search_info_t` 已更名），本文所有函数名均以 8.0.39 源码为准。

**相关文档**
- 上游 SQL 层行读取主链（`row_search_mvcc`、行缓冲转换）见 [`row_search.md`](row_search.md)
- 下游页结构（页头/目录槽/页内二分 `page_cur_search_with_match` 详情）见 [`physical/page_structure.md`](physical/page_structure.md)
- 记录格式与 offsets（`rec_get_offsets`、node pointer 字段解析）见 [`physical/record.md`](physical/record.md)
- 行锁/间隙锁/latch 体系（`index->lock`、`lock_update_split_*` 的锁继承）见 [`lock.md`](lock.md)
- mtr 与 redo 日志（`MLOG_LIST_*` 重放、`mtr_set_log_mode`）见 [`redo_log.md`](redo_log.md)
- undo 与 purge（物理删除入口、delete-mark）见 [`undo_log.md`](undo_log.md)
- LOB 外存字段（`lob::BtrContext`、extern 引用前缀）见 [`physical/lob.md`](physical/lob.md)
- online DDL 与并行构建（`ddl::Loader`/`Builder` 驱动 `Btree_load` 的上下文）见 [`ddl.md`](ddl.md)
- 可见性判断（`up_match/low_match` 供重复键检测、PAGE_MAX_TRX_ID）见 [`mvcc.md`](mvcc.md)



