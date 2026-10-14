# INFORMATION_SCHEMA：跨 server 与引擎的虚拟表机制

> **边界**：本篇**横跨两层**——server 的 I_S 插件框架与调度（`sql/sql_show.cc`）+ InnoDB 那 26 张表的实现（`storage/innobase/handler/i_s.cc`）。以 `INNODB_*` 表族为主要样例，但讲的是**这套机制本身**。
> - **monitor counter 体系**（`INNODB_METRICS` 背后的计数器）见 [`../innodb/monitor.md`](innodb_monitor.md)。
> - **PFS 自身**见 [`../server/infra/pfs.md`](pfs/pfs.md)（本篇只在「四」里做机制对比）。
> - **server 层 status variable** 见 [`../server/infra/variables.md`](../../server/infra/variables.md)。
> - **运维视角（怎么查）** 见 [`../infra/observability/README.md`](README.md) 3.1——本篇讲**机制**，那里给观测入口与补遗：`INNODB_*` 的**第二个来源**（server 层 DD system view 5 张，本篇只覆盖引擎侧 26 张）、**instant DDL 的查询**（`INSTANT_COLS`）、以及 8.0 已移除的 `INNODB_LOCKS` / `INNODB_LOCK_WAITS`。

**一句话机制**：I_S 表**没有 handler、没有文件**——server 把插件声明的 `ST_FIELD_INFO` 列定义翻译成一张**真正的 HEAP 内存临时表**，调用插件的 `fill_table()` 一次性灌满，然后当普通表扫。

---

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [版本演进](#版本演进)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
  - [一、server 侧：框架与调度](#一server-侧框架与调度)
  - [二、引擎侧：InnoDB 的 26 张表](#二引擎侧innodb-的-26-张表)
    - [2.3 加锁与一致性](#23--加锁与一致性每张表口径都不同)
    - [2.4 trx0i_s：适配层与缓存层](#24-trx0i_s适配层与缓存层)
    - [2.5 p_s.cc：PFS 那条出口](#25-p_sccpfs-那条出口)
  - [三、权限：为什么是引擎手写 21 处](#三权限为什么是引擎手写-21-处)
  - [四、I_S / PFS / SHOW 三分天下](#四is--pfs--show-三分天下)
  - [五、版本机制：改列必须递增版本号](#五版本机制改列必须递增版本号)
- [已知限制与坑](#已知限制与坑)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`INFORMATION_SCHEMA` 是一个**虚拟库**：里面的表既没有存储引擎、也没有数据文件，每次查询现场生成。

#### ★ 但 I_S 里装的不是一种东西

"IFS 就是元数据"这个理解**只说对一半**——它实际装着两类完全不同的内容：

| | **元数据**（SQL 标准的原义） | **运行时状态**（MySQL / InnoDB 的扩展） |
|---|---|---|
| 是什么 | 数据库**对象**的信息：有哪些表、列、索引、权限、字符集、引擎 | 实例**此刻**的状态：有哪些事务、buffer pool 里装着哪些页、压缩统计、FTS cache |
| 8.0 的实现 | **DD 系统视图**（`sql/dd/info_schema/`，**不走** `ST_SCHEMA_TABLE`） | 原生表 + 插件表（**本篇重点**） |
| 举例 | `I_S.COLUMNS`、`TABLES`、`STATISTICS`、`*_PRIVILEGES`、`ENGINES` | `INNODB_TRX`、`INNODB_BUFFER_PAGE`、`INNODB_METRICS`、`PROCESSLIST` |
| 读到的会"不是同一时刻"吗 | 不会——元数据稳定，且有 MDL 保护 | ★ **会**——这正是 2.3 那一节存在的理由 |

**本篇主体（InnoDB 那 26 张）属于后者**，它们绝大多数是运行时状态而非元数据（26 张里只有 `INNODB_TABLES`/`TABLESTATS`/`INDEXES`/`COLUMNS`/`VIRTUAL`/`TABLESPACES` 这 6 张是元数据）。

这也就解释了为什么 I_S 会和 monitor counter、PFS 并列成"三条可观测性路径"（见 4.3）——**它的一大半根本不是元数据，而是状态观测**。而"元数据"那一半（8.0 的绝大多数 I_S 表：`COLUMNS` / `TABLES` / `STATISTICS` …）在 8.0 已整体搬到 **DD 系统视图**，与本篇讲的插件机制是两套东西（见 1.1）——**那一半见姊妹篇 [`../server/dd/system_views.md`](../../server/dd/system_views.md)**。

#### 两层的契约

| 层 | 提供什么 |
|---|---|
| **server** | I_S 插件框架（`ST_SCHEMA_TABLE` / `ST_FIELD_INFO`）、注册表、调度链、临时表物化、库级权限 |
| **引擎/插件** | 列定义 + `fill_table()` 回调（数据来源、加锁、逐表权限门槛） |

server 只给框架，**数据从哪来、加不加锁、要不要权限**全是插件自己的事。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | InnoDB 表名带 `SYS_` 前缀（`INNODB_SYS_TABLES`）；有独立的 `INNODB_LOCKS` / `INNODB_LOCK_WAITS` |
| **8.0** | ① `SYS_` 前缀去掉；② **绝大多数 I_S 表改为 DD 系统视图**（不再是 `ST_SCHEMA_TABLE` 机制，见 1.1）；③ InnoDB 字典类表改为扫 `mysql.*` DD 表；④ `INNODB_LOCKS`/`LOCK_WAITS` 被 PFS `data_locks` 取代；⑤ 大量 `SHOW` 被重写成 I_S 或 PFS 查询（见「四」） |

⚠️ 考古痕迹：InnoDB 的列定义数组仍叫 `innodb_sysindex_fields_info`（对应 `INNODB_INDEXES`，名字没跟改）；`trx0i_s` 的锁相关数据结构与注释也没清干净。

---

## 设计思想与理论基础

**① 虚拟表有两种做法，MySQL 两种都用了。**

| | I_S | PFS |
|---|---|---|
| 做法 | **物化到内存临时表**：`create_schema_table()` 造一张 HEAP 表，插件往里写行，server 当普通表扫 | **真 handler**：`ha_perfschema` 实现 `rnd_init`/`rnd_next`，server 眼里就是普通基表 |
| 插件类型 | `MYSQL_INFORMATION_SCHEMA_PLUGIN` | `MYSQL_STORAGE_ENGINE_PLUGIN` |

I_S 的做法意味着**引擎永远不实现 `rnd_next`**——它只是在往 server 造好的表（`table->file` 是临时表的 handler）里写行。

**② 成本放在读路径。** I_S 表全部查询时现场生成，写路径零代价。代价全转给读：可能全表物化、可能持全局 latch、可能开后台事务。这与 monitor counter（写路径累加、读几乎免费）是**相反的成本模型**。

**③ 权限责任被推给了插件。** server 明确声明 *"There are no per table rules for the information schema"*，库级 SELECT 无条件放行，因此**逐表的额外门槛只能由插件在自己的 `fill_table` 里手写**。这是本篇「三」的核心，也是理解引擎侧那 21 处 `check_global_access` 的关键——**不是引擎想重复造轮子，是接口如此**。

**④ `WHERE` 是"提示"不是"下推"。** 接口上 `fill_table` 拿到 `Item *cond`，但那只是给引擎的可选优化输入（注释举例：`WHERE TABLE_SCHEMA='test'` 可只开一个目录）。server 不保证过滤发生，实际是**先全量物化、再由上层 `FilterIterator` 过滤**。

---

## 核心实现

### 一、server 侧：框架与调度

#### 1.1 三种 I_S 表，别混为一谈

| 类型 | 数量 | 机制 |
|---|---|---|
| **原生表** | **13 张** | `sql_show.cc` 的 `schema_tables[]` 数组，写死在 server 里 |
| **DD 系统视图** | 绝大多数 | `sql/dd/info_schema/` 下的系统视图（**8.0 的 `I_S.COLUMNS`/`TABLES`/`STATISTICS` 都在这里**），不走 `ST_SCHEMA_TABLE` |
| **插件表** | InnoDB 26 张等 | `MYSQL_INFORMATION_SCHEMA_PLUGIN`，见 1.2 |

⚠️ 这是最容易误判的一点：`schema_tables[]` 里**没有** `COLUMNS`——因为 8.0 的 `I_S.COLUMNS` 已经是 DD 系统视图了。

原生表那 13 张：`COLUMN_PRIVILEGES`、`ENGINES`、`OPEN_TABLES`、`OPTIMIZER_TRACE`、`PLUGINS`、`PROCESSLIST`、`PROFILING`、`SCHEMA_PRIVILEGES`、`TABLESPACES`、`TABLE_PRIVILEGES`、`USER_PRIVILEGES`、`TMP_TABLE_COLUMNS`、`TMP_TABLE_KEYS`。

`ST_SCHEMA_TABLE` 的两个钩子之外，还有三个字段：

```cpp
struct ST_SCHEMA_TABLE {
  const char *table_name;
  ST_FIELD_INFO *fields_info;
  int (*fill_table)(THD *thd, Table_ref *tables, Item *cond);
  int (*old_format)(THD *thd, ST_SCHEMA_TABLE *schema_table);   // 老 SHOW 兼容
  int (*process_table)(...);                                     // 逐 (db,table) 后处理
  bool hidden;                                                   // 是否对用户隐藏
};
```

- `old_format`：把 I_S 的大写列名改写成老 `SHOW` 的首字母大写短名（`ID` → `Id`）
- `process_table`：**只用于临时表场景**的 `SHOW COLUMNS`/`KEYS`（只有 `TMP_TABLE_COLUMNS`/`TMP_TABLE_KEYS` 设了它）
- `hidden`：**只对 SHOW 类命令可见**，用户直接 `SELECT` 会报 `ER_UNKNOWN_TABLE`。13 张原生表里只有 3 张是 hidden：`OPEN_TABLES`、`TMP_TABLE_COLUMNS`、`TMP_TABLE_KEYS`——它们必须存在，因为 `SHOW OPEN TABLES` 正是通过"重写成 I_S 查询"实现的。

#### 1.2 注册：插件怎么挂进来

两段式：

**① 插件加载时**，server 分配一个**零填充**的 `ST_SCHEMA_TABLE`，把 plugin 名塞进去，然后回调插件自己的 `init()` 由它填 `fields_info` / `fill_table`：

```cpp
  /* Historical Requirement */
  plugin->data = schema_table;  // shortcut for the future
  if (plugin->plugin->init) {
    schema_table->old_format = make_old_format;
    schema_table->table_name = plugin->name.str;
    if (plugin->plugin->init(schema_table)) { ... }
  }
```

⚠️ 因为是零填充，**插件表的 `hidden` 恒为 false**——InnoDB 那 26 张表人人可 `SELECT`。

**② 名字解析时**，`find_schema_table()` **先扫内置数组，扫不到再遍历所有 I_S 插件**：

```cpp
ST_SCHEMA_TABLE *find_schema_table(THD *thd, const char *table_name) {
  for (; schema_table->table_name; schema_table++) {
    if (!my_strcasecmp(system_charset_info, schema_table->table_name, table_name))
      return schema_table;
  }
  schema_table_a.table_name = table_name;
  if (plugin_foreach(thd, find_schema_table_in_plugin,
                     MYSQL_INFORMATION_SCHEMA_PLUGIN, &schema_table_a))
    return schema_table_a.schema_table;
  return nullptr;
}
```

匹配靠的是 **plugin 名 == 表名**——这就是 InnoDB 那 26 个 `st_mysql_plugin` 的 `name` 必须等于表名的原因。

#### 1.3 调度链：从 SQL 到 `fill_table`

```
SELECT * FROM information_schema.innodb_trx
  ↓ ① add_table_to_list()：is_infoschema_db → find_schema_table() → 挂到 Table_ref::schema_table
  ↓ ② open_and_process_table()：只建壳不填数据
  ↓ ③ 优化器：包一层 NewMaterializeInformationSchemaTableAccessPath（带上 condition）
  ↓ ④ MaterializeInformationSchemaTableIterator::Init()：清表 → fill → 标记已填
  ↓ ⑤ do_fill_information_schema_table() → InnoDB 的 fill_table
  ↓ ⑥ 之后按普通临时表扫描（Read 转发给内部迭代器）
```

第 ② 步的注释说得最直白——**开表时只建壳，执行时才填**：

```cpp
  /*
    If this is a placeholder for an information_schema table, create a
    temporary table to represent it. Do not fill it yet - will be filled
    during execution.
  */
```

第 ④ 步**只填一次**（`if (!m_table_list->schema_table_filled)`），所以同一语句内嵌套循环复用同一份物化结果。

#### 1.4 ★ 虚拟表怎么"看起来像一张表"

**答案：造一张真的内存临时表。** `create_schema_table()` 把 `ST_FIELD_INFO` 逐条翻译成 `Item`，再 `create_tmp_table()`：

```cpp
  for (; fields_info->field_name; fields_info++) {
    switch (fields_info->field_type) {
      case MYSQL_TYPE_TINY: ... MYSQL_TYPE_LONGLONG:
        item = new Item_return_int(fields_info->field_name, fields_info->field_length,
                                   fields_info->field_type, fields_info->value);
        item->unsigned_flag = (fields_info->field_flags & MY_I_S_UNSIGNED);
      default:
        /* Don't let unimplemented types pass through. Could be a grave error. */
        assert(fields_info->field_type == MYSQL_TYPE_STRING);
    }
    fields.push_back(item);
    item->set_nullable(fields_info->field_flags & MY_I_S_MAYBE_NULL);
  }
  Temp_table_param *tmp_table_param = new (thd->mem_root) Temp_table_param;
  tmp_table_param->schema_table = true;
  table = create_tmp_table(thd, tmp_table_param, fields, ...);
```

然后标记为系统临时表并**直接授予 SELECT**：

```cpp
  table->s->tmp_table = SYSTEM_TMP_TABLE;
  table_list->grant.privilege = SELECT_ACL;
```

**元数据方面**，server 还把这份列定义物化成 DD 里的一张 system view（这样 `SHOW CREATE TABLE`、列名解析才有东西可查）。

#### 1.5 落行：`schema_table_store_record()` 与溢出转磁盘

插件写完一行调它，它就是往那张 HEAP 表 `ha_write_row`：

```cpp
/*
  Store record to I_S table, convert HEAP table
  to MyISAM if necessary
*/
bool schema_table_store_record(THD *thd, TABLE *table) {
  int error;
  if ((error = table->file->ha_write_row(table->record[0]))) {
    if (create_ondisk_from_heap(thd, table, error, ...))
      return true;
  }
  return false;
}
```

⚠️ **结果集太大时 I_S 查询会静默变成磁盘临时表写入**——这是 I_S 查询一个不显眼但很实际的代价。引擎侧完全不需要知道底层存储换过。

#### 1.6 ★ `fill_table()`：每次查询现场填充

框架篇讲了"声明 → 注册 → 填充"三步，这里补**填充这一步的真实代码**。

##### ① 调用点：每次查询都调一次

```cpp
  // sql/sql_show.cc
  bool res = table_list->schema_table->fill_table(thd, table_list, condition);
```

⇒ **I_S 表没有任何持久化数据**。你每次 `SELECT ... FROM information_schema.xxx`，server 都会调一次 `fill_table()`，由它**现场把行写进一张临时表**，再把结果返回给你。

⚠️ 这带来两条重要后果：

1. **代价随"被观测对象的规模"增长**，而不是随结果集大小。例如 `INNODB_BUFFER_PAGE` 要遍历整个缓冲池的所有页元数据——**不管你加不加 WHERE**（很多 I_S 表的 `WHERE` 无法下推到填充阶段）。
2. **多次查询之间结果可能不一致**（每次都是新的快照）。

##### ② 声明：`ST_FIELD_INFO` 数组

```cpp
struct ST_FIELD_INFO {
  /**
      This is used as column name.
  */
  const char *field_name;
  /**
     For string-type columns, this is the maximum number of
     characters. Otherwise, it is the 'display-length' for the column.
     For the data type MYSQL_TYPE_DATETIME this field specifies the
     number of digits in the fractional part of time value.
  */
  uint field_length;
  /**
     This denotes data type for the column. For the most part, there seems to
     ...
  */
  ...
};
```

每张表声明一个数组，以 `END_OF_ST_FIELD_INFO` 结尾：

```cpp
static ST_FIELD_INFO innodb_trx_fields_info[] = {
    ...
    END_OF_ST_FIELD_INFO};
```

⇒ **列定义就是这张数组**，`field_name` / `field_length` / `field_type` 三个字段决定了一列的元数据。**想确认某列的类型/长度，查这张数组比查文档准**。

##### ③ 实现：以 `trx_i_s_common_fill_table()` 为例

```cpp
static int trx_i_s_common_fill_table(
    THD *thd,          /*!< in: thread */
    Table_ref *tables, /*!< in/out: tables to fill */
    Item *)            /*!< in: condition (not used) */
{
  const char *table_name;
  int ret [[maybe_unused]];
  trx_i_s_cache_t *cache;

  DBUG_TRACE;

  /* deny access to non-superusers */
  if (check_global_access(thd, PROCESS_ACL)) {
    return 0;
  }

  /* minimize the number of places where global variables are referenced */
  cache = trx_i_s_cache;

  /* which table we have to fill? */
  table_name = tables->table_name;

  /* update the cache */
  trx_i_s_cache_start_write(cache);
  trx_i_s_possibly_fetch_data_into_cache(cache);
  trx_i_s_cache_end_write(cache);

  if (trx_i_s_cache_is_truncated(cache)) {
    ib::warn(ER_IB_MSG_599) << "Data in " << table_name
                            << " truncated due to"
                               " memory limit of "
                            ...
```

**逐段解释**：

- **★ `check_global_access(thd, PROCESS_ACL)` 在最前面**：I_S 的事务类表（`INNODB_TRX` 等）**需要 PROCESS 权限**，没权限**静默返回 0 行**（不报错）。这是"查 `INNODB_TRX` 返回空"最常见的原因（不是没事务，是没权限）。
- **★ 缓存机制是这里的关键设计**：
  ```
  trx_i_s_cache_start_write(cache)          ← 加写锁
  trx_i_s_possibly_fetch_data_into_cache()  ← 把 trx_sys 里的当前事务快照进 cache
  trx_i_s_cache_end_write(cache)            ← 释放
  ```
  ⇒ **为什么不直接遍历 `trx_sys`？** 因为遍历事务列表要持有 `trx_sys` 的锁，而填充过程（逐行 `field->store()`）可能很慢。**先把数据拷到自己的 cache，再慢慢格式化**——把"持有引擎锁的时间"压缩到最短。这是 I_S 填充的标准范式。
- **★ 截断警告**：`trx_i_s_cache_is_truncated()` —— 事务太多超出 cache 上限时**截断**，并在错误日志打一条 `ib::warn`。⇒ **看到这条警告意味着你查到的 `INNODB_TRX` 不全**。
- **一张 fill 函数服务多张表**：靠 `tables->table_name` 判断当前要填哪张（`INNODB_TRX` / `INNODB_TRX_LOCK_WAITS`… 共用这个入口），所以 `schema->fill_table = trx_i_s_common_fill_table;` 会出现多次。
- **`Item *` 参数未使用**（注释写明 `condition (not used)`）⇒ ★ **I_S 填充阶段不处理 WHERE**，条件是在填充完成之后由 server 层过滤的。这印证了上面"WHERE 无法下推"的说法。

##### ④ 与 PFS 表的关键差别

| | I_S 表（`fill_table`） | PFS 表 |
|---|---|---|
| 数据来源 | **现场从引擎内部拷快照** | 游走 PFS 自己的内存结构 |
| 结构化方式 | `ST_FIELD_INFO` 数组声明列 | `read_row_values()` 逐列 `store()` |
| 一致性 | 单次查询内一致（cache 快照） | **可能读到"脏"数据**（乐观锁放弃不一致的行） |
| 锁 | 短暂持有引擎锁（拷快照期间） | **完全无锁**（乐观读） |
| 权限 | 部分表需 `PROCESS_ACL` | 由 `setup_actors` / 表级 ACL 控制 |

⇒ 一句话：**I_S 是"每次快照"，PFS 是"无锁游走"**。前者更一致但更贵，后者更轻但可能不准。

### 二、引擎侧：InnoDB 的 26 张表

（这半是同一个机制的插件侧实现。）

#### 2.1 表族全貌

**先澄清三个数**：注册生效 **26 张**；`ST_FIELD_INFO` 数组只有 **21 个**（多表共用）；头文件 `extern` 声明 **29 个**，其中 **3 个是孤儿**（见「已知限制」）。

| 类 | 张数 | 表 | 数据来源 |
|---|---|---|---|
| **事务** | 1 | `INNODB_TRX` | `trx_sys` + `lock_sys`，经 `trx_i_s_cache` 中转 |
| **压缩** | 6 | `INNODB_CMP` / `_RESET`、`_CMP_PER_INDEX` / `_RESET`、`_CMPMEM` / `_RESET` | `page_zip_stat[]`、`buf_pool->zip_free[]` / `buddy_stat[]` |
| **Buffer pool** | 3 | `INNODB_BUFFER_PAGE`、`_BUFFER_PAGE_LRU`、`_BUFFER_STATS` | 遍历 chunk/block、LRU 链、聚合值 |
| **全文检索** | 6 | `INNODB_FT_DEFAULT_STOPWORD`、`_FT_DELETED`、`_FT_BEING_DELETED`、`_FT_INDEX_CACHE`、`_FT_INDEX_TABLE`、`_FT_CONFIG` | 静态常量 / FTS 辅助表 / FTS 内存 cache |
| **字典元数据** | 6 | `INNODB_TABLES`、`_TABLESTATS`、`_INDEXES`、`_COLUMNS`、`_VIRTUAL`、`_TABLESPACES` | **扫 `mysql.*` DD 表** |
| **临时表** | 2 | `INNODB_TEMP_TABLE_INFO`、`_SESSION_TEMP_TABLESPACES` | `dict_sys->table_non_LRU`、临时表空间池 |
| **AHI** | 1 | `INNODB_CACHED_INDEXES` | 扫 `mysql.indexes` + AHI 缓存页统计 |
| **monitor 出口** | 1 | `INNODB_METRICS` | monitor counter 数组（与 [`../innodb/monitor.md`](innodb_monitor.md) 的唯一交集） |

`_RESET` 变体与本体共用列定义数组，只是把 `reset` 参数传成 `true`（读完即清零统计）。

#### 2.2 声明 → 注册 → 填充

**① 声明**：`ST_FIELD_INFO` 数组，`#define IDX_*` **插在数组元素之间**（靠物理顺序保证下标与列序一致）：

```c
/* Every time any column gets changed, added or removed, please remember
to change i_s_innodb_plugin_version_postfix accordingly, so that
the change can be propagated to server */
static ST_FIELD_INFO innodb_trx_fields_info[] = {
#define IDX_TRX_ID 0
    {STRUCT_FLD(field_name, "trx_id"),
     STRUCT_FLD(field_length, MY_INT64_NUM_DECIMAL_DIGITS),
     STRUCT_FLD(field_type, MYSQL_TYPE_LONGLONG), STRUCT_FLD(value, 0),
     STRUCT_FLD(field_flags, MY_I_S_UNSIGNED), STRUCT_FLD(old_name, ""),
     STRUCT_FLD(open_method, 0)},
    ...
    END_OF_ST_FIELD_INFO};
```

两个工程细节：

- **`END_OF_ST_FIELD_INFO` 是哨兵**，用 `MYSQL_TYPE_NULL` 结尾：
```c
#define END_OF_ST_FIELD_INFO                                           \
  {                                                                    \
    STRUCT_FLD(field_name, NULL), STRUCT_FLD(field_length, 0),         \
        STRUCT_FLD(field_type, MYSQL_TYPE_NULL), STRUCT_FLD(value, 0), \
        STRUCT_FLD(field_flags, 0), STRUCT_FLD(old_name, ""),          \
        STRUCT_FLD(open_method, 0)                                     \
  }
```
- **`STRUCT_FLD` 是个兼容宏，非 GCC 下退化成纯值**——说明这套写法是 C-ABI 直填结构体：
```c
#if !defined __STRICT_ANSI__ && defined __GNUC__ && !defined __clang__
#define STRUCT_FLD(name, value) name: value
#else
#define STRUCT_FLD(name, value) value
#endif
```

`ST_FIELD_INFO` 各字段：

| 字段 | 含义 |
|---|---|
| `field_name` | 列名 |
| `field_length` | 字符串＝最大字符数；其他＝显示宽度；`DATETIME` ＝ 小数秒位数 |
| `field_type` | SQL 类型 |
| `field_flags` | 只有两位可用：`MY_I_S_MAYBE_NULL`、`MY_I_S_UNSIGNED`（默认 `NOT NULL` + `SIGNED`） |
| `old_name` / `value` / `open_method` | **InnoDB 侧恒填 `""` / 0**——`open_method` 的注释就是 `// Not used` |

InnoDB 侧还有一份手写的 **C 类型 → `field_type` 映射规约**（写在文件顶部注释里，是"手写列定义时必须遵守的对照表"）：

```
C type              ST_FIELD_INFO::field_type
--------------------------------------------------------------
long                    MYSQL_TYPE_LONGLONG (len=MY_INT64_NUM_DECIMAL_DIGITS)
long unsigned           MYSQL_TYPE_LONGLONG (同上 + MY_I_S_UNSIGNED)
char*                   MYSQL_TYPE_STRING   (len=n)
float                   MYSQL_TYPE_FLOAT    (len=0 ignored)
void*                   MYSQL_TYPE_LONGLONG (同 long unsigned)
boolean (if else)       MYSQL_TYPE_LONG     (len=1)
time_t                  MYSQL_TYPE_DATETIME (len=0 ignored)
--------------------------------------------------------------
```

关于 `value` 和 `open_method`：**源码没有任何注释说明它们的作用**——`ST_FIELD_INFO::value` 在 server 侧定义里只有一句 `int value;`（无注释），`open_method` 注释就是 `// Not used`；InnoDB 全部 21 个数组都填 `0`。结论：**这两个字段是历史遗留，`old_name` 同理**（InnoDB 恒填 `""`，只有 server 自己的老表才用 `old_name` 做 SHOW 兼容）。

**② 注册**：`init` 挂两个钩子 + 插件描述符（26 个插件共用同一个 `i_s_info`、`i_s_common_deinit`、版本号）：

```c
static int innodb_trx_init(void *p) {
  ST_SCHEMA_TABLE *schema = (ST_SCHEMA_TABLE *)p;
  schema->fields_info = innodb_trx_fields_info;      // 列定义
  schema->fill_table = trx_i_s_common_fill_table;    // 填充回调
  return 0;
}
```

**③ 填充**：直接操作 server 造好的 `TABLE`/`Field`，按 `IDX_*` 下标 `store()`，最后 `schema_table_store_record()` 落一行。完整骨架：

```c
static int fill_innodb_trx_from_cache(
    trx_i_s_cache_t *cache, THD *thd, TABLE *table) {
  Field **fields;
  ...
  fields = table->field;
  rows_num = trx_i_s_cache_get_rows_used(cache, I_S_INNODB_TRX);

  for (i = 0; i < rows_num; i++) {
    i_s_trx_row_t *row;
    row = (i_s_trx_row_t *)trx_i_s_cache_get_nth_row(cache, I_S_INNODB_TRX, i);

    /* trx_id */
    OK(fields[IDX_TRX_ID]->store(row->trx_id, true));
    /* trx_state */
    OK(field_store_string(fields[IDX_TRX_STATE], row->trx_state));
    /* trx_started */
    OK(field_store_time_t(fields[IDX_TRX_STARTED], ...));
    ...
    OK(schema_table_store_record(thd, table));
  }
  return 0;
}
```

`OK()` 是"失败立即返回"的宏——注意它返回的是 **1**（与后面「已知限制」里"`fill_table` 永远 `return 0`"的 workaround 自相矛盾）：

```c
#define OK(expr)     \
  if ((expr) != 0) { \
    return 1;        \
  }
```

三个共用的写列辅助（所有表复用）：

- `field_store_string()`：非 NULL 走 `field->store(str, len, system_charset_info)` + `set_notnull()`；NULL 走 `set_null()` 且 **`ret = 0`（视为成功）**
- `field_store_time_t()`：`localtime_r` 转 `MYSQL_TIME`
- `field_store_index_name()`：处理临时索引前缀 `TEMP_INDEX_PREFIX` 不是合法 UTF8MB3 的问题，替换成 `'?'`

#### 2.3 ★ 加锁与一致性：每张表口径都不同

**这是最实用的一节**——查这些表会不会影响生产、读到的是不是同一时刻的数据。

| 表 | 加锁方式 | 一致性级别 |
|---|---|---|
| `INNODB_TRX` | 持 locksys 全局排他 latch + `trx_sys` mutex，**拷进缓存后立即释放** | **真快照**，且 100ms 内跨查询复用 |
| `INNODB_CMP*` / `CMPMEM*` | 各自 mutex 下拷快照，出锁后再渲染 | **快照** |
| `INNODB_BUFFER_PAGE_LRU` | **全程持 `LRU_list_mutex`** | 一致，但代价极高 |
| `INNODB_BUFFER_PAGE` | 无外层锁，仅逐页短暂拿页 mutex | **不一致**（逐块扫描，块间可变） |
| DD 系列 | 逐行 `dict_sys` mutex + mtr **反复拿放** | **不一致**（行间可变），但持 MDL 不会崩 |
| FTS 系列 | 开 InnoDB **后台事务**读辅助表 | 事务读 |

**`INNODB_TRX` 是唯一的真快照。** 它先刷新缓存再从容渲染，持全局 latch 的只是"拷贝"这一瞬间：

```c
  /* The latching is done in the following order:
  acquire trx_i_s_cache_t::rw_lock, X
  acquire locksys exclusive global latch
  acquire trx_sys mutex
  release trx_sys mutex
  release locksys exclusive global latch
  release trx_i_s_cache_t::rw_lock
  acquire trx_i_s_cache_t::rw_lock, S
  release trx_i_s_cache_t::rw_lock */
```

实际取数只有三行——**拷完立刻全放**：

```c
  {
    /* We need to read trx_sys and record/table lock queues */
    locksys::Global_exclusive_latch_guard guard{UT_LOCATION_HERE};
    trx_sys_mutex_enter();
    fetch_data_into_cache(cache);
    trx_sys_mutex_exit();
  }
```

断言确认了"必须排他"的原因（要跨多个 lock_sys shard）：

```c
  /* We are going to iterate over many different shards of lock_sys so we need
  exclusive access */
  ut_ad(locksys::owns_exclusive_global_latch());
  ut_ad(trx_sys_mutex_own());
```

更精妙的是"100ms 内不刷新"——为了让 JOIN 多张 I_S 表看到同一版本：

```c
/** The minimum time that a cache must not be updated after it has been
  read for the last time. We use this technique to ensure that SELECTs which
  join several INFORMATION SCHEMA tables read the same version of the cache. */
  constexpr std::chrono::milliseconds cache_min_idle_time{100};
```

**代价**：写路径上会短暂冻结整个锁子系统。

**`INNODB_BUFFER_PAGE_LRU` 最危险**——整条 LRU 链扫描期间 latch 不释放，还先按链长一次性分配：

```c
  /* Obtain buf_pool->LRU_list_mutex before allocate info_buffer, since
  UT_LIST_GET_LEN(buf_pool->LRU) could change */
  mutex_enter(&buf_pool->LRU_list_mutex);
  lru_len = UT_LIST_GET_LEN(buf_pool->LRU);
  info_buffer = (buf_page_info_t *)my_malloc(PSI_INSTRUMENT_ME,
                    lru_len * sizeof *info_buffer, MYF(MY_WME));
```

**`INNODB_BUFFER_PAGE` 几乎不加锁**——外层无 latch，只在每页内部短暂拿页 mutex：

```c
static void i_s_innodb_buffer_page_get_info(
    const buf_page_t *bpage, ulint pool_id, ulint pos, buf_page_info_t *page_info) {
  BPageMutex *mutex = buf_page_get_mutex(bpage);
  page_info->pool_id = pool_id;
  page_info->block_id = pos;
  mutex_enter(mutex);
  page_info->page_state = buf_page_get_state(bpage);
  /* Only fetch information for buffers that map to a tablespace, ... */
  if (buf_page_in_file(bpage)) { ... }
  ...
  mutex_exit(mutex);
}
```

所以**逐页字段自洽，但整张结果不是同一时刻的状态**。它分块处理，每 10000 页渲染一批：

```c
/** Maximum number of buffer page info we would cache. */
const ulint MAX_BUF_INFO_CACHED = 10000;
```

#### 压缩类表：显式"快照以避免锁序违规"

```c
  /* Create a snapshot of the stats so we do not bump into lock
  order violations with dict_sys->mutex below. */
  mutex_enter(&page_zip_stat_per_index_mutex);
  page_zip_stat_per_index_t snap(page_zip_stat_per_index);
  mutex_exit(&page_zip_stat_per_index_mutex);

  dict_sys_mutex_enter();
  ...
  for (iter = snap.begin(); iter != snap.end(); iter++) {
      const dict_index_t *index = dict_index_find(iter->first);
```

CMPMEM 同理，在 `zip_free_mutex` 下把 buddy 统计拷进局部变量再出锁渲染（注意 `os_rmb` 读屏障）：

```c
    mutex_enter(&buf_pool->zip_free_mutex);
    /* Save buddy stats for buffer pool in local variables. */
    for (uint x = 0; x <= BUF_BUDDY_SIZES; x++) {
      zip_free_len_local[x] = ...;
      os_rmb;
      buddy_stat_local[x] = buf_pool->buddy_stat[x].take_snapshot();
    }
    mutex_exit(&buf_pool->zip_free_mutex);
```

#### DD 元数据系列：锁被反复拿放（最反直觉）

**每扫一行就 `mtr_commit` + `dict_sys_mutex_exit()`，处理完再重新拿**：

```c
  dict_sys_mutex_enter();
  mtr_start(&mtr);
  rec = dd_startscan_system(thd, &mdl, &pcur, &mtr, dd_tables_name.c_str(), &dd_tables);
  while (rec) {
    ...
    dd_process_dd_tables_rec_and_mtr_commit(heap, rec, &table_rec, dd_tables, &mdl_on_tab, &mtr);
    dict_sys_mutex_exit();
    if (table_rec != nullptr) { ... 填充 ... }
    /* 下一轮再 dict_sys_mutex_enter(); mtr_start(&mtr); */
  }
```

所以 `INNODB_TABLES` 等连"chunk 级一致"都没有：**逐行加锁、行间可变**。对 DBA 的含义是多行结果可能逻辑上不自洽（扫到一半某表被 DDL 掉）。持 MDL 保证不会崩。

`INNODB_TABLESTATS` 额外在行上加统计锁（`dict_table_stats_lock(table, RW_S_LATCH)`）。

`INNODB_TEMP_TABLE_INFO` 明确只扫小链表以降低成本：

```c
  /* First populate all temp-table info by acquiring dict_sys->mutex.
  Note: Scan is being done on NON-LRU list which mainly has system
  table entries and temp-table entries. This means 2 things: list
  is smaller so processing would be faster and most of the data
  is relevant */
```

#### 2.4 `trx0i_s`：适配层与缓存层

**它不是通用工具库，只服务 `INNODB_TRX`**，是"内部结构 → I_S 行"的中间层。文件头注释即定位：

```c
/** INFORMATION SCHEMA innodb_trx, innodb_locks and
 innodb_lock_waits tables cache structures and public functions. */
```

中间缓冲的角色（头文件原文）：

```c
/** This is the intermediate buffer where data needed to fill the
INFORMATION SCHEMA tables is fetched and later retrieved by the C++
code in handler/i_s.cc. */
extern trx_i_s_cache_t *trx_i_s_cache;
```

四类职责：

**① 缓存结构与读写锁 API**（`start_read` / `end_read` / `start_write` / `get_rows_used` / `get_nth_row` / `possibly_fetch_data_into_cache` / `is_truncated`）

**② 行结构体**——`i_s_locks_row_t` 里两段注释解释了**为什么用"地址"当 ID 而不是 `trx->id`**：

```c
  uint64_t lock_trx_immutable_id; /*!< transaction address as integer. We need
                        an id which is unique and does not change over time.
                        Unfortunately trx->id is initially equal to 0 for
                        all trxs which still appear to be read only, and it
                        changes to non-zero, once trx needs to perform write.
                        For this reason trx->id is not good enough. */
  uint64_t lock_immutable_id; /*!< lock address as integer. ...
                      We use both lock_immutable_id and lock_trx_immutable_id
                      even though lock_immutable_id is unique, because we need
                      to be able to locate the row in PERFORMANCE_SCHEMA based
                      on the id, and we need a way to verify that the
                      lock_immutable_id is safe to dereference. */
```

第二段直接点出 `trx0i_s` 与 PFS 的耦合——**"locate the row in PERFORMANCE_SCHEMA"**。

**③ 格式化 / 序列化工具**：`trx_i_s_create_lock_id()`（拼 `"%lu:%lu:%lu:%lu:%lu"`）、`trx_i_s_parse_lock_id()`（反向解析）、`p_s_fill_lock_data()` / `fill_locks_row()`（**明确供 PFS 侧调用**）、`TRX_I_S_STRING_COPY` 宏（带内存上限的安全字符串拷贝）。

**④ 容量上限与截断**：

```c
/** The maximum amount of memory that can be consumed by innodb_trx,
innodb_locks and innodb_lock_waits information schema tables. */
constexpr uint32_t TRX_I_S_MEM_LIMIT = 16777216; /* 16 MiB */
```

超限置 `is_truncated = true`，只打一条 warning，结果集照样返回（不完整）：

```c
  if (trx_i_s_cache_is_truncated(cache)) {
    ib::warn(ER_IB_MSG_599) << "Data in " << table_name << " truncated due to"
                            << " memory limit of " << TRX_I_S_MEM_LIMIT << " bytes";
  }
```

#### 2.5 `p_s.cc`：PFS 那条出口

> ⚠️ **纠正一个望文生义的猜测**：`p_s.cc` **不定义任何 PFS 表，也不定义任何 I_S 表**。它是 InnoDB 侧的 **PFS data-lock 迭代器实现**——即"server 定义表结构、引擎填充"里引擎那一半。

文件头与 `@page` 注释开宗明义：

```c
/** InnoDB performance_schema tables interface to MySQL. */
```
```c
  @page PAGE_INNODB_PFS Innodb data lock instrumentation
  To provide content to the performance_schema.data_locks table,
  innodb implements #Innodb_data_lock_iterator.
  Likewise, table performance_schema.data_wait_locks
  is populated with #Innodb_data_lock_wait_iterator.
  ... which imply to take the proper mutex locks when inspecting it.
  The structure to inspect here is the transaction list (#trx_sys)
```

它实现 server 提供的抽象基类（`PSI_engine_data_lock_iterator` 的 `scan` / `fetch`），**不认识 `TABLE`/`Field`**，只往 `PSI_server_data_lock_container` 里塞。注册点是 `mysql_data_lock_register(&innodb_data_lock_inspector)`——不是插件表，是 PSI 服务注册。

**与 I_S 的关系：并列的两条出口，`trx0i_s` 是共享底座**（行结构、lock id 的拼装与解析、`p_s_fill_lock_data` 都由 PFS 侧调用）。

#### ★ 一段极高质量的扫描策略权衡注释

它把三种方案写出来并逐个否决，是理解"为什么 PFS 能下推而 I_S 不能"的最佳材料：

```
  @subsection no_full_scan No full scan
  - Take all necessary locks
  - Scan all the innodb internal locks
  - Report all of them to the performance schema
  - Release all the locks taken
  This implementation materializes the entire table.
  The benefits: The materialized table is consistent
  The problems:
  - The innodb engine is frozen for the entire duration, for a time unpredictable
  - Memory consumption spikes, without bounds
  - Materializing all rows upfront is incompatible with supporting an index
  For example with N = 10,000 transactions, a single scan reports all 10,000
  transaction locks. This alternative is rejected.

  @subsection no_single_row_scan No single row scan
  ... looping N times to find element i, i+1, i+2 ... in a list
  ends up having a complexity in O(N^2), consuming CPU.
  This alternative is rejected.

  @subsection restartable_batch_scan Restartable batch scan
  What is implemented is:
  - Take all necessary locks
  - Resume the scan on innodb internal locks, for a range containing at most
    RANGE transactions
  - Report all the records of transactions in the range to the p_s
  - Release all the locks taken
  Properties:
  - Memory consumption is bounded
  - The duration of mutex locks on innodb structures is bounded
  - The data returned is not consistent, but at least it is "consistent by chunks"
```

`SCAN_RANGE = 256`，且用 `trx_immutable_id` 作为持久游标（因为事务链表既不按 id 也不按地址排序）。加锁方式与 `INNODB_TRX` 完全一致：

```c
  /* We want locks reported in a single scan to be a consistent snapshot. */
  locksys::Global_exclusive_latch_guard guard{UT_LOCATION_HERE};
  IB_mutex_guard trx_sys_mutex_guard{&trx_sys->mutex, UT_LOCATION_HERE};
```

### 三、权限：为什么是引擎手写 21 处

#### server 侧：库级无条件放行，逐表规则主动放弃

```cpp
ACL_internal_access_result IS_internal_schema_access::check(
    Access_bitmask want_access, Access_bitmask *save_priv, bool) const {
  want_access &= ~SELECT_ACL;
  /*
    We don't allow any simple privileges but SELECT_ACL on
    the information_schema database.
  */
  if (unlikely(want_access & DB_ACLS)) return ACL_INTERNAL_ACCESS_DENIED;
  /* Always grant SELECT for the information schema. */
  *save_priv |= SELECT_ACL;
  ...
}
const ACL_internal_table_access *IS_internal_schema_access::lookup(const char *) const {
  /* There are no per table rules for the information schema. */
  return nullptr;
}
```

**`lookup()` 返回 `nullptr` 是硬证据**——server 对 I_S 不做逐表权限建模。开表路径也直接白送 SELECT（`table_list->grant.privilege = SELECT_ACL`）。8.0 里大多已是 DD 系统视图的 I_S 表，连检查都跳过。

server 侧自己只留了**两条**额外门槛：`I_S.FILES` 要求 `PROCESS`；hidden 表/视图拦截（非 SHOW 命令引用报 `ER_NO_SYSTEM_VIEW_ACCESS`）。

#### 引擎侧：因此被迫逐表手写

**共 21 处** `check_global_access(thd, PROCESS_ACL)`，覆盖 26 张表（`_RESET` 变体共享 `_fill_low`）：

```c
  /* deny access to non-superusers */
  if (check_global_access(thd, PROCESS_ACL)) {
    return 0;
  }
```

三种注释措辞代表三个年代：`/* deny access to non-superusers */`（最老 9 处）、`/* Only allow the PROCESS privilege holder ... */`、`/* deny access to user without PROCESS_ACL privilege */`（8.0 新增的表）。⚠️ 注释说 "non-superusers" 但判的是 `PROCESS_ACL`——历史措辞遗留。

**唯一例外：`INNODB_FT_DEFAULT_STOPWORD`** —— 只读编译期常量数组，任何用户可查：

```c
static int i_s_stopword_fill(THD *thd, Table_ref *tables, Item *) {
  /* Fill with server default stopword list in array fts_default_stopword */
  while (fts_default_stopword[i]) {
    OK(field_store_string(fields[STOPWORD_VALUE], fts_default_stopword[i]));
    OK(schema_table_store_record(thd, table));
    i++;
  }
  return 0;
}
```

**为什么无权限是"静默返回空"而不是报错？** `check_global_access()` 内部已经 `my_error()` 过——但注意 InnoDB 的写法是 `if (...) return 0`，这里 `return 0` 表示"填充成功、0 行"。源码无注释解释这个决策。

> 推论（非源码结论）：与 I_S 一贯做法一致——对无权限对象做**行级过滤而非报错**，不泄露"该对象存在与否"；报错反而会把"有你看不了的东西"变成可读信息。

### 四、I_S / PFS / SHOW 三分天下

#### 4.1 I_S 与 PFS 不是同一套机制

| | I_S（含 InnoDB 26 张） | PFS |
|---|---|---|
| 插件类型 | `MYSQL_INFORMATION_SCHEMA_PLUGIN` | `MYSQL_STORAGE_ENGINE_PLUGIN` |
| 注册结构 | `ST_SCHEMA_TABLE` | `handlerton` |
| 取数 | `fill_table()` 灌临时表再扫 | 真表真扫描 `rnd_init`/`rnd_next` |

**唯一交叉点**：引擎可通过 `handlerton::fill_is_table` 往 I_S 表塞数据——目前只有 `I_S.TABLESPACES` 这么干（且已被标记 deprecated）。

#### 4.2 SHOW 的三条出路

| 出路 | 例子 |
|---|---|
| **重写成 PFS 查询** | `SHOW [GLOBAL] STATUS`、`SHOW [GLOBAL] VARIABLES`（`sql_show_status.cc` 文件头注释写明重写成 `performance_schema.global_status` 等）、`SHOW PROCESSLIST`（PFS 可用时） |
| **重写成 I_S 系统视图查询** | `SHOW COLUMNS`/`KEYS`/`DATABASES`/`TABLES`/`TRIGGERS`/`CHARACTER SET`…（`sql/dd/info_schema/show.cc`） |
| **原生实现或走 `schema_tables[]`** | `SHOW CREATE TABLE`（`mysqld_show_create()`）、`SHOW MASTER STATUS`、`SHOW BINLOG EVENTS`、`SHOW ENGINES`/`PLUGINS`/`OPEN TABLES` |

⚠️ `I_S.PROCESSLIST` 已被官方标记 deprecated（推荐 `performance_schema.processlist`）。

#### 4.3 三条可观测性路径的选型

| 维度 | **I_S 表** | **monitor counter** | **status var / PFS** |
|---|---|---|---|
| **数据来源** | 现场遍历内部结构体（或扫 `mysql.*` DD 表） | 常驻计数器数组（`MONITOR_INC` 埋点） | 从内部统计"搬运" / PSI 事件 |
| **更新时机** | **查询时 100% 现场生成**（`INNODB_TRX` 的 100ms 缓存是唯一例外） | **写路径持续累加** | `SHOW STATUS` 调用瞬间才搬运；PFS 是事件发生时写 |
| **读开销** | **最高**（可能全物化、持全局 latch、开后台事务） | **极低**（一次位图判断 + relaxed load/store） | status var 中等；PFS 高 1~2 个数量级 |
| **一致性** | **普遍不一致**；仅 TRX / CMP* / BUFFER_PAGE_LRU 是快照 | 天然不一致（无锁埋点，但计数不丢） | status var 是"语句前快照+增量转移"；PFS 是 chunk 级一致 |
| **持久化** | 否（纯内存虚拟表） | 否 | 否 |
| **粒度** | **逐对象明细**（每页/每事务/每索引/每列），**无时间维度** | 无维度，一个 int64 | status var 全局聚合；PFS 多维（thread/account/user/host/object） |
| **`WHERE` 下推** | ❌ | ❌ | ✅ PFS 支持 |
| **权限** | 几乎全部需 `PROCESS`（25/26） | 需 `PROCESS` | `SHOW STATUS` 无需特殊权限 |
| **适合** | 一次性取证 / 人工排障 / 盘点 | 趋势、阈值告警、长期埋点 | status var：通用采集；PFS：定位"谁/哪条 SQL/慢在哪" |
| **不适合** | **高频轮询** | 细粒度归因、关系型明细 | PFS 默认 mutex/rwlock 关闭，覆盖不全 |
| **已知副作用** | 可能冻结锁子系统、持 LRU latch、开后台事务、静默截断 | 几乎无 | PFS 开启 instrument 显著增加开销 |

（status var 的搬运时机——注意它**不是持续更新**，而是每次 `SHOW STATUS` 才触发一次搬运，与 monitor counter 的"常驻累加"是两种更新时机。）

```c
/** Here we export InnoDB status variables to MySQL. */
static void innodb_export_status() {
  if (innodb_inited) srv_export_innodb_status();
}
```

```
要回答的问题
├─ "现在有哪些 X / 明细列表？"        → I_S
├─ "buffer pool 里都是什么页？"       → I_S（⚠ 代价极高，离峰做）
├─ "现在谁在等谁？"                   → PFS data_locks（可下推）/ I_S INNODB_TRX（仅事务视角）
├─ "系统整体趋势？"                   → monitor（INNODB_METRICS）
└─ "是谁？哪条 SQL？慢在哪？"          → PFS
```

### 五、版本机制：改列必须递增版本号

**两层版本，别混：**

| 层 | 是什么 | 不匹配后果 |
|---|---|---|
| plugin **接口**版本 | `MYSQL_INFORMATION_SCHEMA_INTERFACE_VERSION`（ABI） | 报 "API version ... is too different" |
| plugin **自身**版本 | InnoDB 的 `i_s_innodb_plugin_version_postfix` | DD 里存的 I_S 表元数据被删掉重建 |

第二层的机制：版本写进 DD 里 system view 的 options（`view_options->set(PLUGIN_VERSION_STRING, version)`），**启动时比对**：

```cpp
      if (plugin_int->plugin->version == plugin_version &&
          plugin_int->state == PLUGIN_IS_READY) { ... continue; }
    // Remove metadata from DD if version mismatch.
    if (dd::info_schema::remove_I_S_view_metadata(thd, view->name())) break;
```

即：**server 只认版本号，不 diff 列定义**。所以 InnoDB 注释才强调"改列必须递增 postfix"——不递增的话，升级后 DD 里仍是旧列列表，新列不会出现。

server 自己的 13 张表不用 plugin version，而用 `SERVER_I_S_TABLE_STRING` 标记 + 独立的 I_S 版本门禁。

---

## 六、运维视角：怎么查（区别于前面五章的"机制"）

> 前面讲 **I_S 怎么实现**，本章讲 **运维怎么用它查问题**。
> ⚠️ 更完整的观测手段全景与选型决策树见 [`README.md`](README.md)（本目录总览）；锁等待/死锁的**诊断套路**见 [`../lock/`](../lock/)。

### 6.1 ★ `INNODB_*` 其实有两个来源（前面只覆盖了引擎侧 26 张）

前面「二」讲的是**引擎插件**提供的 26 张。但 I_S 里带 `INNODB_` 前缀的表还有**第二个来源**——**server 层 DD system view**（`sql/dd/impl/system_views/`），共 **5 张**：

| 表 | 位置 | 提供什么 |
|---|---|---|
| `INNODB_DATAFILES` | `sql/dd/impl/system_views/innodb_datafiles.cc` | 表空间对应的数据文件 |
| `INNODB_FIELDS` | `.../innodb_fields.cc` | InnoDB 索引字段的内部信息 |
| `INNODB_FOREIGN` | `.../innodb_foreign.cc` | 外键定义 |
| `INNODB_FOREIGN_COLS` | `.../innodb_foreign_cols.cc` | 外键列 |
| `INNODB_TABLESPACES_BRIEF` | `.../innodb_tablespaces_brief.cc` | 表空间的简要信息 |

**区别**：引擎侧 26 张读的是**引擎运行时内存**；server 层这 5 张读的是 **DD 里持久化的数据**（`mysql.*` DD 表）。⇒ 前者反映"此刻"，后者反映"已持久化"。

### 6.2 常用查询套路

```sql
-- ① 长事务：谁跑了多久、锁了多少行、在跑什么
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS running_sec,
       trx_state, trx_rows_locked, trx_rows_modified, trx_query
  FROM information_schema.INNODB_TRX
 ORDER BY trx_started LIMIT 10;

-- ② ★ 哪些表做过 instant DDL？
SELECT NAME, INSTANT_COLS, TOTAL_ROW_VERSIONS
  FROM information_schema.INNODB_TABLES
 WHERE INSTANT_COLS > 0;
-- 源码（handler/i_s.cc）：
--   fields[INNODB_TABLES_INSTANT_COLS]->store(
--       table->is_upgraded_instant() ? table->get_instant_cols() : 0);
-- ⇒ INSTANT_COLS = 0 表示没做过；> 0 = 第一次 instant ADD COLUMN 前的列数
--   TOTAL_ROW_VERSIONS = 当前行版本号

-- ③ instant 加的列的默认值存在哪
SELECT NAME, HAS_DEFAULT, DEFAULT_VALUE
  FROM information_schema.INNODB_COLUMNS
 WHERE HAS_DEFAULT = 1;

-- ④ 表 / 索引的 InnoDB 内部 id 与 root page
SELECT t.NAME, t.TABLE_ID, t.SPACE, t.ROW_FORMAT,
       i.NAME AS idx, i.INDEX_ID, i.PAGE_NO
  FROM information_schema.INNODB_TABLES t
  JOIN information_schema.INNODB_INDEXES i ON i.TABLE_ID = t.TABLE_ID
 WHERE t.NAME LIKE 'db/%';
```

### 6.3 ⚠️ 8.0 已移除的表（5.7 资料常踩的坑）

| 5.7 的表 | 8.0 的替代 |
|---|---|
| `INNODB_LOCKS` | `performance_schema.data_locks` |
| `INNODB_LOCK_WAITS` | `performance_schema.data_lock_waits` |

⇒ 照着 5.7 资料查 `INNODB_LOCKS` 会报错"表不存在"。这也是本篇「四、I_S / PFS / SHOW 三分天下」的一个具体体现：**锁信息的归属从 I_S 迁到了 PFS**。

---

## 已知限制与坑

| 坑 | 说明 |
|---|---|
| **`WHERE` 不下推** | `fill_table` 的 `Item *cond` 只是"提示"，server 不保证过滤（文档注释明说）。server 侧与引擎侧互证：这是接口设计如此 |
| **`INNODB_BUFFER_PAGE*` 代价极高** | 遍历整个 buffer pool；`_LRU` 全程持 `LRU_list_mutex` 且一次性 malloc 百万级条目 |
| **`INNODB_TRX` 会静默截断** | 16 MiB 上限，超限只打 warning |
| **读 I_S 会开后台事务** | FTS 系列；`FT_INDEX_CACHE` 还会等 sync 完成（可能阻塞） |
| **结果集太大静默转磁盘** | `schema_table_store_record()` 里 `create_ondisk_from_heap()` |
| **`fill_table` 返回非 0 会死锁** | 历史 workaround，`INNODB_TRX` 的填充永远 `return 0`，真实错误被吞掉（与 `OK()` 宏返回 1 自相矛盾）：`/* if this function returns something else than 0 then a deadlock occurs ... see bugs.mysql.com/29900 */` |
| **3 个孤儿声明** | `i_s_innodb_foreign` / `_foreign_cols` / `_datafiles` 全仓库只在头文件命中——`INNODB_SYS_FOREIGN` 这套 5.x 表在 8.0 已移除 |
| **无 `INNODB_LOCKS`** | 8.0 已被 PFS `data_locks` 取代 |
| **`fill_table` 的报错噪音需专门兜底** | `WHERE TABLE_NAME='xxx'` 会让填充函数在每个库里试开表、产生一堆 "table doesn't exist"。为此专门造了个"无限容量"的临时 `Diagnostics_area` 再过滤掉 ERROR |

---

## Misc

### 工程技法

1. **`#define IDX_*` 插在数组元素之间**——靠物理顺序保证下标与列序一致
2. **零填充 + 回调填字段**：server 分配 `ST_SCHEMA_TABLE` 后调插件 `init()`，插件只填自己关心的（代价是 `hidden` 等字段插件改不了）
3. **版本即契约**：server 只比对版本号不 diff 列定义，用最简机制解决"定义漂移"
4. **快照以避免锁序违规**（压缩类表注释明说）
5. **100ms 最小空闲期**——用时间换跨查询一致性
6. **共享底座**：`trx0i_s` 同时服务 I_S 与 PFS 两条出口

### 扩展点：新增一张插件 I_S 表

| # | 层 | 改动 |
|---|---|---|
| 1 | 引擎 | `ST_FIELD_INFO` 数组（`END_OF_ST_FIELD_INFO` 结尾）+ `#define IDX_*` |
| 2 | 引擎 | `fill_table` 回调，注意 `OK()` 与 `schema_table_store_record` |
| 3 | 引擎 | `init()` 挂 `fields_info` + `fill_table` |
| 4 | 引擎 | `st_mysql_plugin` 描述符，**`name` 必须 == 表名** |
| 5 | 引擎 | 加进 `mysql_declare_plugin` 列表（**漏这步表不存在**） |
| 6 | 引擎 | 改了已有列 → 递增 `i_s_innodb_plugin_version_postfix` |
| 7 | 引擎 | 如需权限门槛，在填充函数开头手写 `check_global_access` |

★ 最容易漏的是**第 5 步**——那 3 个孤儿 extern 正是这个错误的化石。

### 易混淆点

| 混淆 | 澄清 |
|---|---|
| `I_S.COLUMNS` 走 `ST_SCHEMA_TABLE`？ | ❌ 8.0 已是 **DD 系统视图**；`schema_tables[]` 里只有 13 张原生表 |
| PFS 表也是 `fill_table` 机制？ | ❌ PFS 是**真 handler**（`rnd_next`），I_S 是物化临时表 |
| 插件表能设 `hidden`？ | ❌ 零填充得来，恒 false |
| 加 `WHERE` 能减少扫描？ | ❌ 只是"提示"，永远先全量物化 |
| 没权限会报错？ | ❌ 静默返回空结果集 |
| `INNODB_TRX` 是实时的？ | ❌ 有 100ms 缓存复用窗口 |
| `SHOW` 都还在原生实现？ | ❌ 大部分已重写成 PFS / I_S 系统视图查询 |
| 表名还有 `SYS_` 前缀？ | ❌ 8.0 已去掉（但内部数组名还留着） |

---

## 参考

- MySQL 8.0 Reference Manual → *INFORMATION_SCHEMA Tables*
- MySQL 8.0 Reference Manual → *The INFORMATION_SCHEMA Plugin Interface*
- MySQL 8.0 Source Code → `sql/sql_show.cc`、`sql/table.h`、`storage/innobase/handler/i_s.cc`
- [../innodb/monitor.md](innodb_monitor.md)（monitor counter 体系）
- [../server/infra/pfs.md](pfs/pfs.md)（PFS instrument 体系）
