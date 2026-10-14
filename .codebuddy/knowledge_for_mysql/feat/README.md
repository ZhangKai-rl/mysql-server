# feat/ —— 跨层主题（server ↔ InnoDB）深度剖析

> 一句话定位：**跨层主题的归位目录**——凡是**同时需要 server 与 InnoDB 两边源码**才能讲清的主题，放这里**一篇写完两层**，并且**深度剖析**（写机制怎么实现，**不是**编排型摘要）。
>
> ★ 判据与目的：**避免 `server/` 与 `innodb/` 各写一篇造成的割裂**。读者带着"SDI 怎么存""分区表怎么实现"这类问题进来，一次读完闭环，不用自己在两层之间来回跳。

## 与按层目录的分工（最重要的一节）

本库有两类文档，**不要互相复制内容**：

| 类型 | 位置 | 内容边界 |
|---|---|---|
| **单层机制** | `server/`、`innodb/` 等按层目录 | **只涉及一层**的机制在这里写透（如 `row_search_mvcc`、MDL、buffer pool）。**全库唯一出处**，改一处即改全库 |
| **跨层主题** | `feat/` | ★ 同时涉及 server 与引擎的主题，在这里**深度剖析**：原理、设计、源码剖析都要够，**写机制怎么实现** |

**为什么不拆成两篇**：跨层的**因果接缝**——谁调用谁、数据怎么在两层间流转、权限/锁/事务责任谁承担——拆成两篇时会直接丢失，读者要自己拼装（SDI 就是实例：server 放弃 I_S 逐表权限建模 ⇒ 引擎被迫手写 21 处 `PROCESS`，这条因果只有在一篇里才看得见）。

**"不要互相复制"仍然成立，但说的不是这里**：`row_search_mvcc` 被 SELECT/UPDATE/DELETE 共用、MDL 被所有 DML/DDL 共用、buffer pool 到处都是——这类**被多个主题共用的单层通用机制**留在按层目录写透，`feat/` 用一句 + 指针引用，**不在本篇复述**。而**跨层主题自身**就属于 `feat/`，在这里写透。

**写法口诀**：跨层主题在 `feat/` 写"这个机制在两层分别怎么实现、接缝在哪"——而不是"每层只写本层做了什么选择"。

## ★ 深度标准：全库统一，只是位置不同

> **一句话**：整个知识库的要求都是一样的，只是放的位置不同，都要求深度广度足够。

`feat/`、`infra/`、`server/`、`innodb/` **同样**要求深度广度足够，**不存在"某个目录可以写得薄"**。视角统一为**内核研发视角**：

- **核心函数深度剖析**——贴关键源码/注释原文，讲清每个分支为什么这么写
- **设计思想**——为什么这么设计、否掉了哪些方案（源码注释里的方案权衡是最佳材料）
- **高级语法讲清楚**——模板、RAII、tag dispatch、反向 pimpl、CRTP、`constexpr` 等，要讲清"**为什么**用这个技法"而不只是"用了"

（与"篇幅基线是诊断信号、不是硬指标"**不冲突**：那条**防注水**，本条要求**该深的必须深**。自查方法：**源码 KB 数 vs 文档行数不成比例** = 深度不足的信号。）

## 篇内骨架：两种形态，按耦合方式选

### 形态 A：按层分节（两层职责清晰可分时）

```markdown
# XXX 深度解析

## 这条操作经过了什么          ← 先给全景调用栈，让读者有骨架
## server 层：...             ← SQL 层的**完整实现**（核心函数剖析 + 设计思想 + 高级语法）
## 接口层：...                ← handler/handlerton 上的关键调用与语义（★ 跨层接缝）
## innodb 层：...             ← 引擎层的**完整实现**（同上）
## 两层的结合点               ← ★ 本篇价值所在：3 个关键接口、数据如何在两层间流转
## 被否掉的方案 / 已知限制    ← 源码注释里的权衡，最有价值的部分
## 常见误解 / 失效场景
## 参考
```

适用：SELECT（解析优化 vs 扫描取行）、DDL（SQL 层语义 vs 引擎重建）、事务提交（server 协调 vs 引擎 prepare/commit）。

### 形态 B：按生命周期（两层动作交织时）

```markdown
# XXX 深度解析

## 建表 / 创建     语法 → 元数据 → DD → 引擎
## 打开 / 装载     ...
## 运行期         ...（每个环节内部都闭环：SQL 层做什么 → 引擎层做什么）
## 销毁 / 变更     ...
```

适用：分区表（算分区号 → 换 `dict_table_t` → 写行，一环扣一环），这类强行按层分节会把一次写入切成两半。

**判据**：两层的动作是"接力"还是"交织"？接力 → 形态 A；交织 → 形态 B。

## 三条硬规则

1. **不为每个引擎单独开篇**。多个 hton 实现同一特性时，在一篇里用"引擎差异"小节对比（例：分区表只有 InnoDB 与 NDB 两家，NDB 走 `HA_USE_AUTO_PARTITION` 自动分区并保留 `nodegroup_id`，未声明 native 分区能力的引擎直接被拒）。否则 N 个引擎 = N 篇，且每篇都要复述一遍 server 层。
2. **机制细节用指针，不用复制**。写"走 `row_search_mvcc` 取行（实现见 [`../innodb/row_search.md`](../innodb/row_search.md)）"，不要再把 `row_search_mvcc` 讲一遍。
3. **特性篇必须闭环**。上游谁调它、下游它调谁、边界在哪，都要交代——哪怕只是一句交叉引用。

## 与 `server/query/` 主链的关系

`server/query/` 是**一条 SQL 的逐步展开**（01 协议 → 11 EXPLAIN），本身已经是"SELECT 的旅程"。

因此 `feat/select.md`（若写）**不应是主链的重写**，而应是：

- 一页纸路线图：这条操作依次经过哪些阶段
- 各层要点 + 指针：每层看主链的哪一篇、哪一节
- 跨层拼接：两层之间的关键接口与数据流转（这是主链逐步展开时不易看出的东西）

## 现有篇目

| 文件 | 形态 | 内容 |
|---|---|---|
| [partitioning.md](partitioning.md) | **功能域大类**（一~八） | 分区表全链路：元数据与 DD 往返 / 运行期路由与裁剪 / 分区级 DDL / 原子性与崩溃恢复 / 统计与代价估算 / 事务锁与死锁 / 复制与并行 / 能力边界 |

> **形态补充**：上面 A/B 是按"两层的耦合方式"分的。`partitioning.md` 走的是第三种——**按功能域分大类，大类之间按生命周期推进**：两层动作交织（算分区号 → 换 `dict_table_t` → 写行）所以类内不适合按层切，而类间又天然是"建表 → 打开 → 运行 → 变更 → 统计 → 事务 → 复制"的顺序。判据一句话：**两层接力 → A；两层交织 → B；交织但主题多、需要分段导航 → 功能域大类**。
| [fts.md](fts.md) | A（按层分节） | 全文检索全链路：语法 → 优化器 → 辅助表与编码 → 打分 |
| [ddl/ddl.md](ddl/ddl.md) | **融合（按主题分章）** | DDL **主篇 = 两层公共骨架**：三种算法的选择策略与 **MDL 三段式** + ★算法判定真相（`mysql_compare_tables` 不参与、`IS_EQUAL_*` 只有三档）；DROP/TRUNCATE/RENAME（+ InnoDB 实现、btr_free）；速查 / binlog / 数据结构 / 调用栈 / 观测。★ CREATE / ALTER 已独立成篇（见下两行）。原 `innodb/ddl.md` 1404 行**未删减**已按主题嵌入 |
| [ddl/create_table.md](ddl/create_table.md) | 跨层（一篇写完两层） | **CREATE TABLE 全链路**：server 侧编排（`mysql_prepare_create_table` 全部合法性检查 / `rea_create_base_table` DD 写入 / `ha_create_table` 接缝）+ **引擎侧落地**（`create_table_info_t` 三段式 / `dict_build_tablespace_for_table` 建 .ibd / `btr_create` 建根页 / 统计置空）；★ **不属于任何算法但被三种算法复用**（COPY 建中间表 / INPLACE rebuild 建新表 / TRUNCATE 重建空表）；★ 为什么先写 DD 再调引擎、为什么原子 DDL 下 CREATE 也要写回滚预留 log；★ 新表统计必为 0 行 |
| [ddl/alter_table.md](ddl/alter_table.md) | 跨层（三种算法共用的**公共前段**） | **ALTER TABLE 编排主线**：`Alter_info` 意图描述 / `Alter_table_ctx` 执行期上下文 / `mysql_prepare_alter_table`（意图 → 动作）、★ **算法决策与 fail-fast**（三条件判定 + `is_inplace_alter_impossible` 一票否决，临时表一律 COPY）、INPLACE vs COPY 的 server 侧差异、`mysql_recreate_table`、`notify_alter_table`；★ **`mysql_alter_table` 剩余四段全剖析**——外键 MDL 大集合（为什么只改一张表却锁一堆表、ADD 的 FK 名为何延迟锁）/ prepare 之后新表定义落地（强制退 COPY 三条件、`#sql-` 中间表 MDL X、`non_dd_table_def` 攥内存不落 DD）/`mysql_inplace_alter_table` 18 步全流程（两次升 X、原子 DDL 在 DD 替换处分岔）/ 收尾链（备份名 `#sql-2-*` + **`quick_rm_table` 与恢复备份名重试 20 次**的 gap lock 死锁出处）；三种算法与并行构建见各自分篇 |
| [ddl/copy_ddl.md](ddl/copy_ddl.md) | 跨层（一篇写完两层） | **COPY 算法**：server 侧编排 + `copy_data_between_tables` 逐段 + 逐行经 `ha_write_row` 走完整写入路径 + **为何一定不 Online**（全程 MDL X）；★ **ALTER 语义完整清单**（截断告警 / 自增 / 生成列逐行求值 / CHECK 约束 / 重复键报源表名）——INPLACE/INSTANT 的语义正确性参照系；与 INPLACE rebuild 的代码级代价对比 |
| [ddl/inplace_ddl.md](ddl/inplace_ddl.md) | 跨层（一篇写完两层） | **INPLACE 算法**：server 侧 MDL 升降级与四件套契约、引擎 `check_if_supported_inplace_alter` 六档返回值如何决定锁、★ **row log**（为什么 apply 两次、第二个短暂阻塞点）；★ WL#5526 奠基设计（row log 七步协议 / **索引在线状态机与僵尸索引** / `TEMP_INDEX_PREFIX` 崩溃安全）；可观测性（3 个 ddl 计数器 + DEBUG_SYNC apply 点） |
| [ddl/instant_ddl.md](ddl/instant_ddl.md) | 跨层（一篇写完两层） | **INSTANT 算法**：只改元数据 + 行版本号、★ 表级/行级/物理**三个 "version" 的辨析**、★ **V1（n_fields）→ V2（row_version）两种记录格式的字节级布局**（COMPACT/DYNAMIC/REDUNDANT/TEMP 四种）、加列与删列差异、WL#11250 与两个坑（回滚行变大 / IMPORT 依赖 .cfg）；可观测性（`TOTAL_ROW_VERSIONS`） |
| [ddl/ddl_engine.md](ddl/ddl_engine.md) | 引擎内部执行引擎 | **8.0 新版 DDL 引擎与并行构建**（`storage/innobase/ddl/` 200 KB）：版本演进（`row0merge.cc` 消失）、Context/Loader/Builder/Btree_load、★★ 新旧对比表、★ 扫描—排序—建树三期算法（**为什么建树仍串行**）、★ 任务队列的同步原语与终止判定（quiescence 终止检测、防丢唤醒、退出连锁、锁外执行）、`Parallel_reader` 动态分片；可观测性（★ `thread/innodb/ib_ddl` 在 PROCESSLIST 看不到）。**并行 DDL 是这套引擎的并发维度，并入本篇不另立** |
| [ddl/atomic_ddl.md](ddl/atomic_ddl.md) | 跨层（横切属性，一篇写完两层） | **★★ 原子 DDL**：server 编排与崩溃安全五条机制 + 引擎 `dd::Table` 条件持久化 + **★ DDL log 的两个相反方向**（CREATE 类 **写后即删**用于回滚 / DROP 类提交后 replay 用于前滚）+ 幂等性 + 四阶段与各操作写入时机表；失败与崩溃恢复路径。可观测性（`innodb_print_ddl_logs`）。所有 DDL 类型共用此保障 |
| [ddl/online_ddl_tools.md](ddl/online_ddl_tools.md) | 外部知识（事实来源＝工具源码，非本仓库） | gh-ost / pt-osc 在线 DDL 工具：存量拷贝与 chunk 分片、增量同步（binlog 订阅 vs 触发器）、cut-over magic lock、cut-over 前强制 ANALYZE、限流与故障。与 `ddl/ddl.md` 互指（原生算法在那边） |
| [sdi.md](sdi.md) | A（按层分节） | SDI 全链路：server 定紧凑 JSON 格式 → handlerton `sdi_*` 回调按引擎能力分派 → InnoDB 用 `SDI_<space_id>` B-tree 存储 → 结合点（`ibd2sdi` 物理读 / `IMPORT ... FROM .sdi` 逻辑重建）。★ 服务器**只写不读** |
| [check_table.md](check_table.md) | A（按层分节） | **CHECK TABLE 全链路**：ADMIN 语句家族（`mysql_admin_table` 一框架四兄弟 + 成员函数指针分派）、check_opt 两套位图（flags/sql_flags）与四档语义、handler 契约与 `HA_ADMIN_*` 全清单、★ InnoDB `ha_innobase::check` 三段流程完整贴码（RR 强制提升/dummy template/结构校验失败不落标 vs 行级失败才 `dict_set_corrupted` 的不对称）、行级扫描器（相邻判序/重复键/含 NULL 放行/并行分片重置）、Msg_type 聚合、REPAIR 边界（InnoDB 无 repair、TRY_ALTER=ALTER 重建的源码依据）、FOR UPGRADE、锁语义、6 条误解 |
| [compression.md](compression.md) | 全景地图（跨机制合集） | **压缩全景**：9 个机制共用 3 算法库的总表（page_zip / TPC / ZBLOB / FTS 单词流 / binlog zstd / 协议压缩 / COMPRESS() / clone 两套）+ 算法库接入清单（zlib-1.2.13/zstd-1.5.5/lz4-1.9.4 调用点）+ ★ **不存在清单**（字段/记录/redo/undo 压缩等 Percona-MariaDB 特性三重证据）+ 6 条常见误解；深度剖析落在各归属文档（btr/fil/lob/fts/binlog/authentication/clone） |
| [auto_increment.md](auto_increment.md) | A（按层分节） | 自增列全链路：三档 AUTOINC 锁、区间分配、计数器持久化 |
| [generated_columns.md](generated_columns.md) | B（生命周期） | 生成列全链路：求值链路、虚拟列索引、隐藏生成列家族 |

## 什么时候该建新篇

满足任一：

- 读者会带着"某个操作/特性怎么实现"的问题来查，而现在的答案散落在 3 篇以上
- 该主题同时改动多个层（语法 → 优化器 → 引擎存储）
- 需要讲清"两层的结合点"，而这个点在按层目录里没人负责写

只有 1 篇且没有独立数据结构时，留在上级目录，不建单文件目录。
