# COPY DDL 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，剖析 `ALGORITHM=COPY` 这一路：server 层如何编排（`mysql_alter_table` 内联的拷贝分支）、`copy_data_between_tables` 如何逐行搬运、以及**它为什么一定不 Online**（拷贝阶段 `MDL_SHARED_NO_WRITE`，rename 前才升 `MDL_EXCLUSIVE`）。
>
> ⚠️ **两处常见误传，源码纠正见正文**：① 不是"全程 MDL X"，而是 **SNW → X 两级**（`sql_table.cc:17578` / `sql_base.cc:2472`）；② **不走 `write_record()`**，而是直接 `to->file->ha_write_row()`（`sql_table.cc:18689`），因此不触发 trigger、不做 DUP 处理。
>
> **边界**：本篇**写完两层**——server 侧的编排与锁，以及"逐行走 SQL 层 INSERT"落到引擎后走的完整写入路径（`ha_write_row` 一路）。算法选择策略与 MDL 三段式的公共骨架见 [`ddl.md`](ddl.md)；INPLACE / INSTANT 两条路见 [`inplace_ddl.md`](inplace_ddl.md)、[`instant_ddl.md`](instant_ddl.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [`copy_data_between_tables` 逐段剖析](#copy_data_between_tables-逐段剖析)
  - [设计权衡：COPY 慢在哪（代码级答案）](#设计权衡copy-慢在哪代码级答案)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`ALGORITHM=COPY` 是 ALTER TABLE 的**整表复制**执行策略：server 层按新定义建临时表 `#sql-xxx`，把源表数据逐行搬过去，最后 rename 原子换名。它是唯一**完全在 server 层完成**的路径——引擎自始至终只做"往一张新表里 INSERT"，不感知 DDL 的存在。

### 用途

两条角色：

1. **兜底路径**：InnoDB 无法 INPLACE 的变更（改列类型、删主键、改表字符集、修改 `NULL ⇄ NOT NULL` 等）只能走 COPY。server 的算法判定（见 [`ddl.md`](ddl.md)）在引擎返回 `HA_ALTER_INPLACE_NOT_SUPPORTED` 时落到这里。
2. **正确性参照系**：INPLACE / INSTANT 的语义正确性，定义上就是"与 COPY 等价"。读 COPY 源码能看清 ALTER 语义的完整清单——截断告警、自增处理、生成列求值、CHECK 约束、重复键报错——这些在 INPLACE 里被拆散到引擎内部，在 COPY 里全摊在明面上。

### COPY 的适用边界：强制走 COPY 的场景

因为**有一类变更在架构上就做不到 INPLACE**。server 层 `is_inplace_alter_impossible()` 给出了强制走 COPY 的完整清单（命中任一即退 COPY，不再问引擎）：

| 场景 | 源码注释给出的理由 |
|---|---|
| **临时表** | *"At the moment we can't handle altering temporary tables without a copy"* |
| **`ALTER TABLE ... ORDER BY`** | 注释列了两条：a) 没有引擎实现它；b) **API 缺乏向引擎传递"表排序"信息的基础设施** |
| **`ENABLE/DISABLE KEYS`** | MyISAM/Heap 专有操作，与其他操作组合时不支持 in-place（单独做时走 `simple_rename_or_index_change`） |
| **CHECK 约束需要（重新）求值** | *"Check constraints are evaluated in the server"* —— 求值在 server，引擎无从执行 |
| **改存储引擎**（显式 `ENGINE=` 或隐式，如非分区表变分区表） | 必须重建为另一引擎的表，跨引擎无法 in-place |
| **表由 5.0 之前的 MySQL 创建** | BUG#6236：早期版本 null 字段数计算错误，fast alter 会让 frm 与数据不同步，无法判断表由哪个版本创建 ⇒ 一律退 COPY |
| **改 GEOMETRY 列的 SRID**（且不是改成 NULL SRID） | 需要校验；改成 NULL SRID 只是元数据变更，仍可 in-place |

把这些归个类，COPY 不可被替代的原因其实是三条：

1. **语义在 server 层**：CHECK 约束、存储生成列的求值逻辑都在 server，引擎的 INPLACE 路径根本不经过 SQL 层，也就无从执行——只能靠"server 逐行 INSERT"来跑这些语义。
2. **API 基础设施缺失**：`ORDER BY` 想 in-place 就得让引擎知道"按什么顺序组织记录"，当前 handler API 没有这个通道，所以只能 COPY。
3. **跨引擎 / 历史包袱 / 兜底**：换引擎、5.0 前的老表、以及引擎自己回答 `HA_ALTER_INPLACE_NOT_SUPPORTED` 的所有情况——COPY 是**唯一不依赖引擎能力的通用实现**。这也解释了为什么 MyISAM、MEMORY 等没有实现 INPLACE 四件套的引擎，它们的所有 ALTER 都是 COPY。

再加两个**人为**入口：`ALGORITHM=COPY` 显式指定，以及 `old_alter_table=ON`（用于规避 INPLACE 的 bug、或做性能对比、或刻意借机整理碎片——COPY 会重写整表，物理上也更紧凑）。

⇒ 所以 COPY 的定位不是"落后"，而是 **"唯一与引擎无关的通用实现 + 语义正确性的定义基准"**。只要还有引擎不实现 INPLACE、还有语义必须在 server 层求值，它就删不掉。

**★★★ 上面那张表的源码出处**（`sql/sql_table.cc:12794`，★ 六个 `return true` 就是完整清单）：

```cpp
/*
  Check if the pending ALTER TABLE operations support the in-place
  algorithm based on restrictions in the SQL layer or given the
  nature of the operations themselves. If in-place isn't supported,
  it won't be necessary to check with the storage engine.

  @return false       In-place is possible, check with storage engine.
  @return true        Incompatible operations, must use table copy.
*/
static bool is_inplace_alter_impossible(TABLE *table,
                                        HA_CREATE_INFO *create_info,
                                        const Alter_info *alter_info) {
  /* At the moment we can't handle altering temporary tables without a copy. */
  if (table->s->tmp_table) return true;                       /* ① 临时表 */

  /*
    For the ALTER TABLE tbl_name ORDER BY ... we always use copy
    algorithm. In theory, this operation can be done in-place by some
    engine, but since a) no current engine does this and b) our current
    API lacks infrastructure for passing information about table ordering
    to storage engine we simply always do copy now.

    ENABLE/DISABLE KEYS is a MyISAM/Heap specific operation ...
  */
  if (alter_info->flags &
      (Alter_info::ALTER_ORDER | Alter_info::ALTER_KEYS_ONOFF))
    return true;                                              /* ② ORDER BY / KEYS ON|OFF */

  /* Check constraints are evaluated in the server ... */
  if (is_any_check_constraints_evaluation_required(alter_info)) return true;  /* ③ */

  /* If the table engine is changed explicitly (using ENGINE clause)
     or implicitly ... a regular alter table (copy) needs to be performed. */
  if (create_info->db_type != table->s->db_type()) return true;              /* ④ 换引擎 */

  /* There was a bug prior to mysql-4.0.25. Number of null fields was
     calculated incorrectly. ... See BUG#6236. */
  if (!table->s->mysql_version) return true;                                 /* ⑤ 5.0 前的老表 */

  /* If we are changing the SRID modifier of a column, we must do a COPY.
     But not if we are changing to the NULL SRID. ... */
  for (const Create_field &new_field_def : alter_info->create_list) {
    if (new_field_def.field != nullptr &&
        new_field_def.field->type() == MYSQL_TYPE_GEOMETRY) {
      const Field_geom *field_geom =
          down_cast<const Field_geom *>(new_field_def.field);
      if (field_geom->get_srid() != new_field_def.m_srid &&
          new_field_def.m_srid.has_value())
        return true;                                                         /* ⑥ 改 SRID */
    }
  }
  return false;
}
```

> ⚠️ **一个常见误传**：**"改字符集"不在这个函数里**。它由默认 handler 实现判定（`sql/handler.cc:4990`）：
> ```cpp
>   if (create_info->used_fields &
>           (HA_CREATE_USED_CHARSET | HA_CREATE_USED_DEFAULT_CHARSET |
>            HA_CREATE_USED_PACK_KEYS | HA_CREATE_USED_MAX_ROWS) ||
>       (table->s->row_type != create_info->row_type))
>     return HA_ALTER_INPLACE_NOT_SUPPORTED;
> ```
> ★ 所以真相是：**"改字符集×PACK_KEYS×MAX_ROWS×ROW_FORMAT"走的是"引擎说不支持"这条路**，不是 SQL 层直接判 COPY。

**★★ 第二道闸门：引擎说 NOT_SUPPORTED 才降级**（`sql_table.cc:16977` → `:17513`）：

```cpp
  /*
    Use copy algorithm if:
    - old_alter_table system variable is set without in-place requested using
      the ALGORITHM clause.
    - Or if in-place is impossible for given operation.
    - Changes to partitioning needs to be handled using table copying
      algorithm unless the engine supports partitioning changes using
      in-place API ...
  */
  if ((thd->variables.old_alter_table && ...) ||
      is_inplace_alter_impossible(table, create_info, alter_info) ||
      (partition_changed && !(table->s->db_type()->partition_flags() &
                              HA_USE_AUTO_PARTITION) && !new_part_info)) {
    if (alter_info->requested_algorithm ==
        Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
      my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), "ALGORITHM=INPLACE",
               "ALGORITHM=COPY");
      return true;          /* ★★ 显式 INPLACE 时不降级，直接报错 */
    }
    if (alter_info->requested_algorithm ==
        Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) { ... "ALGORITHM=INSTANT" ... }
    alter_info->requested_algorithm = Alter_info::ALTER_TABLE_ALGORITHM_COPY;
  }
```
```cpp
/* sql/sql_table.cc:17513 —— 引擎侧降级 */
      case HA_ALTER_INPLACE_NOT_SUPPORTED:
        if (alter_info->requested_algorithm ==
            Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
          ha_alter_info.report_unsupported_error("ALGORITHM=INPLACE",
                                                 "ALGORITHM=COPY");
          goto err_new_table_cleanup;
        }
        // COPY with LOCK=NONE is not supported, no point in trying.
        if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_NONE) {
          ha_alter_info.report_unsupported_error("LOCK=NONE", "LOCK=SHARED");
          goto err_new_table_cleanup;
        }
        use_inplace = false;      /* ★★ 落到 COPY */
        break;
      case HA_ALTER_ERROR:
      default:
        goto err_new_table_cleanup;   /* ★ 硬错误，不降级 */
```
> ★★ **三点纠正**：
> ① 显式 `ALGORITHM=INPLACE` **不会静默降级 COPY，是直接报错** `ER_ALTER_OPERATION_NOT_SUPPORTED`。
> ② `HA_ALTER_ERROR` 是硬错误，**也不降级**——只有 `NOT_SUPPORTED` 才降级。
> ③ 不指定 ALGORITHM 时的完整链路是：`is_inplace_alter_impossible()`（SQL 层）→ `check_if_supported_inplace_alter()`（问引擎）→ `NOT_SUPPORTED` → COPY。

**★ 报错文本**（`share/messages_to_clients.txt:6984`，★ 引擎可附带"原因"）：

```
ER_ALTER_OPERATION_NOT_SUPPORTED         "%s is not supported for this operation. Try %s."
ER_ALTER_OPERATION_NOT_SUPPORTED_REASON  "%s is not supported. Reason: %s. Try %s."
```
```cpp
/* sql/handler.cc:5013 —— 引擎设了 unsupported_reason 就用 REASON 版本 */
void Alter_inplace_info::report_unsupported_error(const char *not_supported,
                                                  const char *try_instead) {
  if (unsupported_reason == nullptr)
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), not_supported, try_instead);
  else
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON, MYF(0), not_supported,
             unsupported_reason, try_instead);
}
```
> ★ InnoDB 常用的 reason：`Cannot change column type INPLACE`、`Dropping a primary key is not allowed without also adding a new primary key`、`Adding foreign keys needs foreign_key_checks=OFF`。

**★ 几个"以为是 COPY 其实不是"的反例**（★ 8.0 已能 INPLACE/INSTANT）：

| 操作 | 8.0.39 实际 |
|---|---|
| **加/删主键** | InnoDB `ADD_PK_INDEX`/`DROP_PK_INDEX` 在 `INNOBASE_ALTER_REBUILD` → **INPLACE 重建**。只有"删了不建"才降级 COPY |
| **`AFTER`/`FIRST` 列重排**（非分区表） | 置 `ALTER_STORED_COLUMN_ORDER`，在 `INNOBASE_ALTER_REBUILD` 里 → **可 INPLACE** |
| `ALTER TABLE ENGINE=<同引擎>`（即 FORCE） | 置 `ALTER_RECREATE` → `RECREATE_TABLE` → `INNOBASE_ALTER_REBUILD` → **INPLACE 重建，不是 COPY** |
| **纯 RENAME** | 走 `is_simple_rename_or_index_change()` 旁路 |

```cpp
/* sql/sql_table.cc:15769 —— ⚠️ 显式 ALGORITHM=COPY 会让"纯 RENAME"也走完整 COPY */
static bool is_simple_rename_or_index_change(const Alter_info *alter_info) {
  return (!(alter_info->flags &
            ~(Alter_info::ALTER_RENAME | Alter_info::ALTER_KEYS_ONOFF)) &&
          alter_info->requested_algorithm !=
              Alter_info::ALTER_TABLE_ALGORITHM_COPY);   /* ★ 最后这个条件是陷阱 */
}
```

### COPY 与引擎层的边界：仅使用通用表操作原语

有个容易说错的地方：COPY **不是不涉及引擎层**——它建表、写数据、改名的每一步都要落到引擎。区别在于**用的是哪一套 API**：

| | COPY | INPLACE |
|---|---|---|
| 用的 API | **通用表操作原语**：`ha_create_table` / `ha_write_row` / `ha_rename_table` / `ha_drop_table` | **DDL 专用四件套**：`prepare_inplace_alter_table` / `inplace_alter_table` / `commit_inplace_alter_table`（+ `check_if_supported_inplace_alter`） |
| 引擎是否知道"这是 DDL" | ❌ **不知道**。它看到的就是"建一张表 + 一堆 INSERT + rename + drop" | ✅ 知道，全程持有 `ha_alter_info` 上下文 |
| 编排者 | server（`mysql_alter_table`）全程驱动 | 引擎负责执行，server 只做 MDL 升降级与调用 |

⇒ 所以更准确的说法是：**COPY 是 server 用引擎的"通用积木"拼出来的 DDL 服务**，而不是"不碰引擎"。

**★ "通用原语"的概念性定义**（`sql/handler.h:3199`，★ 注释明说了什么操作"没有 flag"）：

```cpp
class Alter_inplace_info {
 public:
  /**
     Bits to show in detail what operations the storage engine is
     to execute.

     All these operations are supported as in-place operations by the
     SQL layer. This means that operations that by their nature must
     be performed by copying the table to a temporary table, will not
     have their own flags here (e.g. ALTER TABLE FORCE, ALTER TABLE
     ENGINE).
     ...
  */
  typedef ulonglong HA_ALTER_FLAGS;

  // Add non-unique, non-primary index
  static const HA_ALTER_FLAGS ADD_INDEX = 1ULL << 0;
  ...
  // Add primary index
  static const HA_ALTER_FLAGS ADD_PK_INDEX = 1ULL << 4;
  // Drop primary index
  static const HA_ALTER_FLAGS DROP_PK_INDEX = 1ULL << 5;
  ...
```
> ★★ 这句注释就是"COPY 与引擎层边界"的权威定义：**本质需要拷表的操作，在 INPLACE 能力位图里根本没有自己的 flag**。所以 `handler_flags` 位图是"INPLACE 能表达什么"的边界，超出边界的即 COPY。
> ⚠️ 8.0.39 已用到 bit 49（`ALTER_COLUMN_VISIBILITY`），`ulonglong` 只剩 50–63 可用。

**★ InnoDB 的四个能力集合**（`storage/innobase/handler/handler0alter.cc:114`，★ 判定"哪些能 INPLACE"）：

```cpp
/** Operations for creating secondary indexes (no rebuild needed) */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ONLINE_CREATE =
    Alter_inplace_info::ADD_INDEX | Alter_inplace_info::ADD_UNIQUE_INDEX |
    Alter_inplace_info::ADD_SPATIAL_INDEX;

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
    /* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
    | Alter_inplace_info::RECREATE_TABLE;

/** Operations that InnoDB does not care about */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INPLACE_IGNORE =
    Alter_inplace_info::ALTER_COLUMN_DEFAULT | ... | Alter_inplace_info::ALTER_RENAME | ...;

/** Operation allowed with ALGORITHM=INSTANT */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME | Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN | Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN | Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN;
```
```cpp
/* handler0alter.cc:998 —— ★ 一行判定：不在三集合里的 → COPY */
  if (ha_alter_info->handler_flags &
      ~(INNOBASE_INPLACE_IGNORE | INNOBASE_ALTER_NOREBUILD |
        INNOBASE_ALTER_REBUILD)) {
    if (ha_alter_info->handler_flags &
        Alter_inplace_info::ALTER_STORED_COLUMN_TYPE) {
      /* INSTANT 与 INPLACE 的原因文本还不一样 */
      ha_alter_info->unsupported_reason = innobase_get_err_msg(
          ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_COLUMN_TYPE);
    }
    return HA_ALTER_INPLACE_NOT_SUPPORTED;
  }
```
> ★★ 结论：**"让某个操作强制走 COPY"的最小改动 = 在 `handler.h` 加一个不在上述三个集合里的 flag + 在 `fill_alter_inplace_info()` 里置位，引擎侧零改动**。
> ★ InnoDB 对这个接口**只可能返回两种"支持"值**（`:1353`）：`HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE` 或 `SHARED_LOCK_AFTER_PREPARE`，永不返回 `EXCLUSIVE_LOCK` / `SHARED_LOCK`。

**★ COPY 实际用到的 handler 调用点全清单**（★ 全是通用原语）：

| 步骤 | 位置 | 调用 |
|---|---|---|
| 建中间表 | `sql_table.cc:17688` | `ha_create_table(...)` |
| 打开 | `:17719` | `open_table_uncached(...)` / `open_temporary_table` |
| 加锁 | `:17730` | `lock_tables(...)` |
| **拷贝** | `:17749` | `copy_data_between_tables(...)` |
| 逐行写 | `:18689` | **`to->file->ha_write_row(to->record[0])`** |
| 收尾 | `:18720` | `ha_end_bulk_insert()` |

```cpp
/* sql/sql_table.cc:17738 —— HA_NO_COPY_ON_ALTER 的引擎跳过拷贝 */
  if (!(new_table->file->ha_table_flags() & HA_NO_COPY_ON_ALTER)) {
    new_table->next_number_field = new_table->found_next_number_field;
    THD_STAGE_INFO(thd, stage_copy_to_tmp_table);
    if (copy_data_between_tables(thd, thd->m_stage_progress_psi, table,
                                 new_table, alter_info->create_list, &copied,
                                 &deleted, alter_info->keys_onoff, &alter_ctx))
      goto err_new_table_cleanup;
  } else {
    /* Should be MERGE only */
    assert(new_table->file->ht->db_type == DB_TYPE_MRG_MYISAM);
  }
```

**★★★ 纠正一处常见错误：COPY 不走 `write_record()`**

```cpp
/* sql/sql_table.cc:18689 —— 直接 ha_write_row */
    error = to->file->ha_write_row(to->record[0]);
```
> `sql/sql_table.cc` 全文**没有任何** `write_record` 调用（`write_record` 在 `sql_insert.cc:1793`）。所以 COPY 的逐行写入**绕开了**：trigger、`COPY_INFO` 计数、`DUP_REPLACE` / `ON DUPLICATE KEY UPDATE` 处理。
> 但它**绕不开** `ha_write_row` 内部的引擎完整插入路径（B-tree 下降、redo、undo），也绕不开 `handler.cc:8015` 里的 `binlog_log_row` 钩子。

**这个区别带来三个直接后果**，也解释了 COPY 为什么必然更慢、必然不 online：

1. **没有 row log ⇒ 不可能 online**：row log 是四件套里的 DDL 特化机制（`prepare` 阶段分配）。COPY 压根不走这套 API，引擎无从记录并发 DML，所以**拷贝阶段全程持有 `MDL_SHARED_NO_WRITE`（SNW，可读不可写）**，rename 前才升 X（★ 详见「设计权衡」节，本篇开头"全程 MDL X"的说法需按此修正）。
2. **逐行 INSERT ⇒ 索引随机写**：引擎视角是普通 INSERT，每插一行就要同步维护**全部**二级索引；而 INPLACE 是 `Btree_load` 排序后批量建树（顺序写）。写放大完全不在一个量级。
3. **undo/redo 按普通 DML 产生**：每行 INSERT 都写正常的事务日志；INPLACE 构建期则可以做 DDL 特化（批量建树、日志策略不同）。

**一个例外值得记住**：COPY 仍然依赖引擎的一项 DDL 级特性——**原子 DDL 能力**。server 用 `atomic_replace = 新旧引擎都支持 HTON_SUPPORTS_ATOMIC_DDL` 来决定整表替换是否走"单事务原子提交"（见下节）。所以它不是完全不碰引擎的 DDL 特性，只是**不碰那套执行 API**。

#### ★ COPY DDL 的原子性

**先看清问题**：COPY 是最"多步"的一条路径——建中间表 → 逐行灌数据 → 旧表改名备份 → 中间表改名转正 → 删旧表。任何一步之间崩溃，都可能留下"新旧两份数据"或"半个表"。8.0 之前这就是常态（所以才有 `#sql-xxx` 残骸这种运维话题）。

**8.0 的原子性由三层叠出来**，缺一不可：

| 层 | 机制 | 管什么 | 谁实现 |
|---|---|---|---|
| **DD 事务** | `thd->dd_client()` 的 `store/update/drop` 攒到一个事务，随 `trans_commit_stmt` 提交 | 数据字典的表定义（新定义写入、旧定义删除、FK/触发器/CC 名调整） | server（`sql/dd`） |
| **引擎 DDL log** | 每个引擎操作**前**写一条"回滚预案"到 `mysql.innodb_ddl_log`；提交成功即删除，崩溃后 recovery 重放 | 引擎内物理对象：`.ibd` 文件、B-tree、dict cache 项 | InnoDB（`log0ddl.cc`） |
| **binlog** | `write_bin_log()` 进**事务 cache**，与 DD 事务一起走 2PC（prepare → commit） | 复制一致性：DDL 要么在 binlog 里、要么不在 | server（`binlog`） |

★ 关键点是**第二层**：server 并不直接写 DDL log——它是**引擎在自己的操作里顺带写的**，对 server 透明。COPY 的每一步对应什么预案：

| COPY 的步骤 | 引擎内部写的回滚预案 |
|---|---|
| 建中间表 `#sql-xxx` | `write_delete_space_log`（整表空间删除预案）+ 每棵索引 `write_free_tree_log` + `write_remove_cache_log` |
| 旧表改名 → `#sql-2-<pid>-<tid>` | `write_rename_space_log` / `write_rename_table_log`（崩溃后可把名字改回去） |
| 中间表改名 → 正式名 | 同上 |
| 删旧表（备份名） | `write_drop_log`（清 `innodb_table_metadata`）+ `write_delete_space_log`（file-per-table 时删 .ibd） |

⇒ 于是**成功路径**的收尾很干净：server 提交 DD 事务 → binlog 2PC commit → 调引擎 `post_ddl(thd)` → InnoDB 把这几条预案**删掉**（log 表变空）。而**崩溃路径**：事务没提交 ⇒ 预案还在 ⇒ 下次启动 recovery 逐条 replay，把中间表、改过的名字、删掉的文件按预案清理/回退。这正是 [`atomic_ddl.md`](atomic_ddl.md) 讲的"DDL log 是回滚预案而非执行计划"——**方向与 DROP 类相反**：CREATE 写的是"删除"预案。

下面看 server 怎么决定"攒一个事务"还是"分步提交"——`atomic_replace` 在 COPY 收尾出现十多次，归结起来就是这个分岔：

**★★ 非原子路径：先把 DD 落盘 + 中间提交**（`sql/sql_table.cc:17860`）：

```cpp
  /*
    To ensure DDL atomicity after this point support from both old and
    new engines is necessary. If either of them lacks such support let
    us commit transaction so changes to data-dictionary are more closely
    reflect situations in SEs.

    Also if new SE supports atomic DDL then we have not stored new table
    definition in on-disk data-dictionary so far. It is time to do this
    now if ALTER TABLE as a whole won't be atomic.
  */
  if (!atomic_replace) {
    if ((new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
        thd->dd_client()->store(non_dd_table_def.get()))
      goto err_new_table_cleanup;

    Implicit_substatement_state_guard substatement_guard(thd);
    if (trans_commit_stmt(thd) || trans_commit_implicit(thd))
      goto err_new_table_cleanup;

    // Safety, in-memory dd::Table is no longer totally correct.
    non_dd_table_def.reset();
  }
```
> ★★ 反过来读：**原子路径下中间表的 `dd::Table` 一直不 `store()`**，直到最后随整个事务一起提交——这就是"原子"的实现方式。

**★★ binlog：COPY 只写一条 Query event**（`sql/sql_table.cc:18158`，★ 不是逐行）：

```cpp
  DEBUG_SYNC(thd, "alter_table_before_main_binlog");

  ha_binlog_log_query(thd, create_info->db_type, LOGCOM_ALTER_TABLE,
                      thd->query().str, thd->query().length, alter_ctx.db,
                      alter_ctx.table_name);
  ...
  if (write_bin_log(thd, true, thd->query().str, thd->query().length,
                    atomic_replace && !is_noop))     /* ★★ 第 4 参 is_trans */
    goto err_with_mdl;
```
```cpp
/* sql/sql_table.cc:1206 —— write_bin_log 的 is_trans 参数 */
    is_trans                      if the event changes either
                                  a trans or non-trans engine.
...
    error = thd->binlog_query(THD::STMT_QUERY_TYPE, query, query_length,
                              is_trans, false, false, errcode);
```
> ★★ 两个关键结论：
> ① **COPY 的 N 行数据在 binlog 里就是一条 `ALTER TABLE` 语句**（`STMT_QUERY_TYPE`），从库**重放整条 ALTER**而不是重放 N 行——这是 COPY 与"手工 INSERT...SELECT"的本质差别。
> ② `is_trans = atomic_replace && !is_noop`：原子 DDL 时 binlog 走 **trx cache**，与 DD/SE 一起 2PC 提交；非原子时立即写。

**★★ 失败回滚的两套语义**（`sql/sql_table.cc:18260`，★ 按引擎能力分岔）：

```cpp
err_new_table_cleanup:
  if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {
    if (new_table) close_temporary_table(thd, new_table, true, true);
    else if (!no_ha_table) rm_temporary_table(thd, new_db_type, ...);
  } else {
    if (new_table) close_temporary_table(thd, new_table, true, false);

    if (!(new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
      /* 非原子引擎：显式删中间表 */
      if (no_ha_table) {          // Only remove from DD.
        dd::drop_table(thd, alter_ctx.new_db, alter_ctx.tmp_name,
                       *drop_table_def);
        (void)trans_intermediate_ddl_commit(thd, result);
      } else                      // Remove from both DD and SE.
        (void)quick_rm_table(thd, new_db_type, alter_ctx.new_db,
                             alter_ctx.tmp_name, FN_IS_TMP);
    } else {
      /* 原子引擎：一把回退 */
      trans_rollback_stmt(thd);
      /*
        Full rollback in case we have THD::transaction_rollback_request
        and to synchronize DD state in cache and on disk (as statement
        rollback doesn't clear DD cache of modified uncommitted objects).
      */
      trans_rollback(thd);
    }
  }
```
> ★★ 对比很清楚：非原子引擎 → `dd::drop_table` / `quick_rm_table`（**两步，中间仍有窗口**）；原子引擎 → `trans_rollback`（**一次回退，连同中间的 create + insert + DD 全部消失**）。

**★ 一个原子 DDL 解决的老问题**（`sql/sql_table.cc:18343`）：

```cpp
  if (atomic_replace) {
    /*
      If both old and new storage engines support atomic DDL all changes
      were reverted at this point. So we can safely try to reopen table
      under old name.
    */
  } else {
    /*
      If ALTER TABLE ... RENAME ... ALGORITHM=COPY is non-atomic we can't
      be sure that rename step was reverted, so we simply remove table
      from the list of locked tables.
    */
    if (alter_ctx.is_table_renamed())
      thd->locked_tables_list.unlink_all_closed_tables(thd, nullptr, 0);
  }
```
> ★ 这段话直接说明了 8.0 原子 DDL 修的是什么：**非原子的 COPY+RENAME 无法保证 rename 被回退**。

**分岔点在"切换前"**——`rename 旧表→备份名` 之前：

```cpp
  /*
    To ensure DDL atomicity after this point support from both old and
    new engines is necessary. If either of them lacks such support let
    us commit transaction so changes to data-dictionary are more closely
    reflect situations in SEs.

    Also if new SE supports atomic DDL then we have not stored new table
    definition in on-disk data-dictionary so far. It is time to do this
    now if ALTER TABLE as a whole won't be atomic.
  */
  if (!atomic_replace) {
    if ((new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
        thd->dd_client()->store(non_dd_table_def.get()))   // 新引擎支持、旧不支持 → 此刻才落盘新定义
      goto err_new_table_cleanup;

    Implicit_substatement_state_guard substatement_guard(thd);

    if (trans_commit_stmt(thd) || trans_commit_implicit(thd))   // ★ 非原子 → 提前提交
      goto err_new_table_cleanup;

    non_dd_table_def.reset();
  }
```

★ 注释把理由说得很清楚：**"从这里往后，DDL 原子性需要新旧两个引擎都支持"**——只要有一个不支持，就让 DD 事务先提交，使 DD 更接近引擎的真实状态（牺牲原子性换一致性）。而 `atomic_replace` 时，新表定义**一直不落盘**，攒到最后一刻。

**结尾的提交时机与 post_ddl**：

```cpp
  // Commit if it was not done before in order to be able to reopen tables.
  if (atomic_replace && (trans_commit_stmt(thd) || trans_commit_implicit(thd)))
    goto err_with_mdl;                                    // ★ 原子路径：到这里才统一提交

  if ((new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) && new_db_type->post_ddl)
    new_db_type->post_ddl(thd);                           // ★ 调引擎的 post_ddl 钩子
  if ((old_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) && old_db_type->post_ddl)
    old_db_type->post_ddl(thd);                           // ★ 新旧两个引擎都要调
```

⇒ 两种策略的完整对照：

| | `atomic_replace`（新旧引擎都支持原子 DDL） | 非原子（有一个引擎不支持） |
|---|---|---|
| DD 事务 | **一个 DD 事务贯穿到底**，收尾才 `trans_commit_stmt/implicit` | 每个"引擎操作"后**提前提交**，让 DD 跟上引擎状态 |
| 新表定义 `store()` 时机 | rename 中间表→正式名**之前**才 `store()` | 分岔点就 `store()`（如果新引擎支持） |
| rename 的 flag | `NO_DD_COMMIT`（随大事务一起提交） | 无 flag（即时提交） |
| binlog | 进**事务 cache**（与 DD 事务同生共死） | 普通 cache |
| 失败后果 | 全部回滚，无残骸 | **留残骸**：`quick_rm_table` 清中间表 + **重试 20 次**恢复备份名，失败则表停留备份名状态 |

**引擎侧怎么"提供"这个能力**——server 调的 `post_ddl(thd)` 就是引擎注册的钩子（`handlerton::post_ddl`，InnoDB 在 `ha_innobase` handlerton 上设置）。InnoDB 侧的支撑是三件套：

1. 每个引擎操作（建中间表、rename、drop 旧表）之前都**写 DDL log 回滚预案**；
2. 提交成功 ⇒ 预案删除（log 表干净）；崩溃 ⇒ recovery 重放预案，把半成品清掉；
3. `post_ddl` 在 server 收尾时**统一清理** DDL log（正常路径的收尾，区别于崩溃恢复的重放）。

⇒ 所以"COPY 用原子 DDL 能力"的准确含义是：**server 把 DD 事务的提交时机交给引擎能力决定**——引擎说"我能保证"，server 就攒一个大事务；引擎说"不能"，server 就分步提交并用清理代码兜底。这也是 [`atomic_ddl.md`](atomic_ddl.md) 说的"原子性是 server 与引擎协作的结果"在 COPY 路径上的具体形态。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 之前 | COPY 是 ALTER TABLE 的**唯一**执行方式（官方文档定位为 "the old table-copy method"） |
| 5.6 | online DDL 引入（WL#5526 / WL#5534 系列），INPLACE 成为多数操作首选，COPY 退为兜底 |
| 8.0 | **原子 DDL**：目标引擎支持原子 DDL 时，COPY 不再"提前提交拷贝事务"，整段留在单事务内由引擎 DDL log 收尾（见「主链路」①） |
| 8.0.x | INSTANT 覆盖面持续扩大（8.0.12 起 `ADD COLUMN`），COPY 的兜底范围进一步收窄 |

> COPY 本身没有专属 WL（它不是被"设计"出来的，而是被剩下的问题定义的）；`old_alter_table` 开关至今保留，用于强制走 COPY 做性能对比。

**★ 8.0 的两个关键演进，源码证据**：

**① 原子 DDL**（`sql/sql_table.cc:17191`）：

```cpp
  /*
    Atomic replacement of the table is possible only if both old and new
    storage engines support DDL atomicity.
  */
  bool atomic_replace = (new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
                        (old_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
```
> ★ 且它决定 binlog 是否与 DDL 同事务提交（见原子性节的 `write_bin_log` 第 4 参）。

**② INSTANT ⊂ INPLACE**（`sql/sql_table.cc:17495`，★ 8.0 新增的策略）：

```cpp
      case HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE:
      case HA_ALTER_INPLACE_NO_LOCK:
      case HA_ALTER_INPLACE_INSTANT:
        /*
          Note that any instant operation is also in fact in-place operation.

          It is totally safe to execute operation using instant algorithm if it
          has no drawbacks as compared to in-place algorithm even if user
          explicitly asked for ALGORITHM=INPLACE. Doing so, also allows to
          keep code in engines which support only limited subset of in-place
          ALTER TABLE operations as instant metadata only changes simple.

          If instant algorithm has some downsides to in-place algorithm and user
          explicitly asks for ALGORITHM=INPLACE it is responsibility of storage
          engine to fallback to in-place algorithm execution ...
        */
        break;
```
> ★★ **语义要点**：`ALGORITHM=INPLACE` 可以被**静默升级**成 INSTANT；但 `ALGORITHM=COPY` **永远不会被升级**——它是三者的最低兜底。

**★ INSTANT 白名单的扩大**（`handler0alter.cc:148`，★ 解释"COPY 范围收窄"）：

```cpp
/** Operation allowed with ALGORITHM=INSTANT */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN |
    Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN |     /* ★ 8.0.12+ */
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN;          /* ★ 8.0.29+ */
```
> ★ 其中 `ADD_STORED_BASE_COLUMN` 与 `DROP_STORED_COLUMN` 是后期才加入的——**这些以前必须 COPY 或 INPLACE 重建，现在可 INSTANT**。这正是"COPY 兜底范围持续收窄"的机制。

**★ 8.0 的 handler 接口统一**（`sql/handler.h:6013`，★ 那段设计文档值得读）：

```cpp
    *) Then the handler::check_if_supported_inplace_alter() method is called
       in order to find if the storage engine can carry out changes requested
       by this ALTER TABLE using the in-place or instant algorithm.
       ...
       Note that in cases when there is difference between in-place and
       instant algorithm and user explicitly asked for usage of in-place
       algorithm storage engine MUST return one of values corresponding
       to in-place algorithm and not HA_ALTER_INPLACE_INSTANT from this
       method.
    *) Locking requirements of the in-place algorithm are compared to any
       concurrency requirements specified by user. If there is a conflict
       between them, we either switch to the copy algorithm or emit an error.
```
> ★ 最后一句是 COPY 在 8.0 架构中的正式定位：**"要么退到 copy，要么报错"**。

---

## 理论基础

### 设计思想与权衡

**要解决的问题**：ALTER 改的是"已存在的表"，数据不能丢、中途失败不能留半成品。最朴素的正确做法就是三段式——**建新 → 搬数据 → 原子换名**。rename 是 server 与引擎两层都天然原子的操作，用它做切换点，中间状态（半新的表）对外永远不可见。COPY 就是这个朴素方案的直接实现。

**为什么保留它（而不是删掉）**：

- **完全通用**：不依赖引擎的任何能力——引擎只需支持"建表 + INSERT + rename"。任何存储引擎、任何列变更组合都能做。INPLACE 则要求引擎逐项实现（InnoDB 的 `check_if_supported_inplace_alter` 里长长的拒绝清单就是证据）。
- **语义实现最直白**：类型转换逐行逐字段发生在 `Copy_field` 里、生成列逐行重新求值、CHECK 约束逐行检查——全部走既有 DML 代码路径，没有"为 ALTER 特制"的逻辑，出错面小。

**放弃了什么、代价是什么**：

- **拷贝期 MDL SHARED_NO_WRITE、rename 前才升 EXCLUSIVE**（不 online 的根因）：数据在 server 层搬，server 无法容忍"搬一半时表定义被并发改"，所以执行期整表禁写（★ 但**仍允许读**）。这是通用性换来的——server 层方案拿不到引擎的并发能力。源码见「设计权衡」节的锁时间线。

**★ 官方的设计定位**（`sql/handler.h:6013`，★ 最后一句定义了 COPY 的角色）：

```cpp
    *) Locking requirements of the in-place algorithm are compared to any
       concurrency requirements specified by user. If there is a conflict
       between them, we either switch to the copy algorithm or emit an error.
```
> ⇒ **COPY = "要么退到这里，要么报错"**。它是三算法里唯一的**无条件可行**者，代价是最差的性能与并发。

**★ 一个容易被忽略的设计取舍**：COPY 的"朴素"其实是刻意的——

```cpp
/* sql/sql_table.cc:16287 —— SQL 层总纲 */
  Altering a table can be done in two ways. The table can be modified
  directly using an in-place algorithm, or the changes can be done using
  an intermediate temporary table (copy). In-place is the preferred
  algorithm as it avoids copying table data. The storage engine
  selects which algorithm to use in check_if_supported_inplace_alter()
  based on information about the table changes from fill_alter_inplace_info().
```
- **逐行付 4 次 SQL 层开销**（字段拷贝/类型转换、生成列求值、约束检查、跨 handler 边界的 `ha_write_row`），INPLACE rebuild 在引擎内 bulk load 全部绕开（见「设计权衡」的对照表）。
- **两倍磁盘**：中间表是真实的 InnoDB 表（自己的 .ibd），加索引排序的临时空间。

**简化假设的失效场景**：COPY 的耗时线性于行数，"搬完再说"的假设在大表上失效——这是 gh-ost / pt-osc 这类外部工具出现的直接原因（见 [`online_ddl_tools.md`](online_ddl_tools.md)）。

### 算法与数据结构

| 机制 | 落点 | 为什么这么选 |
|---|---|---|
| `Copy_field` 数组 | 按目标列数分配，每行 `invoke_do_copy()` | 逐字段函数指针，类型变更时自然落到类型转换；没有为"无变更列"做批量 memcpy 的优化——ALTER 场景里总有变更列，收益有限 |
| bulk insert 提示 | `ha_start_bulk_insert(预估行数)` | 行数来自 `info(HA_STATUS_VARIABLE)`，让引擎预分配页与攒批写；是 COPY 仅有的两个补偿优化之一 |
| `skip_alter_undo` | `HA_EXTRA_BEGIN_ALTER_COPY` → 引擎置位 | 中间表在事务提交前对外不可见，undo 无读者——免 undo + 免行锁是安全的；另一个补偿优化 |
| 进度估计 | 行数 × 已拷行数 → PFS stage | 估计值一次性取自优化器统计，不精确（同 online DDL 的"wild guess"问题） |

**★ `copy[]` 数组的构建**（`sql/sql_table.cc:18515`，★ COPY 最核心的数据结构）：

```cpp
  List_iterator<Create_field> it(create);       /* create = alter_info->create_list（新表定义） */
  const Create_field *def;
  copy_end = copy;
  gen_fields_end = gen_fields;
  ...
  for (ptr = to->field; *ptr; ptr++) {
    def = it++;
    if ((*ptr)->is_gcol()) {
      *(gen_fields_end++) = *ptr;               /* 生成列 → 延后求值，不直接拷 */
      continue;
    }
    if (def->field) {
      if (*ptr == to->next_number_field) {
        auto_increment_field_copied = true;
        /*
          If we are going to copy contents of one auto_increment column to
          another auto_increment column it is sensible to preserve zeroes.
          ...
        */
        if (def->field == from->found_next_number_field)
          thd->variables.sql_mode |= MODE_NO_AUTO_VALUE_ON_ZERO;
      }
      (copy_end++)->set(*ptr, def->field);      /* ★★ 逐列装一个 Field::set() 回调 */
    } else { ... }
```
> ★★ 三个数组构成 COPY 的"列映射表"：
> - `copy[]` —— 普通列，逐列一个 `Copy_field`（内含 `Field::set()`，负责**类型转换 + 字符集转换**）
> - `gen_fields[]` —— 生成列 / 表达式默认值列，**延后**在拷贝循环里逐行求值
> - `create_list` —— 新表定义（来自 parser）
>
> ⚠️ 一个易忽略的细节：**从 auto_increment 列拷到 auto_increment 列时会临时打开 `MODE_NO_AUTO_VALUE_ON_ZERO`** —— 为了保留 0 值而不是触发自增。

**★ 双 TABLE 对象并存**（`sql/sql_table.cc:17045`）：

```cpp
  handlerton *new_db_type = create_info->db_type;
  handlerton *old_db_type = table->s->db_type();
  TABLE *new_table = nullptr;        /* ★ 中间表 */
  ha_rows copied = 0, deleted = 0;
```
> ★ `table`（原表）与 `new_table`（中间表）在整个 ALTER 期间**同时打开**，路径/名字由 `alter_ctx.get_tmp_path()` / `alter_ctx.tmp_name` 提供。这也是 COPY 期间内存与文件句柄开销翻倍的原因。

**★ `Alter_info` 的 flag 与算法枚举**（⚠️ 真名不在 `sql_class.h`）：

```cpp
/* sql/sql_alter.h:354 —— 是 Alter_info 的嵌套枚举 */
  enum enum_alter_table_algorithm {
    // In-place if supported, copy otherwise.
    ALTER_TABLE_ALGORITHM_DEFAULT,
    // In-place if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_INPLACE,
    // Instant if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_INSTANT,
    // Copy if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_COPY
  };
```
```cpp
/* sql/sql_alter.h:467 —— 默认 DEFAULT = "能用 INPLACE 就用，否则 COPY" */
        requested_algorithm(ALTER_TABLE_ALGORITHM_DEFAULT),
        requested_lock(ALTER_TABLE_LOCK_DEFAULT),
```
```cpp
/* sql/sql_yacc.yy:9141 —— ALGORITHM 是 ident 不是保留字 */
        | ident
          {
            if (is_identifier($1, "INPLACE"))
              $$= Alter_info::ALTER_TABLE_ALGORITHM_INPLACE;
            else if (is_identifier($1, "INSTANT"))
              $$= Alter_info::ALTER_TABLE_ALGORITHM_INSTANT;
            else if (is_identifier($1, "COPY"))
              $$= Alter_info::ALTER_TABLE_ALGORITHM_COPY;
            else
            {
              my_error(ER_UNKNOWN_ALTER_ALGORITHM, MYF(0), $1.str);
              MYSQL_YYABORT;
            }
          }
```
> ★ 因为是 `ident` 不是保留字，所以**可以有一列真的叫 `algorithm`**——靠 `is_identifier()` 字面量比对区分。写错报 `ER_UNKNOWN_ALTER_ALGORITHM`。

```cpp
/* sql/sql_alter.h:205 —— ALTER_RECREATE 就是 FORCE / ENGINE=<同引擎> 的标记 */
    /// Set for FORCE
    /// Set for ENGINE(same engine)
    /// Set by mysql_recreate_table()
    ALTER_RECREATE = 1ULL << 10,
```
```cpp
/* sql/sql_table.cc:17026 */
  /*
    ALTER TABLE ... ENGINE to the same engine is a common way to
    request table rebuild. Set ALTER_RECREATE flag to force table
    rebuild.
  */
  if (create_info->db_type == table->s->db_type() &&
      create_info->used_fields & HA_CREATE_USED_ENGINE)
    alter_info->flags |= Alter_info::ALTER_RECREATE;
```
> ★ 注意：`ENGINE=<同引擎>` 走的是 `ALTER_RECREATE` → `RECREATE_TABLE` → `INNOBASE_ALTER_REBUILD` ⇒ **INPLACE 重建，不是 COPY**。这与直觉相反。

---

## 核心实现

### 主链路

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

> 必填图：COPY 与 INPLACE rebuild 都遵循"建新 + rename 替换"的骨架，差异只在 ②~⑦ 由谁做、怎么搬——server 层逐行 vs 引擎内 bulk load。

### 中间表的构建：`create_table_impl` 与 `do_not_store_in_dd`

```cpp
/*
  For temporary tables or tables in SEs supporting atomic DDL dd::Table
  object describing new version of table. This object will be created in
  memory in create_table_impl() and will not be put into the on-disk DD
  and DD Object Cache.

  We become responsible for destroying this dd::Table object ...
*/
std::unique_ptr<dd::Table> non_dd_table_def;
{
  Disable_binlog_guard binlog_guard(thd);
  /* Prevent intermediate commits to invoke commit order */
  Implicit_substatement_state_guard substatement_guard(...);

  error = create_table_impl(
      thd, *new_schema, alter_ctx.new_db, alter_ctx.tmp_name,
      alter_ctx.table_name, alter_ctx.get_tmp_path(), create_info, alter_info,
      true, 0, true, true,
      /*
        If target SE supports atomic DDL do not store
        new table version in on-disk DD.
        It is not required to rollback statement in
        case of error and allows to keep correct names
        for pre-existing foreign keys in the dd::Table
        object for new table version.
       */
      (new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL), nullptr, &key_info,
      &key_count, keys_onoff, &fk_key_info, &fk_key_count, alter_ctx.fk_info,
      alter_ctx.fk_count, old_table_def,
      alter_ctx.fk_max_generated_name_number, &non_dd_table_def, nullptr);
}
```

★ 那个 `(new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)` 就是 `do_not_store_in_dd` 参数——**目标引擎支持原子 DDL 时，新表版本的 `dd::Table` 只活在内存里**（`non_dd_table_def`），不落磁盘 DD。注释给了两个理由：出错时不需要回滚 statement；且能保留已存在外键的原名（避开新旧定义的外键名冲突）。这正是 [`atomic_ddl.md`](atomic_ddl.md)「崩溃安全五条机制」第 2 条在 COPY 路径上的源码出处。

> 顺带一提：中间表就是走 [`create_table.md`](create_table.md) 讲的建表路径建出来的——**COPY 复用了 CREATE TABLE 的实现**，这也是那篇里 `rea_create_base_table` / `ha_create_table` 被称为"共用接缝"的原因。

随后进入拷贝（`stage_copy_to_tmp_table` 正是 `SHOW PROCESSLIST` 里 `copy to tmp table` 的来源）：

```cpp
THD_STAGE_INFO(thd, stage_copy_to_tmp_table);
...
// copy ddl
if (copy_data_between_tables(thd, thd->m_stage_progress_psi, table,
                             new_table, alter_info->create_list, &copied,
                             &deleted, alter_info->keys_onoff, &alter_ctx))
  goto err_new_table_cleanup;

DEBUG_SYNC(thd, "alter_after_copy_table");
```

### 拷贝之后的收尾：备份名 + rename（server 侧）

数据拷完之后的收尾比拷贝本身更讲究——**旧表先改名成备份名，新表再转正**：

```cpp
THD_STAGE_INFO(thd, stage_rename_result_table);

if (wait_while_table_is_used(thd, table, HA_EXTRA_PREPARE_FOR_RENAME))
  goto err_new_table_cleanup;      // 等所有还开着这张表的线程退出

/*
  To ensure DDL atomicity after this point support from both old and
  new engines is necessary. If either of them lacks such support let
  us commit transaction so changes to data-dictionary are more closely
  reflect situations in SEs.

  Also if new SE supports atomic DDL then we have not stored new table
  definition in on-disk data-dictionary so far. It is time to do this
  now if ALTER TABLE as a whole won't be atomic.
*/
if (!atomic_replace) {             // ★ 只有"新旧引擎都支持原子 DDL"时才不提前提交
  if ((new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
      thd->dd_client()->store(non_dd_table_def.get()))
    goto err_new_table_cleanup;

  /* Prevent intermediate commits to invoke commit order */
  Implicit_substatement_state_guard substatement_guard(thd);

  if (trans_commit_stmt(thd) || trans_commit_implicit(thd))
    goto err_new_table_cleanup;
}

// ★ 备份表名格式：#sql2-<pid>-<thread_id>
char backup_name[32];
snprintf(backup_name, sizeof(backup_name), "%s2-%lx-%x", tmp_file_prefix,
         current_pid, thd->thread_id());
if (lower_case_table_names) my_casedn_str(files_charset_info, backup_name);

close_all_tables_for_name(thd, table->s, false, nullptr);

/*
  Rename the old version to temporary name to have a backup in case
  anything goes wrong while renaming the new table.

  Take the X metadata lock on this temporary name too. This ensures that
  concurrent I_S queries won't try to open it.
*/
{
  MDL_request backup_name_mdl_request;
  MDL_REQUEST_INIT(&backup_name_mdl_request, MDL_key::TABLE, alter_ctx.db,
                   backup_name, MDL_EXCLUSIVE, MDL_STATEMENT);
  ...
  if (backup_table != nullptr) {          // 备份名竟然已存在 → 直接失败
    my_error(ER_TABLE_EXISTS_ERROR, MYF(0), backup_name);
    if (!atomic_replace) quick_rm_table(...);
    goto err_with_mdl;
  }
}
```

三个要点：

1. **备份名 `#sql2-<pid>-<tid>`** 是旧表的临时归宿。新表转正失败时用它改回原名——非原子路径下这个"改回"还要**重试 20 次**（见 [`atomic_ddl.md`](atomic_ddl.md) 的失败回滚路径）。
2. **备份名也要拿 MDL X**——注释解释得很清楚：防止并发的 `information_schema` 查询去打开这个临时表（它的名字是可预测的，不锁就会被查到）。
3. `if (!atomic_replace)` 这一整段与原子 DDL 直接呼应：**双引擎都支持原子 DDL 时不提前提交**，整个 COPY 留在单个事务里；否则才提交并把新定义落盘——这正是 [`atomic_ddl.md`](atomic_ddl.md) 里 `do_not_store_in_dd` 技巧在 COPY 路径上的落点。

### `copy_data_between_tables` 逐段剖析

核心函数 `copy_data_between_tables`（sql/sql_table.cc，约 300 行），`from`=源表、`to`=新建中间表。COPY 的"搬数据"全在这里。

#### 第一段：准备

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

**③ 免 undo + 免锁**：`to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY)` → `skip_alter_undo=1`（剖析见 [`../../server/handler.md`](../../server/handler.md) 的 handler::extra 章）。

**④ 字段分类：哪些拷贝、哪些重新求值**。准备段遍历新表的每一个字段，把它分到两个数组之一——这个划分决定了后面循环体做什么：

```cpp
  for (ptr = to->field; *ptr; ptr++) {
    def = it++;
    if ((*ptr)->is_gcol()) {
      /* Values in generated columns need to be (re)generated even for
      pre-existing columns, as they might depend on other columns,
      values in which might have changed as result of this ALTER.
      Because of this there is no sense in copying old values for
      these columns. */
      *(gen_fields_end++) = *ptr;                   // 生成列 → 重新求值
      continue;
    }
    if (def->field) {                               // 源表有同名列 → 拷贝
      if (*ptr == to->next_number_field) {
        auto_increment_field_copied = true;
        /* If we are going to copy contents of one auto_increment column to
        another auto_increment column it is sensible to preserve zeroes. */
        if (def->field == from->found_next_number_field)
          thd->variables.sql_mode |= MODE_NO_AUTO_VALUE_ON_ZERO;
      }
      (copy_end++)->set(*ptr, def->field);          // 建立拷贝对
    } else {                                        // 新列
      if ((*ptr)->has_insert_default_general_value_expression())
        *(gen_fields_end++) = *ptr;                // 有生成默认值 → 求值
    }
  }
```

★ 三个分支的语义：**生成列一律重新求值**（注释说明：即使列已存在也要重算，因为它可能依赖本次被改的列，拷旧值没意义）；**源表有同名列 → 进 `copy` 数组**逐字段拷贝；**全新列 → 只有带"生成默认值表达式"的才需要求值，普通默认值由 `set_column_static_defaults` 填**。

另外那个 `MODE_NO_AUTO_VALUE_ON_ZERO` 值得一提：当"把一个自增列拷到另一个自增列"时，**0 值要原样保留**（否则 0 会被当成"请求新自增值"的记号而被换掉）。

**⑤ 读源表的路径是怎么构造的**——用的是 8.0 的 `AccessPath` + `RowIterator` 体系，不是老的 `read_record`：

```cpp
  AccessPath *path = create_table_access_path(thd, from, nullptr, nullptr, nullptr,
                                              /*count_examined_rows=*/false);

  if (order != nullptr && to->s->primary_key != MAX_KEY &&
      to->file->primary_key_is_clustered()) {
    char warn_buff[MYSQL_ERRMSG_SIZE];
    snprintf(warn_buff, ..., "ORDER BY ignored as there is a user-defined clustered index"
                             " in the table '%-.192s'", from->s->table_name.str);
    push_warning(thd, Sql_condition::SL_WARNING, ER_UNKNOWN_ERROR, warn_buff);
    order = nullptr;                                // ★ 有用户聚簇索引 → ORDER BY 被忽略
  }

  if (order != nullptr) {
    fsort.reset(new (thd->mem_root) Filesort(
        thd, {from}, /*keep_buffers=*/false, order, HA_POS_ERROR,
        /*remove_duplicates=*/false, /*force_sort_rowids=*/true, ...));
    path = NewSortAccessPath(thd, path, fsort.get(), order, false);   // 排序包一层
  }

  iterator = CreateIteratorFromAccessPath(thd, path, /*join=*/nullptr,
                                          /*eligible_for_batch_mode=*/true);
  from->sorting_iterator = nullptr;      // 防止 QEP_shared_owner 清理时重复销毁
  if (iterator == nullptr || iterator->Init()) { error = 1; goto err; }
```

★ 两点：

1. **`ALTER TABLE ... ORDER BY` 只在"没有用户定义聚簇索引"时才真正排序**——若表已有主键（InnoDB 聚簇索引），存储顺序由聚簇键决定，`ORDER BY` 无从生效，源码选择**推一条 warning 并置空 `order`**。很多运维脚本里的 `ORDER BY` 就这样被静默忽略了。
2. **排序走的是 `Filesort` 且 `force_sort_rowids=true`**——即"排 rowid 再回表取行"，因为要按新顺序写入；排完由 `SortAccessPath` 包住底层的全表扫描。

⇒ 所以 COPY 的"读"是三层：全表扫描 AccessPath（→ 可选 Filesort）→ `RowIterator` → 逐行 `Read()`。它完全走 SQL 执行器，享受不到任何引擎侧快路径。

#### 第二段：拷贝循环—— 核心

```cpp
while (!(error = iterator->Read())) {          // 逐行读源表（走标准执行器迭代器）
  if (thd->killed) { thd->send_kill_message(); error = 1; break; }
```

注意这个迭代器由 `CreateIteratorFromAccessPath` 建立——即**透过整个 SQL 层**的全表扫描，COPY 的"读"也享受不到任何引擎侧快路径。

**步骤 1：字段拷贝（类型转换点）**

```cpp
for (Copy_field *copy_ptr = copy; copy_ptr != copy_end; copy_ptr++) {
  copy_ptr->invoke_do_copy();
}
```

`Copy_field` 数组按字段数分配。`invoke_do_copy()` 做源→目标字段拷贝；若类型变了（INT→BIGINT、字符集变更），**这里就是实际的类型转换点**，逐字段逐行。

**但"选哪种拷贝方式"是在准备阶段就定好的**——`Copy_field::set()`（`field_conv.cc`）为每个字段对选好函数指针，循环里只管调用。**两级分派**是理解它的关键：

```cpp
void Copy_field::set(Field *to, Field *from) {
  if (to->type() == MYSQL_TYPE_NULL) {        // ① 目标是 NULL 类型 → 直接跳过
    m_do_copy = do_skip;
    return;
  }
  m_from_field = from;
  m_to_field = to;

  m_do_copy2 = get_copy_func();               // ② 底层拷贝函数（按类型选）

  if (m_from_field->is_nullable() || m_from_field->table->is_nullable()) {
    if (m_to_field->is_nullable() || m_to_field->is_tmp_nullable())
      m_do_copy = do_copy_null;               // ③ 源可 NULL → 按目标性质包装
    else if (m_to_field->type() == MYSQL_TYPE_TIMESTAMP)
      m_do_copy = do_copy_timestamp;          //    自动时间戳
    else if (m_to_field == m_to_field->table->next_number_field)
      m_do_copy = do_copy_next_number;        //    自增列
    else
      m_do_copy = do_copy_not_null;           //    目标 NOT NULL：源 NULL 要处理
  } else if (m_to_field->is_nullable()) {
    m_do_copy = do_copy_maybe_null;
  } else {
    m_do_copy = m_do_copy2;                   // ★ ④ 两端都 NOT NULL：直接用底层函数
  }
}
```

★ 也就是说 `m_do_copy` 是**包装层**（处理 NULL / 时间戳 / 自增这些语义），`m_do_copy2` 是**底层类型转换函数**。第 ④ 行是**最快的路径**——两端都不可为 NULL 时，连包装都省了，直接调底层函数。反过来，只要涉及 NULL 可能性，每行都要多一层判断。

底层函数由 `get_copy_func()` 按类型选，其中最能说明"转换代价"的是几何类型分支：

```cpp
  if (m_to_field->type() == MYSQL_TYPE_GEOMETRY) {
    if (m_from_field->type() != MYSQL_TYPE_GEOMETRY ||
        m_to_field->is_nullable() != m_from_field->is_nullable() ||
        m_to_field->table->is_nullable() != m_from_field->table->is_nullable())
      return do_conv_blob;                    // ① 类型/nullable 不同 → 全转换

    // If changing the SRID property of the field, we must do a full conversion.
    if (to_geom->get_srid().has_value() &&
        to_geom->get_srid() != from_geom->get_srid())
      return do_conv_blob;                    // ② ★ 只改 SRID 也要全转换

    // to is same as or a wider type than from
    if (to_geom->get_geometry_type() == from_geom->get_geometry_type() ||
        is_subtype_of(from_geom->get_geometry_type(), to_geom->get_geometry_type()))
      return do_field_eq;                     // ③ ★ 同类型或更宽 → 直接拷，不转换

    return do_conv_blob;
  }
```

⇒ **`do_field_eq` 是"内存直接拷"的快路径**（类型相同或目标更宽时），`do_conv_*` 才是真正的转换。所以 COPY 的逐行代价不是固定的：

- **类型兼容**（`do_field_eq`）⇒ 每行只是一次内存搬移，极快；
- **类型不兼容**（`do_conv_*`）⇒ 每行每个字段都要走一次转换（含字符集转换时的分配与编码），这才是 COPY 慢的主要来源之一。

顺带一条：几何列**仅仅改 SRID 也会强制 `do_conv_blob`**（源码注释点明）——这也是前文 `is_inplace_alter_impossible` 把"改 SRID"列为强制 COPY 场景的又一个佐证。

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
    expr_item = (*ptr)->m_default_val_expr->expr_item;        // 生成默认值
  }
  expr_item->save_in_field(*ptr, false);                      // ★ 逐行求值
}
```

**生成列的值不是从源表拷的，而是逐行重新求值**。源码注释说明两条约束：必须在"拷旧列 + 填新列默认值"**之后**（生成值可能依赖它们）；必须**按表中列顺序**处理（生成列可依赖前面的生成列，不允许前向引用）。

**步骤 4：约束检查 + 写入引擎**

```cpp
error = invoke_table_check_constraints(thd, to);    // CHECK 约束
if (error) break;
error = to->file->ha_write_row(to->record[0]);      // ★ 真正写入引擎
```

`ha_write_row` → `ha_innobase::write_row` → `row_insert`，此时 `skip_alter_undo` 生效（不写 undo、不加锁）。

**错误处理**区分两类：

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

注意错误消息用的是 **`from->s->table_name`（源表名）**——对用户而言他操作的是原表，中间表名绝不外泄。

**进度上报**：

```cpp
found_count++;
mysql_stage_set_work_completed(psi, found_count);            // ★ PFS 进度
thd->get_stmt_da()->inc_current_row_for_condition();         // 行号（错误定位用）
```

这就是 `ALTER TABLE` 在 `performance_schema.events_stages_current` 里能看到 `WORK_COMPLETED / WORK_ESTIMATED` 的来源，估计值来自 `from->file->info(HA_STATUS_VARIABLE)`。

#### 第三段：收尾

```cpp
if (to->file->ha_end_bulk_insert() && error <= 0) { ... }    // 结束批量插入
to->file->ha_extra(HA_EXTRA_END_ALTER_COPY);                 // 复位 skip_alter_undo + 重算统计
if ((!(to->file->ht->flags & HTON_SUPPORTS_ATOMIC_DDL) || from->s->tmp_table) &&
    mysql_trans_commit_alter_copy_data(thd)) error = 1;       // 与①对称
err:
  ...
  if (to->file->ha_external_lock(thd, F_UNLCK)) error = 1;   // 与 F_WRLCK 配对
  if (error < 0 && to->file->ha_extra(HA_EXTRA_PREPARE_FOR_RENAME)) error = 1;
```

`HA_EXTRA_END_ALTER_COPY` 置 `skip_alter_undo=0` 并触发 `alter_stats_rebuild` 重算统计；只有非原子引擎才显式提交拷贝事务；解锁与加锁严格配对。

### 设计权衡：COPY 慢在哪（代码级答案）

循环里**每行要付 4 次 SQL 层开销**：

| 步骤 | 开销 | INPLACE rebuild 是否付 |
|---|---|---|
| ① `invoke_do_copy()` 字段拷贝/类型转换 | 逐字段 | ❌ 引擎内直接按新格式构建 |
| ② 生成列逐行求值 | 逐行表达式求值 | ❌ bulk load 时统一处理 |
| ③ `invoke_table_check_constraints` | 逐行约束检查 | ❌ |
| ④ `ha_write_row` 走 handler 边界 | 逐行跨 SQL→引擎 | ❌ 直接建 B-tree |

**仅有的补偿**：`skip_alter_undo`（免 undo + 免锁）与 `ha_start_bulk_insert`（攒批）——这解释了 COPY 虽慢但仍可用。

**对比 INPLACE rebuild**：后者在 `ddl::Builder` 里扫描聚簇索引后**直接 bulk load 构建 B-tree**，完全不经上述 4 步。这就是"INPLACE 快一个数量级"的代码级依据（构建引擎见 [`ddl_engine.md`](ddl_engine.md)）。

**★★★ 锁的真实时间线**（★ 这是"COPY 有多不 online"的精确答案）：

```
开表  →  MDL_SHARED_UPGRADABLE (SU)
拷贝  →  MDL_SHARED_NO_WRITE  (SNW)   ← 可读，禁写；★ 绝大部分时间在这里
rename →  MDL_EXCLUSIVE        (X)     ← 短暂，仅切换瞬间
```
```cpp
/* sql/sql_table.cc:17578 —— 拷贝阶段升 SNW（不是 X！）*/
    /*
      Otherwise upgrade to SHARED_NO_WRITE.
      Note that under LOCK TABLES, we will already have SHARED_NO_READ_WRITE.
    */
    if (alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE &&
        thd->mdl_context.upgrade_shared_lock(mdl_ticket, MDL_SHARED_NO_WRITE,
                                             thd->variables.lock_wait_timeout))
      goto err_new_table_cleanup;
```
```cpp
/* sql/sql_base.cc:2472 —— rename 前才升 X */
  if (thd->mdl_context.upgrade_shared_lock(table->mdl_ticket, MDL_EXCLUSIVE,
                                           thd->variables.lock_wait_timeout))
    return true;
  tdc_remove_table(thd, TDC_RT_REMOVE_NOT_OWN, table->s->db.str,
                   table->s->table_name.str, false);
```
```cpp
/* sql/parse_tree_nodes.cc:3086 —— ALTER 开表时只拿 SU */
  if (!lex->query_block->add_table_to_list(
          pc->thd, table_name, nullptr, TL_OPTION_UPDATING, TL_READ_NO_INSERT,
          MDL_SHARED_UPGRADABLE))
```
> ★★ **三个实用结论**：① COPY 期间 **SELECT 可以跑**（SNW 允许读），但**所有 DML 被挡**；② `LOCK=NONE` 直接报错，reason 文本是 *"COPY algorithm requires a lock"*；③ `LOCK=EXCLUSIVE` 会在**拷贝开始前**就升 X（`wait_while_table_is_used`），比默认的 SNW 更激进。

```cpp
/* sql/sql_table.cc:17561 —— LOCK=NONE 直接拒 */
    // COPY algorithm doesn't work with concurrent writes.
    if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_NONE) {
      my_error(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON, MYF(0), "LOCK=NONE",
               ER_THD(thd, ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_COPY),
               "LOCK=SHARED");
      goto err_new_table_cleanup;
    }
```

**★ 逐行的三项 SQL 层开销源码**（★ INPLACE 重建跑不到）：

```cpp
/* sql/sql_table.cc:18655 —— 每行：生成列/表达式默认值求值 + CHECK 约束 */
    /*
      Iterate through all generated columns and all new columns which have
      generated defaults and evaluate their values. This needs to happen
      after copying values for old columns and storing default values for
      new columns without generated defaults, as generated values might
      depend on these values.
      OTOH generated columns/generated defaults need to be processed in
      the order in which their columns are present in table ...
    */
    for (ptr = gen_fields; ptr != gen_fields_end; ptr++) {
      Item *expr_item;
      if ((*ptr)->is_gcol()) {
        expr_item = (*ptr)->gcol_info->expr_item;
      } else {
        expr_item = (*ptr)->m_default_val_expr->expr_item;
      }
      expr_item->save_in_field(*ptr, false);
      if (thd->is_error()) { error = 1; break; }
    }
    if (error) break;

    error = invoke_table_check_constraints(thd, to);
    if (error) break;
```
```cpp
/* sql/sql_table.cc:18497 —— 全局开截断检查（每行都判） */
  /*
    We want warnings/errors about data truncation emitted when we
    copy data to new version of table.
  */
  thd->check_for_truncated_fields = CHECK_FIELD_WARN;
```
```cpp
/* sql/sql_table.cc:18515 —— 逐列 Field::set()：类型转换 + 字符集转换 */
      (copy_end++)->set(*ptr, def->field);   /* ★ copy[] 是 Copy_field 数组 */
```
> ★★ **改字符集时最贵的就是最后这一行**：`copy[]` 是逐列的 `Field::set()` 函数指针数组，每行每列都要做一次**编码转换**——这是"改字符集特别慢"的代码级原因。

**★ 两项缓解措施**（★ 说明 COPY 不是纯无优化）：

```cpp
/* sql/sql_table.cc:18505 —— bulk insert */
  to->file->ha_start_bulk_insert(from->file->stats.records);
  mysql_stage_set_work_estimated(psi, from->file->stats.records);
```
```cpp
/* sql/sql_table.cc:18399 —— 拷贝前关恢复日志（只是 hint） */
bool mysql_trans_prepare_alter_copy_data(THD *thd) {
  /*
    Turn off recovery logging since rollback of an alter table is to
    delete the new table so there is no need to log the changes to it.
    ...
  */
  if (ha_enable_transaction(thd, false)) return true;
  return false;
}
```
> ⚠️ `ha_enable_transaction(false)` 只是 **hint**，源码注释明说 *"This hint can be safely ignored"*——**InnoDB 实际上仍会写 redo/undo**。所以"COPY 不写 redo"的说法是错的。

---

## 可观测性

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 拷贝进度 | PFS | `performance_schema.events_stages_current` 的 `WORK_COMPLETED / WORK_ESTIMATED`，来自拷贝循环内的 `mysql_stage_set_work_completed`；估计值 = `info(HA_STATUS_VARIABLE)` 取的行数，不精确 |
| 正在走 COPY | `SHOW PROCESSLIST` | State 显示 `copy to tmp table`；或从 SQL 文本直接确认 `ALGORITHM=COPY` |
| 强制走 COPY | 系统变量 | `old_alter_table=ON`（Session 级）；官方推荐用 `ALGORITHM=COPY` 子句做单语句级对比 |
| 中间表痕迹 | SQL | `information_schema.INNODB_TEMP_TABLE_INFO`（`#sql-` 前缀）；DDL 失败残留时靠它定位 |

**★ 阶段名的真名与埋点**（⚠️ 不是 "Copying to tmp table"）：

```cpp
/* sql/mysqld.cc:12192 */
PSI_stage_info stage_copy_to_tmp_table= { 0, "copy to tmp table",
                                          PSI_FLAG_STAGE_PROGRESS,
                                          PSI_DOCUMENT_ME};
```
```cpp
/* sql/sql_table.cc:17740 —— 进入拷贝前的埋点 */
    THD_STAGE_INFO(thd, stage_copy_to_tmp_table);
```
```cpp
/* sql/sql_table.cc:18507 —— 预估总行数 */
  mysql_stage_set_work_estimated(psi, from->file->stats.records);
```
```cpp
/* sql/sql_table.cc:18711 —— 每完成一行上报 */
    } else {
      DEBUG_SYNC(thd, "copy_data_between_tables_before");
      found_count++;
      mysql_stage_set_work_completed(psi, found_count);
    }
```
> ★★ 因为带 `PSI_FLAG_STAGE_PROGRESS`，**可以直接算百分比**：
> ```sql
> SELECT WORK_COMPLETED, WORK_ESTIMATED,
>        ROUND(100*WORK_COMPLETED/WORK_ESTIMATED,1) AS pct
> FROM performance_schema.events_stages_current
> WHERE EVENT_NAME LIKE '%stage/sql/copy_to_tmp_table%';
> ```
> ⚠️ 注意与**查询执行**的 `"Copying to tmp table"`（首字母大写，P_S 里是 `stage/sql/Copying_to_tmp_table`）区分——那是 SELECT 建临时表，与 DDL 无关。

**★ 成功时客户端看到的行数**（`sql/sql_table.cc:18253`，★ 复用了 INSERT 的文本）：

```cpp
  snprintf(alter_ctx.tmp_name, sizeof(alter_ctx.tmp_name),
           ER_THD(thd, ER_INSERT_INFO), (long)(copied + deleted), (long)deleted,
           (long)thd->get_stmt_da()->current_statement_cond_count());
  my_ok(thd, copied + deleted, 0L, alter_ctx.tmp_name);
```
> ★ 所以 COPY 完成会显示 *"Records: N  Duplicates: 0  Warnings: M"*——格式和 INSERT 一样。N 就是拷贝的行数。

**★ 另一个副作用可见点**：COPY 的 `ha_write_row` 走 `handler.cc:8015` 的通用包装，其中的 `MYSQL_TABLE_IO_WAIT(PSI_TABLE_WRITE_ROW, ...)` **会累加 P_S 的 table io wait**——所以 COPY 的行插入在 `table_io_waits_summary_by_table` 里是**看得见**的（只是记在中间表名下）。

---

## Misc

### 面向二次开发

**扩展点**：新存储引擎若不支持 `HTON_SUPPORTS_ATOMIC_DDL`，COPY 会走"提前提交拷贝事务"分支（`mysql_trans_prepare_alter_copy_data`）——引擎要自己保证拷贝中途崩溃的清理。这是接入新引擎时 COPY 路径的第一个检查点。

**★★ 想让某个操作"强制走 COPY"，三处改动点（按成本从低到高）**：

**改动点 1（成本最低，SQL 层直接判）** —— 在 `is_inplace_alter_impossible()` 加一个 `return true`：

```cpp
/* sql/sql_table.cc:12794 —— 在此函数体内加分支即可 */
static bool is_inplace_alter_impossible(TABLE *table,
                                        HA_CREATE_INFO *create_info,
                                        const Alter_info *alter_info) {
  DBUG_TRACE;
  /* At the moment we can't handle altering temporary tables without a copy. */
  if (table->s->tmp_table) return true;
  /* ↓ 在这里加 */
```
> ★ 命中后 `sql_table.cc:16993` 会把 `requested_algorithm` 置为 COPY。**但用户显式 `ALGORITHM=INPLACE` 时会直接报错**（16997-17002）——这通常是符合预期的。

**改动点 2（跨层，推荐）** —— 新增 `HA_ALTER_FLAGS` 位：

```cpp
/* sql/handler.h:3216 —— ulonglong，8.0.39 已用到 bit 49，剩 50-63 可用 */
  typedef ulonglong HA_ALTER_FLAGS;
  static const HA_ALTER_FLAGS ADD_INDEX = 1ULL << 0;
  ...
  /* 新增：static const HA_ALTER_FLAGS MY_NEW_OP = 1ULL << 50; */
```
> ⚠️ 必须在 `fill_alter_inplace_info()`（`sql_table.cc:11958` 起）里**置位**，否则永远不会被触发。

**改动点 3（引擎侧，通常零改动）**：

```cpp
/* handler0alter.cc:998 —— 只要新位不在三个集合里，InnoDB 自动返回 NOT_SUPPORTED */
  if (ha_alter_info->handler_flags &
      ~(INNOBASE_INPLACE_IGNORE | INNOBASE_ALTER_NOREBUILD |
        INNOBASE_ALTER_REBUILD)) {
    return HA_ALTER_INPLACE_NOT_SUPPORTED;    /* → 自动降级 COPY */
  }
```
> ★★ 所以**最小改动就是"handler.h 加位 + fill_alter_inplace_info 置位"，引擎侧零改动**——这正是上面第 2 段"能力位图即边界"的实践含义。

**★ 一个必须知道的约束**：`ha_alter_info->online` **只在 INPLACE 路径设置**（`sql_table.cc:13509`），COPY 路径压根没有这个变量。所以"给 COPY 加 online 支持"不是开关问题，而是要改架构。

**坑与已知缺陷**：

| 坑 | 说明 |
|---|---|
| 截断只告警 | `CHECK_FIELD_WARN` 意味着 VARCHAR 变短等操作**静默截断数据**（仅 warning），COPY 完成后数据已损——大表上务必先在影子表验证 |
| 重复键错误发生在 DDL 早期 | 与 online DDL 不同，COPY 第一阶段建索引时就会撞重复键（中间表带完整索引拷贝）；错误消息报源表名，排查时注意 |
| 统计信息 | 中间表 CREATE 时统计置 0 行（`DICT_STATS_EMPTY_TABLE`），收尾 `alter_stats_rebuild` 重算；auto_recalc 未开且重算被跳过时（`STATS_PERSISTENT=0`）统计会过时 |

### 易混淆概念

| 概念 | 辨析 |
|---|---|
| COPY 算法 vs `SELECT ... INTO OUTFILE` 式导出 | COPY 的中间表是**引擎内真实表**（InnoDB 表空间），不是文件；搬运用 DML，不走文件 |
| `#sql-` 中间表 vs INPLACE rebuild 的新表 | COPY 的中间表由 server 层建（`create_table_impl` 正常路径）；INPLACE 的新表由引擎在 `prepare_inplace_alter_table` 内部建，server 只拿元数据 |

**★ "COPY 也有 online 标志"是错的**（`sql/sql_table.cc:13509`）：

```cpp
  switch (inplace_supported) {
    case HA_ALTER_ERROR:
    case HA_ALTER_INPLACE_NOT_SUPPORTED:
      assert(0);                    /* ★ INPLACE 路径不该走到这里 */
      [[fallthrough]];
    case HA_ALTER_INPLACE_NO_LOCK:
    case HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE:
      switch (alter_info->requested_lock) {
        case Alter_info::ALTER_TABLE_LOCK_DEFAULT:
        case Alter_info::ALTER_TABLE_LOCK_NONE:
          ha_alter_info->online = true;      /* ★★ 只有这里置 true */
          break;
        ...
```
> ★ 这段代码在 `mysql_inplace_alter_table` 路径上，COPY 完全不经过 ⇒ **COPY 永远 online=false**。

**★ "ALTER 只有三种结果"也是错的**——有第四种 no-op（`sql/sql_table.cc:17425`）：

```cpp
    if (ha_alter_info.handler_flags == 0) {
      /*
        No-op ALTER, no need to call handler API functions.
        ...
        Note that we can end up here if an ALTER statement has clauses
        that cancel each other out (e.g. ADD/DROP identically index).
        Also note that we ignore the LOCK clause here.
      */
      close_temporary_table(thd, altered_table, true, false);
      ...
      is_noop = true;
      goto end_inplace_noop;
    }
```
> ★ 例：`ALTER TABLE t ADD INDEX i(a), DROP INDEX i` → `handler_flags == 0` → **连 INPLACE 都不走**，直接 no-op。

**★ 三算法的完整降级链**（★ 一句话记住源码路径）：

```
sql_table.cc:16993  is_inplace_alter_impossible()        → 预置 COPY（SQL 层，不问引擎）
sql_table.cc:17452  check_if_supported_inplace_alter()   → 问引擎
sql_table.cc:17513  case HA_ALTER_INPLACE_NOT_SUPPORTED   → use_inplace = false
sql_table.cc:17537  if (use_inplace) mysql_inplace_alter_table()  ← INPLACE/INSTANT
                    else fall-through                     ← COPY
```

**★ 读完本篇要记住的 5 个纠正**：

| 常见说法 | 8.0.39 真相 |
|---|---|
| COPY 全程 MDL X | **SNW → X 两级**；拷贝期允许读、禁止写 |
| COPY 走 `write_record()` | **直接 `ha_write_row`**，不触发 trigger、不做 DUP 处理 |
| COPY 逐行写 binlog | **只写一条 Query event**，从库重放整条 ALTER |
| COPY 不写 redo | `ha_enable_transaction(false)` 只是 hint，**InnoDB 照样写** |
| 改字符集由 SQL 层判 COPY | 由**默认 handler 实现**判（连同 PACK_KEYS/MAX_ROWS/ROW_FORMAT） |
| 加/删主键、`AFTER`/`FIRST` 重排必须 COPY | **都可 INPLACE 重建**；只有"删主键不建"才降级 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.12 InnoDB and Online DDL*（COPY 的官方定位、`old_alter_table`、ALGORITHM/LOCK 子句语义）

**WorkLog**
- [WL#5526](https://dev.mysql.com/worklog/task/?id=5526) InnoDB: Online ADD INDEX——online DDL 体系的奠基工作，COPY 退为兜底的时间点；其"fast index create 不拷贝 undo 历史"的限制描述是理解 5.6 前 ALTER 语义的一手材料

**相关文档**
- 上层：server 侧 DDL 编排、算法选择策略、MDL 三段式、原子 DDL 见 [`ddl.md`](ddl.md)
- 另两条路：INPLACE（含 row log 与 online 支柱）见 [`inplace_ddl.md`](inplace_ddl.md)、INSTANT 见 [`instant_ddl.md`](instant_ddl.md)
- INPLACE rebuild 时索引如何被构建（bulk load 而非逐行）见 [`ddl_engine.md`](ddl_engine.md)
- `skip_alter_undo` 在引擎侧的消费点见 [`../../server/handler.md`](../../server/handler.md)
