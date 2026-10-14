# index/ —— 索引（横切主题）

> 本目录是 **"索引"主题**的横切目录，与 [`../infra/`](../infra/) 平级——索引横跨 SQL 层（语法/优化器）、handler 接口、存储引擎（B-tree 存储与维护）三层，任何一层都不是它的全部。
>
> **归属判据**：以"索引"为**第一主语**的篇进本目录；索引只是其一部分的特性篇留在原处（全文检索 [`../feat/fts.md`](../feat/fts.md)、空间 [`../server/datatype/gis.md`](../server/datatype/gis.md)、功能/多值索引 [`../feat/generated_columns.md`](../feat/generated_columns.md)），由下面的分类导航聚合。
>
> **为什么不拆子目录**：本目录现有 **17 篇**，用下面的「九个分类 + 四条纵向底座」导航表组织。按知识库规则"同一阶段 ≥3~4 篇才建子目录"，② 存储结构（5 篇）与纵向底座（4 篇）**已过阈值**；之所以仍不物理拆分，是因为这些篇之间交叉引用极密（如 `secondary.md` 同时属于②存储结构与生命周期、`btr.md` 承载⑧锁与并发），拆成子目录会把"一篇横跨多类"的关系切碎。**当前的取舍是：用导航表表达分类，用实体文件保持交叉自由**——若后续篇数继续增长且出现真正的同质子群（如"辅助结构"攒到 4 篇以上），再考虑建子目录。

## 目录

- [索引的九个分类](#索引的九个分类)
- [分类导航表](#分类导航表)
- [三种读者的阅读路径](#三种读者的阅读路径)
- [本目录的篇](#本目录的篇)
- [待补清单](#待补清单)

---

## 索引的九个分类

九个分类按"索引从被创建到被使用的完整生命周期"排列，每类的**边界**是区分的关键：

| # | 分类 | 讲什么 | 边界（与相邻类的区分） |
|---|------|--------|----------------------|
| ① | **类型** | 用户可见的索引语义：主键/二级/唯一/降序/前缀/功能/多值/全文/空间/不可见 | 类型是**语义**（DDL 里写的那个东西），不是实现；同一种类型在不同引擎实现可以不同 |
| ② | **存储结构** | 索引的**数据结构实现**：B-tree、R-tree、倒排、Hash | 存储是**实现**（物理），类型是**语义**（用户可见）；一个类型绑定一种存储（全文→倒排、空间→R-tree、其余→B-tree） |
| ③ | **访问优化** | 执行期怎么用索引取数据：回表、覆盖索引（index-only）、ICP 下推、MRR、Skip Scan | 访问优化是**执行期行为**（优化器可见、体现在执行计划里），引擎内加速是**透明加速结构**（对 SQL 层不可见） |
| ④ | **引擎内加速** | 引擎内的透明加速：自适应哈希索引、change buffer | 对 SQL 层不可见，不在执行计划里体现；是"索引的索引/缓冲" |
| ⑤ | **选择** | 优化器怎么选索引：cost model、range optimizer、index merge、skip scan、统计基数 | 归优化器层（`server/query/07_optimize/`），此处只导航不做详写 |
| ⑥ | **约束语义** | 索引承载的约束：PRIMARY/UNIQUE 的实体完整性、唯一性检查链路、外键对索引的要求 | 讲"索引为什么必须存在"的规范性理由，而非性能理由 |
| ⑦ | **生命周期 / DDL** | 索引的创建（批量构建 vs 逐条）、online DDL、删除/禁用、ANALYZE 刷新统计 | 索引的"生老病死"，与操作实现（②）相对 |
| ⑧ | **锁与并发** | 索引作为锁的载体：索引树锁（`dict_index_t::lock`，SX 协议）、gap/next-key 依附索引记录、二级索引 purge 回表判定 | **树锁详写在 [`btr.md`](btr.md)**（它是 B-tree 结构操作的伴生机制）；事务锁归锁目录 + MVCC，此处仅导航 |
| ⑨ | **可观测 / 运维** | `SHOW INDEX`、`I_S.STATISTICS`、基数刷新机制、冗余/无用索引检测、索引碎片与重建 | 面向使用者，不是面向实现 |

> **最容易混的两对**：①类型 vs ②存储（语义 vs 实现）；③访问方式 vs ④优化机制（执行期可见 vs 引擎内透明）。

## 分类导航表

> **四条纵向底座**（横贯九分类，不属于任何一格）：
> ① [`metadata.md`](metadata.md)——索引的**元数据对象模型**（`dd::Index` ↔ `dict_index_t` ↔ SDI ↔ `se_private_data` 五键，root page no 是两套字典的桥）；
> ② [`construction.md`](construction.md)——**索引怎么被构造出来**（`KEY`/`KEY_PART_INFO` 与三条构造链路）；
> ③ [`record_format.md`](record_format.md)——**索引记录的存储格式**（三种记录字段组成、trx_id_offset 快速通道、BLOB 前缀）；
> ④ [`physical_storage.md`](physical_storage.md)——**索引的物理存储与持久化**（根页恒定/两段式分配/索引页布局/MLOG redo 体系/bulk load 豁免与 DDL log/空间回收与恢复）。
>
> 类型语义 → 构造 → 对象模型 → 存储结构 → 记录格式 → 物理与持久化，这条链由这四篇补上底座。

| 分类 | 本目录实体篇 | 导航到（一处详写，不复制） |
|------|------------|--------------------------|
| ① 类型 | [types.md](types.md) | 功能/多值/GIPK → [`../feat/generated_columns.md`](../feat/generated_columns.md)；全文类型 → [`../feat/fts.md`](../feat/fts.md)；空间类型 → [`../server/datatype/gis.md`](../server/datatype/gis.md)；**GIPK 专篇 → [`gipk.md`](gipk.md)** |
| ② 存储结构 | [btr.md](btr.md)（B-tree 全套）、[rtree.md](rtree.md)（R-tree 全套）、[inverted.md](inverted.md)（倒排编码） | 空间类型/SRID 语义 → [`../server/datatype/gis.md`](../server/datatype/gis.md)；全文全链路 → [`../feat/fts.md`](../feat/fts.md)；页与记录格式 → [`../innodb/physical/page_structure.md`](../innodb/physical/page_structure.md)、[`../innodb/physical/record.md`](../innodb/physical/record.md) |
| ③ 访问优化 | [access.md](access.md) | range 与 access path → [`../server/query/07_optimize/`](../server/query/07_optimize/)；**ICP 的典型消费者（LIKE 的三种执行位置）** → [`../server/query/15_like.md`](../server/query/15_like.md) §七、§九 |
| ④ 引擎内加速 | [ahi.md](ahi.md)、[ibuf.md](ibuf.md) | — |
| ⑤ 选择与统计 | [stats.md](stats.md) | 优化器怎么用统计：cost model / range optimizer / index merge → [`../server/query/07_optimize/`](../server/query/07_optimize/)；**索引 hints（INDEX/`USE INDEX`/`FORCE INDEX`）→ [`../server/query/07_optimize/11_optimizer_hints.md`](../server/query/07_optimize/11_optimizer_hints.md)**（四层 hint 对象树 / `update_index_hint_map` 集合运算 / range-ref 读取） |
| ⑥ 约束语义 | [constraint.md](constraint.md) | 行锁与隐式锁转换 → [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md) |
| ⑦ 生命周期 | [operations.md](operations.md) | online DDL 通用框架（row log / DDL log）→ [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md) |
| ⑧ 锁与并发 | **本目录不设独立篇**（索引锁归 btr） | **索引树锁**（`dict_index_t::lock`，引擎内部 latch：S/SX/X + intention 解析 + 叶子三兄弟 + SMO 预测裁剪）→ [`btr.md`](btr.md)「索引锁」一节；**索引记录上的事务锁**（record/gap/next-key、隐式锁转换、谓词锁）→ [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)；二级索引可见性 → [`../innodb/mvcc.md`](../innodb/mvcc.md) |
| ⑨ 可观测与运维 | [operations.md](operations.md) | Monitor 与 metrics → [`../innodb/monitor.md`](../infra/observability/innodb_monitor.md) |

## 三种读者的阅读路径

- **想用索引的人**（DBA/开发者）：① 类型 → ⑨ 可观测（基数为什么漂、冗余索引怎么查）→ ⑦ ANALYZE/重建。
- **想调索引的人**（性能优化）：⑤ 选择 → ③ 访问方式（回表 vs 覆盖、ICP 是否生效）→ ④（AHI/change buffer 的收益与代价）→ ② 存储（分裂/合并对写入的代价）。
- **想实现索引的人**（内核开发者）：② 为主线（[btr.md](btr.md) 是 btr 模块全套：搜索逐行/页内二分/插入分裂/删除合并/索引锁/node pointer/批量构建/树生命周期）→ 向上接 handler 接口 → 向下接 [页](../innodb/physical/page_structure.md) 与 [记录格式](../innodb/physical/record.md)。

## 本目录的篇

| 文件 | 分类 | 内容 |
|------|------|------|
| [btr.md](btr.md) | ② 存储结构 | B-tree 结构与操作（btr 模块全套，含压缩页 page_zip 双份结构专节） |
| [types.md](types.md) | ① 类型 | 索引类型总览：语义 vs 实现、全谱系 |
| [gipk.md](gipk.md) | ① 类型 | GIPK 专篇：隐藏自增主键的生成、可见性、DD 打标、binlog 重写与复制 |
| [access.md](access.md) | ③ 访问优化 | 回表与三类访问优化（覆盖索引/ICP/MRR/Skip Scan） |
| [constraint.md](constraint.md) | ⑥ 约束语义 | 索引承载的约束：唯一性、外键 |
| [stats.md](stats.md) | ⑤ 统计 | 索引统计：分层采样、rec_per_key、基数漂移根因 |
| [rtree.md](rtree.md) | ② 存储结构 | R-tree 索引结构全套 |
| [inverted.md](inverted.md) | ② 存储结构 | 倒排索引存储与编码 |
| [operations.md](operations.md) | ⑦ 生命周期 + ⑨ 可观测 | 索引的生老病死与观测 |
| [metadata.md](metadata.md) | 纵向底座 | 索引元数据对象模型（dd::Index ↔ dict_index_t ↔ SDI） |
| [physical_storage.md](physical_storage.md) | 纵向底座 | 索引物理存储与持久化 |
| [transaction.md](transaction.md) | 交叉线 | 索引 × MVCC/purge/唯一性 |
| [construction.md](construction.md) | 纵向底座 | KEY 类与索引构造 |
| [record_format.md](record_format.md) | 纵向底座 | 索引记录存储格式 |
| [partitioning.md](partitioning.md) | 交叉线 | 索引 × 分区 |
| [secondary.md](secondary.md) | ② 存储结构整合 | 二级索引整合专篇 |
| [ahi.md](ahi.md) | ④ 引擎内加速 | 自适应哈希索引 |
| [ibuf.md](ibuf.md) | ④ 引擎内加速 | change buffer（insert buffer） |

## 待补清单

九个分类全部覆盖：①②③④⑤⑥⑦⑨ 有本目录实体篇（待补清单三项——各类型深入机制 / 空间与全文独立篇 / 统计更细机制——已分别落入 [`rtree.md`](rtree.md)、[`inverted.md`](inverted.md)、[`types.md`](types.md) 多值交互节、[`stats.md`](stats.md) 增补节）；**⑧ 锁与并发刻意不独立成篇**——索引树锁是 B-tree 结构操作的伴生机制（保护树高度/分裂/合并，SMO 预测裁剪属于 btr 搜索路径的一部分），脱离 btr 讲会与 [`btr.md`](btr.md) 大量重复，故留在 btr.md「索引锁」一节；索引记录上的事务锁归锁目录。

**候选后续方向**（非待补，按需开工）：全文打分（`tf × idf²`）与倒排编码的衔接细节（现指向 [`../feat/fts.md`](../feat/fts.md)）。
