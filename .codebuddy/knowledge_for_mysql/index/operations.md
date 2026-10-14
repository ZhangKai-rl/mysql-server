# 索引的生命周期与可观测：DDL 与运维

> 基于 MySQL 8.0.39 源码。本篇讲 **索引的"生老病死"**（DDL 生命周期）与**怎么观测它**（`SHOW INDEX` / `I_S` / PFS / sys schema）。
>
> **边界**：批量建页的 `Btree_load` 算法细节见 [`btr.md`](btr.md)；online DDL 的通用框架（row log、DDL log）见 [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md)；统计数值本身怎么算见 [`stats.md`](stats.md)；索引类型见 [`types.md`](types.md)。本篇聚焦**索引对象的生命周期操作**与**运维观测入口**。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 索引的创建
    - [从 `ALTER TABLE` 到 `ddl::Loader` 的完整链路](#从-alter-table-到-ddlloader-的完整链路)
    - [三种算法：COPY / INPLACE / INSTANT 与索引](#三种算法copy--inplace--instant-与索引)
    - [在线建索引的 row log](#在线建索引的-row-log)
    - [并行构建与临时文件](#并行构建与临时文件)
  - 索引的销毁与变更
    - [DROP INDEX：三段式（标记 → 日志 → post_ddl 释放）](#drop-index三段式标记--日志--post_ddl-释放)
    - [RENAME / 可见性 / DISABLE KEYS / 主键增删](#rename--可见性--disable-keys--主键增删)
  - 索引的重建
    - [OPTIMIZE TABLE 的真相](#optimize-table-的真相)
  - 可观测
    - [`SHOW INDEX`：Cardinality 与缓存](#show-indexcardinality-与缓存)
    - [`I_S.STATISTICS` 列与语义](#i_sstatistics-列与语义)
    - [InnoDB 侧的三张 I_S 表](#innodb-侧的三张-i_s-表)
    - [PFS 与 sys schema：无用与冗余索引](#pfs-与-sys-schema无用与冗余索引)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引的生命周期 = **创建（建树 + 回填并发 DML）→ 使用（DML 维护）→ 变更/删除（标记 + 延迟释放）→ 重建**。本篇覆盖首尾三段与全程的可观测入口。

### 用途

- 理解 `ADD INDEX` 为什么"online"、什么时候不 online、失败会怎样；
- 理解 `DROP INDEX` 为什么"瞬间返回"却还有后续 IO；
- 定位索引相关问题：基数、空间、是否被使用、是否冗余。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 在线 DDL（`ADD INDEX` 不阻塞 DML）、row log |
| 8.0.12+ | INSTANT 算法（**但索引操作全不支持 INSTANT**） |
| 8.0.27+ | `innodb_ddl_threads` 并行构建索引（默认 4） |
| 8.0.30 | GIPK、`ALTER INDEX ... VISIBLE/INVISIBLE` 相关 DD 支持 |

**★ 源码里的版本证据**（`sql/dd/dd_version.h` 等，★ 可 grep 验证）：

| 行号 | 原文 | 含义 |
|---|---|---|
| `sql/dd/dd_version.h:71` | `Bug#2620373: Added index_stats.cached_time and table_stats.cached_time.` | ★ 没有 `cached_time` 就没有"缓存 Cardinality"这回事——它是 `INTERNAL_INDEX_COLUMN_CARDINALITY` 的第 11 个参数 |
| `sql/dd/dd_version.h:88` | `Also added explicit indexes for foreign keys instead of relying on these to be created implicitly...` | 8.0.4（DD 80004）起外键列改为**显式索引** |
| `sql/dd/dd_version.h:107` | `WL#1075 adds one column to INFORMATION_SCHEMA.STATISTICS: "EXPRESSION".` | 函数索引表达式列 |
| `sql/dd_table_share.cc:1361` | `// Beginning in 8.0.12 HA_PACK_KEY and HA_BINARY_PACK_KEY are no longer set` | 老索引的 flag 兼容 |
| `sql/sql_table.cc:7727` | `introduced in version 8.0.18`（函数参数就叫 `master_version`） | 函数索引列排最后 + 主从版本兼容 |
| `storage/perfschema/` | `INNODB_CACHED_INDEXES`（8.0 新增 I_S 表） | 缓存索引观测 |

> ⚠️ **`INNODB_SYS_*` 系列在 8.0.39 已全部不存在**（`i_s.cc` 中 grep `INNODB_SYS_` 零命中）——8.0 去掉了 `SYS_` 前缀，现在是 `INNODB_INDEXES` / `INNODB_TABLES` 等。
> ⚠️ **`Innodb_num_indexes` 状态变量在 8.0.39 也不存在**（全库 0 命中）。

---

## 理论基础

### 设计思想与权衡

**1. 在线建索引 = 扫描 + 排序 + 批量装载 + 日志回放。** 四条流水并行：一致性读扫描聚簇索引 → 归并排序 → 自底向上批量建页 → 回放扫描期间的并发 DML（row log）。这样 `ADD INDEX` 只在开始和结束短暂持锁。

**2. "延迟释放"是 DDL 原子性的代价。** `DROP INDEX` 不能立即释放 B-tree（否则 DDL 回滚无法恢复），所以走"打标记 → 写 DDL log → 提交后 post_ddl 重放日志真正释放"三段式。

**3. 统计与结构是分离的。** `OPTIMIZE TABLE` 是"重建结构 + 重算统计"两件事；只统计陈旧时 `ANALYZE` 就够，不必重建。

**4. 可观测分三层**：DD 元数据（`I_S.STATISTICS`：索引定义/可见性/表达式）、引擎内存状态（`INNODB_TABLESTATS`：行数/页数/修改计数）、PFS 使用统计（`table_io_waits_summary_by_index_usage`：索引到底被用过没）。三层回答不同问题。

### ★ 三种 DDL 算法：为什么不是一种

`ALTER TABLE` 的三种算法（COPY / INPLACE / INSTANT）不是"快慢三档"，而是**"需要触碰多少数据"的谱系**——每一档背后是不同的物理约束：

| 算法 | 触碰的数据 | 能这么做的前提（约束） |
|---|---|---|
| **INSTANT** | **0 行**（只改元数据） | 操作可以被表达为"元数据层面的新版本"，且**行格式能表达"这行是老版本"**——8.0 靠 instant 行版本状态机（见 [`record_format.md`](record_format.md)）：老行缺新列时用一个标志位表示"读这列时给默认值"。只有满足这个编码前提的操作（如末尾加列）才能 INSTANT |
| **INPLACE** | **只索引 / 只元数据**（不动行数据） | 行的物理布局**无需改变**——加/删索引、改索引可见性、重命名都属于此类（数据行本身没变，只是多了/少了一棵树） |
| **COPY** | **全表重建** | 操作**必须改变行的物理布局**——改列类型、删列、改字符集、改行格式。无法原地完成，只能建新表 + 拷数据 + 换名 |

★ **为什么不能全部 INSTANT**：因为大部分操作（删列、改类型）会让"老行"无法被正确解释——不像加列那样可以用"默认值"补齐。INSTANT 是 8.0 的一个**窄口子**，只服务于加列这个最高频的操作。

★ **为什么会退化**：优化器先尝试 INSTANT → INPLACE → COPY，逐级降级。一旦操作不满足上一档的约束（例如删列破坏行布局）就降到下一档。**`ALTER TABLE` 的 ALGORITHM 子句只是"上限声明"**——写 `ALGORITHM=INPLACE` 而操作实际需要 COPY 时会**报错**（不是静默降级），这是刻意的设计：让用户显式知情。

**★★★ 纠正一处根本性误解：handler 的返回枚举里根本没有 COPY。**

```cpp
/* sql/handler.h:197 —— 完整，只有 8 个值 */
enum enum_alter_inplace_result {
  HA_ALTER_ERROR,
  HA_ALTER_INPLACE_NOT_SUPPORTED,              /* ★ 返回它 ⇒ SQL 层退化成 COPY 算法 */
  HA_ALTER_INPLACE_EXCLUSIVE_LOCK,             /* ⚠️ InnoDB 从不返回此值（仅接口保留） */
  HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE,
  HA_ALTER_INPLACE_SHARED_LOCK,
  HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE,      /* ★★ ADD/DROP INDEX 的常态：prepare 拿一下 X 锁，之后全程不阻塞 DML */
  HA_ALTER_INPLACE_NO_LOCK,
  HA_ALTER_INPLACE_INSTANT                     /* ★ 8.0 新增 */
};
```

> ★★ 所以**"COPY vs INPLACE" 由 SQL 层决定**——它看 handler 是否返回 `HA_ALTER_INPLACE_NOT_SUPPORTED`；handler 只回答"能不能就地做、需要什么锁"。不存在 `HA_ALTER_INPLACE_RECREATE`。

**★★ InnoDB 侧真正的判据是四个位组**（`storage/innobase/handler/handler0alter.cc:109`，⚠️ **不在 `handler0alter.h`**）：

```cpp
/** Operations for creating secondary indexes (no rebuild needed) */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ONLINE_CREATE =
    Alter_inplace_info::ADD_INDEX | Alter_inplace_info::ADD_UNIQUE_INDEX |
    Alter_inplace_info::ADD_SPATIAL_INDEX;

/** Operations for rebuilding a table in place */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_REBUILD =
    Alter_inplace_info::ADD_PK_INDEX | Alter_inplace_info::DROP_PK_INDEX |
    Alter_inplace_info::CHANGE_CREATE_OPTION |
    Alter_inplace_info::ALTER_COLUMN_NULLABLE |
    Alter_inplace_info::ALTER_COLUMN_NOT_NULLABLE |
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN |
    Alter_inplace_info::RECREATE_TABLE;        /* ★ OPTIMIZE / FORCE / ENGINE= 落在这里 */

/** Operations that InnoDB does not care about */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INPLACE_IGNORE =
    Alter_inplace_info::ALTER_COLUMN_DEFAULT | ... |
    Alter_inplace_info::ALTER_RENAME | ...;    /* ★ RENAME TABLE 落在这里：纯元数据 */

/** Operation allowed with ALGORITHM=INSTANT */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN |
    Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN |
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN;    /* ★★ 注意：没有任何 *_INDEX 位！ */

/** Operations that InnoDB cares about and can perform without rebuild */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_NOREBUILD =
    INNOBASE_ONLINE_CREATE | INNOBASE_FOREIGN_OPERATIONS |
    Alter_inplace_info::DROP_INDEX | Alter_inplace_info::DROP_UNIQUE_INDEX |
    Alter_inplace_info::RENAME_INDEX | ...;    /* ★ DROP INDEX 在这里 */
```

> ★★★ 一句话：**flags 落在 `INNOBASE_INSTANT_ALLOWED` 内 → `INSTANT`；落在 `NOREBUILD|REBUILD` 内 → `INPLACE`；否则 → `NOT_SUPPORTED` → SQL 层做 `COPY`。**
> ★★ **INSTANT 与索引完全无关**——`INNOBASE_INSTANT_ALLOWED` 里没有任何索引位，所以 `innobase_support_instant()` 开头就对含索引操作返回 `INSTANT_IMPOSSIBLE`（`handler0alter.cc:831`）。

**★ 总闸门**（`check_if_supported_inplace_alter()`，`handler0alter.cc:998`）：

```cpp
  if (ha_alter_info->handler_flags &
      ~(INNOBASE_INPLACE_IGNORE | INNOBASE_ALTER_NOREBUILD |
        INNOBASE_ALTER_REBUILD)) {
    ...
    return HA_ALTER_INPLACE_NOT_SUPPORTED;   /* ★★ 不在"可 inplace"集合内 ⇒ COPY */
  }
```

**★ 决定锁级别的落点**（`handler0alter.cc:1353`，★ 函数只有两个"成功"返回值）：

```cpp
  return online ? HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE
                : HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE;
```

`online = false` 的几种典型（★ 都是"INPLACE 可以但不能 LOCK=NONE"）：

```cpp
  if (ha_alter_info->handler_flags & Alter_inplace_info::ADD_SPATIAL_INDEX) {
    ha_alter_info->unsupported_reason = innobase_get_err_msg(
        ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_GIS);
    online = false;                     /* ★ 加 SPATIAL INDEX 需锁 */
  }
  ...
    for (uint i = 0; i < ha_alter_info->index_add_count; i++) {
      const KEY *key = &ha_alter_info->key_info_buffer[...];
      if (key->flags & HA_FULLTEXT) {
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FTS);
        online = false;                 /* ★ 加 FULLTEXT INDEX 需锁 */
        break;
      }
    }
  ...
  } else if ((ha_alter_info->handler_flags & Alter_inplace_info::ADD_PK_INDEX) ||
             innobase_need_rebuild(ha_alter_info)) &&
            (innobase_fulltext_exist(altered_table) || innobase_spatial_exist(...))) {
    online = false;
    if (m_prebuilt->table->fts) {
      ha_alter_info->unsupported_reason = innobase_get_err_msg(ER_INNODB_FT_LIMIT);
      return HA_ALTER_INPLACE_NOT_SUPPORTED;   /* ★ 表上已有 FTS 又要 rebuild ⇒ 连 INPLACE 都不给 */
    }
```

**★ INSTANT 的实质动作在 commit 阶段**（`handler0alter.cc:1621`）——prepare/主阶段是 no-op：

```cpp
  if (is_instant(ha_alter_info)) {
    Instant_ddl_impl<dd::Table> executor(...);
    if (executor.commit_instant_ddl()) return true;   /* ★ 全部实质工作在这一行 */
  }
```

### ★ 为什么 row log 有大小上限

在线建索引期间，并发 DML 的变更进 row log，扫描结束后回放。上限由 `innodb_online_alter_log_max_size` 控制：

- 约束：row log 必须**全部回放完** DDL 才能提交，它是无界增长的（并发写入越多越大）；
- 若不加限制：磁盘被写满、回放时间不可控（可能导致 DDL 永远无法收敛）；
- 超限后果：**DDL 失败并回滚**（已扫描/排序的工作白费），报错 `DB_ONLINE_LOG_TOO_BIG`。

★ 实务含义：大表 + 高并发写入的 `ADD INDEX` 可能因为 row log 超限而失败——要么调大该变量，要么在业务低峰做。

**★ 上限的真身**（⚠️ **不存在 `ROW_LOG_MAX_SIZE` 宏**——全仓 grep 无此符号）：

```cpp
/* storage/innobase/srv/srv0srv.cc:204 */
/** Sort buffer size in index creation */
ulong srv_sort_buf_size = 1048576;
/** Maximum modification log file size for online index creation */
unsigned long long srv_online_max_size;      /* ★★ 这就是上限本体 */
```

```cpp
/* storage/innobase/handler/ha_innodb.cc:22541 —— 系统变量，默认 128MB */
static MYSQL_SYSVAR_ULONGLONG(
    online_alter_log_max_size, srv_online_max_size, PLUGIN_VAR_RQCMDARG,
    "Maximum modification log file size for online index creation", nullptr,
    nullptr, 128 << 20, 65536, ~0ULL, 0);
```

**★★ 判据**（`row0log.cc:367` / `:470`，★ 两处同形）：

```cpp
    const os_offset_t byte_offset = (os_offset_t)log->tail.blocks * srv_sort_buf_size;
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) {
      goto write_failed;      /* ★ 已写满的 block 数 × sort_buf_size 再加一块就超上限 */
    }
```

> ★ 所以 row log 的物理形态是**内存 buffer（`innodb_sort_buffer_size`，默认 1MB）+ 溢出到临时文件**，上限按"整块"计。

**★★★ 纠正："超限立即报错"是错的。** 两处写入现场的处理**不同**：

```cpp
/* (A) 二级索引在线建索引：row_log_online_op()，row0log.cc:392 */
    write_failed:
      /* We set the flag directly instead of
      invoking dict_set_corrupted() here,
      because the index is not "public" yet. */
      index->type |= DICT_CORRUPT;        /* ★★ 只标记索引损坏，不报 "too big" */

/* (B) 表重建：row_log_table_close_func()，row0log.cc:492 */
    write_failed:
      log->error = DB_ONLINE_LOG_TOO_BIG; /* ★★ 唯一的"直接设该错误码"现场 */
```

```cpp
/* (C) apply 阶段才把"损坏"翻译成"too big"，row0log.cc:3768 */
    case DB_INDEX_CORRUPT:
      if (((os_offset_t)index->online_log->tail.blocks + 1) * srv_sort_buf_size >=
          srv_online_max_size) {
        /* The log file grew too big. */
        error = DB_ONLINE_LOG_TOO_BIG;
      }
      [[fallthrough]];
    default:
      index->type |= DICT_CORRUPT;
```

```cpp
/* (D) 抛给用户：handler0alter.cc:6294（非 rebuild）与 :7004（rebuild） */
      case DB_ONLINE_LOG_TOO_BIG:
        my_error(ER_INNODB_ONLINE_LOG_TOO_BIG, MYF(0),
                 get_error_key_name(...));
```

> ★ 错误文本（`share/messages_to_clients.txt:6841`）：*"Creating index '%-.192s' required more than 'innodb_online_alter_log_max_size' bytes of modification log. Please try again."*
> ★ 另一处细节：**一旦被标记 corrupt，后续 DML 直接跳过写日志**（`row_log_online_op()` 开头 `if (index->is_corrupted()) return;`）——所以超限后日志不再增长，但 DDL 注定失败。

**★ row log 有两套，不只是 ADD INDEX**（`row0log.cc:172` 的 `row_log_t` 注释）：

```cpp
/** @brief Buffer for logging modifications during online index creation
All modifications to an index that is being created will be logged by
row_log_online_op() to this buffer.
All modifications to a table that is being rebuilt will be logged by
row_log_table_delete(), row_log_table_update(), row_log_table_insert()
to this buffer.
When head.blocks == tail.blocks, the reader will access tail.block directly.
When also head.bytes == tail.bytes, both counts will be reset to 0 and the
file will be truncated. */
```

> ★ 即：**在线加二级索引** 与 **表重建（rebuild）** 各用一套，都受同一个 `srv_online_max_size` 约束。

### ★ 为什么 online DDL 仍需要短暂的排他窗口

"online"不等于"全程不加锁"。DDL 在**头尾**各有一个短暂排他阶段：

- **prepare 阶段**：要拿 MDL（排他或升级），确保没有并发的 DML/查询正在使用即将被修改的对象；
- **commit 阶段**：要切换表定义 / 切换索引的可见状态，必须独占——否则并发查询会读到"一半新一半旧"的定义。

★ 后果：若此时有**长事务**持有该表的 MDL（哪怕只是个慢 SELECT），DDL 的 prepare 会**阻塞等待**；而 DDL 等待 MDL 又会**阻塞后续所有对该表的请求**（MDL 排队 fairness）——这就是"一个慢查询 + 一个 DDL = 业务雪崩"的机制（见 [`../infra/lock/transactional/mdl.md`](../infra/lock/transactional/mdl.md)）。

### ★ 为什么 DROP INDEX 是"三段式延迟释放"（深化）

第 2 条说了原因是"DDL 原子性的代价"。更完整地看是两个约束叠加：

1. **可回滚**：DDL 也要能回滚（ARIES 的逻辑日志思想），所以在提交前不能真正释放 B-tree；走 DDL log 记录"待释放"，提交后由 `post_ddl` 重放；
2. **不能悬垂释放**：释放索引的那一刻，可能还有并发查询正持有 `TABLE` 对象、甚至正在扫描这棵树。立即释放会让它们引用到已释放的内存/页。所以推迟到 post_ddl（确认无引用后）。

★ 这也是为什么 `DROP INDEX` 在元数据上"瞬间完成"，而**磁盘空间的释放有延迟**。

### 理论溯源

- Online index build（Oracle/MySQL 的经典实现）：扫描 + merge + 日志回放三段式；
- ARIES 的"逻辑 undo/redo"思想在 DDL log 上的应用（DDL 也要可回滚）。

---

## 核心实现

### 索引的创建

#### 从 `ALTER TABLE` 到 `ddl::Loader` 的完整链路

```
CREATE INDEX / ALTER TABLE ADD INDEX
  └─ mysql_alter_table()                       sql/sql_table.cc:16295
      ├─ fill_alter_inplace_info()             计算 handler_flags / 索引增删数量
      ├─ open_table_uncached()                 打开"新定义"的 TABLE
      ├─ (handler_flags == 0 → no-op：ADD/DROP 同索引会抵消)
      ├─ check_if_supported_inplace_alter()    → ha_innobase（handler0alter.cc:958）
      └─ mysql_inplace_alter_table()
          ├─ ha_prepare_inplace_alter_table()  → prepare_inplace_alter_table_impl (5407)
          │     ├─ ddl::create_index()         造 dict_index_t（:4897）
          │     ├─ row_log_allocate(二级索引)  （:4934）
          │     └─ row_log_allocate(聚簇)      （:4962，仅 rebuild）
          ├─ ha_inplace_alter_table()          → inplace_alter_table_impl (6105)
          │     └─ ddl::Context::build()       (ddl0ctx.cc:512)
          │          └─ Loader::build_all()    (ddl0loader.cc:484)
          │               ├─ Cursor::scan(m_builders)   并行扫描聚簇索引 → 临时文件
          │               └─ Loader::load()             (290) 并行排序 + 装载
          │                    └─ Builder::btree_build → Btree_load::build（btr0load.cc:1304）
          │                         └─ Builder::finalize → Flush_observer::flush
          │                                              → row_log_apply（回放）
          ├─ wait_while_table_is_used()        升级到 EXCLUSIVE（短暂）
          └─ ha_commit_inplace_alter_table()  → commit_inplace_alter_table_impl (7414)
```

`Loader::load()` 的并行方式（`ddl0loader.cc:290-345`）：

```cpp
const bool sync = m_ctx.m_max_threads <= 1;     // innodb_ddl_threads <= 1 则同步
for (auto builder : m_builders) {
  /* RTrees are built during the scan phase, using row by row insert. */
  if (!builder->is_spatial_index()) { builder->set_next_state(); add_task(Task{builder}); }
}
if (!sync) {
  for (size_t i = 1; i < m_ctx.m_max_threads; ++i) threads.push_back(std::thread{fn, i});
}
auto err = m_taskq->execute();                  // ★ 当前用户线程也参与执行
```

**R-tree 是例外**：空间索引在扫描阶段就逐行插入（无法排序后自底向上构建），且 **redo 不禁用**。

#### 三种算法：COPY / INPLACE / INSTANT 与索引

InnoDB 的 flag 分类（`handler0alter.cc:114-172`）决定了算法：

```cpp
INNOBASE_ONLINE_CREATE  = ADD_INDEX | ADD_UNIQUE_INDEX | ADD_SPATIAL_INDEX;
INNOBASE_ALTER_REBUILD  = ADD_PK_INDEX | DROP_PK_INDEX | CHANGE_CREATE_OPTION | ... | RECREATE_TABLE;
INNOBASE_ALTER_NOREBUILD= INNOBASE_ONLINE_CREATE | 外键操作 | DROP_INDEX | RENAME_INDEX | ...;
INNOBASE_INSTANT_ALLOWED= 全是列操作（改列名/加删虚拟列/加删存储列）;   // ★ 无索引相关 flag
```

| 索引 DDL | INSTANT | INPLACE | 备注 |
|---|---|---|---|
| `ADD INDEX` / `ADD UNIQUE` / `ADD SPATIAL` / `ADD FULLTEXT` | **否** | **是**（online，返回 `HA_ALTER_INPLACE_NO_LOCK`） | |
| `DROP INDEX` / `DROP UNIQUE` | **否** | **是**（`DROP_INDEX` ∈ NOREBUILD） | |
| `RENAME INDEX` | **否** | **是**（纯字典改名，不动数据） | |
| `ALTER INDEX ... VISIBLE/INVISIBLE` | — | 实质 no-op（只改 DD `is_visible`） | **瞬时** |
| `ADD / DROP PRIMARY KEY` | **否** | **是，但触发 rebuild**（`ADD_PK_INDEX` ∈ REBUILD） | 整表重建，所有二级索引一起重建 |
| `ALTER TABLE ... DISABLE/ENABLE KEYS` | — | **InnoDB 不支持**（普通表返回 `HA_ERR_WRONG_COMMAND`） | 只改 DD `keys_disabled`，索引仍被维护 |

**不是 online（`LOCK=NONE`）的例外**：

1. `ADD/DROP PRIMARY KEY` → 走 rebuild（虽仍用 row log，但重建全表）；
2. 同一条语句里混了任何触发 rebuild 的变更；
3. 只加/删外键且 `foreign_key_checks=ON`（`handler0alter.cc:1017` 直接 NOT_SUPPORTED）；
4. 修改 encryption 属性、字段数超限；
5. **表为空时 server 强制降级**（`sql_table.cc:13501`：ONLINE → SHARED，因为空表时不允许并发插入）。

#### 在线建索引的 row log

`row_log_t`（`row0log.cc:184`）是"扫描期间的并发 DML 暂存区"，本质是一个**内存 block + 临时文件**的日志：

```cpp
struct row_log_t {
  ddl::Unique_os_file_descriptor file;   // 溢出到临时文件
  ib_mutex_t mutex;                      // 保护 error / max_trx / tail
  dict_table_t *table;                   // 非 NULL 表示 rebuild（而非建二级索引）
  bool same_pk;                          // rebuild 时 PK 定义是否未变
  dberr_t error;
  trx_id_t max_trx;
  row_log_buf_t tail;                    // 写端
  row_log_buf_t head;                    // 读端
  const char *path;                      // 临时文件目录（innodb_tmpdir）
};
```

DML 侧钩子（`row0log.ic:48`）——每一条二级索引 DML 都先问一句"我是不是正在被 online build"：

```cpp
switch (dict_index_get_online_status(index)) {
  case ONLINE_INDEX_COMPLETE:  return false;        // 正常索引，不记日志
  case ONLINE_INDEX_CREATION:  row_log_online_op(index, tuple, trx_id); break;
  case ONLINE_INDEX_ABORTED:   break;               // 已中止：不记日志且跳过操作
}
```

**`innodb_online_alter_log_max_size` 超限的后果**（默认 128MB，最小 64KB）：**不是锁表、不是降级，而是本次 DDL 失败**——

```cpp
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) goto write_failed;
    ...
    write_failed:
      index->type |= DICT_CORRUPT;      // 二级索引：标记损坏
      // rebuild 侧：log->error = DB_ONLINE_LOG_TOO_BIG;
```

回放（`row_log_apply`，`row0log.cc:3800`）：

```cpp
rw_lock_x_lock(dict_index_get_lock(index));   // ★ 短暂 X 锁索引（此时阻塞 DML）
error = row_log_apply_ops(trx, index, &dup, stage);
if (error != DB_SUCCESS) { index->type |= DICT_CORRUPT; ... }
else dict_index_set_online_status(index, ONLINE_INDEX_COMPLETE);   // ★ 索引对外可见
rw_lock_x_unlock(...);
row_log_free(log);
```

回放时机在 `Builder::finalize()`（`ddl0builder.cc:1949`）：**先 `Flush_observer::flush()` 把批量装载期间未记 redo 的脏页全部刷盘，再写 `MLOG_INDEX_LOAD` 恢复 redo，最后回放 row log**。顺序不能颠倒——否则回放产生的 redo 与未刷盘的批量装载页会冲突。

#### 并行构建与临时文件

- **`innodb_ddl_threads`（默认 4，范围 1-64）**：并行扫描（受 `innodb_parallel_read_threads`）+ 并行排序归并。`<=1` 时同步执行。
- **`innodb_ddl_buffer_size`（默认 1 MiB，范围 64KB-4GB）**：是**总量**，被 `Context::scan_buffer_size()` 均分给 `n_threads × n_indexes` 套（sort buffer + IO buffer）——**同时建多个索引时每个索引拿到的内存会变少**。
- **临时文件** `ddl::file_t`（`ddl0impl.h:67`）：目录取 `innodb_tmpdir`；排序缓冲满 → `key_buffer_sort()` → `serialize()` 以 append 方式写临时文件并记录每个有序段的偏移，之后 `Merge_file_sort` 多路归并。row log 也复用同一套临时文件机制。

---

### 索引的销毁与变更

#### DROP INDEX：三段式（标记 → 日志 → post_ddl 释放）

| 阶段 | 做什么 | 释放页？ |
|---|---|---|
| PREPARE | `index->to_be_dropped = 1`；若被外键引用则报 `HA_ERR_DROP_INDEX_FK` | 否 |
| INPLACE/COMMIT | `log_ddl->write_free_tree_log()` 往 `mysql.innodb_ddl_log` 插一条记录；`btr_drop_ahi_for_index()`；`dict_index_remove_from_cache()` | **否**（只打日志） |
| **post_ddl**（提交后） | `Log_DDL::post_ddl()` → `replay_free_tree_log()` → `btr_free_if_exists()` → `btr_free_but_not_root()` + `btr_free_root()` | **是**（同步释放） |

```cpp
// log0ddl.cc:1625
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no, ulint index_id) {
  ...
  btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
}
```

这一步往下的整树释放链（`btr_free_if_exists` → `btr_free_but_not_root` + `btr_free_root` + `btr_free_root_invalidate`）在 [`btr.md`](btr.md) 的「整树释放：`btr_free_if_exists` 链」一节有完整源码剖析。三要点先记在这里，因为它们决定了这条路径的正确性：

**★ 第一段：prepare 只打标记**（`handler0alter.cc:5774`）：

```cpp
    if (!n_drop_index) {
      drop_index = nullptr;
    } else {
      /* Flag all indexes that are to be dropped. */
      for (ulint i = 0; i < n_drop_index; i++) {
        ut_ad(!drop_index[i]->to_be_dropped);
        drop_index[i]->to_be_dropped = 1;      /* ★★★ 只是置位，不释放任何东西 */
      }
    }
```

失败回滚也印证了"标记"语义（`handler0alter.cc:5853`）：`drop_index[i]->to_be_dropped = 0;`

**★ 为什么不能立即释放——状态字段与三个证据点**：

```cpp
/* storage/innobase/include/dict0mem.h:1143 */
  /** true if the index is to be dropped; protected by dict_operation_lock */
  unsigned to_be_dropped : 1;   /* ★ 只表示"待删除"，真实内存对象还在字典缓存里被并发访问 */
```

```cpp
/* storage/innobase/dict/dict0stats.cc:162 —— 统计线程会跳过待删索引 */
static inline bool dict_stats_should_ignore_index(const dict_index_t *index) {
  return ((index->type & DICT_FTS) || index->is_corrupted() ||
          dict_index_is_spatial(index) || index->to_be_dropped ||
          !index->is_committed());
}
```

```cpp
/* storage/innobase/btr/btr0sea.cc:1090 —— AHI 在待删索引上留有条目，
   状态 ONLINE_INDEX_ABORTED_DROPPED 的索引其 AHI 清理被推迟到 post_ddl */
      case ONLINE_INDEX_ABORTED_DROPPED:
        /* Since dropping the indexes are delayed to post_ddl,
        this status is similar to ONLINE_INDEX_ABORTED. */
```

**★ 真正的"等引用归零"**（`dict_index_remove_from_cache_low()`，`dict0dict.cc:2669`，⚠️ **定义在 `dict0dict.cc` 不是 `dict0crea.cc`**）：

```cpp
  if (index->online_log) {
    ut_ad(index->online_status == ONLINE_INDEX_CREATION);
    row_log_free(index->online_log);
  }

  btr_search_await_no_reference(table, index, !lru_evict);   /* ★★ AHI 引用计数必须降到 0 */
  rw_lock_free(&index->lock);
  ...
  UT_LIST_REMOVE(table->indexes, index);
  dict_mem_index_free(index);   /* ★ 释放的是 dict_index_t 这个"元数据对象"，不是 B+Tree 页面！ */
```

**★ 第二段：commit 阶段清缓存 + 写 DDL log**（`handler0alter.cc:7247`）：

```cpp
      if (index->page != FIL_NULL) {
        dict_sys_mutex_exit();
        ut_d(dberr_t err =) log_ddl->write_free_tree_log(trx, index, true);  /* ★ 写 FREE_TREE_LOG */
        ut_ad(err == DB_SUCCESS);
        dict_sys_mutex_enter();
      }
      btr_drop_ahi_for_index(index);
      dict_index_remove_from_cache(index->table, index);   /* ★ 把 dict_index_t 从缓存摘掉 */
```

**★★ 第三段：post_ddl 回放**（⚠️ **不是后台线程**）：

```cpp
/* storage/innobase/handler/ha_innodb.cc:5100 */
static void innobase_post_ddl(THD *thd) {
  if (log_ddl != nullptr) {
    dberr_t err = log_ddl->post_ddl(thd);
    if (err != DB_SUCCESS) {
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_DDL_LOG_FAIL_POST_DDL);   /* ★ 失败即崩溃 */
    }
  }
}
/* 注册点 ha_innodb.cc:5223 —— handlerton 钩子，不是线程 */
  innobase_hton->post_ddl = innobase_post_ddl;
```

```cpp
/* SQL 层在 DDL 提交/回滚后同步调用：sql/sql_table.cc:3742 与 :3786 */
    for (handlerton *hton : *post_ddl_htons) hton->post_ddl(thd);
```

```cpp
/* log0ddl.cc:1931 —— 只回放"本线程"写下的记录 */
  ulint thread_id = thd_get_thread_id(thd);
  ...
  dberr_t err = replay_by_thread_id(thread_id);
```

> ★★★ 三处常见误解：**① 没有独立的 post_ddl 后台线程**（不是 `innodb_purge_threads`），是 handlerton 钩子由 SQL 层同步调用；② **日志按 `thread_id` 分片**，每个连接只回放自己写的；③ **回滚路径同样调用 post_ddl**（保证"要么都做要么都不做"）。

**★ DDL log 的类型**（`log0ddl.h:45`，`FREE_TREE_LOG` 就是"删索引"）：

```cpp
enum class Log_Type : uint32_t {
  SMALLEST_LOG = 1,
  /** Drop an index tree */
  FREE_TREE_LOG = 1,
  DELETE_SPACE_LOG, RENAME_SPACE_LOG, DROP_LOG, RENAME_TABLE_LOG,
  REMOVE_CACHE_LOG, ALTER_ENCRYPT_TABLESPACE_LOG, ALTER_UNENCRYPT_TABLESPACE_LOG,
  BIGGEST_LOG = ALTER_UNENCRYPT_TABLESPACE_LOG
};
```

**★ 还有一条"上一轮遗留"的重试清理**（`handler0alter.cc:3236`，★ 说明延迟删除是常态）：`online_retry_drop_dict_indexes()` 会扫描并清掉所有 `ONLINE_INDEX_ABORTED_DROPPED` 状态的索引。

1. **`btr_free_root_check` 先校验再动手**——按 `page_id` 取 root 页，比对页里的 `PAGE_INDEX_ID` 是否等于日志中记的 `index_id`；不等就返回 nullptr。★ 这防的是"页已被复用给别的索引"——DDL log 是崩溃恢复时重放的，期间这个页可能已经归属新索引，**不做这一步会把别人的树删掉**。
2. **`btr_free_but_not_root` 是分步释放**——叶子段与非叶段各一个 `fseg_free_step` 循环（`goto leaf_loop` / `top_loop`），**每个 mtr 只释放一小步**，避免单个 mtr 过大。
3. **释放完 root 页还要 `btr_free_root_invalidate`**——把 root 页的 `PAGE_INDEX_ID` 置 0，避免残留一个"看起来仍是某索引根页"的半成品。

> **结论：DROP INDEX 的 B-tree 空间是提交后由 post_ddl 重放 DDL log 时"立即物理释放"的，不是 purge 回收。** purge 回收的是**行记录**（delete-mark 记录与回滚段历史），与"整棵索引树的段"无关。

最后一个细节：`btr_free_but_not_root` 用 `fseg_free_step*()` **分批**释放（每批一个 mtr），避免单个 mtr 过大——**这就是大索引 DROP 时有短暂 IO 抖动的原因**。

#### RENAME / 可见性 / DISABLE KEYS / 主键增删

- **RENAME INDEX**：`rename_index_in_cache()`（`handler0alter.cc:5260`）只改内存里的 `dict_index_t::name`，不动数据、不改 root page、**极快且 online**。
- **`ALTER INDEX ... VISIBLE/INVISIBLE`**：**InnoDB handler 完全没有对应实现**（grep `set_visible` 在 handler0alter.cc 只命中 `ALTER_COLUMN_VISIBILITY`）。可见性纯粹是 **DD 元数据**（`mysql.indexes.is_visible`），**不触碰任何页**——这是它瞬时的原因（详见 [`types.md`](types.md)）。
- **`DISABLE/ENABLE KEYS`**：`ha_innobase::enable_indexes/disable_indexes` 对普通 InnoDB 表直接返回 `HA_ERR_WRONG_COMMAND`（只对 intrinsic 临时表实现）。server 层退化为"只更新 DD 的 `keys_disabled`"，**索引仍被完整维护**。
- **`ADD/DROP PRIMARY KEY`**：触发 rebuild（整表重建 + 所有二级索引重建，因为二级索引叶子里存的是 PK 值）。`same_pk=false` 时 row log 要记录完整旧 PK + `DB_TRX_ID`/`DB_ROLL_PTR`，回放开销更大。优化：若新旧 PK 顺序一致（`innobase_pk_order_preserved`），聚簇索引可**跳过文件排序**（`skip_pk_sort`）。

---

### 索引的重建

#### OPTIMIZE TABLE 的真相

**完整原文**（`ha_innodb.cc:18122`，★ Doxygen 里就写明它被映射成什么）：

```cpp
/** This is mapped to "ALTER TABLE tablename ENGINE=InnoDB", which rebuilds
 the table in MySQL. */

int ha_innobase::optimize(THD *,          /*!< in: connection thread handle */
                          HA_CHECK_OPT *) /*!< in: currently ignored */
{
  TrxInInnoDB trx_in_innodb(m_prebuilt->trx);

  /* FTS-FIXME: Since MySQL doesn't support engine-specific commands,
  we have to hijack some existing command in order to be able to test
  the new admin commands added in InnoDB's FTS support. For now, we
  use MySQL's OPTIMIZE command, normally mapped to ALTER TABLE in
  InnoDB (so it recreates the table anew), and map it to OPTIMIZE.

  This works OK otherwise, but MySQL locks the entire table during
  calls to OPTIMIZE, which is undesirable. */

  if (innodb_optimize_fulltext_only) {
    if (m_prebuilt->table->fts && m_prebuilt->table->fts->cache &&
        !dict_table_is_discarded(m_prebuilt->table)) {
      fts_sync_table(m_prebuilt->table, false, true, false);
      fts_optimize_table(m_prebuilt->table);   /* ★ 只在此开关打开时才真做 FTS 优化 */
    }
    return (HA_ADMIN_OK);
  } else {
    return (HA_ADMIN_TRY_ALTER);               /* ★★ 默认：InnoDB 自己什么也不做 */
  }
}
```

```cpp
/* sql/handler.h:181 */
#define HA_ADMIN_TRY_ALTER -7
```

**★ SQL 层收到后的处理**（`sql/sql_admin.cc:1237`，★ 你看到的那句提示就出自这里）：

```cpp
      case HA_ADMIN_TRY_ALTER: {
        ...
        protocol->store_string(STRING_WITH_LEN("note"), system_charset_info);
        if (alter_info->flags & Alter_info::ALTER_ADMIN_PARTITION) {
          protocol->store_string(
              STRING_WITH_LEN("Table does not support optimize on "
                              "partitions. All partitions "
                              "will be rebuilt and analyzed."),
              system_charset_info);
        } else {
          protocol->store_string(
              STRING_WITH_LEN("Table does not support optimize, "
                              "doing recreate + analyze instead"),
              system_charset_info);       /* ★★★ 这句提示的出处 */
        }
        ...
        result_code = (open_temporary_tables(thd, table) ||
                       mysql_recreate_table(thd, table, false));
```

> ★★ 所以 `OPTIMIZE TABLE` 的默认路径是：**`mysql_recreate_table()`（= `ALTER TABLE ... FORCE`，走 `RECREATE_TABLE` 位 → `INNOBASE_ALTER_REBUILD` → rebuild）+ 之后 `analyze`**。注释原话：*"so here we close the table, do an ALTER TABLE, reopen the table and do ha_innobase::analyze() on it"*。
> ★ `innodb_optimize_fulltext_only` 默认 **false**（`ha_innodb.cc:22546`），且是 `PLUGIN_VAR_NOCMDARG`（不能在命令行设置）。

server 收到 `HA_ADMIN_TRY_ALTER` 后（`sql_admin.cc:1237`）：提示 "Table does not support optimize, doing recreate + analyze instead" → `mysql_recreate_table()`（`ALTER TABLE ... FORCE` 等价路径，`RECREATE_TABLE` ∈ REBUILD）→ 之后再做一次 `ha_innobase::analyze()`。

**结论：`OPTIMIZE TABLE`（除 FTS）是"真 rebuild + analyze"，不是只做 analyze。**

**什么时候真的需要重建**：

1. `DATA_FREE` 显著（碎片）；
2. 页填充率低——删除/更新造成页内空洞，而 `btr_page_reorganize()` **只在插入放不下时本地重排、不跨页搬迁**，所以空洞不会自动消除；
3. 变更 `ROW_FORMAT`/`KEY_BLOCK_SIZE`/`TABLESPACE`（`innobase_need_rebuild()`）；
4. FTS 表清理已删除文档残留（`fts_optimize_table()`）。

只是统计陈旧 → **`ANALYZE TABLE` 即可，不必重建**。

---

### 可观测

#### `SHOW INDEX`：Cardinality 与缓存

**实现位置纠正**：8.0 的 `SHOW INDEX` **不在** `sql/sql_show.cc`，而是被重写为对系统视图 `SHOW_STATISTICS` 的查询（`sql/dd/info_schema/show.cc:756` 的 `build_show_keys_query()`）。

`Cardinality` 列来自 `INTERNAL_INDEX_COLUMN_CARDINALITY(...)`（`sql/item_func.cc:9483`），底层 `Table_statistics::read_stat()`（`sql/dd/info_schema/table_stats.cc:443`）：

```cpp
if (!is_persistent_statistics_expired(thd, cached_timestamp)) {
  return table_stat_data;        // ★ 缓存未过期：直接用 mysql.index_stats
}
if (hton_implements_get_statistics) read_stat_from_SE(...)   // 走引擎
else                             read_stat_by_open_table(...)  // 退化：开表再取
```

> **运维要点**：`SHOW INDEX` 的 `Cardinality` **默认最多陈旧 24 小时**（`information_schema_stats_expiry`）。想立刻取最新值：`SET information_schema_stats_expiry=0`。

**★★★ Cardinality 的完整公式链**（★ 这是全段最该记住的东西）

`Cardinality` **不是字典里存的列**，是内置函数 `INTERNAL_INDEX_COLUMN_CARDINALITY(...)` 的**运行时计算**结果。终极公式在 InnoDB 侧：

```cpp
/* storage/innobase/handler/ha_innodb.cc:17790 */
      } else {
        uint64_t n_rows = ib_table->stat_n_rows;
        double records =
            (n_rows / innodb_rec_per_key(index, (ulint)column_ordinal_position, n_rows));
        *cardinality = static_cast<ulonglong>(round(records));
      }
```

而 `rec_per_key` 本身是（`ha_innodb.cc:17089`）：

```cpp
  n_diff = index->stat_n_diff_key_vals[i];
  ...
  if (records == 0) {
    /* "Records per key" is meaningless for empty tables.
    Return 1.0 because that is most convenient to the Optimizer. */
    return (1.0);                                  /* ★ 空表：rec_per_key = 1.0 */
  }
  if (n_diff == 0) {
    rec_per_key = static_cast<rec_per_key_t>(records);   /* ★ 没统计过：= records */
  } else if (srv_innodb_stats_method == SRV_STATS_NULLS_IGNORED) {
    ... rec_per_key = (records - n_null) / (n_diff - n_null);
  } else {
    rec_per_key = static_cast<rec_per_key_t>(records) / n_diff;  /* ★ 默认 nulls_equal */
  }
```

**代入后**（默认 `nulls_equal`）：

| 情形 | Cardinality |
|---|---|
| 常规（`n_diff>0`, `rows>0`） | **`round(stat_n_rows / (stat_n_rows / n_diff))` ≈ `round(n_diff)`** ← 这才等于"distinct 值个数" |
| FULLTEXT / SPATIAL | **`stat_n_rows`**（`rec_per_key` 恒为 1.0） |
| 空表 | `0 / 1 = 0` |
| `n_diff == 0`（没 ANALYZE 过） | `round(rows / rows) = 1` |
| `has_records_per_key()==false` | **NULL**（返回 -1） |

> ★★ 所以 **`Cardinality ≈ n_diff` 只在"默认 `nulls_equal`、非前缀、非 FULLTEXT/SPATIAL、且统计过"时才成立**。别把它当成字典里存好的精确值。

**★ 另一个除法的真身**（分区表 / 引擎未实现时的退化路径，`table_stats.cc:830`）：

```cpp
      if (key_index < table->s->keys &&
          key->has_records_per_key(column_ordinal_position)) {
        records = (table->file->stats.records /
                   key->records_per_key(column_ordinal_position));
        records = static_cast<longlong>(round(records));
      } else
        records = -1;  // Treated as NULL
```

**★ `ANALYZE TABLE` 时写 `mysql.index_stats` 用的是同一个除法**（`table_stats.cc:343`）：

```cpp
  if (analyze_table->file->info(HA_STATUS_VARIABLE | ... ) != 0) return true;
                                /* ★ ANALYZE 主动 info() 刷新 stats ⇒ 之后变准 */
      if (key->has_records_per_key(j)) {
        double recs = (analyze_table->file->stats.records / key->records_per_key(j));
        records = static_cast<longlong>(round(recs));
      } else
        records = -1;
```

**★★ "查询即缓存"**：算完会顺手写回 `mysql.index_stats`（`table_stats.cc:538`），但**有前提**：

```cpp
inline bool can_persist_I_S_dynamic_statistics(THD *thd, const char *schema_name,
                                               const char *partition_name) {
  return (thd->variables.information_schema_stats_expiry &&
          !thd->variables.transaction_read_only && !super_read_only &&
          !thd->in_sub_stmt && !read_only && !partition_name &&
          !thd->in_multi_stmt_transaction_mode() &&
          (strcmp(schema_name, "performance_schema") != 0));
}
```

> ★ **在事务里（`BEGIN` / `autocommit=0`）查 `I_S.STATISTICS` 不会回写缓存**——查多少次都是真算。

**★★ `SHOW INDEX` 会不会触发统计重算？默认不会。**

```cpp
/* storage/innobase/handler/ha_innodb.cc:17231 */
  if (flag & HA_STATUS_TIME) {
    if (is_analyze || innobase_stats_on_metadata) {      /* ★ 双重闸门 */
      if (dict_stats_is_persistent_enabled(ib_table)) {
        if (is_analyze) {
          opt = DICT_STATS_RECALC_PERSISTENT;            /* ANALYZE：真重算并落盘 */
        } else {
          /* This is e.g. 'SHOW INDEXES', fetch
          the persistent stats from disk. */
          opt = DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY;   /* ★★ 源码点名 SHOW INDEXES：只"取"不"算" */
        }
      } else {
        opt = DICT_STATS_RECALC_TRANSIENT;
      }
      ret = dict_stats_update(ib_table, opt);
```

```cpp
/* storage/innobase/dict/dict0stats.cc:2895 —— FETCH_ONLY 的语义核心 */
    case DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY:
      if (table->stat_initialized) {
        return (DB_SUCCESS);      /* ★★★ 已在内存 → 直接返回，零 I/O、零重算 */
      }
```

> ★ 默认 `innodb_stats_on_metadata=OFF`（`ha_innodb.cc:22201`），整个 `HA_STATUS_TIME` 分支都被跳过。**真正会重算的只有三种**：`ANALYZE TABLE`、后台自动重算线程、以及**非持久化统计表**在 `innodb_stats_on_metadata=ON` 时。
> ⚠️ 源码一处易混淆点：`innobase_stats_on_metadata` 的 C++ 静态初值在 `ha_innodb.cc:339` 是 **`true`**，但插件变量声明的默认参数是 **`false`**（注释亦写 "off by default"）——**生效默认值是 OFF**。

其他列：`Sub_part`（前缀长度，经 `GET_DD_INDEX_SUB_PART_LENGTH` 做字节/字符换算）、`Collation`（`A`/`D`，降序索引）、`Index_type`（`BTREE`/`FULLTEXT`/`SPATIAL`）、`Comment`（`disabled` = DISABLE KEYS 生效中）、`Packed`（**恒 NULL**）。

> ★ `Null` 列注意是**空串 `''` 不是 `'NO'`**（`IF(col.is_nullable = 1, 'YES','')`）；`INDEX_NAME` 用 `COLLATE utf8mb3_tolower_ci` 故大小写不敏感，而 `INNODB_INDEXES.NAME` 是大小写敏感的。

#### `I_S.STATISTICS` 列与语义

`Statistics_base`（`statistics.cc:43`）的列与来源：

| 列 | 来源表达式 | 语义要点 |
|---|---|---|
| `NON_UNIQUE` | `IF(idx.type='PRIMARY' OR idx.type='UNIQUE',0,1)` | |
| `COLUMN_NAME` | `IF(col.hidden='SQL', NULL, col.name ...)` | **功能索引时 NULL**（隐藏列） |
| `COLLATION` | `CASE icu.order WHEN 'DESC' THEN 'D' WHEN 'ASC' THEN 'A' END` | 降序索引 |
| `SUB_PART` | `GET_DD_INDEX_SUB_PART_LENGTH(...)` | 前缀长度；非前缀为 NULL |
| `INDEX_TYPE` | `CASE WHEN idx.type='SPATIAL' THEN 'SPATIAL' ... ELSE idx.algorithm END` | |
| `COMMENT` | `IF(... INTERNAL_KEYS_DISABLED(tbl.options),'disabled','')` | |
| **`IS_VISIBLE`** | `IF(idx.is_visible,'YES','NO')` | 8.0 新增 |
| **`EXPRESSION`** | `IF(col.hidden='SQL', col.generation_expression_utf8, NULL)` | **功能索引的表达式**：`WHERE EXPRESSION IS NOT NULL` 可筛出功能索引 |
| `CARDINALITY` | `INTERNAL_INDEX_COLUMN_CARDINALITY(...)` | 仅 `Statistics` 视图有 |

过滤条件 `IS_VISIBLE_DD_OBJECT(tbl.hidden, idx.hidden OR icu.hidden, idx.options)`——GIPK 在 `show_gipk_in_create_table_and_information_schema=OFF` 时由此隐藏。

**★ 视图定义原文**（`sql/dd/impl/system_views/statistics.cc:43`，★ 每个列的 SQL 表达式就是上表"来源"列的出处）：

```cpp
Statistics_base::Statistics_base() {
  m_target_def.add_field(FIELD_TABLE_CATALOG, "TABLE_CATALOG", "cat.name" + ...);
  m_target_def.add_field(FIELD_TABLE_SCHEMA, "TABLE_SCHEMA", "sch.name" + ...);
  m_target_def.add_field(FIELD_TABLE_NAME, "TABLE_NAME", "tbl.name" + ...);
  m_target_def.add_field(FIELD_NON_UNIQUE, "NON_UNIQUE",
      "IF (idx.type = 'PRIMARY' OR idx.type = 'UNIQUE',0,1)");
  m_target_def.add_field(FIELD_INDEX_NAME, "INDEX_NAME",
      "idx.name COLLATE utf8mb3_tolower_ci");       /* ★ 大小写不敏感 */
  m_target_def.add_field(FIELD_SEQ_IN_INDEX, "SEQ_IN_INDEX", "icu.ordinal_position");
  m_target_def.add_field(FIELD_COLUMN_NAME, "COLUMN_NAME",
      "IF (col.hidden = 'SQL', NULL, col.name COLLATE utf8mb3_tolower_ci)");
  m_target_def.add_field(FIELD_COLLATION, "COLLATION",
      "CASE WHEN icu.order = 'DESC' THEN 'D' "
      "WHEN icu.order = 'ASC'  THEN 'A' ELSE NULL END");
  m_target_def.add_field(FIELD_SUB_PART, "SUB_PART",
      "GET_DD_INDEX_SUB_PART_LENGTH(icu.length, col.type, col.char_length,"
      " col.collation_id, idx.type)");
  m_target_def.add_field(FIELD_PACKED, "PACKED", "NULL");   /* ★ 恒 NULL */
  m_target_def.add_field(FIELD_NULLABLE, "NULLABLE",
      "IF (col.is_nullable = 1, 'YES','')");                /* ★ 空串不是 'NO' */
  m_target_def.add_field(FIELD_INDEX_TYPE, "INDEX_TYPE",
      "CASE WHEN idx.type = 'SPATIAL' THEN 'SPATIAL' "
      "WHEN idx.algorithm = 'SE_PRIVATE' THEN '' ELSE idx.algorithm END ");
  m_target_def.add_field(FIELD_COMMENT, "COMMENT",
      "IF (idx.type = 'PRIMARY' OR idx.type = 'UNIQUE', "
      "    '',IF(INTERNAL_KEYS_DISABLED(tbl.options),'disabled', ''))");
  m_target_def.add_field(FIELD_INDEX_COMMENT, "INDEX_COMMENT", "idx.comment");
  m_target_def.add_field(FIELD_IS_VISIBLE, "IS_VISIBLE", "IF (idx.is_visible, 'YES', 'NO')");
  m_target_def.add_field(FIELD_EXPRESSION, "EXPRESSION",
      "IF (col.hidden = 'SQL', col.generation_expression_utf8, NULL)");

  m_target_def.add_from("mysql.index_column_usage icu");
  m_target_def.add_from("JOIN mysql.indexes idx ON idx.id=icu.index_id");
  m_target_def.add_from("JOIN mysql.tables tbl ON idx.table_id=tbl.id");
  m_target_def.add_from("JOIN mysql.columns col ON icu.column_id=col.id");
  m_target_def.add_from("JOIN mysql.schemata sch ON tbl.schema_id=sch.id");
  m_target_def.add_from("JOIN mysql.catalogs cat ON cat.id=sch.catalog_id");

  m_target_def.add_where("CAN_ACCESS_TABLE(sch.name, tbl.name)");   /* ★ 权限裁剪 */
  m_target_def.add_where("AND IS_VISIBLE_DD_OBJECT(tbl.hidden, "
                         "idx.hidden OR icu.hidden, idx.options)");
}
```

> ★ **`CAN_ACCESS_TABLE`** 意味着：没权限的表**在结果集里直接消失**，不报错。

**★ `SHOW_STATISTICS` 是 `STATISTICS` 的隐藏子类**（`statistics.cc:127`）——只多两列用于排序：

```cpp
Show_statistics::Show_statistics() {
  m_target_def.set_view_name(view_name());
  m_target_def.add_field(FIELD_INDEX_ORDINAL_POSITION, "INDEX_ORDINAL_POSITION",
                         "idx.ordinal_position");
  m_target_def.add_field(FIELD_COLUMN_ORDINAL_POSITION, "COLUMN_ORDINAL_POSITION",
                         "icu.ordinal_position");
}
```

> ★ 这解释了为什么 `SHOW INDEX` 的输出顺序是"索引序号 → 列序号"，而直接查 `I_S.STATISTICS` 没有这个保证。

#### InnoDB 侧的三张 I_S 表

均在 `storage/innobase/handler/i_s.cc`（**访问需 PROCESS 权限**）：

| 表 | 关键列 | 语义 |
|---|---|---|
| `INNODB_TABLESTATS` | `NUM_ROWS`、`CLUST_INDEX_SIZE`（**页**）、`OTHER_INDEX_SIZE`（**页**）、**`MODIFIED_COUNTER`**、`STATS_INITIALIZED` | 表级统计；`MODIFIED_COUNTER / NUM_ROWS` 判断统计新鲜度；二级索引总空间的权威来源 |
| `INNODB_INDEXES` | `INDEX_ID`、`NAME`、`TYPE`（**位掩码**）、`N_FIELDS`（含系统列）、`PAGE_NO`、`MERGE_THRESHOLD` | **无 `n_diff` 列**（5.7 的 `INNODB_INDEX_STATS` 已移除）——看基数要查 `mysql.innodb_index_stats` |
| `INNODB_TABLESPACES` | `FILE_SIZE`、`ALLOCATED_SIZE`、`AUTOEXTEND_SIZE` | 物理文件大小；与 `stat_index_size × innodb_page_size` 对比可判断"文件是否虚胖" |

**★ `INNODB_INDEXES` 的字段定义**（`i_s.cc:5842`，★ `TYPE` 是**原始位掩码**不是字符串）：

```cpp
static ST_FIELD_INFO innodb_sysindex_fields_info[] = {
#define SYS_INDEX_ID 0
    {STRUCT_FLD(field_name, "INDEX_ID"), ... MYSQL_TYPE_LONGLONG ... MY_I_S_UNSIGNED},
                                /* 0 INDEX_ID：InnoDB 内部索引 ID */
#define SYS_INDEX_NAME 1
    {STRUCT_FLD(field_name, "NAME"), STRUCT_FLD(field_length, NAME_LEN + 1), ...},
                                /* 1 NAME：索引名（大小写敏感，与 I_S.STATISTICS 不同） */
#define SYS_INDEX_TABLE_ID 2
    ... "TABLE_ID" ...           /* 关联 INNODB_TABLES.TABLE_ID */
#define SYS_INDEX_TYPE 3
    ... "TYPE" ...               /* ★ dict_index_t::type 的原始位掩码：
                                    DICT_CLUSTERED/UNIQUE/DICT_FTS/DICT_SPATIAL... */
#define SYS_INDEX_NUM_FIELDS 4
    ... "N_FIELDS" ...           /* ★ 含内部追加的 PK 列，常大于用户可见列数 */
#define SYS_INDEX_PAGE_NO 5
    ... "PAGE_NO" ...            /* root page 号；FIL_NULL 时填 -1 */
#define SYS_INDEX_SPACE 6
    ... "SPACE" ...              /* 表空间 ID */
#define SYS_INDEX_MERGE_THRESHOLD 7
    ... "MERGE_THRESHOLD" ...    /* 索引级页合并阈值，默认 50 */
    END_OF_ST_FIELD_INFO};
```

**★ 填充逻辑**（`i_s.cc:5905`）：

```cpp
static int i_s_dict_fill_innodb_indexes(THD *thd, const dict_index_t *index,
                                        TABLE *table_to_fill) {
  OK(field_store_index_name(fields[SYS_INDEX_NAME], index->name));
  OK(fields[SYS_INDEX_ID]->store(longlong(index->id), true));
  OK(fields[SYS_INDEX_TABLE_ID]->store(longlong(index->table->id), true));
  OK(fields[SYS_INDEX_TYPE]->store(index->type));
  OK(fields[SYS_INDEX_NUM_FIELDS]->store(index->n_fields));
  /* FIL_NULL is UINT32_UNDEFINED */
  if (index->page == FIL_NULL) {
    OK(fields[SYS_INDEX_PAGE_NO]->store(-1));
  } else {
    OK(fields[SYS_INDEX_PAGE_NO]->store(index->page));
  }
  OK(fields[SYS_INDEX_SPACE]->store(index->space));
  OK(fields[SYS_INDEX_MERGE_THRESHOLD]->store(index->merge_threshold));
```

> ★ 数据源：直接扫描 DD 表 `mysql.indexes`（`dd_startscan_system`），逐条还原成 `dict_index_t`——**不遍历 buffer pool**。

**★ 权限**（`i_s.cc:5945`）：`check_global_access(thd, PROCESS_ACL)` 失败时**静默返回空结果集**（不报错）——这是"查 INNODB_INDEXES 什么都没有"的第一排查点。

**★ 真名核查：`INNODB_FIELDS` 不在 `i_s.cc`**，它是 DD 系统视图（`sql/dd/impl/system_views/innodb_fields.cc:34`）：

```cpp
  m_target_def.add_field(FIELD_INDEX_ID, "INDEX_ID",
      "GET_DD_INDEX_PRIVATE_DATA(idx.se_private_data, 'id')");
  m_target_def.add_field(FIELD_NAME, "NAME", "col.name");
  m_target_def.add_field(FIELD_POS, "POS", "fld.ordinal_position - 1");
                       /* ★ POS 是 0-based（与 INNODB_INDEXES 一致），
                          而 I_S.STATISTICS 的 SEQ_IN_INDEX 是 1-based */
  ...
  m_target_def.add_where("AND NOT fld.hidden");
  m_target_def.add_where("AND tbl.engine='INNODB'");   /* ★ 只暴露 InnoDB 可见表 */
```

#### PFS 与 sys schema：无用与冗余索引

**`performance_schema.table_io_waits_summary_by_index_usage`**（`table_tiws_by_index_usage.cc:49`）：列 `COUNT_STAR` / `SUM_TIMER_WAIT` / `_READ/_WRITE/_FETCH/_INSERT/_UPDATE/_DELETE` 各统计族。**`INDEX_NAME IS NULL` 的那一行代表"全表扫描"**。

**`sys.schema_unused_indexes`**（`scripts/sys_schema/views/p_s/schema_unused_indexes.sql`）：

```sql
FROM performance_schema.table_io_waits_summary_by_index_usage t
INNER JOIN information_schema.statistics s ON ...
WHERE t.index_name IS NOT NULL      -- 排除全表扫描行
  AND t.count_star = 0              -- ★ 从未产生任何 IO 等待
  AND t.object_schema != 'mysql'
  AND t.index_name != 'PRIMARY'     -- 排除主键
  AND s.NON_UNIQUE = 1              -- 只保留非唯一二级索引
  AND s.SEQ_IN_INDEX = 1            -- 去重
```

**`sys.schema_redundant_indexes`**（`views/i_s/schema_redundant_indexes.sql`）三种冗余判定：

1. **列完全相同**且（非唯一度更高）或（非唯一度相同但索引名字典序更大）；
2. **非唯一前缀**：`LOCATE(CONCAT(redundant.index_columns, ','), dominant.index_columns) = 1` 且 `redundant.non_unique = 1`；
3. **唯一索引作为另一索引的前缀**：`LOCATE(CONCAT(dominant.index_columns, ','), redundant.index_columns) = 1` 且 `dominant.non_unique = 0`。

（尾逗号保证是完整列边界的前缀匹配，避免 `a` 误匹配 `ab`。）

**★★ 两个 sys 视图的数据源完全不同——这是最容易被搞混的一点**：

| 视图 | 数据源 | 是否需要"跑够久" | 依赖 |
|---|---|---|---|
| `schema_unused_indexes` | **PFS 行为数据** + `I_S.STATISTICS` 元数据 | **是**（否则全报"未使用"） | 需 PFS 开启且累积够久 |
| `schema_redundant_indexes` | **纯 `I_S.STATISTICS`**（经 `x$schema_flattened_keys` 的 `GROUP_CONCAT` 拼列） | **否**（随时可查） | **不碰 PFS** |

```sql
/* scripts/sys_schema/views/i_s/x_schema_flattened_keys.sql:33 —— 冗余视图的唯一数据源 */
  SELECT
    TABLE_SCHEMA, TABLE_NAME, INDEX_NAME,
    MAX(NON_UNIQUE) AS non_unique,
    MAX(IF(SUB_PART IS NULL, 0, 1)) AS subpart_exists,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS index_columns
  FROM INFORMATION_SCHEMA.STATISTICS
  WHERE INDEX_TYPE='BTREE'
    AND TABLE_SCHEMA NOT IN ('mysql','sys','INFORMATION_SCHEMA','PERFORMANCE_SCHEMA')
  GROUP BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME;
```

> ★ 因为只看 `INDEX_TYPE='BTREE'`，**FULLTEXT / SPATIAL 索引不参与冗余判定**；且对前缀索引只标 `subpart_exists=1` 提示人工复核。

**★ PFS 表的关键细节**（`storage/perfschema/table_tiws_by_index_usage.cc:47`）：

```cpp
    "  UNIQUE KEY `OBJECT` (OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME,\n"
    "                       INDEX_NAME) USING HASH\n",
    /* 【注】UNIQUE KEY 用 HASH —— PFS 是内存表，按 (type,schema,name,index) 哈希定位 */
```

> ★ `COUNT_INSERT/UPDATE/DELETE` 也计入——**无用索引的成本体现在 DML 维护上**，不只是查询没用到。
> ★ `&pfs_truncatable_acl` ⇒ 可 `TRUNCATE` 清零重新观察。

**`Handler_read_*` 状态变量**（`mysqld.cc:9786`，判断索引 vs 全表的经典依据）：

| 变量 | 含义 |
|---|---|
| `Handler_read_key` | 按 key 定位一行（等值/ref 访问） |
| `Handler_read_next` | 按索引顺序读下一行（range/index scan） |
| `Handler_read_rnd_next` | **全表扫描**（不看索引） |
| `Handler_read_first` / `_last` / `_prev` / `_rnd` | 索引首/尾/前一行/按 rowid 随机读 |

---

## 相关的系统变量/状态变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `innodb_ddl_threads` | **4**（1-64） | 并行构建索引的线程数；`<=1` 同步 |
| `innodb_ddl_buffer_size` | **1 MiB**（64KB-4GB） | 总量，均分给 `线程数 × 索引数` 套缓冲 |
| `innodb_online_alter_log_max_size` | **128 MB**（64KB-∞） | row log 上限；**超限 = 本次 DDL 失败**（索引标 corrupt），不是降级 |
| `innodb_sort_buffer_size` | 1 MiB | 同时是 row log 的 block 大小 |
| `innodb_tmpdir` | 无（用 tmpdir） | 排序临时文件与 row log 临时文件的目录 |
| `innodb_optimize_fulltext_only` | OFF | ON 时 `OPTIMIZE` 只做 FTS optimize，不重建表 |
| `information_schema_stats_expiry` | **86400**（24h） | `I_S`/`SHOW` 统计缓存过期；`0` = 每次实时取 |
| `sql_require_primary_key` | OFF | 开启后 InnoDB 建无 PK 表报 `ER_TABLE_WITHOUT_PK` |
| `sql_generate_invisible_primary_key` | OFF | GIPK（详见 [`gipk.md`](gipk.md)；类型谱系见 [`types.md`](types.md)） |
| `show_gipk_in_create_table_and_information_schema` | **ON** | 是否在 SHOW/I_S 显示 GIPK |

> ★ **GIPK 已独立成篇**：生成条件 / 物理形态 / 可见性 / DD 打标 / binlog 重写 / 复制 / ALTER 限制全部见 [`gipk.md`](gipk.md)（1048 行）。

**★ 补齐：统计与索引使用相关的其余变量**（★ 前表未列，但都直接影响本篇的可观测结论）

| 变量 | 文件:行 | 默认 | 说明 |
|---|---|---|---|
| `eq_range_index_dive_limit` | `sql/sys_vars.cc:3925` | **200** | ★ "use existing index statistics instead of doing index dives for equality ranges if the number of equality ranges ... >= this number. **If set to 0, index dives are always used.**"——这是"为什么有时候不做 index dive"的开关，与 Cardinality 强相关 |
| `innodb_stats_persistent` | `ha_innodb.cc:22210` | **true** | 持久化统计总开关（表级 `STATS_PERSISTENT` 可覆盖） |
| `innodb_stats_persistent_sample_pages` | `ha_innodb.cc:22224` | **20** | ANALYZE 采样页数 |
| `innodb_stats_transient_sample_pages` | `ha_innodb.cc:22203` | **8** | 非持久化统计的采样页数 |
| `innodb_stats_auto_recalc` | `ha_innodb.cc:22216` | **true** | 自动重算（表改动够多时） |
| `innodb_stats_include_delete_marked` | `ha_innodb.cc:21999` | **false** | 统计是否含 delete-marked 记录 |
| `innodb_stats_method` | `ha_innodb.cc:22977` | **nulls_equal** | NULL 如何参与统计（**直接决定 `rec_per_key` → Cardinality**） |
| `innodb_stats_on_metadata` | `ha_innodb.cc:22197` | **false** | ★ SHOW TABLE STATUS 等元数据命令是否重新采集统计 |
| `innodb_adaptive_hash_index` | `ha_innodb.cc:22231` | **true** | AHI；**关闭它会让 DROP INDEX 的延迟释放更快**（少一个引用源） |
| `innodb_ft_num_word_optimize` | `ha_innodb.cc:22524` | **2000** | 每次 OPTIMIZE 处理多少词 |
| `innodb_ft_cache_size` | `ha_innodb.cc:22496` | 8000000 | FTS cache（**READONLY**） |

```cpp
/* sql/sys_vars.cc:3925 —— 注意它是 HINT_UPDATEABLE */
static Sys_var_uint Sys_eq_range_index_dive_limit(
    "eq_range_index_dive_limit",
    "The optimizer will use existing index statistics instead of "
    "doing index dives for equality ranges if the number of equality "
    "ranges for the index is larger than or equal to this number. "
    "If set to 0, index dives are always used.",
    HINT_UPDATEABLE SESSION_VAR(eq_range_index_dive_limit),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, UINT_MAX32), DEFAULT(200),
    BLOCK_SIZE(1));
```

```cpp
/* ha_innodb.cc:462 —— innodb_stats_method 的三取值 */
static const char *innodb_stats_method_names[] = {
    "nulls_equal", "nulls_unequal", "nulls_ignored", NullS};
```

**★ `optimizer_switch` 中与索引直接相关的 flag**（`sql/sys_vars.cc:3445`）：

| flag | 作用 |
|---|---|
| `index_merge` / `_union` / `_sort_union` / `_intersection` | 索引合并（总开关 + 三种算法） |
| `index_condition_pushdown` | ICP |
| `use_index_extensions` | 二级索引自动扩展使用主键列 |
| `use_invisible_indexes` | 让优化器也考虑 INVISIBLE 索引（**只影响优化器，不影响 DML 维护**） |
| `skip_scan` | 松散索引扫描 |
| `prefer_ordering_index` | 有 `ORDER BY ... LIMIT` 时优先有序索引而非 filesort |
| `mrr` / `mrr_cost_based` / `batched_key_access` | 回表相关（MRR / BKA） |

> ⚠️ 源码一处小不一致：`optimizer_switch_names[]` 数组里有 `use_invisible_indexes`，但 `sys_vars.cc:3479-3487` 的对外注释串里**漏写了**它。

---

## Misc

### 索引 DDL 总表

| 操作 | 算法 | online | 关键函数 | 主要变量 |
|---|---|---|---|---|
| `ADD INDEX` | INPLACE | **是** | `ddl::Loader::build_all` → `Btree_load::build` → `row_log_apply` | `innodb_ddl_threads`、`innodb_ddl_buffer_size`、`innodb_online_alter_log_max_size` |
| `ADD SPATIAL` | INPLACE | 是（但 redo 不禁用、逐行插入） | `RTree_inserter` | 同上 |
| `ADD FULLTEXT` | INPLACE | 是 | `Builder::State::FTS_SORT_AND_BUILD` | 同上 |
| `DROP INDEX` | INPLACE | 是 | `write_free_tree_log` → post_ddl `replay_free_tree_log` → `btr_free_if_exists` | — |
| `RENAME INDEX` | INPLACE | 是 | `rename_index_in_cache` | — |
| `ALTER INDEX VISIBLE/INVISIBLE` | 纯 DD | **瞬时** | 无 InnoDB 实现 | — |
| `ADD/DROP PRIMARY KEY` | INPLACE（rebuild） | 是（但重建全表） | `row_log_table_apply` | `innodb_online_alter_log_max_size` |
| `OPTIMIZE TABLE` | recreate + analyze | 是 | `HA_ADMIN_TRY_ALTER` → `mysql_recreate_table` + `analyze` | `innodb_optimize_fulltext_only` |
| `ANALYZE TABLE` | — | 是 | `ha_innobase::analyze` → `dict_stats_update(PERSISTENT)` | 见 [`stats.md`](stats.md) |

### 可观测入口 × 数据来源 × 能回答什么

| 入口 | 数据来源 | 能回答 |
|---|---|---|
| `SHOW INDEX` / `I_S.STATISTICS` | DD `mysql.indexes` + `index_column_usage` | 有哪些索引、列顺序、是否唯一/降序/前缀/可见、功能索引表达式 |
| `Cardinality` | `stat_n_rows / rec_per_key`（有缓存） | 近似基数（注意 24h 缓存） |
| `mysql.innodb_index_stats` | `dict_stats_save` 写入 | **权威 n_diff / sample_size** |
| `I_S.INNODB_TABLESTATS` | 引擎内存 `dict_table_t` | 行数/页数、**统计新鲜度（MODIFIED_COUNTER）** |
| `I_S.INNODB_INDEXES` | 引擎内存 `dict_index_t` | TYPE 位掩码、root page、MERGE_THRESHOLD |
| PFS `table_io_waits_summary_by_index_usage` | PFS 事件聚合 | **索引到底被用过没**（`count_star=0`） |
| `sys.schema_unused_indexes` / `schema_redundant_indexes` | PFS + I_S | 无用索引 / 冗余索引清单 |
| `Handler_read_*` | server 状态变量 | 索引访问 vs 全表扫描的整体比例 |

### 易混淆点

- **`DROP INDEX` 不是 purge 回收**：purge 回收行记录，整树释放靠 post_ddl 重放 `FREE_TREE_LOG`。
- **`OPTIMIZE TABLE` 是 rebuild + analyze**，不是只 analyze；只统计陈旧用 `ANALYZE`。
- **`ALTER INDEX INVISIBLE` 瞬时**（纯 DD），但 **DML 仍完整维护该索引**。
- **`INNODB_INDEXES` 没有 `N_DIFF_FIELDS`**（5.7 已移除），基数在 `mysql.innodb_index_stats`。
- **`innodb_online_alter_log_max_size` 超限 = DDL 失败**，不是"自动降级为锁表"。
- **`SHOW INDEX` 的 Cardinality 有 24h 缓存**，`information_schema_stats_expiry=0` 才实时。
- **`DISABLE KEYS` 对 InnoDB 不生效**（只改 DD `keys_disabled`，索引照维护）。

**★ 本次源码核查新增的纠正**：

| 想当然 | 真相 |
|---|---|
| handler 返回 `HA_ALTER_INPLACE_*` 决定 COPY/INPLACE/INSTANT | 该枚举**根本没有 COPY**（`sql/handler.h:197`）。COPY 由 SQL 层在收到 `NOT_SUPPORTED` 时自己执行 |
| 存在 `HA_ALTER_INPLACE_RECREATE` | **不存在**。`EXCLUSIVE_LOCK` 存在但 **InnoDB 从不返回** |
| `ROW_LOG_MAX_SIZE` 宏 | **不存在**。上限是 `srv_online_max_size`（`srv0srv.cc:207`） |
| row log 超限立即报错 | **错**。写日志时只标 `DICT_CORRUPT`（建索引）或 `log->error`（重建表），报错在 apply/commit 才翻译成 `ER_INNODB_ONLINE_LOG_TOO_BIG` |
| row log 只服务 ADD INDEX | **错**。在线加二级索引与表重建**两套**，都受同一上限 |
| DROP INDEX 在 prepare 就删 | **错**。prepare 只置 `to_be_dropped=1`；commit 才写日志+清缓存；真释放 `btr_free_if_exists` 在 post_ddl |
| 有独立 post_ddl 后台线程 | **错**。是 handlerton 钩子 `innobase_post_ddl`，由 SQL 层同步调用；日志按 `thread_id` 分片 |
| `dict_index_remove_from_cache` 在 `dict0crea.cc` | **不在**。定义在 `dict0dict.cc:2644` |
| INSTANT 也能加索引 | **不能**。`INNOBASE_INSTANT_ALLOWED`（`handler0alter.cc:149`）不含任何索引位 |
| OPTIMIZE TABLE = InnoDB 内部重建 | **错**。`ha_innobase::optimize()` 默认只回 `HA_ADMIN_TRY_ALTER`，由 SQL 层去 `mysql_recreate_table()` + `analyze` |
| Cardinality = 字典里存的 distinct 值数 | **错**。是 `round(stat_n_rows / rec_per_key)` 的运行时计算；"≈ n_diff" 只在默认 `nulls_equal` 且非前缀/非 FULLTEXT 时成立 |
| SHOW INDEX 会触发统计重算 | **默认不会**。持久化统计走 `FETCH_ONLY_IF_NOT_IN_MEMORY`，`stat_initialized` 为真时 `return DB_SUCCESS` 什么都不做 |
| `sys.schema_redundant_indexes` 来自 PFS | **错**。纯 `I_S.STATISTICS`；只有 `schema_unused_indexes` 依赖 PFS |
| 存在 `INNODB_SYS_INDEXES` 等 | **不存在**（8.0 去 `SYS_` 前缀）；`Innodb_num_indexes` 状态变量也**不存在** |
| `Using index for order by` / `Scanned N databases` | 这些是**索引 EXPLAIN 侧**的历史遗留说法，8.0.39 已不存在（详见 [`../infra/observability/explain_trace.md`](../infra/observability/explain_trace.md)） |

### 一句话总结

索引的"生"是扫描 + 排序 + 批量装载 + row log 回放四条流水并行，"死"是标记 + DDL log 延迟到提交后释放，变更多为纯字典操作；观测则要分清三层——DD 元数据看定义、引擎内存看统计与页数、PFS 看实际使用。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.12 InnoDB and Online DDL*
- *MySQL 8.0 Reference Manual → 15.6.2.6 Optimizing InnoDB DDL Operations*
- *MySQL 8.0 Reference Manual → 27.4 The INFORMATION_SCHEMA STATISTICS Table*
- *MySQL 8.0 Reference Manual → 28 MySQL sys Schema*

**相关文档**
- online DDL 通用框架（row log、DDL log）：[`../feat/ddl/ddl.md`](../feat/ddl/ddl.md)
- 批量建页 `Btree_load` 算法：[`btr.md`](btr.md)
- 统计采样与 `rec_per_key`：[`stats.md`](stats.md)
- 索引类型（不可见索引、GIPK、降序/前缀/功能/多值）：[`types.md`](types.md)
- 索引访问优化：`access.md`
- 优化器侧：[`../server/query/07_optimize/`](../server/query/07_optimize/)
