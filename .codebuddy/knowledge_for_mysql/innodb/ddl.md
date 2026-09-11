# InnoDB 原子 DDL 与 DDL Log 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 DDL 全流程（COPY / INPLACE / INSTANT 三种算法的选择策略与 MDL 三段式）、**DDL 的引擎接口**（handler / handlerton 如何承载 DDL：inplace alter 四件套、handlerton flags 决定路径、原子 DDL 的 `dd::Table` 条件持久化、`ha_extra`）、**并行 DDL**（社区版 8.0.27+ 确有：三阶段并行粒度、Loader 状态机、分片机制）、**Instant DDL 深度剖析**（行版本号机制、记录头版本字节、DD se_private_data）、**DDL 行为速查**（四维评估、VARCHAR 256 边界、Instant 限制、工具切表也要 MDL-X）、**待优化点**（分区级 MDL 等社区版可优化空间）、原子 DDL 与 `mysql.innodb_ddl_log`（记录类型与生命周期）、online DDL 的 row log（格式 / 生成 / 回放 / 两种形态）、DROP TABLE 与 TRUNCATE TABLE 的 InnoDB 实现、B-tree 物理释放（btr_free 机制）。本文件内容由 undo_log.md 的"DDL 与 Undo"章节独立而成——DDL log 只是因"DDL 不记数据 undo"而生，其机制本身与 undo 无直接关系。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [DDL 全流程](#ddl-全流程)
- [DDL 的引擎接口：handler / handlerton 如何承载 DDL](#ddl-的引擎接口handler--handlerton-如何承载-ddl)
- [DDL 行为速查：四维评估与实战陷阱](#ddl-行为速查四维评估与实战陷阱)
- [并行 DDL（社区版 8.0.27+ 确有）](#并行-ddl社区版-8027-确有)
- [Instant DDL 深度剖析](#instant-ddl-深度剖析)
- [待优化点](#待优化点)
- [为什么 DROP/TRUNCATE 不记数据 undo](#为什么-droptruncate-不记数据-undo)
- [DDL log](#ddl-log)
- [online DDL 与 row log](#online-ddl-与-row-log)
- [DDL 操作：各类 DDL 的具体实现](#ddl-操作各类-ddl-的具体实现)
  - [COPY DDL 完整流程](#copy-ddl-完整流程)
  - [DROP TABLE 流程](#drop-table-流程)
  - [TRUNCATE TABLE：rename + drop + create](#truncate-tablerename--drop--create)
  - [B-tree 的物理释放（btr_free 机制）](#b-tree-的物理释放btr_free-机制)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

8.0 原子 DDL 是"DDL 要么完全成功、要么完全回滚"的机制。InnoDB 侧的核心载体是 `mysql.innodb_ddl_log` 表和 `Log_DDL` 类（log0ddl.cc）：DDL 执行期把"破坏性物理操作"的意图写成日志记录，事务提交后再真正执行。

### 用途

解决 5.7 及之前 DDL 中途崩溃导致的不一致（如 `#sql-xxx.ibd` 孤儿文件、表空间状态错乱）。把 free B-tree、删 .ibd、改名等不可回滚的物理操作，用"先记录意图、提交后执行"的方式纳入事务原子性。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | DDL 非原子：崩溃可能留下孤儿 .ibd / 数据字典与实际文件不一致 |
| 8.0 | 引入 `mysql.innodb_ddl_log` 表 + `Log_DDL`（WL#7957 原子 DDL，WL#8347 DROP/TRUNCATE）；`HTON_SUPPORTS_ATOMIC_DDL`；物理操作延迟到提交后 replay |
| 8.0.20+ | DDL log 类型扩展（ALTER ENCRYPT TABLESPACE 等）；`innodb_print_ddl_logs` 调试开关 |

---

## 理论基础

### 设计模式

- **意图日志 / 延迟执行（intent log + deferred execution）**：执行期只向 DDL log 表写入"意图"记录，真正的物理操作推迟到事务提交后。破坏性操作一旦失败无法回滚，必须先记录、后执行
- **命令模式（Command）**：`DDL_Record` 是命令对象，携带类型 + 参数；`Log_DDL::replay` 按 `Log_Type` 分发到 `replay_free_tree_log` / `replay_delete_space_log` / `replay_drop_log` 等具体执行者
- **日志先行（Write-Ahead）**：物理操作执行前日志必须已持久化，与 ARIES 一致

### 相关论文

- **ARIES**（Mohan et al., 1992）：do-undo-redo 恢复模型。原子 DDL 用 DDL log（redo 性质）记录物理操作意图，配合元数据 undo，是 ARIES 思想在 DDL 上的工程化应用

### 算法与数据结构

- **DDL log 记录**：`DDL_Record` 类（log0ddl.h:78），字段 id / thread_id / type / space_id / page_no / index_id / table_id / file_path 等
- **幂等释放**：根页 `PAGE_INDEX_ID` 置 0（`BTR_FREED_INDEX_ID`），replay 前校验 index_id 匹配才释放——崩溃恢复重复 replay 安全
- **extent 级释放**：B-tree 释放粒度是文件段的一个 extent，而非逐页，一个 mtr 一个 extent 控制 mini-transaction 大小

### 类似实现对比

【待补充】PostgreSQL 的 DDL 事务化（DDL 与 DML 同事务）、Oracle 的原子 DDL 机制对比

### 历史背景

5.7 的 DDL 分两阶段（执行 + 元数据更新）且不原子，崩溃恢复靠 `innodb_force_recovery` 人工清理孤儿文件。8.0 引入原子 DDL 的动机：MySQL 官方 DD（data dictionary）表重构（WL#6041）后，元数据本身存储在 InnoDB 表里，DDL 的元数据修改天然有 undo；顺势把物理操作也纳入原子性，形成完整方案。

---

## DDL 全流程

本章给出 DDL 的**骨架**：先把"server 层怎么选算法、MDL 怎么拿放"讲清，后面各章（DDL log、row log、DROP/TRUNCATE）都是这条骨架上的某个环节。

### 三种算法与选择策略

ALTER TABLE 有三种算法，描述的是**执行逻辑**（数据怎么搬），与"是否 Online"（是否阻塞 DML）是**两个正交维度**：

| 算法 | 执行位置 | 数据怎么搬 | 是否 Online | 典型场景 |
|------|---------|-----------|------------|---------|
| **COPY** | server 层 | 建临时表 `#sql-xxx` → `copy_data_between_tables` 逐行走 SQL 层 INSERT | **一定不 Online**（全程 MDL X） | 改列类型、删主键、改表字符集 |
| **INPLACE** | InnoDB 层 | 引擎内部处理；可能只改元数据，也可能 bulk load **重建表** | **不一定**（看引擎返回值） | 加索引、加列、OPTIMIZE |
| **INSTANT** | 元数据 | 只改数据字典，不搬数据不重建 | 是 | 8.0 加列、改默认值、重命名表 |

选择策略（`mysql_alter_table`，sql/sql_table.cc）：

1. 用户显式指定 `ALGORITHM` → 用用户的；不支持就报错（**不会偷偷降级**）
2. 未指定 → 问引擎 `check_if_supported_inplace_alter`，优先 INPLACE，不支持则 COPY
3. 强制走 COPY 的条件：`old_alter_table=ON`（且未指定 INPLACE/INSTANT）、`is_inplace_alter_impossible`、分区变更且引擎不支持 auto-partition

引擎通过 `check_if_supported_inplace_alter` 返回 `enum_alter_inplace_result`，**这个返回值直接决定了 MDL 锁级别**：

| 返回值 | 含义 | MDL |
|--------|------|-----|
| `HA_ALTER_INPLACE_NOT_SUPPORTED` | 不支持 inplace | 走 COPY（全程 X） |
| `HA_ALTER_INPLACE_EXCLUSIVE_LOCK` | 支持 inplace 但需排他 | X（**不 Online**） |
| `HA_ALTER_INPLACE_SHARED_LOCK` | 共享锁 | S |
| `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE` | prepare 后降级为共享 | X → S |
| `HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE` | prepare 后**不锁** | X → 无（**这就是 Online**） |
| `HA_ALTER_INPLACE_INSTANT` | 只改元数据 | 几乎不锁 |

注意 `*_AFTER_PREPARE` 后缀：**prepare 阶段需要 exclusive，prepare 之后可以降级**——这就是 online DDL 的 MDL 降级机制，也是"DDL 期间 DML 为什么能并发"的答案。

### MDL 锁的三段式（online DDL 的骨架）

所有 DDL（不论哪种算法）都遵循这个 MDL 流转：

```
① 开始：拿 MDL EXCLUSIVE，做准备工作
        （建临时表结构 / 分配 row log / 打开新表）
② 执行：降级为 MDL SHARED（或 NO），执行真正的 DDL
        （重建表 / 建索引）—— 此期间 DML 并发 ✅
③ 提交：升级回 MDL EXCLUSIVE，apply row log + 切换可见性
        （rename 换表 / 让新索引生效）—— 此期间阻塞 DML ❌
```

三段分别对应 handler 的三个回调：

```
① prepare_inplace_alter_table()     ← MDL X
② inplace_alter_table()             ← MDL S / NONE（online 窗口）
③ commit_inplace_alter_table()      ← 升回 MDL X（切换窗口）
```

**为什么必须"先升回 X 再切换"**：见「online DDL 与 row log」章——核心是"切换哪个结构对外可见"必须排他；且第二次 apply 要"追平"（`head.total == tail.total`），只要 DML 还在写 row log 就永远追不平。

**Online DDL 并不是绝对不锁表**：如果第 ① 步拿不到 MDL X（表上有慢查询/长事务），后续**所有**操作（包括 SELECT）都会被阻塞，表现为典型的"锁表"。所以线上 DDL 仍要避开长事务。

### COPY 与 INPLACE 表重建（rebuild）为何"像"

它们**确实像**——结果形态都是"建一张新表 → 搬数据 → rename 替换"，都需要额外空间：

| | COPY | INPLACE 表重建（rebuild） |
|---|---|---|
| 执行层 | server 层 | InnoDB 层 |
| 数据搬运 | `copy_data_between_tables` 逐行走 SQL 层 | 扫描聚簇索引 + `ddl::Builder` bulk load 构建索引树 |
| DML | **全程阻塞** | **online**（靠 row log 转发） |
| row log | 不用（DML 进不来） | 必须用 |
| 临时文件 | server 层 `#sql-xxxx` | InnoDB 层 `ibXXXXXX`（在 `--tmpdir` 下） |
| rename | 有 | 有（都换了表） |
| 额外空间 | 需要 | **也需要**（会建 InnoDB 临时数据文件） |

**关键认知**：INPLACE 描述的是"**表**"而不是"数据文件"——只要不创建 server 层临时表就是 INPLACE。但很多 INPLACE DDL 仍会重建表（增加主键、重建主键、删列、调整列顺序、改 ROW_FORMAT、OPTIMIZE 等），都要创建 InnoDB 临时数据文件、需要额外空间。

所以"像"在于结果形态（换表 + 额外空间），区别在于**搬运方式 + 是否 online**。

---

## DDL 的引擎接口：handler / handlerton 如何承载 DDL

前面「DDL 全流程」讲的是** server 层的调度逻辑**。本章落到**接口层**：这些调度最终通过 handler / handlerton 的哪些方法落到引擎？（两层接口的完整剖析见 [`../server/handler.md`](../server/handler.md)，本章只讲与 DDL 相关的部分。）

### inplace alter 四件套 = MDL 三段式的实现载体

`handler` 的四个虚函数（sql/handler.h）：

| 接口 | 行号 | MDL 阶段 | 干什么 |
|------|------|---------|--------|
| `check_if_supported_inplace_alter` | 6175 | 决策 | 返回 `enum_alter_inplace_result` → 决定算法与 MDL 级别 |
| `prepare_inplace_alter_table` | 6257 | ① X 准备 | 建 new_table、分配 row log |
| `inplace_alter_table` | 6294 | ② S/NONE 执行 | bulk load 重建，DML 并发 |
| `commit_inplace_alter_table` | 6352 | ③ X 提交 | apply row log + rename 切换 |

「DDL 全流程」章那张 `enum_alter_inplace_result` 返回值表，就是 `check_if_supported_inplace_alter` 的返回值——**引擎通过这一个函数就决定了整个 DDL 的算法与锁行为**。

### handlerton 的 flags 决定 DDL 路径

```cpp
#define HTON_ALTER_NOT_SUPPORTED (1 << 1)  // 引擎不支持 alter
#define HTON_CAN_RECREATE (1 << 2)         // Delete all is used for truncate
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12)

inline bool ddl_is_atomic(const handlerton *hton) {        // handler.h:2957
  return (hton->flags & HTON_SUPPORTS_ATOMIC_DDL) != 0;
}
```

InnoDB 的声明（ha_innodb.cc:5173）：

```cpp
innobase_hton->flags = HTON_SUPPORTS_EXTENDED_KEYS | HTON_SUPPORTS_FOREIGN_KEYS |
                       HTON_SUPPORTS_ATOMIC_DDL | HTON_CAN_RECREATE |
                       HTON_SUPPORTS_SECONDARY_ENGINE | HTON_SUPPORTS_TABLE_ENCRYPTION |
                       HTON_SUPPORTS_GENERATED_INVISIBLE_PK;
```

两处直接呼应本文其他章节：
- **`HTON_SUPPORTS_ATOMIC_DDL`** → `copy_data_between_tables` 里"是否提前提交"的分岔（见「COPY DDL 完整流程」）
- **`HTON_CAN_RECREATE`**（注释 "Delete all is used for truncate"）→ TRUNCATE 走 rename + drop + create 路径（见「TRUNCATE TABLE」）

### 原子 DDL 的另一半：dd::Table 修改的条件持久化

原子 DDL 常被理解为"DDL log + 提交后 replay"（见「DDL log」章）。但还有一半藏在 handler 接口的注释里（handler.h:6750、6419、6836 三处重复强调）：

```cpp
/* truncate 的注释 handler.h:6750 */
@note  Changes to dd::Table object done by this method will be saved
       to data-dictionary only if storage engine supports atomic DDL
       (i.e. has HTON_SUPPORTS_ATOMIC_DDL flag set).

/* rename_table 的注释 handler.h:6419 */
Storage engines which support atomic DDL (i.e. having
HTON_SUPPORTS_ATOMIC_DDL flag set) are allowed to adjust this object.
```

**机制**：引擎的 handler 方法会**修改传入的 `dd::Table` 对象**（填 se_private_data、改表空间信息等），但这些修改**只有引擎声明支持原子 DDL 时才会被持久化到数据字典**。

**为什么**：持久化是随 DDL 事务提交的。不支持原子 DDL 的引擎，DDL 失败时无法保证"DD 修改"与"引擎内部状态"一起回滚，所以 server 干脆**不接受它对 DD 的修改**——避免出现"DD 已改、引擎没改"的不一致。

这也解释了两个默认实现（handler.h:6754）：

```cpp
virtual int truncate(dd::Table *table_def) { return HA_ERR_WRONG_COMMAND; }
virtual int optimize(THD *, HA_CHECK_OPT *) { return HA_ADMIN_NOT_IMPLEMENTED; }
```

- `optimize` 不实现 → InnoDB 返回 `HA_ADMIN_TRY_ALTER`，让 server 改写为 `ALTER TABLE ... ENGINE=InnoDB`（这正是 OPTIMIZE 的真实实现路径）
- `truncate` 不实现 → 走 `HTON_CAN_RECREATE` 的 recreate 路径

### ha_extra：DDL 如何给引擎下"行为开关"

`handler::extra`（表级虚函数，与 `external_lock` **平级**）是 DDL 用来调整引擎行为的旁路通道。COPY DDL 里用到一对：

- `HA_EXTRA_BEGIN_ALTER_COPY` → `skip_alter_undo = 1`
- `HA_EXTRA_END_ALTER_COPY` → `alter_stats_rebuild()` 重算统计 + 置 0

其作用是令 COPY 往中间表插入时**同时免掉 undo 与行锁**（下游 `row_ins` 据此置 `BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG`）——这是 COPY 算法最关键的优化，部分抵消了它走 SQL 层的劣势。

完整逐行剖析（含 `skip_alter_undo` 在 row0ins.cc 的消费点、roll_ptr 伪造 `1ULL<<55`、"只对 INSERT 生效"的 `ut_ad(!skip_alter_undo)` 断言约束、以及 `extra()` 为何绝不能碰 trx）见 [`../server/handler.md`](../server/handler.md) 的「handler::extra」章。

---

## DDL 行为速查：四维评估与实战陷阱

> 参考月报《云原生数据库 PolarDB MySQL 8.0.2 DDL 介绍》（2023/09）的组织思路，但**所有结论均回社区版 8.0.39 源码核实**，并标注哪些是 PolarDB 私有增强。

### 为什么需要"四个维度"而不是只看算法

月报给的框架很实用：判断一个 DDL 的影响，光知道"COPY/INPLACE/INSTANT"不够，用户真正关心的是**四个正交维度**：

| 维度 | 关心的问题 | 对应本文概念 |
|------|-----------|-------------|
| **是否锁表**（允许并发 DML） | 会不会影响业务写入 | Online 与否（`check_if_supported_inplace_alter` 返回值决定 MDL 级别） |
| **是否重建表** | 要多久（是否随表规模增长） | `need_rebuild` / `new_clustered` |
| **是否只改元数据** | 能否秒级完成 | INSTANT（`innobase_support_instant`） |
| **是否支持并行** | 能否多线程加速 | `innodb_ddl_threads` 等（见下） |

**关键认知**：Online DDL 只在"改元数据"这一步申请表互斥锁（一般 <1s），期间允许读写；非 Online 的 DDL 全程锁表。

### 具体操作 × 四维对照（源码依据）

InnoDB 用三组 `HA_ALTER_FLAGS` 常量（`handler0alter.cc:114-172`）决定每个操作走哪条路：

```cpp
/** Operations for rebuilding a table in place */
INNOBASE_ALTER_REBUILD =
    ADD_PK_INDEX | DROP_PK_INDEX | CHANGE_CREATE_OPTION
    /* CHANGE_CREATE_OPTION needs to check innobase_need_rebuild() */
  | ALTER_COLUMN_NULLABLE | ALTER_COLUMN_NOT_NULLABLE
  | ALTER_STORED_COLUMN_ORDER | DROP_STORED_COLUMN
  | ADD_STORED_BASE_COLUMN
    /* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
  | RECREATE_TABLE;

/** Operation allowed with ALGORITHM=INSTANT */
INNOBASE_INSTANT_ALLOWED =
    ALTER_COLUMN_NAME | ADD_VIRTUAL_COLUMN | DROP_VIRTUAL_COLUMN
  | ALTER_VIRTUAL_COLUMN_ORDER | ADD_STORED_BASE_COLUMN
  | ALTER_STORED_COLUMN_ORDER | DROP_STORED_COLUMN;

/** ...can perform without rebuild */
INNOBASE_ALTER_NOREBUILD =
    ADD_INDEX | ADD_UNIQUE_INDEX | ADD_SPATIAL_INDEX
  | DROP_INDEX | DROP_UNIQUE_INDEX | RENAME_INDEX
  | ALTER_COLUMN_NAME | ALTER_COLUMN_EQUAL_PACK_LENGTH
  | ADD/DROP_VIRTUAL_COLUMN | ALTER_COLUMN_INDEX_LENGTH | ...;
```

由此得到四维对照表：

| 操作 | 重建表 | 秒级 | 锁表 | 并行 | 说明 |
|------|--------|------|------|------|------|
| **ADD COLUMN** | 否/是¹ | ✅/❌ | 否 | 否 | `ADD_STORED_BASE_COLUMN` |
| **DROP COLUMN** | 否/是¹ | ✅/❌ | 否 | 否 | `DROP_STORED_COLUMN` |
| **RENAME COLUMN** | 否 | ✅ | 否 | 否 | `ALTER_COLUMN_NAME`，纯元数据 |
| ADD/DROP VIRTUAL COLUMN | 否 | ✅ | 否 | 否 | 虚拟列不存数据 |
| **ADD INDEX**（二级） | 否 | ❌ | **否**（LOCK=NONE） | **✅** | `INNOBASE_ONLINE_CREATE`，见「并行 DDL」 |
| **DROP INDEX** | 否 | ✅ | 否 | 否 | `DROP_INDEX`，只改元数据 |
| RENAME INDEX | 否 | ✅ | 否 | 否 | `RENAME_INDEX` |
| **ADD / DROP PRIMARY KEY** | **是** | ❌ | **是** | ✅ | `ADD_PK_INDEX`/`DROP_PK_INDEX`，代价最高 |
| **改列类型**（MODIFY/CHANGE） | **是** | ❌ | 视情况 | ✅ | `ALTER_STORED_COLUMN_TYPE` |
| **改列顺序** | 是/否¹ | ✅/❌ | 否 | — | `ALTER_STORED_COLUMN_ORDER` |
| 改 NULL/NOT NULL | **是** | ❌ | 视情况 | ✅ | `ALTER_COLUMN_NULLABLE`/`NOT_NULLABLE` |
| **改列默认值 / 列可见性** | 否 | ✅ | 否 | 否 | 属 `INNOBASE_INPLACE_IGNORE`，InnoDB 不关心 |
| **OPTIMIZE / ALTER ENGINE** | **是** | ❌ | 否 | **✅** | `RECREATE_TABLE` |
| 改表选项（ROW_FORMAT 等） | 是/否¹ | — | 视情况 | ✅ | `CHANGE_CREATE_OPTION` |

**¹ 重要发现——"加/删列有时秒回、有时跑很久"的源码解释**：

`ADD_STORED_BASE_COLUMN`、`DROP_STORED_COLUMN`、`ALTER_STORED_COLUMN_ORDER` **同时出现在** `INNOBASE_ALTER_REBUILD` 和 `INNOBASE_INSTANT_ALLOWED` 两个集合里。源码注释直接点明：

```cpp
Alter_inplace_info::ADD_STORED_BASE_COLUMN
/* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
```

即**加/删列既可能走 INSTANT，也可能退化为 REBUILD**，由 `innobase_need_rebuild()` 判定。退化条件就是「Instant DDL 深度剖析」里那几条限制（压缩表、行大小超限、非末尾加列、有全文索引等）。**踩中任一条 → 从秒级退化为全表重建**，这正是生产上 ADD COLUMN 表现不一致的根因。

### 社区版 8.0.39 **有**并行 DDL（不是 PolarDB 独有）

月报把"并行 DDL"作为 PolarDB 能力介绍，但社区版 8.0.27+ 就引入了（`ha_innodb.cc:1107-1124`）：

```cpp
// 并行读（扫描聚簇索引）线程数
static MYSQL_THDVAR_ULONG(parallel_read_threads, ..., "Number of threads to do parallel read.",
                          4 /* Default */, 1 /* Min */, Parallel_reader::MAX_THREADS /* Max */, 0);

// DDL 可用内存
static MYSQL_THDVAR_ULONG(ddl_buffer_size, ..., "Maximum size of memory to use (in bytes) for DDL.",
                          1048576 /* Default: 1MB */, 65536 /* Min: 64KB */, 4294967295 /* Max */, 0);

// DDL 最大线程数
static MYSQL_THDVAR_ULONG(ddl_threads, ..., "Maximum number of threads to use for DDL.",
                          4 /* Default */, 1 /* Min */, 64 /* Max */, 0);
```

即 `innodb_ddl_threads`（默认 4，范围 1-64）、`innodb_ddl_buffer_size`（默认 1MB）、`innodb_parallel_read_threads`（默认 4）。使用点见 `handler0alter.cc:1473`（`thd_parallel_read_threads`）、`6353`（`thd_ddl_threads`）。

**差异**：PolarDB 是调优到 15-20 倍加速；社区版有机制但加速比没那么夸张。

### 实战陷阱一：VARCHAR 扩展的 256 字节边界

月报重点提示的坑，社区版源码依据在 `sql/field.h:4291-4294`（`Field_json` 的同类实现，VARCHAR 语义一致）：

```cpp
uint32 get_length_bytes() const override {
  assert(m_elt_type == MYSQL_TYPE_VARCHAR);
  return field_length > 255 ? 2 : 1;      // ★ 长度字节数：≤255 用 1 字节，>255 用 2 字节
}
```

`Field_varstring` 用成员 `length_bytes`（field.h:3584，注释 "Store number of bytes used to store length (1 or 2)"）记录该值。

**后果**：把 VARCHAR 从 **≤255 字节扩到 ≥256 字节**会改变 `length_bytes`（1→2），行格式变化 → **无法只改元数据，退化为 COPY（全程锁表）**。

**建议**（月报给的）：varchar 是变长存储、磁盘只存实际长度，建表时**直接把最大长度设到 256 以上**，避免后续扩展踩坑。不确定时可显式 `ALGORITHM=INPLACE` 试，不支持会直接报错而非偷偷降级：

```
ALTER TABLE t ALGORITHM=INPLACE, CHANGE COLUMN c1 c1 VARCHAR(256);
ERROR 0A000: ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

### 实战陷阱二：Instant ADD/DROP COLUMN 的表级限制

社区版的判断在 `dict_table_t::support_instant_add_drop()`（`dict0dict.ic:1371-1376`）：

```cpp
inline bool dict_table_t::support_instant_add_drop() const {
  return (
      !DICT_TF_GET_ZIP_SSIZE(flags) &&                         // 非压缩表（ROW_FORMAT≠COMPRESSED）
      space != dict_sys_t::s_dict_space_id &&                   // 非 DD 表空间
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS_HAS_DOC_ID) &&   // 无 FTS_DOC_ID
      !is_temporary() &&                                        // 非临时表
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS) &&              // 无全文索引
      !is_system_table);                                        // 非系统表
}
```

**即：压缩表、全文索引表、临时表、系统表 → 不支持 Instant**。这点与月报一致（月报还提到"需已有主键""只能加到末尾"——Instant 的语义是新列不落已有行、用默认值，故只能加在末尾；若表无主键，末尾的隐式主键会导致失败）。

**不满足时**：加列退化为 **INPLACE 全表重建**（允许并发 DML、可用并行 DDL 加速）。

### 实战陷阱三：INSTANT 只支持白名单内的操作

社区版 `innobase_support_instant`（handler0alter.cc:821-907）明确了 INSTANT 的操作范围。`INNOBASE_INSTANT_ALLOWED`（148 行起）白名单之外的操作一律 `INSTANT_IMPOSSIBLE`：

```cpp
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN | ...;
```

`INSTANT_OPERATION` 分类（840-849）与几个"不支持"的边界（871-904）：
- `COLUMN_RENAME_ONLY`：仅列重命名 → 需 `ok_to_rename_column`；**若列被外键引用则不允许 INSTANT**（777-801）
- `VIRTUAL_ADD_DROP_ONLY`：仅虚拟列增删
- `VIRTUAL_ADD_DROP_WITH_RENAME`：**"Not supported yet in INPLACE. So not supporting here as well."**（886）→ 不支持
- `INSTANT_ADD` / `INSTANT_DROP`：加/删存储列 → 需 `table->support_instant_add_drop()`

另外 `innobase_need_rebuild`（922）：`if (is_instant(ha_alter_info)) return false;` —— INSTANT 不重建表。

### 实战陷阱四：任何"无锁变更"工具在切表时都要拿 MDL-X

月报强调的一点，值得记住：**gh-ost / pt-osc / DMS 无锁变更，在最后"切表（修改元数据）"时同样至少拿 MDL-X**，Online DDL 也一样。所以：
- 执行 DDL 仍需确保不被大事务/大查询堵塞（否则 MDL-X 等待会连带阻塞后续所有请求，甚至连接打满）
- "无锁"是相对的（仅指数据迁移阶段）

### 社区版 vs PolarDB 差异一览

| 能力 | 社区版 8.0.39 | PolarDB |
|------|--------------|---------|
| 并行 DDL | ✅ 有（`innodb_ddl_threads` 等，8.0.27+） | ✅ 有，调优到 15-20 倍 |
| Instant ADD/DROP COLUMN | ✅ 有（8.0.12+ ADD、8.0.29+ DROP） | ✅ 有 |
| 分区级 MDL | ❌ **社区版无** | ✅ `loose_partition_level_mdl_enabled`，DDL 不影响不涉及分区的 DML |
| 非阻塞 DDL | ❌ 社区版无 | ✅ PolarDB 特性（见月报 2022/10） |

---

## 为什么 DROP/TRUNCATE 不记数据 undo

DROP TABLE 和 TRUNCATE TABLE **不记录用户数据的 undo log**——它们不像 `DELETE FROM t` 那样为每一行产生 undo record。原因：

1. **没有逐行 delete**：DROP/TRUNCATE 是物理操作（删表空间文件、free btree root），不是逐行删除，没有逐行 undo 的产生点
2. **不需要 MVCC 旧版本**：被删或被清空的表数据不需要一致性读的历史版本——表没了或被重建了，没有 read view 会访问旧数据
3. **DDL 隐式提交不可回滚**：DDL 语句本身是原子事务，用户层面不可回滚，不存在"回滚到 DDL 前"的需求

因此 DDL 的原子性不靠数据 undo，靠 DDL log + dd 元数据 undo（详见下文）。

---

## DDL log

### 存储与记录类型

`mysql.innodb_ddl_log` 是普通 InnoDB 表。DDL log 记录通过 `DDL_Log_Table::insert` 插入（log0ddl.h:257），插入本身产生 undo + redo——这个 undo 是为了 DDL 事务失败时回滚对 `innodb_ddl_log` 表自身的修改，是元数据层面保证，与用户数据无关。

记录类型（`Log_Type` 枚举，log0ddl.h:45，uint32_t 节省表空间）：

| 类型 | 值 | 含义 | 对应 replay |
|------|----|------|-------------|
| `FREE_TREE_LOG` | 1 | 释放索引 B-tree | `replay_free_tree_log`（btr_free_if_exists） |
| `DELETE_SPACE_LOG` | 2 | 删除表空间文件 | `replay_delete_space_log`（fil_delete_tablespace） |
| `RENAME_SPACE_LOG` | 3 | 重命名表空间文件 | `replay_rename_space_log`（fil_op_replay_rename_for_ddl） |
| `DROP_LOG` | 4 | 删除 innodb_table_metadata 条目 | `replay_drop_log` |
| `RENAME_TABLE_LOG` | 5 | dict cache 中重命名表 | `replay_rename_table_log` |
| `REMOVE_CACHE_LOG` | 6 | 从 dict cache 移除表 | `replay_remove_cache_log` |
| `ALTER_ENCRYPT_TABLESPACE_LOG` | 7 | 表空间加密 | `replay_alter_encrypt_space_log` |
| `ALTER_UNENCRYPT_TABLESPACE_LOG` | 8 | 表空间解密 | 同上 |

`DDL_Log_Table` 的设计要点：**线程只访问/删除自己的记录**（按 thread_id 区分），无需行锁（log0ddl.h:237 注释）。

### 生命周期

**阶段 1：执行期（DDL 事务未提交）——只写 log，不执行物理操作**

向 `innodb_ddl_log` 表 insert 物理操作记录。此时物理操作本身不执行。`insert_free_tree_log` 注释说得很明确："if committed, will be redo only"（log0ddl.cc:884）——free tree 要等事务提交后通过 replay 才执行。所以 DDL 失败可以干净回滚：物理操作还没做，只需 undo 回滚元数据修改和 ddl_log 表的 insert。

**阶段 2：提交后 post_ddl——真正执行物理操作**

`Log_DDL::post_ddl(thd)`（log0ddl.cc:1905）被调用，`replay_by_thread_id`（1506）按线程 ID 查找该 DDL 写入的所有记录，`replay`（1576）根据类型执行物理操作，完成后 `delete_by_ids` 删除已处理记录。

**post_ddl 在 commit 和 rollback 之后都会被调用**（注释原文 "Replay and clean DDL logs after DDL transaction commits or rollbacks"，log0ddl.h:484）。调用者是 server 层 `hton->post_ddl(thd)`（如 TRUNCATE 走 `Sql_cmd_truncate_table::cleanup_base` sql_truncate.cc:399）→ `innobase_post_ddl`（ha_innodb.cc:5101）。rollback 场景下 replay 是空操作（见下）。

**阶段 3：崩溃恢复——replay_all**

`Log_DDL::recover`（log0ddl.cc:1942）调用 `replay_all()` 重放所有残留 DDL log：

- DDL 事务已提交（binlog 有记录）→ DDL log 记录存在 → replay 确保物理操作完成
- DDL 事务未提交 → undo 回滚该事务 → `innodb_ddl_log` 的 insert 被撤销（记录消失）→ 无 log 可 replay → 表恢复原状

**原子性闭环的判定逻辑**：原子 DDL 的本质是：**物理操作通过 DDL log 延迟到提交后执行，元数据修改通过 undo 可回滚**，两者结合保证 DDL 要么完全成功（提交 + 物理操作 replay 完成），要么完全回滚（undo 回滚元数据，物理操作从未执行）。

关键闭环（回答"为什么这样实现原子性"）：

1. **执行期只写意图**：不可回滚的破坏性操作（free B-tree / 删 .ibd / rename 文件）执行期一律不做，只向 `innodb_ddl_log` insert 记录（普通表 insert，产生 undo + redo，随 DDL 事务持久化）
2. **DDL log 记录的存在性由 undo 决定**：DDL 事务未提交则记录被 undo 撤销；已提交则记录随事务落盘。因此"日志记录是否存在"本身就是"该 DDL 是否已提交"的可靠判据
3. **post_ddl / 崩溃恢复都基于该判据执行物理操作**：查得到记录 → 该 DDL 已提交 → 执行（且物理操作本身产生的 redo 保证崩溃后不丢）；查不到 → 从未提交 → 物理操作从未执行，状态天然一致
4. **为什么破坏性操作必须延迟**：若执行期就 free B-tree，DDL 失败回滚时这些页已归还 FSP_FREE 且可能被其他事务覆盖，物理上无法恢复。延迟使"回滚"永远只需撤销日志记录，无需撤销物理操作

---

## online DDL 与 row log

> 上一节的 DDL log 与这里的 row log 名字都带 log，但**完全是两回事**：DDL log 管"DDL 崩溃恢复/原子性"，row log 管"online DDL 重建期间的 DML 并发"。

### online DDL 的两种形态（先分清，否则会混淆）

**commit 阶段不一定有 rename**——rename 只属于"重建表"，不是所有 DDL 的共性（`commit_cache_rebuild` 内有 `assert(ctx->need_rebuild())`；且 `ut_ad(new_clustered == ctx->need_rebuild())`）。两种形态的 row log 机制完全不同：

| | **加二级索引**（add index，非 rebuild） | **重建表**（rebuild：`OPTIMIZE` / `ALTER ENGINE=InnoDB` / 加主键 / 改列类型） |
|---|---|---|
| row log 挂在哪 | **新索引** `ctx->add_index[a]->online_log` | **旧表聚簇索引** `clust_index->online_log` |
| 记录函数 | `row_log_online_op`（`ROW_OP_INSERT/DELETE`） | `row_log_table_insert/update/delete`（`ROW_T_*`） |
| 记录内容 | 索引 tuple | 整行（含虚拟列、旧 PK） |
| 回放函数 | `row_log_apply` | `row_log_table_apply` |
| 回放次数 | **1 次**（`ddl0builder.cc`，DEBUG_SYNC `row_log_apply_before`） | **2 次** |
| **需要 rename** | **否**（索引建完置 `ONLINE_INDEX_COMPLETE` 即生效） | **是**（新表改名上线、旧表改临时名后删） |

分配分支见 `prepare_inplace_alter_table`：`!ctx->online` 或 `new_clustered` 时**不给 add_index 分配 log**，rebuild 场景则单独给旧表聚簇索引分配。

**为什么 rebuild 要 rename、add index 不要**：两条独立需求——`rebuild` 创建了**一整张新表**（新 `.ibd` + 新聚簇索引），要成为对外可见的表只能 rename 替换；`add index` 只是往现有表挂索引，表没换，无需 rename。而 row log 只服务于 **online**（非 online 会锁表、DML 进不来，压根不分配 row log）。所以因果是：`online → 需要 row log`、`rebuild → 需要 rename`，两条独立，`OPTIMIZE` 恰好同时满足。

### row log 是什么

row log（online log）是 InnoDB online DDL 期间记录用户 DML 的日志，只服务于 online 场景（`!ctx->online` 时不分配）。结构是**「内存块 + 磁盘临时文件」两段式**：

- 结构 `row_log_t`（`row0log.cc`），挂在 `dict_index_t::online_log`（`dict0mem.h`）上，字段含 `file`（`ddl::Unique_os_file_descriptor` 临时文件）、`mutex`、`tail`（写者）、`head`（读者）、`blobs`（off-page 列 page 号映射）、`table`、`same_pk`、`add_cols`、`col_map`（旧列 id → 新列 id）；
- 内存块由 `row_log_block_allocate` 分配，大小 = `srv_sort_buf_size`；写满后经 `row_log_tmpfile` 溢出到磁盘临时文件（`ddl::file_create_low`）；
  - **临时文件命名**：`row_log_tmpfile` → `ddl::file_create_low(log->path)` → `innobase_mysql_tmpfile(path)` → `mysql_tmpfile_path(path, "ib")`，磁盘上真实文件名是 **`ib` 前缀 + 6 位随机字符**（如 `ibA1b2C3`）；目录来自 `thd_innodb_tmpdir(thd)`（`prepare_inplace_alter_table` 中取到的 `path`），`file_create_low` 里若为 nullptr 则退回 `innobase_mysql_tmpdir()`（全局 `--tmpdir`）。PFS 中显示的可读名 "Innodb Merge Temp File"（`Datafile::make_filepath`）只是监控用名，与磁盘真实名不同；
- 读者（apply）追上写者时 reset 计数并 truncate 文件（`row_log_t` 上方注释）——生产者-消费者 + 环形缓冲 + 磁盘溢出；
- 记录点：DML 执行时若 `index->online_log != nullptr`，由 `row_log_table_insert / update / delete`（`row0log.cc`）写入；
- 回放点：`row_log_table_apply`（`row0log.cc`）按主键顺序把 row log 逐条 apply 到新表。

### 记录格式（mrec）

`row_log_table_low`（`row0log.cc`）是 insert/update 的编码入口，一条记录格式：

```
[op:1字节][old_pk:可选][extra_size:1或2字节][记录头extra字节][记录data][虚拟列:可选]
```

- 操作类型枚举（`row0log.cc`）：table rebuild 用 `ROW_T_INSERT=0x41 / ROW_T_UPDATE=0x42 / ROW_T_DELETE=0x43`；secondary index online create 用 `ROW_OP_INSERT=0x61 / ROW_OP_DELETE=0x62`；
- 记录头 `ROW_LOG_HEADER_SIZE=2`；`extra_size` 变长编码（<0x80 一字节，否则 `0x80|high` + `low` 两字节）；
- `old_pk` 仅在主键变化（`!same_pk`）的 update 里携带；
- 记录体用 `rec_serialize_dtuple`（mrec 编码）序列化，回放时 `row_log_table_apply_convert_mrec` 反解。

**三种操作各自记什么**（格式上的关键差异）：

| 操作 | old pk | record data | 虚拟列 | 原因 |
|------|--------|-------------|--------|------|
| `ROW_T_INSERT` | 否 | 是 | 有则记 | 插入新表靠 `col_map` 映射即可定位，无需旧表主键 |
| `ROW_T_UPDATE` | **可能**（主键变化时） | 是 | 有则记 | 新旧表主键不同时，需 old pk 在新表上定位待更新行 |
| `ROW_T_DELETE` | 是 | **否** | **否** | 应用时只需在新表按主键打删除标记，无需行内容 |

**temp 格式（省空间的紧凑行格式）**：row log 里的 record data 用的是 **temp 格式**而非完整 compact/redundant 格式——它**省略记录头信息**，只保留变长字段长度列表 + NULL 标志位 + 字段数据。相关接口：`rec_get_converted_size_temp`（算大小）、`rec_convert_dtuple_to_temp`（tuple→temp）、`rec_init_offsets_temp` / `rec_init_null_and_len_temp`（解析）。temp 格式也用于 sort buffer，目的同样是省空间。

**instant add column 对格式的影响**：做过 instant add column 的表，行记录头多了 `INSTANT_FLAG` 标志位（表示是否存在"字段数量"字段）。row log 记录这类表的 record 时需**额外记 info bits**，解析时（`rec_init_null_and_len_temp`）用 index 上的 instant 标志辅助判断 temp 记录里是否含 info bits。

### 与 DDL log 的区别

| 维度 | row log（online log） | DDL log |
|------|----------------------|---------|
| 记录内容 | 用户 DML（逐行 insert/update/delete） | DDL 物理操作意图（free btree / delete space / rename） |
| 解决的问题 | online DDL 并发正确性（不阻塞 DML） | 崩溃恢复 + 原子性 |
| 载体 | 内存块 + **临时文件**（非 redo 保护） | 磁盘 `mysql.innodb_ddl_log` 表 + redo |
| 持久化 | 否（DDL 失败即作废） | 是（崩溃后 replay） |
| 大小限制 | `innodb_online_alter_log_max_size`（默认 128MB） | 无 |
| 生命周期 | DDL 期间收集 → apply 完释放 | 执行期写 → 提交后/崩溃时 replay |
| 粒度 | 行级 | 表空间 / B-tree 级 |

**为什么 row log 不需要 redo 持久化**：online DDL 期间用户的 DML 是正常提交到旧表的（redo/undo 已保证持久性）；row log 只是"把 DML 转发到新表"的影子记录。DDL 中途崩溃 → 整体回滚（靠 DDL log）→ 新表丢弃 → 旧表 + 已提交 DML 靠 redo 恢复，row log 用不上。**不存在"DDL 半成功、需要恢复 row log"的中间状态**——这个状态被 DDL log 的原子性排除了。真正需要 redo 持久化的是最后的 rename（见下）。

### 回放时机：apply 两次（并发控制根源是 MDL 锁）

```
ha_innobase::inplace_alter()                执行阶段，MDL = SHARED（DML 可并发）
   ...
   第一次 row_log_table_apply()             DEBUG_SYNC: row_log_table_apply1_before
   ← 提前消费、减少积压；apply 完释放锁，DML 恢复

ha_innobase::commit_inplace_alter_table()   提交阶段，MDL 升级为 EXCLUSIVE
   第二次 row_log_table_apply()             DEBUG_SYNC: row_log_table_apply2_before
   ← 追平（head.total == tail.total），接着 rename，全程持锁
```

`row_log_table_apply` 内部对聚簇索引加 X 锁（`rw_lock_x_lock`），而 DML 写 row log 需要 `index->lock` 的 S 锁（`row_log_table_low` 注释 "S-latched or X-latched"），所以**两次 apply 期间都会短暂阻塞 DML**；真正的差异在 apply 之后：

- 第一次 apply 是"清库存"，不是"追平"——apply 完放行，后续 DML 继续进 row log，留给第二次兜底；
- 第二次 apply 是"最后一次追平"（代码断言 `head.total == tail.total`），追平后**立即 rename 切表**。此刻若还放行 DML 落旧表，这些 DML 既没进 row log、也不会 apply 到新表，rename 后旧表被丢弃 → DML 真丢。所以第二次 apply 必须和 rename 绑成一个"不许插入"的原子窗口。

**MDL 为什么必须升级（根源）**：MDL 保护的是"表定义（元数据）的可见性"。执行阶段新结构在构建、旧结构仍是唯一对外可见的，apply 只是"追赶数据"、**不改变任何可见性**，MDL 保持 SHARED 即可；提交阶段要**切换**（rebuild 是 rename 换表，add index 是让新索引进入 `ONLINE_INDEX_COMPLETE`），切换的本质是"改变哪个结构对外可见"，此刻若还有 DML 持有旧定义在写，写入会落到即将被丢弃/失效的结构上 → 数据丢失，所以切换必须在"无人持有旧定义"的时刻完成 → 升级 MDL 为 EXCLUSIVE。

还有一条更硬的约束直接决定了先后顺序：**第二次 apply 要"追平"（`head.total == tail.total`），而只要 DML 还在写，row log 的 tail 就一直涨，head 永远追不上——永远追不平。** 所以因果是"要追平 + 切换 → 必须先让 DML 停 → 升级 MDL EXCLUSIVE → 才能第二次 apply 追平 → rename"，而不是"因为是第二次所以升级"。第一次 apply 不需要追平（有第二次兜底），因此能在 DML 并发下做。

### commit 与 rename（仅 rebuild 场景）

**rename 只发生在 rebuild 场景**——加索引的 online DDL 没有 rename（新索引置 `ONLINE_INDEX_COMPLETE` 即生效）。`commit_inplace_alter_table` 收尾由 `commit_cache_rebuild`（`handler0alter.cc`，内有 `assert(ctx->need_rebuild())`）完成：

```cpp
/* We already committed and redo logged the renames, so this must succeed. */
dict_table_rename_in_cache(ctx->old_table, ctx->tmp_name, false);  // 旧表 → 临时名
dict_table_rename_in_cache(ctx->new_table, old_name, false);       // 新表 → 正式名
```

- rename 是"新表上线"的原子切换点：新表改正式名、旧表改临时名（稍后物理删除）；
- **rename 是 redo logged 的**（注释原文 "redo logged the renames"）——这才是 DDL 的**真正持久化点**，rename 落盘后崩溃也能保证"新表已上线"；
- 因此 **rebuild 场景的** commit 阶段 = 第二次 apply（让新表数据完整）+ rename（切换 + 落盘），二者必须在 MDL exclusive 下连续完成；非 rebuild 的 online DDL（如加索引）只有一次 `row_log_apply`，无 rename。

### 相关系统变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `innodb_online_alter_log_max_size` | 134217728（128MB） | row log 总上限，超了报 `ER_INNODB_ONLINE_LOG_TOO_BIG`，online DDL 失败 |

---

## DDL 操作：各类 DDL 的具体实现

前面几章讲的是 DDL 的**通用骨架**（算法选择、MDL 流转、DDL log、row log）。本章落到**具体每类 DDL 怎么做**。

### COPY DDL 完整流程

COPY 是唯一完全在 server 层完成的路径，也是理解另两种算法的参照系。完整流程（`mysql_alter_table` → `copy_data_between_tables`，sql/sql_table.cc）：

```
① 拿 MDL X（全程持有 → 这是 COPY 不 Online 的根因）
② 建中间表：server 层按新定义建临时表 `#sql-xxxx`
③ to->file->ha_external_lock(F_WRLCK)              通知引擎"要往这张表写"
④ alter_table_manage_keys()                         索引管理（禁用/启用）
⑤ to->file->ha_start_bulk_insert()                 批量插入优化
⑥ to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY)    ★ skip_alter_undo=1：不记 undo、不加锁
⑦ while (iterator->Read()) { ... }                 ★ 逐行读源表 → 逐行写中间表（COPY 的本质）
⑧ to->file->ha_end_bulk_insert()
⑨ to->file->ha_extra(HA_EXTRA_END_ALTER_COPY)      重算统计 + skip_alter_undo=0
⑩ rename：中间表替换原表
```

**与 INPLACE rebuild 的对照**：二者都是"建新 + rename 替换"，差异只在 ②~⑦ 由谁做、怎么搬（见「DDL 全流程」章）。

### `copy_data_between_tables` 源码逐段剖析

核心函数 `copy_data_between_tables`（sql/sql_table.cc:18442，约 300 行），`from`=源表、`to`=新建中间表。COPY 的"搬数据"全在这里。

#### 第一段：准备（18461-18507）

**① 原子 DDL 的分岔**：

```cpp
/*
  If target storage engine supports atomic DDL we should not commit
  and disable transaction to let SE do proper cleanup on error/crash.
  Such engines should be smart enough to disable undo/redo logging
  for target table automatically.
*/
if ((!(to->file->ht->flags & HTON_SUPPORTS_ATOMIC_DDL) ||
     from->s->tmp_table) &&
    mysql_trans_prepare_alter_copy_data(thd))
  return -1;
```

只有**目标引擎不支持原子 DDL**（或源表是临时表）时才"提交并禁用事务"。InnoDB 支持 → **刻意不提前提交**，整个 COPY 留在一个事务里，交给引擎的 DDL log 机制收尾。这是原子 DDL 在 COPY 路径上的落点。

**② 打开写通道**：

```cpp
if (to->file->ha_external_lock(thd, F_WRLCK)) { ... }   // 通知引擎"要写这张表"
/* We need external lock before we can disable/enable keys */
alter_table_manage_keys(thd, to, from->file->indexes_are_disabled(), keys_onoff);
thd->check_for_truncated_fields = CHECK_FIELD_WARN;    // 拷贝期间截断→只告警
from->file->info(HA_STATUS_VARIABLE);                  // 取源表行数（进度条用）
to->file->ha_start_bulk_insert(from->file->stats.records);  // ★ 攒批
```

`CHECK_FIELD_WARN` 使字段截断（如 VARCHAR 变短）**只告警不报错**（ALTER 既有语义）；`ha_start_bulk_insert(records)` 把预估行数传给引擎做预分配与攒批。

**③ 免 undo + 免锁**：`to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY)` → `skip_alter_undo=1`（剖析见 server/handler.md）。

#### 第二段：拷贝循环（18621-18717）—— 核心

```cpp
while (!(error = iterator->Read())) {          // 逐行读源表（走标准执行器迭代器）
  if (thd->killed) { thd->send_kill_message(); error = 1; break; }
```

注意这个迭代器由 `CreateIteratorFromAccessPath` 建于 18606 行——即**透过整个 SQL 层**的全表扫描。

**步骤 1：字段拷贝（类型转换点）**

```cpp
for (Copy_field *copy_ptr = copy; copy_ptr != copy_end; copy_ptr++) {
  copy_ptr->invoke_do_copy();
}
```

`Copy_field` 数组按字段数分配（18475）。`invoke_do_copy()` 做源→目标字段拷贝；若类型变了（INT→BIGINT、字符集变更），**这里就是实际的类型转换点**，逐字段逐行。

**步骤 2：自增值处理**

```cpp
if (to->next_number_field) {
  if (auto_increment_field_copied)
    to->autoinc_field_has_explicit_non_null_value = true;   // 源表有值→保留
  else
    to->next_number_field->reset();                          // 否则引擎重新生成
}
```

保证 ALTER 后自增列从正确位置继续递增。

**步骤 3：生成列 / 生成默认值求值**

```cpp
for (ptr = gen_fields; ptr != gen_fields_end; ptr++) {
  Item *expr_item;
  if ((*ptr)->is_gcol()) {
    expr_item = (*ptr)->gcol_info->expr_item;                 // 生成列
  } else {
    expr_item = (*ptr)->m_default_val_expr->expr_item;          // 生成默认值
  }
  expr_item->save_in_field(*ptr, false);                        // ★ 逐行求值
}
```

**生成列的值不是从源表拷的，而是逐行重新求值**。注释（18658-18668）说明两条约束：必须在"拷旧列 + 填新列默认值"**之后**（生成值可能依赖它们）；必须**按表中列顺序**处理（生成列可依赖前面的生成列，不允许前向引用）。

**步骤 4：约束检查 + 写入引擎**

```cpp
error = invoke_table_check_constraints(thd, to);    // CHECK 约束
if (error) break;
error = to->file->ha_write_row(to->record[0]);      // ★ 真正写入引擎
```

`ha_write_row` → `ha_innobase::write_row` → `row_insert`，此时 `skip_alter_undo` 生效（不写 undo、不加锁）。

**错误处理**区分两类（18691-18710）：

```cpp
if (!to->file->is_ignorable_error(error)) {
  to->file->print_error(error, MYF(0));               // 非重复键→直接报错
} else {
  uint key_nr = to->file->get_dup_key(error);          // 重复键→定位索引
  if (key_nr == 0 && to->key_info[0].key_part[0].field->is_flag_set(AUTO_INCREMENT_FLAG))
    err_msg = ER_THD(thd, ER_DUP_ENTRY_AUTOINCREMENT_CASE);
  print_keydup_error(to, ..., err_msg, MYF(0), from->s->table_name.str);
}
```

注意错误消息用的是 **`from->s->table_name`（源表名）**——对用户而言他操作的是原表。

**进度上报**（18711-18716）：

```cpp
found_count++;
mysql_stage_set_work_completed(psi, found_count);            // ★ PFS 进度
thd->get_stmt_da()->inc_current_row_for_condition();         // 行号（错误定位用）
```

这就是 `ALTER TABLE` 在 `performance_schema.events_stages_current` 里能看到 `WORK_COMPLETED / WORK_ESTIMATED` 的来源，估计值来自 `from->file->info(HA_STATUS_VARIABLE)`。

#### 第三段：收尾（18718-18745）

```cpp
if (to->file->ha_end_bulk_insert() && error <= 0) { ... }    // ⑤ 结束批量插入
to->file->ha_extra(HA_EXTRA_END_ALTER_COPY);                 // ⑥ 复位 skip_alter_undo + 重算统计
if ((!(to->file->ht->flags & HTON_SUPPORTS_ATOMIC_DDL) || from->s->tmp_table) &&
    mysql_trans_commit_alter_copy_data(thd)) error = 1;       // ⑦ 与①对称
err:
  ...
  if (to->file->ha_external_lock(thd, F_UNLCK)) error = 1;   // ⑧ 与 F_WRLCK 配对
  if (error < 0 && to->file->ha_extra(HA_EXTRA_PREPARE_FOR_RENAME)) error = 1;
```

⑥ 置 `skip_alter_undo=0` 并 `alter_stats_rebuild` 重算统计；⑦ 只有非原子引擎才显式提交；⑧ 解锁后计数归零触发 InnoDB `end_stmt`。

#### 设计权衡：COPY 慢在哪（代码级答案）

循环里**每行要付 4 次 SQL 层开销**：

| 步骤 | 开销 | INPLACE rebuild 是否付 |
|---|---|---|
| ① `invoke_do_copy()` 字段拷贝/类型转换 | 逐字段 | ❌ 引擎内直接按新格式构建 |
| ② 生成列逐行求值 | 逐行表达式求值 | ❌ bulk load 时统一处理 |
| ③ `invoke_table_check_constraints` | 逐行约束检查 | ❌ |
| ④ `ha_write_row` 走 handler 边界 | 逐行跨 SQL→引擎 | ❌ 直接建 B-tree |

**仅有的补偿**：`skip_alter_undo`（免 undo + 免锁）与 `ha_start_bulk_insert`（攒批）——这解释了 COPY 虽慢但仍可用。

**对比 INPLACE rebuild**：后者在 `ddl::Builder` 里扫描聚簇索引后**直接 bulk load 构建 B-tree**，完全不经上述 4 步。这就是"INPLACE 快一个数量级"的代码级依据。

### DROP TABLE 流程

入口 `row_drop_table_for_mysql`（row0mysql.cc:3771），核心步骤：

1. 打开表、加 dict 锁、`table->to_be_dropped = true`（:3874）
2. 移除表上所有锁（`lock_remove_all_on_table` :3974）、停用 stats
3. **写 DDL log**（关键，按表空间类型分三条路）：

```
4028:4127:storage/innobase/row/row0mysql.cc
  if (!table->is_temporary() && !file_per_table) {
    err = log_ddl->write_free_tree_log(trx, index, true);   // 共享表空间：释放索引 btree（每个 index 一条）
  }
  ...
  if (!is_temp) {
    log_ddl->write_drop_log(trx, table_id);                  // 记录 drop（清理 innodb_table_metadata）
  }
  ...
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);  // 独立表空间：删 .ibd
```

4. `index->page = FIL_NULL`（:4049）标记所有索引不可用（逻辑删除）
5. `row_drop_table_from_cache`（:4108）从 dict cache 移除
6. 提交后由 post_ddl 统一 replay

---

### TRUNCATE TABLE：rename + drop + create

TRUNCATE 在 InnoDB 走 `HTON_CAN_RECREATE` 路径（ha_innodb.cc:5174），本质是 drop + recreate，从 `innobase_truncate::truncate()`（ha_innodb.cc:14558）看三步：

```
14564:14623:storage/innobase/handler/ha_innodb.cc
  if (m_file_per_table) {
    error = rename_tablespace();      // ① 旧 .ibd 重命名临时名（避免 create 文件冲突）
  }
  ...
  error = innobase_basic_ddl::delete_impl(...);   // ② 删旧表 → row_drop_table_for_mysql，写 DROP 类 DDL log
  ...
  error = innobase_basic_ddl::create_impl(...);   // ③ 建新空表，写 CREATE 类 DDL log
```

设计要点：

- **先 rename 而非直接删**：create 失败时旧文件（临时名）还在，有恢复余地。rename 本身通过 `RENAME_SPACE_LOG` 类型的 DDL log 记录
- TRUNCATE 的"undo"= DROP 阶段 DDL log + CREATE 阶段 DDL log + 两阶段 dd 元数据 undo，全部在同一个 DDL 事务内，提交后 post_ddl 统一 replay
- `m_trx->in_truncate = true`（:14612）走原子 truncate 日志；`m_dd_table->set_se_private_id(INVALID)`（:14596）使 table_id 重新分配
- TRUNCATE vs DROP+CREATE 区别：TRUNCATE 保留外键定义、可保留 autoinc（`m_keep_autoinc`）、复用同一 DDL 事务

---

### B-tree 的物理释放（btr_free 机制）

DROP/TRUNCATE 时 B-tree 到底怎么删，取决于表空间类型。

### 独立表空间（file-per-table）/ 一般表空间：整文件删除，不逐页释放

`row_drop_table_for_mysql` 只写 `DELETE_SPACE_LOG`（row0mysql.cc:4126），提交后：

```
Log_DDL::replay_delete_space_log (log0ddl.cc:1659)
  → fil_delete_tablespace (fil0fil.cc:4658)
    → Fil_shard::space_delete (:4493)
      → os_file_delete (:4641)    ← 整个 .ibd 删除，B-tree 所有页随文件消失
```

注意 `row0mysql.cc:4028` 的判定 `if (!table->is_temporary() && !file_per_table)`——**独立表空间根本不写 FREE_TREE_LOG，一个页都不会释放**。

### 系统表空间 / 共享表空间：逐页释放 B-tree（FREE_TREE_LOG）

文件被其他表共享不能删，必须真正逐页释放。执行期只写 log：

```
4028:4055:storage/innobase/row/row0mysql.cc
  if (!table->is_temporary() && !file_per_table) {
    err = log_ddl->write_free_tree_log(trx, index, true);  // 每个 index 一条 FREE_TREE_LOG
  }
  ...
  index->page = FIL_NULL;  // dict cache 标记索引不可用
```

`write_free_tree_log`（log0ddl.cc:853）→ `insert_free_tree_log`（920）在 `innodb_ddl_log` 表 insert（space_id / root_page_no / index_id）。提交后 replay：

```
1583:1650:storage/innobase/log/log0ddl.cc
  switch (record.get_type()) {
    case Log_Type::FREE_TREE_LOG:
      replay_free_tree_log(record.get_space_id(), record.get_page_no(), record.get_index_id());
      break;
  ...
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no, ulint index_id) {
  btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
```

`btr_free_if_exists`（btr0btr.cc:1010）分四步：

1. `btr_free_root_check`（809）：读根页，校验是 index 页且 `PAGE_INDEX_ID` == index_id，防止崩溃恢复时重复/错误释放
2. `btr_free_but_not_root`（958）：释放除根页外的整棵树
3. `btr_free_root`（771）：释放根页
4. `btr_free_root_invalidate`（795）：根页 `PAGE_INDEX_ID` 写 0（`BTR_FREED_INDEX_ID`），幂等标记

`btr_free_but_not_root`（958-1003）：B-tree 的 LEAF segment 与 TOP segment 是两个独立文件段，分两轮释放：

- leaf_loop：循环 `fseg_free_step(root+PAGE_BTR_SEG_LEAF, ...)`，每个 mtr 释放叶子段的一个 extent
- top_loop：循环 `fseg_free_step_not_header(root+PAGE_BTR_SEG_TOP, ...)` 释放非叶段（不含头页）

`fseg_free_step`（fsp0fsp.cc:3658）：一个 mini-transaction 内释放 segment 的一个分配单元（1 个完整 extent 或 1 个 frag 页），返回 false 表示未释放完，外层循环继续。核心逻辑：

- 有整 extent：`fseg_free_extent`（3591）把 extent 从 FSEG_FULL / FSEG_NOT_FULL / FSEG_FREE 链摘除 → `fsp_free_extent`（1896）`xdes_init` 复位描述符 → `flst_add_last(FSP_FREE)` 归还表空间空闲区（`space->free_len++`）；frag extent 归还 FSP_FREE_FRAG
- 只有 frag 页：`fseg_free_page_low`（3725）逐页释放
- 全部释放完：`fsp_free_seg_inode`（3720）归还 segment inode 页
- AHI：`fseg_free_extent` 内按页 `btr_search_drop_page_hash_when_freed`（3622）摘自适应哈希索引

`btr_free_root`（771-786）：先 `btr_search_drop_page_hash_index` 摘 AHI，再循环 `fseg_free_step` 释放根页所在段。

**设计思想**：free B-tree 是破坏性物理操作，页被释放后若 DDL 回滚无法恢复。用 DDL log 记录意图、提交后执行，配合元数据 undo 可回滚。崩溃恢复重放 FREE_TREE_LOG，靠 `btr_free_root_invalidate` 的 index_id=0 标记保证幂等。释放粒度是 extent 而非逐页，整 extent 一次性归还 FSP_FREE，XDES 描述符直接复位，开销小。

补充：临时表清空走 `row_delete_all_rows`（row0mysql.cc:2491）——`btr_free`（btr0btr.cc:1026，MTR_LOG_NO_REDO）+ `btr_create` 重建根页，不经过 DDL log。

---

## 并行 DDL（社区版 8.0.27+ 确有）

社区版 8.0.39 **支持**并行 DDL（`innodb_ddl_threads` 于 8.0.27 引入），但并行**仅限 InnoDB 层的部分阶段**，不是全链路并行。

### 三个参数（ha_innodb.cc:1107-1124）

```cpp
innodb_parallel_read_threads  // 并行扫描线程数，默认 4，范围 1~MAX_THREADS(256)
innodb_ddl_buffer_size        // DDL 排序总内存，默认 1048576(1MB)，范围 64KB~4GB-1
innodb_ddl_threads            // DDL 并行线程数，默认 4，范围 1~64
```

每个并行扫描线程分到的排序 buffer = `innodb_ddl_buffer_size / innodb_parallel_read_threads`，所以调大扫描线程数时应同步调大 buffer。

相关第四个参数 **`innodb_disable_sort_file_cache`**（默认 OFF，ha_innodb.cc:22486）：控制 DDL 排序临时文件是否**绕过 OS page cache**。排序临时文件创建后（`ddl::file_create`，ddl0ddl.cc:169），若开启则对临时文件 `fcntl(fd, F_SETFL, O_DIRECT)`（`os_file_set_nocache`，os0file.cc:5286）。默认 OFF 走 OS 缓存（归并排序反复读临时文件能命中、更快），但大排序会把热数据页挤出 OS cache；设 ON 用 O_DIRECT 避免污染，代价是排序变慢，且 tmpfs 不支持 O_DIRECT（会告警后回退）。

### 三阶段与并行粒度（关键差异）

| 阶段 | 创建二级索引（CREATE INDEX / ADD INDEX） | 重建表（ALTER ENGINE / OPTIMIZE） |
|------|----------------------------------------|--------------------------------|
| ① 扫描 | **并行**（`parallel_read_threads`，B+树子树粒度） | **单线程**（主键有序，顺序插入新树即可；该参数对重建表**不生效**） |
| ② 排序 | **并行**（`ddl_threads`，粒度 = **临时文件数 = 扫描线程数**） | **并行**（`ddl_threads`，粒度 = **索引级**，每个二级索引一个临时文件） |
| ③ 构建 B+树 | **单线程** | **并行**（`ddl_threads`，**索引级**，每个索引一个构建任务） |

**核心区别**：建索引是"**单个索引内部**并行"（多临时文件归并），重建表是"**多个索引之间**并行"。

### 源码：Loader 任务状态机

`ddl0loader.cc`：`Loader::build_all`（484）→ `prepare()` → `scan_and_build_indexes`（402）。每个索引是一个 `Builder`，由状态机驱动（`Loader::Task::operator()`，2135）：

```cpp
switch (m_builder->get_state()) {
  case Builder::State::SETUP_SORT:          err = m_builder->setup_sort();           break;
  case Builder::State::SORT:               err = m_builder->merge_sort(m_thread_id); break;
  case Builder::State::BTREE_BUILD:        err = m_builder->btree_build();           break;
  case Builder::State::FTS_SORT_AND_BUILD: err = m_builder->fts_sort_and_build();    break;
  case Builder::State::FINISH:             err = m_builder->finish();                break;
  ...
}
```

状态推进 `set_next_state()`（2108）：`SORT → BTREE_BUILD → FINISH`。work 线程从任务队列取任务，**按 Builder 当前状态**决定做排序还是构建——排序任务完成后自动追加该索引的构建任务（`ddl0loader.cc:302-307` 在扫描后为每个 builder `set_next_state()` + `add_task()`）。

**RTree（空间索引）是特例**：`ddl0loader.cc:303` 注释 "RTrees are built during the scan phase, using row by row insert" —— 不走 merge sort，扫描阶段逐行插入。

### 并行扫描如何分片

`Parallel_reader`（`row0pread.cc`）把 B+树按**子树**切成多个分片，每个 work 线程领一个：

1. 用户线程按 **ROOT PAGE 的子树数量预分片**（全程持 INDEX + ROOT 的 S 锁，确保不新增子树）；
2. 分片数对线程数取余的"余量分片"标记为需再分，由 work 线程**二次分片**——此时 INDEX/ROOT 锁已释放、子树结构可能变化，所以按 **RANGE**（而非子树数量）划分；
3. 每个 work 持**一个临时文件 + 一个排序 buffer**；buffer 满则内存排序后写文件 → 得到 N 个"局部有序"文件；
4. 第二阶段对这 N 个文件并行归并排序。

### Parallel_reader 的深层设计（row0pread.h）

`row0pread.h:76-102` 的类注释完整交代了设计意图，值得逐条读：

**三级执行上下文**：一个 `Parallel_reader` 可同时扫多个索引（分区表就是多个索引），每个 `add_scan(trx, config, f)`（312）注册一个 `Scan_ctx`；`Scan_ctx` 又包含多个 `Ctx`，最终被线程执行的是 `Ctx`（一个 Ctx = 一个分片范围）：

```
Parallel_reader                 // 一次并行读的总调度
  ├─ Scan_ctx[]                 // 每个 add_scan 一个（一个索引/分区一个）
  │    └─ Ctx[]                 // 该扫描切出的分片，进 run queue 被线程领取
  └─ m_parallel_read_threads[]  // 工作线程池（IB_thread）
```

**动态分片（work stealing 思想）**：类注释原文（85-91 行）——"If you have 5 sub-trees to scan and 4 threads then it will tag the 5th sub-tree as 'to_be_split'... the first thread that finishes scanning the first set of 4 partitions will then dynamically split the 5th sub-tree and add the newly created sub-trees to the execution context (Ctx) run queue"。即**余量分片由最先空出来的线程动态切分**，而不是等所有线程都扫完——这是解决"分片数 ≠ 线程数"负载不均的关键。

**回调机制**：三个 `std::function`（row0pread.h:139-145）——`Start`（线程启动时初始化局部状态，如分配排序 buffer）、`F`（每扫一行调一次，COUNT* 是 `counter++`、建索引是构建索引 tuple）、`Finish`（线程收尾，flush 剩余 buffer 到临时文件）。`Thread_ctx` 持有线程局部状态（`m_callback_ctx` + `m_blob_heap`）。

**全局线程上限**：`MAX_THREADS = 256`（106）是**所有连接之和**（`s_active_threads` 静态原子计数，`release_threads` 递减），达到后新连接回退单线程扫描。所以 256 不是单条 SQL 的上限，是实例级并行读总预算。

**明确限制**：类注释 102 行 "Secondary index scans are not supported currently"——**并行扫描只支持聚簇索引**，二级索引扫描回退串行。

### 使用建议

- **建索引**：`innodb_parallel_read_threads` 与 `innodb_ddl_threads` **设为相同值**（排序任务数由扫描线程数决定，设更大无效）；
- **重建表**：只调 `innodb_ddl_threads`（按二级索引数量设置），`innodb_parallel_read_threads` 不生效；
- 加大扫描线程时同步加大 `innodb_ddl_buffer_size`。

---

## Instant DDL 深度剖析

### 核心思想：只改元数据 + 行版本号

Instant ADD/DROP COLUMN 的精髓——**已有行一个字节都不改**，靠"行版本号"让不同时代的行共存。

- **表级**：`dict_table_t` 维护 `current_row_version`（当前版本）、`initial_col_count`（建表时列数）、`current_col_count`、`total_col_count`（`rem0wrec.cc` 有打印输出）；
- **行级**：记录头多 **1 字节行版本号**，并置 `REC_IS_VERSIONED` 信息位（`rem0rec.cc:804-811`）：

```cpp
if (is_store_version(index, n_fields)) {
  rec_set_instant_row_version_new(rec, index->table->current_row_version);  // 存 1 字节版本号
  nulls -= 1;                                        // 版本号占一字节（位于 nulls 之前）
  rec_instant_info = Rec_instant_state::REC_IS_VERSIONED;
}
```

- 每次 instant add/drop → `m_dict_table->current_row_version++`（`dict0inst.cc:254`）；
- **新插入的行**才带新版本号（含新列）；**老行保持老版本号**，读取时按该行版本号决定"它实际有哪些列"，缺失的列用 DD 中存的默认值（`DD_INSTANT_COLUMN_DEFAULT`）填充。

**这就是"秒级"的根本原因**：不搬数据、不改已有行，只改 DD 元数据 + 递增版本号。

### 三个 version，别混淆

提到 instant 的 "version"，其实涉及**三个不同层级**，极易混淆：

| 层级 | 字段 | 类型 | 何时变 | 作用 |
|---|---|---|---|---|
| 表定义 | `dict_table_t::version` | uint64_t | **任何 DDL** | 表定义元数据版本 |
| 行时代 | `dict_table_t::current_row_version` | uint32_t | **仅 instant add/drop** | 行的"时代"计数 |
| 物理行 | 记录头 row_version | uint8_t | 行 INSERT 时定格 | 该行属于哪个时代 |

**`version`（表定义元数据版本，dict0mem.h:2151）**：注释原文 "metadata version number of dd::Table::se_private_data()"。它是**整张表定义**（所有列/索引/选项）的版本号，**任何 DDL（不只 instant）都会 +1**。用途是检测表定义是否被并发修改——`dict0dict.cc:4024`：

```cpp
/* If the metadata version is bigger than the one in table, it could be that
   an ALTER TABLE has been rolled back, so metadata in new version should be ignored. */
if (table->version != metadata->get_version()) {
  return get_dirty;
}
```

每次 open 表时 `ha_innodb.cc:7261` 从 DD 取最新值：`ib_table->version = dd_get_version(table_def)`。

**`current_row_version`（行版本号，dict0mem.h:2154）**：注释原文 "Current row version in case columns are added/dropped INSTANTly"。**只有 instant add/drop column 才递增**（`dict0inst.cc:254`）。它是"行时代"的计数——每发生一次 instant 结构变更，表进入新"时代"。`has_row_versions()` 的定义就是 `current_row_version > 0`（dict0mem.h:2499）。

**行记录头 row_version（1 字节）**：物理行里存的值 = **该行 INSERT 时 `current_row_version` 的快照**。老行定格在旧时代，新行用新时代。

**三者关系**：

```
dict_table_t::version              （表定义版本：任何 DDL +1）
dict_table_t::current_row_version  （行版本：仅 instant add/drop +1）
        │ 新行 INSERT 时快照
        ▼
物理行.row_version = 当时的 current_row_version
```

**列级还有 `v_added` / `v_dropped`**：每个 `dict_col_t` 记录自己"在哪一版被 instant 加进来 / 删掉"。读取一行时，用该行 `row_version` 与列的 `v_added`/`v_dropped` 比较，判断这列在这行里是否存在（`v_added ≤ row_version < v_dropped` 才存在），不存在的列用默认值填充。`create_nullables`（dict0mem.cc:657-669）正是遍历所有列、按每个版本的 v_added/v_dropped 累加出 `nullables[version]` 数组。

> 回答"也涉及 dict 的 version 吧"：是的，但要区分——`dict_table_t::version` 是**表定义元数据版本**（任何 DDL 都变），`current_row_version` 才是**行版本**（仅 instant 变），两者是独立字段。

### 两种记录格式：V1（n_fields）→ V2（row_version）

Instant 的记录头有两种编码，对应两个版本时代：

| | **V1**（8.0.12-8.0.28） | **V2**（8.0.29+，当前） |
|---|---|---|
| 记录头存什么 | **字段数 n_fields**（`rec_set_n_fields`，rem0rec.cc:816） | **1 字节 row_version**（`rec_set_instant_row_version_new`，806） |
| 状态枚举 | `Rec_instant_state::REC_IS_INSTANT` | `Rec_instant_state::REC_IS_VERSIONED` |
| 支撑的 DDL | 仅 ADD COLUMN | ADD + DROP COLUMN |
| 判定依据 | `has_instant_cols()` | `has_row_versions()` |

### 字节级布局（COMPACT / DYNAMIC 行格式）

记录头固定 5 字节（`REC_N_NEW_EXTRA_BYTES = 5`，rec.h:133），`rec` 指针指向记录头起点。instant 字段插在 **NULL 位图与记录头之间**，紧贴记录头（`rec - 6`，rem0rec.ic:898/926）：

```
① 普通记录（无 instant）
低地址 ───────────────────────────────────────────────────────▶ 高地址
┌──────────────────┬──────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ 记录头 (5B)       │ 列数据       │
└──────────────────┴──────────┴─────────────────┴────────────┘
                                ↑ rec            ↑ rec+5

② V1（instant ADD COLUMN，8.0.12–8.0.28）：记录头前加 n_fields（1–2 字节）
┌──────────────────┬──────────┬─────────────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ n_fields (1–2B)  │ 记录头 (5B)       │ 列数据       │
└──────────────────┴──────────┴─────────────────┴─────────────────┴────────────┘
                                ↑ rec-6 / rec-7   ↑ rec
  n_fields ≤ 127（REC_N_FIELDS_ONE_BYTE_MAX=0x7F）：1 字节
  n_fields > 127：2 字节，高字节最高位 0x80 作"两字节"标志（REC_N_FIELDS_TWO_BYTES_FLAG）

③ V2（instant ADD/DROP COLUMN，8.0.29+）：记录头前加 row_version（固定 1 字节）
┌──────────────────┬──────────┬──────────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ row_version  │ 记录头 (5B)       │ 列数据       │
│                  │          │    (1B)      │                 │            │
└──────────────────┴──────────┴──────────────┴─────────────────┴────────────┘
                                ↑ rec-6       ↑ rec
```

三态靠记录头 info bits 里的**两个独立标志位**区分（rem0rec.ic:435-448）：

```
                    INSTANT_FLAG   VERSION_FLAG
普通（REC_IS_SIMPLE）      0              0
V1（REC_IS_INSTANT）       1              0
V2（REC_IS_VERSIONED）     0              1
```

源码印证（rem0rec.ic:925 有用户笔记"跟 set_n_fields 都是同一位置：header 前 1byte"）：

```cpp
byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);   // = rec - 6，两种格式写同一位置
```

`rec_set_instant_row_version_new` 固定写 1 字节（`*ptr = row_version`）；`rec_set_n_fields`（rem0rec.ic:897）按 `n_fields ≤ 0x7F` 决定写 1 字节还是 2 字节，2 字节时低字节在前、高字节最高位置 `REC_N_FIELDS_TWO_BYTES_FLAG`。

### 核心函数逐行解析

**写 V1 的 `rec_set_n_fields`**（rem0rec.ic:897）：

```cpp
static inline uint8_t rec_set_n_fields(rec_t *rec, ulint n_fields) {
  byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);   // ① 定位：记录头(5B)前 1 字节 = rec-6
  ut_ad(n_fields < REC_MAX_N_FIELDS);               // ② 断言：字段数 < 1023（REC_MAX_N_FIELDS=1024-1）

  if (n_fields <= REC_N_FIELDS_ONE_BYTE_MAX) {      // ③ 单字节路径：≤127（0x7F）
    *ptr = static_cast<byte>(n_fields);             //    直接写，最高位恒 0，天然区分
    return (1);                                     //    报告占用 1 字节
  }

  --ptr;                                            // ④ 双字节路径：>127，指针再前移
  *ptr++ = static_cast<byte>(n_fields & 0xFF);      //    先写低字节（小端）
  *ptr   = static_cast<byte>(n_fields >> 8);        //    再写高字节（暂不含标志位）
  ut_ad((*ptr & 0x80) == 0);                        // ⑤ 断言高字节最高位空闲（≤1023 保证）
  *ptr |= REC_N_FIELDS_TWO_BYTES_FLAG;              // ⑥ 最高位置 0x80 = "这是两字节"标志
  return (2);                                       //    报告占用 2 字节
}
```

逐行要点：① `rec-6` 是记录头前紧贴的 1 字节，V1/V2 共用这个位置；③ 单字节靠"最高位为 0"区分；④-⑥ 双字节时**低字节在前**（小端），高字节最高位 `0x80` 作"两字节"标志供读取识别。

**写 V2 的 `rec_set_instant_row_version_new`**（rem0rec.ic:920）：

```cpp
static inline void rec_set_instant_row_version_new(rec_t *rec, uint8_t row_version) {
  ut_ad(is_valid_row_version(row_version));          // ① 断言版本号合法
  byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);     // ② 与 n_fields 同一位置 rec-6
  *ptr = static_cast<byte>(row_version);             // ③ 固定 1 字节，直接写
}
```

对比可见 V2 比 V1 简单得多：**固定 1 字节，无变长判断**——因为 row_version 上限远小于 256，单字节够用。

**读取侧的 `rec_get_n_fields_instant`**（rem0rec.ic:708）：

```cpp
ptr = rec - (extra_bytes + 1);                        // ① 同一位置 rec-6
if ((*ptr & REC_N_FIELDS_TWO_BYTES_FLAG) == 0) {      // ② 最高位 0 → 单字节
  *length = 1;
  return (*ptr);                                      //    直接返回字段数
}
*length = 2;                                          // ③ 最高位 1 → 两字节
n_fields = ((*ptr-- & REC_N_FIELDS_ONE_BYTE_MAX) << 8);  // ④ 高字节去掉 0x80 标志
n_fields |= *ptr;                                     // ⑤ 拼上低字节
```

写入和读取互为镜像：写时"低字节前 + 高字节最高位做标志"，读时"最高位判单双 + 去标志 + 拼低字节"。

### REDUNDANT（旧格式）呢？

上面画的都是 **COMPACT / DYNAMIC（NEW-style 新格式）**。REDUNDANT（OLD-style 旧格式）同样支持 instant version，但布局不同：

- 记录头 `REC_N_OLD_EXTRA_BYTES = 6`（rec.h:130），比新格式多 1 字节；
- version 写在 `rec - (6+1) = rec - 7`（`rec_set_instant_row_version_old`，rem0rec.ic:948）；
- **字段寻址是"偏移数组"而非"变长长度列表"**：读取字段 N 时，若该行 versioned 需额外跳过 1 字节（rem0rec.ic:630-635）：

```cpp
uint32_t version_length = 0;
if (rec_old_is_versioned(rec)) {
  version_length = 1;                                  // version 占 1 字节
}
return mach_read_from_1(rec - (REC_N_OLD_EXTRA_BYTES + version_length + n));  // rec-7-n
```

布局（REDUNDANT + version）：

```
低地址 ──────────────────────────────────────────────────────▶ 高地址
┌───────────────┬──────────────┬─────────────────┬────────────┐
│ 字段偏移数组    │ row_version  │ 记录头 (6B)       │ 列数据       │
│  (变长)       │   (1B)       │                 │            │
└───────────────┴──────────────┴─────────────────┴────────────┘
                                ↑ rec            ↑ rec+6
```

写入端在 `rec_convert_dtuple_to_rec_old`（rem0rec.cc:606-613）：

```cpp
if (store_version) {
  rec_set_instant_row_version_old(rec, index->table->current_row_version);  // 写 rec-7
  rec_old_set_versioned(rec, true);   // 旧格式的 versioned 标志位
} else {
  rec_old_set_versioned(rec, false);
}
```

**四种行格式小结**：

| 格式 | 记录头大小 | version 位置 | 字段寻址 |
|---|---|---|---|
| REDUNDANT（OLD） | 6B | rec-7 | 偏移数组 |
| COMPACT（NEW） | 5B | rec-6 | 变长长度列表 |
| DYNAMIC（NEW） | 5B | rec-6 | 变长长度列表（+ off-page 外部存储） |
| TEMP（临时/row log） | 1B | rec-2 | 省略记录头 |

TEMP 格式（`REC_N_TMP_EXTRA_BYTES=1`）用于 row log / sort buffer，只保留 info bits 1 字节，instant n_fields 存在 `rec - 2`（`rec_get_n_fields_instant(rec, REC_N_TMP_EXTRA_BYTES, ...)`，rem0rec.ic:956）。

### 三种记录状态（`rec_convert_dtuple_to_rec_comp` 返回，rem0rec.cc:763；落到记录头在 1078）：

```cpp
enum class Rec_instant_state { REC_IS_SIMPLE, REC_IS_VERSIONED, REC_IS_INSTANT };

switch (rec_state) {
  case Rec_instant_state::REC_IS_SIMPLE:    rec_new_reset_instant_version(rec); break;  // 无 instant
  case Rec_instant_state::REC_IS_VERSIONED: rec_new_set_versioned(rec);        break;  // V2：存版本号
  case Rec_instant_state::REC_IS_INSTANT:   rec_new_set_instant(rec);          break;  // V1：存字段数
}
```

**为什么从 V1 演进到 V2**：V1 存"本行实际字段数"，只够支撑"末尾加列"（新列不落老行，老行字段数少）。要支持 **DROP COLUMN**（被删的列还要保留在老行里）和更复杂的版本语义，字段数就不够表达了，于是 8.0.29 引入 **row_version**——版本号能承载"这行是哪个时代插入的"，据此推导出该行完整列集合。

`is_store_version()`（rem0rec.cc:246）决定新写入用哪种格式：

```cpp
bool is_store_version(const dict_index_t *index, size_t n_tuple_fields) {
  if (!index->has_instant_cols_or_row_versions()) return false;
  return index->table->has_row_versions() || index->table->is_upgraded_instant();
}
```

`is_upgraded_instant()` = 从 8.0.28 及更早升级上来的表（DD 里没有版本信息），仍是 V1 格式，读取时按旧语义解析。

**读取侧按版本解析**：`rec_init_null_and_len_comp`（rem0rec.cc:1317）从记录头解析出 `row_version`（初始 `UINT8_UNDEFINED`），`get_nullable_fields_for_rec`（267）用它取该版本的 nullable 数——`index->get_nullable_in_version(rec_version)`，这就是 `create_nullables` 按版本维护的数组的用途。

### 实现入口与分支

`dict0inst.cc`（文件头注释 "Instant DDL interface implementation"），`Instant_ddl_impl<Table>` 同时实例化 `dd::Table` 与 `dd::Partition`（分区表也支持，但 `dd_part_is_first` 只让第一个分区做元数据变更）。

`commit_instant_ddl()`（194）按 `Instant_Type` 分支：

| Instant_Type | 处理 |
|---|---|
| `INSTANT_NO_CHANGE` | `dd_commit_inplace_no_change(false)` |
| `INSTANT_COLUMN_RENAME` | 同上 + `innobase_discard_table`（重载表定义） |
| `INSTANT_VIRTUAL_ONLY` | 处理 FTS_DOC_ID 隐藏列 + discard table |
| `INSTANT_ADD_DROP_COLUMN` | `dd_copy_private` → drop/add → **`current_row_version++`** |
| `INSTANT_IMPOSSIBLE` | `ut_ad(0)` |

`INSTANT_ADD_DROP_COLUMN` 关键步骤（234-267）：

```cpp
trx_start_if_not_started(m_trx, true, UT_LOCATION_HERE);
dd_copy_private(*m_new_dd_tab, *m_old_dd_tab);
populate_to_be_instant_columns();                        // 算出要加/删哪些列
if (!m_cols_to_drop.empty()) commit_instant_drop_col();  // 先 DROP
if (!m_cols_to_add.empty())  commit_instant_add_col();   // 再 ADD
m_dict_table->current_row_version++;                     // ★ 版本递增
ut_ad(dd_table_has_instant_cols(m_new_dd_tab->table()));
```

### 列元数据存在哪（DD se_private_data）

每列的 `se_private_data`（`dict0dd.cc:2212-2363`）记录：

| key | 含义 |
|---|---|
| `DD_INSTANT_VERSION_ADDED` | = `current_row_version + 1`，该列从哪个版本开始存在 |
| `DD_INSTANT_VERSION_DROPPED` | = `current_row_version + 1`，该列从哪个版本起被删 |
| `DD_INSTANT_PHYSICAL_POS` | 物理位置 |
| `DD_INSTANT_COLUMN_DEFAULT` / `_DEFAULT_NULL` | instant 加列时**老行的默认值** |

**删除列不是真删**：改名成 SE_HIDDEN 列保留（`set_dropped_column_name`，dict0dd.cc:2190），保证老版本行仍能被正确解析。

### 限制（源码依据）

1. **表级**：`dict_table_t::support_instant_add_drop()`（dict0dict.ic:1371）——非压缩表、非 DD 表空间、无 FTS_DOC_ID、非临时表、无全文索引、非系统表；
2. **行大小**：`is_instant_add_drop_possible`（dict0inst.cc:42）若加列后可能超出 `page_rec_max`，直接不允许（注释 "Table is already in a state where possible row size can go beyond permissible size limit"）；
3. **只能加到末尾**：Instant 语义下新列不落老行，只能追加（表无主键时末尾隐式主键会导致失败）；
4. 升级线程不做 instant（`current_thd->is_server_upgrade_thread()`）；
5. 列被外键引用时不允许 instant rename。

### row version 的配套内存结构

`dict_index_t::create_nullables(current_row_version)`（dict0mem.cc:622）为**每个 row version** 维护独立的 nullable 计数（`nullables[MAX_ROW_VERSION + 1]`）——因为不同版本的行，列的可空性不同。

### 如何用 SQL 查询 instant 状态

**判断一张表是否做过 instant DDL**：查 `information_schema.INNODB_TABLES` 的 `TOTAL_ROW_VERSIONS`（它就是 `current_row_version`，填充源码 i_s.cc:5379-5383）：

```sql
SELECT NAME, N_COLS, INSTANT_COLS, TOTAL_ROW_VERSIONS
FROM   information_schema.INNODB_TABLES
WHERE  NAME = 'your_db/your_table';
```

- **`TOTAL_ROW_VERSIONS > 0`** → 做过 instant ADD/DROP（值 = 累计次数，每次 instant add/drop +1，`OPTIMIZE TABLE`/重建表后重置为 0）；
- **`INSTANT_COLS`** → instant 列数，但**有坑**：只有从 8.0.28 及更早升级上来的表（V1 格式，`is_upgraded_instant()`）才有值，8.0.29+ 的 V2 表恒为 0（源码 `table->is_upgraded_instant() ? table->get_instant_cols() : 0`）。**所以判断"做过没有"要看 `TOTAL_ROW_VERSIONS`，不要看 `INSTANT_COLS`**。

字段定义在 `i_s.cc:5285-5298`（源码里有笔记 "追踪当前表执行 instant ddl 的次数，每次重建表重置为 0"）。

**看哪些列是 instant 加的**：查 `INNODB_COLUMNS` 的 `HAS_DEFAULT` / `DEFAULT_VALUE`（填充源码 i_s.cc:6234-6241）：

```sql
SELECT t.NAME, c.NAME AS col, c.HAS_DEFAULT, c.DEFAULT_VALUE
FROM   information_schema.INNODB_TABLES  t
JOIN   information_schema.INNODB_COLUMNS c ON c.TABLE_ID = t.TABLE_ID
WHERE  t.NAME = 'your_db/your_table'
  AND  c.HAS_DEFAULT = 1;      -- 有 instant 默认值 → instant ADD 的列
```

`HAS_DEFAULT=1` 表示该列带 instant 默认值（`column->instant_default != nullptr`），即 instant ADD 的列；instant DROP 的列默认**不显示**（源码里只有 `DBUG show_dropped_column` 调试开关才露出，i_s.cc:6324-6328）。

---

## 待优化点

对比云厂商（PolarDB / AliSQL）的增强，社区版 8.0.39 在 DDL 与并行方面仍有明显可优化空间：

| 优化点 | 社区版现状 | 云厂商增强 | 价值 |
|--------|-----------|-----------|------|
| **分区级 MDL** | ❌ DDL 锁整张表（**所有分区**） | ✅ PolarDB `loose_partition_level_mdl_enabled`：DDL 只锁涉及分区，不影响其他分区的 DML | 大分区表 DDL 可用性（**投入产出比最高**） |
| **并行构建 B+树** | ❌ 建索引第③阶段**单线程** | 部分优化 | 大表建索引第三阶段瓶颈 |
| **重建表扫描并行** | ❌ 单线程（主键有序，设计如此） | — | 重建表扫描阶段无法加速 |
| **并行扫描二级索引** | ❌ 仅支持聚簇索引 | ✅ AliSQL 支持 | 二级索引扫描场景 |
| **CHECK TABLE 并行** | ⚠️ 仅第二次扫描并行，第一次（结构校验）单线程 | — | 大表 CHECK 耗时 |
| **非阻塞 DDL** | ❌ 无 | ✅ PolarDB 非阻塞 DDL | 彻底避免 MDL 等待堆积 |
| **并行 DDL 加速比** | 有机制（8.0.27+） | 调优到 15-20 倍 | 大表 DDL 耗时 |

**优先级建议**：若目标是提升大表 DDL 可用性，**分区级 MDL** 投入产出比最高——它直接消除"只改一个分区却锁住全表"的痛点，且不依赖并行度调优，也不受"重建表扫描必须串行"这类架构约束限制。

---

## 核心调用栈

```
TRUNCATE TABLE（server 层）
  Sql_cmd_truncate_table::execute (sql_truncate.cc:745)
  → ha_innobase::truncate_impl (ha_innodb.cc:15261)
  → innobase_truncate<dd::Table>::exec (ha_innodb.cc:14748)
  → innobase_truncate<Table>::truncate (ha_innodb.cc:14558)
    → rename_tablespace (14653)：fil_rename_tablespace → RENAME_SPACE_LOG
    → innobase_basic_ddl::delete_impl (14226)
      → row_drop_table_for_mysql (row0mysql.cc:3771)
        → write_free_tree_log (4032, 仅共享表空间) → insert_free_tree_log (log0ddl.cc:920)
        → write_drop_log (4115)
        → write_delete_space_log (4126, 仅独立表空间)
    → innobase_basic_ddl::create_impl (14109)：dict_create_index_tree_in_mem → btr_create (btr0btr.cc:832)

提交后 post_ddl：
  Log_DDL::post_ddl (log0ddl.cc:1905) → replay_by_thread_id (1930)
  → replay (1576) 按类型分发
    → FREE_TREE_LOG: replay_free_tree_log (1625)
        → btr_free_if_exists (btr0btr.cc:1010)
          → btr_free_root_check (809) → btr_free_but_not_root (958) → btr_free_root (771) → btr_free_root_invalidate (795)
            → fseg_free_step (fsp0fsp.cc:3658)
              → fseg_free_extent (3591) → fsp_free_extent (1896) → FSP_FREE 列表
    → DELETE_SPACE_LOG: replay_delete_space_log (1659)
        → fil_delete_tablespace (fil0fil.cc:4658) → Fil_shard::space_delete (4493) → os_file_delete (4641)

崩溃恢复：
  Log_DDL::recover (log0ddl.cc:1942) → replay_all (1952) → replay (1576) → 同上
```

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_print_ddl_logs` | OFF | Global | 把 DDL log 的 insert/delete/replay 全流程打印到错误日志（`srv_print_ddl_logs`，ha_innodb.cc:23090），调试原子 DDL 崩溃问题 |

### 状态变量

无专门状态变量。

---

## Misc

### 原子 DDL 与 binlog 的关系

server 层通过 `binlog_query`/GTID 保证 DDL 在 binlog 中有记录。崩溃恢复时：DDL 事务已写 binlog（已提交）→ DDL log 记录存在 → replay 完成物理操作；未写 binlog（未提交）→ undo 回滚。binlog 提交与否是"该不该 replay"的判据。

### 原子 DDL 与 InnoDB 其他机制的关系

- clone/备份：`fil_truncate_tablespace`（fil0fil.cc:4738）用于 undo 表空间 truncate（srv0tmp.cc:143），与本机制无关
- SDI 表：删除表空间前需对 SDI 表加 MDL（`dd_sdi_acquire_exclusive_mdl`），防止 purge 并发操作 SDI（row0mysql.cc:3854）

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB and Online DDL*（各操作支持的 ALGORITHM / LOCK 矩阵）
- *MySQL 8.0 Reference Manual → Atomic Data Definition Statement Support*
- WorkLog: [WL#7977](https://dev.mysql.com/worklog/task/?id=7977) Atomic DDL

**并行 / 深度技术分析**
- [一文读懂 MySQL 并行查询 & DDL](http://www.uml.org.cn/sjjm/202409294.asp?artid=26485)（阿里技术，2024-09，8.0.37 基准）— ★ 并行扫描（B+树子树分片、预分片+二次分片）、**并行建索引 vs 并行重建表的三阶段粒度差异**、参数配比建议、"并行查询 & DDL 仅限 InnoDB 层"

**内核月报 / 技术文章**（阿里云 RDS 数据库内核组）
- [云原生数据库PolarDB MySQL 8.0.2 DDL介绍](http://mysql.taobao.org/monthly/2023/09/03/) — ★ **四维评估框架**（是否锁表 / 重建表 / 秒级 / 并行）、VARCHAR 256 边界陷阱、Instant 加列限制、"无锁变更"工具切表也要 MDL-X。⚠️ 该文讲 PolarDB：**并行 DDL 非 PolarDB 独有**（社区版 8.0.27+ 有 `innodb_ddl_threads`），**分区级 MDL 是 PolarDB 私有**，社区版无
- [MySQL · 源码阅读 · 白话Online DDL](http://mysql.taobao.org/monthly/2021/03/06/) — ★ COPY/INPLACE/INSTANT 对比、**MDL 三段式（X→S→X）**、"INPLACE 不一定 Online"、"INPLACE 也需要额外空间"
- [MySQL · 源码分析 · Row log分析](http://mysql.taobao.org/monthly/2022/03/02/) — ★ row log 四段格式、三种操作各记什么、**temp 格式**、instant add column 对 row log 的扩展
- [MySQL · 特性分析 · instant add column功能解析](http://mysql.taobao.org/monthly/2020/03/01/) — ★ Instant 的记录头格式（**rem0rec.cc:744 源码注释里直接引用这篇**）、V1 时代的 n_fields 存储、读取时按字段数解析
- [MySQL · 源码分析 · 原子DDL的实现过程](http://mysql.taobao.org/monthly/2018/03/02/) — 原子 DDL 背景、8.0 DD 重构、`Dictionary_client` / `Shared_dictionary_cache` / `Storage_adapter`
- [MySQL · 源码分析 · 8.0 原子DDL的实现过程续](http://mysql.taobao.org/monthly/2018/07/)
- [MySQL · 源码分析 · DDL log与原子DDL的实现](http://mysql.taobao.org/monthly/2021/07/)
- [MySQL · 引擎特性 · 8.0 · DDL的那些事](http://mysql.taobao.org/monthly/2020/05/05/)
- [MySQL · 引擎特性 · 8.0 Instant Add Column功能解析](http://mysql.taobao.org/monthly/2020/03/01/)
- [PolarDB ·引擎特性· DDL中MDL锁的优化和演进](http://mysql.taobao.org/monthly/2023/05/)

**相关文档**
- 上游：server 层 DDL 入口与算法选择 `mysql_alter_table`（sql/sql_table.cc）；server↔引擎接口见 [`../server/handler.md`](../server/handler.md)
- 下游：B-tree 释放的 extent 粒度与 FSP_FREE 见本文件「B-tree 的物理释放」；表空间碎片度量见 [`tablespace.md`](tablespace.md)
- undo：DDL 为何不记数据 undo 见本文件；undo 本身见 [`undo_log.md`](undo_log.md)

