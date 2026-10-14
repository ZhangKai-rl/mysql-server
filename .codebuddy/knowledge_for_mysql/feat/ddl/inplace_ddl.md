# INPLACE DDL 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，剖析 `ALGORITHM=INPLACE` 这一路：server 侧的 MDL 升降级与四件套回调契约、引擎侧 `check_if_supported_inplace_alter` 的六档返回值如何直接决定锁级别、以及 online 的支柱——row log（为什么 apply 两次、第二个短暂阻塞点从哪来）。
>
> **边界**：本篇**写完两层**——server 怎么问引擎、按返回值升降锁，引擎怎么在并发 DML 下重建表。"索引具体是怎么被构建出来的"属执行引擎，见 [`ddl_engine.md`](ddl_engine.md)。算法选择策略与 MDL 三段式的公共骨架见 [`ddl.md`](ddl.md)；另两条路见 [`copy_ddl.md`](copy_ddl.md)、[`instant_ddl.md`](instant_ddl.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路：四件套三阶段](#主链路四件套三阶段)
  - [引擎接口：handler / handlerton 如何承载 DDL](#引擎接口handler--handlerton-如何承载-ddl)
  - [ha_extra：DDL 给引擎下的"行为开关"](#ha_extraddl-给引擎下的行为开关)
  - [online DDL 与 row log（引擎侧）](#online-ddl-与-row-log引擎侧)
  - [回放时机：apply 两次（并发控制根源是 MDL 锁）](#回放时机apply-两次并发控制根源是-mdl-锁)
  - [commit 与 rename（仅 rebuild 场景）](#commit-与-rename仅-rebuild-场景)
  - [索引在线状态机与"僵尸索引"](#索引在线状态机与僵尸索引)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`ALGORITHM=INPLACE`：**把"改表"搬进引擎内部执行**——不在 server 层建临时表、不逐行走 SQL 层 INSERT，而是引擎自己扫描旧结构、构建新结构、最后原子切换。它是大多数 ALTER 的默认选择（server 判定顺序 INSTANT → INPLACE → COPY）。

### 用途

- 承接 INSTANT 做不了的变更：加/删二级索引、rebuild 重建（`OPTIMIZE`、`ENGINE=InnoDB`、加主键、改列类型）、加/删列（8.0 大部分场景已 INSTANT 化，但列位置等仍 INPLACE）等。
- **online 的载体**：INPLACE 是三种算法中唯一支持"执行期并发 DML"的非瞬时算法——靠 MDL 降级 + row log 实现。生产环境大表变更的主力路径。

### 版本演进

⚠️ **勘误（8.0 大重构）**：
- **`row0merge.cc` / `row_merge_create_index()` / `row_merge_build_indexes()` / `row_merge_drop_temp_indexes()` 全部不存在**——8.0 已迁到 `storage/innobase/ddl/`：`ddl::create_index()`（`ddl0ddl.cc:184`）、`ddl::Context`（`include/ddl0ddl.h:321`）、`ddl::Loader`（`include/ddl0impl-loader.h:44`）、`ddl::Builder`（`include/ddl0impl-builder.h:48`）
- **`ha_innobase::add_index()` / `drop_index()` 不存在**——ADD INDEX 一律走 prepare/inplace/commit 三阶段
- **`include/ddl0ctx.h` 文件不存在**（只有 `ddl/ddl0ctx.cc`）
- **`row_log_get_error()` 不存在**（只有 `row_log_table_get_error()`）

**三代实现的对照**：

| 代 | 建索引主体 | 排序 | 文件 |
|---|---|---|---|
| 5.5 | `row_merge_build_indexes()` | 单线程 merge sort | `row/row0merge.cc` |
| 5.6/5.7 | `row_merge_build_indexes()` | 支持 `innodb_sort_buffer_size` | `row/row0merge.cc` |
| **8.0** | **`ddl::Context::build()` → `Loader::build_all()` → `Builder`** | **多线程并行（`innodb_ddl_threads`）** | `ddl/ddl0ctx.cc:512`、`ddl/ddl0loader.cc`、`ddl/ddl0builder.cc` |

```cpp
// storage/innobase/ddl/ddl0ctx.cc:512 —— 8.0 的建索引总入口
dberr_t Context::build() noexcept {
  Loader loader{*this};                      // ★ Context 内部直接构造 Loader
  const auto err = cleanup(loader.build_all());
  /* Validate the indexes after the pages have been flushed to disk.
  Otherwise we can deadlock between flushing and is_free page check. */
  ut_ad(err != DB_SUCCESS || loader.validate_indexes());
  return err;
}
```

**`HTON_*` 标志的演进**（`sql/handler.h:2864`）——**共 21 个有效位 + 1 个废弃位**：

```cpp
/* Possible flags of a handlerton (there can be 32 of them) */
#define HTON_NO_FLAGS 0
#define HTON_CLOSE_CURSORS_AT_COMMIT (1 << 0)
#define HTON_ALTER_NOT_SUPPORTED (1 << 1)  // Engine does not support alter
#define HTON_CAN_RECREATE (1 << 2)         // Delete all is used for truncate
#define HTON_HIDDEN (1 << 3)               // Engine does not appear in lists
/*
  Bit 4 was occupied by BDB-specific HTON_FLUSH_AFTER_RENAME flag and is no
  longer used.
*/
#define HTON_NOT_USER_SELECTABLE (1 << 5)
#define HTON_TEMPORARY_NOT_SUPPORTED (1 << 6)
#define HTON_SUPPORT_LOG_TABLES (1 << 7)
#define HTON_NO_PARTITION (1 << 8)
#define HTON_NO_BINLOG_ROW_OPT (1 << 9)
#define HTON_SUPPORTS_EXTENDED_KEYS (1 << 10)
#define HTON_SUPPORTS_FOREIGN_KEYS (1 << 11)

/**
  Engine supports atomic DDL. That is rollback of transaction for DDL
  statement will also rollback all changes in SE, commit of transaction
  of DDL statement will make it durable.
*/
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12)   // ★ DDL 原子性开关

#define HTON_SUPPORTS_PACKED_KEYS (1 << 13)
#define HTON_IS_SECONDARY_ENGINE (1 << 14)
#define HTON_SUPPORTS_SECONDARY_ENGINE (1 << 15)
#define HTON_SUPPORTS_TABLE_ENCRYPTION (1 << 16)
constexpr const decltype(handlerton::flags) HTON_SUPPORTS_ENGINE_ATTRIBUTE{1 << 17};
constexpr const decltype(handlerton::flags) HTON_SUPPORTS_GENERATED_INVISIBLE_PK{1 << 18};
#define HTON_SECONDARY_ENGINE_SUPPORTS_DDL (1 << 19)
#define HTON_NO_TRIGGER_SUPPORT (1 << 20)
#define HTON_SUPPORTS_EXTERNAL_SOURCE (1 << 21)   // ★ 8.0.39 最高位

inline bool ddl_is_atomic(const handlerton *hton) {
  return (hton->flags & HTON_SUPPORTS_ATOMIC_DDL) != 0;
}
```

> ★★ **`bit4` 是空洞**（原 BDB 的 `HTON_FLUSH_AFTER_RENAME`，注释明写 "no longer used"）。**`HTON_ALTER_NOT_SUPPORTED` / `HTON_CAN_RECREATE` 都还在**，不是"老名字已删"。
>
> ⚠️ **另有一组独立的外键能力位**（不是 `hton->flags`，是 `foreign_keys_flags`）：`sql/handler.h:2973-3015`（`HTON_FKS_WITH_PREFIX_PARENT_KEYS` … `HTON_FKS_WITH_EXTENDED_PARENT_KEYS`，bit0~4）。

| 版本 | 变化 |
|------|------|
| 5.1（InnoDB Plugin） | fast index create：建索引不拷贝全表，但**期间锁写**、且不拷贝 undo 历史（只有建完之后启动的事务能用新索引） |
| 5.6 | online DDL 落地：[WL#5526](https://dev.mysql.com/worklog/task/?id=5526)（online ADD INDEX，DML 可并发）+ WL#5534（**新 ALTER API**，即四件套三段式）+ WL#5548（online DROP INDEX）+ WL#5854（online ADD/DROP COLUMN）。`innodb_online_alter_log_max_size`、索引在线状态机、row log 均在此代引入 |
| 8.0 | **构建方式换代**：新 DDL 引擎（`storage/innobase/ddl/`）以"扫描—排序—建树"三期 + 并行构建取代 5.6 的 merge sort 路线；**row log 机制原样沿用**——并发控制的设计经受住了两代实现的考验 |
| 8.0.27+ | 并行 DDL（社区版可用），构建进一步加速 |

> 值得注意的设计连续性：WL#5526 设计的 `row_log_allocate → 扫描 → 排序 → 建树 → row_log_apply → MDL 升级 → commit_inplace_alter_table` 七步骨架，8.0 只把中间"扫描/排序/建树"换成了新引擎，首尾的 online 协议未变。

---

## 理论基础

### 设计思想与权衡

**要解决的问题**：COPY 有两大硬伤——全程 MDL X（不可并发）与逐行 SQL 层开销。INPLACE 的回答是**把执行搬进引擎内部**：引擎知道自己怎么建树最快（bulk load 而非逐行），并且引擎内部有日志机制（row log）能兜住执行期的并发 DML，从而把 MDL 从"全程 X"压缩为"头尾两个短窗口 X"。

**"本来可以那样做，但没有"的对比**：并发正确性理论上也能在 server 层解决——订阅 binlog 增量回放（这正是后来 gh-ost 的方案）。官方选择引擎内方案的根本原因：**row log 的 apply 可以带完整事务语义**（回放时以 REPEATABLE READ 快照判定冲突，能区分"该回放的"与"假重复键"），而 server 层的外部方案只能做到最终一致；且引擎内切换（rename / 状态机翻转）天然原子，不需要 cut-over 魔法锁。代价是这套复杂度（row log、状态机、僵尸索引）必须每个引擎自己实现——通用性让位给性能，这正是不支持 INPLACE 的引擎仍要走 COPY 的原因。

**代价与失效场景**：

- 实现复杂度高：状态机、僵尸索引、两种 row log 形态（见下）都是为 online 付出的复杂度。
- **row log 积压风险**：执行期越长、写越忙，row log 越大；超出 `innodb_online_alter_log_max_size`（默认 128MB）整个 DDL 失败——大表 + 高写入是 INPLACE online 的失效场景。
- 第二次 apply 与 rename 必须在 MDL X 下连做（见「回放时机」），这是 online 窗口收尾处不可避免的一个短暂阻塞。

### 算法与数据结构

| 机制 | 落点 | 为什么这么选 |
|---|---|---|
| `row_log_t` | 挂 `dict_index_t::online_log`，**内存块 + 磁盘临时文件两段式** | 生产者（并发 DML）-消费者（apply）模型；内存块写满溢出磁盘，兼顾低延迟与无限积压 |
| mrec 紧凑编码 | record data 用 **temp 格式**（省略记录头，只留变长长度列表 + NULL 位 + 数据） | row log 可能很大，格式上省一个字节是一个字节；同一格式复用于 sort buffer |
| `online_index_status` 四态状态机 | `dict_index_t::online_status` | 用显式状态区分"可用/建设中/放弃/已删"，支撑失败路径的懒清理（见「状态机」） |
| `enum_alter_inplace_result` 六档返回值 | `check_if_supported_inplace_alter` | 引擎用一个函数同时回答"能不能 INPLACE"与"需要什么锁级别"——server 据此定 MDL 策略 |

---

## 核心实现

### INPLACE 的语义边界与两种子形态

**★ 判定核心：`innobase_need_rebuild()`**（`handler0alter.cc:922`）——**INSTANT 永不 rebuild**：

```cpp
/** Determine if ALTER TABLE needs to rebuild the table.
@param[in]      ha_alter_info   The DDL operation
@return whether it is necessary to rebuild the table */
[[nodiscard]] static bool innobase_need_rebuild(
    const Alter_inplace_info *ha_alter_info) {
  if (is_instant(ha_alter_info)) {          // ★ INSTANT 优先短路
    return (false);
  }

  Alter_inplace_info::HA_ALTER_FLAGS alter_inplace_flags =
      ha_alter_info->handler_flags & ~(INNOBASE_INPLACE_IGNORE);

  if (alter_inplace_flags == Alter_inplace_info::CHANGE_CREATE_OPTION &&
      !(ha_alter_info->create_info->used_fields &
        (HA_CREATE_USED_ROW_FORMAT | HA_CREATE_USED_KEY_BLOCK_SIZE |
         HA_CREATE_USED_TABLESPACE))) {
    /* Any other CHANGE_CREATE_OPTION than changing
    ROW_FORMAT, KEY_BLOCK_SIZE or TABLESPACE can be done
    without rebuilding the table. */
    return (false);                         // ★ 只有这三项才触发 rebuild
  }

  return (!!(ha_alter_info->handler_flags & INNOBASE_ALTER_REBUILD));
}

static inline bool is_instant(const Alter_inplace_info *ha_alter_info) {
  return (ha_alter_info->handler_trivial_ctx !=
          instant_type_to_int(Instant_Type::INSTANT_IMPOSSIBLE));
  //  ★ handler_trivial_ctx 是 uint，InnoDB 用它存 Instant_Type
}
```

**两个"rebuild 判定"如何对齐**（`handler0alter.cc:4945`）——**由断言绑死**：

```cpp
  bool new_clustered = DICT_CLUSTERED & index_defs[0].m_ind_type;
  //  ★ 第二形态：新索引定义[0] 是否为聚簇（server 侧先算）
  ...
  assert(!add_fts_doc_id || new_clustered);
  assert(new_clustered ==
         (innobase_need_rebuild(ha_alter_info) || add_fts_doc_id));
  //  ★★ 关键不变式：add_fts_doc_id 会额外强制 rebuild
  ...
  ut_ad(new_clustered == ctx->need_rebuild());
```

**`ctx->need_rebuild()` 的真正定义**（`handler0alter.cc:174`）——**不是查 flag，是比对象身份**：

```cpp
struct ha_innobase_inplace_ctx : public inplace_alter_handler_ctx {
  ...
  /** NOTE: Determine if the table will be rebuilt.
  @return whether the table will be rebuilt */
  bool need_rebuild() const { return (old_table != new_table); }
  //  ★★ rebuild 判定的唯一真源：新旧 dict_table_t 是否是同一个对象

  dict_table_t *old_table;   // original table (if rebuilt, differs from indexed_table)
  dict_table_t *new_table;   // table where the indexes are being created or dropped
  dict_index_t **add_index;  const ulint *add_key_numbers;  ulint num_to_add_index;
  dict_index_t **drop_index; const ulint num_to_drop_index;
  dict_index_t **rename;     const ulint num_to_rename;
  dict_foreign_t **drop_fk;  const ulint num_to_drop_fk;
  dict_foreign_t **add_fk;   const ulint num_to_add_fk;
  bool online;               // whether to create the indexes online
  trx_t *trx;
  const ulint *col_map;      // mapping of old column numbers to new ones, or NULL
  const ulint add_autoinc;   // added AUTO_INCREMENT column position, or ULINT_UNDEFINED
  ddl::Sequence sequence;
  bool skip_pk_sort;         // whether the order of the clustered index is unchanged
  Alter_stage *m_stage;      // ALTER TABLE stage progress recorder
  ...
  ~ha_innobase_inplace_ctx() override {
    if (fts_drop_aux_vec != nullptr) { fts_free_aux_names(fts_drop_aux_vec); delete fts_drop_aux_vec; }
    ut::delete_(m_stage);
    mem_heap_free(heap);
  }
```

> ⚠️ 真名是 **`ha_innobase_inplace_ctx`**（小写 h），不是 `Ha_innobase_inplace_ctx`。
>
> ★ **两个创建点**：①「轻量 ctx」分支（`new_table == indexed_table`）→ `need_rebuild()==false`；② rebuild 分支（:6016）→ 先传 `m_prebuilt->table`，**`prepare_inplace_alter_table_dict()` 里把 `ctx->new_table` 换成新建对象**（:4639-4786 之间 `dict_mem_table_create()`），从此 `need_rebuild()` 为真。

**轻量分支（无需动数据 / 只改 CREATE OPTION 且不 rebuild）**（`handler0alter.cc:5875`）：

```cpp
  /* ★ add index 形态 vs rebuild 形态的分岔点 */
  if (!(ha_alter_info->handler_flags & INNOBASE_ALTER_DATA) ||
      ((ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) ==
           Alter_inplace_info::CHANGE_CREATE_OPTION &&
       !innobase_need_rebuild(ha_alter_info))) {
    if (heap) {
      ha_alter_info->handler_ctx = new (m_user_thd->mem_root)
          ha_innobase_inplace_ctx(m_prebuilt, drop_index, n_drop_index,
                                  rename_index, n_rename_index, drop_fk,
                                  n_drop_fk, add_fk, n_add_fk,
                                  ha_alter_info->online, heap, indexed_table,
                                  /* new_table 传 indexed_table ⇒ need_rebuild()==false */
                                  col_names, ULINT_UNDEFINED, 0, 0);
    }
    assert(m_prebuilt->trx->dict_operation_lock_mode == 0);
    if (ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) {
      online_retry_drop_dict_indexes(m_prebuilt->table, false);
      //  ★ 清掉上次残留的 ABORTED 索引
    }
    ...
    return dd_prepare_inplace_alter_table(m_user_thd, ctx->old_table,
                                          ctx->new_table, old_dd_tab);
  }
```

**★ 原子切换点：`commit_cache_rebuild(ctx)`**（`handler0alter.cc:7688`）——表名互换 + 旧表打 `to_be_dropped`：

```cpp
    if (new_clustered) {
      /* We will reload and refresh the in-memory foreign key constraint
      metadata. This is a rename operation in preparing for dropping the old
      table. Set the table to_be_dropped bit here, so to make sure DML foreign
      key constraint check does not use the stale dict_foreign_t. This is done
      because WL#6049 (FK MDL) has not been implemented yet. */
      ctx->old_table->to_be_dropped = true;

      if ((ha_alter_info->handler_flags &
           Alter_inplace_info::ALTER_COLUMN_NAME)) {
        innobase_rename_col_discard_foreign(ha_alter_info, table,
                                            ctx->old_table);
      }

      /* Rename the tablespace files. */
      commit_cache_rebuild(ctx);            // ★★ 新旧表互换（原子切换）
      ...
    } else {
      if (!commit_cache_norebuild(ctx, trx)) {   // ★ add-index：把新索引"发布"进 dict cache
        ut_a(!m_prebuilt->trx->check_foreigns);
      }
      innobase_rename_or_enlarge_columns_cache(ha_alter_info, table, ctx->new_table);
      rename_indexes_in_cache(ctx, ha_alter_info);
    }
```

**commit 后的三条不变式断言**（`handler0alter.cc:7970`）：

```cpp
#ifdef UNIV_DEBUG
  dict_index_t *clust_index = ctx0->prebuilt->table->first_index();
  assert(!clust_index->online_log);          // ★① commit 后 row log 必须已清空
  assert(dict_index_get_online_status(clust_index) == ONLINE_INDEX_COMPLETE);
  //                                         // ★② 聚簇索引必须回到 COMPLETE
  for (dict_index_t *index = clust_index; index; index = index->next()) {
    assert(!index->to_be_dropped);           // ★③
  }
#endif /* UNIV_DEBUG */
```

**rebuild 后 prebuilt 的重建**（`handler0alter.cc:7955`）：

```cpp
      row_prebuilt_free(ctx->prebuilt, true);            // ★ 旧 prebuilt 作废
      /* Drop the copy of the old table, which was renamed to ctx->tmp_name
      at the atomic DDL transaction commit. If the system crashes before this
      is completed, some orphan tables with ctx->tmp_name may be recovered. */
      ddl::drop_table(trx, ctx->old_table);              // ★ 删掉改名后的旧表
      ctx->prebuilt =
          row_create_prebuilt(ctx->new_table, altered_table->s->reclength);
      if (update_own_prebuilt) m_prebuilt = ctx->prebuilt;  // ★ handler 指向新表
```

> ★ `ctx->tmp_name` 的生成与 MDL 获取（:7576）：`dict_mem_create_temporary_tablename()` 后立刻 `dd::acquire_exclusive_table_mdl()`——**这是"改名"能原子完成的锁前提**。

这是 INPLACE 最容易误解的一点：**INPLACE 不等于"不新建表"**。它实际包含两个子形态，其中一个同样要建新表、同样要 rename：

| 子形态 | 典型语句 | 是否新建表对象 | 是否有表级 rename | 数据搬迁发生在哪 |
|---|---|---|---|---|
| **no-rebuild** | `ADD INDEX`（二级索引）、`DROP INDEX`、改表名、改表选项 | ❌ 原表上直接增减索引 | ❌ 无 | 引擎内扫描聚簇索引 → 排序 → 建**索引树** |
| **rebuild** | 改列类型/长度、改列顺序、`ADD/DROP PRIMARY KEY`、改行格式、`ALTER ... ENGINE=xx`（OPTIMIZE） | ✅ 建 `ctx->new_table` | ✅ 有 | 引擎内扫描原表 → 建**整张新表（含新聚簇索引）** → 切换 |

源码自己的措辞就是证据——`handler0alter.cc` 里那组 flag 的注释原话是 **"Operations for rebuilding a table in place"**：*原地重建表*。也就是说，"重建表"这件事本身就属于 INPLACE 范畴：

```cpp
/** Operations for rebuilding a table in place */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_REBUILD =
    Alter_inplace_info::ADD_PK_INDEX | Alter_inplace_info::DROP_PK_INDEX |
    Alter_inplace_info::CHANGE_CREATE_OPTION
    /* CHANGE_CREATE_OPTION needs to check innobase_need_rebuild() */
    | Alter_inplace_info::ALTER_COLUMN_NULLABLE |
    Alter_inplace_info::ALTER_COLUMN_NOT_NULLABLE |
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN
    | Alter_inplace_info::RECREATE_TABLE;
```

注意 **加二级索引（`ADD_INDEX`）不在这个列表里**——所以它走 no-rebuild，真的不涉及新表与表级 rename。

**为什么 rebuild 不能"原地改"而必须建新表？** 因为聚簇索引的**物理记录布局**变了：列类型/长度一改，每行记录的字段偏移就变；列顺序一改，字段在记录里的先后就变；加主键则整个 B-tree 的键结构都变。已存在的页没法原地重排（页内记录、页间关系都要重算），只能重新扫描原表、按新定义构建一棵新的聚簇索引，再整体替换。

**那 INPLACE 与 COPY 的区别到底在哪？** 不在"有没有新表"，而在**数据搬迁发生在哪一层**：

- **COPY**：server 层建 `#sql-xxx` 临时表，用 `copy_data_between_tables` **逐行走 `ha_write_row`（完整 SQL 写入路径）**——要经过 server 的写入逻辑、生成列求值、CHECK 约束等；全程 MDL X，**必然不 online**。
- **INPLACE rebuild**：引擎内部建 `ctx->new_table`（一个 `dict_table_t`），用 `ddl::Builder` **直接扫描聚簇索引 + 排序 + 建 B-tree**，完全不经过 SQL 层；期间 MDL 可降级为 S/SU，DML 并发写 row log，**可以 online**。

⇒ 所以"INPLACE"的准确含义是：**引擎内部完成、不经过 server 层复制**，而不是"绝对不动表对象"。理解了这一点，就不会对「INPLACE 里出现 `new_table` 和 rename」感到困惑了——它们只出现在 rebuild 子形态；no-rebuild 子形态（如加索引）确实是原表上操作，没有表级 rename。

### ★ 那 INPLACE + rebuild 与 COPY 有什么区别？

**一句话**：**COPY 要过 server 层的"建临时表 → 逐行拷 → rename"，INPLACE+rebuild 全程在 InnoDB 内部**，不产生 server 层临时表、不写 undo、不逐行过 SQL 层。

| 维度 | COPY | INPLACE + rebuild |
|---|---|---|
| 临时表 | server 层 `#sql-xxx`，走 `mysql_create_table()` | **InnoDB 内部** `dict_mem_create_temporary_tablename()`，DD 里看不见 |
| 逐行拷贝 | `copy_data_between_tables()`，`ha_start_bulk_insert` | `Loader::scan_and_build_indexes()` → `Builder` 并行扫描/排序/bulk load |
| **undo** | **写**（有 `HA_EXTRA_BEGIN_ALTER_COPY` 关掉的例外） | **不写**（`skip_alter_undo`） |
| 并发 DML | 全程禁止（MDL-X / SNW） | 主阶段允许（降级到 SU/SNW） |
| row log | 无 | **有**，且回放两次 |
| binlog | `end_inplace_noop` 处写（:18174） | `mysql_inplace_alter_table()` 内写（:13766） |
| 时机 | `mysql_alter_table()` 尾部 | `mysql_inplace_alter_table()` 内部 |

**★ COPY 关 undo 的两个 extra**（`sql/sql_table.cc:18619` / `:18726`）——这是 COPY 唯一的"优化"：

```cpp
  to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY);   // ★ 拷贝循环开始
  //  ... while (!(error = iterator->Read())) 逐行拷贝 ...
  to->file->ha_extra(HA_EXTRA_END_ALTER_COPY);     // ★ 拷贝循环结束
```

```cpp
// storage/innobase/handler/ha_innodb.cc:18350 —— InnoDB 的实现
    case HA_EXTRA_BEGIN_ALTER_COPY:
      m_prebuilt->table->skip_alter_undo = 1;      // ★ 中间表插入不写 undo
      break;
    case HA_EXTRA_END_ALTER_COPY:
      alter_stats_rebuild(m_prebuilt->table, m_prebuilt->table->name.m_name,
                          m_user_thd);             // ★ 拷完立刻重算统计
      m_prebuilt->table->skip_alter_undo = 0;      // ★ 恢复
      break;
```

> ⚠️ **INPLACE 路径不发送这两个 extra**——它们只在 COPY（`copy_data_between_tables()`）中使用。

> ⚠️ **`handler::ha_extra()` 基本不拦**（`sql/handler.cc:8490`）——只拦 **2 个**多值索引去重 filter（`HA_EXTRA_ENABLE/DISABLE_UNIQUE_RECORD_FILTER`），其余 ~40 个 `HA_EXTRA_*` **原样** `return extra(operation);` 下发给引擎。**`HA_EXTRA_FORCE_REOPEN` / `HA_EXTRA_PREPARE_FOR_RENAME` 都是透传**。
>
> ⚠️ **InnoDB 的 `extra()` 只处理 14 个**，其余走 `default: /* Do nothing */`。**`HA_EXTRA_FORCE_REOPEN` 和 `HA_EXTRA_PREPARE_FOR_RENAME` 在 InnoDB 里被静默忽略**——它们的作用是 server 侧 `wait_while_table_is_used()` 的"意图标记"，不是给引擎的指令。
>
> ⚠️ `enum ha_extra_function` 在 **`include/my_base.h:184`**，不在 `sql/handler.h`。**DDL 专用的只有 2 个**：`HA_EXTRA_BEGIN_ALTER_COPY` / `HA_EXTRA_END_ALTER_COPY`（`my_base.h:411-414`）。

**`old_alter_table` —— 唯一的 COPY 强制开关**（`sql/sys_vars.cc:3288`）：

```cpp
static Sys_var_bool Sys_old_alter_table("old_alter_table",
                                        "Use old, non-optimized alter table",
                                        SESSION_VAR(old_alter_table),
                                        CMD_LINE(OPT_ARG), DEFAULT(false));
```

> ★ **`old_alter_table=ON` 不能覆盖显式 `ALGORITHM=INPLACE` / `ALGORITHM=INSTANT`**——会报 `ER_ALTER_OPERATION_NOT_SUPPORTED`（`sql_table.cc:16977` 的判据里显式排除了这两种）。

既然两者都是"建新表 + 搬数据 + rename"，很容易觉得等价。**但搬迁路径完全不同**，由此带出一串后果差异：

| 维度 | COPY | INPLACE + rebuild |
|---|---|---|
| **新表是什么** | server 层的真实表对象（`#sql-xxx`，有 `TABLE_SHARE`、可被 SQL 访问），由 `create_table_impl` 建 | 引擎内的 `dict_table_t`（`ctx->new_table`），**不暴露给 SQL 层** |
| **数据怎么搬** | `copy_data_between_tables` **逐行 `ha_write_row()`**（`sql_table.cc`），走完整 handler 写入路径，每插入一行要同步维护**所有**索引 | `ddl::Builder` 直接扫描原聚簇索引 → 排序 → **`Btree_load` 批量建树**（每棵索引独立构建） |
| **SQL 层语义** | 逐行走 server 写入逻辑：生成列求值、CHECK 约束、自增分配、逐行唯一性检查 | 物理构建：排序阶段统一检测重复键（见 [`ddl_engine.md`](ddl_engine.md)），不走 server 写入路径 |
| **MDL / online** | 全程 **MDL X**，DML 全程阻塞，**必然不 online** | 执行阶段可降到 S/SU，**DML 并发**，靠 row log 兜住 |
| **索引构建效率** | 每行插入都要更新全部索引 ⇒ 随机写、写放大 | 先排序再批量建树 ⇒ 顺序写，效率高得多 |
| **谁在驱动** | server（`mysql_alter_table`）全程编排 | 引擎四件套，server 只做 MDL 升降级与调用 |
| **定位** | **兜底路径**：引擎不支持 INPLACE 时才退到它（改字符集、部分分区操作等） | 引擎支持时的首选 |

⇒ 一句话概括：**COPY 是"server 层逐行重插"，INPLACE rebuild 是"引擎内扫描重建"。** 两者都需要新表和 rename，但前者的成本（SQL 层逐行、全程 X 锁、索引随机写）远高于后者，这也是优化器/引擎优先选择 INPLACE 的原因。

### 主链路：四件套三阶段

四件套是 MDL 三段式（见 [`ddl.md`](ddl.md)）在引擎侧的实现载体，server 与引擎的协作全程：

```
mysql_alter_table（server）
  ├─ check_if_supported_inplace_alter()     引擎只答一个问题：能不能 INPLACE、要什么锁
  │     → enum_alter_inplace_result 六档 → server 据此决定 MDL 策略（是否可降级）
  ├─ prepare_inplace_alter_table()          ① MDL X：建 new_table 元数据、分配 row log
  ├─ MDL 降级（X → SHARED/None）             ★ online 窗口开启，DML 放行
  ├─ inplace_alter_table()                  ② MDL S/NONE：扫描-排序-建树，DML 并发写 row log
  ├─ MDL 升级（→ EXCLUSIVE）                 ★ 等待所有持有旧定义的事务结束
  └─ commit_inplace_alter_table()           ③ MDL X：apply row log 追平 + rename 切换 + 统计
```

**六档返回值 → MDL 级别**的完整对照表在 [`ddl.md`](ddl.md)「算法判定的真正链条」；要点：`HA_ALTER_INPLACE_NO_LOCK*` 才是 online，带 `*_AFTER_PREPARE` 后缀的表示 prepare 阶段要 X、之后可降级。

#### server 侧：`mysql_inplace_alter_table` 的锁升级（MDL 三段式的 server 侧实现）

前面讲的降级在引擎侧（`downgrade_lock`），**升级在 server 侧**——`mysql_inplace_alter_table` 开头就是：

```cpp
/*
  Upgrade to EXCLUSIVE lock if:
  - This is requested by the storage engine
  - Or the storage engine needs exclusive lock for just the prepare phase
  - Or requested by the user

  Note that we handle situation when storage engine needs exclusive
  lock for prepare phase under LOCK TABLES in the same way as when
  exclusive lock is required for duration of the whole statement.
*/
if (inplace_supported == HA_ALTER_INPLACE_EXCLUSIVE_LOCK ||
    ((inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
      inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) &&
     (thd->locked_tables_mode == LTM_LOCK_TABLES ||
      thd->locked_tables_mode == LTM_PRELOCKED_UNDER_LOCK_TABLES)) ||
    alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE) {
  if (wait_while_table_is_used(thd, table, HA_EXTRA_FORCE_REOPEN))
    goto cleanup;
  /*
    Get rid of all TABLE instances belonging to this thread
    except one to be used for in-place ALTER TABLE.

    This is mostly needed to satisfy InnoDB assumptions/asserts.
  */
  close_all_tables_for_name(thd, table->s, false, table);
  reopen_tables = true;
} else if (inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
           inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) {
  /*
    Storage engine has requested exclusive lock only for prepare phase
    and we are not under LOCK TABLES.
    Don't mark TABLE_SHARE as old in this case, as this won't allow opening
    of table by other threads during main phase of in-place ALTER TABLE.
  */
  if (thd->mdl_context.upgrade_shared_lock(table->mdl_ticket, MDL_EXCLUSIVE,
                                           thd->variables.lock_wait_timeout))
    goto cleanup;

  tdc_remove_table(thd, TDC_RT_REMOVE_NOT_OWN_KEEP_SHARE, table->s->db.str,
                   table->s->table_name.str, false);
}
```

★ 三个要点：

1. **升级到 X 的三种情形**：引擎要求全程 X；或引擎只要求 prepare 阶段 X **但当前处于 `LOCK TABLES` 下**（此时按全程 X 处理）；或用户显式 `LOCK=EXCLUSIVE`。
2. `close_all_tables_for_name()` 的注释很直白：*"mostly needed to satisfy InnoDB assumptions/asserts"*——把本线程除一个之外的所有 TABLE 实例关掉，是为了满足 InnoDB 的断言。
3. ★ **最关键的差异在 else 分支**：引擎只要求 prepare 阶段 X 且不在 `LOCK TABLES` 时，虽然也升级到 X，但 `tdc_remove_table` 用的是 `TDC_RT_REMOVE_NOT_OWN_KEEP_SHARE`——**不把 `TABLE_SHARE` 标记为 old**。注释解释了原因：标记 old 会导致执行阶段其他线程无法打开这张表，那就不是 online 了。

> 这一"标不标记 old 的差别"，就是"INPLACE 但 online"与"INPLACE 但不 online"在 server 侧的分水岭。

接着是升级到 `MDL_SHARED_NO_WRITE`（禁写）的情形：引擎要求 `SHARED_LOCK`，或用户指定 `LOCK=SHARED`。

#### 阶段 ① prepare 在引擎侧做了什么：分配 row log + 取一致性视图

prepare 的核心动作之一是**给需要 online 的对象分配 row log**，两种形态走不同分支（`prepare_inplace_alter_table_dict`）：

```cpp
// 形态一：ADD INDEX —— 给每个新建索引分配（table 参数传 nullptr）
rw_lock_x_lock(&ctx->add_index[a]->lock, UT_LOCATION_HERE);
bool ok = row_log_allocate(ctx->add_index[a], nullptr, true, nullptr,
                           nullptr, path);
rw_lock_x_unlock(&ctx->add_index[a]->lock);

// 形态二：rebuild —— 给旧表聚簇索引分配（table 参数传新表，并带 col_map）
if (new_clustered) {
  dict_index_t *clust_index = user_table->first_index();
  dict_index_t *new_clust_index = ctx->new_table->first_index();
  // 新旧主键顺序一致时可省掉一次排序（skip_pk_sort）
  ctx->skip_pk_sort =
      innobase_pk_order_preserved(ctx->col_map, clust_index, new_clust_index);

  if (ctx->online) {
    /* Allocate a log for online table rebuild. */
    rw_lock_x_lock(&clust_index->lock, UT_LOCATION_HERE);
    bool ok = row_log_allocate(
        clust_index, ctx->new_table,
        !(ha_alter_info->handler_flags & Alter_inplace_info::ADD_PK_INDEX),
        ctx->add_cols, ctx->col_map, path);
    rw_lock_x_unlock(&clust_index->lock);
  }
}
```

★ 两种形态在源码里靠 `row_log_allocate` 的 **`table` 参数**区分——函数开头就有断言 `ut_ad(index->is_clustered() == !!table)`：**传 nullptr 的是二级索引形态（非聚簇），传新表的是重建表形态（挂在聚簇索引上，且必须传 `col_map` 做新旧列映射）**。这与前面"两种形态"表格里的"挂在哪"完全对应。

接着为 online 建索引的扫描**分配一致性读视图**——保证扫描看到的是一个一致快照：

```cpp
if (ctx->online) {
  /* Assign a consistent read view for the index build scan. */
  trx_assign_read_view(ctx->prebuilt->trx);
}
```

> 注意外层那个 `if (ctx->online)`：**非 online 的 DDL 既不分配 row log 也不取读视图**——它全程锁表，没有并发 DML 需要兜住。这再次说明 row log 唯一服务于 online。

##### 在分配 row log 之前：prepare 的前置校验链

`prepare_inplace_alter_table_impl` 在真正动手前有一长串"闸门"，其中几个决定了 DDL 能否继续：

```cpp
  MONITOR_ATOMIC_INC(MONITOR_PENDING_ALTER_TABLE);        // ① 计数器（可观测性）

  if (indexed_table->is_corrupted()) {                    // ② 聚簇索引损坏 → 拒绝
    my_error(ER_CHECK_NO_SUCH_TABLE, MYF(0));
    return true;
  }

  if (dict_table_is_discarded(indexed_table)) {           // ③ DISCARD 过的表
    if (innobase_need_rebuild(ha_alter_info) || type == INSTANT_VIRTUAL_ONLY ||
        type == INSTANT_ADD_DROP_COLUMN)
      my_error(ER_TABLESPACE_DISCARDED, MYF(0), ...); return true;
  }

  if (!(ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE)) {
    /* Nothing to do. Since there is no MDL protected, don't
    try to drop aborted indexes here. */                  // ★ ④ 无事可做时连僵尸索引都不清
    return false;
  }

  if (is_instant(ha_alter_info)) {                        // ⑤ INSTANT
    if (type == Instant_Type::INSTANT_ADD_DROP_COLUMN)
      ut_a(is_valid_row_version(indexed_table->current_row_version + 1));  // ★ 版本号上限
    return false;
  }
```

★ 三处值得单独记：

- **④ 的注释**点出一个容易忽略的约束：无事可做时**不去清理 aborted 索引**，因为此时没有 MDL 保护（`dict_operation_lock_mode == 0`）。⇒ "僵尸索引"的清理只在有 MDL 的路径上做（见「索引在线状态机与僵尸索引」节）。
- **⑤ 里的 `ut_a(is_valid_row_version(current_row_version + 1))`** —— 与 INSTANT 篇讲的 **`MAX_ROW_VERSION = 64`** 是同一条约束的**引擎侧断言点**：instant add/drop column 在 prepare 阶段就要保证版本号 +1 后仍然合法。
- **计数器 `MONITOR_PENDING_ALTER_TABLE`** 是 `ddl_pending_alter` 的埋点（见可观测性节），一进 prepare 就 +1。

**随后复用 CREATE TABLE 的参数解析器**——这印证了"建表是被复用的底座"（见 [`create_table.md`](create_table.md)）：

```cpp
  create_table_info_t info(m_user_thd, altered_table, ha_alter_info->create_info,
                           nullptr, nullptr, tablespace, is_file_per_table,
                           false, 0, 0, false);
  info.set_tablespace_type(is_file_per_table);

  if (ha_alter_info->handler_flags & Alter_inplace_info::CHANGE_CREATE_OPTION) {
    const char *invalid_opt = info.create_options_are_invalid();      // 复用 CREATE 的选项校验
    if (invalid_opt != nullptr) {
      my_error(ER_ILLEGAL_HA_CREATE_OPTION, MYF(0), table_type(), invalid_opt);
      goto err_exit_no_heap;
    }
  }
```

**错误出口也有讲究**——`err_exit_no_heap` 不是单纯返回：

```cpp
err_exit_no_heap:
  assert(m_prebuilt->trx->dict_operation_lock_mode == 0);
  if (ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) {
    online_retry_drop_dict_indexes(m_prebuilt->table, false);   // ★ 失败也要清僵尸索引
  }
  return true;
```

⇒ 与上面 ④ 形成对照：**有实际动作时失败 → 必须清理 aborted 索引**；完全没动作 → 不清理（无 MDL）。这就是为什么 `online_retry_drop_dict_indexes` 会出现在错误路径上。

**再往后是长度与命名校验**，其中一条很隐蔽：

```cpp
  max_col_len = DICT_MAX_FIELD_LEN_BY_FORMAT_FLAG(info.flags());   // 按行格式定上限

  for (新增索引...) {
    if (key->flags & HA_FULLTEXT) { assert(!(key->flags & HA_NOSAME)); continue; }  // FULLTEXT 不支持 UNIQUE
    if (innobase_check_column_length(max_col_len, key)) my_error(ER_INDEX_COLUMN_TOO_LONG, ...);
  }

  /* Check existing index definitions for too-long column
  prefixes as well, in case max_col_len shrunk. */
  for (已有索引...) { if (field->prefix_len > max_col_len) my_error(...); }   // ★
```

★ **已有索引也要重新检查一遍**：注释说明原因 *"in case max_col_len shrunk"*——改行格式（如 COMPACT → DYNAMIC）会让 `max_col_len` 变小，原本合法的旧索引前缀长度可能因此超标。所以这不是冗余检查，而是行格式变更的连带校验。

另外 prepare 里有一大段注释讲清了**表空间位置规则**，值得摘录结论：

- ALTER **不会**把表从 file-per-table 隐式移到 system tablespace；
- 但 `innodb_file_per_table=ON` 时会把表从 system **隐式移到** file-per-table；
- general tablespace 里的表**不会动**，除非显式写 `TABLESPACE=name`；
- 一旦显式移到 `innodb_system`，表会被打上 `SHARED_SPACE` 标记，**此后不能再隐式移回 file-per-table**。

（源码用 `ut_ad(1 == in_system_space + is_file_per_table + in_general_space)` 断言三者互斥且必居其一。）

**`ctx->new_table` 的一个反直觉事实**：它的**初值就是旧表指针**，不是新建的表——

```cpp
  indexed_table = m_prebuilt->table;      // ctx 构造时把旧表当 new_table 传进去
```

所以 `need_rebuild()` 的实现就是一句**指针比较**：

```cpp
  bool need_rebuild() const { return old_table != new_table; }
```

只有判定为 `new_clustered` 时，`prepare_inplace_alter_table_dict`（真正干活的那个子函数，位于 impl 之前约 1000 行）才会真正造一张新表：

```cpp
  const char *new_table_name =
      dict_mem_create_temporary_tablename(ctx->heap, ctx->new_table->name.m_name,
                                          ctx->new_table->id);   // ★ 临时表名
  ...
  ctx->new_table = dict_mem_table_create(new_table_name, space_id, n_cols + n_v_cols,
                                         n_v_cols, n_m_v_cols, flags, flags2);
  ...
  error = row_create_table_for_mysql(ctx->new_table, compression, ...);   // ★ 真的建表
```

★ 这推翻了"prepare 只做计划"的直觉：**prepare 阶段就已经在磁盘上建出一张真实的临时表**（临时名 + `row_create_table_for_mysql`），失败路径要 `dict_table_close_and_drop` 掉它。列定义来自遍历 `altered_table->field[]`，索引定义由 `innobase_create_key_defs` 生成后交给 `ddl::create_index`。

几个 ctx 字段的生效条件也值得记：

| 字段 | 何时被设置 |
|---|---|
| `skip_pk_sort` | 仅 rebuild，且 `innobase_pk_order_preserved(col_map, 旧聚簇, 新聚簇)` 成立时 |
| `add_cols` / `col_map` | 仅 rebuild（`col_map` 由 `innobase_build_col_map` 生成：旧列号 → 新列号） |
| `add_autoinc` | 扫 `create_list` 找**新增**且带 `AUTO_INCREMENT_FLAG` 的列；`autoinc_col_max_value` 取自 `field->get_max_int_value()` |
| `sequence` | 构造时总是创建，但 `m_max_value > 0` 只在 `add_autoinc != ULINT_UNDEFINED` 时成立（自增分配器，分区表多 ctx 共享同一份） |

**INSTANT 路径在 prepare 就返回了**——`is_instant(ha_alter_info)` 成立时直接 `return false`，**不建 ctx、不建 new_table、不建 add_index**，真正的改动推迟到执行阶段的 `Instant_ddl_impl`（见 [`instant_ddl.md`](instant_ddl.md)）。

★ **row log 的挂靠方向，两种形态正好相反**（都在 `prepare_inplace_alter_table_dict` 里）：

- **ADD INDEX**（非 rebuild + online + 非 FTS）→ `row_log_allocate` 给**新索引**分配；
- **rebuild** → `row_log_allocate(clust_index, ctx->new_table, ...)` 挂在**旧表的聚簇索引**上（`clust_index = user_table->first_index()`）。

⇒ 前者日志跟着"要建的对象"走，后者跟着"正在被读的源"走——因为 rebuild 记录的是"原表上发生了什么 DML"，而加索引记录的是"新索引上缺什么"。

**待删索引的标记**：`to_be_dropped = 1` 只在遍历 `index_drop_buffer` 时置位（在 dict mutex 下），失败回退时清 0；**主键不入 `drop_index` 数组**（单独走 FK 检查）；**外键只按名字收集指针，不置 `to_be_dropped`**——FK 的实际删除发生在 commit 阶段。

#### 阶段 ② 执行：引擎侧的执行内容

`ha_innobase::inplace_alter_table_impl` 开头有几个"提前返回"，它们本身信息量就很大：

```cpp
// ① 不涉及数据变更，或是 INSTANT → 什么都不做，直接返回
if (!(ha_alter_info->handler_flags & INNOBASE_ALTER_DATA) ||
    is_instant(ha_alter_info)) {
  return all_ok();
}

// ② 只改 CREATE OPTION 且不需要重建 → 也直接返回
if (((ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) ==
         Alter_inplace_info::CHANGE_CREATE_OPTION &&
     !innobase_need_rebuild(ha_alter_info))) {
  return all_ok();
}
```

★ ① 是**"INSTANT 为什么秒级"的引擎侧证据**：执行阶段直接返回，一个页都不构建。② 说明只改表选项（如 `STATS_PERSISTENT`、`ROW_FORMAT` 元数据部分）同样完全不碰数据。

真正需要构建时：

```cpp
// PFS 进度上报对象（以聚簇索引为依据）
ctx->m_stage = ut::new_withkey<Alter_stage>(UT_NEW_THIS_FILE_PSI_KEY, pk);

if (m_prebuilt->table->ibd_file_missing ||
    dict_table_is_discarded(m_prebuilt->table)) {
  return success();                       // 文件缺失 / 已 discard → 不构建
}

// 虚拟列模板：必要时重建，否则虚拟列求值会用到旧定义
rebuild_templ = ctx->need_rebuild() || ((ha_alter_info->handler_flags & ...));
if ((ctx->new_table->n_v_cols > 0) && rebuild_templ) {
  s_templ = ut::new_withkey<dict_vcol_templ_t>(...);
  innobase_build_v_templ(altered_table, ctx->new_table, s_templ, nullptr, false, nullptr);
  ctx->new_table->vc_templ = s_templ;
}
```

要点：

- **构建本身**（扫描—排序—建树三期、并行调度、临时文件）见 [`ddl_engine.md`](ddl_engine.md)，本篇不重复。
- **虚拟列模板**是个容易被忽略的坑：加虚拟列时必须重建 `vc_templ`，否则求值时用的还是旧表定义。
- 构建完成后**立刻做第一次 apply**（条件与动机见「回放时机」节）。

**其余两处是这一阶段真正的"机制含量"所在。**

##### ★ 隔离级别被强制提到 REPEATABLE READ

调用 `ddl.build()` 之前有一段容易被跳过的代码：

```cpp
  if (ctx->online &&
      trx->isolation_level != trx_t::isolation_level_t::REPEATABLE_READ) {
    /* We must scan the index at an isolation level >= READ COMMITTED, because
    a dirty read will see half written blob references.

    ** Perform a REPEATABLE READ.
    When rebuilding the table online, row_log_table_apply() must not see
    a newer state of the table when applying the log. This is mainly to
    prevent false duplicate key errors, because the log will identify records
    by the PRIMARY KEY, and also to prevent unsafe BLOB access.

    When creating a secondary index online, this table scan must not see
    records that have only been inserted to the clustered index, but have
    not been written to the online_log of index[]. If we performed
    READ UNCOMMITTED, it could happen that the ADD INDEX reaches
    ONLINE_INDEX_COMPLETE state between the time the DML thread has updated
    the clustered index but has not yet accessed secondary index. */

    trx->isolation_level = trx_t::isolation_level_t::REPEATABLE_READ;
  }
```

★ 注释给了**三条理由，分别对应三种故障**：

1. **脏读会看到"写了一半的 BLOB 引用"**——扫描时若隔离级别低于 RC，可能读到尚未完成的外存 BLOB 指针。
2. ★ **防止误报重复键**：`row_log_table_apply()` 是**按 PRIMARY KEY 标识记录**来回放的，如果它能看到比日志更新的表状态，同一条记录可能被应用两次 → `DB_DUPLICATE_KEY` 假错。这就是 RR 在这里的**正确性**作用，而不只是"一致性偏好"。
3. **建二级索引时**：如果扫描看到了"已写入聚簇索引、但还没写进 `online_log` 的"记录，而此刻 ADD INDEX 刚好进入 `ONLINE_INDEX_COMPLETE`，这批记录就漏了。

⇒ 所以 online DDL 的执行阶段**无论用户设了什么隔离级别都按 RR 扫描**，退出前再恢复（`trx->isolation_level = old_isolation_level`）。

##### ★ `clean_up` lambda：第一次 apply + 错误分派

构建的全部收尾都在这个 lambda 里（`err` 即 `ddl.build()` 的返回值），**错误码到 MySQL 报错的映射也在这里**：

```cpp
  auto clean_up = [&](dberr_t err) -> bool {
    if (err == DB_SUCCESS && ctx->online && ctx->need_rebuild()) {
      // 第一次 apply：把执行期间攒下的 row log 大头消化掉
      err = row_log_table_apply(ctx->thr, m_prebuilt->table, altered_table,
                                ctx->m_stage);
    }

    if (s_templ) {                       // 虚拟列模板用完即释放并还原
      dict_free_vc_templ(s_templ);
      ut::delete_(s_templ);
      ctx->new_table->vc_templ = old_templ;
    }

    switch (err) {
      case DB_SUCCESS:          return success();
      case DB_DUPLICATE_KEY:    ... print_keydup_error(...); break;
      case DB_ONLINE_LOG_TOO_BIG:
        assert(ctx->online);
        my_error(ER_INNODB_ONLINE_LOG_TOO_BIG, MYF(0), ...); break;
      case DB_INDEX_CORRUPT:    my_error(ER_INDEX_CORRUPT, MYF(0), ...); break;
      default:                  my_error_innodb(err, ...);
    }
    return true;
  };

  ddl::Context ddl(trx, m_prebuilt->table, ctx->new_table, ctx->online,
                   ctx->add_index, ctx->add_key_numbers, ctx->num_to_add_index,
                   altered_table, ctx->add_cols, ctx->col_map, ctx->add_autoinc,
                   ctx->sequence, ctx->skip_pk_sort, ctx->m_stage, add_v,
                   eval_table, thd_ddl_buffer_size(...), thd_ddl_threads(...));
  const auto err = clean_up(ddl.build());
```

三个值得记的点：

1. ★ **`DB_ONLINE_LOG_TOO_BIG` → `ER_INNODB_ONLINE_LOG_TOO_BIG`**：这就是 `innodb_online_alter_log_max_size` 超限的报错出处，且 `assert(ctx->online)` 说明**只有 online 路径才可能有这个错**（非 online 时 DML 不并发，不会有 row log 积压）。
2. ★ **`DB_DUPLICATE_KEY` 定位出错索引时有一步"减一"修正**：`error_key_num` 是 InnoDB 内部索引序号，而 `key_info_buffer` 是 server 层数组；**表没有显式 PRIMARY KEY 时，InnoDB 自动建的 `GEN_CLUST_INDEX` 会在 InnoDB 索引列表里占位置 0，但 server 层不知道它**——所以此时 `num_to_add_index > key_count`，要用 `error_key_num - 1` 才能对上（源码注释完整解释了这一点）。`error_key_num == ULINT_UNDEFINED` 或 `== 0` 则是 FTS_DOC_ID 隐藏索引的情况，报的是"假重复键"。
3. **`ddl::Context` 的构造参数就是这一阶段的全部输入**：`add_index` / `add_key_numbers` / `num_to_add_index`（要建哪些索引）、`add_cols` + `col_map`（加的列与新旧列映射）、`add_autoinc`、`sequence`、`skip_pk_sort`（有主键时跳过排序的优化）、`add_v`（虚拟列）、`eval_table`（用哪张表求值虚拟列）、以及 buffer_size / threads 两个并行度参数——它们的含义详见 [`ddl_engine.md`](ddl_engine.md)。

#### 阶段 ③ commit 在引擎侧做了什么：X 锁 → 第二次 apply → 切换 → 统计

`commit_inplace_alter_table_impl`（storage/innobase/handler/handler0alter.cc）是四件套里最长的一段，按流程分六块：

```text
① 入参分流：commit=false → 转 rollback_inplace_alter_table（commit 阶段回滚＝撤销）；
   instant 或无实际操作 → 直接返回
② 对每个分区 ddl::lock_table(LOCK_X)          ★ MDL X 之外还要 InnoDB 内部锁
③ rebuild 场景的准备：FTS 摘除 + 生成临时表名 #sql-ib<id> 并拿 MDL
   + row_mysql_lock_data_dictionary（独占 DD）+ 停后台统计
④ 进入 DD 事务并分岔：rebuild → commit_try_rebuild / no-rebuild → commit_try_norebuild
⑤ 内存与文件的切换：rename、FK cache、autoinc、FTS 发布、drop 旧表
⑥ 统计重算
```

（文件名与函数入口：`storage/innobase/handler/handler0alter.cc` 的 `commit_inplace_alter_table_impl`。下面按这六块逐段看源码，**不依赖行号**——行号会随版本漂移，机制不会。）

下面按实际源码逐段看。注意一个前提：**这一切都在分区级 ctx 数组上循环**（`ctx_array`，单表时是 `ctx_single`）——分区表的每个分区各有一份上下文，所以下面每个动作都是 for 循环包着的。

##### ① 起事务 + 对每个分区拿 InnoDB 表级 X 锁

```cpp
  trx_start_if_not_started_xa(m_prebuilt->trx, true, UT_LOCATION_HERE);

  for (inplace_alter_handler_ctx **pctx = ctx_array; *pctx; pctx++) {
    ha_innobase_inplace_ctx *ctx = static_cast<ha_innobase_inplace_ctx *>(*pctx);
    assert(ctx->prebuilt->trx == m_prebuilt->trx);

    /* Exclusively lock the table, to ensure that no other
    transaction is holding locks on the table while we
    change the table definition. The MySQL meta-data lock
    should normally guarantee that no conflicting locks
    exist. However, FOREIGN KEY constraints checks and any
    transactions collected during crash recovery could be
    holding InnoDB locks only, not MySQL locks. */

    error = ddl::lock_table(m_prebuilt->trx, ctx->old_table, LOCK_X);
    if (error != DB_SUCCESS) { my_error_innodb(error, ...); return true; }
  }

  DEBUG_SYNC(m_user_thd, "innodb_alter_commit_after_lock_table");

  const bool new_clustered = ctx0->need_rebuild();                        // ★分岔点
```

★ MDL 只能约束走 MySQL 层的加锁者——**外键检查、崩溃恢复收集的事务只持 InnoDB 锁**，MDL 管不到，所以引擎层必须再拿一次；且是**逐分区**拿。`` 的 `new_clustered` 就是后面所有分支的总开关（rebuild vs no-rebuild）。

##### ② rebuild 场景：FTS 摘除 + 生成临时名 + 锁 DD + 停后台统计

```cpp
  if (new_clustered) {
    for (每个 ctx) {
      if (ctx->old_table->fts) {
        ut_ad(!ctx->old_table->fts->add_wq);            // 断言：无待处理 FTS 工作队列
        fts_optimize_remove_table(ctx->old_table);      // 旧表摘出 FTS 优化器
      }
      if (ctx->new_table->fts) {
        ut_ad(!ctx->new_table->fts->add_wq);
        fts_optimize_remove_table(ctx->new_table);      // 新表同样
      }
    }
  }

  /* Generate the temporary name for old table, and acquire mdl lock on it. */
  for (每个 ctx) {
    if (ctx->need_rebuild()) {
      ctx->tmp_name = dict_mem_create_temporary_tablename( "#sql-ib<id>"
          ctx->heap, ctx->new_table->name.m_name, ctx->new_table->id);
      dict_name::get_table(ctx->tmp_name, db_str, tbl_str);
      MDL_ticket *mdl_ticket = nullptr;
      if (dd::acquire_exclusive_table_mdl(thd, db_str.c_str(), tbl_str.c_str(),
                                          false, &mdl_ticket))          // ★ 为临时名也拿 MDL X
        return true;
    }
  }

  /* Latch the InnoDB data dictionary exclusively so that no deadlocks
  or lock waits can happen in it during the data dictionary operation. */
  row_mysql_lock_data_dictionary(trx, UT_LOCATION_HERE);

  /* Prevent the background statistics collection from accessing the tables. */
  for (;;) {
    bool retry = false;
    for (每个 ctx) {
      if (new_clustered && !dict_stats_stop_bg(ctx->old_table))          // ★ 停后台统计
        retry = true;
      ...
```

三个细节：

1. ★ **为临时名 `#sql-ib<id>` 也要拿 MDL X**——rename 过程中旧表要顶这个名字，如果不先占住，别的会话可能正好建/开同名表，rename 就会撞车或误伤。
2. ★ **`row_mysql_lock_data_dictionary` 的注释说明了目的**：*"so that no deadlocks or lock waits can happen in it"*——不是单纯"互斥"，而是**独占 DD 以避免后续 DD 操作之间发生死锁/锁等待**。这是一次性把整段 DD 操作变成"串行临界区"的做法。
3. ★ **停后台统计带 retry 循环**（`for(;;)` + `retry`）：`dict_stats_stop_bg` 可能失败（正有统计在跑），失败就整轮重来——因为后面要动表定义，后台统计线程不能再来读。

★ 最容易看漏的一点：**第二次 apply 不在本函数里**，而在 `commit_try_rebuild` 内部 `row_log_table_apply(...)`，仅 `ctx->online`）；apply 前会重建虚拟列模板。失败映射与阶段 ② 相同（`DB_DUPLICATE_KEY` → `print_keydup_error`，`DB_ONLINE_LOG_TOO_BIG` → `ER_INNODB_ONLINE_LOG_TOO_BIG`）。而 **no-rebuild 的加索引在此阶段不再 apply**——它的 apply 只有**一次**，且发生在**执行阶段的 Builder 内部**（`ddl/ddl0builder.cc` 的 `row_log_apply(...)`，紧跟在 `write_redo(m_index)` 之后，两侧有 `row_log_apply_before/after` 两个 DEBUG_SYNC 点），此后不再有第二次。

⇒ 所以"apply 两次"**只适用于 rebuild 形态**，这是两种形态在回放上的根本差异，根源在于：rebuild 期间新旧表并存、commit 阶段还要再切换一次，切换前后各有增量；而加索引只往现有表上挂一棵新索引，构建完（执行阶段）把日志追平即可，commit 阶段只是把它转正。

##### LOCK_X 的注释与临时表名

```cpp
/* Exclusively lock the table, to ensure that no other
transaction is holding locks on the table while we
change the table definition. The MySQL meta-data lock
should normally guarantee that no conflicting locks
exist. However, FOREIGN KEY constraints checks and any
transactions collected during crash recovery could be
holding InnoDB locks only, not MySQL locks. */
error = ddl::lock_table(m_prebuilt->trx, ctx->old_table, LOCK_X);
```

★ MDL 只能约束走 MySQL 层的加锁者——外键检查、崩溃恢复收集的事务**只持 InnoDB 锁**，MDL 管不到，所以引擎层必须再拿一次。且是对**每个分区**逐一加锁。

rebuild 场景会先给旧表生成临时表名 `#sql-ib<id>` 并**为这个名字也拿 MDL**附近）——因为 rename 过程中旧表要顶这个名字，别的会话不能拿它。

**进入 DD 事务后 rebuild 走 `commit_try_rebuild`**，它是"第二次 apply + 前置校验 + 让位"的实际执行体：

```cpp
[[nodiscard]] inline bool commit_try_rebuild(Alter_inplace_info *ha_alter_info,
                                             ha_innobase_inplace_ctx *ctx,
                                             TABLE *altered_table,
                                             const TABLE *old_table, trx_t *trx,
                                             const char *table_name) {
  dict_table_t *rebuilt_table = ctx->new_table;
  dict_table_t *user_table = ctx->old_table;

  assert(ctx->need_rebuild());
  assert(trx->dict_operation_lock_mode == RW_X_LATCH);   // ★ 进入时必须已持 DD X latch
  assert(!(ha_alter_info->handler_flags & Alter_inplace_info::DROP_FOREIGN_KEY) ||
         ctx->num_to_drop_fk > 0);

  for (dict_index_t *index = rebuilt_table->first_index(); index; index = index->next()) {
    assert(dict_index_get_online_status(index) == ONLINE_INDEX_COMPLETE);
    assert(index->is_committed());
    if (index->is_corrupted()) { my_error(ER_INDEX_CORRUPT, MYF(0), index->name()); return true; }
  }

  if (innobase_update_foreign_try(ctx, trx, table_name)) return true;    // 外键落定

  /* Clear the to_be_dropped flag in the data dictionary cache of user_table. */
  for (ulint i = 0; i < ctx->num_to_drop_index; i++) {
    dict_index_t *index = ctx->drop_index[i];
    assert(index->table == user_table);
    assert(index->is_committed());
    assert(index->to_be_dropped);
    index->to_be_dropped = 0;                            // ★ 清标记
  }

  /* We copied the table. Any indexes that were requested to be dropped
  were not created in the copy of the table. Apply any last bit of the
  rebuild log and then rename the tables. */

  if (ctx->online) {
    DEBUG_SYNC_C("row_log_table_apply2_before");         // ★ 复现"收尾阻塞"的标准注入点

    if (ctx->new_table->n_v_cols > 0) {                  // 虚拟列模板（本次带 true）
      s_templ = ut::new_withkey<dict_vcol_templ_t>(...);
      innobase_build_v_templ(altered_table, ctx->new_table, s_templ, nullptr, true, nullptr);
      ctx->new_table->vc_templ = s_templ;
    }

    error = row_log_table_apply(ctx->thr, user_table, altered_table,
                                ...->m_stage);           // ★ 第二次 apply

    if (s_templ) { dict_free_vc_templ(s_templ); ...; ctx->new_table->vc_templ = nullptr; }

    ulint err_key = thr_get_trx(ctx->thr)->error_key_num;
    switch (error) {
      case DB_DUPLICATE_KEY:
        if (err_key == ULINT_UNDEFINED) dup_key = nullptr;       // FTS_DOC_ID 隐藏索引
        else if (ctx->num_to_add_index > ha_alter_info->key_count)
          dup_key = &ha_alter_info->key_info_buffer[err_key - 1]; // ★ GEN_CLUST_INDEX 占位
        else
          dup_key = &ha_alter_info->key_info_buffer[err_key];
        print_keydup_error(...); return true;
      case DB_ONLINE_LOG_TOO_BIG:
        my_error(ER_INNODB_ONLINE_LOG_TOO_BIG, ...); return true;   // row log 超限
      case DB_INDEX_CORRUPT: my_error(ER_INDEX_CORRUPT, ...); return true;
      default: my_error_innodb(...); return true;
    }
  }
```

★ 三段各有一个要点：

1. **入口断言 `dict_operation_lock_mode == RW_X_LATCH`**——本函数假定调用方已独占 DD（外层 `row_mysql_lock_data_dictionary` 保证）；同时断言"要删外键时 `num_to_drop_fk` 必须 > 0"，防止 flag 与上下文不一致。
2. ★ **为什么 rebuild 不需要逐索引删除**：注释原话 *"Any indexes that were requested to be dropped were not created in the copy of the table"*——要删的索引**压根没在新表里建**，所以只需把旧表缓存里的 `to_be_dropped` 标记清掉，随后整个旧表一起 drop 即可。这是 rebuild 与 no-rebuild 在"删索引"上的根本差异。
3. **第二次 apply 的错误分派里，`err_key` 定位有两处特判**——与执行阶段 `clean_up` 里那套完全同构：`ULINT_UNDEFINED` 是 FTS_DOC_ID 隐藏索引（报不出具体索引名）；无显式主键时 `GEN_CLUST_INDEX` 在 InnoDB 索引列表里占位置 0，所以 `num_to_add_index > key_count` 时要 **`err_key - 1`** 才能对上 server 层的 `key_info_buffer`。

函数末尾还有两段值得看：

```cpp
  /* The new table must inherit the flag from the "parent" table. */
  if (dict_table_is_discarded(user_table)) {
    rebuilt_table->ibd_file_missing = true;
    rebuilt_table->flags2 |= DICT_TF2_DISCARDED;        // ★ 继承 DISCARD 状态
  }

  if (user_table->get_ref_count() > 1) {
    /* This should only occur when an innodb_memcached
    connection with innodb_api_enable_mdl=off was started
    before commit_inplace_alter_table() locked the data
    dictionary. We must roll back the ALTER TABLE, because
    we cannot drop a table while it is being used.

    Normally, n_ref_count must be 1, because purge
    cannot be executing on this very table as we are
    holding MDL lock. */
    my_error(ER_TABLE_REFERENCED, MYF(0));
    return true;
  }
```

★ 两个"守卫"：

- **discard 状态必须继承**——旧表若是 `DISCARD TABLESPACE` 状态，新表要接过 `ibd_file_missing` 与 `DICT_TF2_DISCARDED`，否则切换后表的表空间状态会凭空"恢复"。
- ★ **`n_ref_count > 1` 时拒绝**：注释指出唯一已知场景是 **`innodb_memcached` 连接且 `innodb_api_enable_mdl=off`**（且在本函数锁 DD 之前就已启动）——它绕过了 MDL，所以**引擎层的引用计数是最后一道防线**；否则正常情况下引用计数必然是 1（持 MDL 时 purge 不会在本表上跑）。这也是"为什么 InnoDB 要自己维护 `n_ref_count` 而不能只信 MDL"的实证。

##### ★ rename 切换：先"彩排"再执行

切换分两步：**先给文件改名"彩排"**（`fil_rename_precheck`，检查目标文件名可用，避免改到一半失败留下两个半成品），**再执行字典缓存里的两次改名**：

```cpp
inline void commit_cache_rebuild(ha_innobase_inplace_ctx *ctx) {
  DBUG_TRACE;
  DEBUG_SYNC_C("commit_cache_rebuild");
  assert(ctx->need_rebuild());
  assert(dict_table_is_discarded(ctx->old_table) ==
         dict_table_is_discarded(ctx->new_table));      // ① 新旧表 discard 状态必须一致

  const char *old_name =
      mem_heap_strdup(ctx->heap, ctx->old_table->name.m_name);   // ★ ② 先存下旧名

  /* We already committed and redo logged the renames,
  so this must succeed. */
  error = dict_table_rename_in_cache(ctx->old_table, ctx->tmp_name, false);  // ③ 旧表让位
  ut_a(error == DB_SUCCESS);

  error = dict_table_rename_in_cache(ctx->new_table, old_name, false);       // ④ 新表顶替
  ut_a(error == DB_SUCCESS);
}
```

★ 四点：

1. **顺序是"先让位、后顶替"**——两张表不能同时叫原名，必须先把旧表改到临时名 `#sql-ib<id>`，新表才能顶上；反过来会撞名。
2. ★ **`old_name` 必须先 `mem_heap_strdup` 存一份**：紧接着 ③ 就要改 `old_table` 的名字，若直接用它的 `name.m_name` 指针，到 ④ 时指针已指向临时名——新表会被改成临时名，整张表就"消失"了。
3. ★ **两次都 `ut_a`**——注释给出理由：*"We already committed and redo logged the renames, so this must succeed"*。到此 DD 事务与 redo 都已提交，内存改名**不允许失败**，失败只能是 bug，直接崩。
4. **断言 discard 状态一致**（①）：新旧表的 `DISCARD TABLESPACE` 标记必须相同，否则说明状态在 rebuild 中变了。

旧表本体随后由 `ddl::drop_table` 删除，而 `write_drop_log` 已提前写入 DDL log——**崩溃后由 DDL log 兜底清理**（见 [`atomic_ddl.md`](atomic_ddl.md)）。

**no-rebuild 分支走 `commit_try_norebuild`**，它比 rebuild 版短得多，因为"转正"的只是索引：

```cpp
[[nodiscard]] inline bool commit_try_norebuild(Alter_inplace_info *ha_alter_info,
                                               ha_innobase_inplace_ctx *ctx, trx_t *trx,
                                               const char *table_name) {
  assert(!ctx->need_rebuild());
  assert(trx->dict_operation_lock_mode == RW_X_LATCH);
  assert(!(ha_alter_info->handler_flags & Alter_inplace_info::DROP_FOREIGN_KEY) ||
         ctx->num_to_drop_fk > 0);

  for (ulint i = 0; i < ctx->num_to_add_index; i++) {
    dict_index_t *index = ctx->add_index[i];
    assert(dict_index_get_online_status(index) == ONLINE_INDEX_COMPLETE);
    assert(!index->is_committed());          // ★ 与 rebuild 版相反
    if (index->is_corrupted()) {
      /* Report a duplicate key error for the index that was
      flagged corrupted, most likely because a duplicate value was
      inserted (directly or by rollback) after
      ha_innobase::inplace_alter_table() completed.
      TODO: report this as a corruption with a detailed reason once
      WL#6379 has been implemented. */
      my_error(ER_DUP_UNKNOWN_IN_INDEX, MYF(0), index->name());
      return true;
    }
  }

  if (innobase_update_foreign_try(ctx, trx, table_name)) return true;
  return false;
}
```

★ 三个与 rebuild 版的关键差异：

1. **断言相反**：rebuild 版断言新表索引 `is_committed()`（构建时已提交），这里断言 **`!index->is_committed()`**——因为 no-rebuild 的新索引在此刻**还没转正**。
2. **没有第二次 row log apply** —— 加索引的两次 apply 都在前两个阶段完成（执行阶段 `clean_up` 里那次是第二次），所以这个函数里完全没有 `row_log_*` 调用。
3. **索引损坏的报错是 `ER_DUP_UNKNOWN_IN_INDEX`** —— 注释解释成因：*most likely because a duplicate value was inserted (directly or by rollback) after `ha_innobase::inplace_alter_table()` completed*。即在构建完成后、转正之前，又有重复值被写进来（并发 DML 或回滚），索引被标 corrupted，但此时已无法定位具体是哪条记录 ⇒ 只能报"索引中有未知重复键"，并留下 TODO（WL#6379 后改成带原因的 corruption 报错）。

**真正的"转正"发生在随后的 `commit_cache_norebuild`**：

```cpp
  /* Check if the column, part of an index to be dropped is part of any
  other index which is not being dropped. If not, then set the ord_part
  of the column to 0. Here the columns are collected first. */
  get_col_list_to_be_dropped(ctx, drop_list, v_drop_list);

  for (ulint i = 0; i < ctx->num_to_add_index; i++) {
    dict_index_t *index = ctx->add_index[i];
    assert(dict_index_get_online_status(index) == ONLINE_INDEX_COMPLETE);
    assert(!index->is_committed());
    index->set_committed(true);                     // ★ 转正：新索引进入字典
  }

  if (ctx->num_to_drop_index) {
    for (每个待删索引) {
      /* Replace the indexes in foreign key constraints if needed. */
      if (!dict_foreign_replace_index(index->table, ctx->col_names, index))
        found = false;
    }
    for (每个待删索引) {
      if (index->type & DICT_FTS) fts_drop_index(index->table, index, trx, ...);

      /* It is a single table tablespace and the .ibd file is
      missing if root is FIL_NULL, do nothing. */
      if (index->page != FIL_NULL) {
        dict_sys_mutex_exit();                      // ★ 写 DDL log 前放全局锁
        ut_d(dberr_t err =) log_ddl->write_free_tree_log(trx, index, true);
        ut_ad(err == DB_SUCCESS);
        dict_sys_mutex_enter();
      }
      btr_drop_ahi_for_index(index);
      ...
```

★ 三点：

1. **转正就是一行 `set_committed(true)`** —— 索引此前一直是"在线构建中"的状态（`ONLINE_INDEX_COMPLETE` 但 `!is_committed()`），这一步才让它正式进入字典、对优化器可见。这正是 WL#5526 第七步"临时索引转正"在 8.0 的落地形态。
2. **`get_col_list_to_be_dropped` 有个连带动作**：收集"要删索引涉及的列"，若这些列**不在任何保留下来的索引里**，就把列的 `ord_part` 清零——即列不再参与任何索引排序。
3. ★ **删索引时同时做两件事**：先用 `dict_foreign_replace_index` 把外键约束里对该索引的引用换掉（否则外键会指向已删索引），再写 `write_free_tree_log` 让崩溃后能释放索引树——同样遵守"写 DDL log 前放 `dict_sys` mutex"的规矩，而 `index->page != FIL_NULL` 的判断则排除了"单表表空间但 .ibd 缺失"的情况。

**统计重算与 FTS 发布**紧随切换之后：rebuild 先 `dict_stats_drop_table` 删掉旧统计，再 `alter_stats_rebuild` 重算；no-rebuild 走 `alter_stats_norebuild`。★ 源码里有一条 TODO 注释承认这段**本可以在 MDL 降级之后再执行**——也就是说统计重算目前仍在 X 锁窗口内，它是"DDL 最后卡一下"的组成部分之一。FTS 则分两头：rebuild 前先 `fts_optimize_remove_table` 把新旧表都从全文优化器摘掉，切换成功后 `fts_add_index` 发布新全文索引并 `fts_optimize_add_table`（注释说明：**只有不存在合适 DOC_ID 列时才会触发 rebuild**），最后 drop FTS 辅助表。

**错误处理没有集中的 fail 标签**，靠一个 `fail` 布尔变量加一处 `goto rollback_trx`（仅 autoinc 失败走）。清理动作因场景而异：rebuild **关闭并丢弃 `new_table`**；no-rebuild 则用 `innobase_rollback_sec_index` **删除已经建好的垃圾二级索引**；随后 `row_mysql_unlock_data_dictionary` 并返回 true，交给上层回滚语句、由 DD 事务回滚恢复旧表定义。★ 其中最惊人的一条注释出现在切换失败分支附近：此时 `*.frm` 尚未替换、dict cache 已处于不一致状态，**源码选择打 warning 后继续，并建议"杀掉服务器重启"**——这说明 rename 后半段的失败在进程内是无解的，重启（走崩溃恢复）才是恢复手段。

**两处收尾细节**：一是 `to_be_dropped` 在 rename 之前就提前置位，用于防止并发 DML 继续引用**陈旧的 FK 元数据**，注释注明这是 WL#6049 未实现部分留下的补丁；二是本函数内崩溃注入点非常密集（`crash_before_commit` / `after_commit` / `before_update_stats` 等），每一步之间都是刻意留出的断点，用来验证恢复路径。结尾断言 **online log 已清空且索引状态为 `ONLINE_INDEX_COMPLETE`**，`MONITOR_ATOMIC_DEC(PENDING_ALTER_TABLE)` 之后返回 false；调用方 `ha_innobase::commit_inplace_alter_table` 在成功且非 instant 时再调 `dd_commit_inplace_alter_table` 落 DD 元数据。

WL#5526 设计时的七步原型（5.6，帮助理解协议来历）：`row_log_allocate`（分配 online log）→ 扫描聚簇索引写 merge 文件 → 归并排序 → 插入新 B+树 → `row_log_apply`（索引"发布"）→ **MySQL 升级 MDL** → `commit_inplace_alter_table`（临时索引转正）。8.0 把中间三步换成了 `ddl::Builder`（[`ddl_engine.md`](ddl_engine.md)），首尾协议不变。

### 整体架构：`Alter_inplace_info`、ctx 生命周期与回滚契约

主链路之上还有三条贯穿全局的约定，它们决定了四件套为什么长成这样。

#### `Alter_inplace_info`：add 用下标、drop 用指针

server 与引擎之间传递"要改哪些索引"靠三个数组，但**它们的元素类型并不一致**，这是最容易读错的一处（`sql/handler.h`）：

```cpp
  /** Size of key_info_buffer array. */
  uint key_count;
  KEY *key_info_buffer;                  // 新表定义的全部 KEY

  /** Size of index_drop_buffer array. */
  uint index_drop_count;
  /**
     Array of pointers to KEYs to be dropped belonging to the TABLE instance
     for the old version of the table.
  */
  KEY **index_drop_buffer;               // ① ★ 指针数组

  /** Size of index_add_buffer array. */
  uint index_add_count;
  /**
     Array of indexes into key_info_buffer for KEYs to be added,
     sorted in increasing order.
  */
  uint *index_add_buffer;                // ② ★ 下标数组，且递增有序

  KEY_PAIR *index_rename_buffer;         // ③ 改名：一对指针（旧 KEY + 新 KEY）
```

★ 为什么一个是指针、一个是下标？因为**两者的数据来源不同**：

- **要删的索引属于"旧表"**——旧 `TABLE` 实例有自己的 `KEY` 数组，那些对象**不在 `key_info_buffer` 里**，所以只能直接给指针（注释明说 *"belonging to the TABLE instance for the old version of the table"*）。
- **要加的索引属于"新表"**——它们就在 `key_info_buffer` 中，所以给**下标**即可，配合 `key_count` 定位。

⇒ 读引擎代码时看到 `index_add_buffer[i]` 要立刻意识到这是**下标**，得写成 `key_info_buffer[index_add_buffer[i]]` 才是真正的 KEY；而 `index_drop_buffer[i]` 直接就是 `KEY *`。混用会直接读坏内存。

另外那条 **"sorted in increasing order"** 不是随口一提：InnoDB 遍历新增索引时依赖这个顺序（按 `key_info_buffer` 的序号递增创建），保证多列索引与其前缀索引的创建次序正确。

#### ctx 生命周期：成功路径由 server 的析构器销毁

`handler_ctx` 是引擎的私有上下文（`ha_innobase_inplace_ctx`），它的所有权规则有点特别：

- **server 侧**在 `Alter_inplace_info` 析构时统一销毁——`~Alter_inplace_info() { destroy(handler_ctx); }`；
- **引擎侧成功路径不 `delete`**，而是把 `ha_alter_info->handler_ctx = nullptr`（如 `handler0alter.cc:10787`、`10811`）交还给 server，由上面的析构器收尾；
- 失败/回滚路径才由引擎自己清理。

⇒ 于是形成一条清晰的所有权链：**引擎分配 →（成功）置 nullptr 交还 → server 析构销毁**；引擎自己只在异常路径上删。这也解释了为什么引擎侧的成功分支里几乎看不到 `delete ctx`——不是泄漏，是移交了。

#### ★ 回滚契约：handler 层没有 rollback API

这是最容易被误解的一点：**`handler` 基类里没有 `rollback_inplace_alter_table()` 虚函数**。server 层要撤销 INPLACE，调的是：

```cpp
ha_alter_info->commit_inplace_alter_table(commit = false);
```

即**"提交接口的 `commit=false` 形态就是回滚"**。InnoDB 的对应实现（`handler0alter.cc:7442`）：

```cpp
  if (!commit) {
    /* A rollback is being requested. So far we may at
    most have created some indexes. If any indexes were to
    be dropped, they would actually be dropped in this
    method if commit=true. */
    const bool ret =
        rollback_inplace_alter_table(ha_alter_info, table, m_prebuilt);
    return ret;
  }
```

★ 这条注释是理解回滚语义的钥匙：走到这一步时，**最多只是"创建"了一些索引**——**要删除的索引此刻还没真删**，它们只在 `commit=true` 分支里才被真正 drop。所以回滚**不需要"把删掉的索引恢复回来"**（没删过），只需清理掉"已建但还没转正的新索引"。

（注意 `rollback_inplace_alter_table` 是 InnoDB **内部的 `[[nodiscard]] inline` 辅助函数**（`handler0alter.cc:6418`），不是 handler 接口——别因为同名就以为引擎层有独立的回滚 API。）

它的本体有两条值得记的顺序约束：

```cpp
[[nodiscard]] inline bool rollback_inplace_alter_table(
    const Alter_inplace_info *ha_alter_info, const TABLE *table,
    row_prebuilt_t *prebuilt) {
  ha_innobase_inplace_ctx *ctx =
      static_cast<ha_innobase_inplace_ctx *>(ha_alter_info->handler_ctx);

  if (!ctx || !ctx->trx) {
    /* If we have not started a transaction yet,
    (almost) nothing has been or needs to be done. */
    goto func_exit;                          // ① 事务没开始 ⇒ 基本无事可做
  }

  row_mysql_lock_data_dictionary(ctx->trx, UT_LOCATION_HERE);

  if (ctx->need_rebuild()) {
    /* The table could have been closed in commit phase */
    if (ctx->new_table != nullptr) {
      /* DML threads can access ctx->new_table via the
      online rebuild log. Free it first. */              // ② ★ 先后顺序
      innobase_online_rebuild_log_free(prebuilt->table);
      dict_table_close(ctx->new_table, true, false);
      ...
```

1. **`!ctx || !ctx->trx` 直接跳转**——事务都还没起，说明 prepare 阶段就没真正动手，几乎无需清理。
2. ★ **rebuild 形态下必须先 `innobase_online_rebuild_log_free()` 再 `dict_table_close()`**——注释点明原因：*DML 线程可能正通过 online rebuild log 访问 `ctx->new_table`*。若先关表，那些还在日志路径上的并发 DML 就会访问到已释放的对象。⇒ 这个顺序不是风格问题，是并发安全要求。

非 rebuild 分支的断言同样印证了"删除尚未发生"：`assert(!(handler_flags & ADD_PK_INDEX))`、`assert(ctx->new_table == prebuilt->table)`——非 rebuild 时新旧表本就是同一个对象。

### 引擎接口：handler / handlerton 如何承载 DDL

⚠️ **勘误**：
- **不存在 `ha_innobase::notify_table_changed()`**——InnoDB **未 override**（全仓 `storage/innobase/**` grep 结果 0），只有 ndb 实现（`ha_ndbcluster.cc:16613`）
- **`ha_rollback_inplace_alter_table()` 不存在**——回滚复用 `ha_commit_inplace_alter_table(..., commit=false, ...)`
- **`ha_inplace_alter_table()` 是 inline**（定义在 `handler.h:6191`，无 .cc 实现）
- **不存在 `Commit_credits` / `m_extra_status`**（全仓 0 命中）

**四个 `ha_*` 包装**（`sql/handler.cc:4925` + `sql/handler.h:6178`）：

```cpp
bool handler::ha_prepare_inplace_alter_table(TABLE *altered_table,
                                             Alter_inplace_info *ha_alter_info,
                                             const dd::Table *old_table_def,
                                             dd::Table *new_table_def) {
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);
  mark_trx_read_write();                 // ★ 标记读写事务 → 触发 InnoDB trx 分配
  return prepare_inplace_alter_table(altered_table, ha_alter_info,
                                     old_table_def, new_table_def);
}

bool handler::ha_commit_inplace_alter_table(TABLE *altered_table,
                                            Alter_inplace_info *ha_alter_info,
                                            bool commit,
                                            const dd::Table *old_table_def,
                                            dd::Table *new_table_def) {
  /*
    At this point we should have an exclusive metadata lock on the table.
    The exception is if we're about to roll back changes (commit= false).
  */
  assert(ha_thd()->mdl_context.owns_equal_or_stronger_lock(
             MDL_key::TABLE, table->s->db.str, table->s->table_name.str,
             MDL_EXCLUSIVE) || !commit);          // ★★ commit=true 时必须已持 X MDL
  return commit_inplace_alter_table(altered_table, ha_alter_info, commit,
                                    old_table_def, new_table_def);
}
```

```cpp
  bool ha_inplace_alter_table(TABLE *altered_table,      // ★ inline，定义在头文件
                              Alter_inplace_info *ha_alter_info,
                              const dd::Table *old_table_def,
                              dd::Table *new_table_def) {
    return inplace_alter_table(altered_table, ha_alter_info, old_table_def,
                               new_table_def);
  }

  void ha_notify_table_changed(Alter_inplace_info *ha_alter_info) {  // ★ inline
    notify_table_changed(ha_alter_info);
  }
```

**三个基类默认实现**（全是"空实现 + 成功"）：

```cpp
  virtual bool prepare_inplace_alter_table(TABLE *, Alter_inplace_info *,
                                           const dd::Table *, dd::Table *) {
    return false;
  }
  virtual bool inplace_alter_table(TABLE *, Alter_inplace_info *,
                                   const dd::Table *, dd::Table *) {
    return false;
  }
  virtual bool commit_inplace_alter_table(TABLE *, Alter_inplace_info *,
                                          bool, const dd::Table *, dd::Table *) {
    /* Nothing to commit/rollback, mark all handlers committed! */
    ha_alter_info->group_commit_ctx = nullptr;   // ★★ 必须清，否则分区 handler 认为还有未提交分区
    return false;
  }
  virtual void notify_table_changed(Alter_inplace_info *) {}
```

> ★★ **两条关键契约注释**：
> - `commit_inplace_alter_table()` 的 doxygen 明写 *"**This is the place where SE changes happen for instant algorithm.**"* —— INSTANT 的真正变更在 commit，不在 prepare/inplace。
> - *"If this function with commit= true reports error, it will be called again with commit= false."* —— **commit 失败会自动再调一次 rollback**。
> - 分区语义：*"commit_inplace_alter_table() might be called without prepare_inplace_alter_table() having been called first for a given partition"*。

**`inplace_alter_handler_ctx` 的生命周期契约**（`sql/handler.h:3168`）：

```cpp
/**
  In-place alter handler context.

  This is a superclass intended to be subclassed by individual handlers
  in order to store handler unique context between in-place alter API calls.

  The handler is responsible for creating the object. This can be done
  as early as during check_if_supported_inplace_alter().

  The SQL layer is responsible for destroying the object.   // ★★ 销毁方是 SQL 层

  @see Alter_inplace_info
*/
class inplace_alter_handler_ctx {
 public:
  inplace_alter_handler_ctx() = default;
  virtual void set_shared_data(const inplace_alter_handler_ctx *ctx) {}
  virtual ~inplace_alter_handler_ctx() = default;
};
```

```cpp
  ~Alter_inplace_info() { destroy(handler_ctx); }    // sql/handler.h:3580 —— 真销毁点
```

> ★★ **引擎不应自己 `delete` ctx**。`mysql_alter_table()` 栈上的 `Alter_inplace_info ha_alter_info(...)`（:17378）出作用域时自动 `destroy()`。

**`check_if_supported_inplace_alter()` 的基类默认实现**（`sql/handler.cc:4961`）——其他引擎（非 InnoDB）的默认白名单：

```cpp
enum_alter_inplace_result handler::check_if_supported_inplace_alter(
    TABLE *altered_table, Alter_inplace_info *ha_alter_info) {
  HA_CREATE_INFO *create_info = ha_alter_info->create_info;

  Alter_inplace_info::HA_ALTER_FLAGS inplace_offline_operations =
      Alter_inplace_info::ALTER_COLUMN_EQUAL_PACK_LENGTH |
      Alter_inplace_info::ALTER_COLUMN_NAME |
      Alter_inplace_info::ALTER_COLUMN_DEFAULT |
      Alter_inplace_info::CHANGE_CREATE_OPTION |
      Alter_inplace_info::ALTER_RENAME | Alter_inplace_info::RENAME_INDEX |
      Alter_inplace_info::ALTER_INDEX_COMMENT |
      Alter_inplace_info::CHANGE_INDEX_OPTION |
      Alter_inplace_info::ALTER_COLUMN_INDEX_LENGTH;

  /* Is there at least one operation that requires copy algorithm? */
  if (ha_alter_info->handler_flags & ~inplace_offline_operations)
    return HA_ALTER_INPLACE_NOT_SUPPORTED;
  ...
```

> ★ 注意：**默认实现也给 `HA_ALTER_INPLACE_NOT_SUPPORTED`**，而不是 `EXCLUSIVE_LOCK`。所以"不知道怎么办就走 COPY"是全局默认。

`handler` 的四个虚函数（sql/handler.h）：

| 接口 | MDL 阶段 | 干什么 |
|------|---------|--------|
| `check_if_supported_inplace_alter` | 决策 | 返回 `enum_alter_inplace_result` → 决定算法与 MDL 级别 |
| `prepare_inplace_alter_table` | ① X 准备 | 建 new_table、分配 row log |
| `inplace_alter_table` | ② S/NONE 执行 | bulk load 重建，DML 并发 |
| `commit_inplace_alter_table` | ③ X 提交 | apply row log + rename 切换 |

「算法判定」章那张 `enum_alter_inplace_result` 返回值表，就是 `check_if_supported_inplace_alter` 的返回值——**引擎通过这一个函数就决定了整个 DDL 的算法与锁行为**。InnoDB 的实现里长长的一串 `HA_ALTER_INPLACE_NOT_SUPPORTED` 拒绝清单（改列类型、删主键等），正是 COPY 作为兜底存在的直接证据。

#### `check_if_supported_inplace_alter` 的答复：INSTANT 判定与拒绝清单

函数最靠前的是 **INSTANT 判定**——它决定这条 DDL 能不能连表都不碰：

```cpp
// ① 行版本号已达上限（1 字节 row_version 撑不住了）→ INSTANT 不可能
if (...) {
  if (ha_alter_info->alter_info->requested_algorithm ==
      Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
    my_error(ER_INNODB_MAX_ROW_VERSION, MYF(0), m_prebuilt->table->name.m_name);
    return HA_ALTER_ERROR;                  // ★ 只有显式指定 INSTANT 才报错
  }
  /* INSTANT can't be done any more. Fall back to INPLACE. */
  break;                                    // ★ 默认模式：静默降级，继续往下判 INPLACE
}
// ② 加列后行大小可能超限 → 同一套处理
else if (!Instant_ddl_impl<dd::Table>::is_instant_add_drop_possible(
             ha_alter_info, table, altered_table, m_prebuilt->table)) {
  if (ha_alter_info->alter_info->requested_algorithm ==
      Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
    my_error(ER_INNODB_INSTANT_ADD_DROP_NOT_SUPPORTED_MAX_SIZE, MYF(0));
    return HA_ALTER_ERROR;
  }
  break;
}
// ③ 表可能非空（error_if_not_empty）→ 也只能回退
else if (ha_alter_info->error_if_not_empty) {
  break;
}
[[fallthrough]];
case Instant_Type::INSTANT_NO_CHANGE:
case Instant_Type::INSTANT_VIRTUAL_ONLY:
case Instant_Type::INSTANT_COLUMN_RENAME:
  ha_alter_info->handler_trivial_ctx = instant_type_to_int(instant_type);
  return HA_ALTER_INPLACE_INSTANT;            // ★ 真正的 INSTANT 出口
```

★ 注意这个模式：**INSTANT 判定失败时默认静默降级（`break` 继续往下走判 INPLACE），只有用户显式写了 `ALGORITHM=INSTANT` 才报 `HA_ALTER_ERROR`**——与 [`alter_table.md`](alter_table.md) 里"显式 ALGORITHM 做不到即 fail-fast"是同一条设计，只是这次的判定发生在引擎而不是 server。

再往下是一排 `HA_ALTER_INPLACE_NOT_SUPPORTED`，**每拒绝一条都会填 `unsupported_reason`**（供 server 报错时给出人话原因）：

```cpp
/* Only support NULL -> NOT NULL change if strict table sql_mode is set.
Fall back to COPY for conversion if not strict tables. */
if ((ha_alter_info->handler_flags & Alter_inplace_info::ALTER_COLUMN_NOT_NULLABLE) &&
    !thd_is_strict_mode(m_user_thd)) {
  ha_alter_info->unsupported_reason =
      innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_NOT_NULL);
  return HA_ALTER_INPLACE_NOT_SUPPORTED;      // 非严格模式下 NULL→NOT NULL 只能 COPY
}

/* DROP PRIMARY KEY is only allowed in combination with ADD PRIMARY KEY. */
if ((ha_alter_info->handler_flags & (Alter_inplace_info::ADD_PK_INDEX |
                                     Alter_inplace_info::DROP_PK_INDEX)) ==
    Alter_inplace_info::DROP_PK_INDEX) {
  ha_alter_info->unsupported_reason =
      innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_NOPK);
  return HA_ALTER_INPLACE_NOT_SUPPORTED;      // 单独删主键 → COPY
}

/* 列从 NOT NULL 改 NULL，而该列是隐式主键 → 只能 Copy */
if ((ha_alter_info->handler_flags & Alter_inplace_info::ALTER_COLUMN_NULLABLE)) {
  const uint my_primary_key = altered_table->s->primary_key;
  if (UNIV_UNLIKELY(my_primary_key >= MAX_KEY) &&
      !row_table_got_default_clust_index(m_prebuilt->table)) {
    ha_alter_info->unsupported_reason =
        innobase_get_err_msg(ER_PRIMARY_CANT_HAVE_NULL);
    return HA_ALTER_INPLACE_NOT_SUPPORTED;
  }
}
```

★ 这些"拒绝"就是 COPY 存在的理由：引擎每说一次 `NOT_SUPPORTED`，server 就把这条 DDL 推向 COPY 分支（完整判定链条见 [`ddl.md`](ddl.md)）。

#### ★ 一道白名单闸门：为什么"改列类型"恒走 COPY

`check_if_supported_inplace_alter` 里真正决定生死的不是那一长串逐个拒绝，而是**一条白名单**：凡是落在三个 flag 组之外的 handler flag，一律 `NOT_SUPPORTED`：

```cpp
  if (ha_alter_info->handler_flags &
      ~(INNOBASE_INPLACE_IGNORE | INNOBASE_ALTER_NOREBUILD | INNOBASE_ALTER_REBUILD)) {
    if (... & Alter_inplace_info::ALTER_STORED_COLUMN_TYPE) {
      /* 改列类型：给出 unsupported_reason */
    }
    return HA_ALTER_INPLACE_NOT_SUPPORTED;      // ★ 统一出口
  }
```

三个白名单组覆盖的语义分别是：

| flag 组 | 含义 | 代表操作 |
|---|---|---|
| `INNOBASE_INPLACE_IGNORE` | 引擎不关心（纯元数据） | 改默认值、改注释、改列可见性、CHECK 约束、索引选项 |
| `INNOBASE_ALTER_NOREBUILD` | 可原地做、**不重建表** | `ADD/DROP INDEX`、`RENAME INDEX`、改列名、外键操作 |
| `INNOBASE_ALTER_REBUILD` | 可原地做、**但要重建表** | `ADD/DROP PRIMARY KEY`、改 NULL 属性、改列顺序、加存储列、`RECREATE_TABLE` |

⇒ **改列类型（`ALTER_STORED_COLUMN_TYPE`）不在任何一组里**，所以它虽然有一段"专门的报错分支"（只是填 `unsupported_reason`），最终仍在同一句 `NOT_SUPPORTED` 被兜住——**改列类型恒走 COPY**，这就是它无法 INPLACE 的源码根据。

另外几处靠前的直接拒绝也值得一提：新 raw 表空间（`created_new_raw()`）→ `NOT_SUPPORTED`；`high_level_read_only || srv_force_recovery` → **`HA_ALTER_ERROR`**（这是本函数**唯一**的"硬错误"档，其余拒绝都是"我不支持，你换算法"）；加密属性变更（none ↔ 非 none）→ `NOT_SUPPORTED`；字段数超 `REC_MAX_N_USER_FIELDS` 时**故意不拦**，让 server 走 COPY 再失败（这样报错更准确）。

#### ★★ 反直觉：InnoDB 从不返回 `NO_LOCK` / `SHARED_LOCK` / `EXCLUSIVE_LOCK`

**这就是"MDL 三段式"的源码映射表**（`sql/sql_table.cc:13349` 升锁 + `:13546` 降锁）：

| `inplace_supported` | prepare **前** | prepare **后**（主阶段） | `ha_alter_info->online` |
|---|---|---|---|
| `HA_ALTER_INPLACE_INSTANT` | **不升级** | 不降级 | false |
| `HA_ALTER_INPLACE_NO_LOCK` | **不升级**（S） | 不降级（S） | **true** |
| `HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE` | 升 **X** | 降 **MDL_SHARED_UPGRADABLE** | **true** |
| `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE` | 升 **X** | 降 **MDL_SHARED_NO_WRITE** | false |
| `HA_ALTER_INPLACE_SHARED_LOCK` | 升 **MDL_SHARED_NO_WRITE** | 保持 SNW | false |
| `HA_ALTER_INPLACE_EXCLUSIVE_LOCK` | 升 **X** | 保持 X | false |

**升锁段原文**（`sql_table.cc:13349`）：

```cpp
  /*
    Upgrade to EXCLUSIVE lock if:
    - This is requested by the storage engine
    - Or the storage engine needs exclusive lock for just the prepare phase
    - Or requested by the user
    ...
  */
  if (inplace_supported == HA_ALTER_INPLACE_EXCLUSIVE_LOCK ||
      ((inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
        inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) &&
       (thd->locked_tables_mode == LTM_LOCK_TABLES ||
        thd->locked_tables_mode == LTM_PRELOCKED_UNDER_LOCK_TABLES)) ||
      alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE) {
    if (wait_while_table_is_used(thd, table, HA_EXTRA_FORCE_REOPEN))
      goto cleanup;                                  // ★ 走 wait_while_table_is_used
    /*
      Get rid of all TABLE instances belonging to this thread
      except one to be used for in-place ALTER TABLE.
      This is mostly needed to satisfy InnoDB assumptions/asserts.
    */
    close_all_tables_for_name(thd, table->s, false, table);
    reopen_tables = true;
  } else if (inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
             inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) {
    /*
      Storage engine has requested exclusive lock only for prepare phase
      and we are not under LOCK TABLES.
      Don't mark TABLE_SHARE as old in this case, as this won't allow opening
      of table by other threads during main phase of in-place ALTER TABLE.
    */
    if (thd->mdl_context.upgrade_shared_lock(table->mdl_ticket, MDL_EXCLUSIVE,
                                             thd->variables.lock_wait_timeout))
      goto cleanup;                                  // ★ 走 MDL upgrade

    tdc_remove_table(thd, TDC_RT_REMOVE_NOT_OWN_KEEP_SHARE, table->s->db.str,
                     table->s->table_name.str, false);
    //  ★★ 只摘 TDC 不摘 share —— 保证主阶段别人还能开表
  }
```

**降锁段原文**（`sql_table.cc:13546`）：

```cpp
    /*
      Downgrade the lock if storage engine has told us that exclusive lock was
      necessary only for prepare phase (unless we are not under LOCK TABLES) and
      user has not explicitly requested exclusive lock.
    */
    if ((inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
         inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) &&
        !(thd->locked_tables_mode == LTM_LOCK_TABLES ||
          thd->locked_tables_mode == LTM_PRELOCKED_UNDER_LOCK_TABLES) &&
        (alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE)) {
      /* If storage engine or user requested shared lock downgrade to SNW. */
      if (inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
          alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED)
        table->mdl_ticket->downgrade_lock(MDL_SHARED_NO_WRITE);
      else {
        assert(inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE);
        table->mdl_ticket->downgrade_lock(MDL_SHARED_UPGRADABLE);
        //  ★★ 降回开表时的 SU —— 这就是 LOCK=NONE 的真正含义
      }
    }
```

**`online` 标志的置位点**（`sql_table.cc:13509`）——它是给 SE 看的"真·在线"标记：

```cpp
  switch (inplace_supported) {
    case HA_ALTER_INPLACE_NO_LOCK:
    case HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE:
      switch (alter_info->requested_lock) {
        case Alter_info::ALTER_TABLE_LOCK_DEFAULT:
        case Alter_info::ALTER_TABLE_LOCK_NONE:
          ha_alter_info->online = true;      // ★★ 只有这两种组合才置 true
          break;
        case Alter_info::ALTER_TABLE_LOCK_SHARED:
        case Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE:
          break;
      }
      break;
    ...
```

> ★★ **`online = true` 是 InnoDB 决定"要不要分配 row log"的唯一依据**（`prepare_inplace_alter_table_dict()` 里 `if (new_clustered || !ctx->online || ...) { /* No need to allocate a modification log */ }`）。
>
> ★ **commit 前的二次升级用的是 `HA_EXTRA_PREPARE_FOR_RENAME`，不是 `HA_EXTRA_FORCE_REOPEN`**：
> ```cpp
>     // Upgrade to EXCLUSIVE before commit.
>     if (wait_while_table_is_used(thd, table, HA_EXTRA_PREPARE_FOR_RENAME))
>       goto rollback;
> ```
> 两者都被 InnoDB 的 `extra()` 静默忽略（`default:` 分支）——它们只是 server 侧"驱赶其它 TABLE 实例"的意图标记。

**三个 PFS stage 就是这三个阶段**（`sql_table.cc:13507/13568/13610`）：

```cpp
  THD_STAGE_INFO(thd, stage_alter_inplace_prepare);   // "preparing for alter table"
  ...
  THD_STAGE_INFO(thd, stage_alter_inplace);           // "altering table"
  ...
  THD_STAGE_INFO(thd, stage_alter_inplace_commit);    // "committing alter table to storage engine"
```

翻阅本函数所有 34 处 `return`，它实际**只返回 5 个值中的 4 类**：`NOT_SUPPORTED`、`HA_ALTER_ERROR`、`INSTANT`，以及：

```cpp
  return online ? HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE
                : HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE;
```

★ 也就是说——**所谓 `LOCK=NONE` 在 InnoDB 里是 `NO_LOCK_AFTER_PREPARE`，不是不带 `AFTER_PREPARE` 后缀的 `NO_LOCK`**。两者的差别正是理解在线 DDL 的关键：带后缀意味着 **prepare 阶段与 commit 阶段仍然要拿排他 MDL**，只是"主阶段"（扫描-构建）不加锁。任何 online DDL 都必然有这两个短暂的排他窗口——这正是长事务能把 online DDL 卡住的原因（详见 [`ddl.md`](ddl.md) 的 MDL 三段式）。

而 `online` 被置为 false（从而降级到 `SHARED_LOCK_AFTER_PREPARE`，即主阶段只允许读、不允许写）的情形有五类，源码里逐条列出：

1. 新增 AUTO_INCREMENT 列；
2. 在**新加的虚拟列**上建索引；
3. `ADD SPATIAL INDEX`；
4. rebuild 且表中保留 FTS / SPATIAL 索引（若 `m_prebuilt->table->fts` 已存在则直接 `NOT_SUPPORTED`）；
5. 新增 FULLTEXT 索引。

⇒ 注意：**`online = false` 不等于拒绝**——它仍走 INPLACE，只是主阶段不允许并发写。这与"引擎不支持"是两回事。

#### handlerton 的 flags 决定 DDL 路径

```cpp
#define HTON_ALTER_NOT_SUPPORTED (1 << 1)  // 引擎不支持 alter
#define HTON_CAN_RECREATE (1 << 2)         // Delete all is used for truncate
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12)

inline bool ddl_is_atomic(const handlerton *hton) {        // handler.h
  return (hton->flags & HTON_SUPPORTS_ATOMIC_DDL) != 0;
}
```

InnoDB 的声明（ha_innodb.cc）：

```cpp
innobase_hton->flags = HTON_SUPPORTS_EXTENDED_KEYS | HTON_SUPPORTS_FOREIGN_KEYS |
                       HTON_SUPPORTS_ATOMIC_DDL | HTON_CAN_RECREATE |
                       HTON_SUPPORTS_SECONDARY_ENGINE | HTON_SUPPORTS_TABLE_ENCRYPTION |
                       HTON_SUPPORTS_GENERATED_INVISIBLE_PK;
```

两处直接呼应本目录其他篇：
- **`HTON_SUPPORTS_ATOMIC_DDL`** → `copy_data_between_tables` 里"是否提前提交"的分岔（见 [`copy_ddl.md`](copy_ddl.md)）
- **`HTON_CAN_RECREATE`**（注释 "Delete all is used for truncate"）→ TRUNCATE 走 rename + drop + create 路径（见 [`ddl.md`](ddl.md)）

### ha_extra：DDL 给引擎下的"行为开关"

`handler::extra`（表级虚函数，与 `external_lock` **平级**）是 DDL 用来调整引擎行为的旁路通道。COPY DDL 里用到一对：

- `HA_EXTRA_BEGIN_ALTER_COPY` → `skip_alter_undo = 1`
- `HA_EXTRA_END_ALTER_COPY` → `alter_stats_rebuild()` 重算统计 + 置 0

其作用是令 COPY 往中间表插入时**同时免掉 undo 与行锁**（下游 `row_ins` 据此置 `BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG`）——这是 COPY 算法最关键的优化，部分抵消了它走 SQL 层的劣势。

完整逐行剖析（含 `skip_alter_undo` 在 row0ins.cc 的消费点、roll_ptr 伪造 `1ULL<<55`、"只对 INSERT 生效"的断言约束、以及 `extra()` 为何绝不能碰 trx）见 [`../../server/handler.md`](../../server/handler.md) 的「handler::extra」章。

### online DDL 与 row log（引擎侧）

> 上一节的 DDL log 与这里的 row log 名字都带 log，但**完全是两回事**：DDL log 管"DDL 崩溃恢复/原子性"，row log 管"online DDL 重建期间的 DML 并发"。逐项对照见本节末尾的表格。

#### online DDL 的两种形态（先分清，否则会混淆）

**两种形态在 `prepare_inplace_alter_table_dict()` 里的分岔**（`handler0alter.cc:4851` / `:4947`）：

| | **① add-index 形态**（`!new_clustered`） | **② rebuild 形态**（`new_clustered`） |
|---|---|---|
| row log 挂在哪 | **每个新增二级索引** `index->online_log` | **旧表的聚簇索引** `clust_index->online_log` |
| `row_log_allocate()` 第二参 | `nullptr` | **`ctx->new_table`** |
| 回放次数 | **1 次**（`Builder::finish()` 里） | **2 次**（inplace 末尾 + commit） |
| 回放函数 | `row_log_apply()` | `row_log_table_apply()` |
| `new_table` 与 `old_table` | **同一对象** | **不同对象** |

```cpp
  } else {                                          // ★ add-index 形态
    assert(!innobase_need_rebuild(ha_alter_info));  // ★★ 不变式
    assert(old_table->s->primary_key == altered_table->s->primary_key);
    ...
  }
```

```cpp
    } else {
      rw_lock_x_lock(&ctx->add_index[a]->lock, UT_LOCATION_HERE);
      bool ok = row_log_allocate(ctx->add_index[a], nullptr, true, nullptr,
                                 nullptr, path);
      //                              ★ table=NULL → 记的是"索引级"变化（mrec 是索引元组）
      rw_lock_x_unlock(&ctx->add_index[a]->lock);
      if (!ok) { error = DB_OUT_OF_MEMORY; goto error_handling; }
    }
```

```cpp
  if (new_clustered) {                              // ★ rebuild 形态
    dict_index_t *clust_index = user_table->first_index();
    dict_index_t *new_clust_index = ctx->new_table->first_index();
    ctx->skip_pk_sort =
        innobase_pk_order_preserved(ctx->col_map, clust_index, new_clust_index);

    if (ctx->online) {
      /* Allocate a log for online table rebuild. */
      rw_lock_x_lock(&clust_index->lock, UT_LOCATION_HERE);
      bool ok = row_log_allocate(
          clust_index, ctx->new_table,       // ★★ table=new_table → 记的是"表级"变化（mrec 是完整行）
          !(ha_alter_info->handler_flags & Alter_inplace_info::ADD_PK_INDEX),
          ctx->add_cols, ctx->col_map, path);
      rw_lock_x_unlock(&clust_index->lock);
      ...
    }
  }

  if (ctx->online) {
    /* Assign a consistent read view for the index build scan. */
    trx_assign_read_view(ctx->prebuilt->trx);       // ★ 扫描用的 read view 在 prepare 末就固定
  }
```

**★ add-index 的 row log 只回放一次，且在 `Builder::finish()` 里**（`ddl/ddl0builder.cc:1968`）——**不在 `handler0alter.cc`**：

```cpp
  if (err == DB_SUCCESS) {
    write_redo(m_index);
    DEBUG_SYNC(m_ctx.thd(), "row_log_apply_before");
    err = row_log_apply(m_ctx.m_trx, m_index, m_ctx.m_table, m_local_stage);
    //                                        ★★ add-index 的唯一回放点
    DEBUG_SYNC(m_ctx.thd(), "row_log_apply_after");
  }
```

> ⚠️ **常见误解**：以为两种形态都在 `handler0alter.cc` 里回放。**add-index 的回放藏在 `ddl0builder.cc`**，这是 8.0 重构后最容易漏掉的一处。

**★ rebuild 的第一次回放：`clean_up` lambda**（`handler0alter.cc:6235`）：

```cpp
  auto clean_up = [&](dberr_t err) -> bool {
    DEBUG_SYNC_C("alter_table_update_log");

    if (err == DB_SUCCESS && ctx->online && ctx->need_rebuild()) {
      DEBUG_SYNC_C("row_log_table_apply1_before");
      err = row_log_table_apply(ctx->thr, m_prebuilt->table, altered_table,
                                ctx->m_stage);   // ★ 第 1 次 apply（并发 DML 仍可写）
    }
```

> ★★ **为什么要两次**：执行阶段（全表扫描重建）可能持续很久，期间 DML 一直在往 row log 里塞。如果攒到 commit 才一次性 apply，commit 会阻塞极长时间。**第一次 apply 时新 DML 仍可并发写；第二次 apply 时必须阻止新 DML**（否则 rename 前后状态不一致）——这就是 online DDL 那个"短暂阻塞点"。

**`inplace_alter_table_impl()` 的完整上下文**（`handler0alter.cc:6105`）：

```cpp
  if (!(ha_alter_info->handler_flags & INNOBASE_ALTER_DATA) ||
      is_instant(ha_alter_info)) {
    return all_ok();                     // ★ 无需动数据 / INSTANT → 直接返回
  }
  if (((ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) ==
           Alter_inplace_info::CHANGE_CREATE_OPTION &&
       !innobase_need_rebuild(ha_alter_info))) {
    return all_ok();                     // ★ 只改 CREATE OPTION 且不需 rebuild
  }
  ...
  const auto trx = m_prebuilt->trx;
  const auto old_isolation_level = trx->isolation_level;

  if (ctx->online &&
      trx->isolation_level != trx_t::isolation_level_t::REPEATABLE_READ) {
    /* ... When rebuilding the table online, row_log_table_apply() must not see
    a newer state of the table when applying the log. This is mainly to
    prevent false duplicate key errors, because the log will identify records
    by the PRIMARY KEY, and also to prevent unsafe BLOB access. ... */
    trx->isolation_level = trx_t::isolation_level_t::REPEATABLE_READ;
  }                                      // ★★ 隔离级别"强制提升"

  ddl::Context ddl(trx, m_prebuilt->table, ctx->new_table, ctx->online,
                   ctx->add_index, ctx->add_key_numbers, ctx->num_to_add_index,
                   altered_table, ctx->add_cols, ctx->col_map, ctx->add_autoinc,
                   ctx->sequence, ctx->skip_pk_sort, ctx->m_stage, add_v,
                   eval_table, thd_ddl_buffer_size(m_prebuilt->trx->mysql_thd),
                   thd_ddl_threads(m_prebuilt->trx->mysql_thd));

  const auto err = clean_up(ddl.build());
  //                        ★★ build() → Loader::build_all() → Builder 并行扫描/排序/加载

  trx->isolation_level = old_isolation_level;   // ★ 还原
  return err;
```

**`ddl::Context` 构造里的两条硬断言**（`ddl/ddl0ctx.cc:71`）：

```cpp
  ut_a(!m_online ||
       trx->isolation_level == trx_t::isolation_level_t::REPEATABLE_READ);
  //  ★★ 契约：online ⇒ 调用方必须已把隔离级别提到 RR

  m_need_observer = m_old_table != m_new_table;
  //  ★ rebuild ⇒ 必须 Flush_observer（bulk load 关了 redo，apply row log 前必须刷脏页）
```

> ⚠️ **不存在 `ha_set_iso_level()`**——真实做法是**直接赋值** `trx->isolation_level = ...`，且**只升不降**（只在 `!= RR` 时改），在 `inplace_alter_table_impl()` 里改、`build()` 返回后立刻还原。
>
> ⚠️ **prepare/commit 里都没有 `trx_set_dict_operation(TRX_DICT_OP_INDEX)`**——8.0.39 prepare 只调 `trx_start_if_not_started_xa(trx, true)`；`trx->ddl_operation` 由 `trx0trx.cc:1395` 从 THD 推导。

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

#### row log 是什么

row log（online log）是 InnoDB online DDL 期间记录用户 DML 的日志，只服务于 online 场景（`!ctx->online` 时不分配）。结构是**「内存块 + 磁盘临时文件」两段式**：

- 结构 `row_log_t`（row0log.cc），挂在 `dict_index_t::online_log`（dict0mem.h）上，字段含 `file`（`ddl::Unique_os_file_descriptor` 临时文件）、`mutex`、`tail`（写者）、`head`（读者）、`blobs`（off-page 列 page 号映射）、`table`、`same_pk`、`add_cols`、`col_map`（旧列 id → 新列 id）；

分配函数 `row_log_allocate` 把这些字段一次性铺开，是整个 row log 机制的起点：

```cpp
bool row_log_allocate(dict_index_t *index, dict_table_t *table, bool same_pk,
                      const dtuple_t *add_cols, const ulint *col_map,
                      const char *path) {
  ut_ad(!dict_index_is_online_ddl(index));
  ut_ad(index->is_clustered() == !!table);    // ★ 两种形态的判据：聚簇 ⇔ 传了 table
  ut_ad(!table || index->table != table);     // rebuild：log 挂在旧表索引上，table 指向新表
  ut_ad(same_pk || table);
  ut_ad(!table || col_map);                   // rebuild 必须带新旧列映射
  ut_ad(rw_lock_own(dict_index_get_lock(index), RW_LOCK_X));  // 必须在索引 X 锁下分配

  log = ut::new_withkey<row_log_t>(UT_NEW_THIS_FILE_PSI_KEY);
  mutex_create(LATCH_ID_INDEX_ONLINE_LOG, &log->mutex);

  log->blobs = nullptr;
  log->table = table;         // ★ add index 时 nullptr；rebuild 时是新表
  log->same_pk = same_pk;     // ★ 主键是否不变——决定 update 要不要记 old pk
  log->add_cols = add_cols;
  log->col_map = col_map;     // 旧列 id → 新列 id
  log->error = DB_SUCCESS;
  log->max_trx = 0;
  log->tail.blocks = log->tail.bytes = 0;   // 写者位置：并发 DML 推进 tail
  log->tail.total = 0;
  log->tail.block = log->head.block = nullptr;
  log->head.blocks = log->head.bytes = 0;   // 读者位置：apply 推进 head
  log->head.total = 0;
  log->path = path;           // 溢出用的临时文件目录
  log->n_old_col = index->table->n_cols;
  log->n_old_vcol = index->table->n_v_cols;

  dict_index_set_online_status(index, ONLINE_INDEX_CREATION);  // ★ 状态机：进入"建设中"
  index->online_log = log;

  MONITOR_ATOMIC_INC(MONITOR_ONLINE_CREATE_INDEX);   // ddl_online_create_index 计数器
  return true;
}
```

三个要点：① **`tail` / `head` 双指针**就是生产者-消费者的全部——DML 推进 `tail`，apply 推进 `head`，两者差值就是积压量（`row_log_estimate_work` 用它估算进度）；② `same_pk` 直接对应前面记录格式表里"`ROW_T_UPDATE` 是否携带 old pk"；③ **状态机在这里从 `COMPLETE` 翻到 `CREATION`**（见「状态机」节），此后该索引不接受查找、所有 DML 转写 row log。
- 内存块由 `row_log_block_allocate` 分配，大小 = `srv_sort_buf_size`；写满后经 `row_log_tmpfile` 溢出到磁盘临时文件；

**写满溢出与"超限"判定都在收尾函数 `row_log_table_close_func` 里**——这也是 `innodb_online_alter_log_max_size` 的源码落点：

```cpp
static void row_log_table_close_func(row_log_t *log,
                                     IF_DEBUG(const byte *b, ) ulint size,
                                     ulint avail) {
  ut_ad(mutex_own(&log->mutex));

  if (size >= avail) {                    // ① 当前块装不下了 → 溢出到临时文件
    dberr_t err;
    IORequest request(IORequest::ROW_LOG | IORequest::WRITE);
    const os_offset_t byte_offset =
        (os_offset_t)log->tail.blocks * srv_sort_buf_size;

    // ★ 超限判定：块偏移 + 块大小 超过上限 → 打错误位
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) {
      goto write_failed;
    }

    if (size == avail) {
      ut_ad(b == &log->tail.block[srv_sort_buf_size]);
    } else {
      ut_ad(b == log->tail.buf + size);
      memcpy(log->tail.block + log->tail.bytes, log->tail.buf, avail);
    }

    if (!row_log_tmpfile(log)) {          // 确保临时文件已创建
      log->error = DB_OUT_OF_MEMORY;
      goto err_exit;
    }

    err = os_file_write_int_fd(request, "(modification log)", log->file.get(),
                               log->tail.block, byte_offset, srv_sort_buf_size);

    log->tail.blocks++;                   // 块计数 +1
    if (err != DB_SUCCESS) {
    write_failed:
      log->error = DB_ONLINE_LOG_TOO_BIG; // ★ 超限：只打标记，不立即中断
    }
    memcpy(log->tail.block, log->tail.buf + avail, size - avail);
    log->tail.bytes = size - avail;
  } else {
    log->tail.bytes += size;              // 还装得下 → 只在内存块里推进
  }

  log->tail.total += size;                // ★ 写者指针：每条 DML 记录都推进它
err_exit:
  mutex_exit(&log->mutex);
}
```

★ 三个要点：① **`srv_online_max_size` 限制的是临时文件的总大小**（`tail.blocks × srv_sort_buf_size`），内存块始终只有一份；② 超限时**只给 `log->error` 打 `DB_ONLINE_LOG_TOO_BIG` 标记而不立即中断 DML**——这与 WL#5526"DML 不受影响"的说法一致，错误要等后续 apply 读到 error 位才上报；③ `tail.total` 就是"追平断言"里 `head.total == tail.total` 的写者计数，**每条 DML 记录都在这里 +size**。
  - **临时文件命名**：`row_log_tmpfile` → `ddl::file_create_low` → `innobase_mysql_tmpfile` → `mysql_tmpfile_path`，磁盘上真实文件名是 **`ib` 前缀 + 6 位随机字符**（如 `ibA1b2C3`）；目录来自 `thd_innodb_tmpdir(thd)`，为空则退回全局 `--tmpdir`。PFS 中显示的可读名 "Innodb Merge Temp File" 只是监控用名，与磁盘真实名不同；
- 读者（apply）追上写者时 reset 计数并 truncate 文件——生产者-消费者 + 环形缓冲 + 磁盘溢出；
- 记录点：DML 执行时若 `index->online_log != nullptr`，由 `row_log_table_insert / update / delete` 写入；
- 回放点：`row_log_table_apply` 按主键顺序把 row log 逐条 apply 到新表。

#### 记录格式（mrec）

`row_log_table_low` 是 insert/update 的编码入口，一条记录格式：

```
[op:1字节][old_pk:可选][extra_size:1或2字节][记录头extra字节][记录data][虚拟列:可选]
```

- 操作类型枚举（row0log.cc）：table rebuild 用 `ROW_T_INSERT=0x41 / ROW_T_UPDATE=0x42 / ROW_T_DELETE=0x43`；secondary index online create 用 `ROW_OP_INSERT=0x61 / ROW_OP_DELETE=0x62`；
- 记录头 `ROW_LOG_HEADER_SIZE=2`；`extra_size` 变长编码（<0x80 一字节，否则 `0x80|high` + `low` 两字节）；
- `old_pk` 仅在主键变化（`!same_pk`）的 update 里携带；

**回放时怎么在新表上定位"这一行"**——靠 `row_log_table_get_pk_col` 取主键值：

```cpp
static dberr_t row_log_table_get_pk_col(dict_index_t *index,
                                        const dict_field_t *ifield,
                                        dfield_t *dfield, mem_heap_t *heap,
                                        const rec_t *rec, const ulint *offsets,
                                        ulint i, const page_size_t &page_size,
                                        ulint max_len) {
  const byte *field;
  ulint len;

  field = rec_get_nth_field_instant(rec, offsets, i, index, &len);

  ut_ad(len != UNIV_SQL_ADD_COL_DEFAULT);

  if (field == nullptr) {
    // ★ 主键列是 instant 加进来的，老行里根本没有这个字段
    ib::error(ER_IB_PRIMARY_KEY_IS_INSTANT, index->get_field(i)->name(),
              ulong(i), index->table_name, index->name());
    return (DB_INDEX_CORRUPT);
  }

  if (len == UNIV_SQL_NULL) {
    return (DB_INVALID_NULL);                    // 主键不能为 NULL
  }

  if (rec_offs_nth_extern(index, offsets, i)) {  // 外部存储（blob）取前缀
    ...
    len = lob::btr_copy_externally_stored_field_prefix(...);
    if (len >= max_len + 1) {
      return (DB_TOO_BIG_INDEX_COL);
    }
  }
```

★ 这段有个 **online 与 instant 的交叉点**，值得单独记：**主键列若是 instant 加进来的，老行里不存在该字段**，回放时 `rec_get_nth_field_instant` 取不到值 → 报 `ER_IB_PRIMARY_KEY_IS_INSTANT` 并返回 `DB_INDEX_CORRUPT`。这正是官方限制 *"Instantly added columns cannot be PK, unless a table rebuild"*（见 [`instant_ddl.md`](instant_ddl.md)）的**实现层原因**——不是策略上不想支持，而是回放路径根本定位不到行。
- 记录体用 `rec_serialize_dtuple`（mrec 编码）序列化，回放时 `row_log_table_apply_convert_mrec` 反解。

**三种操作各自记什么**（格式上的关键差异）：

| 操作 | old pk | record data | 虚拟列 | 原因 |
|------|--------|-------------|--------|------|
| `ROW_T_INSERT` | 否 | 是 | 有则记 | 插入新表靠 `col_map` 映射即可定位，无需旧表主键 |
| `ROW_T_UPDATE` | **可能**（主键变化时） | 是 | 有则记 | 新旧表主键不同时，需 old pk 在新表上定位待更新行 |
| `ROW_T_DELETE` | 是 | **否** | **否** | 应用时只需在新表按主键打删除标记，无需行内容 |

**temp 格式（省空间的紧凑行格式）**：row log 里的 record data 用的是 **temp 格式**而非完整 compact/redundant 格式——它**省略记录头信息**，只保留变长字段长度列表 + NULL 标志位 + 字段数据。相关接口：`rec_get_converted_size_temp`（算大小）、`rec_convert_dtuple_to_temp`（tuple→temp）、`rec_init_offsets_temp` / `rec_init_null_and_len_temp`（解析）。temp 格式也用于 sort buffer，目的同样是省空间。

**instant add column 对格式的影响**：做过 instant add column 的表，行记录头多了 `INSTANT_FLAG` 标志位（表示是否存在"字段数量"字段）。row log 记录这类表的 record 时需**额外记 info bits**，解析时（`rec_init_null_and_len_temp`）用 index 上的 instant 标志辅助判断 temp 记录里是否含 info bits。

#### 与 DDL log 的区别

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

#### 引擎侧执行阶段的收尾：第一次 apply 在哪触发

在 `ha_innobase::inplace_alter_table_impl` 的 `clean_up` lambda 里——**构建一完成就做第一次 apply**：

```cpp
auto clean_up = [&](dberr_t err) -> bool {
  DEBUG_SYNC_C("alter_table_update_log");

  if (err == DB_SUCCESS && ctx->online && ctx->need_rebuild()) {
    DEBUG_SYNC_C("row_log_table_apply1_before");
    /*
      为什么要分两次？因为执行阶段（全表扫描重建）可能持续很久，期间 DML 一直在往
      row log 里塞。如果攒到 commit 阶段才一次性 apply，commit 阶段会阻塞极长时间
      （apply 时要加锁阻止新 DML）。所以在执行阶段末尾先 apply 一轮，把大头消化掉，
      commit 阶段只需 apply 少量增量，把最终的阻塞窗口压缩到毫秒级。

      第一次 apply 时，新 DML 仍可继续并发写；
      第二次 apply 时，必须阻止新 DML（否则 rename 前后状态不一致）——这就是 online
      DDL 那个"短暂阻塞点"。
    */
    err = row_log_table_apply(ctx->thr, m_prebuilt->table, altered_table,
                              ctx->m_stage);
  }
  ...
```

★ 三个条件同时成立才做第一次 apply：`err == DB_SUCCESS`（构建成功）、`ctx->online`（online 才有 row log）、`ctx->need_rebuild()`。

⇒ 注意最后那个条件：**ADD INDEX 不是 rebuild，所以只 apply 一次**（`row_log_apply`，在 `ddl0builder.cc` 里）；只有重建表才 apply 两次。这与前面「两种形态」表里"回放次数"一栏完全对应。

★ 那段源码注释把两次 apply 的动机说得比任何外部资料都清楚：**第一次不是为了追平，而是为了压缩 commit 阶段的阻塞窗口**——把大头提前消化掉，让最后的排他窗口只有毫秒级。

> 构建本身（扫描—排序—建树三期、并行与终止检测）见 [`ddl_engine.md`](ddl_engine.md)，本篇不重复。

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

```cpp
dberr_t row_log_table_apply(...) {
  rw_lock_x_lock(dict_index_get_lock(clust_index), UT_LOCATION_HERE);  // ① 聚簇索引加 X 锁

  if (!clust_index->online_log) {
    ut_ad(dict_index_get_online_status(clust_index) == ONLINE_INDEX_COMPLETE);
    /* This function should not be called unless
    rebuilding a table online. Build in some fault tolerance. */
    error = DB_ERROR;
  } else {
    ddl::Dup dup = {clust_index, table, clust_index->online_log->col_map, 0};
    error = row_log_table_apply_ops(thr, &dup, stage);

    // ② ★ 追平断言：成功返回时，读者必须已经追上写者
    ut_ad(error != DB_SUCCESS || clust_index->online_log->head.total ==
                                     clust_index->online_log->tail.total);
  }

  rw_lock_x_unlock(dict_index_get_lock(clust_index));
  return error;
}
```

★ 那句 `ut_ad(... head.total == tail.total)` 是"第二次 apply 必须追平"的**代码级证据**：它断言的是"成功即追平"。反过来看，第一次 apply 之所以能在 DML 并发下执行，正因为**它没有这个断言**——那时允许 head 追不上 tail，差额留给第二次兜底。一个断言把两次 apply 的语义差异钉得死死的；真正的差异在 apply 之后：

- 第一次 apply 是"清库存"，不是"追平"——apply 完放行，后续 DML 继续进 row log，留给第二次兜底；
- 第二次 apply 是"最后一次追平"（代码断言 `head.total == tail.total`），追平后**立即 rename 切表**。此刻若还放行 DML 落旧表，这些 DML 既没进 row log、也不会 apply 到新表，rename 后旧表被丢弃 → DML 真丢。所以第二次 apply 必须和 rename 绑成一个"不许插入"的原子窗口。

**MDL 为什么必须升级（根源）**：MDL 保护的是"表定义（元数据）的可见性"。执行阶段新结构在构建、旧结构仍是唯一对外可见的，apply 只是"追赶数据"、**不改变任何可见性**，MDL 保持 SHARED 即可；提交阶段要**切换**（rebuild 是 rename 换表，add index 是让新索引进入 `ONLINE_INDEX_COMPLETE`），切换的本质是"改变哪个结构对外可见"，此刻若还有 DML 持有旧定义在写，写入会落到即将被丢弃/失效的结构上 → 数据丢失，所以切换必须在"无人持有旧定义"的时刻完成 → 升级 MDL 为 EXCLUSIVE。

还有一条更硬的约束直接决定了先后顺序：**第二次 apply 要"追平"（`head.total == tail.total`），而只要 DML 还在写，row log 的 tail 就一直涨，head 永远追不上——永远追不平。** 所以因果是"要追平 + 切换 → 必须先让 DML 停 → 升级 MDL EXCLUSIVE → 才能第二次 apply 追平 → rename"，而不是"因为是第二次所以升级"。第一次 apply 不需要追平（有第二次兜底），因此能在 DML 并发下做。

#### add index 形态的回放：`row_log_apply_ops`

前面讲的都是 rebuild 形态（`row_log_table_apply`）。**add index 走的是另一个函数** `row_log_apply_ops`，结构相似但有几处关键差异：

```cpp
static dberr_t row_log_apply_ops(const trx_t *trx, dict_index_t *index,
                                 ddl::Dup *dup, Alter_stage *stage) {
  ut_ad(dict_index_is_online_ddl(index));
  ut_ad(!index->is_committed());                     // ★ 索引尚未"发布"
  ut_ad(rw_lock_own(dict_index_get_lock(index), RW_LOCK_X));
  ut_ad(index->online_log);
  ...
next_block:
  ut_ad(rw_lock_own(dict_index_get_lock(index), RW_LOCK_X));

  stage->inc(row_log_progress_inc_per_block());      // ★ PFS 进度：每处理一块 +1

  if (trx_is_interrupted(trx)) {
    goto interrupted;
  }

  error = index->online_log->error;                  // ★ 超限错误位在这里才被读到
  if (error != DB_SUCCESS) {
    goto func_exit;
  }

  if (index->is_corrupted()) {
    error = DB_INDEX_CORRUPT;
    goto func_exit;
  }

  if (UNIV_UNLIKELY(index->online_log->head.blocks >
                    index->online_log->tail.blocks)) {
    ib::error(ER_IB_MSG_962) << "Unexpected end of temporary file for index "
                             << index->name;
```

★ 两点：① 前面说"超限时只给 `log->error` 打标记"，**读取它的地方就在这里**——所以 `ER_INNODB_ONLINE_LOG_TOO_BIG` 是在**回放阶段**才报出来的，而不是 DML 写下那条记录的瞬间；② `head.blocks > tail.blocks` 这个防御性检查表明读者块数永远不会超过写者，一旦超过说明临时文件已经损坏。

> 与 rebuild 形态的另一个差别：add index 只回放**一次**（在 `ddl0builder.cc` 里，DEBUG_SYNC 点 `row_log_apply_before`），且**没有 rename**——回放完把索引状态翻到 `ONLINE_INDEX_COMPLETE` 就算发布。

### commit 与 rename（仅 rebuild 场景）

**rename 只发生在 rebuild 场景**——加索引的 online DDL 没有 rename（新索引置 `ONLINE_INDEX_COMPLETE` 即生效）。`commit_inplace_alter_table` 收尾由 `commit_cache_rebuild`（handler0alter.cc，内有 `assert(ctx->need_rebuild())`）完成：

```cpp
/* We already committed and redo logged the renames, so this must succeed. */
dict_table_rename_in_cache(ctx->old_table, ctx->tmp_name, false);  // 旧表 → 临时名
dict_table_rename_in_cache(ctx->new_table, old_name, false);       // 新表 → 正式名
```

- rename 是"新表上线"的原子切换点：新表改正式名、旧表改临时名（稍后物理删除）；
- **rename 是 redo logged 的**（注释原文 "redo logged the renames"）——这才是 DDL 的**真正持久化点**，rename 落盘后崩溃也能保证"新表已上线"；
- 因此 **rebuild 场景的** commit 阶段 = 第二次 apply（让新表数据完整）+ rename（切换 + 落盘），二者必须在 MDL exclusive 下连续完成；非 rebuild 的 online DDL（如加索引）只有一次 `row_log_apply`，无 rename。

### 索引在线状态机与"僵尸索引"

每个在建/被删索引都带一个显式状态（`dict_index_t::online_status`，枚举定义于 dict0mem.h）：

```
ONLINE_INDEX_COMPLETE        0   正常可用
ONLINE_INDEX_CREATION            建设中：不接受查找，DML 全部转写 row log
ONLINE_INDEX_ABORTED             建失败/放弃：等引用归零后懒删除
ONLINE_INDEX_ABORTED_DROPPED     已从字典与表空间删除：等 dict cache 淘汰
```

**为什么失败不能立即删——僵尸索引的由来**：`commit_inplace_alter_table` 阶段若 **MDL 升级失败**（被长事务堵住超时），索引树本身已经建成，但此刻**其他线程可能正持有该索引树的引用**（`n_ref_count > 0`），物理删除会崩。只能打 `ABORTED` 标记——DML 看到这个状态时对它的写入直接走"已缓冲"空操作——等引用归零后由后台懒删除。这个"建好了却上不了线、又删不掉"的中间物，就是源码注释里的 "zombie" 索引。

**状态翻转的两条出口就在 apply 结束时**（`row_log_apply` 收尾）：

```cpp
if (error != DB_SUCCESS) {
  ut_a(!dict_table_is_discarded(index->table));
  /* We set the flag directly instead of
  invoking dict_set_corrupted() here,
  because the index is not "public" yet. */
  index->type |= DICT_CORRUPT;
  index->table->drop_aborted = true;

  dict_index_set_online_status(index, ONLINE_INDEX_ABORTED);   // ★ 打 ABORTED 标记
} else {
  ut_ad(dup.m_n_dup == 0);                                      // 转正前不应存在重复键
  dict_index_set_online_status(index, ONLINE_INDEX_COMPLETE);   // ★ 转正：对外可见
}

log = index->online_log;
index->online_log = nullptr;    // ★ 摘掉 log —— 此后 DML 不再写 row log
rw_lock_x_unlock(dict_index_get_lock(index));
row_log_free(log);
```

★ `index->online_log = nullptr` 这一行是"索引发布"的**物理含义**：log 一旦摘掉，后续 DML 就不再被记入 row log，索引也就正式成为"当前可见结构"。成功路径走 `COMPLETE`，失败路径走 `ABORTED`（并置 `DICT_CORRUPT` + `drop_aborted` 等后台清理）——源码注释特意说明此时索引用 `type |= DICT_CORRUPT` 而非 `dict_set_corrupted()`，因为**它还不是 public 的**。

**崩溃安全靠命名约定**：新索引在建成转正前，名字带 `TEMP_INDEX_PREFIX`（`\377`）前缀——崩溃恢复时凡见此前缀即知是不完整索引，统一清理（`dict0load.cc` 加载字典时按前缀识别）。状态机 + 命名前缀双保险，让"建索引中途崩溃"无需任何额外恢复日志。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_online_alter_log_max_size` | 134217728（128MB） | Global | row log 总上限；超限 DML 不受影响，DDL 报 `ER_INNODB_ONLINE_LOG_TOO_BIG` 失败 |
| `innodb_tmpdir` | 空（用 `--tmpdir`） | Session | row log / merge 临时文件的目录；大表 online 前可指到独立盘 |

### 观测对象 → 手段 速查

⚠️ **勘误**：**不存在任何 DDL / row log / online-index 相关的 InnoDB 状态变量**。`ha_innodb.cc` 的 `innodb_status_variables[]`（:1126 起）里正则扫 `ddl|index|alter|online|sort|merge` **唯一命中是 `truncated_status_writes`**（与 DDL 无关）。`num_index_pages_written` / `secondary_index_triggered_*` 全仓 0 命中（8.0 已移除）。

**→ DDL 进度只能靠 PFS stage event。**

**server 侧 3 个 stage**（`sql/mysqld.cc:12180`）：

```cpp
PSI_stage_info stage_alter_inplace_prepare= { 0, "preparing for alter table", 0, PSI_DOCUMENT_ME};
PSI_stage_info stage_alter_inplace= { 0, "altering table", 0, PSI_DOCUMENT_ME};
PSI_stage_info stage_alter_inplace_commit= { 0, "committing alter table to storage engine", 0, PSI_DOCUMENT_ME};
PSI_stage_info stage_copy_to_tmp_table= { 0, "copy to tmp table", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
```

**InnoDB 侧 8 个 stage**（`srv0srv.cc:811`）——**全部带 `PSI_FLAG_STAGE_PROGRESS`，所以能看百分比**：

```cpp
PSI_stage_info srv_stage_alter_table_end = {
    0, "alter table (end)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_flush = {
    0, "alter table (flush)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_insert = {
    0, "alter table (insert)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_log_index = {
    0, "alter table (log apply index)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_log_table = {
    0, "alter table (log apply table)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_merge_sort = {
    0, "alter table (merge sort)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_table_read_pk_internal_sort = {
    0, "alter table (read PK and internal sort)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
PSI_stage_info srv_stage_alter_tablespace_encryption = {
    0, "alter tablespace (encryption)", PSI_FLAG_STAGE_PROGRESS, PSI_DOCUMENT_ME};
```

**完整阶段序列**（`ut0stage.h:303`）：

```
stage/innodb/alter table (read PK and internal sort)
  → alter table (merge sort)
  → alter table (insert)
  → alter table (log apply index)   ← add-index 形态
   |  alter table (log apply table) ← rebuild 形态
  → alter table (flush)
  → alter table (end)
```

```cpp
inline void Alter_stage::begin_phase_log_index() {
  change_phase(&srv_stage_alter_table_log_index);
}
inline void Alter_stage::begin_phase_log_table() {
  change_phase(&srv_stage_alter_table_log_table);
}
inline void Alter_stage::begin_phase_end() {
  change_phase(&srv_stage_alter_table_end);
}
```

> ★★ **实战用法**：`SELECT ... FROM performance_schema.events_stages_current WHERE event_name LIKE 'stage/innodb/alter%'`，看 `WORK_COMPLETED / WORK_ESTIMATED` 得到百分比。**"log apply table" 阶段卡住 = 第二次 row log apply 在消化增量**，通常很快；"read PK and internal sort" 卡住 = 主扫描在跑。

**`srv_print_ddl_logs`**（唯一的 DDL log 观测开关）：

```cpp
// storage/innobase/handler/ha_innodb.cc:23096
static MYSQL_SYSVAR_BOOL(
    print_ddl_logs, srv_print_ddl_logs, PLUGIN_VAR_OPCMDARG,
    "Print all DDl logs to MySQL error log (off by default)", nullptr, nullptr,
    false);
```
> 16 个读点全在 `storage/innobase/log/log0ddl.cc`（954/1058/1149/1251/1306/1375/1435/1467/1579/1636/1743/1817/1859/1877/1924/1935）。

**DDL 相关的 8 个系统变量**：

| sysvar | C 变量 | 级别 | 默认 | 范围 |
|---|---|---|---|---|
| `old_alter_table` | `thd->variables.old_alter_table` | SESSION | false | bool |
| `innodb_ddl_threads` | THDVAR | **SESSION** | 4 | 1–64 |
| `innodb_ddl_buffer_size` | THDVAR | **SESSION** | 1 MiB | 64 KiB–4 GiB-1 |
| `innodb_parallel_read_threads` | THDVAR | **SESSION** | 4 | 1–`Parallel_reader::MAX_THREADS` |
| `innodb_online_alter_log_max_size` | **`srv_online_max_size`** | GLOBAL(dynamic) | 128 MiB | 64 KiB – ~0 |
| `innodb_sort_buffer_size` | `srv_sort_buf_size` | GLOBAL(**READONLY**) | 1 MiB | 64 KiB–64 MiB |
| `innodb_ft_sort_pll_degree` | **`ddl::fts_parser_threads`** | GLOBAL(**READONLY**) | 2 | 1–16 |
| `innodb_ft_num_word_optimize` | `fts_num_word_optimize` | GLOBAL | 2000 | 1000–10000 |

```cpp
// ha_innodb.cc:1107 —— 三个 SESSION 级 THDVAR（⚠️ 不是 GLOBAL）
static MYSQL_THDVAR_ULONG(parallel_read_threads, PLUGIN_VAR_RQCMDARG,
                          "Number of threads to do parallel read.", nullptr,
                          nullptr, 4, 1, Parallel_reader::MAX_THREADS, 0);
static MYSQL_THDVAR_ULONG(ddl_buffer_size, PLUGIN_VAR_RQCMDARG,
                          "Maximum size of memory to use (in bytes) for DDL.",
                          nullptr, nullptr, 1048576, 65536, 4294967295, 0);
static MYSQL_THDVAR_ULONG(ddl_threads, PLUGIN_VAR_RQCMDARG,
                          "Maximum number of threads to use for  DDL.", nullptr,
                          nullptr, 4, 1, 64, 0);
```

```cpp
// ha_innodb.cc:22530
static MYSQL_SYSVAR_ULONG(ft_sort_pll_degree, ddl::fts_parser_threads,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "InnoDB Fulltext search parallel sort degree, will "
                          "round up to nearest power of 2 number",
                          nullptr, nullptr, 2, 1, 16, 0);
static MYSQL_SYSVAR_ULONG(sort_buffer_size, srv_sort_buf_size,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "Memory buffer size for index creation", nullptr,
                          nullptr, 1048576, 65536, 64 << 20, 0);
static MYSQL_SYSVAR_ULONGLONG(
    online_alter_log_max_size, srv_online_max_size, PLUGIN_VAR_RQCMDARG,
    "Maximum modification log file size for online index creation", nullptr,
    nullptr, 128 << 20, 65536, ~0ULL, 0);      // ★ 默认 128MB
```

**★ `innodb_online_alter_log_max_size` 的三处判定**（`row0log.cc`）——它限制的是**落盘 block 的偏移**：

```cpp
// ① index 版写入时（row0log.cc:361）
    const os_offset_t byte_offset = (os_offset_t)log->tail.blocks * srv_sort_buf_size;
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) {
      goto write_failed;                       // ★ 下一块会越界 → 写入失败
    }

// ② table 版写入时（row0log.cc:459）—— 同式

// ③ apply 阶段二次判定（row0log.cc:3766）—— 转成 DB_ONLINE_LOG_TOO_BIG
    case DB_INDEX_CORRUPT:
      if (((os_offset_t)index->online_log->tail.blocks + 1) *
              srv_sort_buf_size >= srv_online_max_size) {
        /* The log file grew too big. */
        error = DB_ONLINE_LOG_TOO_BIG;
      }
```

```cpp
// handler0alter.cc:7004 —— 最终报错
      case DB_ONLINE_LOG_TOO_BIG:
        my_error(ER_INNODB_ONLINE_LOG_TOO_BIG, MYF(0),
                 get_error_key_name(err_key, ha_alter_info, rebuilt_table));
```

> ★★ **反直觉**：row log 临时文件按 **`srv_sort_buf_size`**（`innodb_sort_buffer_size`，默认 1 MiB）为块单位增长。**调大 `innodb_sort_buffer_size` 会同时改变 row log 的块粒度与有效上限判定粒度**。所以"row log 太大"不一定是 DML 太多，也可能是 sort buffer 太大导致块粒度变粗。
>
> ⚠️ **C 变量名是 `srv_online_max_size`**，不是 `innobase_online_alter_log_max_size`（全仓 0 命中）。

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 正在在线建的索引数 | `innodb_metrics` | `ddl_online_create_index`（计数器名见 srv0mon.cc）；**ddl 子系统默认 disabled**，需 `SET GLOBAL innodb_monitor_enable='ddl'` |
| 后台延迟删表/索引 | `innodb_metrics` | `ddl_background_drop_tables`、`ddl_pending_alter_table`；僵尸索引的清理进度看这里 |
| row log 临时文件 | PFS | `performance_schema.events_statements` / 文件事件里名为 "Innodb Merge Temp File" 的可读名（磁盘真实名是 `ib` + 6 位随机） |
| DDL 进度 | PFS | `events_stages_current` 的 `WORK_COMPLETED / WORK_ESTIMATED`；**为什么不准**见下方源码 |

**进度为什么不许准**（源码给的答案）：`WORK_ESTIMATED` 来自 `row_log_estimate_work`，它只按"当前还剩几个块"估算：

```cpp
ulint row_log_estimate_work(const dict_index_t *index) {
  if (index == nullptr || index->online_log == nullptr) {
    return (0);                       // 没有 row log（非 online）就没有进度可报
  }
  const row_log_t *l = index->online_log;
  const ulint bytes_left = static_cast<ulint>(l->tail.total - l->head.total);
  const ulint blocks_left = bytes_left / srv_sort_buf_size;

  return (blocks_left * row_log_progress_inc_per_block());
}
```

★ 它算的是**当前积压量**（`tail - head`），而 online DDL 执行期间 DML 还在不停推进 `tail`——"剩余工作量"是个**移动靶**。这就是进度条忽快忽慢甚至倒退的根源（与 [`ddl.md`](ddl.md) 观测章的"进度不准"一致，官方注释亦自认是 wild guess）。
| apply 阻塞点 | DEBUG_SYNC | `row_log_table_apply1_before` / `row_log_table_apply2_before` / `row_log_apply_before` 三个同步点，是复现"第二次 apply 阻塞"的标准手段 |
| DDL 超限失败 | error log | `ER_INNODB_ONLINE_LOG_TOO_BIG`——先查 `innodb_online_alter_log_max_size` 与写负载，而不是盲目重试 |

---

## Misc

### 面向二次开发

**扩展点**：给引擎新增一种"可 online"的 DDL 操作类型，要动三处——`check_if_supported_inplace_alter` 加接受分支（决定锁级别）、`prepare_inplace_alter_table` 加 row log 分配分支、`inplace_alter_table` / `commit_inplace_alter_table` 加对应构建与 apply 逻辑。锁级别选错（如对需要追平的操作返回 `NO_LOCK`）会直接丢数据，六档返回值的语义约束见 [`ddl.md`](ddl.md)。

**坑与已知缺陷**：

| 坑 | 说明 |
|---|---|
| 僵尸索引占用空间 | MDL 升级失败留下的 `ABORTED` 索引要等引用归零才删；`innodb_metrics` 的 `ddl_background_drop_*` 长期非零说明有积压 |
| 新唯一索引的重复键报错不指名索引 | WL#5526 记录的历史 anomaly 且沿用至今：apply 完成后、MDL 提交前，DML 撞新索引的重复键时该索引对 server 尚不可见，错误无法报出索引名 |
| 长事务是头号敌人 | 执行期它让 row log 积压（超限即失败），收尾期它让 MDL 升级等待（并制造僵尸索引）——观测与治理都围绕它 |

### 易混淆概念

| 概念 | 辨析 |
|---|---|
| INPLACE ≠ online | INPLACE 是算法（不在 server 层建临时表）；online 是锁行为（并发 DML 能进）。六档返回值里带 `*_AFTER_PREPARE` 的仍会阻塞写。见 [`ddl.md`](ddl.md)「Online 到底指什么」 |
| row log vs DDL log | 前者管并发正确性（临时文件、非持久），后者管原子性（系统表 + redo）。见上方对照表 |
| `online_log`（内存结构）vs online log 磁盘文件 | `dict_index_t::online_log` 是 `row_log_t` 结构头；真正的记录在它指向的临时文件里 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.12 InnoDB and Online DDL*（LOCK 子句语义、空间需求、失败条件）

**WorkLog**
- [WL#5526](https://dev.mysql.com/worklog/task/?id=5526) InnoDB: Online ADD INDEX——**online DDL 的奠基设计**：row log 七步协议、`online_index_status` 状态机、僵尸索引、`innodb_online_alter_log_max_size` 与四件套接口均出自此 WL；8.0 沿用其并发协议

**相关文档**
- 上层：server 侧 DDL 编排、算法选择策略、MDL 三段式、原子 DDL 见 [`ddl.md`](ddl.md)
- 索引/表具体如何被构建（Loader/Builder/Btree_load、并行与终止检测）见 [`ddl_engine.md`](ddl_engine.md)
- 另两条路：COPY 见 [`copy_ddl.md`](copy_ddl.md)、INSTANT 见 [`instant_ddl.md`](instant_ddl.md)
