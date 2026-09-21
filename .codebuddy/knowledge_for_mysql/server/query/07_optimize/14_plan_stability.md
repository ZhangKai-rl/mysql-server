# 14 计划稳定性与优化器能力横向对照

> 本篇不是 MySQL 的实现剖析，而是一篇**横向对照**——讲清 MySQL 优化器在"计划稳定性、GROUP BY、下推"三个维度上**有什么、缺什么**，以及业界（Oracle / HyPer / Presto）怎么做。这三个维度是 MySQL OLAP 能力偏弱的集中体现；理解"缺什么"比理解"有什么"更能看清它的定位。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、计划回归与 SPM：计划稳定性的两种思路](#一计划回归与-spm计划稳定性的两种思路)
- [二、GROUP BY 优化的三条路线](#二group-by-优化的三条路线)
- [三、下推优化的横向对照](#三下推优化的横向对照)
- [参考](#参考)

---

## 设计思想与理论基础

### 一、计划回归与 SPM：计划稳定性的两种思路

#### 1.1 计划回归（plan regression）

CBO 依赖**统计信息**做决策。当统计刷新、数据量变化、版本升级、参数调整时，优化器可能为同一条 SQL **突然换一个计划**——而新计划**不一定更好**，甚至更差（代价模型失真、统计偏差）。线上一条跑得好好的 SQL 某天突然变慢，就是**计划回归**。

#### 1.2 SPM：SQL Plan Management（Oracle）

SPM（Oracle 11g 引入）是一种**预防性机制**，核心是 **SQL plan baseline（计划基线）**：与一条 SQL 关联的一组**可接受计划**。

关键点：**计划不再由优化器自由生成，而是受基线约束**——

- 优化器只从 baseline 里**选择**，不自由探索
- 新计划要先**验证**，验证通过才能进 baseline

工作流程三段：

| 阶段 | 动作 | 关键点 |
|---|---|---|
| **捕获（capture）** | 把 SQL 的计划纳入 baseline | 自动（`optimizer_capture_sql_plan_baselines`）或从 AWR/游标缓存手动加载 |
| **选择（select）** | 优化器从 baseline 里挑代价最小的 | 只挑 `enabled + accepted` 的计划 |
| **进化（evolve）** | 新计划先"试用" | 验证性能更优才 `accepted`，否则丢弃 |

一个计划的三个状态：`ENABLED`（可考虑）/ `ACCEPTED`（验证过，正式成员）/ `FIXED`（锁死，即使有更优也不换）。

**SPM 与 hints 的本质区别**：

> **hint 是"锁死一个计划"，SPM 是"管理一个计划集合并让它受控进化"。**

hint 静态、一次性、DBA 人肉维护；SPM 自动、持续、允许计划随数据缓慢演进——但演进是**受控**的（先验证再接受）。

#### 1.3 为什么 MySQL 和 Presto 都没有

SPM 需要一整套"计划身份（signature）+ 基线存储 + 进化验证"的基础设施。

| 系统 | 计划管理手段 | 本质 |
|---|---|---|
| **MySQL** | `optimizer hints` + `optimizer_switch` | 手动逃生舱 |
| **Presto/Trino** | session 属性（`join_distribution_type`、`join_reordering_strategy`） | 会话级手动控制 |
| **Oracle** | SPM（plan baseline） | 自动、受控进化 |

MySQL 与 Presto 都停在了"手动"这一档。分布式引擎（Presto）环境更多变、计划更难基线化，普遍不做 SPM。MySQL 则是从未投入这个方向——它选择了更轻的 hints + switch 作为手动逃生舱。

#### 1.4 云厂商的 SPM：谁在 MySQL 生态里补上了

社区 MySQL 没有 SPM，但云厂商在各自产品里做了。调研结论：

| 产品 | 计划管理 | 说明 |
|---|---|---|
| 社区 MySQL | ❌ 只有 hints + `optimizer_switch` | — |
| AliSQL（阿里云 MySQL 内核） | ❌ 未见独立 SPM | 增强在别处（线程池 / 审计等） |
| Aurora **MySQL** 版 | ❌ 只有 optimizer hints | 与社区一致 |
| **Aurora PostgreSQL** 版 | ✅ `apg_plan_mgmt` 扩展 | 真实 SPM：`capture_plan_baselines` / `Evolve_Plan_Baselines` / `Validate_Plans` |
| **PolarDB MySQL 版 / PolarDB-X** | ✅ 「执行计划管理」 | 阿里云自研，`BASELINE` 指令 + baseline |
| Oracle | ✅ 原生 SPM | 源头 |

**两个值得注意的点**：

1. **Aurora 只在 PG 版做了 SPM，MySQL 版没做**。`apg_plan_mgmt` 仅存在于 Aurora PostgreSQL 版，Aurora MySQL 版仍是 hints。侧面说明"给 MySQL 加 SPM"更难——MySQL 优化器没有 PG 那种清晰的 plan node 序列化机制，计划的"身份"不好锚定。

2. **PolarDB-X 的 SPM 直接借鉴 Oracle，但做了云化适配**：

| 维度 | PolarDB-X SPM | Oracle SPM |
|---|---|---|
| 接口 | `BASELINE ADD/FIX/LOAD/PERSIST/...` 命令 | `DBMS_SPM` 包 |
| 计划键 | **参数化 SQL 文本**（`?` 占位符） | SQL signature |
| 状态 | `FIXED` / `ACCEPTED` | `FIXED` / `ACCEPTED` / `ENABLED` |
| 前置 | 先过 **Plan Cache**，复杂查询才进 SPM | 共享池与 baseline 分离 |
| 分布式 | 计划含 `Gather` / `ParallelHashJoin` / `LogicalView` 算子 | 无 |

核心机制同构：**参数化 → baseline → 多计划选代价最小 → `FIXED` 固定 → 自动演化**。PolarDB-X 的独到之处在于把 **Plan Cache**（参数化 SQL 的计划复用）与 **Plan Management**（基线固化）绑在一起——这比 Oracle 更贴合云上"高并发 + 计划稳定"的需求。

### 二、GROUP BY 优化的三条路线

`JOIN + GROUP BY` 的朴素执行是**先 join 出全部中间结果，再 group by**。当 join 产生海量中间行、而聚合后结果很小（如"每个分组取前几名"）时，中间结果膨胀成瓶颈。三种系统给出了三种解法：

| | GROUP BY 优化思路 | 系统定位 |
|---|---|---|
| **MySQL** | 单库内部：loose index scan + 临时表/filesort | 单机 OLTP |
| **HyPer** | **groupjoin** 融合算子（grouping 融合进 join） | 内存 HTAP |
| **Presto/Trino** | **aggregation pushdown**（下推数据源）+ partial/final 分布式聚合 | 分布式联邦查询 |

**HyPer 的 groupjoin**（论文《Accelerating Queries with Group-By and Join By Groupjoin》）：把 grouping **融合进 join**，在 join 过程中就做聚合，避免物化完整中间结果。前提是分组键与 join 键有特定关系、且 join 不会"复制"分组。本质是 **group-by pushdown** 的激进形式。

**Presto 的 aggregation pushdown**：把整个 `GROUP BY` 下推到数据源执行（EXPLAIN 里不出现 `Aggregate` 算子）；下推失败则退化为两阶段分布式聚合：

```
Aggregate(PARTIAL)   -- 各节点局部聚合
      ↓ shuffle
Aggregate(FINAL)     -- 全局聚合
```

复杂 grouping（`ROLLUP` / `CUBE` / `GROUPING SETS`）不下推，本地执行。

**三种思路没有谁对谁错，而是被系统定位逼出来的**：

- MySQL 是单机，只能在内部优化，所以走索引/临时表
- HyPer 是内存库、追求极致，所以敢做 groupjoin 这种激进融合
- Presto 是联邦引擎，数据在别人那里，所以第一反应是"下推"，下推不了才自己分布式算

**MySQL 的代价**：它的 GROUP BY 一定在 join **之后**做（先 join 再 `AggregateIterator` 聚合），唯一的 GROUP BY 优化是**单表**的 loose index scan（见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)）。所以面对 `JOIN + GROUP BY` 时中间结果膨胀无法避免。

### 三、下推优化的横向对照

#### 3.1 MySQL 的下推清单

MySQL 是单机，它的"下推"不是推到别的数据源，而是**从 SQL 层推到自己的存储引擎层（handler）**。全景如下：

| 下推优化 | 作用 | 系统讲解 |
|---|---|---|
| **ICP**（Index Condition Pushdown） | 把 WHERE 条件下推到引擎，在索引遍历时就过滤（减少回表） | [`10_plan_refinement.md`](10_plan_refinement.md)、[`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) |
| **MRR**（Multi-Range Read） | 批量化随机 IO（rowid 排序后顺序读） | [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)、[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **derived_condition_pushdown** | 把外层条件推进派生表（物化前过滤） | [`logical/04_logical_join.md`](logical/04_logical_join.md)、[`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) |
| **index_merge** | 多索引取交集/并集 | [`physical/07_access_method.md`](physical/07_access_method.md)、[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **skip scan**（8.0 新增） | 联合索引跳首列扫描 | [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **BKA**（Batched Key Access） | 批量 key 做 MRR 索引查找 | [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md) |
| **push_to_engines** | 把 join/条件整体下推给引擎（如 NDB 集群） | [`10_plan_refinement.md`](10_plan_refinement.md) |

#### 3.2 Presto/Trino 的 pushdown 体系

Presto 的"下推"是**把算子推到别的数据源**（MySQL、PG、Hive 等），方向完全不同：

| pushdown 类型 | 作用 |
|---|---|
| **Predicate** | 下推 WHERE 过滤 |
| **Projection** | 只读需要的列 |
| **Dereference** | 只读 ROW 类型的指定字段 |
| **Aggregation** | 下推 GROUP BY / 聚合 |
| **Join** | 把多表 join 委托给数据源 |
| **Limit / Top-N** | 下推 LIMIT / ORDER BY LIMIT |

成功下推时，EXPLAIN 里对应的算子（`Filter` / `Aggregate` / `Join` / `TopN`）**消失**，只剩 `TableScan`。

#### 3.3 差异的本质

| | MySQL | Presto |
|---|---|---|
| 下推方向 | SQL 层 → **自己的存储引擎层**（handler） | 引擎 → **别的数据源**（connector） |
| 收益 | 减少回表、减少随机 IO | 减少网络传输、借力底层数据库的优化 |
| 粒度 | 条件、key 批处理、join | 条件、投影、聚合、join、limit 全套 |

> MySQL 的"下推"是单机内部的纵深优化；Presto 的"下推"是联邦架构的水平借力。两者名字都叫 pushdown，解决的问题和方向完全不同。

---

## 四、国产内核优化器对照

> 这一节把视线从 MySQL 本体转向**国产数据库内核**。写它的动机：对内核研发这个定位，"社区版 MySQL 缺了什么、国产内核补了什么"是最直接的横向参照——尤其是面试与实际工作中经常被问到。★ 特性细节以各产品官方文档为准，这里只列**可确认的、与社区 MySQL 的差异**，不展开未经证实的细节。

| 内核 | 底层 | 优化器/执行器相对社区 MySQL 的差异 |
|---|---|---|
| **PolarDB MySQL** | MySQL 8.0 | ★ **分布式并行查询（Parallel Query，PQ）**：单条 SQL 的扫描/join 并行化，利用多核提升 CPU/IO 利用率。这是社区 MySQL 没有的——社区版只有 InnoDB 并行读与 DDL 并行，**没有查询级并行** |
| **OceanBase** | 自研 | 优化器 = 查询改写 + **动态规划（DP）+ Skyline 剪枝**；支持**并行执行（DOP）**。对比 MySQL 的 greedy 搜索，OB 用更完整的 DP 框架 + 帕累托剪枝 |
| **TiDB** | 自研（Go） | ★ **Cascades 优化器**（4.0+ 引入）：规则 + 代价搜索的 Cascades 框架。这正是 MySQL 经典优化器**没有**采用、hypergraph 只是部分借鉴的框架（见 [`17_optimizer_decisions.md`](17_optimizer_decisions.md) 理论基础） |
| **openGauss** | PostgreSQL | 基于 PG 内核，优化器在 PG 基础上增强（列存、向量化）。与 MySQL 家族差异最大，属于 PG 血统 |

### 对"内核研发"的三点启示

1. **"并行执行"是国产内核的普遍补强点**：PolarDB、OB 都做了查询级并行，而社区 MySQL 至今没有。这说明"并行查询"是从社区 MySQL 出发做内核研发时，最被反复触碰的空白之一（可对照本目录各篇「社区边界澄清」里反复出现的"社区版没有 X"）。

2. **优化器框架的分野**：MySQL（greedy / hypergraph）vs TiDB（Cascades）vs OB（DP + Skyline）。理解这四种框架，比死记某个内核的 API 更有价值——它们对应的是[`17_optimizer_decisions.md`](17_optimizer_decisions.md) 里"规则 vs 代价"这条主轴的不同取舍。

3. **社区边界澄清的价值**：本目录每一篇末尾的「社区边界澄清」之所以重要，正是因为**国产内核/云厂商的能力常常被误当成社区 MySQL 的能力**（如 HeatWave、并行查询、向量化执行）。读任何一篇前，先确认"这是社区版行为，还是某个内核/云的能力"。

---

## 参考

**理论 / 论文**
- *SQL Plan Management*（Oracle 11g，plan baseline 机制）
- Gubner, Leis, et al. *Accelerating Queries with Group-By and Join By Groupjoin*（HyPer 的 groupjoin 算子）
- *Efficient generation of query plans containing group-by, join...*（grouping 与 join 的代数重排序）

**官方文档**
- Trino 官方 *Pushdown*（predicate / projection / aggregation / join / limit pushdown）

**相关文档**
- MySQL 计划管理替代方案：[`11_optimizer_hints.md`](11_optimizer_hints.md)
- GROUP BY 单表优化（loose index scan）：[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
- 各下推优化：见「三、下推优化的横向对照」表内链接
