# MySQL 8.0.39 源码知识库

> 基于 MySQL 8.0.39 源码的深度解析文档。重点讲设计思想、理论溯源与核心实现；结论均经源码核实。

## 从哪开始

**想理解一条 SQL 怎么执行** → [`server/query/README.md`](server/query/README.md)（主链全景 + 6 层中间表示图）

**想查某个具体机制** → 用下面的目录索引

---

## 一条 SQL 的完整旅程（跨目录）

```
客户端
  │
  ▼
┌──────────────────────────────────────────────────┐
│ server/query/    协议 → 解析 → 优化 → 迭代器执行  │  ← 主链，27 篇
└──────────────────┬───────────────────────────────┘
                   │ ha_rnd_next / ha_index_next
┌──────────────────▼───────────────────────────────┐
│ server/handler.md   handler / handlerton 分界面   │
└──────────────────┬───────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────┐
│ innodb/          buffer pool / MVCC / 锁 / 日志   │
└──────────────────┬───────────────────────────────┘
                   │ 提交
┌──────────────────▼───────────────────────────────┐
│ server/replication/   binlog 组提交 → 复制         │
└──────────────────────────────────────────────────┘
```

---

## 目录索引

### server/ —— Server 层

**SQL 处理主链**：见 **[query/README.md](server/query/README.md)**（27 篇：协议分发 → 解析 → contextualize → prepare → 优化 → AccessPath → 迭代器执行 → DML，含 `06_optimize/` 优化器 11 篇、`runtime/` 执行期专题 6 篇）

| 文件 / 目录 | 内容 |
|---|---|
| [handler.md](server/handler.md) | server ↔ 存储引擎分界面：handler/handlerton 分层、prebuilt、行定位 |
| [mdl.md](server/mdl.md) | MDL 元数据锁：表结构保护、双兼容性矩阵排队语义、wait-for graph 死锁检测 |
| [auth/](server/auth/) | 认证与授权：security_context.md（认证上下文）、mfa.md（多因素认证）、definer.md（definer 与 SQL SECURITY） |
| [replication/](server/replication/) | 复制：binlog.md（格式/组提交/2PC）、gtid.md、replication.md（主从）、prpl.md（并行复制） |
| [datatype/](server/datatype/) | 数据类型：json.md（JSON 二进制/部分更新/索引）、gis.md（空间/R-tree） |
| [infra/](server/infra/) | 通用机制：list.md（侵入式链表）、dbug.md（DBUG 框架）、pfs_and_memory.md（PFS 与内存）、vio.md（VIO 通信抽象） |

### [innodb/](innodb/) —— 存储引擎

| 文件 | 内容 |
|------|------|
| [buffer_pool.md](innodb/buffer_pool.md) | LRU、flush list、预读、page cleaner |
| [mvcc.md](innodb/mvcc.md) | read view、可见性判断、半一致性读 |
| [row_search.md](innodb/row_search.md) | `row_search_mvcc`、游标推进（承接 handler.md） |
| [page_structure.md](innodb/page_structure.md) | **B+ 树页物理结构**：FIL header/page header、infimum/supremum、heap + 单向链表 + page directory 稀疏索引、`page_cur_search` 页内二分 |
| [lock.md](innodb/lock.md) | 行锁、间隙锁、死锁检测 |
| [redo_log.md](innodb/redo_log.md) | redo 格式、LSN、崩溃恢复 |
| [undo_log.md](innodb/undo_log.md) | undo 表空间、purge |
| [query_graph.md](innodb/query_graph.md) | InnoDB 内部执行模型（fork/thr/node） |
| [lob.md](innodb/lob.md) | 大对象存储（JSON 部分更新的物理基础） |
| [ddl.md](innodb/ddl.md) | online DDL |
| [fts.md](innodb/fts.md) | 全文索引 |
| [parallel_scan.md](innodb/parallel_scan.md) | InnoDB 内部并行扫描（`Parallel_reader`）+ 澄清社区无 SQL 层并行查询 |
| [tablespace.md](innodb/tablespace.md) | 表空间碎片度量（`DATA_FREE` / `FREE_EXTENTS` 计算链路）、两层碎片辨析 |
| [io.md](innodb/io.md) | I/O 路径：Buffered I/O vs O_DIRECT、`innodb_flush_method`、redo 刷盘、double buffering |
| [trx.md](innodb/trx.md) | InnoDB 事务（`trx_t`） |

### [feat/](feat/) —— 跨层端到端特性

> 这些主题**同时改动多个层**（语法 → 优化器 → 引擎存储），按"特性"归位，不按层拆开。判据见下面「第一步：决定放哪」与「边界归属」两节；**篇内怎么组织（按层分节 vs 生命周期）见 [`feat/README.md`](feat/README.md)**。

| 文件 | 内容 |
|---|---|
| [partitioning.md](feat/partitioning.md) | **分区表**（全链路单篇）：`partition_info` 元数据模型、DD 持久化与"DD→文本→重解析"往返、分区裁剪（假索引复用 range 优化器）、`Partition_helper` 的 DML 路由、`ha_innopart` 的分区上下文切换与 per-partition `dict_table_t`、TRUNCATE/EXCHANGE/ADD-DROP PARTITION、分区统计与能力边界 |

### [log/](log/) —— 服务器日志

[error_log.md](log/error_log.md) · [general_log.md](log/general_log.md) · [slow_log.md](log/slow_log.md)

### [cloud/](cloud/) —— 云环境

[cloud_db.md](cloud/cloud_db.md) · [cloud_networking.md](cloud/cloud_networking.md)

---

## 如何新增文档

> 单篇的写法（章节骨架、必填项）见 [`_template.md`](_template.md)。本节讲**文档之间**的组织规则：放哪、怎么编号、边界怎么划。

### 第一步：决定放哪

```
新主题
  │
  ├─ ★ 是"跨层端到端特性"吗？（同时改 SQL 层与引擎层：分区、全文索引、GIS…）
  │     是 → feat/，一篇讲完整条链路；引擎差异写进篇内小节，
  │          不为每个引擎单独开篇（详见「边界归属」）
  │
  ├─ 是"一条 SQL 必经的一步"吗？
  │     是 → server/query/ 下加数字编号（如 09_dml.md）
  │
  ├─ 是"主链某一步的横向展开"吗？（该步内部还有大量独立机制）
  │     是 → 进该步的子目录（06_optimize/ 或 runtime/）
  │
  ├─ 是"观察/调试主链的工具"吗？（EXPLAIN、trace）
  │     是 → server/query/ 下，但在 README 归入「横切内容」，不占主链步骤
  │
  ├─ 是 server 与存储引擎的接口本身吗？
  │     是 → server/handler.md（唯一的边界文档，留在 server/ 根）
  │
  ├─ 是存储引擎内部实现吗？
  │     是 → innodb/
  │
  └─ 其他 server 层主题
        ├─ 数据类型（JSON/GIS/时间类型…）  → server/datatype/
        ├─ binlog 与复制                    → server/replication/
        ├─ 被全局复用的通用机制             → server/infra/
        └─ 服务器日志                       → log/
```

### 第二步：判断该独立成篇，还是并入现有篇

这是最容易做错的一步。两个方向都要考虑：

**该独立成篇** —— 满足任一条：

- 有**自己的一整套数据结构**，讲不透就会拖累宿主篇
  （例：range 优化有 `SEL_TREE`/`SEL_ARG` 区间森林 + 六种访问方法，从"访问方法选择"里独立出来成 `08_range_optimizer.md`）
- 是**另一条并行的主链**，而非现有链的分支
  （例：DML 与 SELECT 共用前段但执行完全不同，独立为 `09_dml.md`，没有塞进 `runtime/`）
- 是**通用子系统**，被多个上层复用
  （例：LOB 是 InnoDB 通用机制，从 `json.md` 拆出为 `innodb/lob.md`）

**该并入现有篇** —— 满足任一条：

- 拆开会让**主链断头**，读者走到边界就掉出目录
  （例：协议层收包原为 `server/net.md`，并入 `query/01`——否则主链第一步"SQL 文本从哪来"在目录外）
- 内容是**同一件事的两半**，分开必有一半无处安放
  （例：收包与回包是协议层的两个方向，独立成文会让回包没地方写）
- 篇幅撑不起一篇，且没有独立的数据结构

### 第三步：够不够建子目录

- **同一阶段 ≥ 3~4 篇** → 建子目录
  （例：逻辑优化 4 篇 → `logical/`，物理优化 4 篇 → `physical/`）
- **只有 1 篇** → 留在上级目录，**不建单文件目录**
  （例：`10_plan_refinement.md` 是计划改进阶段的唯一一篇，留在 `06_optimize/` 根）
- 建了子目录后，上级 README 必须**按阶段分组**展示，否则读者看不出层次

### 边界归属：第一主语原则

跨层主题（一件事横跨 SQL 层、接口层、引擎层）按**"第一主语是谁"**归位，而不是按"涉及谁"：

| 内容 | 第一主语 | 归属 |
|---|---|---|
| 迭代器怎么调 handler 取行 | SQL 层执行器 | `query/09_executor_iterator.md` |
| `position`/`ref`/`rnd_pos` 的接口语义 | 存储引擎接口 | `server/handler.md` |
| `row_search_mvcc`、游标推进 | InnoDB | `innodb/row_search.md` |
| DML 的 buffer row id 两阶段读 | SQL 层 DML | `query/09_dml.md` |

一份内容只在**一处**详写，其他篇用交叉引用指向它——不要因为"这里也相关"就复制一遍。跨层主题在文首加一行"边界"说明谁讲什么。

**例外：跨层端到端特性不走按层归位，进 `feat/`。** 判据（满足任一）：

1. **同时改动多个层**：分区表同时改 SQL 层（语法、元数据、优化器、MDL）与引擎层（存储、分区 DDL）；全文索引、GIS 同理。
2. **可能被多个引擎实现**：按引擎拆篇会让 N 个引擎变成 N 篇，且每篇都要复述一遍 server 层。正确做法是一篇 + "引擎差异"小节（例：分区表只 InnoDB/NDB 两家，NDB 走 `HA_USE_AUTO_PARTITION` 自动分区并保留 `nodegroup_id`，未声明 native 分区能力的引擎直接被拒）。
3. **读者需要一次读完闭环**：分区表的 DML 路径横跨 `ph_write_row`（SQL 层算分区号）与 `set_partition`（引擎换 `dict_table_t`），中间只隔一层虚调用，拆开则两篇都断头。

不属于此类的仍按层归位：纯引擎内部机制（buffer pool、redo）→ `innodb/`；纯 SQL 层机制（MDL、优化器）→ `server/`；**两层接口本身** → `server/handler.md`（它属于"接口"而非"特性"）。

### 编号规则

- 主链用数字前缀表示**时间顺序**（`01_` → `09_`）
- 子目录内**编号全局连续、不重置**：`06_optimize/logical/02_` 一直排到 `physical/09_`
  - 这样跨篇引用"见 03 篇"**永远有效**，移动文件不必改引用
  - 用**路径**区分命名空间（写 `06_optimize/01_cost_model`），不靠"移出去平级"来避免编号冲突
- 横切内容（`runtime/`、EXPLAIN）不参与主链编号语义，README 里单独分组

### 命名

- 全库**英文文件名**、小写、下划线分隔
- 名字要能看出**内容归属**：`04_logical_join.md` / `06_join_order.md` 一眼能分清逻辑优化和物理优化；反例是曾经的 `06_join_order_access.md`，把 greedy + range + const 全挤在一起，看不出是物理优化

### 收尾检查清单

新增或移动文档后，这四件必做：

1. **改上级 README** 的文件索引（含新篇的一句话内容摘要）
2. **改被影响篇的交叉引用**（尤其是拆分/合并时，原篇的指向会失效）
3. **跑链接校验**——相对路径层级最易错（`../` 还是 `../../`）：
   ```bash
   cd .codebuddy/knowledge_for_mysql
   for f in $(find . -name "*.md"); do d=$(dirname "$f"); \
     for l in $(grep -o "](\([^)#]*\.md\)[^)]*)" "$f" | sed 's/](//;s/)$//;s/#.*//'); do \
       [ -e "$d/$l" ] || echo "BROKEN: $f -> $l"; done; done
   ```
4. **写好本篇的「参考」章节**（论文 / 官方文档 / 内核月报，按相关性取舍）

---

## 准入标准

> 上一章讲**放哪**（组织），本章讲**够不够格进、要写成什么样**（质量）。
>
> 本章每一条都来自真实返工——不是假想的规范。

### 零、先明确：什么是好文档

**目标是内核月报式的深度剖析**——讲透设计思想与核心算法，**不是**教科书式的"是什么 / 有哪些 / 怎么配置"罗列。

读者是"想真正理解 MySQL 设计的人"，不是"想查某个函数在第几行的人"。

一篇合格的文档要能回答三个问题，按重要性排序：

| # | 问题 | 落在哪一章 |
|---|---|---|
| **1** | **怎么设计的？为什么这么设计？用了什么思想理论？**（权衡取舍、被否决的方案、论文溯源、他库对比） | 「理论基础」——本章是文档价值的核心 |
| **2** | **重点难点是什么，怎么讲清？**（反直觉处、隐含假设、复杂状态机） | 「核心实现」——**贴关键代码 + 逐段解释算法**，只给结论等于没写透 |
| **3** | **怎么演进到今天的？**（各版本变了什么，以及**为什么变**） | 「概述·版本演进」记变化清单；「理论基础」记为什么变 |

**加一条结构要求：模块要闭环。** 读者读完这一篇（或这个子目录），对该模块应形成完整认知，不留"讲了一半"的断口——上游谁调它、下游它调谁、边界在哪，都要交代（哪怕只一句交叉引用）。

> **不写源码位置索引。** 读者会自己看代码，不需要文档告诉他"在哪个文件第几行"。函数名/结构名照常出现——它们是**指称用的术语**（讨论 `choose_table_order` 做了什么），不是坐标。也不要专门列"源码位置速查表"。
>
> 已有文档里的行号和速查表不必回头删（仍然有效），新增内容按上面来。

### 一、三条硬门槛

| 门槛 | 含义 | 不满足的表现 |
|---|---|---|
| **有理论基础** | 讲清设计思想、权衡、理论溯源 | 只有"代码这么写的"，没有"为什么这么写" |
| **讲透实现** | 关键代码贴出来并逐段解释算法 | 只列结论、只给函数名清单 |
| **闭环** | 该模块认知完整，边界与上下游交代清楚 | 读完不知道这一步之后发生什么 |

**说的东西必须真实存在**——函数名、结构成员、默认值都经本仓库源码核实（见「三、事实核验」）。

**基线版本是本仓库的 MySQL 8.0.39。** 其他版本只在"版本演进"里作为对比出现。

### 二、重点难点怎么讲

一篇里通常只有**一两个真正的难点**，把它们讲透比均匀铺开更有价值。哪些算难点：

- **反直觉**：排序/物化/聚合发生在 `Init()` 而非 `Read()`；迭代器不拥有行缓冲
- **易混淆的命名**：handler 的 `ref`（行位置）与 EXPLAIN 的 `ref` 列（访问类型）毫无关系；PSI batch mode 是**监控**批量不是计算批量
- **有隐含假设**：ROR-intersection 的选择率乘数把不同列当独立相乘——这既是设计简化，也是 bad case 的根因
- **状态机复杂**：NLJ 的 NULL 补全为什么必须区分"第一条内层行"

讲透的标准：**解释清"为什么不得不这样"**，而不只是"它是这样"。

> 设计里的简化假设带来的**失效场景**、以及**阈值下的退化行为**（如 `eq_range_index_dive_limit` 只缓解 index dive、不缓解 SEL_ARG 树构造），写在「理论基础 → 设计思想与权衡」里——它们是权衡的另一半，跟产生它们的设计决策放在一起才讲得通。

### 三、事实核验：六类高频错误

**这六类是本知识库真实踩过的坑**（虽然定位精度要求降低了，但"说的东西必须真实存在"这条不放松）：

#### A. 函数名不存在（最高频）

来源：注释里的**伪代码**、**历史版本**名字、**其他分支**（MariaDB/Percona）名字。

| 写错的名字 | 8.0.39 实际 | 错因 |
|---|---|---|
| `find_min_ror_intersection_scan` | `get_best_ror_intersect` + `find_intersect_order` | 只是伪代码注释 |
| `Optimize_table_order::best()` | `choose_table_order()` | 5.x 历史名字 |
| `prepare_unique` | 已内联进 `IndexMergeIterator::Init()` | **头注释里的方法名**，代码已重构 |
| `MRR_HANDLER_BUFFER` | `HANDLER_BUFFER` | MariaDB 的名字 |
| `ha_autocommit_or_rollback` | `trans_commit_stmt` | 已删除 |

> ★ **注释里出现的函数名不算存在**——头注释常年不更新，`index_merge.h` 至今写着"implemented in `prepare_unique`"，而那个方法早就没了。每个函数名都要 grep 确认。

#### B. 结构成员已删除或从不存在

`POSITION::sjmat_lookup_tables`（8.0.39 已删）、`IndexMergeIterator::m_dedup`（不存在，是优化器侧局部变量）、`TableRowIterator::m_expected_rows`（在各子类上，不在基类）。

> 贴结构体时打开头文件逐字段对照，不凭记忆列字段。

#### C. 默认值凭印象

`subquery_to_derived` 默认 **OFF**（常被误认为 ON）；`KEY_COMPARE_COST` 是 **0.05**（不是 0.1）。

> 必须从 `sys_vars.cc` / `opt_costconstants.cc` 的定义处读出。

#### D. 实现机制的版本前提变了

8.0.39 的**多表** UPDATE 延迟缓冲用**临时表**，`open_cached_file + position()` 只用于**单表** UPDATE 改扫描键的快路径。旧资料混为一谈。

> 写"X 用 Y 实现"时，确认这条路径在 8.0.39 真的还走 Y。

#### E. 把别家特性当社区特性

| 特性 | 实际归属 |
|---|---|
| `inlist2join` | Percona Server |
| table elimination | MariaDB |
| 列相关性 / dependency 统计 | PolarDB 在做，社区**无** |
| 向量执行、列式加速 | HeatWave（商业闭源），社区**无** |
| SQL 层并行查询 | 社区**无**（只有 InnoDB 内部 `Parallel_reader`） |

> 读商业版/其他分支资料时逐个 grep 确认。**社区没有的，宁可写"没有"也不能含糊带过。**

#### F. 把注释/文档当实现

头注释描述的算法可能已被重构（见 A 的 `prepare_unique`）。**以实际代码为准，注释只作线索。**

### 四、外部材料（月报 / 论文 / 博客）怎么消化

**外部材料提供视角与理论，源码提供事实。**

这几轮的经验：**月报的函数名和行号几乎全部对不上 8.0.39**（多基于 5.7 或某分支）。流程固定三步：

1. **读材料，提取"问题视角"与"理论脉络"**：它关注什么问题、指出什么 bad case、背后是哪篇论文的思路
2. **回源码核实机制与函数名**
3. **写文档时以源码为准，并显式记录纠正**

案例：月报《Index-Merge 代价估算原理》给了极好的问题视角（为什么 index merge 常选错），但它的 4 个函数名在 8.0.39 全不存在。最终文档保留了**算法思路和 bad case 根因**，函数名全部换成源码实际的。

> **禁止**照抄外部材料的函数名/公式而不核实。这是本知识库返工最多的原因。

### 五、什么不该进

| 不该进 | 处理 |
|---|---|
| 无法落到源码的传闻、性能玄学 | 不写 |
| 别家分支/商业版的私有实现细节 | 只在「理论基础 → 他库对比」里提及，明确标注归属 |
| 从外部材料抄来但没核实的内容 | 核实后再写 |
| 已被本版本删除的机制 | 只进"版本演进"作为沿革 |
| 同一内容在第二处重写 | 交叉引用指向唯一详写处 |

### 六、"社区没有 X" 这类澄清怎么处理

读者常问相邻概念（向量执行？并行查询？IN list 转 join？）。这类澄清**有价值**，但要：

1. **给证据**（"全库搜 `inlist2join` 返回 0 匹配"），不是断言
2. **说清替代品**——社区用什么解决同类问题
3. **说清谁有**——哪个分支/商业版的特性
4. **不为"没有"单独造一篇**——挂在最相关的篇里

> 例外：`innodb/parallel_scan.md` 单独成篇，因为有真实的 `Parallel_reader` 可讲，"社区无 SQL 层并行查询"只是其中一节。

### 七、提交前自检

**内容五条**（组织四条见上一章「收尾检查清单」）：

1. **「理论基础」写了吗**——设计思想、权衡、理论溯源、演进动机
2. **关键代码贴了并逐段解释了吗**——只给结论等于没写透
3. **模块闭环吗**——上下游、边界都交代了吗
4. **难点讲透了吗**——能解释"为什么不得不这样"，不只是"它是这样"
5. **函数名/成员/默认值核实过吗**——不照抄外部材料、不信注释里的函数名

---

## 其他

[`_template.md`](_template.md) 单篇文档模板（含动笔前自检清单）

参考资料（论文 / 官方文档 / 内核月报 / 书籍）在**每篇文档末尾的「参考」章节**，理论溯源在各篇的「理论基础」章节。
