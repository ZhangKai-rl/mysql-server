# 表（Table）：元数据与通用操作

> **定位**：SQL 层"表"这条线的完整视图——三层元数据对象、表缓存、以及表的全部通用操作（开表 / 锁表 / CREATE / ALTER / RENAME / TRUNCATE / DROP / FLUSH）。
> **边界**：MDL 的细粒度语义见 [`../infra/lock/transactional/mdl.md`](../infra/lock/transactional/mdl.md)；server↔引擎分界面见 [`handler.md`](handler.md)；InnoDB 侧 `dict_table_t` 与 COPY/INPLACE/INSTANT 算法选型见 [`../innodb/ddl.md`](../innodb/ddl.md)；表空间级 DDL（CREATE TABLESPACE）见 [`../innodb/physical/tablespace.md`](../innodb/physical/tablespace.md)；分区表见 [`../feat/partitioning.md`](../feat/partitioning.md)；临时表见 [`query/runtime/07_temptable.md`](query/runtime/07_temptable.md)。

---

## 概述

### 是什么

"表"在 server 层有四层表示，一次 `SELECT` 全部要走到：

```
DD（mysql.tables / dd::Table）        ← 持久化定义（8.0 起取代 frm）
   ↓ 首次访问时加载（DDL 后失效重建）
TABLE_SHARE                          ← 定义层，全局唯一，所有 thd 共享
   ↓ 实例化（每 thd 每表一份）
TABLE                                ← 运行态，会话私有，用完回缓存
   ↓ 语句引用
Table_ref（8.0 前称 TABLE_LIST）      ← 语句级，FROM 子句串成链表
```

引擎侧还有一个**对偶对象** `dict_table_t`（InnoDB 字典）：由 `TABLE_SHARE` 在 `ha_innobase::open` 时触发加载。二者是同一张表的两种视图（server 元数据 vs 引擎字典），分区表是一对多（见下）。

### 三个缓存 / 实例池的分工（易混）

| 缓存 | 管的对象 | 参数 | miss 代价 |
|---|---|---|---|
| `table_definition_cache`（TDC） | `TABLE_SHARE` | `table_definition_cache`（+ `_instances` 分片） | 访问 DD、重建 SHARE |
| `table_open_cache` | `TABLE` 实例 | `table_open_cache` / `_instances` | 按 SHARE 实例化 + 锁 |
| InnoDB `dict_sys` | `dict_table_t` | — | 读引擎字典表 |

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.6 前 | 全局 table cache 一把大锁 |
| 5.6 | `table_open_cache_instances` 分片（默认 16） |
| 8.0 | 元数据切到 **DD**（原子化、crash-safe DDL）；`TABLE_LIST` 改名 `Table_ref` |
| 8.0.29 | instant ADD/DROP COLUMN（见 `../innodb/ddl.md`） |

---

## 表元数据

### TABLE_SHARE：全局共享的定义层

一个表一份，从 DD 构建后由 TDC 缓存，引用计数管理生命周期。关键字段：

```cpp
struct TABLE_SHARE {
  Field **found_next_number_field;  // 指向自增列（share 的 field 数组中）
  uint next_number_index;           // 自增列所在索引的序号
  uint next_number_key_offset;      // 自增列在该索引内的字节偏移
  uint next_number_keypart;         // 自增列是该索引的第几个 keypart
  /*----*/
  uint null_bytes, last_null_bit_pos; // NULL 位图字节数；最后一个 nullable 字段的 bit 位
  /*----*/
  ulint idx_cond_n_cols;            // ICP（索引条件下推）涉及的列数；0 = 无下推
};
```

- **自增列为什么存四处**：`next_number_index/offset/keypart` 让引擎能定位"自增列在哪个索引的哪个位置"——`AUTO_INCREMENT` 与 GIPK 的字典定位都依赖它。
- `null_bytes/last_null_bit_pos` 是 NULL 位图的 **server 侧描述**，与 InnoDB `rec_init_null_and_len_comp` 解析行时读到的位图是同一物的两面（一个描述布局、一个读字节）。
- `idx_cond_n_cols` 是 ICP 的列数（0 表示没有下推条件），发动机在 engine 侧过滤后由 `handler` 接口回传。

### TABLE：会话私有的运行态

从表缓存的 `free_tables` 取出、置 `in_use`；语句结束**放回缓存而非销毁**。关键运行时字段：

```cpp
struct TABLE {
  Field *next_number_field;                       // 运行时自增列指针（取自 share）
  Field *found_next_number_field;                 // 非空则本表有自增列
  bool autoinc_field_has_explicit_non_null_value;
    // true = 用户显式提供非 NULL 自增值。
    // MODE_NO_AUTO_VALUE_ON_ZERO 下显式给的 0 不再触发生成；
    // 否则 0 / NULL 视为"待生成"。
};
```

**SHARE vs TABLE 的分工**：定义（列类型、索引、自增定位）在 SHARE 全局共享；运行态（行 buffer、自增运行值、handler 指针）在 TABLE 会话私有。

### Table_ref（原 TABLE_LIST）

把语句 FROM 中的多张表串成链表，每个节点持有一个 `TABLE*`。8030 叫 `TABLE_LIST`、8039 改名 `Table_ref`——同一类型的两代命名，读老文章时需注意。

### TABLE_SHARE ↔ dict_table_t：一对多的特例

普通表 1:1；**分区表是一对多**——`ha_innopart` 的每个分区各有一个 `dict_table_t`：

```cpp
class Ha_innopart_share : public Partition_share {
  dict_table_t **m_table_parts;   // 每个分区一个 InnoDB 字典对象
};
class ha_innopart : public ha_innobase, public Partition_helper, public Partition_handler {};
```

（DDL 时分区级操作需遍历 `m_table_parts`，这也是分区表 DDL 更慢的原因之一，详见 `../feat/partitioning.md`。）

★ `dict_table_t` 的**内部结构剖析**不在本篇——那是 InnoDB 的对象（`dict0mem.h:1939`），详见 [`dd/innodb_dict.md`](dd/innodb_dict.md) 第 1 章（标识定位 / instant 三组计数器 / autoinc 系列 / 来自 DD vs 运行态）。本篇只负责 server 层视角：`TABLE_SHARE` / `TABLE` 与它如何对应。

---

## 表的通用操作

### 1. 开表：open_table

以 `INSERT INTO t1 VALUES(1)` 为例（函数名核实于 8.0.39）：

```
mysql_execute_command
  → Sql_cmd_dml::execute → prepare
    → open_tables_for_query (sql_base.cc)
      → open_tables
        → open_and_process_table
          → open_table (sql_base.cc)
            → Table_cache::get_table (table_cache.h)
```

#### open_table 逐段解析（`sql_base.cc:2810`）

**① 前置检查与 key 构造**

```cpp
assert(!is_temporary_table(table_list) && !table_list->is_derived());   // 临时/派生表不走这里
if (check_stack_overrun(thd, STACK_MIN_SIZE_FOR_OPEN, ...)) return true; // 开表很吃栈
if (!(flags & MYSQL_OPEN_IGNORE_KILLED) && thd->killed) return true;

/* 只读事务里要写锁 → 拒绝（日志表除外，否则日志功能会失效） */
if (table_list->mdl_request.is_write_lock_request() && thd->tx_read_only &&
    !(flags & (MYSQL_LOCK_LOG_TABLE | MYSQL_OPEN_HAS_MDL_LOCK))) {
  my_error(ER_CANT_EXECUTE_IN_READ_ONLY_TRANSACTION, MYF(0));
  return true;
}
/* FLUSH TABLES 对 DD / I_S / P_S 表无效，故加 IGNORE_FLUSH */
if (table_list->is_system_view || belongs_to_dd_table(table_list) || belongs_to_p_s(table_list))
  flags |= MYSQL_OPEN_IGNORE_FLUSH;

key_length = get_table_def_key(table_list, &key);   // "db\0table\0" 形式的缓存 key
```

**② LOCK TABLES 模式：在已打开表里找"最佳匹配"**

```cpp
if (thd->locked_tables_mode && !(flags & MYSQL_OPEN_GET_NEW_TABLE) && ...) {
  int best_distance = INT_MIN;
  for (table = thd->open_tables; table; table = table->next) {
    if (key 匹配 && alias 匹配(大小写无关) && table->query_id != thd->query_id) {
      int distance = ((int)table->reginfo.lock_type - (int)table_list->lock_descriptor().type);
      /* distance < 0 没找到合适的；== 0 正好；> 0 锁比需要的更强 */
      if ((best_distance < 0 && distance > best_distance) ||
          (distance >= 0 && distance < best_distance)) {
        best_distance = distance; best_table = table;
        if (best_distance == 0) break;      // 完美匹配
      }
    }
  }
  if (best_table) { ...; goto reset; }       // 复用已锁的 TABLE，不走后面的 MDL 流程
}
```

★ `distance = 已有锁类型 − 请求锁类型` 这个"最佳匹配"算法：优先精确匹配，其次选**比请求更强**的最接近的锁。若一个都没找到（distance 恒 < 0），就交给后续报错——注释明说这是为了"能给出'锁模式不对'的错误信息"，而不是简单返回"表没锁"。

**③ MDL：获取与 S→X 升级**

```cpp
if (open_table_get_mdl_lock(thd, ot_ctx, table_list, flags, &mdl_ticket) ||
    mdl_ticket == nullptr) return true;
```

`open_table_get_mdl_lock`（:2598）内部还包含**对全局读锁（GRL）的防护**（`set_force_dml_deadlock_weight` 等，见 :3085 附近，配合 `ot_ctx->set_has_protection_against_grl()`）。

★ **CREATE TABLE 场景的锁升级**（呼应上文 CREATE 段）：

```cpp
if (table_list->open_strategy == Table_ref::OPEN_IF_EXISTS ||
    table_list->open_strategy == Table_ref::OPEN_FOR_CREATE) {
  if (check_if_table_exists(thd, table_list, &exists)) return true;   // 查 DD
  if (!exists) {
    if (table_list->open_strategy == Table_ref::OPEN_FOR_CREATE && ...) {
      MDL_deadlock_handler mdl_deadlock_handler(ot_ctx);
      thd->push_internal_handler(&mdl_deadlock_handler);
      bool wait_result = thd->mdl_context.upgrade_shared_lock(
          table_list->mdl_request.ticket, MDL_EXCLUSIVE, thd->variables.lock_wait_timeout);
      thd->pop_internal_handler();
    }
    return false;                                                      // 表不存在，无需继续开
  }
}
```

即：先拿 S 锁 → 查 DD 发现不存在 → **升级为 X 锁**再建表。升级期间挂 `MDL_deadlock_handler`，把死锁转成 `Open_table_context` 的退避动作。

**④ Table cache 查找与 FLUSH 失效判定**

```cpp
retry_share : {
  Table_cache *tc = table_cache_manager.get_cache(thd);   // 按 thread id 选分片
  tc->lock();
  if (!table_list->is_view())
    table = tc->get_table(thd, key, key_length, &share);  // 取空闲 TABLE 或至少拿到 SHARE

  if (table) {
    if (!(flags & MYSQL_OPEN_IGNORE_FLUSH)) {
      /* 比较 TABLE_SHARE::version 与全局 refresh_version */
      ...
    }
  }
}
```

`get_table` 路径：锁分片 mutex → 查 `m_cache` 哈希（key = `db+table`）→ 命中则从该 element 的 `free_tables` 弹一个 TABLE、置 `in_use`；未命中则至少返回 `TABLE_SHARE*`，供后面实例化。

★ FLUSH 失效的判定靠**版本比较**：`TABLE_SHARE::version` vs 全局 `refresh_version`。源码注释（:3163）解释了为什么只持分片锁就够安全——修改 `version` 必须同时持有 `LOCK_open`（TDC 锁）和所有分片锁，所以持有一个分片锁时读到的版本是稳定的。

**⑤ 实例化：从 SHARE 造 TABLE**

```cpp
dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
const dd::Table *table_def = nullptr;
thd->dd_client()->acquire(share->db.str, share->table_name.str, &table_def);  // ① 取 DD 定义

if (table_def && table_def->hidden() == dd::Abstract_table::HT_HIDDEN_SE) {
  my_error(ER_NO_SUCH_TABLE, ...); goto err_lock;                    // ② SE 隐藏表对外不可见
}

table = (TABLE *)my_malloc(key_memory_TABLE, sizeof(*table), MYF(MY_WME));
error = open_table_from_share(thd, share, alias,
                              HA_OPEN_KEYFILE | HA_OPEN_RNDFILE | HA_GET_INDEX | HA_TRY_READ_ONLY,
                              EXTRA_RECORD, thd->open_options, table, false, table_def);

if (error) {
  destroy(table); my_free(table);
  if (error == 7)       ot_ctx->request_backoff_action(Open_table_context::OT_DISCOVER, table_list);
  else if (error == 8)  ot_ctx->request_backoff_action(Open_table_context::OT_FIX_ROW_TYPE, table_list);
  else if (share->crashed) ot_ctx->request_backoff_action(Open_table_context::OT_REPAIR, table_list);
  goto err_lock;
} else if (share->crashed) {
  /* 崩溃表只对 ALTER / REPAIR / CHECK / SHOW CREATE 放行，其余报 ER_CRASHED_ON_USAGE */
}

if (open_table_entry_fini(thd, share, table_def, table)) { ... }      // ③ 触发器等收尾
/* Add new TABLE object to table cache for this connection. */
Table_cache *tc = table_cache_manager.get_cache(thd);                 // ④ 放回本线程分片
```

★ **三种 backoff action** 是 `Open_table_context` 的重试机制：开表失败不直接报错，而是"退避并采取动作"——

| 错误码 / 状态 | 动作 | 含义 |
|---|---|---|
| `error == 7` | `OT_DISCOVER` | 触发**表发现**（如 `.frm`/字典缺失，让引擎重新导入） |
| `error == 8` | `OT_FIX_ROW_TYPE` | 修**行类型**（老格式行需要转换） |
| `share->crashed` | `OT_REPAIR` | 尝试**自动修复**（`auto_repair_table`） |

`open_table_from_share` 内部会调 `handler::ha_open` → `ha_innobase::open`，此时 InnoDB 侧加载 `dict_table_t`（即 SHARE 与引擎字典的对接点）。

**`Table_cache` 分片结构**（`table_cache.h`）：

| 类 | 职责 |
|---|---|
| `Table_cache_manager`（:170） | 单例；`MAX_TABLE_CACHES=64`、`DEFAULT=16`；按 thread id 路由 |
| `Table_cache`（:68） | 每分片：`m_unused_tables` 链 + `m_cache`（db+table → element）哈希表 + 自己的 mutex |
| `Table_cache_element`（:230） | 每（分片，表）一个：`used_tables`/`free_tables` 两条 `I_P_List` + `share` 反向指针 |

★ **设计动机**（`table_cache.h:53` 注释原文）：多数语句**不需要**去中心 definition cache 拿 TABLE、**不需要**锁全局 `LOCK_open`——只锁自己 thread id 对应的分片即可。这与 InnoDB Fil 层的 68 分片（见 `../innodb/physical/tablespace.md`）是同一思路在两层的各自落地。

`TABLE_SHARE::cache_element[]` 记录"该 SHARE 在每个分片里的 element"，跨分片遍历一个表的所有 TABLE 实例（FLUSH、DDL 排他）用 `Table_cache_iterator`。

### 开表的三个疑难机制

#### 一、Open_table_context：开表失败不是立即报错，而是"退避 + 动作 + 重试"

开表失败（表定义损坏、行格式过旧、MDL 冲突）时，`open_table` 不直接返回错误，而是调 `ot_ctx->request_backoff_action(action, table_list)` 记下动作，由外层 `open_tables` 调 `recover_from_failed_open()` 执行后重试。

**五种动作**（`sql_base.h:498`）：

| 动作 | 触发 | 含义 |
|---|---|---|
| `OT_NO_ACTION` | — | 无可恢复动作 |
| `OT_BACKOFF_AND_RETRY` | MDL 死锁/超时 | 释放已拿的锁、等待后重试（需 `can_back_off()`） |
| `OT_REOPEN_TABLES` | 表版本失效（FLUSH 后） | 重开所有表 |
| `OT_DISCOVER` | `open_table_from_share` 返回 7 | 让存储引擎**重新发现**表定义 |
| `OT_REPAIR` | `share->crashed` | `auto_repair_table` 自动修复 |
| `OT_FIX_ROW_TYPE` | 返回 8 | 修老的行类型（要改 DD，需提交） |

★ **能不能退避，取决于建 context 时有没有锁**：

```cpp
bool can_back_off() const { return !m_has_locks; }
/* Whether we had any locks when this context was created.
   If we did, they are from the previous statement of a transaction,
   and we can't safely do back-off (and release them). */
```

即：**事务中前序语句持有的锁不能被退避释放**——只有"这条语句自己刚拿的锁"才能退。这是理解"为什么多语句事务里某些开表失败不可恢复"的关键。

**`recover_from_failed_open()` 的执行**（`sql_base.cc:4205`）：

```cpp
if ((m_action == OT_REPAIR || m_action == OT_DISCOVER || m_action == OT_FIX_ROW_TYPE) &&
    (m_flags & MYSQL_OPEN_FAIL_ON_MDL_CONFLICT)) {
  my_error(ER_WARN_I_S_SKIPPED_TABLE, ...);      // ① I_S 查询：跳过坏表并警告
  return true;
}

MDL_deadlock_discovery_repair_handler handler;   // ② DEADLOCK 时标记事务回滚
m_thd->push_internal_handler(&handler);

switch (m_action) {
  case OT_BACKOFF_AND_RETRY:
  case OT_REOPEN_TABLES:
    break;                                        // ③ 不做事，交给调用方重试

  case OT_DISCOVER:
    lock_table_names(m_thd, m_failed_table, nullptr, get_timeout(), 0);  // 拿 X 名锁
    tdc_remove_table(m_thd, TDC_RT_REMOVE_ALL, ...);                     // 清掉旧 SHARE
    ha_create_table_from_engine(m_thd, ...);                             // 引擎重新发现
    m_thd->mdl_context.rollback_to_savepoint(start_of_statement_svp());  // ④ 只放到语句开始
    break;

  case OT_REPAIR:
    lock_table_names(...); tdc_remove_table(...);
    auto_repair_table(m_thd, m_failed_table);
    m_thd->mdl_context.rollback_to_savepoint(start_of_statement_svp());
    break;

  case OT_FIX_ROW_TYPE:
    assert(!m_thd->mdl_context.has_locks());       // ⑤ 必须无 MDL
    if (m_thd->in_active_multi_stmt_transaction()) {
      my_error(ER_LOCK_OR_ACTIVE_TRANSACTION, MYF(0));   // 活跃事务中不允许
      result = true; break;
    }
    ...
}
```

① ★ **I_S 查询遇到坏表是"跳过 + 警告"**（`ER_WARN_I_S_SKIPPED_TABLE`），而不是让整个 `information_schema` 查询失败——由 `MYSQL_OPEN_FAIL_ON_MDL_CONFLICT` 标志区分。

④ ★ `rollback_to_savepoint(start_of_statement_svp())` 是精髓：为"发现/修复"临时拿的 X 名锁**用完就放**，但**保留事务中前序语句的锁**。这正是 `m_has_locks` 存在的意义。

⑤ `OT_FIX_ROW_TYPE` 要**提交 DD 修改**，所以绝不能在活跃多语句事务里做——连 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 都不能隐式结束，只能报错 `ER_LOCK_OR_ACTIVE_TRANSACTION`。

#### 二、refresh_version 与 SHARE 淘汰：一套"版本号 + LRU"的惰性失效

先看**锁协议**（`sql_base.cc:238` 的九条注释，节选关键的）：

```
3) LOCK_open 保护 share 的初始化及其成员；但 get_table_share 读表定义时会
   **临时释放** LOCK_open 以提高并发，用 COND_open 条件变量同步。
5) oldest_unused_share / end_of_unused_share / share->next / prev
   只能在持 LOCK_open 时读写（即 TDC 的 LRU 链）。
7) refresh_version 只能在持 LOCK_open **且所有 table cache mutex** 时更新。
8) share->version 初始化时持 LOCK_open；更新时还要持所有 table cache mutex。
   因此：持有任一 mutex 时，通过引用找到的 share 其 version 不会变。
```

第 8 条正是 `open_table` 里"只持分片锁就能安全比较 `share->version` 与 `refresh_version`"的依据。

**FLUSH TABLES 的做法**（`:1178`）：

```cpp
/* incrementing of refresh_version and removal of unused tables and
   shares from TDC happens atomically under protection of LOCK_open ... */
refresh_version++;                                             // ① 推进全局版本

/* Get rid of all unused TABLE and TABLE_SHARE instances. By doing
   this we automatically close all tables which were marked as "old". */
while (oldest_unused_share->next)                               // ② 清光所有未使用的 SHARE
  table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
```

★ 注释（`:3357`）点明：**`refresh_version` 只在 FLUSH TABLES 时改变**。所以这套机制是"FLUSH 推进版本号 + 清空空闲 SHARE，正在使用的 SHARE 靠版本比较被判定为旧、等引用归零后自然淘汰"——**不是遍历清空，而是自然淘汰**。

**并发打开同一个 share**（`COND_open`，`:270`）：

```cpp
/* If a thread calls get_table_share, it releases the LOCK_open mutex while
   reading the definition from file. If a different thread calls get_table_share
   for the same share at this point in time, it will find the share in the TDC,
   but with the m_open_in_progress flag set to true. This will make the
   (second) thread wait for the COND_open condition ... */
```

即：线程 A 持 LOCK_open 建 SHARE → 临时释放锁去读 DD/文件 → 线程 B 进来发现 `m_open_in_progress=true` → 等 `COND_open` → A 完成后广播唤醒。**避免了"读文件时长期占着全局锁"**。

#### 三、TDC 内部结构：一个 hash + 一条 LRU，8.0 无分片

```cpp
using Table_definition_cache =
    malloc_unordered_map<std::string,
                         std::unique_ptr<TABLE_SHARE, Table_share_deleter>>;
Table_definition_cache *table_def_cache;                    // 全局唯一
static TABLE_SHARE *oldest_unused_share, end_of_unused_share;  // LRU 双向链表（哨兵）
```

- **hash**：key 是 `table_cache_key`（`db\0table\0`），value 是 `unique_ptr<TABLE_SHARE>`（出表自动析构）；
- **LRU**：`oldest_unused_share → ... → end_of_unused_share` 双向链，**只串未被使用的 SHARE**（有引用就不在链上）；
- **淘汰**（`:566`、`:1006`）：

```cpp
while (table_def_cache->size() > table_def_size && oldest_unused_share->next)
  table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
```

超过 `table_definition_cache`（变量 `table_def_size`）就从 **LRU 最旧端**淘汰，直到低于阈值。

★ **纠正一个常见误解**：**8.0 的 TDC 没有分片**。5.6/5.7 曾有 `table_definition_cache_instances` 把 TDC 分片，8.0.39 里**该变量已不存在**（只剩 `table_definition_cache` 一个大小参数），TDC 回归为单个 `table_def_cache` + 全局 `LOCK_open`。**分片的只有 TABLE 实例缓存**（`table_open_cache_instances`，16/64 个 `Table_cache`）。

所以准确的分工是：

| 缓存 | 结构 | 并发保护 | 分片？ |
|---|---|---|---|
| TDC（`TABLE_SHARE`） | 一个 hash + 一条 unused LRU | 全局 `LOCK_open`（+ `COND_open` 应对长耗时打开） | **否** |
| table open cache（`TABLE`） | N 个 `Table_cache`（各含 hash + unused 链） | 每分片一把 mutex | **是**（默认 16） |

这也解释了为什么 8.0 里"表定义非常多"仍可能成为瓶颈（TDC 是全局锁），而"连接非常多"则因为 TABLE 实例缓存分片而扩展性良好。

### 2. 锁表：lock_tables

入口 `lock_tables`（`sql_base.cc:6824`），两段前置逻辑值得单独说。

**① "无表也要锁"的早退路径**

```cpp
if (!tables && !thd->lex->requires_prelocking()) {
  /* Even though we are not really locking any tables mark this statement as
     one that has locked its tables, so we won't call this function second time
     for the same execution of the same statement. */
  thd->lex->lock_tables_state = Query_tables_list::LTS_LOCKED;
  int ret = thd->decide_logging_format(tables);
  return ret;
}
```

即使没有表要锁（如 `SELECT 1`、`SHOW`），也要把 `lock_tables_state` 置为 `LTS_LOCKED`——**这是个一次性幂等标记**，防止同一语句重复进 `lock_tables`。

**② LOCK TABLES 模式下不要重复加锁**

```cpp
if (!thd->locked_tables_mode) {
  assert(thd->lock == nullptr);      // "You must lock everything at once"
  ...
```

注释（:6853）解释了为什么用 `locked_tables_mode` 而不是 `thd->lock` 判断：存储函数里 `drop table t3; create temporary t3; insert into t3;` 之后 `thd->lock` 可能为 0，但 `locked_tables_mode` 仍开着——此时再去锁临时表会导致**内存泄漏**。

**③ ★ prelocking 阶段跳过临时表的 store_lock / external_lock**

```cpp
for (table = tables; table; table = table->next_global) {
  if (!table->is_placeholder() &&
      /* Do not call handler::store_lock()/external_lock() for temporary
         tables from prelocking list. */
      ...
```

源码用了近 30 行注释（:6872-6898）解释这个跳过，核心是：

> "InnoDB considers this breaking of promise about operation type and fails on assertion."

场景：语句用到两个存储例程，一个会创建并**修改**临时表、另一个只**读**它。prelocking 阶段的 `store_lock()/external_lock()` 只告知引擎"读"，修改信息要等到例程执行时的 `handler::start_stmt()` 才告知。InnoDB 认为这是**违背了"操作类型承诺"**，直接断言失败。解决办法是统一不在 prelocking 阶段对这类临时表调用这两个接口。

**④ 主链路**

```
lock_tables (sql_base.cc)
  → mysql_lock_tables
    → lock_tables_check          （过滤 log 表等）
    → get_lock_data              （收集每表 handler 的 THR_LOCK_DATA）
    │    └─ table->file->store_lock   （ha_innobase::store_lock：按请求类型设置）
    → lock_external
    │    └─ handler::ha_external_lock → ha_innobase::external_lock
    │         （设 m_prebuilt->select_lock_type，供 row_search_mvcc 使用）
    → thr_multi_lock             （server 层表锁）
```

★ **InnoDB 不走 server 层表锁**：`ha_innobase::lock_count()` 恒返回 0，`get_lock_data` 收集不到 `THR_LOCK_DATA`，`thr_multi_lock` 被跳过——并发控制交给 InnoDB 行锁 + server 层 MDL。`external_lock` 的真正作用是把语句锁意图（`TL_READ`/`TL_WRITE`）翻译成 `m_prebuilt->select_lock_type`（`LOCK_NONE`/`LOCK_S`/`LOCK_X`），后续 `row_search_mvcc` 依此加锁。

### 3. CREATE TABLE

主入口 `mysql_create_table`（`sql_table.cc:10045`）。按"准备 → 加锁 → 建表 → 善后 → 记日志"五段逐段解析。

**第一段：引擎选择与原子 DDL 前置检查**

```cpp
handlerton *actual_hton = get_viable_handlerton_for_create(
    thd, create_table->table_name, *create_info);      // ① 决定引擎
if (actual_hton == nullptr) return true;
create_info->db_type = actual_hton;

if (create_info->m_transactional_ddl) {                // ② START TRANSACTION 下建表
  if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
    my_error(ER_NOT_ALLOWED_WITH_START_TRANSACTION, MYF(0),
             "with engine that does not support atomic DDL.");
    result = true; goto end;
  }
  if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {  // 临时表也不行
    my_error(ER_NOT_ALLOWED_WITH_START_TRANSACTION, MYF(0), "to create temporary tables.");
    result = true; goto end;
  }
}
```

① `get_viable_handlerton_for_create` 决定用哪个引擎（显式 `ENGINE=` 或 `default_storage_engine`，不可用时按 `NO_ENGINE_SUBSTITUTION` 决定报错还是替换）。

② ★ **"在显式事务里建表"的两个禁令**：引擎不支持原子 DDL（无法把 DDL 纳入事务回滚）、或建的是临时表——都直接报错。这解释了"为什么 `BEGIN; CREATE TEMPORARY TABLE` 会失败"。

**第二段：加锁（MDL + 外键相关表）**

```cpp
if (open_tables(thd, &thd->lex->query_tables, &not_used, 0) ||
    thd->decide_logging_format(thd->lex->query_tables)) { ... }
```

★ 注释（:10084）说明了 MDL 的获取方式：**先取 S 锁判断表是否存在，不存在再升级为 X 锁**——避免一上来就 X 锁阻塞所有读。

```cpp
/* 新建表若带外键，要把"父表 / 子表 / FK 名"都 X 锁住 */
if (!(create_table->table || create_table->is_view()) &&          // 表不存在
    !(create_info->options & HA_LEX_CREATE_TMP_TABLE) &&          // 非临时表
    (create_info->db_type->flags & HTON_SUPPORTS_FOREIGN_KEYS)) { // 引擎支持 FK
  if (collect_fk_parents_for_new_fks(thd, ..., MDL_EXCLUSIVE, ..., &fk_invalidator) ||
      collect_fk_children(thd, ..., MDL_EXCLUSIVE, &mdl_requests) ||
      collect_fk_names_for_new_fks(thd, ..., 0 /* No pre-existing FKs */, &mdl_requests) ||
      thd->mdl_context.acquire_locks(&mdl_requests, thd->variables.lock_wait_timeout)) {
    result = true; goto end;
  }
}
```

★ 建一个"带外键的新表"要锁三类东西：**新 FK 指向的父表**（X）、**当前以该表名为父表的子表**（X，防止建表瞬间有孤儿）、**FK 约束名**（防重名）。四个跳过条件：表已存在/是视图、临时表、引擎不支持 FK。

**第三段：建表前的语义加工**

```cpp
if (prepare_check_constraints_for_create(thd, ..., alter_info)) { ... }   // CHECK 约束

if (!thd->variables.explicit_defaults_for_timestamp)
  promote_first_timestamp_column(&alter_info->create_list);               // ① timestamp 提升

if (is_generate_invisible_primary_key_mode_active(thd) &&
    is_candidate_table_for_invisible_primary_key_generation(create_info, alter_info)) {
  if (validate_and_generate_invisible_primary_key(thd, alter_info)) { ... }
  is_pk_generated = true;                                                  // ② GIPK
}

result = mysql_create_table_no_lock(thd, create_table->db, ..., &is_trans, &post_ddl_ht);
```

① `explicit_defaults_for_timestamp=OFF`（非默认）时，**第一个 TIMESTAMP 列被自动提升**为 `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`。

② ★ **GIPK**（Generated Invisible Primary Key）：开了 `sql_generate_invisible_primary_key` 且表无显式主键时，自动生成一个隐藏自增主键列。它本质是**隐藏生成列**，与功能索引/多值索引同属一个机制家族（见 [`../feat/generated_columns.md`](../feat/generated_columns.md)）。`is_pk_generated` 这个标志会在下面的 binlog 段引发特殊处理。

`mysql_create_table_no_lock`（:9135）才是真正干活：建 `TABLE_SHARE` → `ha_create_table` → `handler::ha_create` → `ha_innobase::create`（建 `dict_table_t` + 表空间 + 聚簇索引 root）→ 写 DD。8.0 的 **原子 DDL**：DD 修改、引擎建表、binlog 三者在同一事务语义下提交。

**第四段：binlog —— GIPK 的特殊处理**

```cpp
if (!result) {
  if (create_info->options & HA_LEX_CREATE_TMP_TABLE)
    thd->get_transaction()->mark_created_temp_table(Transaction_ctx::STMT);

  if (!thd->is_current_stmt_binlog_format_row() ||
      (thd->is_current_stmt_binlog_format_row() &&
       !(create_info->options & HA_LEX_CREATE_TMP_TABLE))) {
    thd->add_to_binlog_accessed_dbs(create_table->db);

    if ((create_table->table == nullptr && !create_table->is_view()) && is_pk_generated) {
      /* GIPK：不能记原语句，要记"实际做了什么" */
      ...open_table / open_temporary_table...
      result = store_create_info(thd, create_table, &query, create_info, true, false);
      result = write_bin_log(thd, true, query.ptr(), query.length(), is_trans);
      tdc_remove_table(thd, TDC_RT_REMOVE_NOT_OWN, ...);   // 清掉未提交的 share
      close_thread_table(thd, &thd->open_tables);
    } else {
      result = write_bin_log(thd, true, thd->query().str, thd->query().length(), is_trans);
    }
  }
}
```

★★ **这里有个非常值得注意的设计**：如果主键是 GIPK **生成**的，binlog **不记录用户原始语句**，而是用 `store_create_info()` 重新生成一条**把隐藏主键显式写出来**的 CREATE TABLE 语句再记。源码注释（:10200）解释得清楚：

> "we don't write to binary log value of `@@sql_generate_invisible_primary_key` variable, but rely on logging what really has been done instead."

即：备库的 `sql_generate_invisible_primary_key` 可能与主库不同，若记原语句，备库重放可能**不生成**主键 → 主从表结构不一致。所以必须记"实际结果"。这也是少数几处 MySQL 记 binlog 时用"重造语句"而非原语句的场景。

另外两条规则：建表失败不记 binlog；**RBR + 临时表不记**（临时表只在本连接有意义）。

**第五段：善后（外键与视图）**

```cpp
if (!(create_table->table || create_table->is_view()) && !result &&
    (create_info->db_type->flags & HTON_SUPPORTS_FOREIGN_KEYS)) {
  ...acquire new_table from DD...
  dd::warn_on_deprecated_prefix_key_partition(...);          // ① 弃用告警
  assert(is_trans);
  if (adjust_fk_children_after_parent_def_change(...) ||     // ② 更新子表 FK 定义
      adjust_fk_parents(thd, ..., true, nullptr))            // ③ 更新父表引用计数
    result = true;
}
if (!result) {
  Uncommitted_tables_guard uncommitted_tables(thd);
  result = update_referencing_views_metadata(thd, create_table, !is_trans, &uncommitted_tables);
}
```

新表成为别人的父表后，要回写**子表的 FK 定义**（②）与**父表的引用关系**（③）；最后还要**更新引用该表的视图元数据**（`update_referencing_views_metadata`）——这就是为什么建表可能"莫名其妙"地慢：视图依赖链要一并刷新。

要点小结：`HA_CREATE_INFO` 承载全部 CREATE 选项（含 `m_implicit_tablespace_autoextend_size`）；整条链路上有**四类锁**（目标表 MDL X、FK 父表 X、FK 子表 X、FK 名）与**两次元数据回写**（外键 + 视图）。

#### ★ 建表的 DD 缓存边界（月报 2022/01 强调、易忽略）

`CREATE TABLE` 涉及的 dd::Table 对象，其**缓存生命周期**有精确的边界：

```cpp
// ① 双重保存
Storage_adapter::store()              // 落盘到 InnoDB（DD 表）
register_uncommitted_object()         // ★ clone 进本线程 uncommitted registry

// ② 提交时（trans_commit_implicit → commit_modified_objects）
invalidate()                          // 防 Shared_dictionary_cache 残留同名表
erase(m_registry_uncommitted)
erase(m_registry_dropped)
```

三个要点：

1. ★ **普通用户表的 `dd::Table` 不进 `Shared_dictionary_cache`**——只落盘 + 走本线程 uncommitted registry；只有 **bootstrap 阶段**（`is_dd_system_thread` 且 stage < FINISHED）才会 put 进共享缓存。所以"建一张新表"不会污染别的连接可见的共享缓存。
2. ★ **tablespace 例外**：建表创建 tablespace 时，`fil_space_create` 会把表空间信息留在**全局 `fil_system`（Fil_shard）**——这是"会残留的内存缓存"的例外（表对象不残留、表空间对象残留）。
3. **`acquire_for_modification()` 是双重 clone**：register 时 clone 一次进 uncommitted，取用时再 clone 一次用于建 TABLE_SHARE。

#### ★ `set_missed` 防击穿（共享缓存并发 miss 控制）

多线程同时 cache miss 同一个对象时，`Shared_multi_map` 用 `set_missed()` 防"读穿透风暴"：第一个 miss 的线程标记 `missed` 并去读存储引擎，其他线程看到 `missed` 状态**等待**（而不是各自都去读盘），第一个完成后回填并唤醒。这是两级缓存体系里并发正确性的关键一环——否则"冷表第一次被 N 个连接同时打开"会变成 N 次重复的 DD 读 + N 份重复构建。

#### `m_needs_reopen`：TABLE 的"需要重开"标记

`TABLE::m_needs_reopen` 记录"这个 TABLE 实例与当前定义不一致，需要重新打开"——典型触发是 `FLUSH TABLES` / 其他连接 DDL 之后，本线程持有的 TABLE 被标记需重开。语义：**TABLE 实例不主动自我失效，靠标记 + 下次使用前检查**（与 `refresh_version` 的 SHARE 淘汰是同一体系的两种力度：SHARE 级淘汰 vs TABLE 级标记）。

### 4. ALTER TABLE：算法决策的完整判据

> **边界**：本篇只讲 **SQL 层如何决策**（问引擎、该降级还是报错）。三种算法在 InnoDB 内部的实现见 [`../innodb/ddl.md`](../innodb/ddl.md) 的对应章节（该文档已完整覆盖，不必在本篇重复）：
>
> - **COPY 的重建** → 「COPY DDL 完整流程」+「`copy_data_between_tables` 源码逐段剖析」
> - **INPLACE 的 row log** → 「online DDL 与 row log」（`row_log_t` 结构、临时文件命名、mrec 记录格式、三种操作各记什么、temp 紧凑格式、与 DDL log 的区别、apply 两次的时机）
> - **8.0.29 instant 行版本** → 「Instant DDL 深度剖析」（三个 version 的辨析、V1(n_fields)→V2(row_version) 两种记录格式、字节级布局、核心函数逐行解析、REDUNDANT 的处理）
>
> 另：「online DDL 的两种形态」「MDL 锁的三段式（online DDL 的骨架）」也在该文档中，是理解"为什么 INPLACE 仍要在开始和结束时拿 MDL"的前提。

主入口 `mysql_alter_table`（`sql_table.cc:16295`）。算法决策分**三轮**。

#### 第一轮：语法层互斥检查

```cpp
if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
    alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_DEFAULT) {
  my_error(ER_WRONG_USAGE, MYF(0), "ALGORITHM=INSTANT", "LOCK=NONE/SHARED/EXCLUSIVE");
  return true;
}
```

★ `ALGORITHM=INSTANT` **不能与 `LOCK=` 同时指定**——INSTANT 根本不加锁，指定 LOCK 无意义，所以直接报 `ER_WRONG_USAGE`。

#### 第二轮：明显不可能 inplace 的情况，直接定为 COPY

```cpp
if ((alter_info->requested_algorithm != Alter_info::ALTER_TABLE_ALGORITHM_INPLACE &&
     alter_info->requested_algorithm != Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) ||
    is_inplace_alter_impossible(table, create_info, alter_info) ||
    (partition_changed && !(table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION) &&
     !new_part_info)) {
  /* 用户显式指定了 INPLACE/INSTANT 就报错，否则静默改为 COPY */
  if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), "ALGORITHM=INPLACE", "ALGORITHM=COPY");
    return true;
  }
  if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), "ALGORITHM=INSTANT", "ALGORITHM=COPY");
    return true;
  }
  alter_info->requested_algorithm = Alter_info::ALTER_TABLE_ALGORITHM_COPY;
}
```

`is_inplace_alter_impossible()` 判断"这个改动本质上不可能原地做"（如改引擎、某些分区变更）。

#### 第三轮：问引擎，按返回值决定"降级 / 报错 / 接受"

```cpp
// Ask storage engine whether to use copy or in-place
enum_alter_inplace_result inplace_supported =
    table->file->check_if_supported_inplace_alter(altered_table, &ha_alter_info);
```

**决策权在引擎**：SQL 层把 `Alter_inplace_info`（含要做的全部改动）交给 handler，引擎返回六档结果之一，SQL 层据此 switch：

| 引擎返回 | 含义 | SQL 层行为 |
|---|---|---|
| `HA_ALTER_INPLACE_INSTANT` | 只改元数据，秒完成 | 接受（break） |
| `HA_ALTER_INPLACE_NO_LOCK[_AFTER_PREPARE]` | 全程/prepare 后可不加锁 | 接受 |
| `HA_ALTER_INPLACE_SHARED_LOCK[_AFTER_PREPARE]` | 需 S 锁 | 请求 `LOCK=NONE` → **报错**；否则接受 |
| `HA_ALTER_INPLACE_EXCLUSIVE_LOCK` | 需 X 锁 | 见下 |
| `HA_ALTER_INPLACE_NOT_SUPPORTED` | 不支持 inplace | 请求 INPLACE → 报错；请求 `LOCK=NONE` → 报错；否则 **降级 COPY** |
| `HA_ALTER_ERROR` | 出错 | 直接失败 |

★ **最关键的一段——"该降级还是该报错"，取决于用户是显式指定还是 DEFAULT**：

```cpp
case HA_ALTER_INPLACE_EXCLUSIVE_LOCK:
  // If SHARED lock and no particular algorithm was requested, use COPY.
  if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED &&
      alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_DEFAULT) {
    use_inplace = false;                      // ① 静默降级 COPY
  }
  // Otherwise, if weaker lock was requested, report error.
  else if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_NONE ||
           alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED) {
    ha_alter_info.report_unsupported_error("LOCK=NONE/SHARED", "LOCK=EXCLUSIVE");
    goto err_new_table_cleanup;               // ② 报错，不降级
  }
  break;
```

即：

- `ALTER TABLE t ...`（DEFAULT）+ 引擎要 X 锁 + 你请求了 `LOCK=SHARED` → **悄悄改成 COPY**；
- `ALTER TABLE t ..., ALGORITHM=INPLACE, LOCK=SHARED`（显式）+ 引擎要 X 锁 → **直接报错**，不偷偷降级。

★ **这个"显式就不降级"的原则贯穿整个决策**：用户没表态时 InnoDB 怎么快怎么来（可降级）；用户明确要求了就严格遵守，做不到就报错。这也解释了生产环境的一个常见困惑——"我加了 `ALGORITHM=INPLACE` 反而失败了，不加却能成功"。

**INSTANT 的特殊规则**（不降级，只报错）：

```cpp
if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
    inplace_supported != HA_ALTER_INPLACE_INSTANT &&
    inplace_supported != HA_ALTER_ERROR) {
  ha_alter_info.report_unsupported_error("ALGORITHM=INSTANT", "ALGORITHM=COPY/INPLACE");
  goto err_new_table_cleanup;
}
```

请求 INSTANT 而引擎说"只能 INPLACE/COPY" → **报错而非降级**。因为 INSTANT 的语义就是"不做任何数据搬迁"，退化为 INPLACE 违背用户意图。

**INSTANT 与 INPLACE 的关系**（源码注释 :17498）：

> "any instant operation is also in fact in-place operation. It is totally safe to execute operation using instant algorithm ... even if user explicitly asked for ALGORITHM=INPLACE."

即：**用户要 INPLACE、引擎返回 INSTANT 是完全可以接受的**（INSTANT 是 INPLACE 的特例且无副作用）；反过来若 instant 相对 inplace 有缺点（如 instant ADD 后每行多 1 字节版本头），**由引擎自己降级**返回 `HA_ALTER_INPLACE_NO_LOCK`，而不是让 SQL 层操心。

#### 决策之后

```cpp
if (use_inplace) {
  mysql_inplace_alter_table(thd, ..., inplace_supported, ...);   // INPLACE / INSTANT 路径
} else {
  /* COPY：建新表 → 拷数据 → rename 交换 */
}
```

另外还有个 **noop 优化**（:17447）：若解析后发现这个 ALTER 什么都没改，直接 `is_noop = true` 跳过——`ALTER TABLE t ALGORITHM=INSTANT` 在某些场景下"秒回"就是这个路径（或真正的 instant）。

### 5. RENAME TABLE

- `mysql_rename_tables`（`sql_rename.cc:156`：多表批量改名）
- `mysql_rename_table`（`sql_table.cc:10603`：单表），同时要处理**外键**引用（`old_fk_db/old_fk_name` 参数）——改名时引用它的子表外键需同步更新。

### 6. TRUNCATE TABLE

`sql_truncate.cc` 分两条路：

- **普通表**：`handler_truncate_base`（:168）→ 引擎 `ha_truncate`；InnoDB 实现是**删旧表重建新表**（而非逐行删），并分配新 table_id；
- **临时表**：`handler_truncate_temporary`（:260）走另一条路径（不涉及 DD 原子提交）。

注释里有个易漏点（:601）：`handler_truncate()` 可能更新了 DD 里的表定义，因此**必须把 TABLE_SHARE 从 TDC 移除**，且失败时也要移除。

### 7. DROP TABLE

主入口 `mysql_rm_table`（`sql_table.cc:1550`）。删表比建表更"危险"，所以前置检查更多，逐段看：

**第一段：三道禁令**

```cpp
// DROP table is not allowed in the XA_IDLE or XA_PREPARED transaction states.
if (thd->get_transaction()->xid_state()->check_xa_idle_or_prepared(true)) return true;

if (thd->decide_logging_format(tables)) return true;    // MIXED 模式下 DROP 临时表要先定格式

/* Disable drop of enabled log tables, must be done before name locking */
for (table = tables; table; table = table->next_local) {
  if (query_logger.check_if_log_table(table, true)) {   // ① 启用中的 general/slow log 表
    my_error(ER_BAD_LOG_STATEMENT, MYF(0), "DROP");
    return true;
  }
}
```

① **正在启用的日志表（general_log / slow_log）不能被 DROP**——且检查必须在**加锁之前**完成。

**第二段：加锁（分 LOCK TABLES 与非 LOCK TABLES 两条路）**

```cpp
if (!drop_temporary) {
  if (!thd->locked_tables_mode) {
    if (lock_table_names(thd, tables, nullptr, thd->variables.lock_wait_timeout, 0) ||
        lock_trigger_names(thd, tables))
      return true;
  } else {
    /* LOCK TABLES 模式：临时表不取 MDL */
    ...
    table->table = find_table_for_mdl_upgrade(thd, table->db, table->table_name, false);
    table->mdl_request.ticket = table->table->mdl_ticket;
    if (wait_while_table_is_used(thd, table->table, HA_EXTRA_FORCE_REOPEN)) return true;
    ...
    if (acquire_backup_lock) acquire_shared_backup_lock(thd, ...);
  }

  if (rm_table_do_discovery_and_lock_fk_tables(thd, tables)) return true;  // 外键相关表
  if (lock_check_constraint_names(thd, tables)) return true;               // CHECK 约束名
}
```

★ **临时表在 LOCK TABLES 下刻意不取 MDL**，源码注释（:1600）解释了原因：

> "There can't be metadata locks for temporary tables: they are local to the session. Later in this function we release the MDL lock only if `table->mdl_request.ticket` is not NULL. Thus here we ensure that we won't release the metadata lock on the base table locked with LOCK TABLES as a side effect of temporary table drop."

即：临时表与基表可能同名，若给临时表也分配 ticket，删临时表时会**误放基表的 MDL**。这里用 `assert(table->mdl_request.ticket == nullptr)` 把这个不变式钉死。

`wait_while_table_is_used` 会**等待所有正在使用该表的连接释放**（这就是为什么大表 DROP 有时"卡住"——它在等别的查询结束，而不是删得慢）。

**第三段：真正删除**

```cpp
std::vector<MDL_ticket *> safe_to_release_mdl;
{
  // This Auto_releaser needs to go out of scope before we start releasing
  // metadata locks below. Otherwise we end up having acquired objects for
  // which we no longer have any locks held.
  dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
  std::set<handlerton *> post_ddl_htons;
  Foreign_key_parents_invalidator fk_invalidator;

  thd->push_internal_handler(&err_handler);            // Drop_table_error_handler
  error = mysql_rm_table_no_locks(thd, tables, if_exists, drop_temporary, false,
                                  &not_used, &post_ddl_htons, &fk_invalidator,
                                  &safe_to_release_mdl);
  thd->pop_internal_handler();
}
```

★ 那段作用域注释值得注意：`Auto_releaser` 必须**在释放 MDL 之前出作用域**——否则会出现"持有已解锁对象"的窗口。这是 8.0 引入 DD 缓存后的典型约束。

`mysql_rm_table_no_locks`（:3119）是实际干活的函数：`ha_delete_table` → `ha_innobase::delete_table` → `row_drop_table_for_mysql`（InnoDB 侧释放段、删 `dict_table_t`、删 `.ibd`）→ 删 DD 记录。它也被 `DROP DATABASE` 等**已持锁**的场景复用（名字里的 `no_locks` 就是这个意思）。

`safe_to_release_mdl` 收集"可以安全释放"的 MDL ticket——非临时表的锁要等到事务提交后才放（保证原子 DDL 语义）。

要点：`drop_temporary` 参数区分临时表路径；整条链路上要处理**四类依赖**：触发器（`lock_trigger_names`）、外键表（`rm_table_do_discovery_and_lock_fk_tables`）、CHECK 约束名、以及引用它的视图。删表慢的常见原因不是删数据，而是 `wait_while_table_is_used` 在等并发查询退出。

### 8. FLUSH TABLES / 缓存失效

通过 `refresh_version` 推进实现：TABLE_SHARE 上记版本号，开表时发现版本落后则重新加载。DDL/FLUSH 推进全局版本号，让所有缓存中的 SHARE 逻辑失效——**不是遍历清空，而是让旧版本对象自然淘汰**。

---

## 列与行：本篇与 `record.md` 的边界

列和行是表的组成部分，所以**"元数据/属性"层面放在本篇**；**物理编码层面放在 [`../innodb/physical/record.md`](../innodb/physical/record.md)**。判断标准一句话：**问"表里有什么、每列什么属性"→ 本篇；问"这一行在页里的字节怎么排"→ record.md**。

| 问题 | 归宿 |
|---|---|
| 表有几列、列名/类型/字符集/是否可空、默认值 | **本篇**（SQL / DD 层元数据） |
| 自增列、生成列、多值列、溢出列、表达式默认值 | **本篇**（列属性） |
| 三个隐藏系统列（`DB_ROW_ID`/`DB_TRX_ID`/`DB_ROLL_PTR`） | [`record.md`](../innodb/physical/record.md)（行的物理构成） |
| 行头 5 字节怎么排、NULL 位图在哪、变长长度 1 还是 2 字节 | `record.md` |
| offsets 数组布局与四个标志位（NULL/EXTERNAL/DEFAULT/DROP） | `record.md` |
| instant ADD/DROP 后行怎么变、行版本状态机 | `record.md` |
| 行格式转换模板（MySQL ↔ InnoDB） | [`handler.md`](handler.md)（`row_prebuilt_t::mysql_template`） |

### 列属性一览

| 属性 | 说明 | 深入阅读 |
|---|---|---|
| 自增列 | 一个表只能一个，必须是某索引第一列；`TABLE_SHARE` 四处定位信息 | [`../feat/auto_increment.md`](../feat/auto_increment.md) |
| 生成列（VIRTUAL / STORED） | 值为表达式，虚拟列不落盘 | [`../feat/generated_columns.md`](../feat/generated_columns.md) |
| 多值列 | JSON 数组 + 多值索引 | [`query/07_optimize/13_functional_mv_index.md`](query/07_optimize/13_functional_mv_index.md) |
| 溢出列（BLOB/TEXT） | 超过阈值存到 off-page LOB 页 | [`../innodb/physical/lob.md`](../innodb/physical/lob.md) |
| **列默认值** | 字面量默认值；8.0.13+ 支持**表达式默认值** | 见下 |
| 三个隐藏系统列 | 恒存在，无 PK 时 `DB_ROW_ID` 作聚簇键 | `record.md` |

**列默认值与表达式默认值**：普通 `DEFAULT` 是字面量；8.0.13 起支持 `DEFAULT(expr)`（如 `DEFAULT (UUID())`、`DEFAULT (CURRENT_DATE + INTERVAL 1 YEAR)`）。它与生成列的区别常被混淆：

| | 表达式默认值 `DEFAULT(expr)` | 生成列 `AS (expr)` |
|---|---|---|
| 何时求值 | **插入时**求一次（未提供该列时） | 读时求值（VIRTUAL）或写入时（STORED） |
| 能否被覆盖 | 能（显式给值就不用默认值） | 不能（值恒等于表达式） |
| 能否索引 | 需显式建索引 | 虚拟列可建二级索引（物化在索引页） |
| 表达式是否能引用其他列 | 不能（只能是常量/函数/不引用本表列） | 能（可引用同行其他列） |

**行的存在形态（六种，分三组）**——常说"三种"是简化，完整清单如下：

| 组 | 形态 | 编码 / 结构 | 用在哪 |
|---|---|---|---|
| **server 层** | ① MySQL 行格式 | `TABLE->record[0]`（NULL 位图 + 列数据） | server 层读写行的标准形态 |
| | ② **key value 格式** | `key_copy` 产生的字节流 | 索引查找、排序、range 边界；★ **true VARCHAR 恒用 2 字节长度前缀**（行内表示是 1 或 2 字节，由 `mysql_length_bytes` 决定） |
| **InnoDB 逻辑层** | ③ `dtuple_t` / `dfield_t` | 逻辑元组（指针 + 长度 + 类型） | InnoDB 内部计算、比较、插入 |
| | ④ **temp 格式** | 紧凑行，**省略记录头**，只留变长长度 + NULL 标志 + 数据 | sort buffer、row log 的记录体（`rec_convert_dtuple_to_temp` / `rec_init_offsets_temp`） |
| | ⑤ **mrec** | `[op:1][old_pk:可选][extra_size:1-2][记录头][数据][vcol]` | row log 的序列化编码（online DDL 转发 DML），`rec_serialize_dtuple` |
| **InnoDB 物理层** | ⑥ `rec_t` | 页内字节，四种行格式 REDUNDANT / COMPACT / DYNAMIC / COMPRESSED | 最终落盘形态 —— [`record.md`](../innodb/physical/record.md) 的领域 |

主转换链：

```
MySQL 行格式  ←row_mysql_convert_row_to_innobase / row_mysql_store_row_in_mysql→  dtuple_t
dtuple_t      ←rec_convert_dtuple_to_rec / row_build→                              rec_t
MySQL 行格式  ←key_copy / key_restore→                                             key value
dtuple_t      ←rec_convert_dtuple_to_temp / rec_init_offsets_temp→                 temp
dtuple_t      ←rec_serialize_dtuple / row_log_table_apply_convert_mrec→            mrec
```

★ 记住 ④⑤⑥ 三种"非标准"形态的存在，才能理解为什么 `row_search_mvcc` 里会出现 `clust_templ_for_sec` 这类坐标换算（见 [`handler.md`](handler.md)）——它们服务于不同的优化目标：temp 为省空间、mrec 为 online DDL 转发、rec_t 为持久化。

转换所需的**模板**（每列在三种坐标下的偏移/长度/是否外置）缓存在 `row_prebuilt_t::mysql_template` 中，见 [`handler.md`](handler.md)。

## 表类型

| 类型 | 说明 | 归宿 |
|---|---|---|
| 普通表 | 默认 | 本篇 |
| 临时表 | `CREATE TEMPORARY TABLE`；`NON_TRANSACTIONAL_TMP_TABLE` 不进表锁计数 | [`query/runtime/07_temptable.md`](query/runtime/07_temptable.md) |
| 内部临时表 / intrinsic table | 优化器自建；InnoDB intrinsic 表可关闭锁与 redo | 同上 |
| 分区表 | `ha_innopart`，每分区一个 `dict_table_t` | [`../feat/partitioning.md`](../feat/partitioning.md) |
| 视图 / 派生表（placeholder） | 走 `resolve_placeholder_tables`，可 merge 或物化 | [`query/07_optimize/logical/02_subquery.md`](query/07_optimize/logical/02_subquery.md) |
| 系统表 / IS 表 | DD 视图化 | — |

---

## Misc

### 容易误解的概念

- **"open_table 后表存在哪"**：定义在 `TABLE_SHARE`（全局唯一），运行态在 `TABLE`（会话私有、缓存复用），语句通过 `Table_ref` 引用 `TABLE`。
- **两个缓存不要混**：TDC 管 SHARE、`table_open_cache` 管 TABLE 实例；调大后者却不调前者，仍会反复访问 DD。
- **分区表的 SHARE 只有一个**，但底层 `dict_table_t` 有 N 个（每分区一个）。
- **TRUNCATE 后 table_id 变化**：InnoDB 重建表，旧的事务必然失效。

### 深挖补遗（本篇已展开的三处"疑难"）

以下三条原为待办，现已在「开表的三个疑难机制」一节完整展开：

1. **`Open_table_context` 与重试逻辑**——五种 action、`can_back_off()` 的 `m_has_locks` 约束、`recover_from_failed_open()` 执行细节、I_S 坏表跳过；
2. **`refresh_version` 与 SHARE 淘汰时序**——九条锁协议、`FLUSH TABLES` 的版本推进 + 清空空闲 share、`COND_open` 并发打开；
3. **TDC 内部结构**——hash + unused LRU、按 `table_def_size` 淘汰，并**纠正**：8.0 已无 `table_definition_cache_instances` 分片。

**分区表 DDL 的逐分区遍历**已由 [`../feat/partitioning.md`](../feat/partitioning.md) 完整覆盖，本篇不重复：

- **对象模型**（`m_table_parts`）：`dict_table_t **m_table_parts`（每分区一个表对象，名 `db/tbl#P#p0`）+ `m_index_mapping[N × 索引数]`——O(N) 膨胀，这正是分区上限 8192 与"分区数上千后 DDL/开表急剧变慢"的根因；
- **打开表的遍历**（3.3）：`Ha_innopart_share::open_table_parts` → `for (dd_part : dd_table->leaf_partitions())` → `open_one_table_part` → `dd_open_table<dd::Partition>`；★ **不存在"只打开被裁剪到的分区"**（`lock_partitions` 位图不传到引擎，社区版全库无按需打开）；
- **DDL 的遍历**（3.10）：TRUNCATE PARTITION 的 `for (dd_part : leaf_partitions())` + `is_partition_used()` 跳过未用分区；ANALYZE 逐分区 `update_table_stats`；CHECK 逐分区 `ha_innobase::check()` + `check_misplaced_rows`；DISCARD/IMPORT 按 `read_partitions` 逐分区；
- **锁粒度**（3.12）：行锁/表锁粒度是 `dict_table_t`，每分区一张表 ⇒ **天然就是分区级表锁**。

## 参考

- [MySQL 中的 TABLE 对象 — martji](https://martji.github.io/2020/12/05/mysql/table-object/)
- [MySQL 中的元数据管理 — 谢榕彪](https://zhuanlan.zhihu.com/p/623872012)
- [MySQL 如何准备开启一个表 — Miao Zheyu](https://mzyee.github.io/posts/mysql/meta/)
- [【华为云】MySQL open_table 流程解析](https://bbs.huaweicloud.com/blogs/440753)
- [InnoDB：DDL（1）— Skywalker](https://zhuanlan.zhihu.com/p/446044092)
- [MySQL open table — 天将降大任于斯人也](https://blog.51cto.com/u_12497420/3357542)
- [MySQL 开表相关分析 — modb](https://www.modb.pro/db/1861292073970839552)
- *MySQL 8.0 Reference Manual → [CREATE TABLE / ALTER TABLE Statement](https://dev.mysql.com/doc/refman/8.0/en/sql-statements.html)*
