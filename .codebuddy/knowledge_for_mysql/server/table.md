# 表（Table）：元数据与通用操作

> **定位**：SQL 层"表"这条线的完整视图——三层元数据对象、表缓存、以及表的全部通用操作（开表 / 锁表 / CREATE / ALTER / RENAME / TRUNCATE / DROP / FLUSH）。
> **边界**：MDL 的细粒度语义见 [`../infra/lock/transactional/mdl.md`](../infra/lock/transactional/mdl.md)；server↔引擎分界面见 [`handler.md`](handler.md)；InnoDB 侧 `dict_table_t` 与 COPY/INPLACE/INSTANT 算法选型见 [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md)；表空间级 DDL（CREATE TABLESPACE）见 [`../innodb/physical/tablespace.md`](../innodb/physical/tablespace.md)；分区表见 [`../feat/partitioning.md`](../feat/partitioning.md)；临时表见 [`query/runtime/07_temptable.md`](query/runtime/07_temptable.md)。

---

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [三个缓存 / 实例池的分工（易混）](#三个缓存--实例池的分工易混)
  - [版本演进](#版本演进)
- [表元数据](#表元数据)（TABLE_SHARE / TABLE / Table_ref / ↔ `dict_table_t`）
- [表元信息：DD 读取路径与两套缓存](#表元信息dd-读取路径与两套缓存)（★ `get_table_share` + `m_open_in_progress`/`COND_open` / `open_table_def` 六个 `fill_*` + `prepare_share` 逐段源码 / `TABLE_SHARE ↔ dict_table_t` 一对多 / DD 缓存 vs TDC 两套缓存 / 四层构建链 + `dd_open_table`→`dict_table_open_on_name` 完整调用栈）
- [`Table_ref`：★ SQL 层的表引用（深度剖析）](#table_ref-sql-层的表引用深度剖析)（★ `next_local`/`next_global` 双链 + `embedding`/`join_list` 树 / 名字解析三元组 / 六种特殊形态 / prelocking 占位符 / 生命周期）
- [表的通用操作](#表的通用操作)
  - [1. 开表：open_table](#1-开表open_table) / [开表的三个疑难机制](#开表的三个疑难机制)
  - [2. 锁表：lock_tables](#2-锁表lock_tables)（★ 两层锁模型 / `lock_tables`→`mysql_lock_tables` 四步 / `get_lock_data` 与 `lock_count` / 外部锁 / thr_lock 排序协议 / MDL 死锁检测与 victim 权重）
  - [3. MDL 体系](#3-mdl-体系)（★ 五个类职责 / 源码里的 scoped + object 兼容矩阵 / 11 种锁类型逐个用途 / 三种 duration / schema scoped IX）
  - [4. thr_lock 体系](#4-thr_lockserver-层引擎表锁体系)（★ `THR_LOCK` 四队列 / 12 种 `thr_lock_type` / `write_lock_count` 防读饿死）
  - [5. 闩锁全景](#5--闩锁mutex--latch全景)（★ `LOCK_open` 在 8.0 的语义变化 / `Table_cache::m_lock` 16 分区 / `LOCK_thd_data` / `COND_open` / `dict_sys->mutex` / 锁层次图与六条协议）
  - [3. CREATE TABLE](#3-create-table)（★ [`rea_create_base_table` 逐段剖析](#-rea_create_base_table上面那句里的写-dd--建-se-表到底怎么排的)：DD store ↔ SE 建表的编排顺序 / 中间提交 / binlog cache 选择）
  - [4. ALTER TABLE：算法决策的完整判据](#4-alter-table算法决策的完整判据)
  - [5. RENAME TABLE](#5-rename-table) / [6. TRUNCATE TABLE](#6-truncate-table) / [7. DROP TABLE](#7-drop-table) / [8. FLUSH TABLES / 缓存失效](#8-flush-tables--缓存失效)
- [列与行：本篇与 `record.md` 的边界](#列与行本篇与-recordmd-的边界)
- [`TABLE_SHARE` 与 `TABLE`：成员全景与生命周期（对象视角）](#table_share-与-table成员全景与生命周期对象视角)（对象视角：模板 vs 实例 / table_cache_key / 位图本体与指针 / key_info 深拷贝 / 缓存分层 / SHARE 三条死法 / version 机制）
  - ★ **成员覆盖 100%（105/105）与 100%（89/89）**——对标 `field.md` 的 97%。续篇 1.6–1.13（统计与直方图 / 外键与 CHECK / 二级引擎 / 临时表 handler / 键元数据 / 分区 / schema_read_only）与 2.6–2.13（JSON 部分更新 / 成本模型 / 触发器与 CHECK 运行态 / 迭代器 / 隐藏列 / HANDLER 与锁位置 / binlog 与失效标记）
- [表统计信息：★ 从引擎到优化器的搬运管道](#表统计信息-从引擎到优化器的搬运管道)（★ `ha_statistics` / `HA_STATUS_*` / `records_in_range` index dive / rec_per_key / `fetch_number_of_rows` / ANALYZE 与失效闭环）
- [表类型](#表类型)（`dd::enum_table_type` 四种 / `Table_ref` 判定矩阵 / 临时表 / I_S 与 system view / table function / CTE / 形态判别树）
- [Misc](#misc)（容易误解的概念 / 深挖补遗）
- [参考](#参考)

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
| 8.0.29 | instant ADD/DROP COLUMN（见 `../feat/ddl/ddl.md`） |

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

## 表元信息：DD 读取路径与两套缓存

> **源码坐标**：`[O1] sql/sql_base.cc`（`get_table_share` / `release_table_share`）· `[O2] sql/table.cc`（`alloc_table_share`）· `[O4] sql/dd_table_share.cc`（`open_table_def` 与六个 `fill_*`）· `[O5] sql/dd/impl/cache/dictionary_client.cc`（`Dictionary_client::acquire`）· `[O6] storage/innobase/handler/ha_innodb.cc`（`ha_innobase::open`）· `[O7] storage/innobase/dict/dict0dd.cc`（`dd_open_table`）· `[O8] storage/innobase/handler/ha_innopart.{h,cc}`（`Ha_innopart_share`）

### 1. `get_table_share()`：TDC 查找 + `m_open_in_progress` 占位 + `COND_open`

```cpp
// [O1] sql/sql_base.cc
TABLE_SHARE *get_table_share(THD *thd, const char *db, const char *table_name,
                             const char *key, size_t key_length, bool open_view,
                             bool open_secondary) {
  TABLE_SHARE *share;
  bool open_table_err = false;

  mysql_mutex_assert_owner(&LOCK_open);    // ★ 契约：进入时必须持有 LOCK_open
  // 能对表做任何操作的前提是已经持有某种 MDL
  assert(thd->mdl_context.owns_equal_or_stronger_lock(MDL_key::TABLE, db,
                                                      table_name, MDL_SHARED));

  for (;;) {
    auto it = table_def_cache->find(string(key, key_length));   // ★ TDC hash 查找
    if (it == table_def_cache->end()) {
      // ★ 要读 DD 就得先有 SCHEMA 上的 IX MDL；没有就放 LOCK_open 去拿，再重来
      if (thd->mdl_context.owns_equal_or_stronger_lock(
              MDL_key::SCHEMA, db, "", MDL_INTENTION_EXCLUSIVE)) {
        break;                             // 已有 IX，跳出循环去新建 share
      }
      mysql_mutex_unlock(&LOCK_open);      // ★★ 持 LOCK_open 时绝不碰 DD/MDL 等待
      if (dd::mdl_lock_schema(thd, db, MDL_TRANSACTION)) {
        mysql_mutex_lock(&LOCK_open);      // 恢复契约
        return nullptr;
      }
      mysql_mutex_lock(&LOCK_open);
      continue;                            // 重加后必须重查 hash
    }
    share = it->second.get();
    if (!share->m_open_in_progress)
      return process_found_table_share(thd, share, open_view);  // 命中且完整

    // ★ 别人正在打开这张表（正在读 DD）→ 等 COND_open；wait 会原子放 LOCK_open
    mysql_cond_wait(&COND_open, &LOCK_open);
    // 醒来后必须回到循环重查：share 可能已被打开失败者从 TDC 摘掉
  }

  if (!(share = alloc_table_share(db, table_name, key, key_length,
                                  open_secondary))) {
    return nullptr;
  }

  // ★ table_map_id 在 LOCK_open 保护下分配，且必须在 emplace 之前，
  //   保证从 cache 里读到的 share 一定有 id
  assign_new_table_id(share);
  table_def_cache->emplace(to_string(share->table_cache_key),
                           unique_ptr<TABLE_SHARE, Table_share_deleter>(share));

  /*
    ★ 必须在放 LOCK_open 之前 ++ref_count：否则 tdc_remove_table() /
      TABLE_SHARE::wait_for_old_version 可能把这个 share 删掉。
    ★ m_open_in_progress=true 表示「已占位但未填充完」。
  */
  share->increment_ref_count();      // Mark in use
  share->m_open_in_progress = true;  // Mark being opened

  mysql_mutex_unlock(&LOCK_open);    // ★★ 放锁读 DD

  {
    // 顺序很重要：schema 与 abstract_table 都要在 releaser 析构时按序释放/解锁
    dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
    const dd::Schema *sch = nullptr;
    const dd::Abstract_table *abstract_table = nullptr;
    open_table_err = true;                 // 先假设失败，简化下面分支
    if (thd->dd_client()->acquire(share->db.str, &sch) ||
        thd->dd_client()->acquire(share->db.str, share->table_name.str,
                                  &abstract_table)) {
    } else if (sch == nullptr)
      my_error(ER_BAD_DB_ERROR, MYF(0), share->db.str);
    else if (abstract_table == nullptr)
      my_error(ER_NO_SUCH_TABLE, MYF(0), share->db.str, share->table_name.str);
    else if (abstract_table->type() == dd::enum_table_type::USER_VIEW ||
             abstract_table->type() == dd::enum_table_type::SYSTEM_VIEW) {
      if (!open_view)                      // 想开表却撞上视图
        my_error(ER_NO_SUCH_TABLE, MYF(0), share->db.str,
                 share->table_name.str);
      else {
        share->is_view = true;
        const dd::View *tmp_view =
            dynamic_cast<const dd::View *>(abstract_table);
        share->view_object = tmp_view->clone();   // ★ 克隆一份视图对象挂在 share 上
        share->table_category =
            get_table_category(share->db, share->table_name);
        thd->status_var.opened_shares++;
        open_table_err = false;
      }
    } else {
      // ★ 从 DD 拿到了 dd::Table，用它填充 TABLE_SHARE
      assert(abstract_table->type() == dd::enum_table_type::BASE_TABLE);
      open_table_err = open_table_def(
          thd, share, *dynamic_cast<const dd::Table *>(abstract_table));

      if (!open_table_err) update_schema_options(sch, share);

      // 直方图也要在 LOCK_open 之外读：读直方图会递归开 mysql 表
      if (!open_table_err && read_histograms(thd, share, sch, abstract_table))
        open_table_err = true;
    }
  }

  mysql_mutex_lock(&LOCK_open);            // ★ 取回锁再改标志位
  share->m_open_in_progress = false;
  mysql_cond_broadcast(&COND_open);        // ★ 唤醒所有等这张表的人（成功/失败都要）

  if (open_table_err) {
    share->error = true;                   // 让等待者能识别「打开失败」而非「还在开」
    share->decrement_ref_count();
    table_def_cache->erase(to_string(share->table_cache_key));  // 从 TDC 摘掉
    return nullptr;
  }

  share->m_psi = PSI_TABLE_CALL(get_table_share)(
      (share->tmp_table != NO_TMP_TABLE), share);
  return share;
}
```

`ref_count` 的对称面（决定 SHARE 何时真正消失）：

```cpp
// [O1] sql/sql_base.cc
void release_table_share(TABLE_SHARE *share) {
  mysql_mutex_assert_owner(&LOCK_open);    // ★ 同样要求持有 LOCK_open
  assert(share->ref_count() != 0);
  if (share->decrement_ref_count() == 0) {
    if (share->has_old_version() || table_def_shutdown_in_progress)
      table_def_cache->erase(to_string(share->table_cache_key));  // 旧版本直接删
    else {
      /* 挂到 unused share LRU 的尾部（最近使用） */
      share->prev = end_of_unused_share.prev;
      *end_of_unused_share.prev = share;
      end_of_unused_share.prev = &share->next;
      share->next = &end_of_unused_share;

      if (table_def_cache->size() > table_def_size) {
        // ★ TDC 上限由 table_definition_cache 决定，按 LRU 淘汰最久未用
        table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
      }
    }
  }
}
```

`alloc_table_share()` 只做「骨架」——**不**填任何 DD 内容，也不参与 TDC 淘汰：

```cpp
// [O2] sql/table.cc
    new (share) TABLE_SHARE(refresh_version, open_secondary);  // ★ version = 当前刷新号
    share->set_table_cache_key(key_buff, key, key_length);
    share->path.str = path_buff;
    share->normalized_path.str = share->path.str;
    ...
    /*
      alloc_table_share() 可以在无锁情况下被调用（如 ha_create_table...），
      所以这里不分配 table_map_id，改用一个不可能用到的值；真正的 id
      在 open_table() 里于 LOCK_open 保护下分配。
    */
    share->table_map_id = ~0ULL;
    share->cached_row_logging_check = -1;

    memset(cache_element_array, 0,
           table_cache_instances * sizeof(*cache_element_array));
    share->cache_element = cache_element_array;   // ★ 每个 TC 分区一个 element 槽
    share->mem_root = std::move(mem_root);
    mysql_mutex_init(key_TABLE_SHARE_LOCK_ha_data, &share->LOCK_ha_data,
                     MY_MUTEX_INIT_FAST);         // ★ 引擎侧 share 的保护锁
```

### 2. `open_table_def()` → 六个 `fill_*`（+ `prepare_share()` 收尾）

> 补充坐标：`[O9] sql/default_values.cc`（`find_record_length` / `prepare_default_value_buffer_and_table_share`）· `[O10] sql/table.cc`（`create_key_part_field_with_prefix_length` / `open_table_from_share`）

#### 2.0 入口：**顺序就是依赖顺序**

```cpp
// [O4] sql/dd_table_share.cc:2290
// 填充table_share。从dd::Table导入table meta data.
bool open_table_def(THD *thd, TABLE_SHARE *share, const dd::Table &table_def) {
  DBUG_TRACE;

  MEM_ROOT *old_root = thd->mem_root;
  thd->mem_root = &share->mem_root;  // Needed for make_field()++
  // ★ 把 thd 的 mem_root 临时换成 share 的：后面 make_field() / lex_string_strmake()
  //   分配出来的 Field、列名、DEFAULT 值缓冲全部**落在 share->mem_root**，
  //   于是随 share 一起生灭，不需要逐个 free。函数返回前必须换回去。
  share->blob_fields = 0;            // HACK

  // Fill the TABLE_SHARE with details.
  // ★ 六个 fill 严格按序，顺序不能换：
  //   表级（含记录长度/default_values）→ 列（造 Field）→ 索引（key_part->field 要用上一步
  //   的 Field*）→ 分区（要 m_part_info 决定 handler 是否 partitioned）→ 外键 → CHECK
  bool error = (fill_share_from_dd(thd, share, &table_def) ||
                fill_columns_from_dd(thd, share, &table_def) ||
                fill_indexes_from_dd(thd, share, &table_def) ||
                fill_partitioning_from_dd(thd, share, &table_def) ||
                fill_foreign_keys_from_dd(share, &table_def) ||
                fill_check_constraints_from_dd(share, &table_def));

  thd->mem_root = old_root;

  // ★ 收尾：临时造一个 handler 问引擎能力，再算 primary_key / key_length /
  //   column_bitmap_size / blob_field / all_set
  if (!error) error = prepare_share(thd, share, &table_def);

  if (!error) {
    share->table_category = get_table_category(share->db, share->table_name);
    thd->status_var.opened_shares++;   // ★ Opened_table_definitions 状态量的来源
    return false;
  }
  return true;
}
```

调用点只有一处（`get_table_share()` 里，`[O1] sql/sql_base.cc`）：DD 里取到的是**视图**就走 `share->view_object = tmp_view->clone()` 分支，只有 `dd::enum_table_type::BASE_TABLE` 才进 `open_table_def()`。

#### 2.1 ★ `fill_share_from_dd()`：表级属性 + **记录布局的"定长三件套"**

```cpp
// [O4] sql/dd_table_share.cc:563
/** Fill TABLE_SHARE from dd::Table object */
static bool fill_share_from_dd(THD *thd, TABLE_SHARE *share,
                               const dd::Table *tab_obj) {
  const dd::Properties &table_options = tab_obj->options();   // ★ DD 的 options 是个 KV 串
  ...
  // Read table engine type
  LEX_CSTRING engine_name = lex_cstring_handle(tab_obj->engine());
  ...
  plugin_ref tmp_plugin = ha_resolve_by_name_raw(thd, engine_name);
  if (tmp_plugin) {
    ...
    plugin_unlock(nullptr, share->db_plugin);
    share->db_plugin = my_plugin_lock(nullptr, &tmp_plugin);  // ★ db_plugin / db_type
    // ★ 注意：plugin 是 **锁住** 的（my_plugin_lock），TABLE_SHARE 活着期间引擎插件不能被卸载
  } else {
    my_error(ER_UNKNOWN_STORAGE_ENGINE, MYF(0), engine_name.str);
    return true;
  }
  ...
  share->db_low_byte_first = true;   // 先给个默认值，后面 prepare_share() 由 handler 改写
  ...
  share->db_create_options = 0;
  table_options.get("pack_record", &bool_opt);
  if (bool_opt) share->db_create_options |= HA_OPTION_PACK_RECORD;
  ...
  share->db_options_in_use = share->db_create_options;
  ...
  // Collation ID
  share->table_charset = dd_get_mysql_charset(tab_obj->collation_id());
  ...
  // Row type. First one really used by the storage engine.
  share->real_row_type = dd_get_old_row_format(tab_obj->row_format());
  ...
  share->mysql_version = tab_obj->mysql_version_id();   // ★ 建表时的 server 版本（兼容性判断用）
  ...
  table_options.get("key_block_size", &share->key_block_size);

  // Prepare the default_value buffer.
  // ★★ 这一步才是"记录布局"的真正来源：fields / reclength / null_bytes /
  //     default_values / rec_buff_length / stored_rec_length 全部在这里落地
  if (prepare_default_value_buffer_and_table_share(thd, *tab_obj, share))
    return true;
  ...
  // Read tablespace name
  if (fill_tablespace_from_dd(thd, share, tab_obj)) return true;
  ...
```

`share->fields` / `reclength` / `null_bytes` 的计算 —— `[O9] sql/default_values.cc:101`：

```cpp
static bool find_record_length(const dd::Table &table, size_t min_length,
                               TABLE_SHARE *share) {
  // Get the table property 'pack_record' and initialize out parameters.
  bool pack_record;
  if (table.options().get("pack_record", &pack_record)) return true;

  assert(share);
  share->fields = 0;          // ★ 列数在这里被填（只数"用户可见 + SE 非隐藏"列）
  share->null_fields = 0;
  share->reclength = 0;
  ulong leftover_bits = pack_record ? 0 : 1;
  // ★ 非 pack_record 时第 0 个 bit 被"占掉"，因为它同时兼作"记录已被删除"标志

  // Loop over columns, count nullable and bit fields and find record length.
  for (const dd::Column *col_obj : table.columns()) {
    // Skip hidden columns
    if (col_obj->is_se_hidden()) continue;   // ★ 例如 InnoDB 的 DB_ROW_ID/DB_TRX_ID/DB_ROLL_PTR
                                            //   以及 FTS_DOC_ID，都不算 MySQL 的列

    // Check if the field may be NULL.
    if (col_obj->is_nullable()) share->null_fields++;
    ...
    // Increment record length.
    share->reclength += column_pack_length(*col_obj);
    share->fields++;
  }

  // Find preamble length and add it to the total record length.
  share->null_bytes = (share->null_fields + leftover_bits + 7) / 8;
  share->last_null_bit_pos = (share->null_fields + leftover_bits) & 7;
  share->reclength += share->null_bytes;   // ★ null 位图也算进 reclength

  // Hack to avoid bugs with small static rows in MySQL.
  share->reclength = std::max<size_t>(min_length, share->reclength);
  share->stored_rec_length = share->reclength;
  // ★ stored_rec_length 之后会被 fill_columns_from_dd() 的"第二趟"修正：
  //   虚拟生成列排在记录末尾，不计入 stored 部分

  return false;
}
```

`default_values` 与 `rec_buff_length` —— `[O9] sql/default_values.cc:263`：

```cpp
bool prepare_default_value_buffer_and_table_share(THD *thd,
                                                  const dd::Table &table,
                                                  TABLE_SHARE *share) {
  ...
  handler *file = nullptr;
  handlerton *engine = share->db_type();
  // ★ 先临时造一个 handler，只为问两件事：min_record_length() 与 extra_rec_buf_length()
  if (!(file = get_new_handler(nullptr,
                               table.partition_type() != dd::Table::PT_NONE,
                               thd->mem_root, engine))) { ... }

  // Get the minimal and extra record buffer lengths from the handler.
  size_t extra_length = file->extra_rec_buf_length();
  size_t min_length =
      static_cast<size_t>(file->min_record_length(share->db_create_options));
  destroy(file);     // ★ 用完立刻销毁：这里不 ha_open，只是借 handler 查能力

  // Get the number of columns, record length etc.
  if (find_record_length(table, min_length, share)) return true;

  // Adjust buffer size and allocate the default value buffer.
  share->rec_buff_length = ALIGN_SIZE(share->reclength + 1 + extra_length);
  //                                                    ↑ ★ 多 1 字节：Field::ptr 指向
  //                                                      record[0]-1，第 0 字节留给 null 标志
  if (share->reclength) {
    // ★★ share->default_values = 整张表的"模板行"。
    //    Field 的 ptr 以它为基准；TABLE 侧 clone 出来的 Field 只是整体平移一个 offset。
    share->default_values = reinterpret_cast<uchar *>(
        share->mem_root.Alloc(share->rec_buff_length));
    if (!share->default_values) return true;

    memset(share->default_values, 0, share->reclength);
    ...
    set_pack_record_and_unused_preamble_bits(
        (share->db_create_options & HA_OPTION_PACK_RECORD), preamble_bits,
        share->default_values);
  }

  return false;
}
```

⇒ **一句话**：`default_values` 是 `TABLE_SHARE` 里那一行"标准答案"，后续 `TABLE::record[0]` 只是它的副本，`Field::ptr` 的偏移体系全部由它定义。

#### 2.2 ★ `fill_columns_from_dd()`：**两趟扫描**（存储列先，虚拟生成列后）

```cpp
// [O4] sql/dd_table_share.cc:1113
static bool fill_columns_from_dd(THD *thd, TABLE_SHARE *share,
                                 const dd::Table *tab_obj) {
  // Allocate space for fields in TABLE_SHARE.
  uint fields_size = ((share->fields + 1) * sizeof(Field *));
  share->field = (Field **)share->mem_root.Alloc((uint)fields_size);
  memset(share->field, 0, fields_size);   // ★ 多分配 1 个 slot 作 NULL 终止符
  share->vfields = 0;
  share->gen_def_field_count = 0;

  // Iterate through all the columns.
  uchar *null_flags [[maybe_unused]];
  uchar *null_pos, *rec_pos;
  null_flags = null_pos = share->default_values;          // ★ null 位区从模板行头开始
  rec_pos = share->default_values + share->null_bytes;    // ★ 数据区紧跟在 null 位区后面
  uint null_bit_pos =
      (share->db_create_options & HA_OPTION_PACK_RECORD) ? 0 : 1;
  uint field_nr = 0;
  bool has_vgc = false;
  for (const dd::Column *col_obj : tab_obj->columns()) {
    // Skip hidden columns
    if (col_obj->is_se_hidden()) continue;

    /*
      Fill details of each column.

      Skip virtual generated columns at this point. They reside at the end of
      the record, so we need to do separate pass, to evaluate their offsets
      correctly.
    */
    // ★ 第一趟：只放"真实存储"的列
    if (!col_obj->is_virtual()) {
      if (fill_column_from_dd(thd, share, col_obj, null_pos, null_bit_pos,
                              rec_pos, field_nr))
        return true;

      if ((null_bit_pos += column_preamble_bits(col_obj)) > 7) {
        null_pos++;
        null_bit_pos -= 8;
      }

      rec_pos += share->field[field_nr]->pack_length_in_rec();
    } else
      has_vgc = true;

    field_nr++;
  }

  if (has_vgc) {
    /*
      Additional pass to put virtual generated columns at the end of the
      record is required.
    */
    // ★ 第二趟：虚拟生成列追加到记录末尾，并把"存储部分"的长度截断在这里
    if (share->stored_rec_length >
        static_cast<ulong>(rec_pos - share->default_values))
      share->stored_rec_length = (rec_pos - share->default_values);

    field_nr = 0;

    for (const dd::Column *col_obj2 : tab_obj->columns()) {
      if (col_obj2->is_se_hidden()) continue;

      if (col_obj2->is_virtual()) {
        if (fill_column_from_dd(thd, share, col_obj2, null_pos, null_bit_pos,
                                rec_pos, field_nr))
          return true;

        if ((null_bit_pos += column_preamble_bits(col_obj2)) > 7) {
          null_pos++;
          null_bit_pos -= 8;
        }

        rec_pos += share->field[field_nr]->pack_length_in_rec();
      }

      field_nr++;
    }
  }

  // Make sure the scan of the columns is consistent with other data.
  // ★ 三个 assert 就是"两趟扫描"的自证：null 位区算出来的长度必须与 2.1 的
  //   find_record_length() 完全一致，否则说明列定义与记录布局推导不一致
  assert(share->null_bytes == (null_pos - null_flags + (null_bit_pos + 7) / 8));
  assert(share->last_null_bit_pos == null_bit_pos);
  assert(share->fields == field_nr);

  return (false);
}
```

`fill_column_from_dd()` 最终落到 `make_field(...)`（`[O4] sql/dd_table_share.cc:807`），由 `dd::Column` 的 `type()`/`char_length()`/`is_nullable()`/`default_value()` 造出具体的 `Field_*` 子类，并把默认值**写进** `share->default_values`（这是 `default_values` 名字的由来）。

⇒ **易错点**：虚拟生成列位于记录末尾，因此 `stored_rec_length ≤ reclength`；读一行时 InnoDB 只回填前 `stored_rec_length` 字节，虚拟列的值由 `TABLE` 侧的 `Item` 现算（`unpack_value_generator()`）。

#### 2.3 ★ `fill_indexes_from_dd()`：`keys` / `key_parts` / `rec_per_key` 的**唯一分配点**

```cpp
// [O4] sql/dd_table_share.cc:1465
static bool fill_indexes_from_dd(THD *thd, TABLE_SHARE *share,
                                 const dd::Table *tab_obj) {
  share->keys_for_keyread.init(0);
  share->keys_in_use.init();
  share->visible_indexes.init();

  uint32 primary_key_parts = 0;

  bool use_extended_sk = ha_check_storage_engine_flag(
      share->db_type(), HTON_SUPPORTS_EXTENDED_KEYS);
  // ★ InnoDB 支持"二级索引自动追加主键列"（extended key），所以 key_part 数要多算

  // Count number of keys and total number of key parts in the table.
  assert(share->keys == 0 && share->key_parts == 0);

  for (const dd::Index *idx_obj : tab_obj->indexes()) {
    // Skip hidden indexes
    if (idx_obj->is_hidden()) continue;     // ★ 用于全文索引倒排表等，不对 SQL 暴露

    share->keys++;                          // ★ share->keys
    uint key_parts = 0;
    for (const dd::Index_element *idx_ele : idx_obj->elements()) {
      if (!idx_ele->is_hidden()) key_parts++;
    }
    share->key_parts += key_parts;          // ★ share->key_parts

    // Primary key (or candidate key replacing it) is always first if exists.
    if (idx_obj->ordinal_position() == 1) primary_key_parts = key_parts;
  }

  // Allocate and fill KEY objects.
  if (share->keys) {
    KEY_PART_INFO *key_part;
    ulong *rec_per_key;
    rec_per_key_t *rec_per_key_float;
    uint total_key_parts = share->key_parts;

    if (use_extended_sk)
      total_key_parts += (primary_key_parts * (share->keys - 1));
    // ★ 每个二级索引尾部追加 primary_key_parts 个主键列 ⇒ 多 (keys-1)×pk_parts

    //
    // Alloc rec_per_key buffer
    //
    if (!(rec_per_key =
              (ulong *)share->mem_root.Alloc(total_key_parts * sizeof(ulong))))
      return true; /* purecov: inspected */

    if (!(rec_per_key_float = (rec_per_key_t *)share->mem_root.Alloc(
              total_key_parts * sizeof(rec_per_key_t))))
      return true; /* purecov: inspected */

    //
    // Alloc buffers to hold keys and key_parts
    //
    if (!(share->key_info = share->mem_root.ArrayAlloc<KEY>(share->keys)))
      return true; /* purecov: inspected */

    if (!(key_part =
              share->mem_root.ArrayAlloc<KEY_PART_INFO>(total_key_parts)))
      return true; /* purecov: inspected */

    //
    // Alloc buffer to hold keynames
    //
    if (!(share->keynames.type_names = (const char **)share->mem_root.Alloc(
              (share->keys + 1) * sizeof(char *))))
      return true; /* purecov: inspected */
    memset(share->keynames.type_names, 0, ((share->keys + 1) * sizeof(char *)));

    share->keynames.type_names[share->keys] = nullptr;
    share->keynames.count = share->keys;

    const dd::Index *index_at_pos[MAX_INDEXES];
    uint key_nr = 0;
    for (const dd::Index *idx_obj : tab_obj->indexes()) {
      if (idx_obj->is_hidden()) continue;

      if (fill_index_from_dd(thd, share, idx_obj, key_nr)) return true;

      index_at_pos[key_nr] = idx_obj;

      share->keys_in_use.set_bit(key_nr);

      if (idx_obj->is_visible() && is_spatial_index_usable(*idx_obj))
        share->visible_indexes.set_bit(key_nr);   // ★ 8.0 的 INVISIBLE INDEX

      key_nr++;
    }

    // Update keyparts now
    key_nr = 0;
    do {
      // Assign the key_part_info buffer
      KEY *keyinfo = &share->key_info[key_nr];
      keyinfo->key_part = key_part;
      // ★★ rec_per_key 是"切片"：每个 KEY 拿走大数组里属于自己的那一段。
      //    open_table_from_share() 里 TABLE::key_info 是 memcpy 出来的，
      //    这两个指针被原样拷过去 ⇒ **所有 TABLE 实例与 SHARE 共用同一份统计**。
      keyinfo->set_rec_per_key_array(rec_per_key, rec_per_key_float);
      keyinfo->set_in_memory_estimate(IN_MEMORY_ESTIMATE_UNKNOWN);

      fill_index_elements_from_dd(share, index_at_pos[key_nr], key_nr);

      key_part += keyinfo->user_defined_key_parts;
      rec_per_key += keyinfo->user_defined_key_parts;
      rec_per_key_float += keyinfo->user_defined_key_parts;

      /*
        Add PK parts if engine supports PK extension for secondary keys.
      */
      keyinfo->actual_key_parts = keyinfo->user_defined_key_parts;
      if (use_extended_sk && key_nr && !(keyinfo->flags & HA_NOSAME)) {
        keyinfo->unused_key_parts = primary_key_parts;
        key_part += primary_key_parts;
        rec_per_key += primary_key_parts;
        rec_per_key_float += primary_key_parts;
        share->key_parts += primary_key_parts;
      }

      // Initialize the rec per key arrays
      for (uint kp = 0; kp < keyinfo->actual_key_parts; ++kp) {
        keyinfo->rec_per_key[kp] = 0;
        keyinfo->set_records_per_key(kp, REC_PER_KEY_UNKNOWN);
        // ★ 初值是 UNKNOWN，不是 0——优化器第一次访问时才会去 handler::info() 拉
      }

      key_nr++;
    } while (key_nr < share->keys);
  }

  return (false);
}
```

> **勘误**：`share->keys_for_parts` 在 8.0.39 中**已不存在**（5.6 遗留成员，全仓库 0 匹配）。对应语义由 `KEY::user_defined_key_parts` / `actual_key_parts` / `unused_key_parts` 三个成员承担：`user_defined` 是 DDL 里写出来的部分，`unused` 是引擎自动追加的主键列，`actual = user_defined`（遍历时按 `actual_key_parts` 走，`unused_key_parts` 靠 `key_part += ` 跳过）。

#### 2.4 `fill_partitioning_from_dd()`：分区表达式被"反生成成字符串"

```cpp
// [O4] sql/dd_table_share.cc:1906（结尾段）
static bool fill_partitioning_from_dd(THD *thd, TABLE_SHARE *share,
                                      const dd::Table *tab_obj) {
  if (tab_obj->partition_type() == dd::Table::PT_NONE) return false;
  ...
  partition_info *part_info;
  part_info = new (&share->mem_root) partition_info;   // ★ 落在 share mem_root
  ...
  char *buf;
  uint buf_len;

  // Turn off ANSI_QUOTES and other SQL modes which affect printing of
  // generated partitioning clause.
  Sql_mode_parse_guard parse_guard(thd);

  // ★ 把解析好的分区信息"打印"回 SQL 子句文本
  buf = generate_partition_syntax(part_info, &buf_len, true, true, false,
                                  nullptr);
  if (!buf) return true;

  // ★ 存进 share，之后每个 TABLE 在 open_table_from_share() 里用
  //   unpack_partition_info() 重新 parse 成 Item 树 ⇒ 分区表达式是 **每 TABLE 一份 Item**
  share->partition_info_str = strmake_root(&share->mem_root, buf, buf_len);
  if (!share->partition_info_str) return true;

  share->partition_info_str_len = buf_len;
  share->m_part_info = part_info;   // ★ 决定 get_new_handler(..., partitioned=true)
  return (false);
}
```

⇒ 这个"对象 → 文本 → 再 parse"的往返看着浪费，实则是**共享与私有的分界**：`partition_info` 骨架在 SHARE 上共享，但表达式里的 `Item` 是有状态的（绑定 THD、有 `fixed` 标志），必须每 `TABLE` 一份。

#### 2.5 `fill_foreign_keys_from_dd()`：只拷"名字 + 规则"，不解析被引用表

```cpp
// [O4] sql/dd_table_share.cc:2155
static bool fill_foreign_keys_from_dd(TABLE_SHARE *share,
                                      const dd::Table *tab_obj) {
  assert(share->foreign_keys == 0 && share->foreign_key_parents == 0);

  share->foreign_keys = tab_obj->foreign_keys().size();
  share->foreign_key_parents = tab_obj->foreign_key_parents().size();
  // ★ 两个方向都记：foreign_keys = 本表作为子表；foreign_key_parents = 本表作为父表

  if (share->foreign_keys) {
    if (!(share->foreign_key =
              (TABLE_SHARE_FOREIGN_KEY_INFO *)share->mem_root.Alloc(
                  share->foreign_keys * sizeof(TABLE_SHARE_FOREIGN_KEY_INFO))))
      return true;

    uint i = 0;
    for (const dd::Foreign_key *fk : tab_obj->foreign_keys()) {
      if (lex_string_strmake(&share->mem_root,
                             &share->foreign_key[i].referenced_table_db,
                             fk->referenced_table_schema_name().c_str(),
                             fk->referenced_table_schema_name().length()))
        return true;
      ...
      share->foreign_key[i].update_rule = fk->update_rule();
      share->foreign_key[i].delete_rule = fk->delete_rule();
      share->foreign_key[i].columns = fk->elements().size();
      ...
      for (const dd::Foreign_key_element *fk_el : fk->elements()) {
        if (lex_string_strmake(&share->mem_root,
                               &share->foreign_key[i].column_name[j],
                               fk_el->column().name().c_str(),
                               fk_el->column().name().length()))
          return true;
        ++j;
      }
      ++i;
    }
  }
  ...
```

⇒ **为什么只存字符串**：SHARE 层只需要"有没有外键、涉及哪些列"以便决定 prelocking 要连带锁哪些表；真正的父表解析（`dict_foreign_t`、`dict_table_t::foreign_set`）由 InnoDB 在 `dd_open_table()` 里做，那里有 `dict_sys->mutex` 保护，可以安全建立对象图。

#### 2.6 `fill_check_constraints_from_dd()`：只存表达式**文本**

```cpp
// [O4] sql/dd_table_share.cc:2250
static bool fill_check_constraints_from_dd(TABLE_SHARE *share,
                                           const dd::Table *tab_obj) {
  assert(share->check_constraint_share_list == nullptr);

  if (tab_obj->check_constraints().size() > 0) {
    share->check_constraint_share_list = new (&share->mem_root)
        Sql_check_constraint_share_list(&share->mem_root);
    if (share->check_constraint_share_list == nullptr) return true;  // OOM

    if (share->check_constraint_share_list->reserve(
            tab_obj->check_constraints().size()))
      return true;  // OOM

    for (auto &cc : tab_obj->check_constraints()) {
      // Check constraint name.
      LEX_CSTRING name;
      if (lex_string_strmake(&share->mem_root, &name, cc->name().c_str(),
                             cc->name().length()))
        return true;  // OOM

      // Check constraint expression (clause).
      LEX_CSTRING check_clause;
      if (lex_string_strmake(&share->mem_root, &check_clause,
                             cc->check_clause().c_str(),
                             cc->check_clause().length()))
        return true;  // OOM

      // Check constraint state.
      bool is_cc_enforced =
          (cc->constraint_state() == dd::Check_constraint::CS_ENFORCED);

      share->check_constraint_share_list->push_back(
          Sql_check_constraint_share(name, check_clause, is_cc_enforced));
    }
  }

  return false;
}
```

⇒ SHARE 侧只有"名字 + 文本 + 是否 ENFORCED"三元组；`Item` 树在 `open_table_from_share()` 里由 `unpack_value_generator(..., VGS_CHECK_CONSTRAINT, ...)` per-`TABLE` 解析，落进 `TABLE::table_check_constraint_list`。

#### 2.7 ★ `prepare_share()`：临时 handler、主键定位、`column_bitmap_size`

```cpp
// [O4] sql/dd_table_share.cc:261
static bool prepare_share(THD *thd, TABLE_SHARE *share,
                          const dd::Table *table_def) {
  my_bitmap_map *bitmaps;
  handler *handler_file = nullptr;

  // Mark 'system' tables (tables with one row) to help the Optimizer.
  share->system =
      ((share->max_rows == 1) && (share->min_rows == 1) && (share->keys == 0));
  ...
  /* Allocate handler */
  // ★ 临时 handler：只为拿 table_flags / low_byte_first / 走一遍 key_part 修正
  if (!(handler_file = get_new_handler(share, (share->m_part_info != nullptr),
                                       &share->mem_root, share->db_type()))) {
    ...
  }
  if (handler_file->set_ha_share_ref(&share->ha_share)) { ... }
  // ★★ 就是这一步把 handler 的 ha_share 指针钉到 &share->ha_share 上，
  //     后面 open_table_from_share() 与 ha_innobase::open() 都从这里取 SE 侧 share
  share->db_low_byte_first = handler_file->low_byte_first();

  /* Fix key->name and key_part->field */
  if (share->keys) {
    ...
    uint primary_key = (uint)(
        find_type(primary_key_name, &share->keynames, FIND_TYPE_NO_PREFIX) - 1);
    ...
    for (uint key = 0; key < share->keys; key++, keyinfo++) {
      ...
      keyinfo->name = share->keynames.type_names[key];
      ...
      for (uint i = 0; i < keyinfo->user_defined_key_parts; key_part++, i++) {
        ...
        Field *field = key_part->field;

        key_part->type = field->key_type();
        if (field->is_nullable()) {
          key_part->null_offset = field->null_offset(share->default_values);
          key_part->null_bit = field->null_bit;
          key_part->store_length += HA_KEY_NULL_LENGTH;
          keyinfo->flags |= HA_NULL_PART_KEY;
          keyinfo->key_length += HA_KEY_NULL_LENGTH;
        }
        if (field->type() == MYSQL_TYPE_BLOB ||
            field->real_type() == MYSQL_TYPE_VARCHAR ||
            field->type() == MYSQL_TYPE_GEOMETRY) {
          key_part->store_length += HA_KEY_BLOB_LENGTH;
          if (i + 1 <= keyinfo->user_defined_key_parts)
            keyinfo->key_length += HA_KEY_BLOB_LENGTH;
        }
        key_part->init_flags();

        if (field->is_virtual_gcol()) keyinfo->flags |= HA_VIRTUAL_GEN_KEY;

        setup_key_part_field(share, handler_file, primary_key, keyinfo, key, i,
                             &usable_parts, true);
        ...
        if (key_part->length != field->key_length()) {
          ... key_part->key_part_flag |= HA_PART_KEY_SEG;   // ★ 前缀索引标记
        }
        ...
      }
      keyinfo->actual_flags = keyinfo->flags;
      ...
      keyinfo->usable_key_parts = usable_parts;  // Filesort
      share->max_key_length = std::max(share->max_key_length,
          keyinfo->key_length + keyinfo->user_defined_key_parts);
      share->total_key_length += keyinfo->key_length;
      ...
    }
    if (primary_key < MAX_KEY && (share->keys_in_use.is_set(primary_key))) {
      share->primary_key = primary_key;
      ...
    } else
      share->primary_key = MAX_KEY;  // we do not have a primary key
  } else
    share->primary_key = MAX_KEY;
  destroy(handler_file);   // ★ 用完即毁，不 ha_open

  if (share->found_next_number_field) {
    Field *reg_field = *share->found_next_number_field;
    /* Check that the auto-increment column is the first column of some key. */
    if ((int)(share->next_number_index = (uint)find_ref_key(
                  share->key_info, share->keys, share->default_values,
                  reg_field, &share->next_number_key_offset,
                  &share->next_number_keypart)) < 0) {
      ...   // ★ AUTO_INCREMENT 列必须是某个索引的第一列，否则报错
    } else
      reg_field->set_flag(AUTO_INCREMENT_FLAG);
  }

  if (share->blob_fields) {
    Field **ptr;
    uint k, *save;

    /* Store offsets to blob fields to find them fast */
    if (!(share->blob_field = save = (uint *)share->mem_root.Alloc(
              (uint)(share->blob_fields * sizeof(uint)))))
      return true;  // OOM error message already reported
    for (k = 0, ptr = share->field; *ptr; ptr++, k++) {
      if ((*ptr)->is_flag_set(BLOB_FLAG) || (*ptr)->is_array()) (*save++) = k;
    }
  }

  // ★★ TABLE 侧那 8 张位图的大小就是这里算出来的
  share->column_bitmap_size = bitmap_buffer_size(share->fields);
  if (!(bitmaps = (my_bitmap_map *)share->mem_root.Alloc(
            share->column_bitmap_size))) {
    return true; /* purecov: inspected */
  }
  bitmap_init(&share->all_set, bitmaps, share->fields);
  bitmap_set_all(&share->all_set);

  return false;
}
```

#### 2.8 `dd::Table` → `TABLE_SHARE` 字段映射速查

| `dd::Table` / `dd::Column` / `dd::Index` | `TABLE_SHARE` 成员 | 填充处 |
|---|---|---|
| `engine()` | `db_plugin`（`handlerton`，**加 plugin 锁**） | `fill_share_from_dd` |
| `options().get("pack_record"/"pack_keys"/"checksum"/…)` | `db_create_options` / `db_options_in_use` | `fill_share_from_dd` |
| `collation_id()` | `table_charset` | `fill_share_from_dd` |
| `row_format()` | `real_row_type` | `fill_share_from_dd` |
| `mysql_version_id()` | `mysql_version` | `fill_share_from_dd` |
| **columns（非 `is_se_hidden`）** | `fields` / `null_fields` / `reclength` / `null_bytes` / `stored_rec_length` / `rec_buff_length` / `default_values` | `find_record_length` + `prepare_default_value_buffer_and_table_share` |
| 每个 `dd::Column` | `field[]`（`Field*`）/ `vfields` / `gen_def_field_count` / `blob_fields` | `fill_columns_from_dd` |
| `indexes()`（非 hidden） | `keys` / `key_parts` / `key_info` / `keynames` / `keys_in_use` / `visible_indexes` | `fill_indexes_from_dd` |
| `dd::Index_element` | `KEY_PART_INFO::length` / `store_length` / `fieldnr` / `field` / `offset` | `fill_index_elements_from_dd` |
| `partition_type()` / `partition_expression()` | `m_part_info` / `partition_info_str` / `auto_partitioned` | `fill_partitioning_from_dd` |
| `foreign_keys()` / `foreign_key_parents()` | `foreign_key[]` / `foreign_key_parent[]`（**只存名字**） | `fill_foreign_keys_from_dd` |
| `check_constraints()` | `check_constraint_share_list`（**只存文本**） | `fill_check_constraints_from_dd` |
| （后处理） | `primary_key` / `keys_for_keyread` / `max_key_length` / `total_key_length` / `column_bitmap_size` / `all_set` / `blob_field` / `system` | `prepare_share` |

#### 2.9 六个 `fill_*` 落到哪一组（与「成员全景」分组对应，速查）

```cpp
// [O4] fill_share_from_dd() —— 表级选项组 / 引擎组
  const dd::Properties &table_options = tab_obj->options();
  if (table_options.exists("secondary_engine")) {
    table_options.get("secondary_engine", &share->secondary_engine, &share->mem_root);
  }
  LEX_CSTRING engine_name = lex_cstring_handle(tab_obj->engine());   // ★ 引擎名
  plugin_ref tmp_plugin = ha_resolve_by_name_raw(thd, engine_name);
  if (tmp_plugin) {
    plugin_unlock(nullptr, share->db_plugin);
    share->db_plugin = my_plugin_lock(nullptr, &tmp_plugin);  // ★ db_plugin / db_type
  } else {
    my_error(ER_UNKNOWN_STORAGE_ENGINE, MYF(0), engine_name.str);
    return true;
  }
  table_options.get("max_rows", &share->max_rows);       // ★ 统计/选项组
  table_options.get("min_rows", &share->min_rows);
  share->table_charset = dd_get_mysql_charset(tab_obj->collation_id());  // ★ 字符集组

// [O4] fill_columns_from_dd() —— 列/记录布局组
  share->field = (Field **)share->mem_root.Alloc((uint)fields_size);
  null_flags = null_pos = share->default_values;         // ★ 默认行缓冲区
  rec_pos = share->default_values + share->null_bytes;
  uint null_bit_pos = (share->db_create_options & HA_OPTION_PACK_RECORD) ? 0 : 1;
  for (const dd::Column *col_obj : tab_obj->columns()) {
    if (col_obj->is_se_hidden()) continue;               // ★ 跳过引擎隐藏列
    // ★ 虚拟生成列留到第二趟：它们必须排在记录末尾，否则偏移算错
    if (!col_obj->is_virtual()) {
      if (fill_column_from_dd(thd, share, col_obj, null_pos, null_bit_pos,
                              rec_pos, field_nr))
        return true;
      if ((null_bit_pos += column_preamble_bits(col_obj)) > 7) {
        null_pos++;
        null_bit_pos -= 8;
      }
      rec_pos += share->field[field_nr]->pack_length_in_rec();
    } else
      has_vgc = true;
    field_nr++;
  }
  /* … 第二趟填虚拟列 … */

// [O4] fill_indexes_from_dd() —— 键与索引组 / 统计组
  bool use_extended_sk = ha_check_storage_engine_flag(
      share->db_type(), HTON_SUPPORTS_EXTENDED_KEYS);   // ★ 二级索引自动追加 PK 后缀
  for (const dd::Index *idx_obj : tab_obj->indexes()) {
    if (idx_obj->is_hidden()) continue;
    share->keys++;                                       // ★ keys / key_parts 计数
    for (const dd::Index_element *idx_ele : idx_obj->elements())
      if (!idx_ele->is_hidden()) share->key_parts++;
  }
  uint total_key_parts = share->key_parts;
  if (use_extended_sk)
    total_key_parts += (primary_key_parts * (share->keys - 1));
  rec_per_key = (ulong *)share->mem_root.Alloc(total_key_parts * sizeof(ulong));
  rec_per_key_float = (rec_per_key_t *)share->mem_root.Alloc(
      total_key_parts * sizeof(rec_per_key_t));           // ★ 后面被 TABLE 浅共享
  share->key_info = share->mem_root.ArrayAlloc<KEY>(share->keys);
  key_part = share->mem_root.ArrayAlloc<KEY_PART_INFO>(total_key_parts);
  share->keynames.type_names =
      (const char **)share->mem_root.Alloc((share->keys + 1) * sizeof(char *));

// [O4] fill_partitioning_from_dd() —— 分区组
  if (tab_obj->partition_type() == dd::Table::PT_NONE) return false;
  // DD 只记录主引擎怎么分区，secondary engine 的表不能用这份信息
  if (share->is_secondary_engine()) return false;
  part_info = new (&share->mem_root) partition_info;      // ★ share->m_part_info
  part_info->default_engine_type = hton;
  switch (tab_obj->partition_type()) {
    case dd::Table::PT_RANGE_COLUMNS: part_info->column_list = true;
                                      part_info->list_of_part_fields = true;
                                      [[fallthrough]];
    case dd::Table::PT_RANGE:  part_info->part_type = partition_type::RANGE; break;
    case dd::Table::PT_LIST:   part_info->part_type = partition_type::LIST;  break;
    case dd::Table::PT_HASH:   part_info->part_type = partition_type::HASH;  break;
    /* … LINEAR_HASH / KEY_51 / KEY_55 / AUTO … */
  }

// [O4] fill_foreign_keys_from_dd() —— 外键组（只拷名字与规则，不解析被引用的表）
  share->foreign_keys = tab_obj->foreign_keys().size();
  share->foreign_key_parents = tab_obj->foreign_key_parents().size();
  for (const dd::Foreign_key *fk : tab_obj->foreign_keys()) {
    share->foreign_key[i].update_rule = fk->update_rule();
    share->foreign_key[i].delete_rule = fk->delete_rule();
    share->foreign_key[i].columns = fk->elements().size();
  }

// [O4] fill_check_constraints_from_dd() —— CHECK 约束组（只存表达式字符串）
  for (auto &cc : tab_obj->check_constraints()) {
    LEX_CSTRING check_clause;      // ★ 只存字符串，不解析
    lex_string_strmake(&share->mem_root, &check_clause,
                       cc->check_clause().c_str(), cc->check_clause().length());
    bool is_cc_enforced =
        (cc->constraint_state() == dd::Check_constraint::CS_ENFORCED);
    share->check_constraint_share_list->push_back(
        Sql_check_constraint_share(name, check_clause, is_cc_enforced));
  }
```

> ★ 分工：`fill_*` 只填**数据**；`prepare_share()` 才造临时 `handler` 问引擎能力；表达式（生成列、默认值、CHECK）的解析推迟到 `open_table_from_share()` 里 per-`TABLE` 地做。

### 3. ★ `TABLE_SHARE` ↔ `dict_table_t`

#### 3.1 非分区表：`ha_innobase::open()` 一对一

```cpp
// [O6] storage/innobase/handler/ha_innodb.cc
int ha_innobase::open(const char *name, int, uint open_flags,
                      const dd::Table *table_def) {
  dict_table_t *ib_table;
  assert(table_share == table->s);          // ★ handler::table_share 就是 TABLE::s
  ...
  ib_table = thd_to_innodb_session(thd)->lookup_table_handler(norm_name);

  if (ib_table == nullptr) {
    dict_sys_mutex_enter();                 // ★ 8.0 用 dict_sys_mutex_enter（旧名 dict_mutex_enter 已废）
    ib_table = dict_table_check_if_in_cache_low(norm_name);   // ★ 按 "db/table" 查字典缓存
    if (ib_table != nullptr) {
      if (ib_table->is_corrupted()) { ... ib_table = nullptr; cached = true; }
      else if (ib_table->refresh_fk) {
        ib_table->acquire_with_lock();
        dict_sys_mutex_exit();              // ★ 读 DD 前放字典锁
        dd::cache::Dictionary_client *client = dd::get_dd_client(thd);
        dd::cache::Dictionary_client::Auto_releaser releaser(client);
        dberr_t err = dd_table_load_fk(client, ib_table->name.m_name, nullptr,
                                       ib_table, &table_def->table(), thd, false,
                                       ..., &fk_tables);
        dict_sys_mutex_enter();
        ib_table->refresh_fk = false;
        cached = true;
      } else if (ib_table->discard_after_ddl) {
      reload:
        btr_drop_ahi_for_table(ib_table);
        dict_table_remove_from_cache(ib_table);   // DDL 后必须重新加载
        ib_table = nullptr;
      } else {
        cached = true;
        if (!dd_table_match(ib_table, table_def)) {   // ★ 用 DD 版本号/定义校对
          dict_set_corrupted(ib_table->first_index());
          dict_table_remove_from_cache(ib_table);
          ib_table = nullptr;
        } else {
          ib_table->acquire_with_lock();              // ★ 命中：引用计数 +1
        }
      }
      if (ib_table != nullptr && table_def != nullptr)
        ib_table->version = dd_get_version(table_def); // ★ 对齐 DD 版本号
    }
    dict_sys_mutex_exit();

    if (!cached) {
      // ★ 按 DD 定义现场造 dict_table_t
      ib_table = dd_open_table(client, table, norm_name, table_def, thd);
      if (!ib_table) { set_my_errno(ENOENT); return HA_ERR_NO_SUCH_TABLE; }
    }
  } else {
    ib_table->acquire();
    ut_ad(ib_table->is_intrinsic());
  }

  /* m_share 里存着指向 dict_index_t 的裸指针（无 pin），因此必须
     在拿到 dict_table_t 之后分配、在释放 dict_table_t 之前销毁 */
  if (ib_table != nullptr) {
    m_share = get_share(name);
    ...
  }

  // ★ MySQL 列数与 InnoDB 列数对不上 → 标损坏（FTS_DOC_ID 隐藏列要减 1）
  if (ib_table != nullptr &&
      ((!DICT_TF2_FLAG_IS_SET(ib_table, DICT_TF2_FTS_HAS_DOC_ID) &&
        table->s->fields != dict_table_get_n_tot_u_cols(ib_table)) || ...)) {
    ib::warn(ER_IB_MSG_556) << "Table " << norm_name << " contains …";
    ib_table->first_index()->type |= DICT_CORRUPT;
    ...
  }

  innobase_copy_frm_flags_from_table_share(ib_table, table->s);  // ★ server → InnoDB 反向同步
  dict_stats_init(ib_table);

  m_prebuilt = row_create_prebuilt(ib_table, table->s->reclength);
  m_prebuilt->default_rec = table->s->default_values;   // ★ 默认行地址传进 prebuilt
  m_prebuilt->m_mysql_table = table;                    // ★ row_prebuilt_t 里保存
  m_prebuilt->m_mysql_handler = this;                   //   TABLE* 与 ha_innobase*
}
```

```cpp
// [O7] storage/innobase/dict/dict0dd.cc —— DD → dict_table_t
template <typename Table>
dict_table_t *dd_open_table(dd::cache::Dictionary_client *client,
                            const TABLE *table, const char *norm_name,
                            const Table *dd_table, THD *thd) {
  dict_table_t *m_table = nullptr;
  dict_names_t fk_list;
  m_table = dd_open_table_one(client, table, norm_name, dd_table, thd, fk_list);
  if (m_table != nullptr && !fk_list.empty())
    dd_open_fk_tables(fk_list, false, thd);   // ★ 顺带把外键引用到的表也打开
  return m_table;
}
// 显式实例化两份：整表用 dd::Table，分区用 dd::Partition
template dict_table_t *dd_open_table<dd::Table>(...);
template dict_table_t *dd_open_table<dd::Partition>(...);

// dd_open_table_one() 内：
  dict_table_t *m_table = dd_fill_dict_table(
      dd_table, table, norm_name, nullptr, zip_allowed, strict, thd, implicit);
  int ret = dd_fill_dict_index(dd_table->table(), table, m_table, thd);
  /* … 再从 se_private_data 里取 AUTO_INCREMENT / version … */
  /* … 再从 dd_index->se_private_data() 里取出每个索引的 space id / root page /
        trx_id，填进 dict_index_t —— 这一步才是「DD 的物理元信息落地到 InnoDB」 */
```

#### 3.2 ★ 分区表：一个 `TABLE_SHARE` 对 N 个 `dict_table_t`

```cpp
// [O8] storage/innobase/handler/ha_innopart.h
/** InnoDB partition specific Handler_share. */
class Ha_innopart_share : public Partition_share {
 private:
  /** Array of all included table definitions (one per partition). */
  dict_table_t **m_table_parts;        // ★★ 每个分区一个 dict_table_t

  /** Maps MySQL index number to InnoDB index per partition. */
  dict_index_t **m_index_mapping;
  uint m_tot_parts;
  uint m_index_count;
  uint m_ref_count;
  /** Pointer back to owning TABLE_SHARE. */
  TABLE_SHARE *m_table_share;          // ★★ 反向指回 TABLE_SHARE
 public:
  inline dict_table_t *get_table_part(uint part_id) const {
    ut_ad(m_table_parts != nullptr);
    ut_ad(part_id < m_tot_parts);
    return (m_table_parts[part_id]);
  }
  static dict_table_t **open_table_parts(THD *thd, const TABLE *table,
                                         const dd::Table *dd_table,
                                         partition_info *part_info,
                                         const char *table_name);
  bool set_table_parts_and_indexes(partition_info *part_info,
                                   dict_table_t **table_parts);
};
```

```cpp
// [O8] storage/innobase/handler/ha_innopart.cc
  // ★ 从 TABLE_SHARE::ha_share 里拿 Ha_innopart_share（受 LOCK_ha_data 保护）
  lock_shared_ha_data();
  m_part_share = static_cast<Ha_innopart_share *>(get_ha_share_ptr());
  if (m_part_share == nullptr) {
    m_part_share = new (std::nothrow) Ha_innopart_share(table_share);
    set_ha_share_ptr(static_cast<Handler_share *>(m_part_share));  // ★ 回写 share->ha_share
  }
  if (m_part_share->has_table_parts()) {
    m_part_share->increment_ref_counts();      // ★ 别的 TABLE 已开过：只加引用
  } else {
    // ★ 要读 DD 开分区 → 必须先放 LOCK_ha_data
    //   （关表路径是「持 table cache 锁再拿 LOCK_ha_data」，反过来会死锁）
    unlock_shared_ha_data();
    dict_table_t **table_parts = Ha_innopart_share::open_table_parts(
        thd, table, table_def, m_part_info, norm_name);
    lock_shared_ha_data();
    // ★ 再检查一遍：放锁期间可能已被别的线程填好，那么丢弃自己这份
    if (m_part_share->set_table_parts_and_indexes(m_part_info, table_parts))
      goto share_error;
  }

dict_table_t **Ha_innopart_share::open_table_parts(...) {
#ifdef UNIV_DEBUG
  if (table->s->tmp_table == NO_TMP_TABLE)
    mysql_mutex_assert_not_owner(&table->s->LOCK_ha_data);   // ★ 不能持锁进来
#endif
  uint tot_parts = part_info->get_tot_partitions();
  dict_table_t **table_parts = ...(sizeof(dict_table_t *) * tot_parts);
  uint i = 0;
  for (const auto dd_part : dd_table->leaf_partitions()) {   // ★ 只遍历叶子分区
    std::string partition;
    dict_name::build_partition(dd_part, partition);
    std::string part_table;
    // ★ 分区在 InnoDB 里就是一个独立的 dict_table_t，名字形如 "db/tbl#P#p0"
    dict_name::build_table("", table_name, partition, false, false, part_table);
    if (open_one_table_part(client, thd, table, dd_part, part_table.c_str(),
                            &table_parts[i])) { ... }
    i++;
  }
  return (table_parts);
}
```

#### 3.3 双向指针与生命周期

| 方向 | 指针 | 位置 |
|---|---|---|
| SHARE → 引擎 share | `TABLE_SHARE::ha_share`（`Handler_share*`） | `open_table_from_share()` 里 `file->set_ha_share_ref(&share->ha_share)` 建立 |
| handler → 引擎 share 的**地址** | `handler::ha_share`（`Handler_share**`） | 多个 `handler` 共享同一个 `TABLE_SHARE::ha_share` 槽，谁先创建谁填 |
| handler → SHARE | `handler::table_share` | `ha_innobase::open()` 首行 `assert(table_share == table->s)` |
| TABLE ↔ InnoDB | `row_prebuilt_t::m_mysql_table`（`TABLE*`）/ `m_mysql_handler` | `row0mysql.h` |
| `Ha_innopart_share` → SHARE | `Ha_innopart_share::m_table_share` | `[O8]` |

⚠️ **8.0.39 中 `dict_table_t` 没有 `mysql_handle` 成员**（旧版本有过），InnoDB 侧回到 server 层靠的是 `row_prebuilt_t::m_mysql_table` / `m_mysql_handler`。

★ **生命周期先后**：`dd::Table`（DD 缓存对象）→ `TABLE_SHARE`（TDC）→ `TABLE` + `handler` → `dict_table_t`（`dict_sys` 缓存）。销毁严格逆序：`ha_close()` 放 `dict_table_t` → `closefrm()` 放 `TABLE` → `release_table_share()` 放 `TABLE_SHARE`（`ref_count==0` 才真正从 TDC 摘掉）→ `Auto_releaser` 放 `dd::Table`。源码注释也写明：`m_share` 保存无 pin 的 `dict_index_t*`，所以必须「晚于 `dict_table_t` 分配、早于它销毁」。

### 4. ★ 两套缓存不是一回事

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 第 1 层：DD 缓存（sql/dd/）                                                │
│   Shared_dictionary_cache（全局，按对象类型分区，LRU + 引用计数）             │
│        │ miss → Storage_adapter → 真读 mysql.tables / mysql.columns / …     │
│        ▼                                                                    │
│   Dictionary_client（每 THD 一个）                                          │
│     ├─ m_registry_committed     本地已获取对象登记表（含 Auto_releaser 链）    │
│     ├─ m_registry_uncommitted   本事务未提交的修改（只本 client 可见）         │
│     └─ m_registry_dropped                                                   │
│   缓存的是：dd::Table / dd::Schema / dd::Tablespace … 的**对象图**            │
│   失效触发：DDL 提交时 Shared_dictionary_cache::drop()                      │
└──────────────────────────────────────────────────────────────────────────┘
                              │  open_table_def()：一次性"翻译"
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 第 2 层：TDC —— table_def_cache（sql/sql_base.cc）                          │
│   缓存的是：TABLE_SHARE（**server 层展开后的行布局/KEY/位图/字符集/…**）       │
│   key = "db\0table"；上限 table_definition_cache；按 unused LRU 淘汰          │
│   失效触发：refresh_version 推进（FLUSH TABLES）/ tdc_remove_table()          │
└──────────────────────────────────────────────────────────────────────────┘
                              │  open_table_from_share()
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 第 3 层：Table_cache（16 分区，sql/table_cache.h）                           │
│   缓存的是：TABLE（**带 handler、已 ha_open 的实例**）                        │
│   上限 table_open_cache；按 m_unused_tables LRU 淘汰                         │
└──────────────────────────────────────────────────────────────────────────┘
                              │  handler::ha_open()
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 第 4 层：dict_sys（InnoDB）缓存的是 dict_table_t / dict_index_t              │
└──────────────────────────────────────────────────────────────────────────┘
```

```cpp
// [O5] sql/dd/impl/cache/dictionary_client.cc —— 三级查找
template <typename K, typename T>
bool Dictionary_client::acquire(const K &key, const T **object, ...) {
  Timestamp_timezone_guard ts(m_thd);       // DD 对象一律按 UTC 缓存

  // ① 先查「本事务未提交对象」登记表 —— 自己改的自己要看得见
  acquire_uncommitted(key, &uncommitted_object, &dropped);
  if (uncommitted_object || dropped) { *object = uncommitted_object; return false; }

  // ② 再查本 client 的 committed 登记表
  m_registry_committed.get(key, &element);
  if (element) {
    ...
    assert(MDL_checker::is_read_locked(m_thd, *object));  // ★ 必须有 MDL 保护才敢用
    *object = element->object();
    return false;
  }

  // ③ 都没有 → 全局共享缓存（再 miss 就真去读 InnoDB 的 mysql.* 系统表）
  if (Shared_dictionary_cache::instance()->get(m_thd, key, &element)) return true;
  if (element) {
    ...
    m_registry_committed.put(element);
    m_current_releaser->auto_release(element);   // ★ 交给当前 Auto_releaser 托管
    *object = element->object();
    assert(MDL_checker::is_read_locked(m_thd, *object));
  }
  return false;
}
```

★ **为什么必须两层**：DD 缓存的是「权威元数据的对象图」（面向 DDL 与一致性）；TDC 缓存的是「展开成可直接执行形态的 `TABLE_SHARE`」——列偏移、`null_bytes`、`rec_buff_length`、`key_info` 数组、字符集、位图大小，这些从 DD 推导代价很高。两者**失效来源不同**（DDL 提交 vs `refresh_version`）、**粒度也不同**（一个 `dd::Table` 可对应不同 secondary engine 的多个 `TABLE_SHARE`），所以不能合并。

### 5. `DD → TABLE_SHARE → dict_table_t` 构建链

```
 mysql.tables / mysql.columns / mysql.indexes / mysql.partitions …（InnoDB 系统表）
          │  Shared_dictionary_cache::get()（miss 时 Storage_adapter 真读）
          ▼
     dd::Table  （const，引用计数，全局共享；Auto_releaser 托管生命周期）
          │  get_table_share()：放 LOCK_open 后 acquire()
          │  open_table_def()
          ├── fill_share_from_dd()            → db_plugin / db_type / db_create_options /
          │                                     table_charset / max_rows / min_rows / 引擎选项
          ├── fill_columns_from_dd()          → share->field[] / fields / null_bytes /
          │                                     rec_buff_length / default_values / vfields
          ├── fill_indexes_from_dd()          → share->key_info[] / key_parts / keynames /
          │                                     rec_per_key[] / keys_in_use / visible_indexes
          ├── fill_partitioning_from_dd()     → share->m_part_info / partition_info_str
          ├── fill_foreign_keys_from_dd()     → foreign_key[] / foreign_key_parent[]
          └── fill_check_constraints_from_dd()→ check_constraint_share_list
          │  prepare_share()（临时造一个 handler 问引擎能力）
          ▼
     TABLE_SHARE ──── ha_share ────► Handler_share*
          │                            ├ 非分区：INNOBASE_SHARE（含 idx_trans_tbl）
          │                            └ 分区：Ha_innopart_share（含 m_table_parts[]）
          │  open_table_from_share()：clone Field[]、key_info 深拷贝、8 张位图
          ▼
     TABLE ──── file ────► handler（ha_innobase / ha_innopart）
                                │ ha_open(outparam, path, flags, table_def)
                                ▼
              ha_innobase::open()
                 dict_sys_mutex_enter()
                   dict_table_check_if_in_cache_low("db/tbl")
                      └ miss → dd_open_table<dd::Table>()
                                   └ dd_open_table_one()
                                        ├ dd_fill_dict_table() → dict_table_t
                                        └ dd_fill_dict_index() → dict_index_t
                                          （再从 se_private_data 取 space id / root page）
                                ▼
                          dict_table_t ──► dict_sys 缓存（按名字 hash + LRU）

     ── 分区表的一对多 ───────────────────────────────────────────────
     TABLE_SHARE::ha_share ──► Ha_innopart_share
                                 m_table_share  ──► 回指 TABLE_SHARE
                                 m_tot_parts    =  part_info->get_tot_partitions()
                                 m_table_parts[0] ──► dict_table_t  "db/tbl#P#p0"
                                 m_table_parts[1] ──► dict_table_t  "db/tbl#P#p1"
                                        …
                                 m_index_mapping[] ─► dict_index_t（MySQL keynr × 分区）
     ⇒ 一个 TABLE_SHARE ⇒ 一个 Ha_innopart_share ⇒ N 个 dict_table_t
```

> **勘误**：图里的 `dd_open_table<dd::Table>()` 是 5.7→8.0 过渡期的写法，8.0.39 中该函数**已不存在**。真实入口是 `dd_open_table()`（非模板）。下面逐层给出源码。

#### 5.1 `dd_open_table()` → `dd_open_table_one()` → `dict_table_open_on_name()`

```cpp
// [O7] storage/innobase/dict/dict0dd.cc:5448
dict_table_t *dd_open_table(dd::cache::Dictionary_client *client,
                            const TABLE *table, const char *norm_name,
                            const Table *dd_table, THD *thd) {
  dict_table_t *m_table = nullptr;
  dict_names_t fk_list;

  m_table = dd_open_table_one(client, table, norm_name, dd_table, thd, fk_list);
  // ★ dd_open_table_one() 内部：
  //   dd_table_open_on_name()（拿/建 dict_table_t）
  //     → dd_fill_dict_table()：列、row_format、space id、flags…
  //     → dd_fill_dict_index()：每个索引的 dict_index_t，从 se_private_data 取 root page

  /* If there is foreign table references to this table, we will
  try to open them */
  if (m_table != nullptr && !fk_list.empty()) {
    dd_open_fk_tables(fk_list, false, thd);   // ★ 顺带把被引用（父）表也打开
  }

  return m_table;
}
```

⇒ **外键父表的打开是"顺带"的**：`dd_open_table_one()` 收集 `fk_list`，`dd_open_fk_tables()` 再逐个打开。这也是为什么 SHARE 层只记外键**名字**（见 2.5）——对象图由 InnoDB 在自己持有 `dict_sys->mutex` 时建立。

```cpp
// [O7] storage/innobase/dict/dict0dict.cc:1062
/** Returns a table object and increment its open handle count.
 NOTE! This is a high-level function to be used mainly from outside the
 'dict' module. Inside this directory dict_table_get_low
 is usually the appropriate function.
 @return table, NULL if does not exist */
dict_table_t *dict_table_open_on_name(
    const char *table_name,       /*!< in: table name */
    bool dict_locked,             /*!< in: true=data dictionary locked */
    bool try_drop,                /*!< in: true=try to drop any orphan
                                  indexes after an aborted online
                                  index creation */
    dict_err_ignore_t ignore_err) /*!< in: error to be ignored when
                                  loading a table definition */
{
  dict_table_t *table;
  DBUG_TRACE;

  if (!dict_locked) {
    dict_sys_mutex_enter();     // ★ 全局 dict_sys->mutex（InnoDB 字典大锁）
  }

  std::string table_str(table_name);
  /* Check and convert 5.7 table name. We always keep 8.0 format name in cache
  during upgrade. */
  if (dict_name::is_partition(table_name)) {
    dict_name::rebuild(table_str);   // ★ 分区表名 "db/tbl#P#p0" 的规范化
  }
  table = dict_table_check_if_in_cache_low(table_str.c_str());

  if (table == nullptr) {
    table = dict_load_table(table_name, true, ignore_err);  // ★ 真去读 SYS_TABLES 等
  }

  ut_ad(!table || table->cached);

  if (table != nullptr) {
    if (ignore_err == DICT_ERR_IGNORE_NONE && table->is_corrupted()) {
      /* Make life easy for drop table. */
      dict_table_prevent_eviction(table);   // ★ 坏表钉住不被淘汰，好让 DROP 还能找到它

      if (!dict_locked) {
        dict_sys_mutex_exit();
      }

      ib::info(ER_IB_MSG_175) << "Table " << table->name
                              << " is corrupted. Please drop the table"
                                 " and recreate it";
      return nullptr;
    }

    if (table->can_be_evicted) {
      dict_move_to_mru(table);   // ★ InnoDB 自己有一条 LRU（dict_sys->table_LRU）
    }

    table->acquire();            // ★★ ref_count++（与 dict_table_close() 配对）
  }

  ut_ad(dict_lru_validate());

  if (!dict_locked) {
    dict_table_try_drop_aborted_and_mutex_exit(table, try_drop);
  }

  return table;
}
```

⇒ 与 server 侧对照：`TABLE_SHARE` 的引用计数是 `m_ref_count`（`LOCK_open` 保护），`dict_table_t` 的是 `ref_count`（`dict_sys->mutex` + 原子）。两套缓存各自 LRU、各自淘汰，**互不知情**——只靠 DDL 提交时的显式失效（`dd_table_close` / `tdc_remove_table`）维持一致。

#### 5.2 关闭侧：`ha_innobase::close()` → `row_prebuilt_free()` → `dd_table_close()`

```cpp
// [O6] storage/innobase/handler/ha_innodb.cc:7680
int ha_innobase::close() {
  DBUG_TRACE;

  if (m_prebuilt->m_temp_read_shared) {
    temp_prebuilt_vec *vec = m_prebuilt->table->temp_prebuilt;
    ut_ad(m_prebuilt->table->is_intrinsic());
    vec->erase(std::remove(vec->begin(), vec->end(), m_prebuilt), vec->end());
  }

  // ★★ 顺序不可换：m_share（INNOBASE_SHARE）里存着指向 dict_index_t 的**裸指针**
  //   （idx_trans_tbl），必须在 dict_table_t 释放之前先释放，否则留下悬垂指针
  free_share(m_share);

  // ★ dict_table_close() 不在这里直接调，而在 row_prebuilt_free() 内部
  row_prebuilt_free(m_prebuilt, false);

  if (m_upd_buf != nullptr) {
    ut_ad(m_upd_buf_size != 0);
    my_free(m_upd_buf);
    m_upd_buf = nullptr;
    m_upd_buf_size = 0;
  }

  MONITOR_INC(MONITOR_TABLE_CLOSE);

  /* Tell InnoDB server that there might be work for utility threads */
  srv_active_wake_master_thread();

  return 0;
}
```

```cpp
// storage/innobase/row/row0mysql.cc:957（尾部）
void row_prebuilt_free(row_prebuilt_t *prebuilt, bool dict_locked) {
  DBUG_TRACE;
  ut_a(prebuilt->magic_n == ROW_PREBUILT_ALLOCATED);
  ut_a(prebuilt->magic_n2 == ROW_PREBUILT_ALLOCATED);

  prebuilt->magic_n = ROW_PREBUILT_FREED;
  prebuilt->magic_n2 = ROW_PREBUILT_FREED;
  ...
  prebuilt->pcur->reset();
  prebuilt->clust_pcur->reset();

  ut::free(prebuilt->mysql_template);
  ...
  if (prebuilt->rtr_info) {
    rtr_clean_rtr_info(prebuilt->rtr_info, true);
  }

  // ★★ handler 关闭时 dict_table_t 的释放点就在这里：ref_count--
  if (prebuilt->table) {
    ut_ad(!prebuilt->table->is_fts_aux());
    dd_table_close(prebuilt->table, nullptr, nullptr, dict_locked);
  }

  prebuilt->m_lob_undo.destroy();

  mem_heap_free(prebuilt->heap);
}
```

⇒ **完整调用栈（开表一路到 `dict_table_t`）**：

```
open_tables()                                   sql/sql_base.cc:5768
└ open_and_process_table()                      sql/sql_base.cc:4919
   └ open_table()                               sql/sql_base.cc:2810
      ├ open_table_get_mdl_lock()               sql/sql_base.cc:2598  （MDL）
      ├ Table_cache::get_table()                sql/table_cache.h     （命中则复用 TABLE）
      ├ get_table_share_with_discover()         sql/sql_base.cc
      │  └ get_table_share()                    sql/sql_base.cc:645   （LOCK_open + COND_open）
      │     └ open_table_def()                  sql/dd_table_share.cc:2290
      │        ├ fill_share_from_dd()           sql/dd_table_share.cc:563
      │        │  └ prepare_default_value_buffer_and_table_share()  sql/default_values.cc:263
      │        │     └ find_record_length()     sql/default_values.cc:101
      │        ├ fill_columns_from_dd()         sql/dd_table_share.cc:1113
      │        ├ fill_indexes_from_dd()         sql/dd_table_share.cc:1465
      │        ├ fill_partitioning_from_dd()    sql/dd_table_share.cc:1906
      │        ├ fill_foreign_keys_from_dd()    sql/dd_table_share.cc:2155
      │        ├ fill_check_constraints_from_dd() sql/dd_table_share.cc:2250
      │        └ prepare_share()                sql/dd_table_share.cc:261
      ├ open_table_from_share()                 sql/table.cc:2874     （clone Field / key_info / ha_open）
      │  └ handler::ha_open()                   sql/handler.cc:2784
      │     └ ha_innobase::open()               storage/innobase/handler/ha_innodb.cc:7177
      │        ├ dict_table_check_if_in_cache_low()   （dict_sys->mutex）
      │        └ dd_open_table()                storage/innobase/dict/dict0dd.cc:5448
      │           └ dd_open_table_one()
      │              └ dd_table_open_on_name()
      │                 └ dict_table_open_on_name()  storage/innobase/dict/dict0dict.cc:1062
      ├ Table_cache::add_used_table()           sql/table_cache.h
      └ TABLE::init()                           sql/table.cc:4088
```

---

## `Table_ref`：★ SQL 层的表引用（深度剖析）

> 源码坐标：`[T1] sql/table.h`（`class Table_ref`，8.0 由 `TABLE_LIST` 改名）· `[T8] sql/sql_parse.cc`（`Query_block::add_table_to_list()`）· `[T4] sql/sql_lex.h`（`add_to_query_tables` / nest 管理）· `[T6] sql/table.cc`（`new_nested_join` / `fetch_number_of_rows`）· `[T17] sql/sql_list.h`（`SQL_I_List::link_in_list`）· `[T15] sql/sql_view.cc`（视图绑定）· `[T14] sql/sp_head.cc`（prelocking 占位符）· `[T2] sql/sql_base.cc`（开表旁路 / 重名检查）· `[T18] include/thr_lock.h`（`Lock_descriptor`）· `[T11] sql/table_function.h`（`Table_function`）

> **分工导航**：本节只讲 **SQL 层"表引用"对象本身**。`Table_ref::table`（`TABLE*`）如何被 `open_table()` 填上、MDL 请求如何被 `lock_tables()` 消费、`TABLE_SHARE`/`TABLE` 的成员与缓存分层，见后文「表的通用操作」「`TABLE_SHARE` 与 `TABLE`」两节，此处不重复。

### 1. 为什么它不叫"表"：改名动机与核心成员

8.0 起 `TABLE_LIST` 正式改名 `Table_ref`。改名动机是**这个对象从来就不是"表"**——它只是 FROM 子句里的一个**引用槽位**，同一槽位在不同阶段可代表基表、视图、派生表、CTE 引用、JSON_TABLE 结果、I_S 内存表，甚至一个**纯嵌套连接节点**。先看构造（其余 8 个重载只是"要不要显式给 MDL type / alias / 长度"的组合）：

```cpp
// [T1] sql/table.h
  /**
    Creates a Table_ref object with pre-allocated strings for database,
    table and alias.
  */
  Table_ref(TABLE *table_arg, const char *db_name_arg, size_t db_length_arg,
            const char *table_name_arg, size_t table_name_length_arg,
            const char *alias_arg, enum thr_lock_type lock_type_arg)
      : db(db_name_arg),
        table_name(table_name_arg),
        alias(alias_arg),
        m_map(1),                          // 单表引用：表号 0 → 位图 bit0
        table(table_arg),                  // 已开表时直接绑 TABLE*；未开表为 nullptr
        m_lock_descriptor{lock_type_arg},  // 引擎层 thr_lock 类型（TL_READ/TL_WRITE...）
        db_length(db_length_arg),
        table_name_length(table_name_length_arg) {
    // ★ 注意：MDL 请求用的是 db + table_name（真实名），不是 alias。
    // 别名只影响名字解析，不影响加哪把元数据锁。
    MDL_REQUEST_INIT(&mdl_request, MDL_key::TABLE, db, table_name,
                     mdl_type_for_dml(m_lock_descriptor.type), MDL_TRANSACTION);
  }
```

核心成员（按语义分组摘录）：

```cpp
// [T1] sql/table.h
  /*
    List of tables local to a subquery or the top-level SELECT (used by
    SQL_I_List). Considers views as leaves (unlike 'next_leaf' below).
    Created at parse time in Query_block::add_table_to_list() ->
    table_list.link_in_list().
    —— 同层/同 query block 的扁平解析顺序链；视图算叶子
  */
  Table_ref *next_local{nullptr};
  /* link in a global list of all queries tables
     —— 跨 query block 的全局扁平链，头是 LEX::query_tables */
  Table_ref *next_global{nullptr}, **prev_global{nullptr};
  // 名字三元组：真实库名 / 真实表名 / 对外可见名（别名或表名）
  const char *db{nullptr}, *table_name{nullptr}, *alias{nullptr};
  size_t db_length{0};
  size_t table_name_length{0};

  /* link to query_block where this table was used */
  Query_block *query_block{nullptr};

  /** Table level optimizer hints for this table.  */
  Opt_hints_table *opt_hints_table{nullptr};
  /* Hints for query block of this table. */
  Opt_hints_qb *opt_hints_qb{nullptr};

  void set_lock(const Lock_descriptor &descriptor) {
    m_lock_descriptor = descriptor;
  }
  const Lock_descriptor &lock_descriptor() const { return m_lock_descriptor; }
```

```cpp
// [T1] sql/table.h —— 私有：编号/bitmap 必须对齐
 private:
  /**
    The members below must be kept aligned so that (1 << m_tableno) == m_map.
    A table that takes part in a join operation must be assigned a unique
    table number.
  */
  uint m_tableno{0};   ///< Table number within query block
  table_map m_map{0};  ///< Table map, derived from m_tableno
  /**
     If this table or join nest is the Y in "X [LEFT] JOIN Y ON C", this
     member points to C. May also be generated from JOIN ... USING clause.
     It may be modified only by permanent transformations (permanent = done
     once for all executions of a prepared statement).
     —— ON 条件。永久变换才能改（PS 重执行时不能改）
  */
  Item *m_join_cond{nullptr};
  bool m_is_sj_or_aj_nest{false};
```

```cpp
// [T1] sql/table.h
  /*
    List of nodes in a nested join tree, that should be considered as
    leaves with respect to name resolution. The leaves are: views,
    top-most nodes representing NATURAL/USING joins, subqueries, and
    base tables. All of these Table_ref instances contain a
    materialized list of columns. The list is local to a subquery.
    —— 名字解析用的"叶子"链（视图/NATURAL JOIN 顶节点/基表才是叶子）
  */
  Table_ref *next_name_resolution_table{nullptr};
  /* Field names in a USING clause for JOIN ... USING. */
  List<Index_hint> *index_hints{nullptr};
  TABLE *table{nullptr};        /* opened table —— 开表后绑定，本节的核心出口 */
  Table_id table_id{};          /* table id (from binlog) for opened table */
  Query_result_union *derived_result{nullptr};  // 派生表物化结果的搬运槽

  /*
    Holds the function used as the table function —— JSON_TABLE 的载体
  */
  Table_function *table_function{nullptr};

 private:
  /**
     This field is set to non-null for derived tables and views. It points
     to the Query_expression representing the derived table/view.
     E.g. for a query
     @verbatim SELECT * FROM (SELECT a FROM t1) b @endverbatim
  */
  Query_expression *derived{nullptr}; /* Query_expression of derived table */

  /// If non-NULL, the CTE which this table is derived from.
  Common_table_expr *m_common_table_expr{nullptr};
  /**
    If the user has specified column names with the syntaxes "table name
    parenthesis column names": ... CREATE VIEW v(column_names) AS ...
    then this points to the list of column names. NULL otherwise.
  */
  const Create_col_name_list *m_derived_column_names{nullptr};

 private:
  LEX *view{nullptr};          /* link on VIEW lex for merging —— 非 null 即"是视图" */
  enum_view_algorithm effective_algorithm{VIEW_ALGORITHM_UNDEFINED};
  Lock_descriptor m_lock_descriptor;   // 引擎 thr_lock 类型 + NOWAIT/SKIP 行行为
```

```cpp
// [T1] sql/table.h —— 嵌套连接三件套
  /**
    Is non-NULL if this table reference is a nested join, ie it represents
    the inner tables of an outer join, the tables contained in the
    parentheses of an inner join (eliminated during resolving), the tables
    referenced in a derived table or view, in a semi-join nest, the tables
    from the subquery.
  */
  NESTED_JOIN *nested_join{nullptr};
  /// The nested join containing this table reference.
  Table_ref *embedding{nullptr};
  /// The join list immediately containing this table reference
  mem_root_deque<Table_ref *> *join_list{nullptr};
```

```cpp
// [T1] sql/table.h —— 视图/派生表展开后的产物
  ST_SCHEMA_TABLE *schema_table{nullptr}; /* Information_schema table */
  /// Array of selected expressions from a derived table or view.
  Field_translator *field_translation{nullptr};
  Field_translator *field_translation_end{nullptr};
  /*
    List (based on next_local) of underlying tables of this view. I.e. it
    does not include the tables of subqueries used in the view. Is set only
    for merged views.
    —— merge 视图展开后，底层表挂在这条 next_local 链上
  */
  Table_ref *merge_underlying_list{nullptr};
  mem_root_deque<Table_ref *> *view_tables{nullptr};
  Table_ref *belong_to_view{nullptr};    /* most upper view this table belongs to */
  Table_ref *referencing_view{nullptr};
  /*
    List of all base tables local to a subquery including all view
    tables. Unlike 'next_local', this in this list views are *not*
    leaves. Created in setup_tables() -> make_leaf_tables().
  */
  Table_ref *next_leaf{nullptr};
```

```cpp
// [T1] sql/table.h —— 开表策略、类型期望、prelocking、MDL
  /**
     Specifies which kind of table should be open for this element
     of table list.
  */
  enum_open_type open_type{OT_TEMPORARY_OR_BASE};
  dd::enum_table_type required_type{};   // 见「表类型」一节：BASE_TABLE / USER_VIEW / SYSTEM_VIEW

  /*
    This Table_ref object is just placeholder for prelocking, it will be
    used for implicit LOCK TABLES only and won't be used in real statement.
  */
  bool prelocking_placeholder{false};

  enum {
    OPEN_NORMAL = 0,          /* Normal open. */
    OPEN_IF_EXISTS,           /* Associate a table share only if the table exists. */
    OPEN_FOR_CREATE,          /* ...Also upgrade metadata lock to exclusive if table doesn't exist. */
    OPEN_STUB                 /* Don't associate a table share. */
  } open_strategy{OPEN_NORMAL};
  bool internal_tmp_table{false};
  /** true if an alias for this table was specified in the SQL. */
  bool is_alias{false};
  /** true if the table is referred to in the statement using a fully
      qualified name (@<db_name@>.@<table_name@>). */
  bool is_fqtn{false};
  // True, If this is a system view
  bool is_system_view{false};
  MDL_request mdl_request;               // 内联的 MDL 请求，生命周期与 Table_ref 同
  /// True if right argument of LEFT JOIN; ... RIGHT JOINs are converted to LEFT JOIN
  bool outer_join{false};
```

`Lock_descriptor` 定义在引擎锁头文件（8.0 新增 `action` 字段承载 `SKIP LOCKED`/`NOWAIT`）：

```cpp
// [T18] include/thr_lock.h
enum thr_locked_row_action { THR_DEFAULT, THR_WAIT, THR_NOWAIT, THR_SKIP };

struct Lock_descriptor {
  thr_lock_type type{TL_UNLOCK};        // 引擎表锁类型：TL_READ / TL_WRITE / ...
  thr_locked_row_action action{THR_DEFAULT};  // 行级等待行为
};
```

### 2. ★★ 双链结构：`next_local` / `next_global` 与 `embedding` / `join_list`

这是全类最精妙的地方：**同一个 `Table_ref` 对象同时挂在多条侵入式链表上**，靠不同的"next 指针成员"区分。挂链原语：

```cpp
// [T17] sql/sql_list.h
  inline void link_in_list(T *element, T **next_ptr) {
    elements++;
    (*next) = element;   // 前驱的 next 指向新元素
    next = next_ptr;     // 链表对象的 next 指向"新元素的 next 字段"（尾指针）
    *next = nullptr;
  }
```

两条扁平链的挂入点分别在 `add_table_to_list()` 的末尾：

```cpp
// [T8] sql/sql_parse.cc —— Query_block::add_table_to_list()
  /*
    Link the current table reference in a local list (list for current select).
    Notice that as a side effect here we set the next_local field of the
    previous table reference to 'ptr'. Here we also add one element to the
    list 'table_list'.
  */
  m_table_list.link_in_list(ptr, &ptr->next_local);   // ← next_local：本 query block
  ptr->next_name_resolution_table = nullptr;
  ptr->partition_names = partition_names;
  /* Link table in global list (all used tables) */
  lex->add_to_query_tables(ptr);                      // ← next_global：整条语句
```

`next_global` 的挂链带 `prev_global`（二级指针，便于 O(1) 摘链，例如 `LOCK TABLES`、`FLUSH` 临时截断链）：

```cpp
// [T4] sql/sql_lex.h
  /*
    Direct addition to the list of query tables.
    If you are using this function, you must ensure that the table
    object, in particular table->db member, is initialized.
  */
  void add_to_query_tables(Table_ref *table) {
    *(table->prev_global = query_tables_last) = table;
    query_tables_last = &table->next_global;
  }
```

**两链的区别与遍历场景**：

| | `next_local` | `next_global` |
|---|---|---|
| 范围 | 单个 `Query_block`（`Query_block::m_table_list`） | 整条语句的所有 `Query_block`（`LEX::query_tables`） |
| 顺序 | 解析顺序，含所有嵌套层级的表 | 解析顺序，跨子查询/派生表/view 展开 |
| 典型遍历 | `setup_tables()`、`make_leaf_tables()`、视图 merge 后的 `merge_underlying_list` | `open_tables()` / `lock_tables()` / `find_dup_table()` / binlog prelock / `FLUSH TABLES` 改 MDL 类型 |
| 源码例 | `for (Table_ref *t = m_table_list.first; t; t = t->next_local)` | `for (Table_ref *t = lex->query_tables; t; t = t->next_global)` |

嵌套**不是**靠 `next_local` 表达的，而是靠 `embedding` + `join_list` + `nested_join->m_tables`（`mem_root_deque`）三层：

```cpp
// [T6] sql/table.cc
Table_ref *Table_ref::new_nested_join(MEM_ROOT *allocator, const char *alias,
                                      Table_ref *embedding,
                                      mem_root_deque<Table_ref *> *belongs_to,
                                      Query_block *select) {
  assert(belongs_to && select);

  Table_ref *const join_nest = new (allocator) Table_ref;
  if (join_nest == nullptr) return nullptr;

  join_nest->nested_join = new (allocator) NESTED_JOIN;   // 有 nested_join ⇒ 这是"嵌套节点"而非真表
  if (join_nest->nested_join == nullptr) return nullptr;

  join_nest->db = "";          // 嵌套节点没有真实库名/表名，只有 alias
  join_nest->db_length = 0;
  join_nest->table_name = "";
  join_nest->table_name_length = 0;
  join_nest->alias = alias;

  join_nest->embedding = embedding;   // 父嵌套（NULL ⇒ 顶层）
  join_nest->join_list = belongs_to;  // 自己所在的 deque
  join_nest->query_block = select;
  join_nest->nested_join->first_nested = NO_PLAN_IDX;

  join_nest->nested_join->m_tables.clear();   // 子节点 deque
  return join_nest;
}
```

```cpp
// [T8] sql/sql_parse.cc —— 括号进入/退出时切换"当前 nest"
bool Query_block::init_nested_join(THD *thd) {
  Table_ref *const ptr = Table_ref::new_nested_join(
      thd->mem_root, "(nested_join)", embedding, m_current_table_nest, this);
  if (ptr == nullptr) return true;

  m_current_table_nest->push_front(ptr);        // 先把自己放进父 deque
  embedding = ptr;                              // 再下潜一层
  m_current_table_nest = &ptr->nested_join->m_tables;
  return false;
}

Table_ref *Query_block::end_nested_join() {
  assert(embedding);
  ptr = embedding;
  m_current_table_nest = ptr->join_list;        // 回到父 deque
  embedding = ptr->embedding;                   // 回到父嵌套
  nested_join = ptr->nested_join;
  if (nested_join->m_tables.size() == 1) {
    // ★ 只有一个孩子 ⇒ 括号是多余的，孩子提上来，nest 消失（内层 inner join 括号在解析期就被消掉）
    Table_ref *embedded = nested_join->m_tables.front();
    m_current_table_nest->pop_front();
    embedded->join_list = m_current_table_nest;
    embedded->embedding = embedding;
    m_current_table_nest->push_front(embedded);
    ptr = embedded;
  } else if (nested_join->m_tables.empty()) {
    m_current_table_nest->pop_front();
    ptr = nullptr;  // return value
  }
  return ptr;
}
```

`nest_join()` 负责把最近 `table_cnt` 个表"收进"一个新 nest——这正是 `t1 LEFT JOIN (t2 JOIN t3)` 里内层的成型方式：

```cpp
// [T8] sql/sql_parse.cc
Table_ref *nest_join(THD *thd, Query_block *select, Table_ref *embedding,
                     mem_root_deque<Table_ref *> *jlist, size_t table_cnt,
                     const char *legend) {
  Table_ref *const ptr = Table_ref::new_nested_join(thd->mem_root, legend,
                                                    embedding, jlist, select);
  if (ptr == nullptr) return nullptr;

  mem_root_deque<Table_ref *> *const embedded_list = &ptr->nested_join->m_tables;

  for (uint i = 0; i < table_cnt; i++) {
    Table_ref *table = jlist->front();
    jlist->pop_front();
    table->join_list = embedded_list;   // 改指向：孩子现在属于内层 deque
    table->embedding = ptr;             // 改父：孩子的父亲是新 nest
    embedded_list->push_back(table);
    if (table->natural_join) ptr->is_natural_join = true;
  }
  jlist->push_front(ptr);               // nest 顶替孩子的位置
  return ptr;
}
```

★ **关键洞察**：`next_local` 在 `add_table_to_list()` 里一次性串好，之后**永不因嵌套而改**（它是解析顺序的扁平链）；`nest_join()` 只改 `embedding` / `join_list`，把"树形"信息全部压进 deque 与父指针。**"树"和"链"是两套正交视图**。

### ASCII 图：`SELECT * FROM t1 LEFT JOIN (t2 JOIN t3) ON t1.a=t2.a`

```
                            Query_block (顶层)
             m_table_list (SQL_I_List) ──► 头 = t1 ; 尾指针 = &t3->next_local
             m_current_table_nest ──────► 顶层 deque（解析结束后）

  ── 视图 1：next_local / next_global（两条扁平链，与嵌套无关）───────────

   next_local:   t1 ──► t2 ──► t3 ──► nullptr        （解析顺序，同 query block）
   next_global:  lex->query_tables ──► t1 ──► t2 ──► t3 ──► nullptr
                                       ▲      ▲      ▲
                        prev_global ───┘      │      │   （二级指针，O(1) 摘链）

  ── 视图 2：embedding / join_list / nested_join->m_tables（真正的树）─────

          nest#outer  "(nested_join)"   outer_join = true
          ┌──────────────────────────────────────────────────┐
          │ nested_join != nullptr  ⇒ 这是嵌套节点，不是表      │
          │ embedding  = nullptr        （顶层）                │
          │ join_list  = &sl->m_table_list（顶层 deque）         │
          │ m_join_cond = (t1.a = t2.a)  ← LEFT JOIN 的 ON      │
          │ m_tables (deque):                                   │
          │     [0] = t1      [1] = nest#inner                  │
          └───────┬──────────────────────────┬─────────────────┘
                  │ embedding                 │ embedding
                  ▼                           ▼
     ┌──────────────────┐        ┌───────────────────────────────┐
     │ t1  (叶子/基表)    │        │ nest#inner "(nested_join)"    │
     │ db/table_name/alias│       │ outer_join = false            │
     │ nested_join=nullptr│       │ embedding = nest#outer        │
     │ join_list = &outer │       │ join_list  = &outer.m_tables  │
     │        .m_tables   │       │ m_tables (deque): [t2, t3]    │
     └──────────────────┘        └──────┬───────────────┬────────┘
                                        │ embedding     │ embedding
                                        ▼               ▼
                                ┌──────────────┐ ┌──────────────┐
                                │ t2 (基表)     │ │ t3 (基表)     │
                                │ nested_join=∅ │ │ nested_join=∅ │
                                │ join_list =   │ │ join_list =   │
                                │  &inner.m_tab │ │  &inner.m_tab │
                                └──────────────┘ └──────────────┘

  遍历口诀：
    · 加 MDL / 开表 / 锁表  → 走 next_global（一条链扫完所有表）
    · 名字解析              → 走 next_name_resolution_table（视图/NATURAL 顶节点当叶子）
    · 求 join 顺序 / 外连接语义 → 走 embedding 链向上找 outer_join
```

向上找"我是否某个外连接的 inner 表"就是顺着 `embedding` 爬：

```cpp
// [T1] sql/table.h
  bool is_inner_table_of_outer_join() const {
    if (outer_join) return true;                      // 自己就是 LEFT JOIN 的右操作数
    for (Table_ref *emb = embedding; emb; emb = emb->embedding) {
      if (emb->outer_join) return true;               // 任一祖先是外连接 ⇒ 我是 inner
    }
    return false;
  }
```

### 3. ★ 名字解析：`db` / `table_name` / `alias`

```cpp
// [T8] sql/sql_parse.cc —— Query_block::add_table_to_list()
  const char *alias_str = alias ? alias : table_name->table.str;
  if (!alias) /* Alias is case sensitive */
  {
    if (table_name->sel) {                   // 派生表必须有别名，否则 ER_DERIVED_MUST_HAVE_ALIAS
      my_error(ER_DERIVED_MUST_HAVE_ALIAS, MYF(0));
      return nullptr;
    }
    if (!(alias_str = (char *)thd->memdup(alias_str, table_name->table.length + 1)))
      return nullptr;
  }
  ...
  ptr->table_name = table_name->table.str;
  ptr->table_name_length = table_name->table.length;
  ptr->alias = alias_str;
  ptr->is_alias = alias != nullptr;          // 是否 SQL 里显式写了 AS
  ptr->table_function = table_name->table_function;
  if (table_name->db.str) {
    ptr->is_fqtn = true;                     // 写全了 db.tbl
    ptr->db = table_name->db.str;
    ptr->db_length = table_name->db.length;
  } else {
    // ★ 未限定名先按 CTE 名查（WITH 定义的名字可以遮蔽真实表）
    bool found_cte = false;
    if ((table_options & TL_OPTION_ALIAS) == 0) {
      if (find_common_table_expr(thd, table_name, ptr, pc, &found_cte))
        return nullptr;
    }
    if (!found_cte && lex->copy_db_to(&ptr->db, &ptr->db_length))   // 否则用当前库补全
      return nullptr;
  }
```

三者语义差别：

- **`db` + `table_name`**：对象的**真实身份**。MDL key、DD 查找、权限检查、`required_type` 校验、binlog 记录全部用这一对；`alias` **完全不参与**。这也是为什么 `SELECT * FROM db1.t AS x` 加的锁是 `db1.t` 的锁。
- **`alias`**：**名字解析域内的唯一可见名**。`Item_field` 解析 `x.col` 时匹配的是 `alias`；`db.table_name` 这种限定写法则要求 `db` 与 `Table_ref::db` 相等。
- **重名检查用 `alias` 而非表名**（同一 query block 内别名必须唯一，哪怕指向同一张表两次也要取不同别名）：

```cpp
// [T8] sql/sql_parse.cc
  /* check that used name is unique */
  if (lock_type != TL_IGNORE) {
    Table_ref *first_table = get_table_list();
    if (lex->sql_command == SQLCOM_CREATE_VIEW)
      first_table = first_table ? first_table->next_local : nullptr;
    for (Table_ref *tables = first_table; tables; tables = tables->next_local) {
      if (!my_strcasecmp(table_alias_charset, alias_str, tables->alias) &&
          !strcmp(ptr->db, tables->db)) {
        my_error(ER_NONUNIQ_TABLE, MYF(0), alias_str);   // ERROR 1066: Not unique table/alias
        return nullptr;
      }
    }
  }
```

⇒ 改名后读代码时 `Table_ref` / `TABLE` 的界线一目了然：**`Table_ref` 是 SQL 层引用（解析期产物，一条语句一份），`TABLE` 是已打开的引擎表对象（可被缓存复用）**。两者通过 `Table_ref::table` 单向关联，且大量时刻 `table == nullptr`。

### 4. 特殊形态与判定函数

源码里自带一张形态清单（注意源码注释里仍是旧名 `TABLE_LIST`，8.0.39 未同步改注释）：

```cpp
// [T1] sql/table.h
/*
  Table reference in the FROM clause.

  These table references can be of several types that correspond to
  different SQL elements. Below we list all types of TABLE_LISTs with
  the necessary conditions to determine when a Table_ref instance
  belongs to a certain type.

  1) table (Table_ref::view == NULL)
     - base table
       (Table_ref::derived == NULL)
     - subquery - Table_ref::table is a temp table
       (Table_ref::derived != NULL)
     - information schema table
       (Table_ref::schema_table != NULL)
       NOTICE: for schema tables Table_ref::field_translation may be != NULL
  2) view (Table_ref::view != NULL)
     - merge    (Table_ref::effective_algorithm == VIEW_ALGORITHM_MERGE)
           also (Table_ref::field_translation != NULL)
     - temptable(Table_ref::effective_algorithm == VIEW_ALGORITHM_TEMPTABLE)
           also (Table_ref::field_translation == NULL)
  3) nested table reference (Table_ref::nested_join != NULL)
     - table sequence - e.g. (t1, t2, t3)
     - general JOIN
     - NATURAL JOIN
       (Table_ref::natural_join != NULL)
       - JOIN ... USING
         (Table_ref::join_using_fields != NULL)
     - semi-join
       ;
*/
```

判定函数（全部在 `[T1] sql/table.h`）：

```cpp
  /// Return true if this represents a named view
  bool is_view() const { return view != nullptr; }

  /// Return true if this represents a derived table (an unnamed view)
  bool is_derived() const { return derived != nullptr && view == nullptr; }

  /// Return true if this represents a named view or a derived table
  bool is_view_or_derived() const { return derived != nullptr; }

  /// Return true if this represents a table function
  bool is_table_function() const { return table_function != nullptr; }

  /**
     @returns true if this is a recursive reference inside the definition of a
     recursive CTE.
     @note that it starts its existence as a dummy derived table, until the
     end of resolution when it's not a derived table anymore, just a reference
     to the materialized temporary table. Whereas a non-recursive
     reference to the recursive CTE is a derived table.
  */
  bool is_recursive_reference() const { return m_is_recursive_reference; }

  /// @returns true if this is a base table (permanent or temporary)
  bool is_base_table() const {
    return !(is_view_or_derived() || is_table_function() ||
             is_recursive_reference());
  }

  bool is_internal() const {
    return is_derived() || is_recursive_reference() || is_table_function();
  }

  /**
    @returns true for a table that is a placeholder, ie a derived table,
    a view, a recursive reference, a table function or a schema table.
    A table is also considered to be a placeholder if it does not have a
    TABLE object for some other reason.
  */
  bool is_placeholder() const {
    return is_view_or_derived() || is_recursive_reference() ||
           is_table_function() || schema_table || table == nullptr;
  }
```

⚠️ **命名陷阱**：8.0.39 **没有** `is_materialized_derived()`，对应语义是 `uses_materialization()`；"是否 merge"是 `is_merged()`——都不是布尔成员而是 `effective_algorithm` 的取值：

```cpp
// [T1] sql/table.h
  /// Return true if this is a derived table or view that is merged
  bool is_merged() const { return effective_algorithm == VIEW_ALGORITHM_MERGE; }
  void set_merged() {
    assert(effective_algorithm == VIEW_ALGORITHM_UNDEFINED);
    effective_algorithm = VIEW_ALGORITHM_MERGE;
  }
  /// Return true if this is a materializable derived table/view
  bool uses_materialization() const {
    return effective_algorithm == VIEW_ALGORITHM_TEMPTABLE;
  }
  void set_uses_materialization() {
    // @todo We should do this only once, but currently we cannot:
    // assert(effective_algorithm == VIEW_ALGORITHM_UNDEFINED);
    assert(effective_algorithm != VIEW_ALGORITHM_MERGE);
    effective_algorithm = VIEW_ALGORITHM_TEMPTABLE;
  }
```

**视图**：`view` 是**单独一套 `LEX`**——视图定义要独立解析、独立做名字解析、独立做权限检查（SQL SECURITY）：

```cpp
// [T1] sql/table.h
  /**
    Set the LEX object of a view (will also define this as a view).
    @note: The value 1 is used to indicate a view but without a valid
           query object. Use only if the LEX object is not going to
           be used in later processing.
  */
  void set_view_query(LEX *lex) { view = lex; }

  /// Return the valid LEX object for a view.
  LEX *view_query() const {
    assert(view != nullptr && view != (LEX *)1);
    return view;
  }

  /// Set the query expression of a derived table or view.
  /// (Will also define this as a derived table, unless it is a named view.)
  void set_derived_query_expression(Query_expression *query_expr) {
    derived = query_expr;
  }
  /// Return the query expression of a derived table or view.
  Query_expression *derived_query_expression() const {
    assert(derived);
    return derived;
  }
```

解析视图定义时**必须换掉 `thd->lex`**，失败时还要把 `view` 复位（否则这个 `Table_ref` 会永久"是视图"）——用 RAII 保证：

```cpp
// [T15] sql/sql_view.cc
/// RAII class to ease error handling in parse_view_definition()
class Make_view_tracker {
 public:
  Make_view_tracker(THD *thd, Table_ref *view_ref, bool *result)
      : thd(thd), old_lex(thd->lex), view_ref(view_ref), result(result) {}
  ~Make_view_tracker() {
    if (thd->lex != old_lex) {
      lex_end(thd->lex);   // Terminate processing of view LEX
      thd->lex = old_lex;  // Needed for prepare_security
    }
    if (*result && view_ref->is_view()) {
      delete view_ref->view_query();
      view_ref->set_view_query(nullptr);  // view_ref is no longer a VIEW
    }
  }
...
  view_ref->open_type = OT_BASE_ONLY;     // 视图引用本身不再去开临时表
  // A parsed view requires its own LEX object
  LEX *const old_lex = thd->lex;
  LEX *const view_lex = (LEX *)new (thd->mem_root) st_lex_local;
  if (!view_lex) return true;

  bool result = false;
  Make_view_tracker view_tracker(thd, view_ref, &result);

  thd->lex = view_lex;
  view_ref->set_view_query(view_lex);      // ← 此刻起 is_view() 为真
  ...
  if ((result = parser_state.init(thd, view_ref->select_stmt.str,
                                  view_ref->select_stmt.length)))
    return true;
```

> 分工导航：视图 merge / 物化的决策链、CHECK OPTION、`updatable_base_table()` 属「视图解析与展开」，见 [`query/07_optimize/20_view_resolution.md`](query/07_optimize/20_view_resolution.md)。

**派生表**：`add_table_to_list()` 里直接把 `Table_ident::sel`（`Query_expression*`）塞进 `derived`——`ptr->set_derived_query_expression(table_name->sel);`

**schema table（I_S）**：在 `add_table_to_list()` 里就地判定并绑定 `ST_SCHEMA_TABLE*`：

```cpp
// [T8] sql/sql_parse.cc
  if (!ptr->is_derived() && !ptr->is_table_function() &&
      is_infoschema_db(ptr->db, ptr->db_length)) {
    dd::info_schema::convert_table_name_case(
        const_cast<char *>(ptr->db), const_cast<char *>(ptr->table_name));

    bool hidden_system_view = false;
    ptr->is_system_view = dd::get_dictionary()->is_system_view_name(
        ptr->db, ptr->table_name, &hidden_system_view);
    ...
    if (ptr->is_system_view) {
      if (thd->lex->sql_command != SQLCOM_CREATE_VIEW) {
        // ★ 隐藏系统视图（I_S 在 8.0 是 DD 上的系统视图）只允许 SHOW 命令走
        if (thd->lex->query_block && hidden_system_view &&
            !(thd->lex->query_block->active_options() & OPTION_SELECT_FOR_SHOW)) {
          my_error(ER_NO_SYSTEM_VIEW_ACCESS, MYF(0), ptr->table_name);
          return nullptr;
        }
        // I_S.FILES 需要 PROCESS 权限
        if (!strcmp(ptr->table_name, "FILES") && check_global_access(thd, PROCESS_ACL))
          return nullptr;
      }
    } else {
      schema_table = find_schema_table(thd, ptr->table_name);
      ...
      if (schema_table) {
        ptr->schema_table = schema_table;      // ← 非 null 即 is_placeholder()
      }
    }
  }
```

**table function（JSON_TABLE）**：`Table_function` 是独立类层次，`Table_ref::table_function` 只持有一份；结果表是一张真实的 `TABLE`（临时表）：

```cpp
// [T11] sql/table_function.h
/**
  Class representing a table function.
*/
class Table_function {
 protected:
  /// Table function's result table
  TABLE *table;
  /// Whether the table function was already initialized
  bool inited;
 public:
  explicit Table_function() : table(nullptr), inited(false) {}
  virtual ~Table_function() = default;
  /**
    Create, but not instantiate the result table
    @param thd         thread handler
    @param options     options to create table
    @param table_alias table's alias
    @returns ...
  */
...
class Table_function_json final : public Table_function {
  /// Array of JSON Data Source for each NESTED PATH clause
  std::array<JT_data_source, MAX_NESTED_PATH> m_jds;
  /// List of fields for tmp table creation
  List<Json_table_column> m_vt_list;
  ...
  /// JSON_TABLE's data source expression
  Item *source;
```

**CTE**：`Table_ref` 通过 `m_common_table_expr` 反指 CTE 定义；CTE 侧反持所有引用点：

```cpp
// [T1] sql/table.h
/**
  After parsing, a Common Table Expression is accessed through a
  Table_ref. This class contains all information about the CTE which the
  Table_ref needs.

  @note that before and during parsing, the CTE is described by a
  PT_common_table_expr.
*/
class Common_table_expr {
 public:
  Common_table_expr(MEM_ROOT *mem_root)
      : references(mem_root), recursive(false), tmp_tables(mem_root) {}
  TABLE *clone_tmp_table(THD *thd, Table_ref *tl);
  bool substitute_recursive_reference(THD *thd, Query_block *sl);
  /// Remove one table reference.
  void remove_table(Table_ref *tr);
  /// Empties the materialized CTE and informs all of its clones.
  bool clear_all_references();
  /**
     All references to this CTE in the statement, except those inside the
     query expression defining this CTE.
     In other words, all non-recursive references.
  */
  Mem_root_array<Table_ref *> references;
  /// True if it's a recursive CTE
  bool recursive;
  /**
    List of all TABLE_LISTSs reading/writing to the tmp table created to
    materialize this CTE. Due to shared materialization, only the first one
    has a TABLE generated by create_tmp_table(); other ones have a TABLE
    generated by open_table_from_share().
  */
  Mem_root_array<Table_ref *> tmp_tables;
```

递归引用在解析期被标记（同一 CTE 体内出现第二次引用 ⇒ `recursive = true`）：

```cpp
// [T8] sql/sql_parse.cc
  if (in_self) {
    m_postparse.recursive = true;
    if (tl->set_recursive_reference()) {
      my_error(ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE, MYF(0), name().str);
      return true;
    }
  } else {
    if (m_postparse.references.push_back(tl))
      return true;
    if (m_column_names.size()) tl->set_derived_column_names(&m_column_names);
  }
  tl->set_common_table_expr(&m_postparse);   // 反指：这个 Table_ref 引用的是该 CTE
```

### 5. ★ prelocking 与 `prelocking_placeholder`

prelocking = **语句执行前把"可能间接用到的表"一并锁上**。比如调用一个存储过程，过程体里查了 `t9`——解析顶层语句时根本不知道 `t9` 存在，所以 sp/trigger/view 展开阶段会**凭空 new 一批 `Table_ref` 追加到 `lex->query_tables` 尾部**，并打上占位标记：

```cpp
// [T14] sql/sp_head.cc
      Table_ref *table = new (tab_buff) Table_ref(
          key_buff, stab->db_length, key_buff + stab->db_length + 1,
          stab->table_name_length,
          key_buff + stab->db_length + 1 + stab->table_name_length + 1,
          stab->lock_type, mdl_lock_type);

      table->is_system_view = dd::get_dictionary()->is_system_view_name(
          table->db, table->table_name);
      table->cacheable_table = true;
      table->prelocking_placeholder = true;   // ← 关键：只用于隐式 LOCK TABLES
      table->belong_to_view = belong_to_view;
      table->trg_event_map = stab->trg_event_map;

      /* Everything else should be zeroed */
      **query_tables_last_ptr = table;        // 手工挂 next_global（不走 add_table_to_list）
      table->prev_global = *query_tables_last_ptr;
      *query_tables_last_ptr = &table->next_global;
```

占位的三个后果：

1. **开表时失败静默**——表可能根本不存在（过程体里那条语句这次不一定执行）：

```cpp
// [T2] sql/sql_base.cc —— open_tables()
  } else if (tables->prelocking_placeholder) {
    /*
      For the tables added by the pre-locking code, attempt to open
      the table but fail silently if the table does not exist.
      The real failure will occur when/if a statement attempts to use
      that table.
    */
    No_such_table_error_handler no_such_table_handler;
    thd->push_internal_handler(&no_such_table_handler);
```

2. **不参与重名检查**（`SELECT ... FROM t` 与过程里同名的 `t` 不该报 1066）：

```cpp
// [T2] sql/sql_base.cc —— find_dup_table()
    /*
      Skip if marked to be excluded (could be a derived table) or if
      entry is a prelocking placeholder.
    */
    if (res->query_block && !res->query_block->exclude_from_table_unique_test &&
        !res->prelocking_placeholder)
      break;
```

3. **不参与 `mysql_lock_tables()` 的 TABLE 收集**：

```cpp
// [T2] sql/sql_base.cc —— lock_tables()
          !(table->prelocking_placeholder &&
            table->table->s->tmp_table != NO_TMP_TABLE)) {
        *(ptr++) = table->table;
      }
```

锁类型降级规则也依赖它（非 LOCK TABLES 且过程不改数据时，占位表只需 `TL_READ`）：

```cpp
// [T2] sql/sql_base.cc
  /*
    table_list is placeholder for prelocking.
    Ignore prelocking_placeholder status for non "LOCK TABLE" statement's
    table_list objects when routine_modifies_data is false.
  */
  if (table_list->prelocking_placeholder &&
      (routine_modifies_data || thd->in_lock_tables))
    return TL_READ_NO_INSERT;
```

### 6. 生命周期

```
 ① 创建（解析期，每个 FROM 项一次）
    Query_block::add_table_to_list()
      new (thd->mem_root) Table_ref           ← 分配在语句 MEM_ROOT，无析构
      ├─ 填 db / table_name / alias / is_alias / is_fqtn
      ├─ set_derived_query_expression(table_name->sel)  → 可能是派生表/CTE
      ├─ MDL_REQUEST_INIT(&ptr->mdl_request, ...)       → 元数据锁请求（还没去拿锁）
      ├─ m_table_list.link_in_list(ptr, &ptr->next_local)   ← 挂 local 链
      └─ lex->add_to_query_tables(ptr)                      ← 挂 global 链
    Query_block::init_nested_join()/nest_last_join()        ← 造嵌套节点，改 embedding/join_list
    sp_head::add_used_tables_to_list() 等                   ← 追加 prelocking 占位符

 ② prepare / resolve（绑表之前）
    setup_tables()           → 分配 m_tableno/m_map，建 next_leaf
    resolve_derived()        → 视图/派生表展开
      merge：填 merge_underlying_list（把底层表搬进外层）→ 该 Table_ref 永不绑 TABLE*
      物化：set_uses_materialization()，执行期建临时表
    check_single_table() / updatable_base_table() / view_check_option() ...

 ③ 绑到 TABLE*（开表）
    open_tables() → open_table() → tl->table = table;   ← 见后文「开表」一节
    （视图/派生表/schema table/table function 走各自路径，
      merge 视图则 tl->table 始终为 nullptr ⇒ is_placeholder() 为真）

 ④ 执行
    JOIN::optimize() 用 tl->table->file->stats、tl->map()、tl->tableno()
    物化派生表：derived_result 填临时表，tl->table 指向它

 ⑤ 收尾
    close_thread_tables() 释放/归还 TABLE*（tl->table 随之失效或归缓存）
    Query_expression::cleanup() 做语义层清理
    Table_ref 本身不 delete —— 随 thd->mem_root 在语句结束时整体释放
    （PS 复用时靠 save_properties()/restore_properties() 保住可变形字段）
```

---

## 表的通用操作

### 1. 开表：open_table

> **本节源码坐标**：`[O1] sql/sql_base.cc` · `[O2] sql/table.cc` · `[O3] sql/table_cache.h` · `[O4] sql/table.h`
> **衔接**：MDL 五类/兼容矩阵/11 种锁类型/三种 duration、`Table_cache::m_lock` 16 分区、`LOCK_open`、`COND_open`、`LOCK_ha_data`、`dict_sys->mutex` 的层次图见「锁表」章；`Open_table_context` 五种退避 action、`refresh_version` 九条锁协议、TDC 内部结构见下一节「开表的三个疑难机制」。本节只讲**开表本身的完整流程与锁序列**。

#### 1.1 ★ MDL type 从哪来：由 `lock_descriptor()` 推导

开表时 `Table_ref::mdl_request` 的 type **不是** `open_table()` 临时决定的，而是在 `Table_ref` 构造时由 THR_LOCK 类型映射出来的：

```cpp
// [O4] sql/table.h —— 开表时 MDL type 的唯一推导入口
/**
  Derive type of metadata lock to be requested for table used by a DML
  statement from the type of THR_LOCK lock requested for this table.
*/
inline enum enum_mdl_type mdl_type_for_dml(enum thr_lock_type lock_type) {
  return lock_type >= TL_WRITE_ALLOW_WRITE
             ? (lock_type == TL_WRITE_LOW_PRIORITY ? MDL_SHARED_WRITE_LOW_PRIO
                                                   : MDL_SHARED_WRITE)
             : MDL_SHARED_READ;
}

// [O4] sql/table.h —— Table_ref 每个构造函数都做同一件事
  MDL_REQUEST_INIT(&mdl_request, MDL_key::TABLE, db, table_name,
                   mdl_type_for_dml(m_lock_descriptor.type), MDL_TRANSACTION);
                   // ★ type 由 thr_lock_type 推导；duration 默认 MDL_TRANSACTION

// [O2] sql/table.cc —— 语句执行前统一刷一遍（PS/SP 重执行、多表 UPDATE 改锁时用）
void init_mdl_requests(Table_ref *table_list) {
  for (; table_list; table_list = table_list->next_global)
    MDL_REQUEST_INIT(&table_list->mdl_request, MDL_key::TABLE, table_list->db,
                     table_list->table_name,
                     mdl_type_for_dml(table_list->lock_descriptor().type),
                     MDL_TRANSACTION);
}
```

⇒ **DML 的 MDL type 是 `TL_*` 的函数**（`TL_WRITE*`→SW/SWLP，其余→SR）。DDL / `LOCK TABLES` 不走这条映射，而是在 parser 里直接指定强锁，并在 `open_tables()` 开头用 `lock_table_names()` **批量**申请。

#### 1.2 `open_table_get_mdl_lock()`：`try_acquire_lock()` 与 `acquire_lock()` 的分工

```cpp
// [O1] sql/sql_base.cc
static bool open_table_get_mdl_lock(THD *thd, Open_table_context *ot_ctx,
                                    Table_ref *table_list, uint flags,
                                    MDL_ticket **mdl_ticket) {
  MDL_request *mdl_request = &table_list->mdl_request;   // ★ 类型来自 1.1 的推导
  MDL_request new_mdl_request;

  if (flags &
      (MYSQL_OPEN_FORCE_SHARED_MDL | MYSQL_OPEN_FORCE_SHARED_HIGH_PRIO_MDL)) {
    // PREPARE 用 MDL_SHARED 覆盖；I_S 查询用 MDL_SHARED_HIGH_PRIO 覆盖。
    // 目的是让 PREPARE / I_S 不被并发的 LOCK TABLES WRITE 拖住。
    MDL_REQUEST_INIT_BY_KEY(&new_mdl_request, &mdl_request->key,
                            (flags & MYSQL_OPEN_FORCE_SHARED_MDL)
                                ? MDL_SHARED
                                : MDL_SHARED_HIGH_PRIO,
                            MDL_TRANSACTION);
    mdl_request = &new_mdl_request;
  } else if (thd->variables.low_priority_updates &&
             mdl_request->type == MDL_SHARED_WRITE && ...) {
    // @@low_priority_updates=1：SW 降级为 SWLP，避免饿死并发 LOCK TABLES READ
    MDL_REQUEST_INIT_BY_KEY(&new_mdl_request, &mdl_request->key,
                            MDL_SHARED_WRITE_LOW_PRIO, MDL_TRANSACTION);
    mdl_request = &new_mdl_request;
  }

  if (flags & MYSQL_OPEN_FAIL_ON_MDL_CONFLICT) {
    // ★ 只有 I_S 扫描走这里：try_acquire_lock() 是「无等待」版本
    //   冲突就返回，绝不等 —— 否则与 LOCK TABLES / 存储函数里已锁的表
    //   形成 MDL 死锁检测器看不见的环（只能靠超时解开）
    if (thd->mdl_context.try_acquire_lock(mdl_request)) return true;
    if (mdl_request->ticket == nullptr) {
      my_error(ER_WARN_I_S_SKIPPED_TABLE, MYF(0), mdl_request->key.db_name(),
               mdl_request->key.name());
      return true;   // ★ 跳过这张表，而不是等待
    }
  } else {
    // ★ 正常开表：acquire_lock() 会等待。DML 请求的 S/SR/SW/SWLP 属于
    //   unobtrusive 集合 → MDL fast path（只做一次原子加，不进链表）；
    //   DDL 要的 SU/SRO/SNW/SNRW/X 属于 obtrusive → 必须进 m_rwlock 做完整判定。
    //   ⇒ 这就是「DML 快、DDL 慢」的根因。
    MDL_deadlock_handler mdl_deadlock_handler(ot_ctx);
    thd->push_internal_handler(&mdl_deadlock_handler);
    // ★ 若本次开表可退避重试，就把自己压成 DML 权重，甘当死锁牺牲者
    thd->mdl_context.set_force_dml_deadlock_weight(ot_ctx->can_back_off());

    bool result =
        thd->mdl_context.acquire_lock(mdl_request, ot_ctx->get_timeout());

    thd->mdl_context.set_force_dml_deadlock_weight(false);
    thd->pop_internal_handler();

    // ★ 失败但可退避 → 错误被吞掉，交给 Open_table_context 重试
    if (result && !ot_ctx->can_recover_from_failed_open()) return true;
  }
  *mdl_ticket = mdl_request->ticket;
  return false;
}
```

#### 1.3 ★ `open_table()`：锁序列主战场

```cpp
// [O1] sql/sql_base.cc
bool open_table(THD *thd, Table_ref *table_list, Open_table_context *ot_ctx) {
  TABLE *table = nullptr;
  TABLE_SHARE *share = nullptr;
  const char *key;
  size_t key_length;
  const char *alias = table_list->alias;
  uint flags = ot_ctx->get_flags();
  MDL_ticket *mdl_ticket = nullptr;
  int error = 0;

  // ★ 临时表与派生表不允许走这里
  assert(!is_temporary_table(table_list) && !table_list->is_derived());
  assert(table_list->is_view() || table_list->table == nullptr);

  if (check_stack_overrun(thd, STACK_MIN_SIZE_FOR_OPEN, (uchar *)&alias))
    return true;                                     // 开表很吃栈
  if (!(flags & MYSQL_OPEN_IGNORE_KILLED) && thd->killed) return true;

  // 只读事务里不允许申请表级写锁（日志表除外，否则 general/slow log 会哑掉）
  if (table_list->mdl_request.is_write_lock_request() && thd->tx_read_only &&
      !(flags & (MYSQL_LOCK_LOG_TABLE | MYSQL_OPEN_HAS_MDL_LOCK))) {
    my_error(ER_CANT_EXECUTE_IN_READ_ONLY_TRANSACTION, MYF(0));
    return true;
  }

  key_length = get_table_def_key(table_list, &key);  // ★ TDC / TC 的统一 key："db\0table"

  if ((flags & (MYSQL_OPEN_HAS_MDL_LOCK | MYSQL_OPEN_SECONDARY_ENGINE)) == 0) {
    /*
      ★① GRL 保护：非 LOCK TABLES 下要拿写锁 → 先拿 GLOBAL IX（STATEMENT 级），
         这样本语句会被活跃的 FLUSH TABLES WITH READ LOCK 挡住。
    */
    if (table_list->mdl_request.is_write_lock_request() &&
        !ot_ctx->has_protection_against_grl()) {
      MDL_request protection_request;
      if (thd->global_read_lock.can_acquire_protection()) return true;
      MDL_REQUEST_INIT(&protection_request, MDL_key::GLOBAL, "", "",
                       MDL_INTENTION_EXCLUSIVE, MDL_STATEMENT);
      ...
      if (thd->mdl_context.acquire_lock(&protection_request,
                                        ot_ctx->get_timeout())) return true;
      ot_ctx->set_has_protection_against_grl();
    }

    // ★② 表级 MDL（见 1.2）
    if (open_table_get_mdl_lock(thd, ot_ctx, table_list, flags, &mdl_ticket) ||
        mdl_ticket == nullptr) {
      return true;
    }
  } else {
    mdl_ticket = table_list->mdl_request.ticket;   // 调用方已持有锁
  }

  // ★③ open_strategy 分支：开表语义的分水岭
  if (table_list->open_strategy == Table_ref::OPEN_IF_EXISTS ||
      table_list->open_strategy == Table_ref::OPEN_FOR_CREATE) {
    bool exists;
    if (check_if_table_exists(thd, table_list, &exists)) return true;  // 问 DD
    if (!exists) {
      if (table_list->open_strategy == Table_ref::OPEN_FOR_CREATE && ...) {
        // ★ CREATE TABLE：表不存在 → 把 S 升级成 X（可能等待/死锁→退避）
        bool wait_result = thd->mdl_context.upgrade_shared_lock(
            table_list->mdl_request.ticket, MDL_EXCLUSIVE,
            thd->variables.lock_wait_timeout);
        if (wait_result) return true;
      }
      return false;    // ★ 静默返回 false：表不存在不是错误（IF NOT EXISTS 语义）
    }
  } else if (table_list->open_strategy == Table_ref::OPEN_STUB)
    return false;      // ★ OPEN_STUB：永远不打开底层表，但 MDL 照拿

retry_share : {
  Table_cache *tc = table_cache_manager.get_cache(thd);   // ★ 按 tid%16 选分区
  tc->lock();                                             // ★④ Table_cache::m_lock

  if (!table_list->is_view())
    table = tc->get_table(thd, key, key_length, &share);  // 见 1.4

  if (table) {
    /* 命中未使用的 TABLE */
    if (!(flags & MYSQL_OPEN_IGNORE_FLUSH)) {
      /*
        只持「本分区 table cache 锁」就敢比较 version，是因为：
        更新 refresh_version 与 TABLE_SHARE::version 必须同时持有 LOCK_open
        **和所有** table cache 锁；因此只持本分区锁时，二者不会被并发改写。
      */
      if (thd->open_tables &&
          thd->open_tables->s->version() != share->version()) {
        tc->release_table(thd, table);
        tc->unlock();
        (void)ot_ctx->request_backoff_action(
            Open_table_context::OT_REOPEN_TABLES, nullptr);
        return true;
      }
    }
    tc->unlock();                                         // ★★ 全程未碰 LOCK_open
    thd->status_var.table_open_cache_hits++;
    goto table_found;
  } else if (share) {
    // 有 share 但没有空闲 TABLE：加 LOCK_open 后 ++ref_count。
    // ★ 顺序很关键：先加 LOCK_open，再放 tc->lock()，与刷新路径严格对称
    mysql_mutex_lock(&LOCK_open);
    tc->unlock();
    share->increment_ref_count();
    goto share_found;
  } else {
    tc->unlock();
  }
}

  // ★⑤ 走到这里：table cache 里连 share 都没有 → 进 TDC / 读 DD
  mysql_mutex_lock(&LOCK_open);
  if (!(share = get_table_share_with_discover(
            thd, table_list, key, key_length,
            flags & MYSQL_OPEN_SECONDARY_ENGINE, &error))) {
    mysql_mutex_unlock(&LOCK_open);
    if (error == 7 && !thd->is_error())
      (void)ot_ctx->request_backoff_action(Open_table_context::OT_DISCOVER,
                                           table_list);
    return true;
  }

share_found:
  if (!(flags & MYSQL_OPEN_IGNORE_FLUSH)) {
    if (share->has_old_version()) {
      // 撞上「滚动 FLUSH」留下的旧版本 share：放引用 → 等旧版本消失 → retry
      release_table_share(share);
      mysql_mutex_unlock(&LOCK_open);
      bool wait_result =
          tdc_wait_for_old_version(thd, table_list->db, table_list->table_name,
                                   ot_ctx->get_timeout(), deadlock_weight);
      if (wait_result) return true;
      goto retry_share;
    }
    if (thd->open_tables &&
        thd->open_tables->s->version() != share->version()) {
      // FLUSH TABLES 推进了 refresh_version：退避，整体重开
      release_table_share(share);
      mysql_mutex_unlock(&LOCK_open);
      (void)ot_ctx->request_backoff_action(Open_table_context::OT_REOPEN_TABLES,
                                           nullptr);
      return true;
    }
  }

  mysql_mutex_unlock(&LOCK_open);           // ★⑥ 放 LOCK_open

  {
    // ★⑦ 无锁状态下再取一次 dd::Table —— 这次是给 handler::ha_open() 用的
    dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
    const dd::Table *table_def = nullptr;
    if (!(flags & MYSQL_OPEN_NO_NEW_TABLE_IN_SE) &&
        thd->dd_client()->acquire(share->db.str, share->table_name.str,
                                  &table_def)) {
      goto err_lock;
    }

    if (!(table = (TABLE *)my_malloc(key_memory_TABLE, sizeof(*table),
                                     MYF(MY_WME))))
      goto err_lock;

    error = open_table_from_share(          // ★⑧ 见 1.5
        thd, share, alias,
        ((flags & MYSQL_OPEN_NO_NEW_TABLE_IN_SE)
             ? 0
             : ((uint)(HA_OPEN_KEYFILE | HA_OPEN_RNDFILE | HA_GET_INDEX |
                       HA_TRY_READ_ONLY))),
        EXTRA_RECORD, thd->open_options, table, false, table_def);
    if (error) { ... goto err_lock; }
  }
  {
    // ★⑨ 新 TABLE 入本线程的 table cache 分区
    Table_cache *tc = table_cache_manager.get_cache(thd);
    tc->lock();
    if (tc->add_used_table(thd, table)) { tc->unlock(); goto err_lock; }
    tc->unlock();
  }
  thd->status_var.table_open_cache_misses++;

table_found:
  table->mdl_ticket = mdl_ticket;          // ★ TABLE 反向记住 MDL ticket
  table->next = thd->open_tables;
  thd->set_open_tables(table);
  table->reginfo.lock_type = TL_READ;      // 先假设读，lock_tables() 再校正

reset:
  table->reset();                          // 清优化期残留状态（TABLE 是复用的）
  table->set_created();
  table_list->set_updatable();
  table_list->set_insertable();
  table_list->table = table;               // ★ Table_ref ← TABLE 绑定完成
  table->init(thd, table_list);
  return false;

err_lock:
  mysql_mutex_lock(&LOCK_open);
  release_table_share(share);
  mysql_mutex_unlock(&LOCK_open);
  return true;
}
```

### ★ 精确加锁 / 放锁顺序表（与「锁层次图」呼应）

| # | 加锁 | 放锁 | 持锁期间做的事 | 备注 |
|---|---|---|---|---|
| 1 | `MDL:TABLE` SNW/SNRW/X（批量） | 事务/语句末 | — | 仅 DDL / `LOCK TABLES`，`lock_table_names()` |
| 2 | `MDL:GLOBAL` IX（STATEMENT） | 语句末 | — | 仅写锁请求且无 GRL 保护 |
| 3 | `MDL:TABLE` S/SR/SW/SWLP | 事务末 | — | `open_table_get_mdl_lock()` |
| 4 | `MDL:TABLE` S→X 升级 | — | `check_if_table_exists()` | 仅 `OPEN_FOR_CREATE` 且表不存在 |
| 5 | `Table_cache::m_lock`（tid%16） | 出 `retry_share` 块 | `get_table()` | **命中 TABLE 时全程不碰 `LOCK_open`** |
| 6 | `LOCK_open` | `share_found` 之后 | `++ref_count` / `get_table_share()` | 5→6 是「先加后放」，不可颠倒 |
| 6a | （`LOCK_open` 内）`mysql_cond_wait(&COND_open, &LOCK_open)` | — | 等别人的 `m_open_in_progress` | 原子释放 `LOCK_open` |
| 6b | **临时释放** `LOCK_open` | 读完 DD 立刻取回 | `dd_client->acquire()` + `open_table_def()` | 读 DD 会递归开表，绝不能持 `LOCK_open` |
| 6c | （放锁前）`MDL:SCHEMA` IX | 事务末 | — | 只在 TDC 未命中且未持有 IX 时 |
| 7 | — | `LOCK_open` | `has_old_version()` / version 比较 | |
| 8 | 无锁 | — | `dd_client->acquire()` 取 `dd::Table` | 给 `ha_open()` |
| 9 | 无锁（InnoDB 内部 `dict_sys_mutex_enter()`） | — | `open_table_from_share()` → `ha_open()` | 引擎锁由引擎自己管 |
| 10 | `Table_cache::m_lock` | 立即 | `add_used_table()` | 可能触发 `free_unused_tables_if_necessary()` |

★ **两条铁律**：(a) `LOCK_open` 与任一 `Table_cache::m_lock` 可同时持有，顺序只能是「先 `m_lock` → 后 `LOCK_open`」；(b) 持 `LOCK_open` 时**绝不**访问 DD、绝不调 `ha_open()`。

#### 1.4 `Table_cache`：查找 / 命中 / 淘汰

```cpp
// [O3] sql/table_cache.h
TABLE *Table_cache::get_table(THD *thd, const char *key, size_t key_length,
                              TABLE_SHARE **share) {
  TABLE *table;
  assert_owner();                          // ★ 调用者必须持本分区的 m_lock
  *share = nullptr;

  std::string key_str(key, key_length);
  const auto el_it = m_cache.find(key_str);      // ★ 按 table_cache_key 找 element
  if (el_it == m_cache.end()) return nullptr;    // ★ 连 share 都没有 → 返回 NULL
  Table_cache_element *el = el_it->second.get();

  *share = el->share;                      // ★ 至少把 share 带出去（即使没有空闲 TABLE）

  if ((table = el->free_tables.front())) {
    assert(!table->in_use);
    el->free_tables.remove(table);         // 从该表的空闲链摘下
    unlink_unused_table(table);            // ★ 从本分区全局 LRU 链摘下
    el->used_tables.push_front(table);     // 挂到该表的在用链
    table->in_use = thd;                   // ★ 副作用：标记归属
  }
  return table;                            // NULL ⇒ 只有 share，需新建 TABLE
}

void Table_cache::release_table(THD *thd, TABLE *table) {
  Table_cache_element *el =
      table->s->cache_element[table_cache_manager.cache_index(this)];
  assert_owner();
  assert(!table->s->has_old_version());    // ★ 旧版本 share 的 TABLE 不许回池
  table->in_use = nullptr;
  el->used_tables.remove(table);
  el->free_tables.push_front(table);       // 回到该表的空闲链头
  link_unused_table(table);                // ★ 挂到本分区 LRU 的「最近使用」端
  free_unused_tables_if_necessary(thd);    // ★ 超限则淘汰，且淘汰的是「最久未用」
}

void Table_cache::free_unused_tables_if_necessary(THD *thd) {
  // table_cache_size 可在线调小，所以要循环删到满足为止
  if (m_table_count > table_cache_size_per_instance && m_unused_tables) {
    mysql_mutex_lock(&LOCK_open);          // ★ 淘汰要碰 share → 必须 LOCK_open
    while (m_table_count > table_cache_size_per_instance && m_unused_tables) {
      TABLE *table_to_free = m_unused_tables;   // ★ LRU 头 = 最久未用
      remove_table(table_to_free);
      intern_close_table(table_to_free);        // ★ 真正 closefrm + 释放
      thd->status_var.table_open_cache_overflows++;
    }
    mysql_mutex_unlock(&LOCK_open);
  }
}
```

⇒ 读法：`m_cache` 是 `key → Table_cache_element` 的 hash，`Table_cache_element` 里挂 `free_tables`（空闲）与 `used_tables`（在用）两条链；`m_unused_tables` 是**跨表**的环形 LRU，只串空闲 TABLE，尾部为 MRU。淘汰阈值是每实例的 `table_open_cache / table_cache_instances`。

#### 1.5 `open_table_from_share()`：从 share 造出可用的 `TABLE`

```cpp
// [O2] sql/table.cc（节选，按执行顺序）
  new (outparam) TABLE();
  outparam->in_use = thd;
  outparam->s = share;                     // ★ TABLE → SHARE 单向指针

  if (!(prgflag & SKIP_NEW_HANDLER)) {
    // ★ 造 handler（ha_innobase / ha_innopart / …），分区表用 m_part_info 判定
    if (!(outparam->file = get_new_handler(share, share->m_part_info != nullptr,
                                           root, share->db_type())))
      goto err;
    // ★ handler 记住「我该去 share->ha_share 里取 Handler_share」
    if (outparam->file->set_ha_share_ref(&share->ha_share)) goto err;
  }

  records = 0;
  if ((db_stat & HA_OPEN_KEYFILE) || (prgflag & DELAYED_OPEN)) records = 1;
  if (prgflag & (READ_ALL + EXTRA_RECORD)) records++;   // EXTRA_RECORD ⇒ record[1]

  // ★ 一次分配：records 条记录缓冲 + null_bytes
  record = root->ArrayAlloc<uchar>(share->rec_buff_length * records +
                                   share->null_bytes);
  outparam->record[0] = record;
  if (records > 1) outparam->record[1] = record + share->rec_buff_length;
  else             outparam->record[1] = outparam->record[0];  // Safety

  outparam->field = root->ArrayAlloc<Field *>(share->fields + 1);
  record = (uchar *)outparam->record[0] - 1;          // ★ Field 的 ptr 从 1 起算
  outparam->null_flags = (uchar *)record + 1;         // ★ record[0] 开头就是 null_flags

  // ★ 新 record[0] 与 share->default_values 之间的偏移；
  //   之后每个 clone 出来的 Field 都要 move_field_offset() 平移到新记录上
  move_offset = outparam->record[0] - share->default_values + ...;

  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);  // ★ 每个 Field 深拷贝一份
    *field_ptr = new_field;
    new_field->init(outparam);
    new_field->move_field_offset(move_offset);        // ★ 指向本 TABLE 的 record[0]
  }
  (*field_ptr) = nullptr;                             // End marker

  // ★ key_info 深拷贝（见下方）
  if (share->key_parts) {
    if (create_key_part_field_with_prefix_length(outparam, root)) goto err;
  }

  // ★ 8 张位图一次分配（必须在生成列之前 —— fix_fields 会用位图）
  bitmap_size = share->column_bitmap_size;
  bitmaps = root->ArrayAlloc<uchar>(bitmap_size * 8);
  bitmap_init(&outparam->def_read_set, (my_bitmap_map *)bitmaps, share->fields);
  bitmap_init(&outparam->def_write_set, (my_bitmap_map *)(bitmaps + bitmap_size), ...);
  bitmap_init(&outparam->tmp_set, ...);
  bitmap_init(&outparam->cond_set, ...);
  bitmap_init(&outparam->def_fields_set_during_insert, ...);
  bitmap_init(&outparam->fields_for_functional_indexes, ...);
  bitmap_init(&outparam->pack_row_tmp_set, ...);
  bitmap_init(&outparam->read_set_internal, ...);
  outparam->default_column_bitmaps();      // read_set/write_set 指向默认位图

  /* 生成列（vfield[]）、默认值表达式（gen_def_fields_ptr[]）、
     CHECK 约束（table_check_constraint_list）在此 unpack 成 Item 树 */

  // ★★ 终于到引擎开表
  if ((ha_err = (outparam->file->ha_open(
           outparam, share->normalized_path.str, ... , table_def)))) {
      if (ha_err == HA_ERR_TABLE_DEF_CHANGED)      error = 7;  // → OT_DISCOVER
      else if (ha_err == HA_ERR_ROW_FORMAT_CHANGED) error = 8;  // → OT_FIX_ROW_TYPE
      goto err;
  }
  if (db_stat) thd->status_var.opened_tables++;
```

`key_info` 的深拷贝是**整块 memcpy**，只有前缀索引才额外 `new_field()`：

```cpp
// [O2] sql/table.cc
bool create_key_part_field_with_prefix_length(TABLE *table, MEM_ROOT *root) {
  TABLE_SHARE *share = table->s;
  KEY *key_info = nullptr;
  KEY_PART_INFO *key_part = nullptr;
  uint n_length;

  assert(share->key_parts);
  n_length =
      share->keys * sizeof(KEY) + share->key_parts * sizeof(KEY_PART_INFO);

  if (!(key_info = static_cast<KEY *>(root->Alloc(n_length)))) return true;
  table->key_info = key_info;                          // ★ 一次性两块，紧邻分配
  key_part = (reinterpret_cast<KEY_PART_INFO *>(key_info + share->keys));

  memcpy(key_info, share->key_info, sizeof(*key_info) * share->keys);
  memcpy(key_part, share->key_info[0].key_part,
         (sizeof(*key_part) * share->key_parts));
  // ★ KEY::rec_per_key / rec_per_key_float 是「指针」，memcpy 只是把指针值拷过来
  //   → TABLE 与 SHARE 共享同一份统计数组（浅共享）；
  //     而 KEY 数组本身与 KEY_PART_INFO 数组是 TABLE 私有的（深拷贝）

  for (KEY *key_info_end = key_info + share->keys; key_info < key_info_end;
       key_info++) {
    key_info->table = table;                           // ★ 回指本 TABLE
    key_info->key_part = key_part;                     // ★ 重挂到本 TABLE 的 key_part
    for (KEY_PART_INFO *key_part_end = key_part + key_info->actual_key_parts;
         key_part < key_part_end; key_part++) {
      Field *field = key_part->field = table->field[key_part->fieldnr - 1];
      // ★ 重挂：指向本 TABLE 新 clone 出来的 Field（不是 share->field[]）
      if (field->key_length() != key_part->length &&
          !field->is_flag_set(BLOB_FLAG)) {
        // 前缀索引：另造一个 length 被截短的 Field
        field = key_part->field = field->new_field(root, table);
        field->set_field_length(key_part->length);
      }
    }
    key_part += key_info->unused_key_parts;            // 跳过扩展 PK 后缀
  }
  return false;
}
```

最后 `TABLE::init()` 里那组断言，正是"TABLE 是循环复用的、回池前必须擦干净"的**可验证清单**：

```cpp
// [O2] sql/table.cc
  // ★ TABLE 是循环复用的：下面这些断言保证上一次用完已 reset 干净
  assert(!const_table && !nullable && !force_index && !force_index_order);
  assert(!force_index_group && insert_values == nullptr);
  assert(file->ft_handler == nullptr && !reginfo.impossible_range);
  assert(pos_in_table_list == nullptr);
  assert(!key_read);
  assert(merge_keys.is_clear_all() && possible_quick_keys.is_clear_all());

  // ★ prelocking_placeholder 的表不绑生成列：它的 cleanup 由真正用到它的
  //   子语句负责，这里绑了会漏掉 cleanup_value_generator_items()
  if (!pos_in_table_list->prelocking_placeholder) {
    bind_value_generators_to_fields();
  }
```

#### 1.6 开表锁序列时序图

```
        THD                                     共享资源 / 临界区
         │
 open_tables()
    ├─ mysql_ha_flush()                     （先清被 flush 的 HANDLER）
    ├─ lock_table_names() ─────────────────► MDL:TABLE SNW/SNRW/X   ← 仅 DDL/LOCK TABLES
    │                                         （批量一次申请，避免死锁与饥饿）
    └─ open_and_process_table() → open_table()
         │
 open_table()
    ①  GRL 保护（写锁请求且未持保护）
         └─────────────────────────────────► MDL:GLOBAL IX (STATEMENT)
    ②  open_table_get_mdl_lock()
         ├── try_acquire_lock()  ─ 仅 I_S，冲突即跳过（不等，避免隐形死锁环）
         └── acquire_lock()      ─ 其余；S/SR/SW/SWLP ⇒ MDL fast path（原子加）
                                            SU/SRO/SNW/SNRW/X ⇒ slow path（进链表）
                                            ─────────────────► MDL:TABLE S/SR/SW/…
    ③  OPEN_IF_EXISTS / OPEN_FOR_CREATE
         └── check_if_table_exists()；表不存在 ⇒ upgrade_shared_lock(S→X)
         OPEN_STUB ⇒ 直接 return false（不打开底层表，MDL 照拿）
    ④  tc = table_cache_manager.get_cache(thd)   // tid % table_cache_instances
         tc->lock() ══════════════════════════► Table_cache::m_lock[分区]
         tc->get_table(key)
            ├─ 命中空闲 TABLE ── tc->unlock() ─► goto table_found   ★全程不碰 LOCK_open
            ├─ 只有 share ────── LOCK_open+ → tc->unlock() → ++ref_count → share_found
            └─ 都没有 ────────── tc->unlock()
    ⑤  LOCK_open+ ══════════════════════════► LOCK_open
         get_table_share_with_discover() → get_table_share()（详见「表元信息」章）
            ├─ table_def_cache->find(key)
            ├─ miss → LOCK_open− → MDL:SCHEMA IX → LOCK_open+ → 重试 find
            ├─ m_open_in_progress → mysql_cond_wait(&COND_open, &LOCK_open)
            └─ 新建：alloc_table_share() → assign_new_table_id() → emplace
                     → ++ref_count → m_open_in_progress=true
                     → LOCK_open− ─────────► dd_client->acquire()（读 DD，可递归开表！）
                                             open_table_def() 填 SHARE
                     → LOCK_open+ → m_open_in_progress=false
                                  → mysql_cond_broadcast(&COND_open)
    ⑥  share_found：has_old_version() / version 不一致 ⇒ OT_REOPEN_TABLES 退避
         LOCK_open−
    ⑦  dd_client->acquire(db, table_name, &table_def)   ← 无 LOCK_open，给 ha_open 用
    ⑧  open_table_from_share()   ★ 全程无全局锁
         ├─ TABLE/field[]/record[0,1]/8 张位图/key_info 深拷贝（rec_per_key 浅共享）
         └─ file->ha_open(table_def) ─────► InnoDB：dict_sys_mutex_enter()
                                              ├ dict_table_check_if_in_cache_low()
                                              └ dd_open_table<dd::Table>() → dict_table_t
    ⑨  tc->lock() → add_used_table() → tc->unlock()
    ⑩  table_found：mdl_ticket 挂钩；table->next = thd->open_tables；TABLE::init()
```
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

> **本节源码坐标（全文只在节首列一次）**
> `[L1] sql/sql_base.cc` · `[L2] sql/lock.cc` · `[L3] sql/mdl.h` · `[L4] sql/mdl.cc` · `[L5] include/thr_lock.h` · `[L6] mysys/thr_lock.cc` · `[L7] sql/table_cache.h` · `[L8] sql/sql_class.h` · `[L9] sql/table.h` · `[L10] sql/sys_vars.cc` · `[L11] sql/debug_lock_order.cc` · `[L12] storage/innobase/include/dict0dict.h` · `[L13] storage/innobase/dict/dict0dict.cc` · `[L14] storage/innobase/handler/ha_innodb.cc`

> **衔接提示（已在前文覆盖，本节不再展开）**：`Table_ref::prelocking_placeholder` 的三种后果、开表时 `MDL_REQUEST_INIT` 的构造、`Table_ref::lock_descriptor()` 与 `Lock_descriptor`、以及 `TABLE::lock_position / lock_data_start / lock_count` 在 `MYSQL_LOCK` 两个并行数组中的下标含义，均见「开表」一节；`TABLE::open_by_handler` 对锁断言的豁免见「HANDLER」一节。本节从这些前提出发，只讲**锁是怎么被真正拿到的**。

#### 2.0 为什么是"两层锁"：MDL + thr_lock

一次 `UPDATE t SET c=c+1 WHERE id=1` 在 8.0 里要拿**两组完全不同语义**的锁。只有理解"为什么不能只留一层"，后面所有代码才有意义。

```
时间轴（单条 UPDATE，autocommit=1，InnoDB）
──────────────────────────────────────────────────────────────────────────
 T0 open_tables()
    │  a) MDL_key::SCHEMA(db)  ── scoped IX      （仅 DDL 才拿，DML 不拿）
    │  b) MDL_key::TABLE(db,t) ── MDL_SHARED_WRITE（SW）
    │     ├─ MDL_lock::m_rwlock 只是在改 ticket 队列的一瞬间持有
    │     └─ 拿不到 → MDL_wait::timed_wait()，受 @@lock_wait_timeout 约束
    │                 等待中由 MDL_context::find_deadlock() 做环检测
    ▼
 T1 lock_tables() → mysql_lock_tables()
    │  c) lock_external()：handler::ha_external_lock(F_WRLCK)
    │     InnoDB 在此标记 sql_stat_start=true、select_lock_type=LOCK_X
    │  d) thr_multi_lock()：把 THR_LOCK_DATA 挂进 TABLE_SHARE 的 THR_LOCK
    │     ├─ THR_LOCK::mutex 只在入队/出队一瞬间持有
    │     └─ 拿不到 → wait_for_lock() 条件变量睡眠（InnoDB 实际 lock_count=0，
    │                 这一层对 InnoDB 几乎无冲突，对 MyISAM/MEMORY 是主战场）
    ▼
 T2 执行：row_search_for_mysql() → InnoDB 行锁 / 意向表锁 IS,IX / X
    受 @@innodb_lock_wait_timeout 约束，由 InnoDB 自己的死锁检测兜底
──────────────────────────────────────────────────────────────────────────
 T3 ha_external_lock(F_UNLCK) → thr_multi_unlock() → 语句结束释放 MDL_STATEMENT
```

| 维度 | MDL（元数据锁） | thr_lock（server 层表锁） |
|---|---|---|
| 保护对象 | **表定义**（名字、列、DD 对象）不被并发 DDL 改掉 | **表数据**的读写互斥（对非事务引擎尤其关键） |
| 粒度/命名空间 | `MDL_key`：GLOBAL/BACKUP_LOCK/TABLESPACE/SCHEMA/TABLE/FUNCTION/… | 每 `TABLE_SHARE` 一个 `THR_LOCK` |
| 请求/凭证 | `MDL_request` → `MDL_ticket` | `THR_LOCK_DATA` |
| 谁决定加不加 | SQL 层按语句类型静态推导 | `handler::store_lock()` 由**引擎自己**决定返回几个 `THR_LOCK_DATA` |
| 死锁 | 有（`find_deadlock()` + victim 权重） | **没有**（靠"一次拿全 + 排序"协议避免，见 2.5） |
| 超时变量 | `lock_wait_timeout` | `lock_wait_timeout`（同一个） |
| InnoDB 参与度 | 全量参与 | 基本不参与（`lock_count()==0`） |

**一句话**：MDL 防"你读的时候我 `DROP TABLE`"；thr_lock 防"你写 MyISAM 表的时候我也写"。InnoDB 有自己的 MVCC + 行锁，所以对 thr_lock 层"交白卷"，但仍必须走 MDL，否则 DDL 与 DML 会互相踩踏。

#### 2.1 `lock_tables()`：`Table_ref` 链表 → `TABLE*` 数组 → `thd->lock`

```cpp
// [L1] sql/sql_base.cc
/**
  Lock all tables in a list.

  You can't call lock_tables() while holding thr_lock locks, as
  this would break the dead-lock-free handling thr_lock gives us.
  You must always get all needed locks at once.      // ← 协议核心，见 2.5
*/

bool lock_tables(THD *thd, Table_ref *tables, uint count, uint flags) {
  Table_ref *table;
  DBUG_TRACE;
  /*
    不能在 prelocked 模式下再遇到 requires_prelocking() 的语句：
    prelocking 是"一次把存储过程/触发器里会用到的表全打开全锁上"的算法，
    它本身要求"所有锁一次性拿齐"，嵌套会破坏这个前提。
  */
  assert(thd->locked_tables_mode <= LTM_LOCK_TABLES ||
         !thd->lex->requires_prelocking());
  /* 同一条语句不允许锁两次 */
  assert(thd->lex->lock_tables_state == Query_tables_list::LTS_NOT_LOCKED);

  if (!tables && !thd->lex->requires_prelocking()) {
    // 没有表也要打上 LTS_LOCKED 标记，保证同一语句不会二次进入本函数
    thd->lex->lock_tables_state = Query_tables_list::LTS_LOCKED;
    int ret = thd->decide_logging_format(tables);
    return ret;
  }

  /*
    关键分支：thd->locked_tables_mode == LTM_NONE 才真的去加锁。
    为什么不能只判 thd->lock？见注释里的经典例子：
    存储函数里 "drop table t3; create temporary t3; insert into t3;"
    drop 之后 thd->lock 变 0，但 locked_tables_mode 仍在，
    此时若再去锁临时表 t3 会内存泄漏。
  */
  if (!thd->locked_tables_mode) {
    assert(thd->lock == nullptr);  // You must lock everything at once
    TABLE **start, **ptr;

    // 在语句 MEM_ROOT 上分配 TABLE* 数组，容量 = 上层传进来的 count
    if (!(ptr = start = (TABLE **)thd->alloc(sizeof(TABLE *) * count)))
      return true;

    for (table = tables; table; table = table->next_global) {
      if (!table->is_placeholder() &&                  // 视图/派生表未物化 → 跳过
          !(table->prelocking_placeholder &&
            table->table->s->tmp_table != NO_TMP_TABLE)) {
        *(ptr++) = table->table;                       // ← 真正参与加锁的表
      }
    }
    // 上面被跳过的两类：
    //  1) is_placeholder()：视图、派生表、schema 占位符，没有底层 TABLE
    //  2) prelocking_placeholder && 临时表：prelocking 阶段刻意
    //     不调用 handler::store_lock()/external_lock()，改由子语句
    //     执行时的 handler::start_stmt() 告知引擎真实操作类型。
    //     原因见源码长注释：CREATE TEMPORARY ... IF NOT EXISTS 会让
    //     prelocking 阶段只看到"读"，执行阶段才看到"写"，
    //     InnoDB 会认为你违背了操作类型承诺而断言失败。

    DEBUG_SYNC(thd, "before_lock_tables_takes_lock");

    // 唯一的加锁入口；ptr-start 是实际收集到的表数（可能 < count）
    if (!(thd->lock =
              mysql_lock_tables(thd, start, (uint)(ptr - start), flags)))
      return true;

    DEBUG_SYNC(thd, "after_lock_tables_takes_lock");

    // 隐式 LOCK TABLES：prelocking 成功后进入 LTM_PRELOCKED
    if (thd->lex->requires_prelocking() &&
        thd->lex->sql_command != SQLCOM_LOCK_TABLES) {
      Table_ref *first_not_own = thd->lex->first_not_own_table();
      for (table = tables; table && table != first_not_own;
           table = table->next_global) {
        if (!table->is_placeholder()) {
          table->table->query_id = thd->query_id;
          if (check_lock_and_start_stmt(thd, thd->lex, table)) {
            mysql_unlock_tables(thd, thd->lock);       // 半途失败要回滚已拿的锁
            thd->lock = nullptr;
            return true;
          }
        }
      }
      mark_real_tables_as_free_for_reuse(first_not_own);
      thd->enter_locked_tables_mode(LTM_PRELOCKED);
    }
  } else {
    /*
      已在 LOCK TABLES 模式下：不再走 mysql_lock_tables()，
      只对由 IS 查询隐式打开的 DD 表补一个 ha_external_lock(F_RDLCK)，
      因为这条路没经过 mysql_lock_tables()，引擎不知道该用什么锁类型。
    */
    if (in_LTM(thd)) {
      for (table = tables; table; table = table->next_global) {
        TABLE *tbl = table->table;
        if (tbl && belongs_to_dd_table(table)) {
          assert(tbl->file->get_lock_type() == F_UNLCK);
          tbl->file->init_table_handle_for_HANDLER();
          tbl->file->ha_external_lock(thd, F_RDLCK);
        }
      }
    }

    Table_ref *first_not_own = thd->lex->first_not_own_table();
    for (table = tables; table && table != first_not_own;
         table = table->next_global) {
      if (table->is_placeholder()) continue;

      /*
        存储函数/触发器里不允许修改正在被外层语句使用的表。
        lock_descriptor().type >= TL_WRITE_ALLOW_WRITE 就是要"写"；
        若同一 TABLE_SHARE 已被本语句占用（query_id 相同）但
        TABLE 实例不同（query_id 不同）→ ER_CANT_UPDATE_USED_TABLE_IN_SF_OR_TRG
      */
      if (thd->locked_tables_mode >= LTM_PRELOCKED &&
          table->lock_descriptor().type >= TL_WRITE_ALLOW_WRITE) {
        for (TABLE *opentab = thd->open_tables; opentab;
             opentab = opentab->next) {
          if (table->table->s == opentab->s && opentab->query_id &&
              table->table->query_id != opentab->query_id) {
            my_error(ER_CANT_UPDATE_USED_TABLE_IN_SF_OR_TRG, MYF(0),
                     table->table->s->table_name.str);
            return true;
          }
        }
      }
      if (check_lock_and_start_stmt(thd, thd->lex, table)) return true;
    }

    // LOCK TABLES 下又需要 prelocking → LTM_PRELOCKED_UNDER_LOCK_TABLES
    if (thd->lex->requires_prelocking()) {
      mark_real_tables_as_free_for_reuse(first_not_own);
      thd->locked_tables_mode = LTM_PRELOCKED_UNDER_LOCK_TABLES;
    }
  }

  thd->lex->lock_tables_state = Query_tables_list::LTS_LOCKED;
  int ret = thd->decide_logging_format(tables);
  return ret;
}
```

**要点提炼**

1. `lock_tables()` 本身**不加任何锁**，它只是"收集 + 转交"。真正的加锁在 `mysql_lock_tables()`。
2. `LOCK TABLES` 语句的路径特殊：进入 `lock_tables()` 时 `locked_tables_mode` 已经是 `LTM_LOCK_TABLES`，因此走 `else` 分支，跳过了 `mysql_lock_tables()` —— 表在 `LOCK TABLES` 执行时已经被锁好并存进了 `thd->locked_tables_list`，后续语句只需要 `check_lock_and_start_stmt()`。
3. `count` 是**上限**而非实际数量：`ptr - start` 才是真实表数，因为 placeholder 与 prelocking 临时表会被过滤。

#### 2.2 `mysql_lock_tables()`：四步走

```cpp
// [L2] sql/lock.cc
/* Map the return value of thr_lock to an error from errmsg.txt */
static int thr_lock_errno_to_mysql[] = {0, ER_LOCK_ABORTED,
                                        ER_LOCK_WAIT_TIMEOUT, ER_LOCK_DEADLOCK};

MYSQL_LOCK *mysql_lock_tables(THD *thd, TABLE **tables, size_t count,
                              uint flags) {
  int rc;
  MYSQL_LOCK *sql_lock;
  // MYSQL_LOCK_IGNORE_TIMEOUT → 用 LONG_TIMEOUT（FLUSH/DD 内部操作）
  ulong timeout = (flags & MYSQL_LOCK_IGNORE_TIMEOUT)
                      ? LONG_TIMEOUT
                      : thd->variables.lock_wait_timeout;

  DBUG_TRACE;

  // ① 语义检查：log/rpl 表、performance_schema 表、只读模式等的合法性
  if (lock_tables_check(thd, tables, count, flags)) return nullptr;

  // ② 向每个 handler 索取 THR_LOCK_DATA，回填 TABLE 的三个下标
  if (!(sql_lock = get_lock_data(thd, tables, count, GET_LOCK_STORE_LOCKS)))
    return nullptr;

  if (!(thd->state_flags & Open_tables_state::SYSTEM_TABLES))
    THD_STAGE_INFO(thd, stage_system_lock);   // SHOW PROCESSLIST 看到 "System lock"

  ulonglong lock_start_usec = my_micro_time();

  // ③ 先 external_lock：把"我要读/写这张表"告诉存储引擎
  if (sql_lock->table_count &&
      lock_external(thd, sql_lock->table, sql_lock->table_count)) {
    reset_lock_data_and_free(&sql_lock);
    goto end;
  }

  // thr_multi_lock() 会就地重排数组！所以先复制一份到后半段，
  // 把"原始顺序"保留在前半段（解锁、remove 都要用原始顺序）
  memcpy(sql_lock->locks + sql_lock->lock_count, sql_lock->locks,
         sql_lock->lock_count * sizeof(*sql_lock->locks));

  // ④ 真正加 server 层表锁；出错时把 external_lock 一并回滚
  rc = thr_lock_errno_to_mysql[(int)thr_multi_lock(
      sql_lock->locks + sql_lock->lock_count, sql_lock->lock_count,
      &thd->lock_info, timeout)];

  if (rc) {
    if (sql_lock->table_count)
      (void)unlock_external(thd, sql_lock->table, sql_lock->table_count);
    reset_lock_data_and_free(&sql_lock);
    if (!thd->killed) my_error(rc, MYF(0));   // ER_LOCK_WAIT_TIMEOUT 等
  }

end:
  if (!(flags & MYSQL_OPEN_IGNORE_KILLED) && thd->killed) {
    thd->send_kill_message();
    if (sql_lock) { mysql_unlock_tables(thd, sql_lock); sql_lock = nullptr; }
  }
  if (thd->variables.session_track_transaction_info > TX_TRACK_NONE)
    track_table_access(thd, tables, count);

  thd->inc_lock_usec(my_micro_time() - lock_start_usec);
  return sql_lock;
}
```

★ **顺序不能反**：`external_lock` 必须在 `thr_multi_lock` **之前**。因为 `handler::external_lock()` 会 `reset_template()`、`sql_stat_start = true`，引擎需要先知道本次语句的读写意图；而且失败回滚时 `unlock_external` 也必须先做。

#### 2.3 `get_lock_data()`：`lock_count` 为什么可能 > 1

```cpp
// [L2] sql/lock.cc —— 这是理解 TABLE::lock_count / lock_data_start / lock_position 的源头
static MYSQL_LOCK *get_lock_data(THD *thd, TABLE **table_ptr, size_t count,
                                 uint flags) {
  uint i, tables, lock_count;
  MYSQL_LOCK *sql_lock;
  THR_LOCK_DATA **locks, **locks_buf, **locks_start;
  TABLE **to, **table_buf;

  assert((flags == GET_LOCK_UNLOCK) || (flags == GET_LOCK_STORE_LOCKS));

  // 第一遍：预算容量。
  //   tables     ← THR_LOCK_DATA 个数   （注意：file->lock_count() 可能 > 1）
  //   lock_count ← TABLE 个数
  for (i = tables = lock_count = 0; i < count; i++) {
    TABLE *t = table_ptr[i];
    // 非事务型临时表（MyISAM temp）根本没有 THR_LOCK_DATA
    if (t->s->tmp_table != NON_TRANSACTIONAL_TMP_TABLE) {
      tables += t->file->lock_count();
      lock_count++;
    }
  }

  /*
    为什么 THR_LOCK_DATA* 申请 2 倍？
    thr_multi_lock() 会就地重排这个数组，但它改不了 TABLE 里的下标；
    所以前半段保存"原始顺序"，后半段是给 thr_multi_lock() 打乱用的副本。
  */
  if (!(sql_lock = (MYSQL_LOCK *)my_malloc(
            key_memory_MYSQL_LOCK,
            sizeof(*sql_lock) + sizeof(THR_LOCK_DATA *) * tables * 2 +
                sizeof(table_ptr) * lock_count, MYF(0))))
    return nullptr;

  locks = locks_buf = sql_lock->locks = (THR_LOCK_DATA **)(sql_lock + 1);
  to    = table_buf = sql_lock->table  = (TABLE **)(locks + tables * 2);
  sql_lock->table_count = lock_count;

  for (i = 0; i < count; i++) {
    TABLE *table;
    enum thr_lock_type lock_type;
    THR_LOCK_DATA **org_locks = locks;

    if ((table = table_ptr[i])->s->tmp_table == NON_TRANSACTIONAL_TMP_TABLE)
      continue;
    lock_type = table->reginfo.lock_type;              // 由开表时 store_lock 阶段定下
    assert(lock_type != TL_WRITE_DEFAULT && lock_type != TL_READ_DEFAULT &&
           lock_type != TL_WRITE_CONCURRENT_DEFAULT);  // 三个 *__DEFAULT 必须已被解析

    locks_start = locks;
    /*
      引擎自己决定要交几个 THR_LOCK_DATA：
        MyISAM/MEMORY/HEAP → 1 个（表级锁实体）
        分区表 / MERGE     → N 个（每个子表一个，因为底层是 N 张物理表）
        InnoDB             → 0 个（lock_count() == 0，不参与 server 层表锁）
      GET_LOCK_UNLOCK 时传 TL_IGNORE，表示"我只是来取锁结构好解锁"。
    */
    locks = table->file->store_lock(
        thd, locks, (flags & GET_LOCK_UNLOCK) ? TL_IGNORE : lock_type);

    if (flags & GET_LOCK_STORE_LOCKS) {
      table->lock_position   = (uint)(to - table_buf);          // 在 table[] 中的下标
      table->lock_data_start = (uint)(locks_start - locks_buf); // 在 locks[] 中的起始
      table->lock_count      = (uint)(locks - locks_start);     // 占几个 THR_LOCK_DATA
    }
    *to++ = table;
    if (locks)
      for (; org_locks != locks; org_locks++) {                 // 回填 PSI 打点
        (*org_locks)->debug_print_param = (void *)table;
        (*org_locks)->m_psi = table->file->m_psi;
      }
  }
  /*
    最终 lock_count 用实际值（locks - locks_buf），不用预算的 tables：
    MERGE 表在"刚打开还没 attach 子表"时 store_lock() 会少返回，
    而 lock_count() 恒返回 attach 后的数量 —— 宁可多分配，不能越界。
  */
  sql_lock->lock_count = locks - locks_buf;
  return sql_lock;
}
```

> ★ **结论**：`TABLE::lock_count` 对分区表/MERGE 是 `N`，对 InnoDB 是 `0`，对 MyISAM 是 `1`。这正是 `MYSQL_LOCK` 要用**两个并行数组**（`table[]` 与 `locks[]`）加三个下标去映射的原因。

#### 2.4 `lock_external()`：把意图告诉引擎

```cpp
// [L2] sql/lock.cc
static int lock_external(THD *thd, TABLE **tables, uint count) {
  uint i;
  int lock_type, error;
  for (i = 1; i <= count; i++, tables++) {
    assert((*tables)->reginfo.lock_type >= TL_READ);
    lock_type = F_WRLCK;                       /* 默认：排他 */
    // reginfo.lock_type 落在 TL_READ..TL_READ_NO_INSERT 之间才是 F_RDLCK
    if ((*tables)->db_stat & HA_READ_ONLY ||
        ((*tables)->reginfo.lock_type >= TL_READ &&
         (*tables)->reginfo.lock_type <= TL_READ_NO_INSERT))
      lock_type = F_RDLCK;

    if ((error = (*tables)->file->ha_external_lock(thd, lock_type))) {
      print_lock_error(error, (*tables)->file->table_type());
      while (--i) {                            // 逐个回滚已成功的
        tables--;
        (*tables)->file->ha_external_lock(thd, F_UNLCK);
        (*tables)->current_lock = F_UNLCK;
      }
      return error;
    } else {
      (*tables)->db_stat &= ~HA_BLOCK_LOCK;
      (*tables)->current_lock = lock_type;
    }
  }
  return 0;
}
```

```cpp
// [L14] storage/innobase/handler/ha_innodb.cc —— InnoDB 收到 F_WRLCK 后做什么（节选）
  // note：InnoDB 在这里只做"记账"，并不真的去拿表锁
  m_prebuilt->sql_stat_start = true;      // 本 handle 开始一条新语句
  m_prebuilt->hint_need_to_fetch_extra_cols = 0;
  reset_template();

  if (lock_type == F_WRLCK) {
    /* If this is a SELECT, then it is in UPDATE TABLE ...
       or SELECT ... FOR UPDATE */
    m_prebuilt->select_lock_type = LOCK_X;      // ← 后续行锁按 X 走
    m_stored_select_lock_type = LOCK_X;
  }
  ut_ad(!(lock_type == F_RDLCK && m_prebuilt->select_lock_type == LOCK_X));

  if (lock_type != F_UNLCK) {
    *trx->detailed_error = 0;
    innobase_register_trx(ht, thd, trx);        // 把 trx 挂到 thd 上
  }
```

⇒ MySQL 调 `::external_lock` 只是**设一个"意向"表级锁**；真正的行为（一致性读视图何时开、行锁是 `LOCK_NONE` 还是 `LOCK_S`/`LOCK_X`）由隔离级别 + `is_dd_table/skip_locking` 在 `row_search_for_mysql()` 时才定。

#### 2.5 锁等待与死锁检测：两条完全不同的路

**thr_lock 侧：没有死锁检测，靠"协议"避免死锁**

```cpp
/* [L6] mysys/thr_lock.cc —— thr_multi_lock() 之前先排序 */
#define LOCK_CMP(A, B)                      \
  ((uchar *)(A->lock) - (uint)((A)->type) < \
   (uchar *)(B->lock) - (uint)((B)->type))

static void sort_locks(THR_LOCK_DATA **data, uint count) {
  THR_LOCK_DATA **pos, **end, **prev, *tmp;
  /* Sort locks with insertion sort (fast because almost always few locks) */
  for (pos = data + 1, end = data + count; pos < end; pos++) {
    tmp = *pos;
    if (LOCK_CMP(tmp, pos[-1])) {
      prev = pos;
      do { prev[0] = prev[-1]; }
      while (--prev != data && LOCK_CMP(tmp, prev[-1]));
      prev[0] = tmp;
    }
  }
}

enum enum_thr_lock_result thr_multi_lock(THR_LOCK_DATA **data, uint count,
                                         THR_LOCK_INFO *owner,
                                         ulong lock_wait_timeout) {
  THR_LOCK_DATA **pos, **end;
  if (count > 1) sort_locks(data, count);      // ← 按 (lock地址, type) 全局定序
  for (pos = data, end = data + count; pos < end; pos++) {
    enum enum_thr_lock_result result =
        thr_lock(*pos, owner, (*pos)->type, lock_wait_timeout);
    if (result != THR_LOCK_SUCCESS) {          // 有一个失败：把前面拿到的全放开
      thr_multi_unlock(data, (uint)(pos - data));
      return result;
    }
  }
  thr_lock_merge_status(data, count);
  return THR_LOCK_SUCCESS;
}
```

⇒ 这就是 `lock_tables()` 注释里 "You must always get all needed locks at once" 的**技术原因**：

```
  若允许 T1 先拿 A 再等 B，T2 先拿 B 再等 A → 死锁。
  thr_lock 没有检测环的能力，于是用两条硬约束替代：
    (1) 一次性提交全部请求（lock_tables 一次拿全）
    (2) 按 (THR_LOCK* 地址, thr_lock_type) 全局全序排序后串行获取
  ⇒ 等待图中不可能出现环。代价是"部分失败立刻全放"，宁可重来也不等。
```

**等待本身**：`wait_for_lock()` 在 `THR_LOCK::mutex` 保护下入 `read_wait`/`write_wait` 队列，然后 `mysql_cond_timedwait()` 睡在**自己的** `THD::suspend` 条件变量上。

```cpp
/* [L6] mysys/thr_lock.cc */
static enum enum_thr_lock_result wait_for_lock(struct st_lock_list *wait,
                                               THR_LOCK_DATA *data,
                                               THR_LOCK_INFO *owner,
                                               bool in_wait_list,
                                               ulong lock_wait_timeout) {
  struct timespec wait_timeout;
  enum enum_thr_lock_result result = THR_LOCK_ABORTED;

  if (!in_wait_list) {
    (*wait->last) = data;          /* Wait for lock：挂进等待队列尾 */
    data->prev = wait->last;
    wait->last = &data->next;
  }
  locks_waited++;
  data->cond = owner->suspend;     /* 让别人可以 abort 我 */

  enter_cond_hook(nullptr, data->cond, &data->lock->mutex,
                  &stage_waiting_for_table_level_lock, &old_stage, ...);

  set_timespec(&wait_timeout, lock_wait_timeout);
  while (!is_killed_hook(nullptr) || in_wait_list) {
    int rc = mysql_cond_timedwait(data->cond, &data->lock->mutex, &wait_timeout);
    if (data->cond == nullptr) break;        // 授权者把 cond 置 NULL = 拿到/被 abort
    if (is_timeout(rc)) { result = THR_LOCK_WAIT_TIMEOUT; break; }
  }

  if (data->cond || data->type == TL_UNLOCK) {
    if (data->cond) {                        // abort 或 timeout：从等待队列摘掉
      if (((*data->prev) = data->next)) data->next->prev = data->prev;
      else wait->last = data->prev;
      data->type = TL_UNLOCK;                /* No lock */
      wake_up_waiters(data->lock);
    }
  } else {
    result = THR_LOCK_SUCCESS;
  }
  mysql_mutex_unlock(&data->lock->mutex);
  return result;
}
```

**MDL 侧：有真正的死锁检测**。`MDL_context::acquire_lock()` 拿不到锁后：

```cpp
/* [L4] sql/mdl.cc */
  lock = ticket->m_lock;                      // 此时 m_rwlock 已被 try_acquire_lock_impl 写着
  m_wait.reset_status();                      // 清空自己的"等待槽"，防止错失唤醒
  if (lock->needs_notification(ticket)) lock->notify_conflicting_locks(this);
  mysql_prlock_unlock(&lock->m_rwlock);       // ← 睡眠前必须放开 m_rwlock！

  will_wait_for(ticket);                      // 在 wait-for 图上加一条边
  DEBUG_SYNC(get_thd(), "mdl_acquire_lock_wait");

  find_deadlock();                            // ← 环检测

  /* 需要通知 SE / 需要检查连接存活时，切成 1 秒一轮的短等待循环 */
  if (lock->needs_notification(ticket) || lock->needs_connection_check()) {
    while (cmp_timespec(&abs_shortwait, &abs_timeout) <= 0) {
      wait_status = m_wait.timed_wait(m_owner, &abs_shortwait, false,
                                      mdl_request->key.get_wait_state_name());
      if (wait_status != MDL_wait::WS_EMPTY) break;
      if (lock->needs_connection_check() && !m_owner->is_connected()) {
        if (!m_wait.set_status(MDL_wait::KILLED)) wait_status = MDL_wait::KILLED;
        break;
      }
      if (lock->needs_notification(ticket)) {
        mysql_prlock_wrlock(&lock->m_rwlock);
        lock->notify_conflicting_locks(this);
        mysql_prlock_unlock(&lock->m_rwlock);
      }
      set_timespec(&abs_shortwait, 1);
    }
  }
```

```cpp
// [L4] sql/mdl.cc —— find_deadlock() 与 victim 选择
void MDL_context::find_deadlock() {
  while (true) {
    // 每次搜索都用全新 visitor —— victim 选择算法依赖这一点
    Deadlock_detection_visitor dvisitor(this);
    MDL_context *victim;
    if (!visit_subgraph(&dvisitor)) break;      /* No deadlocks are found! */
    victim = dvisitor.get_victim();
    (void)victim->m_wait.set_status(MDL_wait::VICTIM);
    victim->unlock_deadlock_victim();
    if (victim == this) break;
    // 去掉一条"别人的"边可能还有别的环，所以要再搜一次
  }
}
```

```cpp
// [L3] sql/mdl.h —— 权重（越小越容易被牺牲）
  static const uint DEADLOCK_WEIGHT_CO  = 0;
  static const uint DEADLOCK_WEIGHT_DML = 25;
  static const uint DEADLOCK_WEIGHT_ULL = 50;
  static const uint DEADLOCK_WEIGHT_DDL = 100;

// [L4] sql/mdl.cc —— MDL_ticket::get_deadlock_weight()
  if (m_lock->key.mdl_namespace() == MDL_key::USER_LEVEL_LOCK)
    return DEADLOCK_WEIGHT_ULL;         // GET_LOCK() 是用户显式要的，别轻易杀
  if (m_lock->key.mdl_namespace() == MDL_key::GLOBAL ||
      m_type >= MDL_SHARED_UPGRADABLE)
    return DEADLOCK_WEIGHT_DDL;         // DDL 不期望收到死锁错误
  return DEADLOCK_WEIGHT_DML;           // 优先牺牲 DML
```

★ **MDL 与 thr_lock 的"跨层"联动**：`MDL_object_lock::notify_conflicting_locks()` 会调 `mysql_lock_abort_for_thread()`，后者直接 `thr_abort_locks_for_thread()` 打断对方正在等的 server 层表锁——这是 `MDL_SHARED`（HANDLER）升级到 X 时防死锁的标准手段：

```cpp
/* [L2] sql/lock.cc */
void mysql_lock_abort_for_thread(THD *thd, TABLE *table) {
  MYSQL_LOCK *locked;
  if ((locked = get_lock_data(thd, &table, 1, GET_LOCK_UNLOCK))) {
    for (uint i = 0; i < locked->lock_count; i++)
      thr_abort_locks_for_thread(locked->locks[i]->lock,
                                 table->in_use->thread_id());
    my_free(locked);
  }
}
```

**`lock_wait_timeout` vs `innodb_lock_wait_timeout`**

| | `lock_wait_timeout` | `innodb_lock_wait_timeout` |
|---|---|---|
| 作用域 | **server 层**：MDL 与 thr_lock | **仅 InnoDB 行锁/表锁** |
| 代码位置 | `mysql_lock_tables()` 里读出传给 `thr_multi_lock()`；MDL 走 `set_timespec(&abs_timeout, lock_wait_timeout)` | InnoDB `lock_sys` 等待 |
| 默认 | 31536000 秒（1 年） | 50 秒 |
| 报错 | `ER_LOCK_WAIT_TIMEOUT` (1205) | `ER_LOCK_WAIT_TIMEOUT` (1205)，但 `SHOW ENGINE INNODB STATUS` 里有行锁详情 |
| 可否动态 | 全局 + 会话 | 全局 + 会话 |

### 3. MDL 体系

#### 3.1 五个类各自干什么

| 类 | 生命周期/归属 | 职责 |
|---|---|---|
| `MDL_key` | 值语义，嵌在 request/lock 里 | **锁的名字**。`namespace + db + name`（+ 可选 column）打成一段定长 buffer，`MAX_MDLKEY_LENGTH = 1+NAME_LEN+1+NAME_LEN+1` |
| `MDL_request` | **外部**分配（通常在 `Table_ref` / 语句 MEM_ROOT 上） | 一次**未决的**请求：想要什么 type、什么 duration。拿不到时 `ticket == nullptr` |
| `MDL_ticket` | **MDL 子系统内部分配** | 已授予锁的凭证。持有 `m_type/m_ctx/m_lock/m_is_fast_path/m_hton_notified` |
| `MDL_context` | 每个 `THD` 一个 `thd->mdl_context` | 一个连接持有的全部 ticket（按 duration 三分）+ 等待/死锁检测的落脚点 |
| `MDL_lock` | 全局单例 `MDL_map mdl_locks`（无锁 hash + hazard pointer） | 某个 key 上的**锁对象本身**：granted/waiting 队列 + `m_rwlock` + fast-path 计数器 |

```cpp
/* [L3] sql/mdl.h —— 锁的名字 */
struct MDL_key {
 public:
  enum enum_mdl_namespace {
    GLOBAL = 0, BACKUP_LOCK, TABLESPACE, SCHEMA, TABLE, FUNCTION, PROCEDURE,
    TRIGGER, EVENT, COMMIT, USER_LEVEL_LOCK, LOCKING_SERVICE, SRID,
    ACL_CACHE, COLUMN_STATISTICS, RESOURCE_GROUPS, FOREIGN_KEY,
    CHECK_CONSTRAINT, NAMESPACE_END
  };
  const char *db_name() const   { return m_ptr + 1; }
  enum_mdl_namespace mdl_namespace() const { return (enum_mdl_namespace)(m_ptr[0]); }
  void mdl_key_init(enum_mdl_namespace mdl_namespace, const char *db,
                    const char *name) { m_ptr[0] = (char)mdl_namespace; ... }
};

/* 未决请求：外部（Table_ref / MEM_ROOT）分配 */
class MDL_request {
 public:
  enum_mdl_type type{MDL_INTENTION_EXCLUSIVE};
  enum_mdl_duration duration{MDL_STATEMENT};
  MDL_request *next_in_list{nullptr};      // 串进 MDL_request_list
  MDL_request **prev_in_list{nullptr};
  MDL_ticket *ticket{nullptr};             // 拿到后才非 NULL
  MDL_key key;
  const char *m_src_file{nullptr}; uint m_src_line{0};

  bool is_write_lock_request() const {
    return (type >= MDL_SHARED_WRITE && type != MDL_SHARED_READ_ONLY);
  }
  /* 强锁：DDL / LOCK TABLES 级别 */
  bool is_ddl_or_lock_tables_lock_request() const {
    return type >= MDL_SHARED_UPGRADABLE;
  }
};
#define MDL_REQUEST_INIT(R, P1, P2, P3, P4, P5) \
  (*R).init_with_source(P1, P2, P3, P4, P5, __FILE__, __LINE__)

/* 已授予的锁凭证：MDL 子系统内部分配 */
class MDL_ticket : public MDL_wait_for_subgraph {
 private:
  enum enum_mdl_type m_type;      // 实际授予的类型
  MDL_context *m_ctx;             // 属主
  MDL_lock *m_lock;               // 指向锁对象
  bool m_is_fast_path;            // 走没走 fast path（不入 m_granted 列表）
  bool m_hton_notified;           // 获取/释放时需不需要通知 SE
};

/* 每个连接一个 */
class MDL_context {
 public:
  bool try_acquire_lock(MDL_request *mdl_request);
  bool acquire_lock(MDL_request *mdl_request, Timeout_type lock_wait_timeout);
  bool acquire_locks(MDL_request_list *requests, Timeout_type lock_wait_timeout);
  bool upgrade_shared_lock(MDL_ticket *mdl_ticket, enum_mdl_type new_type,
                           Timeout_type lock_wait_timeout);
  void release_statement_locks();
  void release_transactional_locks();
  MDL_wait m_wait;
 private:
  MDL_ticket_store m_ticket_store;              // 按 duration 三分的 ticket 集合
  MDL_wait_for_subgraph *m_waiting_for{nullptr};// wait-for 图上的出边
  mysql_prlock_t m_LOCK_waiting_for;            // 保护上面这个指针
  bool m_needs_thr_lock_abort{false};
  bool m_force_dml_deadlock_weight{false};
};

/* [L4] sql/mdl.cc —— 锁对象本体 */
class MDL_lock {
  typedef unsigned short bitmap_t;              // 每种 mdl type 占一个 bit
  mysql_prlock_t m_rwlock;                      // 保护 granted/waiting 队列
  Ticket_list m_granted;                        // 已授予的 ticket 链表
  Ticket_list m_waiting;                        // 等待中的 ticket 链表
  std::atomic<fast_path_state_t> m_fast_path_state;  // packed 计数器 + 3 个标志位
  ulong m_hog_lock_count;                       // 连续授予的 X/SNRW/SNW 次数
  ulong m_piglet_lock_count;                    // 连续授予的 SW 次数
  uint  m_current_waiting_incompatible_idx;     // 当前用哪张优先级矩阵
  const MDL_lock_strategy *m_strategy;          // scoped 还是 object
};
```

★ **fast path / slow path**（8.0 的核心优化）：`S/SH/SR/SW` 这种"彼此兼容、也几乎不与自己冲突"的类型，获取时**只做一次原子加法**改 `m_fast_path_state`，完全不动 `m_rwlock` 和链表；`SU/SRO/SNW/SNRW/X` 这些"obtrusive"类型才走慢路径，真正加 `m_rwlock` 写锁、判矩阵、入队。

#### 3.2 锁类型矩阵（源码原文）

`[L4] sql/mdl.cc` 里 `MDL_lock::MDL_lock_strategy` 有两个实例：`m_scoped_lock_strategy`（GLOBAL/BACKUP_LOCK/TABLESPACE/SCHEMA/COMMIT 命名空间）和 `m_object_lock_strategy`（TABLE/FUNCTION/…）。

**scoped（schema 级）只有三种类型：IX、S、X**

```cpp
const MDL_lock::MDL_lock_strategy MDL_lock::m_scoped_lock_strategy = {
    /**
                 | Type of active   |
         Request |   scoped lock    |
          type   | IS(*)  IX   S  X |
        ---------+------------------+
        IS       |  +      +   +  + |
        IX       |  +      +   -  - |
        S        |  +      -   +  - |
        X        |  +      -   -  - |
      (*) IS 与一切兼容，所以根本不做记账
    */
    {MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED),                  // IS
     MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_INTENTION_EXCLUSIVE),     // IX
     0, 0, 0, 0, 0, 0, 0, 0,
     MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED) |
         MDL_BIT(MDL_INTENTION_EXCLUSIVE)},                          // X
    ...
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0},   // unobtrusive increment：只有 IX
    false, nullptr, nullptr,
    &MDL_lock::scoped_lock_fast_path_granted_bitmap, nullptr};
```

**object（表级）完整矩阵**

```
         Request  |  Granted requests for lock            |
          type    | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X  |
        ----------+---------------------------------------------+
        S         | +   +   +   +    +    +   +    +    +    -  |
        SH        | +   +   +   +    +    +   +    +    +    -  |
        SR        | +   +   +   +    +    +   +    +    -    -  |
        SW        | +   +   +   +    +    +   -    -    -    -  |
        SWLP      | +   +   +   +    +    +   -    -    -    -  |
        SU        | +   +   +   +    +    -   +    -    -    -  |
        SRO       | +   +   +   -    -    +   +    +    -    -  |
        SNW       | +   +   +   -    -    -   +    -    -    -  |
        SNRW      | +   +   -   -    -    -   -    -    -    -  |
        X         | -   -   -   -    -    -   -    -    -    -  |
      @note 若本 context 已持有"更强"的类型，会经 find_ticket() 直接授予。
      @note IX 不用于 object 锁。
```

**逐个用途**（依据 `[L3] sql/mdl.h` 的 `enum_mdl_type` 注释）：

| 类型 | 用在哪 | 关键点 |
|---|---|---|
| `MDL_INTENTION_EXCLUSIVE` (IX) | **只用于 scoped**（GLOBAL/SCHEMA）。DDL 先拿 schema 上的 IX，防 `DROP DATABASE` 与并发 DDL 打架 | 与 IX 互相兼容，与 S/X 冲突 |
| `MDL_SHARED` (S) | 只关心**元数据**、不看数据：存储过程、PREPARE；也被 HANDLER 借用（要兼容 `LOCK TABLES WRITE` 的 SNRW） | 与 X 冲突，与其余全部兼容 |
| `MDL_SHARED_HIGH_PRIO` (SH) | 填充 `INFORMATION_SCHEMA`、`SHOW` | ★ **高优先**：无视排队的 X 请求直接授予。代价：持有 SH 的连接**不许再等任何资源**，否则会让 X 饿死 |
| `MDL_SHARED_READ` (SR) | `SELECT`、子查询、`LOCK TABLES ... READ` | 只允许配 `TL_READ*` 表锁 |
| `MDL_SHARED_WRITE` (SW) | `INSERT/UPDATE/DELETE`、`SELECT ... FOR UPDATE` | DML 的标配 |
| `MDL_SHARED_WRITE_LOW_PRIO` (SWLP) | `INSERT/UPDATE/DELETE ... LOW_PRIORITY` | 优先级低于 SRO |
| `MDL_SHARED_UPGRADABLE` (SU) | `ALTER TABLE` 第一阶段 | 允许并发读写数据，可升级到 SNW/SNRW/X |
| `MDL_SHARED_READ_ONLY` (SRO) | `LOCK TABLES ... READ` | 与 SW/SWLP/SU 冲突 → 拿引擎读锁时不会打架 |
| `MDL_SHARED_NO_WRITE` (SNW) | `ALTER TABLE` 拷数据阶段 | 允许并发 SELECT，禁止 UPDATE |
| `MDL_SHARED_NO_READ_WRITE` (SNRW) | `LOCK TABLES ... WRITE` | 只与 S/SH 兼容，可升级到 X |
| `MDL_EXCLUSIVE` (X) | `CREATE/DROP/RENAME TABLE`，DDL 关键阶段 | 与一切冲突 |

#### 3.3 `acquire_lock()` 流程与锁升级

```cpp
/* [L4] sql/mdl.cc（骨架） */
bool MDL_context::acquire_lock(MDL_request *mdl_request,
                               Timeout_type lock_wait_timeout) {
  if (lock_wait_timeout == 0) {          // 零超时：走 try_acquire，不做死锁检测
    if (try_acquire_lock(mdl_request)) return true;
    if (!mdl_request->ticket) { my_error(ER_LOCK_WAIT_TIMEOUT, MYF(0)); return true; }
    return false;
  }
  MDL_lock *lock; MDL_ticket *ticket = nullptr;
  struct timespec abs_timeout; MDL_wait::enum_wait_status wait_status;
  set_timespec(&abs_timeout, lock_wait_timeout);          // 临界区外先把超时算好

  if (try_acquire_lock_impl(mdl_request, &ticket)) return true;
  if (mdl_request->ticket) return false;                  // 快路径直接成功

  lock = ticket->m_lock;                                  // m_rwlock 已被写锁持有
  m_wait.reset_status();
  if (lock->needs_notification(ticket)) lock->notify_conflicting_locks(this);
  mysql_prlock_unlock(&lock->m_rwlock);
  will_wait_for(ticket);                                  // wait-for 图加边
  find_deadlock();
  wait_status = m_wait.timed_wait(m_owner, &abs_timeout, true,
                                  mdl_request->key.get_wait_state_name());
  done_waiting_for();                                     // 摘边
  ...
}
```

★ **为什么 `acquire_locks()` 要排序**：`lock_table_names()` 用 `MDL_context::acquire_locks()`，它先把 `MDL_request_list` 排序再逐个获取 —— 与 thr_lock 的 `sort_locks()` 同理，保证两个 DDL 不会因加锁顺序相反而死锁。

**升级**：`upgrade_shared_lock()` 把 SU/SNRW 升成 X。`ALTER TABLE` 典型序列：`SU`（解析、准备）→ `SNW`（拷数据，别人还能读）→ `X`（最后换表名）。升级同样要过 `find_deadlock()`。

#### 3.4 三种 duration 与释放时机

```cpp
/* [L3] sql/mdl.h */
enum enum_mdl_duration {
  MDL_STATEMENT = 0,   /* 语句结束（或事务结束）时自动释放 */
  MDL_TRANSACTION,     /* 事务结束时自动释放 */
  MDL_EXPLICIT,        /* 跨越语句和事务，必须 MDL_context::release_lock() 显式释放 */
  MDL_DURATION_END
};
```

| duration | 谁在用 | 何时释放 |
|---|---|---|
| `MDL_STATEMENT` | 绝大多数 DML 的事务性引擎表、DD 临时访问 | `release_statement_locks()`：语句末尾；`ROLLBACK TO SAVEPOINT` 时 `rollback_to_savepoint()` |
| `MDL_TRANSACTION` | 事务内的表（`thd->in_active_multi_stmt_transaction()` 时自动升级） | `release_transactional_locks()`：COMMIT / ROLLBACK |
| `MDL_EXPLICIT` | `LOCK TABLES`、`HANDLER ... OPEN`、XA 挂起的锁 | `release_lock()` / `UNLOCK TABLES` / `HANDLER ... CLOSE` |

#### 3.5 scoped lock：DDL 先拿 schema 意向锁

```cpp
// [L1] sql/sql_base.cc —— 开表时若发现表不在 TDC 里，要求本线程持有 schema IX
  if (thd->mdl_context.owns_equal_or_stronger_lock(
          MDL_key::SCHEMA, db, "", MDL_INTENTION_EXCLUSIVE)) {
    break;    // 有 schema IX，说明是"自己"在动这个 schema，可以安全继续
  }
```

```
  DROP DATABASE works correctly with concurrent DDL:
    IX metadata locks in SCHEMA namespace are acquired for DDL statements
    which can update metadata in the schema (i.e. which acquire SU, SNW,
    SNRW and X locks on schema objects) and aren't acquired for DML.
```

⇒ **schema 上的 IX 与 schema 上的 S/X 冲突，但 IX 之间彼此兼容**。于是 `DROP DATABASE`（schema X）会被任何一个并发 DDL（schema IX）挡住，而两个并发 DDL（各自在不同表上）互不干扰。这是"意向锁"思想的经典实现，与 InnoDB 的 IS/IX 表锁同构。

### 4. thr_lock（server 层引擎表锁）体系

#### 4.1 三个结构体

```cpp
// [L5] include/thr_lock.h
extern mysql_mutex_t THR_LOCK_lock;              // 保护 thr_lock_thread_list

/* 锁的属主标识：地址唯一标识一个线程 */
struct THR_LOCK_INFO {
  my_thread_id thread_id;
  mysql_cond_t *suspend;            // 该线程睡觉用的条件变量（&THD::suspend）
};

/* 一次"锁请求"的实体：侵入式双向链表节点 */
struct THR_LOCK_DATA {
  THR_LOCK_INFO *owner{nullptr};    // 谁持有
  THR_LOCK_DATA *next{nullptr}, **prev{nullptr};
  THR_LOCK *lock{nullptr};          // 指向表级锁对象
  mysql_cond_t *cond{nullptr};      // 等待中：非 NULL = 还在等；被授予时置 NULL
  thr_lock_type type{TL_IGNORE};
  void *status_param{nullptr};      // MyISAM 用它传 mi_get_status/mi_update_status 的上下文
  void *debug_print_param{nullptr};
  struct PSI_table *m_psi{nullptr};
};

/* 每张表（每个 TABLE_SHARE::THR_LOCK）一个 */
struct THR_LOCK {
  LIST list{nullptr, nullptr, nullptr};   // 挂进全局 thr_lock_thread_list
  mysql_mutex_t mutex;                    // 保护下面四个队列（只在入队/出队瞬间持有）
  struct st_lock_list read_wait;
  struct st_lock_list read;
  struct st_lock_list write_wait;
  struct st_lock_list write;
  /* write_lock_count is incremented for write locks and reset on read locks */
  ulong write_lock_count{0};              // 连续授予的写锁数，防读饿死
  uint read_no_write_count{0};            // 当前 TL_READ_NO_INSERT 的个数
  void (*get_status)(void *, int){nullptr};
  void (*copy_status)(void *, void *){nullptr};
  void (*update_status)(void *){nullptr};   /* Before release of write */
  void (*restore_status)(void *){nullptr};  /* Before release of read */
  bool (*check_status)(void *){nullptr};    /* 判断能否 concurrent insert */
};
```

#### 4.2 锁类型枚举逐个说明

```cpp
// [L5] include/thr_lock.h
enum thr_lock_type {
  TL_IGNORE = -1,                    /* UNLOCK ANY LOCK：get_lock_data(GET_LOCK_UNLOCK) 用 */
  TL_UNLOCK,                         /* UNLOCK ANY LOCK */
  TL_READ_DEFAULT,                   /* 仅 parser 用！open_tables() 时按 binlog 格式
                                        和表类别（log table）解析成 TL_READ 或
                                        TL_READ_NO_INSERT */
  TL_READ,                           /* 普通读锁 */
  TL_READ_WITH_SHARED_LOCKS,         /* SELECT ... LOCK IN SHARE MODE */
  TL_READ_HIGH_PRIORITY,             /* 高于 TL_WRITE，允许 concurrent insert */
  TL_READ_NO_INSERT,                 /* 读，但不允许 concurrent insert */
  TL_WRITE_ALLOW_WRITE,              /* 写锁，但允许别的线程也读也写（BDB 遗留语义） */
  TL_WRITE_CONCURRENT_DEFAULT,       /* 仅 parser 用 → thd->insert_lock_default */
  TL_WRITE_CONCURRENT_INSERT,        /* concurrent insert 用的写锁，允许读 */
  TL_WRITE_DEFAULT,                  /* 仅 parser 用 → thd->update_lock_default */
  TL_WRITE_LOW_PRIORITY,             /* 优先级低于 TL_READ 的写锁 */
  TL_WRITE,                          /* 普通写锁 */
  TL_WRITE_ONLY                      /* 直接报错拒绝新请求（不排队） */
};
```

| 类型 | 语义 | 典型场景 |
|---|---|---|
| `TL_READ` | 共享读，允许别的读、也允许 concurrent insert | 普通 `SELECT` |
| `TL_READ_WITH_SHARED_LOCKS` | 共享读 + 后续会加行共享锁 | `SELECT ... LOCK IN SHARE MODE` |
| `TL_READ_HIGH_PRIORITY` | 读锁但**高于** `TL_WRITE` | `SELECT HIGH_PRIORITY` |
| `TL_READ_NO_INSERT` | 读，**禁止** concurrent insert | `LOCK TABLES ... READ`（源码注释：现在只为它服务） |
| `TL_WRITE_ALLOW_WRITE` | 标记"有人在读写"，但不排斥别人的读写 | 历史 BDB 语义；现在是兼容矩阵中最宽松的一档 |
| `TL_WRITE_CONCURRENT_INSERT` | 写，但表尾部可并发插入 | MyISAM `INSERT` 到无空洞的表 |
| `TL_WRITE_LOW_PRIORITY` | 写，但优先级低于所有读 | `INSERT/UPDATE ... LOW_PRIORITY` |
| `TL_WRITE` | 普通排他写 | `UPDATE/DELETE/INSERT` 的常规路径 |
| `TL_WRITE_ONLY` | 不排队，新请求直接 `THR_LOCK_ABORTED` | 表被 `FLUSH TABLES`/删除时的"关门"态 |

#### 4.3 `thr_lock()` 核心逻辑

```cpp
// [L6] mysys/thr_lock.cc
enum enum_thr_lock_result thr_lock(THR_LOCK_DATA *data, THR_LOCK_INFO *owner,
                                   enum thr_lock_type lock_type,
                                   ulong lock_wait_timeout) {
  THR_LOCK *lock = data->lock;
  enum enum_thr_lock_result result = THR_LOCK_SUCCESS;
  struct st_lock_list *wait_queue;

  data->next = nullptr; data->cond = nullptr;
  data->type = lock_type;
  data->owner = owner;                       /* Must be reset ! */

  MYSQL_START_TABLE_LOCK_WAIT(locker, &state, data->m_psi, PSI_TABLE_LOCK, lock_type);
  mysql_mutex_lock(&lock->mutex);            // ← 临界区开始

  if ((int)lock_type <= (int)TL_READ_NO_INSERT) {
    /* ================= 请求读锁 ================= */
    if (lock->write.data) {
      /* 已有写锁。三种情况可以"插队"直接给：
         (a) 同一线程自己持有
         (b) 已有写锁类型 < TL_WRITE_CONCURRENT_INSERT（即 ALLOW_WRITE）
         (c) 已有写锁是 CONCURRENT_INSERT 且我要的是 HIGH_PRIORITY 以上 */
      if (thr_lock_owner_equal(data->owner, lock->write.data->owner) ||
          (lock->write.data->type < TL_WRITE_CONCURRENT_INSERT) ||
          ((lock->write.data->type == TL_WRITE_CONCURRENT_INSERT) &&
           ((int)lock_type <= (int)TL_READ_HIGH_PRIORITY))) {
        (*lock->read.last) = data;           /* Add to running FIFO */
        data->prev = lock->read.last;
        lock->read.last = &data->next;
        if (lock_type == TL_READ_NO_INSERT) lock->read_no_write_count++;
        locks_immediate++;
        goto end;
      }
      if (lock->write.data->type == TL_WRITE_ONLY) {
        data->type = TL_UNLOCK;
        result = THR_LOCK_ABORTED;           /* Can't wait for this one */
        goto end;
      }
    } else if (!lock->write_wait.data ||                          // 没有排队的写
               lock->write_wait.data->type <= TL_WRITE_LOW_PRIORITY ||
               lock_type == TL_READ_HIGH_PRIORITY ||
               has_old_lock(lock->read.data, data->owner)) {      // 我已有读锁
      (*lock->read.last) = data;             /* 直接给 */
      ...
      goto end;
    }
    /* 有活跃写锁，或有高优先写锁在排队 → 让位给写者 */
    wait_queue = &lock->read_wait;
  } else {
    /* ================= 请求写锁 ================= */
    // 引擎不支持 concurrent insert 时，把 CONCURRENT_INSERT 升级成 TL_WRITE
    if (lock_type == TL_WRITE_CONCURRENT_INSERT && !lock->check_status)
      data->type = lock_type = thr_upgraded_concurrent_insert_lock;
    ...
  }
  /* Can't get lock yet;  Wait for it */
  result = wait_for_lock(wait_queue, data, owner, false, lock_wait_timeout);
  MYSQL_END_TABLE_LOCK_WAIT(locker);
  return result;
end:
  mysql_mutex_unlock(&lock->mutex);
  MYSQL_END_TABLE_LOCK_WAIT(locker);
  return result;
}
```

**读写优先级与 `write_lock_count`**：释放锁后 `wake_up_waiters()` 决定叫醒谁，这是"写优先 + 防饿死"的实现：

```cpp
static void wake_up_waiters(THR_LOCK *lock) {
  if (!lock->write.data) {                   /* 没有活跃写锁 */
    data = lock->write_wait.data;
    if (!lock->read.data) {                  /* 完全没有锁在用 */
      /* 先放 TL_WRITE / TL_WRITE_ONLY 优先级的写锁 */
      if (data &&
          (data->type != TL_WRITE_LOW_PRIORITY || !lock->read_wait.data ||
           lock->read_wait.data->type < TL_READ_HIGH_PRIORITY)) {
        if (lock->write_lock_count++ > max_write_lock_count) {
          /* ★ 连续授予太多把写锁 → 清零并唤醒所有等待的读锁，防读饿死 */
          lock->write_lock_count = 0;
          if (lock->read_wait.data) { free_all_read_locks(lock, false); goto end; }
        }
        for (;;) {
          /* 从 write_wait 摘下，放进 write */
          (*lock->write.last) = data;
          if (data->type == TL_WRITE_CONCURRENT_INSERT &&
              (*lock->check_status)(data->status_param))
            data->type = TL_WRITE;           /* Upgrade lock */
          mysql_cond_t *cond = data->cond;
          data->cond = nullptr;              /* 标记"已授予" */
          mysql_cond_signal(cond);
          ...
        }
      }
    }
  }
}
```

`thr_unlock()` 则是对称地从队列摘除、按类型调 `update_status`（写）/ `restore_status`（读）、再 `wake_up_waiters()`。

### 5. ★ 闩锁（mutex / latch）全景

> 本节回答"**保护数据结构**的那些短命锁"，与第 3/4 节"**保护语义对象**的长命锁"是两个层次。判断标准很简单：**闩锁临界区里绝不允许等待 MDL 或行锁**；反过来，持 MDL/thr_lock 时可以去取闩锁。

#### 5.1 `LOCK_open`：8.0 里它已经不是"全局大锁"了

```cpp
// [L1] sql/sql_base.cc 的权威注释（这就是 8.0 的答案）
/**
  LOCK_open protects the following variables/objects:

  1) The table_def_cache
     This is the hash table mapping table name to a table share object.
  2) last_table_id
     Generation of a new unique table_map_id ...
  3) LOCK_open protects the initialisation of the table share object and
     all its members, however, it does not protect reading the .frm file
     from where the table share is initialised. In get_table_share, the
     lock is temporarily released while opening the table definition in
     order to allow a higher degree of concurrency.
  4) share->ref_count is updated each time a new table object is created
     that refers to a table share. This update is protected by LOCK_open.
  5) oldest_unused_share, end_of_unused_share and share->next/prev ...
  6) table_def_shutdown_in_progress ... (需 LOCK_open + 所有 table cache mutex)
  7) refresh_version  (需 LOCK_open + 所有 table cache mutex)
  8) share->version   (需 LOCK_open + 所有 table cache mutex)
  9) share->m_flush_tickets
*/
mysql_mutex_t LOCK_open;
```

★ **8.0 的变化**：5.x 时代 `LOCK_open` 是"开表大锁"，几乎每个 `open_table()` 都要抢。8.0 把热路径拆走了——`TABLE` 对象的取放走 **`Table_cache::m_lock`（分区）**，只有创建/销毁 `TABLE_SHARE`、改 `table_def_cache`、LRU 淘汰、版本刷新才碰 `LOCK_open`。所以现在的定位是"**TDC 元数据目录锁**"，不再是"开表锁"。

#### 5.2 `Table_cache::m_lock`（★ 分区锁）

```cpp
// [L7] sql/table_cache.h
/**
  Cache for open TABLE objects.

  The idea behind this cache is that most statements don't need to
  go to a central table definition cache to get a TABLE object and
  therefore don't need to lock LOCK_open mutex.
  Instead they only need to go to one Table_cache instance (the
  specific instance is determined by thread id) and only lock the
  mutex protecting this cache.
  DDL statements that need to remove all TABLE objects from all caches
  need to lock mutexes for all Table_cache instances, but they are rare.
*/
class Table_cache {
 private:
  /**
    The table cache lock protects the following data:
    1) m_unused_tables list.
    2) m_cache hash.
    3) used_tables, free_tables lists in Table_cache_element objects.
    4) m_table_count
    5) the element in TABLE_SHARE::cache_element[] for this cache
    6) in_use member in TABLE object.
    7) 另外：改 refresh_version / table_def_shutdown_in_progress /
       TABLE_SHARE::version 需要同时持有所有 cache 的 mutex
  */
  mysql_mutex_t m_lock;
  std::unordered_map<std::string, std::unique_ptr<Table_cache_element>> m_cache;
  TABLE *m_unused_tables;
  ulong m_table_count;
};

class Table_cache_manager {
 public:
  /** Maximum supported number of table cache instances. */
  static const int MAX_TABLE_CACHES = 64;
  /** Default number of table cache instances */
  static const int DEFAULT_MAX_TABLE_CACHES = 16;
  Table_cache *get_cache(THD *thd) {
    return &m_table_cache[thd->thread_id() % table_cache_instances];   // ← 按 thread_id 分片
  }
 private:
  Table_cache m_table_cache[MAX_TABLE_CACHES];
};
```

```cpp
// [L10] sql/sys_vars.cc —— 分区数现状
static Sys_var_ulong Sys_table_cache_instances(
    "table_open_cache_instances", "The number of table cache instances",
    READ_ONLY GLOBAL_VAR(table_cache_instances), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, Table_cache_manager::MAX_TABLE_CACHES),   // 1..64
    DEFAULT(Table_cache_manager::DEFAULT_MAX_TABLE_CACHES),  // 默认 16
    BLOCK_SIZE(1), ...);
```

⇒ **8.0 默认 16 个分区**（可选 1–64），**只读变量**（改需重启），按 `thread_id % table_cache_instances` 选分区。取一张已缓存的 `TABLE` 只需抢 1/16 的锁，这是 8.0 高并发开表吞吐的主要来源。

#### 5.3 `LOCK_thd_data`

```cpp
// [L8] sql/sql_class.h
  /**
    Protects THD data accessed from other threads.
    The attributes protected are:
    - thd->is_killable (used by KILL statement and shutdown).
    - thd->user_vars (user variables, inspected by monitoring)
    Is locked when THD is deleted.
  */
  mysql_mutex_t LOCK_thd_data;
```

层次上它是**最外层**的会话级锁：`KILL`、线程生命周期、`ha_data` 备份/恢复（`Attachable_transaction`）都靠它。规则是"**持有它时不要再往下取别的锁**"。

#### 5.4 `LOCK_ha_data`

```cpp
// [L9] sql/table.h
  mysql_mutex_t LOCK_ha_data;  /* To protect access to ha_data */
```

`[L11] sql/debug_lock_order.cc` 的依赖图显示 `LOCK_open → TABLE_SHARE::LOCK_ha_data`，且这条边带 `LOOP FLAG`——历史上它俩有过环。它保护 `TABLE_SHARE::ha_data[]`（引擎挂在 share 上的私有数据），**只保护非临时表**（临时表的 share 不共享）。

#### 5.5 `COND_open`

```cpp
// [L1] sql/sql_base.cc —— 不是锁，是条件变量，和 LOCK_open 配对
/**
  COND_open synchronizes concurrent opening of the same share:

  If a thread calls get_table_share, it releases the LOCK_open mutex while
  reading the definition from file. If a different thread calls
  get_table_share for the same share at this point in time, it will find
  the share in the TDC, but with the m_open_in_progress flag set to true.
  This will make the (second) thread wait for the COND_open condition,
  while the first thread completes opening the table definition.

  When the first thread is done reading the table definition, it will set
  m_open_in_progress to false and broadcast the COND_open condition.
*/
mysql_cond_t COND_open;
```

```cpp
  while (share->m_open_in_progress)
    mysql_cond_wait(&COND_open, &LOCK_open);     // get_table_share 里

  mysql_mutex_lock(&LOCK_open);
  share->m_open_in_progress = false;
  mysql_cond_broadcast(&COND_open);              // open_table_def 结束时
```

⇒ ★ 意义：**"读表定义"是慢 IO，不能占着 `LOCK_open`**。于是先插一个 `m_open_in_progress=true` 的占位 share，放锁去读 DD，别人撞见占位就睡 `COND_open`，读完广播唤醒重查。

#### 5.6 MDL 子系统内部的锁

| 锁 | 位置 | 保护什么 | 持有者/时机 |
|---|---|---|---|
| `MDL_lock::m_rwlock`（`mysql_prlock_t`） | `[L4] sql/mdl.cc` | 某 key 的 `m_granted`/`m_waiting` ticket 链表、`m_hog_lock_count`、`m_piglet_lock_count`、fast-path 标志位 | 只在"判断能否授予 + 入队/出队"这一瞬持有；**睡眠前必定释放**。读偏好以免引入假死锁 |
| `MDL_wait::m_LOCK_wait_status` + `m_COND_wait_status` | `[L3] sql/mdl.h` | 每个 context 的等待槽 `m_wait_status` | 等待期间由别的线程（授权者/死锁检测器）短暂持有，用于 `set_status(GRANTED/VICTIM/KILLED/TIMEOUT)` 并 signal |
| `MDL_context::m_LOCK_waiting_for`（prlock） | `[L3] sql/mdl.h` | `m_waiting_for` 指针（wait-for 图的出边） | `will_wait_for()` / `done_waiting_for()` / `visit_subgraph()` |

```cpp
/* [L3] sql/mdl.h */
class MDL_wait {
 public:
  // WS_EMPTY since EMPTY conflicts with #define in system headers on some platforms.
  enum enum_wait_status { WS_EMPTY = 0, GRANTED, VICTIM, TIMEOUT, KILLED };
  bool set_status(enum_wait_status result_arg);
  enum_wait_status timed_wait(MDL_context_owner *owner, struct timespec *abs_timeout,
                              bool signal_timeout, const PSI_stage_info *wait_state_name);
 private:
  mysql_mutex_t m_LOCK_wait_status;
  mysql_cond_t  m_COND_wait_status;
  enum_wait_status m_wait_status;
};
```

#### 5.7 InnoDB 侧

**`dict_sys->mutex`**（`[L12] storage/innobase/include/dict0dict.h`）

```cpp
/* 注意：8.0 里旧名 dict_mutex_enter() 已改名为 dict_sys_mutex_enter() */
#define dict_sys_mutex_enter() mutex_enter(&dict_sys->mutex)
#define dict_sys_mutex_exit()  mutex_exit(&dict_sys->mutex)

struct dict_sys_t {
  // note: 保护 dict_table_t / dict_index_t
  DictSysMutex mutex;          /*!< mutex protecting the data dictionary;
                               protects also the disk-based dictionary system
                               tables; this mutex serializes CREATE TABLE and
                               DROP TABLE, as well as reading the dictionary
                               data for a table from system tables */
  row_id_t row_id;             /*!< the next row id to assign */
  hash_table_t *table_hash;    /*!< hash table of the tables, based on name */
  hash_table_t *table_id_hash; /*!< hash table of the tables, based on id */
  ...
};
```

这是 InnoDB 数据字典缓存的**全局大锁**，序列化 `CREATE/DROP TABLE` 与表定义的加载/淘汰。

**`dict_table_stats_lock()`**（`[L13] storage/innobase/dict/dict0dict.cc`）—— 每表一个 **RW latch**：

```cpp
void dict_table_stats_lock(dict_table_t *table, ulint latch_mode) {
  ut_ad(table != nullptr);
  ut_ad(table->magic_n == DICT_TABLE_MAGIC_N);

  os_once::do_or_wait_for_done(&table->stats_latch_created,
                               dict_table_stats_latch_alloc, table);

  if (table->stats_latch == nullptr) {
    /* This is a dummy table object that is private in the current
       thread and is not shared between multiple threads, thus we
       skip any locking. */
    return;                                   // ← 私有快照对象，无锁
  }
  switch (latch_mode) {
    case RW_S_LATCH: rw_lock_s_lock(table->stats_latch, UT_LOCATION_HERE); break;
    case RW_X_LATCH: rw_lock_x_lock(table->stats_latch, UT_LOCATION_HERE); break;
    case RW_NO_LATCH: [[fallthrough]];
    default: ut_error;
  }
}
```

保护 `stat_n_rows / stat_clustered_index_size / stat_sum_of_other_index_sizes / stat_modified_counter` 等统计值。`handler::info()`/`ha_statistics` 路径会用它；**传 `HA_STATUS_NO_LOCK` 即可跳过**（呼应「表统计信息」一节）。

**InnoDB 其它常见 latch**：`dict_table_t::mutex`、`index->lock`（rw_lock，保护索引树结构）、`lock_sys` 全局 latch、`trx->mutex`、`fil_space->latch`、`buf_page` 的 `buf_block_t::lock`（页面 latch）、`autoinc_lock`。它们全部位于**最内层**。

#### 5.8 ★ 锁层次（lock order）图

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │ L0  会话/生命周期                                                        │
  │     THD::LOCK_thd_data            （KILL、THD 析构、ha_data 换出）        │
  │       └─ THD::LOCK_thd_sysvar（必须在 LOCK_thd_data 之内、               │
  │                                 LOCK_global_system_variables 之外）      │
  ├────────────────────────────────────────────────────────────────────────┤
  │ L1  语义锁：元数据（跨语句/事务，可长时间持有，可睡眠）                    │
  │     MDL_key::GLOBAL → BACKUP_LOCK → SCHEMA(scoped IX) → TABLE           │
  │       ├─ 瞬时：MDL_lock::m_rwlock      （改 ticket 队列）                │
  │       ├─ 瞬时：MDL_wait::m_LOCK_wait_status（被别人叫醒）                │
  │       └─ 瞬时：MDL_context::m_LOCK_waiting_for（wait-for 图加/摘边）      │
  ├────────────────────────────────────────────────────────────────────────┤
  │ L2  语义锁：server 层表锁 thr_lock                                       │
  │     THR_LOCK::mutex（入队/出队瞬间）→ 睡在 THD::suspend 条件变量          │
  │     全局链表由 THR_LOCK_lock 保护                                        │
  ├────────────────────────────────────────────────────────────────────────┤
  │ L2' 旁路：开表/关表的"目录"锁（只在 open/close 路径，不参与 DML 执行）      │
  │     LOCK_open  ──配对──  COND_open                                      │
  │        ↓（LOCK_open → TABLE_SHARE::LOCK_ha_data，debug_lock_order 有登记）│
  │     Table_cache::m_lock（16 个分区，按 thread_id 选）                    │
  │        ↓                                                                │
  │     TABLE_SHARE::LOCK_ha_data                                           │
  ├────────────────────────────────────────────────────────────────────────┤
  │ L3  引擎 latch（InnoDB，最内层，持有时间必须极短）                         │
  │     dict_sys->mutex（dict_sys_mutex_enter）→ dict_table_t::mutex         │
  │       → stats_latch（rw_lock_t，HA_STATUS_NO_LOCK 可跳过）               │
  │       → index->lock → 页面 latch（buf_block_t::lock）                    │
  │     lock_sys 全局 latch / trx->mutex / fil_space->latch                  │
  └────────────────────────────────────────────────────────────────────────┘

  箭头方向 = 允许的获取顺序（只能从上往下，不能回头）。

  实测依据（[L11] sql/debug_lock_order.cc 的 DEPENDENCY GRAPH 样例）：
      LOCK_open  →  mutex/innodb/dict_sys_mutex
      LOCK_open  →  mutex/innodb/dict_table_mutex
      LOCK_open  →  mutex/sql/TABLE_SHARE::LOCK_ha_data   （LOOP FLAG）
      LOCK_open  →  cond/sql/COND_open
      LOCK_open  →  mutex/sql/MDL_wait::LOCK_wait_status
      LOCK_open  →  mutex/mysys/THR_LOCK_myisam / THR_LOCK_heap / THR_LOCK_open
      LOCK_table_cache → mutex/innodb/dict_sys_mutex
      LOCK_table_cache → mutex/myisam/MYISAM_SHARE::intern_lock
```

**谁先谁后、为什么（避免死锁的协议）**

1. ★ **MDL 永远在 thr_lock 之前**。理由：`MDL_SHARED` 的注释写得很直白——SNRW 升级 X 时若对方正等着本线程持有的表级锁，就会形成"MDL 等 thr_lock"的环；解法是 MDL 层用 `notify_conflicting_locks()` → `mysql_lock_abort_for_thread()` → `thr_abort_locks_for_thread()` **主动打断**下层的表锁等待。即：**上层可以撤销下层的等待，反过来不行**。
2. **目录锁（`LOCK_open` / `Table_cache::m_lock`）在 MDL 之后、引擎 latch 之前**。开表时一定先 `MDL_context::acquire_lock()`，拿到 MDL 才去 TDC 找 share。`get_table_share()` 里那句 `thd->mdl_context.owns_equal_or_stronger_lock(MDL_key::SCHEMA, db, "", MDL_INTENTION_EXCLUSIVE)` 就是这个顺序的断言。
3. **引擎 latch 永远在最内**。InnoDB 的 `ha_innobase::external_lock()` / `info()` 都是在 MDL + thr_lock 已经就位后才可能碰到 `dict_sys->mutex` 与 `stats_latch`。
4. **同层内也要定序**，否则照样死锁：
   - thr_lock：`thr_multi_lock()` 先按 `(THR_LOCK* 地址, type)` 排序，且"一次拿全、部分失败立即全放"。
   - MDL：`acquire_locks()` 先排序请求（注释明说 DDL 之间因此通常不会死锁）；`LOCK TABLES` 这种天然乱序的场景退化到靠 `DEADLOCK_WEIGHT_DML` 选 victim。
5. ★ **等锁时手里不能有任何闩锁**。`wait_for_lock()` 在睡 `THD::suspend` 前会 `mysql_mutex_unlock(&lock->mutex)`；`MDL_context::acquire_lock()` 在 `timed_wait()` 前会 `mysql_prlock_unlock(&lock->m_rwlock)`。这是"闩锁不能跨越等待"这条铁律的直接体现。
6. **例外都有显式标记**：`debug_lock_order.cc` 对 `LOCK_open → TABLE_SHARE::LOCK_ha_data`、`LOCK_open → THD::LOCK_current_cond` 打了 `LOOP FLAG`，意味着这些边依赖额外的上下文才安全，属于被工具登记在案的历史遗留。

**一句话收束**：`lock_tables()` 是 `Table_ref` → `TABLE*` → `MYSQL_LOCK` 的**收集器**；`mysql_lock_tables()` 是 `external_lock` + `thr_multi_lock` 的**执行器**；MDL 负责"别在我读的时候改表定义"、thr_lock 负责"别在我写的时候碰数据"、闩锁负责"别在我改链表/缓存的时候看它们"——三层各司其职，靠"由外到内、等待必放锁"的协议串成一个无环的世界。
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

#### ★ `rea_create_base_table`：上面那句里的"写 DD + 建 SE 表"到底怎么排的

上面那句把 `mysql_create_table_no_lock` 的内部压缩成了一条链。**真正编排"DD 写入 ↔ SE 建表"顺序的是 `rea_create_base_table()`**（`sql/sql_table.cc`），它是整条 CREATE TABLE 的**核心**。DD 侧的字段映射见 [`dd/dd.md`](dd/dd.md) §2.6，这里讲**编排**。

**① 先造对象树，再决定要不要存**

```cpp
  std::unique_ptr<dd::Table> table_def_res = dd::create_table(
      thd, sch_obj, table_name, create_info, create_fields, key_info, keys,
      keys_onoff, fk_key_info, fk_keys, check_cons_spec, file);

  if (!table_def_res) return true;

  dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
```

**② ★ 两条分支：`do_not_store_in_dd` 与否**

```cpp
  if (do_not_store_in_dd) {
    /*
      Clean up code assumes that SE supports atomic DDL if do_not_store_in_dd
      was requested, so we can simply rollback our changes.

      ha_create_table() won't work correctly if dd::Table object is not stored
      in the data-dictionary.

      For data-dictionary tables we rely on Dictionary_client::store() to update
      their table definition.
    */
    assert(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
    assert(no_ha_table);
    assert(!dd::get_dictionary()->get_dd_table(db, table_name));

    *table_def_ptr = std::move(table_def_res);
    table_def = table_def_ptr->get();
  } else {
    bool result = thd->dd_client()->store(table_def_res.get());

    if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
        !thd->is_plugin_fake_ddl())
      result = trans_intermediate_ddl_commit(thd, result);

    if (result) return true;

    if (thd->dd_client()->acquire_for_modification(db, table_name, &table_def))
      return true;
  }
```

三个要点：

- **`do_not_store_in_dd`** 只用于 DD 自己的表（自举期还写不了），靠三个 `assert` 保证"调用方一定是原子 DDL + 不建 SE 表"；清理时直接回滚即可
- ★ **`trans_intermediate_ddl_commit()`**：**只有非原子 DDL 引擎才需要**。因为这类引擎的 DDL 无法参与事务回滚，DD 的改动必须先**落盘提交**，否则引擎建表失败时 DD 里的记录撤不掉
- `store()` 之后要 `acquire_for_modification()` —— 拿到的 `table_def` 才能被 `ha_create_table()` 修改（引擎回填 `se_private_data`）

**③ `no_ha_table`：只建元数据不建 SE 部分时，分区表要显式开表验证**

```cpp
      /*
        For partitioned tables we can't find some problems with table
        until table is opened. Therefore in order to disallow creation
        of corrupted tables we have to try to open table as the part
        of its creation process.
        In cases when both .FRM and SE part of table are created table
        is implicitly open in ha_create_table() call.
        In cases when we create .FRM without SE part we have to open
        table explicitly.
      */
```

即：**分区表的某些问题只有开表才暴露**，所以没走 `ha_create_table()`（不会隐式开表）时必须显式 `open_table_def` + `open_table_from_share` 验一遍。验证失败且已提交过，则要显式 `dd::drop_table()` + 中间提交。

**④ 引擎建表 + 失败清理**

```cpp
  if (ha_create_table(thd, path, db, table_name, create_info, false, false,
                      table_def)) {
    /*
      Remove table from data-dictionary if it was added and rollback
      won't do this automatically.
    */
    if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
      /* We ignore error from dd_drop_table() as we anyway return 'true' failure below. */
      bool result = dd::drop_table(thd, db, table_name, *table_def);
      if (!thd->is_plugin_fake_ddl())
        (void)trans_intermediate_ddl_commit(thd, result);
    }
    return true;
  }
```

注释直说：**回滚不会自动清掉 DD 里的记录**，所以必须显式 `dd::drop_table()`。而原子 DDL 引擎（如 InnoDB）不需要——事务回滚会连 DD 一起撤。

**⑤ ★ binlog 用哪个 cache，是这里决定的**

```cpp
  /*
    If the SE supports atomic DDL, we can use the trx binlog cache.
    Otherwise we must use the statement cache.
  */
  if (binlog_to_trx_cache != nullptr)
    *binlog_to_trx_cache =
        (create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
```

⇒ **原子 DDL 引擎 → trx cache**（DDL 与 binlog 在同一事务提交，保证一致性）；**非原子 → statement cache**（DDL 无法回滚，binlog 只能按语句记）。这个出参一路传回 `mysql_create_table` 的第四段，决定 `write_bin_log()` 的行为。

**⑥ `post_ddl` 钩子**

```cpp
  if ((create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
      create_info->db_type->post_ddl)
    *post_ddl_ht = create_info->db_type;
```

原子 DDL 引擎可注册 `post_ddl`，在事务提交**之后**再做收尾（此时元数据已可见）。

**小结——顺序本身就携带语义**：

```
dd::create_table()（造对象树）
  → store()（写 DD）
  → [非原子引擎] trans_intermediate_ddl_commit()（先落盘）
  → acquire_for_modification()（拿到可改的 table_def）
  → ha_create_table()（引擎建表 + 回填 se_private_data）
  → 失败则 [非原子引擎] 显式 dd::drop_table() + 中间提交
  → 决定 binlog cache 类型 / 登记 post_ddl
```

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

> **边界**：本篇只讲 **SQL 层如何决策**（问引擎、该降级还是报错）。三种算法在 InnoDB 内部的实现见 [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md) 的对应章节（该文档已完整覆盖，不必在本篇重复）：
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

## `TABLE_SHARE` 与 `TABLE`：成员全景与生命周期（对象视角）

> **本章边界**：前几章是"**操作视角**"（开表/锁表/DDL/缓存池）。本章是"**对象视角**"——这两个类本身长什么样、谁持有谁、活多久。
> 不重复：`open_table_from_share()` 的 `clone()`→`init()`→`move_field_offset()` 三步见 [`field.md`](field.md) 第五章；`record[0]` 字节布局见 [`infra/structure/data_repr.md`](../infra/structure/data_repr.md)；引擎侧如何消费位图见 [`handler.md`](handler.md)。
>
> 源码坐标：`[T1] sql/table.h` · `[T2] sql/table.cc` · `[T3] sql/sql_base.cc` · `[T4] sql/dd_table_share.cc` · `[T5] sql/table_cache.h`

**一句话定位**：`TABLE_SHARE` 是**表定义的内存模板**（全实例 1 份，与会话无关）；`TABLE` 是**一次打开的实例**（N 份，含 `THD*`、`handler*`、行缓冲、位图、优化期产物）。

### 1. `TABLE_SHARE` 成员全景

#### 1.0 先看"自包含"性质（理解销毁的钥匙）

```cpp
// [T2] —— SHARE 自带 MEM_ROOT，绝大多数成员都分配在这块 root 上
/**
  Release resources (plugins) used by the share and free its memory.
  TABLE_SHARE is self-contained -- it's stored in its own MEM_ROOT.
  Free this MEM_ROOT.
*/
void TABLE_SHARE::destroy() { ... }
```

⇒ SHARE 的销毁是"**整块内存释放**"，不需要逐个成员 free。

#### 1.1 标识组：★ `table_cache_key` 为什么是 key

```cpp
// [T1]
  LEX_CSTRING table_cache_key{nullptr, 0};
  LEX_CSTRING db{nullptr, 0};
  LEX_CSTRING table_name{nullptr, 0};
  LEX_STRING path{nullptr, 0};
  LEX_CSTRING normalized_path{nullptr, 0};
```

```cpp
// [T1] —— db / table_name 只是指向 key 缓冲区内部的指针（靠嵌入的 '\0' 分隔）
  void set_table_cache_key(char *key_buff, size_t key_length) {
    table_cache_key.str = key_buff;
    table_cache_key.length = key_length;
    db.str = table_cache_key.str;
    db.length = strlen(db.str);
    table_name.str = db.str + db.length + 1;
    table_name.length = strlen(table_name.str);
  }
```

```cpp
// [T3] —— key 的拼法："db\0table_name\0"
static size_t create_table_def_key(const char *db_name, const char *table_name,
                                   char *key) {
  return strmake(strmake(key, db_name, NAME_LEN) + 1, table_name, NAME_LEN) -
         key + 1;
}
```

★ 三点：① 它是 **TDC 的哈希键**、**MDL key 的后缀**、**线程临时表链表的查找键**——一个缓冲三处复用；② 靠嵌入 `\0` 分隔，`db`/`table_name` 不单独分配内存；③ 临时表尾部追加 `server_id` + `pseudo_thread_id`（8 字节），使同名临时表在会话间天然隔离。

#### 1.2 列与布局组（模板行长什么样）

```cpp
// [T1]
  Field **field{nullptr};          ///< 模板列数组（长度 fields+1，末元素 nullptr 作结束标记）
  uchar *default_values{nullptr};  ///< 模板行 = 默认值的字节镜像
  ulong reclength{0};              /* Recordlength */
  ulong stored_rec_length{0};      /* 不含"只生成不存储"的生成列 */
  uint rec_buff_length{0};         /* TABLE::record[] 每份缓冲的实际分配大小 */
  uint null_bytes{0}, last_null_bit_pos{0};
  uint fields{0}, null_fields{0}, blob_fields{0}, varchar_fields{0};
  uint vfields{0};                 ///< 生成列数
  uint *blob_field{nullptr};       ///< blob 列在 field 数组中的下标快查表
  MY_BITMAP all_set;               ///< "全 1"位图，供 TABLE::use_all_columns()
```

⇒ `share->field[i]->ptr` 落在 `default_values` 里（**模板**）；`rec_buff_length` 比 `reclength` 略大（尾部 safety/引擎私有区）。字节布局细节见 `data_repr.md`。

#### 1.3 键与索引组（完全静态）

```cpp
// [T1]
  uint keys{0}, key_parts{0}, max_key_length{0};
  uint primary_key{0};
  uint next_number_index{0};       /* autoincrement key number */
  uint next_number_key_offset{0};
  uint next_number_keypart{0};
  Key_map keys_in_use;             /* 未被 DISABLE KEYS 禁掉的索引（含 invisible）*/
  Key_map visible_indexes;
  Key_map keys_for_keyread;        /* 可用于 covering index 的索引 */
```

> ⚠️ **8.0.39 命名**：自增键是 `next_number_index`（不是 `next_number_key`）；可见索引是**位图** `visible_indexes`（不是 `visible_index_count`）。无主键时 `primary_key == MAX_KEY`。

#### 1.4 引擎与选项组

```cpp
// [T1]
  plugin_ref db_plugin{nullptr};     /* 引擎插件引用（带引用计数）*/
  inline handlerton *db_type() const {
    return db_plugin ? plugin_data<handlerton *>(db_plugin) : NULL;
  }
  enum row_type row_type = {}, real_row_type = {};
  tmp_table_type tmp_table{NO_TMP_TABLE};
  const CHARSET_INFO *table_charset{nullptr};
  uint db_create_options{0};         /* HA_CREATE_INFO::table_options 拷贝 */
  uint db_options_in_use{0};         /* 引擎真正生效的选项 */
  bool system{false};
  bool db_low_byte_first{false};     /* ★ 字节序——Field 整数读写据此决定 */
```

⇒ **SHARE 持有引擎插件的引用**——这是它必须先于所有 TABLE 存在、最后才销毁的原因。`db_low_byte_first` 由引擎汇报，是典型的"模板级常量"。

#### 1.5 ★ 生命周期组

```cpp
// [T1]
  TABLE_SHARE *next{nullptr}, **prev{nullptr};   /* 全局"未使用 SHARE"双向链 */
  Table_cache_element **cache_element{nullptr};  /* ★ 反向索引：我在各 cache 实例里的投影 */
  Wait_for_flush_list m_flush_tickets;           /* FLUSH 等待者队列 */
  Table_id table_map_id;                         /* 行格式 binlog 的 table_id */
 private:
  unsigned int m_ref_count{0};                   /* 多少个 TABLE 引用我 */
  unsigned long m_version{0};                    /* 与全局 refresh_version 比对判定过期 */
```

```cpp
// [T1] —— 引用计数只能在 LOCK_open 下改
  unsigned int increment_ref_count() {
    assert(assert_ref_count_is_locked(this));
    assert(!m_open_in_progress);
    return ++m_ref_count;
  }
  bool has_old_version() const { return version() != refresh_version; }
  void clear_version();                          /* 置 0 = 打过期标记，等最后引用释放后自删 */
```

★ 三点：① `m_ref_count` = "当前有多少个 `TABLE` 引用我"，**任何 TABLE 存在 ⇒ SHARE 必定存在**；② `next/**prev` 是 **TDC 的 LRU 链**（`oldest_unused_share` 头=最久未用）；③ `cache_element[]` 是 SHARE→TABLE 的**必经之路**（淘汰 SHARE 要顺着它收回所有 TABLE）。

#### 1.6–1.12 成员全景（续）：补齐未覆盖的 45 个成员

> 源码坐标增补：`[T19] sql/dd_table_share.cc`（`fill_share_from_dd` 系列回填）· `[T24] sql/sql_check_constraint.h`（CHECK 约束两类对象）· `[T25] sql/key.h`（`KEY::rec_per_key`）· `[T30] sql/sql_tmp_table.cc`（临时表两阶段计数）· `[T32] sql/handler.cc` · `[T33] sql/histograms/histogram.h`

⚠️ 注意：`TABLE_SHARE` 在 8.0.39 中是 **`struct`**（`sql/table.h`），不是 `class`。

#### 1.6 ★ 统计与直方图组：`stats_sample_pages` / `m_histograms` / `base_rec_per_key` / `base_rec_per_key_float`

这一组管"表级统计参数的持久值 + 列级直方图的共享缓存 + 物化派生表生成键的 rec_per_key 基线数组"，是本文「表统计信息」一节的 SHARE 侧底座：凡是"从 DD 读出、跨 TABLE 实例共享、不需要每个 TABLE 各存一份"的统计，都落在这里。

```cpp
// [T1] sql/table.h —— TABLE_SHARE 头部：直方图容器与访问入口
struct TABLE_SHARE {
  TABLE_SHARE() = default;

  /**
    Create a new TABLE_SHARE with the given version number.
    @param version the version of the TABLE_SHARE
    @param secondary set to true if the TABLE_SHARE represents a table
                     in a secondary storage engine
  */
  TABLE_SHARE(unsigned long version, bool secondary)
      : m_version(version), m_secondary_engine(secondary) {}

  /*
    A map of [uint, Histogram] values, where the key is the field index. The
    map is populated with any histogram statistics when it is loaded/created.
  */
  malloc_unordered_map<uint, const histograms::Histogram *> *m_histograms{
      nullptr};

  /**
    Find the histogram for the given field index.

    @param field_index the index of the field we want to find a histogram for

    @retval nullptr if no histogram is found
    @retval a pointer to a histogram if one is found
  */
  const histograms::Histogram *find_histogram(uint field_index) const;
```

```cpp
// [T1] sql/table.h —— STATS_SAMPLE_PAGES / STATS_AUTO_RECALC 的表级默认值
  // Can only be 1,2,4,8 or 16, but use uint32_t since that how it is
  // represented in InnoDB
  std::uint32_t key_block_size{0}; /* create key_block_size, if used */
  uint stats_sample_pages{0};      /* number of pages to sample during
                                   stats estimation, if used, otherwise 0. */
  enum_stats_auto_recalc
      stats_auto_recalc{}; /* Automatic recalc of stats.
                              Zero-initialized to HA_STATS_AUTO_RECALC_DEFAULT
                            */
```

```cpp
// [T1] sql/table.h —— 物化派生表"生成键"的 rec_per_key 基线数组（base = baseline）
  /**
    Records per key array, used for materialized derived tables.
    This is a contiguous array, with size given by max_tmp_key_parts.
    The array is shared with all TABLE objects referring to this table share.
  */
  ulong *base_rec_per_key{nullptr};
  /**
    Records per key array, float rep., used for materialized derived tables.
    This is a contiguous array, with size given by max_tmp_key_parts.
    The array is shared with all TABLE objects referring to this table share.
  */
  rec_per_key_t *base_rec_per_key_float{nullptr};
```

`stats_sample_pages` / `stats_auto_recalc` 的回填点与取值语义：

```cpp
// [T19] sql/dd_table_share.cc —— fill_share_from_dd()：从 DD 的 options 属性读出
  // Stats_sample_pages
  if (table_options.exists("stats_sample_pages"))
    table_options.get("stats_sample_pages", &share->stats_sample_pages);

  // Stats_auto_recalc
  if (table_options.exists("stats_auto_recalc")) {
    table_options.get("stats_auto_recalc", &option_value);
    share->stats_auto_recalc = (enum_stats_auto_recalc)option_value;
  }
```

```cpp
// [T32] sql/handler.cc —— HA_CREATE_INFO 未显式指定时回落到 SHARE 的持久值
  if (!(used_fields & HA_CREATE_USED_STATS_SAMPLE_PAGES))
    stats_sample_pages = share->stats_sample_pages;
```

直方图的加载路径（`read_histograms()` 由 `get_table_share()` 在 SHARE 首次构建时调用）：

```cpp
// [T2] sql/sql_base.cc
static bool read_histograms(THD *thd, TABLE_SHARE *share,
                            const dd::Schema *schema,
                            const dd::Abstract_table *table_def) {
  dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
  MDL_request_list mdl_requests;
  for (const auto column : table_def->columns()) {
    if (column->is_se_hidden()) continue;

    MDL_key mdl_key;
    dd::Column_statistics::create_mdl_key(schema->name(), table_def->name(),
                                          column->name(), &mdl_key);

    MDL_request *request = new (thd->mem_root) MDL_request;
    MDL_REQUEST_INIT_BY_KEY(request, &mdl_key, MDL_SHARED_READ, MDL_STATEMENT);
    mdl_requests.push_front(request);
  }

  if (thd->mdl_context.acquire_locks(&mdl_requests,
                                     thd->variables.lock_wait_timeout))
    return true; /* purecov: deadcode */

  for (const auto column : table_def->columns()) {
    if (column->is_se_hidden()) continue;

    const histograms::Histogram *histogram = nullptr;
    if (histograms::find_histogram(thd, schema->name().c_str(),
                                   table_def->name().c_str(),
                                   column->name().c_str(), &histogram)) {
      // Any error is reported by the dictionary subsystem.
      return true; /* purecov: deadcode */
    }

    if (histogram != nullptr) {
      /*
        Make a clone of the histogram so it survives together with the
        TABLE_SHARE in case the original histogram is thrown out of the
        dictionary cache.
      */
      const histograms::Histogram *histogram_copy =
          histogram->clone(&share->mem_root);
      share->m_histograms->emplace(column->ordinal_position() - 1,
                                   histogram_copy);
    }
  }

  return false;
}
```

```cpp
// [T5] sql/table.cc —— 访问入口与析构（直方图随 SHARE 一起销毁）
const histograms::Histogram *TABLE_SHARE::find_histogram(
    uint field_index) const {
  if (m_histograms == nullptr) return nullptr;

  const auto found = m_histograms->find(field_index);
  if (found == m_histograms->end()) return nullptr;

  return found->second;
}

// TABLE_SHARE::destroy() 片段：
  /* The mutex is initialized only for shares that are part of the TDC */
  if (tmp_table == NO_TMP_TABLE) mysql_mutex_destroy(&LOCK_ha_data);

  delete m_histograms;
  m_histograms = nullptr;
```

**★ `base_rec_per_key` 与 `KEY::rec_per_key` 的关系（本节最需要讲清的一点）**

```cpp
// [T25] sql/key.h —— KEY 侧：两个"指针"，不是两块独立内存
  /**
    Array of AVG(number of records with the same field value) for 1st ... Nth
    key part. 0 means 'not known'. For internally created temporary tables,
    this member can be nullptr.
  */
  ulong *rec_per_key{nullptr};
  ...
  rec_per_key_t *rec_per_key_float{nullptr};

  bool has_records_per_key(uint key_part_no) const {
    return ((rec_per_key_float &&
             rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN) ||
            (rec_per_key && rec_per_key[key_part_no] != 0));
  }

  rec_per_key_t records_per_key(uint key_part_no) const {
    /*
      If the storage engine has provided rec per key estimates as float
      then use this. If not, use the integer version.
    */
    if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)
      return rec_per_key_float[key_part_no];

    return (rec_per_key[key_part_no] != 0)
               ? static_cast<rec_per_key_t>(rec_per_key[key_part_no])
               : REC_PER_KEY_UNKNOWN;
  }

  void set_rec_per_key_array(ulong *rec_per_key_arg,
                             rec_per_key_t *rec_per_key_float_arg) {
    rec_per_key = rec_per_key_arg;
    rec_per_key_float = rec_per_key_float_arg;
  }
```

两条不同的内存路线，务必分清：

```cpp
// [T19] sql/dd_table_share.cc —— 路线 A：基表。数组直接分配在 share->mem_root 上，
// 再让每个 KEY 的指针指向其中的一段（全表一份，所有 TABLE 实例共享）
    // Alloc rec_per_key buffer
    if (!(rec_per_key =
              (ulong *)share->mem_root.Alloc(total_key_parts * sizeof(ulong))))
      return true; /* purecov: inspected */

    // Alloc rec_per_key_float buffer
    if (!(rec_per_key_float = (rec_per_key_t *)share->mem_root.Alloc(
              total_key_parts * sizeof(rec_per_key_t))))
      return true; /* purecov: inspected */
  ...
    do {
      // Assign the key_part_info buffer
      KEY *keyinfo = &share->key_info[key_nr];
      keyinfo->key_part = key_part;
      keyinfo->set_rec_per_key_array(rec_per_key, rec_per_key_float);
      keyinfo->set_in_memory_estimate(IN_MEMORY_ESTIMATE_UNKNOWN);
      ...
      // Initialize the rec per key arrays
      for (uint kp = 0; kp < keyinfo->actual_key_parts; ++kp) {
        keyinfo->rec_per_key[kp] = 0;
        keyinfo->set_records_per_key(kp, REC_PER_KEY_UNKNOWN);
      }
```

```cpp
// [T5] sql/table.cc —— TABLE::key_info 是"结构体深拷贝、指针浅共享"：
// KEY 数组 + KEY_PART_INFO 数组复制一份到 TABLE 的 MEM_ROOT，
// 但 rec_per_key / rec_per_key_float 指针仍指向 SHARE 的那两块数组
bool create_key_part_field_with_prefix_length(TABLE *table, MEM_ROOT *root) {
  TABLE_SHARE *share = table->s;
  KEY *key_info = nullptr;
  KEY_PART_INFO *key_part = nullptr;
  uint n_length;

  n_length =
      share->keys * sizeof(KEY) + share->key_parts * sizeof(KEY_PART_INFO);

  // Allocate new memory for table.key_info
  if (!(key_info = static_cast<KEY *>(root->Alloc(n_length)))) return true;

  table->key_info = key_info;
  key_part = (reinterpret_cast<KEY_PART_INFO *>(key_info + share->keys));

  // Copy over the key_info from share to table.
  memcpy(key_info, share->key_info, sizeof(*key_info) * share->keys);
  memcpy(key_part, share->key_info[0].key_part,
         (sizeof(*key_part) * share->key_parts));
```

```cpp
// [T5] sql/table.cc —— 路线 B：物化派生表的"生成键"。SHARE 显式持有基线数组，
// TABLE 只负责把自己的 KEY 指针挂到基线的某个偏移上
  /*
    Allocate only key parts; key names and rec_per_key are shared
    with TABLE_SHARE object.
  */
  base_key_parts = s->mem_root.ArrayAlloc<KEY_PART_INFO>(new_key_part_count);
  ...
    cur_key->key_part = key_part_info;
    cur_key->set_rec_per_key_array(s->base_rec_per_key + key_part_offs,
                                   s->base_rec_per_key_float + key_part_offs);
    cur_key->table = this;

    /* Initialize rec_per_key and rec_per_key_float */
    for (uint kp = 0; kp < key_part_count; ++kp) {
      cur_key->rec_per_key[kp] = 0;
      cur_key->set_records_per_key(kp, REC_PER_KEY_UNKNOWN);
    }
  ...
  if (!invisible) {
    KEY &sk = s->key_info[keyno];
    sk = *cur_key;
    sk.table = nullptr;  // catch any illegal access
    sk.set_rec_per_key_array(nullptr, nullptr);   // SHARE 模板副本不带统计
  }
```

```cpp
// [T5] sql/table.cc —— 扩容：旧值 memcpy 搬迁，新增段复位为 UNKNOWN
    ulong *old_rec_per_key = s->base_rec_per_key;
    rec_per_key_t *old_rec_per_key_float = s->base_rec_per_key_float;

    s->base_rec_per_key = static_cast<ulong *>(
        s->mem_root.Alloc(sizeof(ulong) * new_key_part_count));
    if (s->base_rec_per_key == nullptr) return true;
    s->base_rec_per_key_float = static_cast<rec_per_key_t *>(
        s->mem_root.Alloc(sizeof(rec_per_key_t) * new_key_part_count));
    if (s->base_rec_per_key_float == nullptr) return true;

    for (uint i = 0; i < new_key_part_count; i++) {
      s->base_rec_per_key[i] = 0;
      s->base_rec_per_key_float[i] = REC_PER_KEY_UNKNOWN;
    }

    // Copy the existing data to the new arrays:
    if (old_rec_per_key != nullptr)
      memcpy(s->base_rec_per_key, old_rec_per_key,
             sizeof(ulong) * old_key_part_count);
    if (old_rec_per_key_float != nullptr)
      memcpy(s->base_rec_per_key_float, old_rec_per_key_float,
             sizeof(rec_per_key_t) * old_key_part_count);
```

> ★ **小结（`base_rec_per_key` 为什么存在）**：基表的 `rec_per_key` 数组物理上就分配在 `share->mem_root`，`KEY::rec_per_key` 只是指向它的一段指针，`open_table_from_share()` 复制的是 `KEY` 结构体本身而不是统计量——所以"统计信息天然共享"对基表是**隐式**的。而物化派生表的键是**优化器在执行期才用 `TABLE::add_tmp_key()` 造出来的**（DD 里根本没有这些键），同一个 SHARE 下可能挂着多个 TABLE 实例，必须有一个"所有人都认的同一份基线"，于是 SHARE 显式持有 `base_rec_per_key`。
> ★ 注意是**"挂指针"不是"拷贝值"**——`add_tmp_key()` 把 `s->base_rec_per_key + key_part_offs` 直接赋给 `cur_key->rec_per_key`；`alloc_tmp_keys()` 扩容时才真正 memcpy 搬迁。SHARE 自己的 `s->key_info[]` 模板副本则被刻意置成 `set_rec_per_key_array(nullptr, nullptr)` + `table = nullptr`，以阻止任何人在 SHARE 层读统计值。

#### 1.7 ★ 外键与 CHECK 约束组：`foreign_keys` / `foreign_key` / `foreign_key_parent` / `foreign_key_parents` / `check_constraint_share_list`

这一组管"prelocking（预锁）算法所需的父/子外键双向链"与"CHECK 约束的**定义模板**"——它们只需要"名字 + 规则 + 列名"这类轻量信息，够 prelocking 打开父表/子表即可，因此缓存在 SHARE 上避免每次开表都去读 DD。

```cpp
// [T1] sql/table.h —— 两个元素类型（SHARE 只缓存"够用"的最小集）
typedef struct Table_share_foreign_key_info {
  LEX_CSTRING referenced_table_db;
  LEX_CSTRING referenced_table_name;
  /**
    Name of unique key matching FK in parent table, "" if there is no
    unique key.
  */
  LEX_CSTRING unique_constraint_name;
  dd::Foreign_key::enum_rule update_rule, delete_rule;
  uint columns;
  /**
    Arrays with names of referencing columns of the FK.
  */
  LEX_CSTRING *column_name;
} TABLE_SHARE_FOREIGN_KEY_INFO;

typedef struct Table_share_foreign_key_parent_info {
  LEX_CSTRING referencing_table_db;
  LEX_CSTRING referencing_table_name;
  dd::Foreign_key::enum_rule update_rule, delete_rule;
} TABLE_SHARE_FOREIGN_KEY_PARENT_INFO;
```

```cpp
// [T1] sql/table.h —— 四个成员（注意单数/复数命名不对称：数在前、数组在后）
  /**
    Arrays with descriptions of foreign keys in which this table participates
    as child or parent. We only cache in them information from dd::Table object
    which is sufficient for use by prelocking algorithm/to check if table is
    referenced by a foreign key.
  */
  uint foreign_keys{0};
  TABLE_SHARE_FOREIGN_KEY_INFO *foreign_key{nullptr};
  uint foreign_key_parents{0};
  TABLE_SHARE_FOREIGN_KEY_PARENT_INFO *foreign_key_parent{nullptr};

  // List of check constraint share instances.
  Sql_check_constraint_share_list *check_constraint_share_list{nullptr};
```

```cpp
// [T1] sql/table.h —— 对外谓词
  /** Returns whether this table is referenced by a foreign key. */
  bool is_referenced_by_foreign_key() const { return foreign_key_parents != 0; }
```

```cpp
// [T19] sql/dd_table_share.cc —— fill_foreign_keys_from_dd()：双向链的填充（节选）
static bool fill_foreign_keys_from_dd(TABLE_SHARE *share,
                                      const dd::Table *tab_obj) {
  assert(share->foreign_keys == 0 && share->foreign_key_parents == 0);

  share->foreign_keys = tab_obj->foreign_keys().size();
  share->foreign_key_parents = tab_obj->foreign_key_parents().size();

  if (share->foreign_keys) {
    if (!(share->foreign_key =
              (TABLE_SHARE_FOREIGN_KEY_INFO *)share->mem_root.Alloc(
                  share->foreign_keys * sizeof(TABLE_SHARE_FOREIGN_KEY_INFO))))
      return true;
    ...
    for (const dd::Foreign_key *fk : tab_obj->foreign_keys()) {
      ...  // referenced_table_db / referenced_table_name / unique_constraint_name
      share->foreign_key[i].update_rule = fk->update_rule();
      share->foreign_key[i].delete_rule = fk->delete_rule();

      share->foreign_key[i].columns = fk->elements().size();
      ...  // column_name[] 逐个 lex_string_strmake 进 share->mem_root
    }
  }

  if (share->foreign_key_parents) {
    if (!(share->foreign_key_parent =
              (TABLE_SHARE_FOREIGN_KEY_PARENT_INFO *)share->mem_root.Alloc(
                  share->foreign_key_parents *
                  sizeof(TABLE_SHARE_FOREIGN_KEY_PARENT_INFO))))
      return true;
    ...
    for (const dd::Foreign_key_parent *fk_p : tab_obj->foreign_key_parents()) {
      ...  // referencing_table_db / referencing_table_name
      share->foreign_key_parent[i].update_rule = fk_p->update_rule();
      share->foreign_key_parent[i].delete_rule = fk_p->delete_rule();
    }
  }
  return false;
}
```

```cpp
// [T19] sql/dd_table_share.cc —— open_table_def() 中的调用顺序（约束/外键同一批填充）
  // Fill the TABLE_SHARE with details.
  bool error = (fill_share_from_dd(thd, share, &table_def) ||
                fill_columns_from_dd(thd, share, &table_def) ||
                fill_indexes_from_dd(thd, share, &table_def) ||
                fill_partitioning_from_dd(thd, share, &table_def) ||
                fill_foreign_keys_from_dd(share, &table_def) ||
                fill_check_constraints_from_dd(share, &table_def));
```

```cpp
// [T19] sql/dd_table_share.cc —— CHECK 约束只存"字符串形式的定义"
static bool fill_check_constraints_from_dd(TABLE_SHARE *share,
                                           const dd::Table *tab_obj) {
  assert(share->check_constraint_share_list == nullptr);

  if (tab_obj->check_constraints().size() > 0) {
    share->check_constraint_share_list = new (&share->mem_root)
        Sql_check_constraint_share_list(&share->mem_root);
    ...
    for (auto &cc : tab_obj->check_constraints()) {
      // Check constraint name.
      LEX_CSTRING name; ...
      // Check constraint expression (clause).
      LEX_CSTRING check_clause; ...
      // Check constraint state.
      bool is_cc_enforced =
          (cc->constraint_state() == dd::Check_constraint::CS_ENFORCED);

      share->check_constraint_share_list->push_back(
          Sql_check_constraint_share(name, check_clause, is_cc_enforced));
    }
  }
  return false;
}
```

**外键双向链示意**（`parent` 与 `child` 各自持有一条链，方向相反）：

```
           库 db1.parent（父表，被引用方）
   +-------------------------------------------+
   | TABLE_SHARE(parent)                       |
   |   foreign_keys        = 0      <- 本表没定义 FK |
   |   foreign_key         = nullptr           |
   |   foreign_key_parents = 1                 |
   |   foreign_key_parent[0] = {               |
   |       referencing_table_db   = "db1",      |
   |       referencing_table_name = "child",    |   // 谁引用了我
   |       update_rule / delete_rule }          |
   +-------------------------------------------+
                      ^                    |
        FK(child.c1 -> parent.pid)          |  同一条 FK 在 DD 里既写进
                      |                    v  parent.foreign_key_parents
           库 db1.child（子表，定义方）          也写进 child.foreign_keys
   +-------------------------------------------+
   | TABLE_SHARE(child)                        |
   |   foreign_keys  = 1                       |
   |   foreign_key[0] = {                      |
   |       referenced_table_db   = "db1",       |
   |       referenced_table_name = "parent",    |   // 我引用了谁
   |       unique_constraint_name= "PRIMARY"    |   // 父表侧匹配的唯一键
   |       update_rule / delete_rule,          |
   |       columns = 1, column_name = {"c1"} } |
   |   foreign_key_parents = 0                 |
   +-------------------------------------------+

   用途：prelocking 在"打开表之前"就要知道还有哪些表必须一并打开并加锁，
        因此只缓存 schema/table/列名/规则这些"够用"的信息，不建 Item 树。
```

#### 1.8 二级引擎与表选项组

这一组管"从 DD 的 `options`/属性里读出的建表选项与二级引擎信息"——全部是 SHARE 侧的**只读模板**。

```cpp
// [T1] sql/table.h
  uchar *default_values{nullptr};      /* row with default values */
  LEX_STRING comment{nullptr, 0};      /* Comment about table */
  LEX_STRING compress{nullptr, 0};     /* Compression algorithm */
  LEX_STRING encrypt_type{nullptr, 0}; /* encryption algorithm */

  /** Secondary storage engine. */
  LEX_CSTRING secondary_engine{nullptr, 0};
  /** Secondary engine load status */
  bool secondary_load{false};

  LEX_STRING connect_string{nullptr, 0};  /* FEDERATED 的连接串 */

  LEX_CSTRING engine_attribute = EMPTY_CSTR;
  LEX_CSTRING secondary_engine_attribute = EMPTY_CSTR;
  ...
  ha_rows min_rows{0}, max_rows{0}; /* create information */
  ulong avg_row_length{0};          /* create information */
  ulong mysql_version{0};           /* 0 if .frm is created before 5.0 */
  ...
  /**
    Storage media to use for this table (unless another storage
    media has been specified on an individual column - in versions
    where that is supported)
  */
  ha_storage_media default_storage_media{HA_SM_DEFAULT};
```

```cpp
// [T1] sql/table.h —— 二级引擎的"身份"三件套（m_secondary_engine 是 protected）
  /// Does this TABLE_SHARE represent a table in a primary storage engine?
  bool is_primary_engine() const { return !m_secondary_engine; }

  /// Does this TABLE_SHARE represent a table in a secondary storage engine?
  bool is_secondary_engine() const { return m_secondary_engine; }
  /// Does this TABLE_SHARE represent a primary table that has a shadow
  /// copy in a secondary storage engine?
  bool has_secondary_engine() const {
    return is_primary_engine() && secondary_engine.str != nullptr;
  }
  ...
 protected:  // To allow access from unit tests.
  /// Does this TABLE_SHARE represent a table in a secondary storage engine?
  bool m_secondary_engine{false};
```

```cpp
// [T19] sql/dd_table_share.cc —— 各选项的回填点（节选）
  // Average row length
  if (table_options.exists("avg_row_length")) {
    table_options.get("avg_row_length", &option_value);
    share->avg_row_length = static_cast<ulong>(option_value);
  }
  ...
  // mysql version
  share->mysql_version = tab_obj->mysql_version_id();
  ...
  // key block size
  table_options.get("key_block_size", &share->key_block_size);
  ...
  // Storage media flags
  if (table_options.exists("storage")) {
    uint32 storage_option_value = 0;
    table_options.get("storage", &storage_option_value);
    share->default_storage_media =
        static_cast<ha_storage_media>(storage_option_value);
  } else
    share->default_storage_media = HA_SM_DEFAULT;
  ...
  // Read comment
  dd::String_type comment = tab_obj->comment();
  if (comment.length()) {
    share->comment.str =
        strmake_root(&share->mem_root, comment.c_str(), comment.length() + 1);
    share->comment.length = comment.length();
  }

  // Copy SE attributes into share's memroot
  share->engine_attribute = LexStringDupRootUnlessEmpty(
      &share->mem_root, tab_obj->engine_attribute());
  share->secondary_engine_attribute = LexStringDupRootUnlessEmpty(
      &share->mem_root, tab_obj->secondary_engine_attribute());
  ...
  // Read Connection strings / Compress / Encrypt type
  if (table_options.exists("connection_string"))
    table_options.get("connection_string", &share->connect_string,
                      &share->mem_root);
  if (table_options.exists("compress"))
    table_options.get("compress", &share->compress, &share->mem_root);
  if (table_options.exists("encrypt_type"))
    table_options.get("encrypt_type", &share->encrypt_type, &share->mem_root);

  // Read secondary load option.
  if (table_options.exists("secondary_load"))
    table_options.get("secondary_load", &share->secondary_load);
```

★ `secondary_engine`（字符串）与 `m_secondary_engine`（bool）要分清：前者是"主引擎表在哪个二级引擎里有一份影子拷贝"，配合 `has_secondary_engine()` 使用；后者是"**这个 SHARE 本身就是**为二级引擎表建的"——同一张表在 TDC 里可以同时存在**两个 SHARE**（主/次），靠 `m_secondary_engine` 区分，这也是 `alloc_table_share(..., bool open_secondary)` 最后一个参数的落点。

#### 1.9 临时表 handler 组

这一组管"内部临时表两阶段生命周期的引用计数"与"物化派生表生成键的槽位分配"。

```cpp
// [T1] sql/table.h
  MEM_ROOT mem_root;
  /**
    Used to allocate new handler for internal temporary table when the
    size limitation of the primary storage engine is exceeded.
  */
  MEM_ROOT *alloc_for_tmp_file_handler{nullptr};
  ...
  /**
    Only for internal temporary tables.
    Count of TABLEs (having this TABLE_SHARE) which have a "handler"
    (table->file!=nullptr) which is open (ha_open() has been called).
  */
  uint tmp_handler_count{0};

  /**
    Only for internal temporary tables.
    Count of TABLEs (having this TABLE_SHARE) which have opened this table.
  */
  uint tmp_open_count{0};
  ...
  /**
    For materialized derived tables; @see add_derived_key().
    'first' means: having the lowest position in key_info.
  */
  uint first_unused_tmp_key{0};
  /**
    For materialized derived tables: allocated size of key_info array.
  */
  uint max_tmp_keys{0};
  /**
    For materialized derived tables: allocated size of base_key_parts array of
    all TABLE objects. Used for generated keys.
  */
  uint max_tmp_key_parts{0};
  ...
  /// For materialized derived tables; @see add_derived_key().
  Query_block *owner_of_possible_tmp_keys{nullptr};
```

两阶段计数器的契约写在文件头注释里：

```cpp
// [T30] sql/sql_tmp_table.cc（文件头注释节选）
  Since a temporary table may be in any of the two stages, we use two
  counter members in the TABLE_SHARE to count the number of TABLEs in each
  of the stages: tmp_handler_count and tmp_open_count.
  tmp_handler_count is incremented in setup_tmp_table_handler().
  tmp_open_count is incremented in open_tmp_table().
  ...
  If the table is created, TABLE_SHARE::tmp_open_count is decremented.
  If there are no remaining active TABLE objects, delete the table contents
  by calling TABLE::file->ha_drop_table(), otherwise close it by calling
  TABLE::file->ha_close().
  ...
  If there are no remaining active tables and the storage engine is still
  locked, unlock the plugin and disassociate it from the TABLE_SHARE object,
  and decrement TABLE_SHARE::tmp_handler_count.
```

```cpp
// [T30] sql/sql_tmp_table.cc —— 计数器的实际增减点
  share->alloc_for_tmp_file_handler = thd->mem_root;

  table->file = get_new_handler(share, false, share->alloc_for_tmp_file_handler,
                                share->db_type());
  if (table->file == nullptr) return true;

  share->tmp_handler_count++;          // 阶段一：拿到 handler
  ...
  table->s->tmp_open_count++;          // 阶段二：真正实例化
  table->set_created();
  ...
  if (table->is_created()) {
    if (--share->tmp_open_count > 0) {
      table->file->ha_close();
    } else  // no more open 'handler' objects
      table->file->ha_drop_table(table->s->table_name.str);
  }
  ...
  if (--share->tmp_handler_count == 0 && share->db_plugin != nullptr) {
    plugin_unlock(nullptr, share->db_plugin);
    share->db_plugin = nullptr;
  }
```

★ `alloc_for_tmp_file_handler` 存在的唯一理由：临时表默认用 TempTable，一旦超过 `temptable_max_ram` 就要**临时换成 InnoDB 的 handler**，这时新 handler 必须从一个比 `thd->mem_root` 活得更久的 MEM_ROOT 上分配（否则语句结束就悬垂），所以把 `thd->mem_root` 记在 SHARE 上：

```cpp
// [T30] sql/sql_tmp_table.cc
  if (error == HA_ERR_RECORD_FILE_FULL &&
      table->s->db_type() == temptable_hton) {
    table->file = get_new_handler(
        table->s, false, share->alloc_for_tmp_file_handler, innodb_hton);
    error = table->file->create(share->table_name.str, table, &create_info,
                                nullptr);
  }
```

#### 1.10 键元数据组

这一组管"键名/键长/ENUM·SET 取值域"这类纯元数据，全部随 SHARE 常驻、被所有 TABLE 共享。

```cpp
// [T1] sql/table.h
/**
  Definition of name for generated keys, owned by TABLE_SHARE
*/
struct Key_name {
  char name[NAME_CHAR_LEN];
};
  ...
  TYPELIB keynames;            /* Pointers to keynames */
  TYPELIB *intervals{nullptr}; /* pointer to interval info */
  mysql_mutex_t LOCK_ha_data;  /* To protect access to ha_data */
  ...
  uint max_key_length{0};    /* Length of the longest key */
  uint max_unique_length{0}; /* Length of the longest unique key */
  uint total_key_length{0};
  ...
  /**
    Array of names for generated keys, used for materialized derived tables.
    Shared among all TABLE objects referring to this table share.
  */
  Key_name *key_names{nullptr};
```

```cpp
// [T19] sql/dd_table_share.cc —— 原生键的 keynames（TYPELIB）与两个长度统计
    if (!(share->keynames.type_names = (const char **)share->mem_root.Alloc(
              (share->keys + 1) * sizeof(char *))))
      return true; /* purecov: inspected */
    memset(share->keynames.type_names, 0, ((share->keys + 1) * sizeof(char *)));

    share->keynames.type_names[share->keys] = nullptr;
    share->keynames.count = share->keys;
  ...
      share->total_key_length += keyinfo->key_length;
      /*
        MERGE tables do not have unique indexes. But every key could be
        unique ...（下略）
      */
      if ((keyinfo->flags & HA_NOSAME) ||
          (ha_option & HA_ANY_INDEX_MAY_BE_UNIQUE))
        share->max_unique_length =
            std::max(share->max_unique_length, keyinfo->key_length);
```

`max_unique_length` 是"去重（Unique）时行缓冲的最大键长"，被 `sql/log_event.cc` 与 `sql/sql_insert.cc` 按它申请临时键缓冲（`my_alloca(table->s->max_unique_length)` / `my_safe_alloca(table->s->max_unique_length, MAX_KEY_LENGTH)`）。
`intervals` 是 ENUM/SET 列的取值域数组（`TYPELIB` 数组，`share->intervals + interval_nr - 1` 定位到某列的域），在 `make_field()` 建 `Field_enum`/`Field_set` 时传入。

#### 1.11 分区组

这一组管"分区元数据的**模板**与重新解析入口"——SHARE 存一份 `partition_info`（省内存），每个 TABLE 再通过解析 `partition_info_str` 生成自己的 `TABLE::part_info`。

```cpp
// [T1] sql/table.h
  /**
    Partition meta data. Allocated from TABLE_SHARE::mem_root,
    created when reading from the dd tables,
    used as template for each TABLE instance.
    The reason for having it on the TABLE_SHARE is to be able to reuse the
    partition_elements containing partition names, values etc. instead of
    allocating them for each TABLE instance.
    TODO: Currently it is filled in and then only used for generating
    the partition_info_str. The plan is to clone/copy/reference each
    TABLE::part_info instance from it.
    What is missing before it can be completed:
    1) The partition expression, currently created only during parsing which
       also needs the current TABLE instance as context for name resolution etc.
    2) The partition values, currently the DD stores them as text so it needs
       to be converted to field images ...
  */
  partition_info *m_part_info{nullptr};
  // TODO: Remove these four variables:
  /**
    Filled in when reading from frm.
    This can simply be removed when removing the .frm support,
    since it is already stored in the new DD.
  */
  bool auto_partitioned{false};
  /**
    Storing the full partitioning clause (PARTITION BY ...) which is used
    when creating new partition_info object for each new TABLE object by
    parsing this string.
    These two will be needed until the missing parts above is fixed.
  */
  char *partition_info_str{nullptr};
  uint partition_info_str_len{0};
```

```cpp
// [T19] sql/dd_table_share.cc —— 字符串由 SHARE 生成；[T5] 再由每个 TABLE 解析回去
  share->partition_info_str = strmake_root(&share->mem_root, buf, buf_len);
  if (!share->partition_info_str) return true;

  share->partition_info_str_len = buf_len;
  share->m_part_info = part_info;
```

```cpp
// [T5] sql/table.cc —— open_table_from_share()：per-TABLE 的 partition_info 从这里重建
  tmp = mysql_unpack_partition(
      thd, share->partition_info_str, share->partition_info_str_len, outparam,
      is_create_table, engine_type, &work_part_info_used);
  if (tmp) { ... }
  outparam->part_info->is_auto_partitioned = share->auto_partitioned;
```

#### 1.12 布局与杂项组

这一组管"行布局偏移 / 生成默认值列数 / binlog 判定缓存 / ha_share 互斥 / P_S 探针 / 系统表结构校验缓存"，是四个彼此无关但都必须随 SHARE 常驻的小字段。

```cpp
// [T1] sql/table.h
  uint rowid_field_offset{0}; /* Field_nr +1 to rowid field */
  ...
  /// Number of generated fields
  uint vfields{0};
  /// Number of fields having the default value generated
  uint gen_def_field_count{0};
  ...
  /*
    Cache for row-based replication table share checks that does not
    need to be repeated. Possible values are: -1 when cache value is
    not calculated yet, 0 when table *shall not* be replicated, 1 when
    table *may* be replicated.
  */
  int cached_row_logging_check{0};
  ...
  /**
    Cache the checked structure of this table.

    The pointer data is used to describe the structure that
    a instance of the table must have. Each element of the
    array specifies a field that must exist on the table.

    The pointer is cached in order to perform the check only
    once -- when the table is loaded from the disk.
  */
  const TABLE_FIELD_DEF *table_field_def_cache{nullptr};

  /** Main handler's share */
  Handler_share *ha_share{nullptr};

  /** Instrumentation for this table share. */
  PSI_table_share *m_psi{nullptr};
```

`rowid_field_offset` 是"单列整数主键"的快捷通道（`Field_nr + 1`，0 表示没有），解析 `_rowid` 伪列时用：`field = table->field[table->s->rowid_field_offset - 1];`

`LOCK_ha_data` 保护的是 `TABLE_SHARE::ha_share`（引擎侧共享结构），**只对非临时表生效**：

```cpp
// [T32] sql/handler.cc
Handler_share *handler::get_ha_share_ptr() {
  assert(ha_share && table_share);
#ifndef NDEBUG
  if (table_share->tmp_table == NO_TMP_TABLE)
    mysql_mutex_assert_owner(&table_share->LOCK_ha_data);
#endif
  return *ha_share;
}

void handler::lock_shared_ha_data() {
  assert(table_share);
  if (table_share->tmp_table == NO_TMP_TABLE)
    mysql_mutex_lock(&table_share->LOCK_ha_data);
}
void handler::unlock_shared_ha_data() {
  assert(table_share);
  if (table_share->tmp_table == NO_TMP_TABLE)
    mysql_mutex_unlock(&table_share->LOCK_ha_data);
}
```

`m_psi` 与 `cached_row_logging_check` 的生命周期：

```cpp
// [T2] sql/sql_base.cc —— SHARE 建好即注册 P_S 探针
  share->m_psi = PSI_TABLE_CALL(get_table_share)(
      (share->tmp_table != NO_TMP_TABLE), share);
// [T5] sql/table.cc —— destroy() 里注销
  PSI_TABLE_CALL(release_table_share)(m_psi);
```

```cpp
// [T32] sql/handler.cc —— 三态缓存：-1 未算 / 0 不复制 / 1 可复制
static bool check_table_binlog_row_based(THD *thd, TABLE *table) {
  if (table->s->cached_row_logging_check == -1) {
    int const check(table->s->tmp_table == NO_TMP_TABLE &&
                    !table->no_replicate &&
                    binlog_filter->db_ok(table->s->db.str));
    table->s->cached_row_logging_check = check;
  }

  assert(table->s->cached_row_logging_check == 0 ||
         table->s->cached_row_logging_check == 1);

  return (thd->is_current_stmt_binlog_format_row() &&
          table->s->cached_row_logging_check &&
          (thd->variables.option_bits & OPTION_BIN_LOG) &&
          mysql_bin_log.is_open());
}
```

```cpp
// [T5] sql/table.cc —— 系统表结构校验只做一次：比对"期望的" TABLE_FIELD_DEF 指针
  /* Whether the table definition has already been validated. */
  if (table->s->table_field_def_cache == table_def) goto end;
  ...
  if (!error) table->s->table_field_def_cache = table_def;
```

```cpp
// [T1] sql/table.h —— schema 级只读（ALTER SCHEMA ... READ ONLY，8.0.22+）
    Schema's read only mode - ON (true) or OFF (false). This is filled in
    when the share is initialized with meta data from DD. If the schema is
    altered, the tables and share are removed. This can be done since
    ALTER SCHEMA acquires exclusive meta data locks on the tables in the
    schema. We set this only for non-temporary tables. Otherwise, the value
    of the member below is 'NOT_SET'.
  */
  enum class Schema_read_only { NOT_SET, RO_OFF, RO_ON };
  Schema_read_only schema_read_only{Schema_read_only::NOT_SET};
```

★ 三态而不是 bool 的原因写在注释里：`NOT_SET` 表示"这张表不需要关心 schema 只读"（**临时表**恒为 `NOT_SET`），`RO_OFF`/`RO_ON` 才是真实状态。回填点在开表时：

```cpp
// [T2] sql/sql_base.cc —— get_table_share() 初始化 SHARE 时随 DD 元数据填入
      share->schema_read_only = TABLE_SHARE::Schema_read_only::RO_ON;
      ...
      share->schema_read_only = TABLE_SHARE::Schema_read_only::RO_OFF;
```

```cpp
// sql/sql_db.cc —— 消费点：schema 只读时拒绝 DML/DDL
  if (thread_can_ignore_schema_read_only(thd)) return false;
  ...
    if (share->schema_read_only == TABLE_SHARE::Schema_read_only::RO_ON) {
```

⇒ 注意它**只对非临时表有值**：`ALTER SCHEMA` 会拿 schema 内所有表的排他 MDL，因此可以直接把相关的 SHARE 与 TABLE **从缓存里移除**而不必逐个更新标志——这正是注释里 "the tables and share are removed" 的含义。

#### 1.13 小结：SHARE 侧成员的归属判据

★ 上述 7 组 45 个成员可以用**一条判据**串起来：**"随表定义终身不变、且可跨 TABLE 实例共享 ⇒ 放 SHARE"**。分区的 `m_part_info`、外键双向链、CHECK 约束模板、键元数据、`keynames`/`intervals` 无一例外；反过来，需要 per-TABLE 状态的（如 CHECK 的 Item 树）就落到 TABLE 侧（见 2.8）。

与本文其它章节的衔接：

- **与「表统计信息」**：那一节讲 `handler::stats`（`records` / `data_file_length` 等，随 TABLE/引擎浮动）与优化器如何消费；本节补齐它的**上游**——`stats_sample_pages`/`stats_auto_recalc` 是引擎采样参数的持久值，`m_histograms` 是列级直方图的 SHARE 级缓存（`clone()` 进 `share->mem_root`），`base_rec_per_key*` 是物化派生表生成键的唯一权威副本。三者共同回答"统计信息在 SHARE 与 TABLE 之间如何分割"：**值在 TABLE/引擎，模板与共享基线在 SHARE**。
- **与「生命周期与缓存分层」**：SHARE 三条死法中的"TDC 淘汰"会连带销毁 `m_histograms`、psi 探针、`LOCK_ha_data`。
- **与「表类型」**：`tmp_handler_count` / `tmp_open_count` / `alloc_for_tmp_file_handler` / `first_unused_tmp_key` / `max_tmp_keys` / `max_tmp_key_parts` / `owner_of_possible_tmp_keys` **只对内部临时表与物化派生表有意义**，对基表恒为 0/nullptr——这正是 `tmp_table_type` 分野在 SHARE 上的投影。`m_secondary_engine` 则是"同一张表在 TDC 里存在两个 SHARE"这一特殊表类型的开关。

### 2. `TABLE` 成员全景

#### 2.1 四个"身份指针"

```cpp
// [T1]
  TABLE_SHARE *s{nullptr};          /* 定义我 */
  handler *file{nullptr};           /* 操作我 */
  THD *in_use{nullptr};             /* ★ 占用我——nullptr 即"空闲可复用"的判据 */
  Table_ref *pos_in_table_list{nullptr};  /* 回到语法树的反向指针 */
```

#### 2.2 ★ 列与行缓冲组

```cpp
// [T1]
  Field **field{nullptr};           /* ★ SHARE::field 的克隆 */
  uchar *record[2]{nullptr, nullptr};
  uchar *write_row_record{nullptr};
  uchar *insert_values{nullptr};    /* INSERT ... ON DUPLICATE KEY UPDATE */
  uchar *null_flags{nullptr};       /* == record[0]，NULL 位在行首 */
  uchar *null_flags_saved{nullptr}; /* 外连接补 NULL 前的备份 */
  ptrdiff_t default_values_offset() const {
    return (ptrdiff_t)(s->default_values - record[0]);   /* ★ 实例行 ↔ 默认行的偏移 */
  }
```

⇒ 克隆的**唯一目的**是让每个 `Field` 的 `ptr` 指向**本实例**的 `record[0]`（而不是 SHARE 的 `default_values`），偏移量就是 `record[0] - default_values`。三步细节见 `field.md`。

#### 2.3 ★ 位图组（本体 vs 指针）

```cpp
// [T1] —— def_* 是"本体"（8 份一次性从 TABLE 的 mem_root 切出），read/write_set 是"指针"
  MY_BITMAP def_read_set, def_write_set, tmp_set, pack_row_tmp_set;
  MY_BITMAP cond_set;                       /* 仅用于 condition_fanout_filter */
  MY_BITMAP def_fields_set_during_insert;
  MY_BITMAP read_set_internal;              /* 不含虚拟生成列的隐含基列 */
  MY_BITMAP *read_set{nullptr};
  MY_BITMAP *write_set{nullptr};
```

```cpp
// [T1] —— ★ 正因为是指针，才有三种"切换"
  void column_bitmaps_set(MY_BITMAP *read_set_arg, MY_BITMAP *write_set_arg);
  inline void use_all_columns() { column_bitmaps_set(&s->all_set, &s->all_set); }
  inline void default_column_bitmaps() {
    read_set = &def_read_set; write_set = &def_write_set;
  }
```

**谁设置这些位图**：

```cpp
// [T2]
void TABLE::mark_column_used(Field *field, enum enum_mark_columns mark) {
  switch (mark) {
    case MARK_COLUMNS_READ:
      bitmap_set_bit(read_set, field->field_index());
      bitmap_set_bit(&read_set_internal, field->field_index());
      covering_keys.intersect(part_of_key);        /* ★ 逐列收窄覆盖索引候选 */
      if (field->is_virtual_gcol()) mark_gcol_in_maps(field);
      break;
    case MARK_COLUMNS_WRITE:
      bitmap_set_bit(write_set, field->field_index());
      break;
    ...
  }
}
```

⇒ 解析/优化阶段每个被 `fix_field` 解析到的列打自己的 bit；`mark_columns_used_by_index()` 临时切到 `tmp_set`（只读某索引覆盖的列）。★ **`TABLE::init()` → `clear_column_bitmaps()` 清空默认位图并复位指针**——这正是 PS/存储过程重复执行时位图能"回到干净状态"的原因（TABLE 是长期复用对象，位图状态绝不能跨语句泄漏）。

#### 2.4 键组：★ `TABLE::key_info` 是深拷贝，不是共享指针

```cpp
// [T2]
bool create_key_part_field_with_prefix_length(TABLE *table, MEM_ROOT *root) {
  TABLE_SHARE *share = table->s;
  ...
  if (!(key_info = static_cast<KEY *>(root->Alloc(n_length)))) return true;
  table->key_info = key_info;
  memcpy(key_info, share->key_info, sizeof(*key_info) * share->keys);   /* ★ 深拷贝 */
  ...
      Field *field = key_part->field = table->field[key_part->fieldnr - 1];  /* ★ 重绑到实例列 */
      if (field->key_length() != key_part->length && !field->is_flag_set(BLOB_FLAG)) {
        field = key_part->field = field->new_field(root, table);             /* 前缀索引专用 Field */
        field->set_field_length(key_part->length);
      }
```

⇒ **SHARE 的 `key_info` 是模板，TABLE 的 `key_info` 是绑定了实例列的可执行副本**。另注意 `covering_keys` / `quick_keys` / `possible_quick_keys` / `merge_keys` 全是**优化期产物**（`covering_keys` 起点是 `s->keys_for_keyread`，被 `mark_column_used()` 逐列收窄），都在 `TABLE::reset()` 里清空。

#### 2.5 状态组

```cpp
// [T1]
  query_id_t query_id{0};            /* 本表当前属于哪条语句（LOCK TABLES 预锁用）*/
  uint8 m_status{0};                 /* record[0] 里装的是什么 */
  bool null_row{false}, const_table{false}, key_read{false}, no_keyread{false};
#define STATUS_NOT_STARTED 1
#define STATUS_NOT_FOUND   2
#define STATUS_UPDATED    16
#define STATUS_NULL_ROW   32
#define STATUS_DELETED    64
```

⇒ 实时统计**不在 TABLE 上**，在 `table->file->stats`（引擎侧）。

#### 2.6–2.12 成员全景（续）：补齐未覆盖的 36 个成员

> 源码坐标增补：`[T22] sql/opt_costmodel.h`（`Cost_model_table`）· `[T26] sql/record_buffer.h` · `[T27] sql/sql_sort.h`（`Sort_result`）· `[T28] sql/iterators/basic_row_iterators.{h,cc}` · `[T29] sql/filesort.cc` · `[T31] sql/lock.cc` · `[T34] sql/rpl_record.cc` · `[T23] sql/sql_trigger.h`
> ⚠️ `TABLE` 在 8.0.39 中同样是 **`struct`**。另需澄清一处常见误记：`Partial_update_info` 定义在 **`sql/table.cc`**（不是 `sql/json_diff.h`），`sql/table.h` 里只有前向声明 `struct Partial_update_info;`；`sql/json_diff.h` 只提供 `Json_diff` / `Json_diff_vector`。

#### 2.6 ★ 部分更新组（JSON partial update）：`m_partial_update_columns` / `m_partial_update_info` / `m_record_buffer`

这一组管"UPDATE 一个 JSON 列时，只把文档中被改动的那几段字节写回引擎、并只把 diff 写进 binlog"——8.0 面向 JSON 的核心写路径优化。

```cpp
// [T1] sql/table.h —— 两个私有成员：一个"资格位图"，一个"执行期状态"
 private:
  /**
    Bitmap that tells which columns are eligible for partial update in an
    update statement.

    The bitmap is lazily allocated in the TABLE's mem_root when
    #mark_column_for_partial_update() is called.
  */
  MY_BITMAP *m_partial_update_columns{nullptr};

  /**
    Object which contains execution time state used for partial update
    of JSON columns.

    It is allocated in the execution mem_root by #setup_partial_update() if
    there are columns that have been marked as eligible for partial update.
  */
  Partial_update_info *m_partial_update_info{nullptr};
```

```cpp
// [T5] sql/table.cc —— Partial_update_info：逐列的二元/逻辑 diff 收集器
struct Partial_update_info {
  Partial_update_info(const TABLE *table, const MY_BITMAP *columns,
                      bool logical_diffs)
      : m_binary_diff_vectors(current_thd->mem_root, table->s->fields, nullptr),
        m_logical_diff_vectors(current_thd->mem_root,
                               logical_diffs ? table->s->fields : 0, nullptr) {
    MEM_ROOT *const mem_root = current_thd->mem_root;
    const size_t bitmap_size = table->s->column_bitmap_size;

    auto buffer = static_cast<my_bitmap_map *>(mem_root->Alloc(bitmap_size));
    if (buffer != nullptr) {
      bitmap_init(&m_enabled_binary_diff_columns, buffer, table->s->fields);
      bitmap_copy(&m_enabled_binary_diff_columns, columns);
    }

    buffer = static_cast<my_bitmap_map *>(mem_root->Alloc(bitmap_size));
    if (buffer != nullptr) {
      bitmap_init(&m_enabled_logical_diff_columns, buffer, table->s->fields);
      if (logical_diffs)
        bitmap_copy(&m_enabled_logical_diff_columns, columns);
      else
        bitmap_clear_all(&m_enabled_logical_diff_columns);
    }

    for (uint i = bitmap_get_first_set(columns); i != MY_BIT_NONE;
         i = bitmap_get_next_set(columns, i)) {
      m_binary_diff_vectors[i] = new (mem_root) Binary_diff_vector(mem_root);

      if (logical_diffs) {
        Json_diff_vector::allocator_type alloc(mem_root);
        m_logical_diff_vectors[i] = new (mem_root) Json_diff_vector(alloc);
      }
    }
  }

  ~Partial_update_info() {
    for (auto v : m_logical_diff_vectors) destroy(v);
  }

  /**
    The columns for which partial update using binary diffs is enabled
    in the current row.
  */
  MY_BITMAP m_enabled_binary_diff_columns;

  /**
    The columns for which partial update using logical JSON diffs is
    enabled in the current row.
  */
  MY_BITMAP m_enabled_logical_diff_columns;

  /**
    The binary diffs that have been collected for the current row.

    The Binary_diff_vector objects live entirely in a MEM_ROOT, so
    there is no need to destroy them when this object is destroyed.
  */
  Mem_root_array<Binary_diff_vector *> m_binary_diff_vectors;

  /**
    The logical diffs that have been collected for JSON operations in
    the current row. ...
  */
  Mem_root_array<Json_diff_vector *> m_logical_diff_vectors;

  /**
    A buffer that can be used to hold the partially updated column value while
    performing the update in memory.
  */
  String m_buffer;

  /// Should logical JSON diffs be collected in addition to binary diffs?
  bool collect_logical_diffs() const {
    /*
      We only allocate logical diff vectors when we want logical diffs
      to be collected, so check if we have any.
    */
    return !m_logical_diff_vectors.empty();
  }
};
```

三件套的协作（标记 → 建立 → 逐行收集/回写 → 跨行清空 → 语句末销毁）：

```cpp
// [T5] sql/table.cc
bool TABLE::mark_column_for_partial_update(const Field *field) {
  assert(field->table == this);
  if (m_partial_update_columns == nullptr) {
    MY_BITMAP *map = new (&mem_root) MY_BITMAP;
    my_bitmap_map *buf =
        static_cast<my_bitmap_map *>(mem_root.Alloc(s->column_bitmap_size));
    if (map == nullptr || buf == nullptr || bitmap_init(map, buf, s->fields))
      return true; /* purecov: inspected */
    m_partial_update_columns = map;
  }

  bitmap_set_bit(m_partial_update_columns, field->field_index());
  return false;
}

bool TABLE::setup_partial_update(bool logical_diffs) {
  DBUG_TRACE;
  assert(m_partial_update_info == nullptr);
  THD *thd = current_thd;

  if (!has_columns_marked_for_partial_update()) return false;
  ...
  m_partial_update_info = new (thd->mem_root)
      Partial_update_info(this, m_partial_update_columns, logical_diffs);
  return thd->is_error();
}

// ★ 是否收集"逻辑 diff"取决于 binlog_row_value_options=PARTIAL_JSON 且确实在记 RBR
bool TABLE::setup_partial_update() {
  THD *thd = current_thd;

  bool logical_diffs =
      (thd->variables.binlog_row_value_options & PARTIAL_JSON_UPDATES) != 0 &&
      mysql_bin_log.is_open() &&
      (thd->variables.option_bits & OPTION_BIN_LOG) != 0 &&
      log_bin_use_v1_row_events == 0 &&
      thd->is_current_stmt_binlog_format_row();
  return setup_partial_update(logical_diffs);
}

void TABLE::cleanup_partial_update() {
  DBUG_TRACE;
  destroy(m_partial_update_info);
  m_partial_update_info = nullptr;
}

String *TABLE::get_partial_update_buffer() {
  assert(m_partial_update_info != nullptr);
  return &m_partial_update_info->m_buffer;
}

void TABLE::clear_partial_update_diffs() {
  DBUG_TRACE;
  if (m_partial_update_info != nullptr) {
    for (auto v : m_partial_update_info->m_binary_diff_vectors)
      if (v != nullptr) v->clear();

    bitmap_copy(&m_partial_update_info->m_enabled_binary_diff_columns,
                m_partial_update_columns);
    ...
  }
}

const Binary_diff_vector *TABLE::get_binary_diffs(const Field *field) const {
  if (!is_binary_diff_enabled(field)) return nullptr;
  return m_partial_update_info->m_binary_diff_vectors[field->field_index()];
}

// ★ 相邻/重叠区间合并，保证 diff 列表始终是"不相交且有序"的
bool TABLE::add_binary_diff(const Field *field, size_t offset, size_t length) {
  Binary_diff_vector *diffs =
      m_partial_update_info->m_binary_diff_vectors[field->field_index()];

  /*
    Find the first diff that does not end before the diff we want to insert.
    That is, we find the first diff that is either overlapping with the diff we
    want to insert, adjacent to the diff we want to insert, or comes after the
    diff that we want to insert.

    In the case of overlapping or adjacent diffs, we want to merge the diffs
    rather than insert a new one.
  */
  Binary_diff_vector::iterator first_it =
      std::lower_bound(diffs->begin(), diffs->end(), offset,
                       [](const Binary_diff &diff, size_t start_offset) {
                         return diff.offset() + diff.length() < start_offset;
                       });
  ...
}

// 一旦某列在"当前行"上判定不可部分更新，就地撤销
void TABLE::disable_binary_diffs_for_current_row(const Field *field) {
  // Remove the diffs collected for the column.
  m_partial_update_info->m_binary_diff_vectors[field->field_index()]->clear();
  // Mark the column as disabled.
  bitmap_clear_bit(&m_partial_update_info->m_enabled_binary_diff_columns,
                   field->field_index());
}
```

```cpp
// [T1] sql/table.h —— m_record_buffer：多-row-read 预读缓冲
  uchar *record[2]{nullptr, nullptr}; /* Pointer to records. mysql format的行数据 */
  ...
  /// Buffer for use in multi-row reads. Initially empty.
  Record_buffer m_record_buffer{0, 0, nullptr};
```

```cpp
// [T26] sql/record_buffer.h
class Record_buffer {
  /// The maximum number of records that can be stored in the buffer.
  ha_rows m_max_records;
  /// The number of bytes available for each record.
  size_t m_record_size;
  /// The number of records currently stored in the buffer.
  ha_rows m_count = 0;
  /// The @c uchar buffer that holds the records.
  uchar *m_buffer;
  /// Tells if end-of-range was found while filling the buffer.
  bool m_out_of_range = false;
```

> ★ **勘误式澄清**：`m_record_buffer` 常被误归入部分更新三件套，实际它与 JSON 无关——它是 multi-range read 的**行预读缓冲**（`TableScanIterator` 构造时按 `expected_rows` 决定要不要开）。部分更新真正用来暂存"改了一半的 JSON 值"的缓冲是 `Partial_update_info::m_buffer`（一个 `String`，由 `TABLE::get_partial_update_buffer()` 暴露）。二者同名不同物。

**部分更新三件套协作图**：

```
  ┌── 优化期（prepare）───────────────────────────────────────────────┐
  │ UPDATE t SET j = JSON_SET(j, '$.a', 1)                            │
  │   └─> TABLE::mark_column_for_partial_update(field j)              │
  │         惰性分配 m_partial_update_columns（TABLE::mem_root）       │
  │         bitmap_set_bit(..., j->field_index())                     │
  └──────────────────────────────────────────────────────────────────┘
                              │
  ┌── 执行期开始（第一次写行前）──────────────────────────────────────┐
  │ TABLE::setup_partial_update()                                      │
  │   logical_diffs = binlog_row_value_options & PARTIAL_JSON_UPDATES  │
  │                   && binlog 开 && 当前语句 RBR && 非 v1 事件        │
  │   new Partial_update_info(this, m_partial_update_columns, logical) │
  │        ├── m_enabled_binary_diff_columns  = 资格位图的副本          │
  │        ├── m_enabled_logical_diff_columns = 资格位图 or 全 0        │
  │        ├── m_binary_diff_vectors[i]  (Binary_diff_vector)          │
  │        ├── m_logical_diff_vectors[i] (Json_diff_vector, 可选)      │
  │        └── m_buffer (String)  ← 这里才是"半个 JSON"的暂存区         │
  └──────────────────────────────────────────────────────────────────┘
                              │
  ┌── 每一行 ─────────────────────────────────────────────────────────┐
  │ 1) Field_json 计算新值，把"哪些字节区间变了"喂给                    │
  │      TABLE::add_binary_diff(field, offset, length)                 │
  │        → lower_bound/upper_bound 合并相邻+重叠区间                  │
  │      （若发现本行不适用：disable_binary_diffs_for_current_row()）    │
  │ 2) 引擎侧：handler::update_row() 读 get_binary_diffs(field)        │
  │        → InnoDB 只重写这些 (offset,length) 片段，而不是整列          │
  │ 3) binlog 侧：若 collect_logical_diffs()，rpl_record.cc 把          │
  │      Json_diff_vector 序列化成 PARTIAL_JSON 更新事件                 │
  │ 4) 行间：TABLE::clear_partial_update_diffs() 清空 vector 并把        │
  │      enabled 位图重置回"资格位图"                                   │
  └──────────────────────────────────────────────────────────────────┘
                              │
  ┌── 语句结束 ───────────────────────────────────────────────────────┐
  │ close_thread_tables() → TABLE::cleanup_partial_update()           │
  │     destroy(m_partial_update_info); m_partial_update_info = nullptr│
  │ （m_partial_update_columns 属 TABLE::mem_root，随 TABLE 复用而重置） │
  └──────────────────────────────────────────────────────────────────┘
```

备机回放侧的对应逻辑印证了"每个 Partial 事件都要先清空再收集"：

```cpp
// [T34] sql/rpl_record.cc
    if ((event_value_options & PARTIAL_JSON_UPDATES) != 0 && !only_seek) {
      if (table->has_columns_marked_for_partial_update())
        /*
          partial_update_info has been initialized already (so this is
          not the first row of the statement having the PARTIAL_JSON
          bit set).  Clear the diff vector between rows.
        */
        table->clear_partial_update_diffs();
      else {
        /*
          partial_update_info has not been initialized ... Initialize
          partial_update_info to allow the optimizer to collect partial
          diffs when applying any diff. ...
        */
```

#### 2.7 ★ 成本模型与优化器状态组

这一组管"本次查询对这张表的索引可用性判定 + 成本常数句柄 + 分区裁剪结果"——**纯 per-TABLE 的执行期状态**，每次 `TABLE::reset()` / `init()` 都要清零。

```cpp
// [T1] sql/table.h
  /*
    A set of keys that can be used in the query that references this
    table.

    All indexes disabled on the table's TABLE_SHARE (see TABLE::s) will be
    subtracted from this set upon instantiation. Thus for any TABLE t it holds
    that t.keys_in_use_for_query is a subset of t.s.keys_in_use. Generally we
    must not introduce any new keys here (see setup_tables).

    The set is implemented as a bitmap.
  */
  Key_map keys_in_use_for_query;
  /* Map of keys that can be used to calculate GROUP BY without sorting */
  Key_map keys_in_use_for_group_by;
  /* Map of keys that can be used to calculate ORDER BY without sorting */
  Key_map keys_in_use_for_order_by;
  KEY *key_info{nullptr}; /* data of keys defined for the table */
  /**
    Key part array for generated keys, used for materialized derived tables.
    This is a contiguous array, with size given by s->max_tmp_key_parts.
  */
  KEY_PART_INFO *base_key_parts{nullptr};
```

```cpp
// [T1] sql/table.h
  bool force_index{false};
  /**
    Flag set when the statement contains FORCE INDEX FOR ORDER BY
    See Table_ref::process_index_hints().
  */
  bool force_index_order{false};

  /**
    Flag set when the statement contains FORCE INDEX FOR GROUP BY
    See Table_ref::process_index_hints().
  */
  bool force_index_group{false};
  bool const_table{false};
  /// True if writes to this table should not write rows and just write keys.
  bool no_rows{false};
```

```cpp
// [T1] sql/table.h
  /**
    The result of applying a unique operation (by row ID) to the table, if done.
    In particular, this is done in some forms of index merge.
  */
  Sort_result unique_result;
  partition_info *part_info{nullptr}; /* Partition related information */
  /* If true, all partitions have been pruned away */
  bool all_partitions_pruned_away{false};

 private:
  /// Cost model object for operations on this table
  Cost_model_table m_cost_model;
```

```cpp
// [T27] sql/sql_sort.h —— Sort_result：三种"结果落在哪"的形态
class Sort_result {
 public:
  Sort_result() : sorted_result_in_fsbuf(false), sorted_result_end(nullptr) {}

  bool has_result_in_memory() const {
    return sorted_result || sorted_result_in_fsbuf;
  }

  bool has_result() const {
    return has_result_in_memory() || (io_cache && my_b_inited(io_cache));
  }

  IO_CACHE *io_cache{nullptr};
  /**
    If the entire result fits in memory, we skip the merge phase. ...
    This new buffer is [sorted_result ... sorted_result_end]
    @see save_index()
  */
  bool sorted_result_in_fsbuf{false};
  unique_ptr_my_free<uchar> sorted_result{nullptr};
  uchar *sorted_result_end{nullptr};

  ha_rows found_records{0};  ///< How many records in sort.
};
```

索引提示的编译入口（三个位图 + 三个 force 标志全部由 index hints 编译而来）：

```cpp
// [T5] sql/table.cc
bool Table_ref::process_index_hints(const THD *thd, TABLE *tbl) {
  /* initialize the result variables */
  tbl->keys_in_use_for_query = tbl->keys_in_use_for_group_by =
      tbl->keys_in_use_for_order_by = tbl->s->usable_indexes(thd);

  /* index hint list processing */
  if (index_hints) {
    ...
    /* process FORCE INDEX as USE INDEX with a flag */
    if (!index_order[INDEX_HINT_FORCE].is_clear_all()) {
      tbl->force_index_order = true;
      index_order[INDEX_HINT_USE].merge(index_order[INDEX_HINT_FORCE]);
    }

    if (!index_group[INDEX_HINT_FORCE].is_clear_all()) {
      tbl->force_index_group = true;
      index_group[INDEX_HINT_USE].merge(index_group[INDEX_HINT_FORCE]);
    }
    ...
    /* apply USE INDEX */
    if (!index_join[INDEX_HINT_USE].is_clear_all() || have_empty_use_join)
      tbl->keys_in_use_for_query.intersect(index_join[INDEX_HINT_USE]);
    if (!index_order[INDEX_HINT_USE].is_clear_all() || have_empty_use_order)
      tbl->keys_in_use_for_order_by.intersect(index_order[INDEX_HINT_USE]);
    if (!index_group[INDEX_HINT_USE].is_clear_all() || have_empty_use_group)
      tbl->keys_in_use_for_group_by.intersect(index_group[INDEX_HINT_USE]);

    /* apply IGNORE INDEX */
    tbl->keys_in_use_for_query.subtract(index_join[INDEX_HINT_IGNORE]);
    tbl->keys_in_use_for_order_by.subtract(index_order[INDEX_HINT_IGNORE]);
    tbl->keys_in_use_for_group_by.subtract(index_group[INDEX_HINT_IGNORE]);
  }

  /* make sure covering_keys don't include indexes disabled with a hint */
  tbl->covering_keys.intersect(tbl->keys_in_use_for_query);
  return false;
}
```

```cpp
// [T22] sql/opt_costmodel.h
class Cost_model_table {
 public:
  /**
    Initializes the cost model object.
    This function must be called before calling any cost estimation
    functions for a query. It should also be called when starting
    optimization of a new query in case any cost estimate constants
    have changed.
  */
  void init(const Cost_model_server *cost_model_server, const TABLE *table);

  double row_evaluate_cost(double rows) const { ... }
  double key_compare_cost(double keys) const { ... }
```

`all_partitions_pruned_away` 的置位点与效果：`range_optimizer/partition_pruning.cc` 里 `if (bitmap_is_clear_all(&part_info->read_partitions)) { table->all_partitions_pruned_away = true; return false; }`，随后 `sql_optimizer.cc` 在 `extract_empty_table` 分支把它当作"零行表"直接 `mark_const_table(tab, nullptr)`，`sql_update.cc` / `sql_delete.cc` 则据此把 `no_rows` 置真并跳过扫描。`TABLE::reset()` 每次都把它清回 `false`。

#### 2.8 ★ 触发器与 CHECK 约束运行态

这一组管"绑定了本 TABLE 的、已经 Item 化/可求值的约束执行器"——与 SHARE 侧的字符串模板严格分工。

```cpp
// [T1] sql/table.h
  Field *fts_doc_id_field{nullptr}; /* Set if FTS_DOC_ID field is present */

  /* Table's triggers, 0 if there are no of them */
  Table_trigger_dispatcher *triggers{nullptr};   // [T23] sql/sql_trigger.h
  Table_ref *pos_in_table_list{nullptr}; /* Element referring to this table */
  /* Position in thd->locked_table_list under LOCK TABLES */
  Table_ref *pos_in_locked_tables{nullptr};
  ...
  // List of table check constraints.
  Sql_table_check_constraint_list *table_check_constraint_list{nullptr};
```

```cpp
// [T24] sql/sql_check_constraint.h —— SHARE 侧（模板）与 TABLE 侧（运行态）的类型分工
/**
  Class to represent check constraint in the TABLE_SHARE.

  The instance of Sql_check_constraint_share contains information as name,
  state and expression in string form. This information is obtained from
  the data-dictionary. The check expression is not in itemized (materialized)
  form here.
*/
class Sql_check_constraint_share {
 public:
  Sql_check_constraint_share(const LEX_CSTRING &name,
                             const LEX_CSTRING &expr_str, bool is_enforced)
      : m_name(name), m_expr_str(expr_str), m_is_enforced(is_enforced) {}

  /// Constraint name.
  LEX_CSTRING &name() { return m_name; }
  /// Check expression in string form.
  LEX_CSTRING &expr_str() { return m_expr_str; }
  /// Check constraint state (enforced / not enforced)
  bool is_enforced() { return m_is_enforced; }

 private:
  /// Check constraint name.
  LEX_CSTRING m_name{nullptr, 0};
  /// Check constraint expression.
  LEX_CSTRING m_expr_str{nullptr, 0};
  /// Check constraint state.
  bool m_is_enforced{true};
};

/**
  Class to represent check constraint in the TABLE instance.

  The Sql_table_check_constraint is a Sql_check_constraint_share with reference
  to the parent TABLE instance and itemized (materialized) form of check
  constraint expression.
  Sql_table_check_constraint is prepared from the Sql_check_constraint_share of
  TABLE_SHARE instance.
*/
class Sql_table_check_constraint : public Sql_check_constraint_share {
 public:
  /// Value generator.
  Value_generator *value_generator() { return m_val_gen; }
  /// Reference to owner table.
  TABLE *table() const { return m_table; }
 private:
  /// Value generator for the check constraint expression.
  Value_generator *m_val_gen{nullptr};
  /// Parent table reference.
  TABLE *m_table{nullptr};
};

/// Type for the list of Sql_check_constraint_share elements.
using Sql_check_constraint_share_list =
    Mem_root_array<Sql_check_constraint_share>;
/// Type for the list of Sql_table_check_constraint elements.
using Sql_table_check_constraint_list =
    Mem_root_array<Sql_table_check_constraint>;
```

```cpp
// [T5] sql/table.cc —— SHARE 模板 → TABLE 运行态的"展开"发生在 open_table_from_share()
  if (share->check_constraint_share_list != nullptr) {
    assert(share->check_constraint_share_list->size() > 0);

    outparam->table_check_constraint_list =
        new (root) Sql_table_check_constraint_list(root);
    if (outparam->table_check_constraint_list == nullptr) goto err;  // OOM

    if (outparam->table_check_constraint_list->reserve(
            share->check_constraint_share_list->size()))
      goto err;  // OOM

    // Unpack check constraint expressions.
    for (const Sql_check_constraint_share &cc_share :
         *share->check_constraint_share_list) {
      Value_generator *val_gen_ptr = nullptr;
      if (parse_gcol_expr(thd, outparam, ..., cc_share.name().str,
                          nullptr, is_create_table, &error_reported))
        goto err;

      outparam->table_check_constraint_list->push_back(
          Sql_table_check_constraint(cc_share.name(), cc_share.expr_str(),
                                     cc_share.is_enforced(), val_gen_ptr,
                                     outparam));
    }
```

```cpp
// [T5] sql/table.cc —— 运行期：求值所需的列要进 read/write set
void TABLE::mark_check_constraint_columns(bool is_update) {
  assert(table_check_constraint_list != nullptr);

  bool bitmap_updated = false;
  for (Sql_table_check_constraint &tbl_cc : *table_check_constraint_list) {
    if (tbl_cc.is_enforced()) {
      /*
        For update operation, check constraint should be evaluated if it is
        dependent on any of the updated column.
      */
```

**分工一览**：

```
  SHARE 侧（模板，跨 TABLE 共享）                TABLE 侧（运行态，per-TABLE）
  ┌───────────────────────────────┐        ┌──────────────────────────────────┐
  │ check_constraint_share_list   │  展开  │ table_check_constraint_list       │
  │  Mem_root_array<              │ ─────► │  Mem_root_array<                  │
  │    Sql_check_constraint_share>│ (open  │    Sql_table_check_constraint>    │
  │   · name      (LEX_CSTRING)   │  table │   · 继承 name / expr_str /        │
  │   · expr_str  (字符串)         │  from  │     is_enforced                   │
  │   · is_enforced(bool)         │  share)│   · m_val_gen : Value_generator*  │
  │   —— 无 Item 树、无 TABLE 指针  │        │       （已 parse 的表达式 + Item 树）│
  └───────────────────────────────┘        │   · m_table   : TABLE*            │
                                           └──────────────────────────────────┘
  生命周期：随 SHARE，TDC 淘汰才销毁          生命周期：随 TABLE，每次开表重建，
                                            语句末 cleanup_items() 释放 Item
```

`triggers` 同理：`Table_trigger_dispatcher` 持有**已解析到本 TABLE 字段上**的触发器动作（`sp_head` + Item 树），因此必须在 TABLE 侧；这也是"`fields_set_during_insert` 只对基表有意义、临时表不能有触发器"注释的由来（`[T1]`：`Triggers can not be defined for temporary tables, so this bitmap does not matter for temporary tables.`）。

#### 2.9 执行期迭代器组

这一组管"filesort / 去重 / INTERSECT·EXCEPT 计数器"的执行期句柄与语义标志——它们让 `TABLE` 在**不知道具体迭代器类型**的前提下，能在 `filesort_free_buffers()` 里统一回收资源。

```cpp
// [T1] sql/table.h
  /**
    Initialized in Item_func_group_concat::setup for appropriate
    temporary table if GROUP_CONCAT is used with ORDER BY | DISTINCT
    and BLOB field count > 0.
  */
  Blob_mem_storage *blob_storage{nullptr};

  /**
    Not owned by the TABLE; used only from filesort_free_buffers().
    See comments on SortingIterator::CleanupAfterQuery().
  */
  SortingIterator *sorting_iterator{nullptr};
  SortingIterator *duplicate_removal_iterator{nullptr};
```

```cpp
// [T1] sql/table.h —— INTERSECT / EXCEPT 的三件套（含大段设计注释，节选）
  /// A priori unlimited. We pass this on to TableScanIterator at construction
  /// time, q.v., to limit the number of rows out of an EXCEPT or INTERSECT.
  /// For these set operations, we do not know enough to enforce the limit at
  /// materialize time (as for UNION): only when reading the rows with
  /// TableScanIterator do we check the counters.
  ha_rows m_limit_rows{HA_POS_ERROR};

 private:
  /// The set counter. It points to the field in the materialized table
  /// holding the counter used to compute INTERSECT and EXCEPT, in record[0].
  /// For EXCEPT [DISTINCT | ALL] and INTERSECT DISTINCT this is a simple 64
  /// bits counter. For INTERSECT ALL, it is subdivided into two sub counters
  /// cf. class HalfCounter, cf. MaterializeQueryBlock. See set_counter().
  Field_longlong *m_set_counter{nullptr};

  /// True if we have EXCEPT, else we have INTERSECT. See is_except() and
  /// is_intersect().
  bool m_except{false};

  /// If m_set_counter is set: true if last block has DISTINCT semantics,
  /// either because it is marked as such, or because we have computed this
  /// to give an equivalent answer. If false, we have ALL semantics.
  /// It will be true if any DISTINCT is given in the merged N-ary set
  /// operation. See is_distinct().
  bool m_last_operation_is_distinct{false};

 public:
  /// Test if this tmp table stores the result of a UNION set operation or
  /// a single table.
  bool is_union_or_table() const { return m_set_counter == nullptr; }
  bool is_intersect() const { return m_set_counter != nullptr && !m_except; }
  bool is_except() const { return m_set_counter != nullptr && m_except; }
  bool is_distinct() const { return m_last_operation_is_distinct; }

  void set_set_counter(Field_longlong *set_counter, bool is_except) {
    m_set_counter = set_counter;
    m_except = is_except;
  }
  void set_distinct(bool distinct) { m_last_operation_is_distinct = distinct; }
  Field_longlong *set_counter() { return m_set_counter; }
```

```cpp
// [T29] sql/filesort.cc —— 唯一回收点
void filesort_free_buffers(TABLE *table, bool full) {
  table->unique_result.sorted_result.reset();
  assert(!table->unique_result.sorted_result_in_fsbuf);
  table->unique_result.sorted_result_in_fsbuf = false;

  if (full) {
    if (table->sorting_iterator != nullptr) {
      table->sorting_iterator->CleanupAfterQuery();
      table->sorting_iterator = nullptr;  // The TABLE object can be reused.
    }
    if (table->duplicate_removal_iterator != nullptr) {
      table->duplicate_removal_iterator->CleanupAfterQuery();
      table->duplicate_removal_iterator =
          nullptr;  // The TABLE object can be reused.
    }
  }
}
```

```cpp
// [T28] sql/iterators/basic_row_iterators.cc —— m_limit_rows 只在集合运算表上生效
TableScanIterator::TableScanIterator(THD *thd, TABLE *table,
                                     double expected_rows,
                                     ha_rows *examined_rows)
    : TableRowIterator(thd, table),
      m_record(table->record[0]),
      m_expected_rows(expected_rows),
      m_examined_rows(examined_rows),
      m_limit_rows(table->set_counter() != nullptr ? table->m_limit_rows
                                                   : HA_POS_ERROR) {}
  ...
      if (++m_stored_rows > m_limit_rows) {
        return HandleError(HA_ERR_END_OF_FILE);
      }
```

★ 注意"不拥有"语义：`sorting_iterator` / `duplicate_removal_iterator` 由**迭代器树**拥有，TABLE 只保存裸指针以便 `filesort_free_buffers()` 回调；因此 `sql_update.cc` / `sql_delete.cc` / `sql_select.cc` 在把迭代器从树里摘出来单独持有时，都会先 `table->sorting_iterator = nullptr;` 并注明 "to avoid double-destroy of the SortingIterator"。

#### 2.10 隐藏列 / 生成列 / 函数索引组

这一组管"临时表头部的隐藏列、带默认表达式的列、函数索引的隐藏生成列、BLOB 深拷贝开关、FTS_DOC_ID 列"。

```cpp
// [T1] sql/table.h
  /**
    A bitmap marking the hidden generated columns that exists for functional
    indexes.
  */
  MY_BITMAP fields_for_functional_indexes;
  ...
  THD *in_use{nullptr};  // 持有该table的线程
  Field **field{nullptr}; /* Pointer to fields */
  /// Count of hidden fields, if internal temporary table; 0 otherwise.
  uint hidden_field_count{0};
  ...
  Field *next_number_field{nullptr};       /* Set if next_number is activated */
  Field *found_next_number_field{nullptr}; /* Set on open */
  /// Pointer to generated columns
  Field **vfield{nullptr};
  /// Pointer to fields having the default value generated
  Field **gen_def_fields_ptr{nullptr};
  /// Field used by unique constraint
  Field *hash_field{nullptr};
```

```cpp
// [T1] sql/table.h
  bool null_row{false};

  bool copy_blobs{false}; /* copy_blobs when storing */
```

```cpp
// [T5] sql/table.cc —— 三者的初始化点（open_table_from_share 内）
    bitmap_init(&outparam->fields_for_functional_indexes,
                (my_bitmap_map *)(bitmaps + bitmap_size * 5), share->fields);
  ...
        // Mark hidden generated columns for functional indexes.
        if ((*field_ptr)->is_field_for_functional_index()) {
          bitmap_set_bit(&outparam->fields_for_functional_indexes,
                         (*field_ptr)->field_index());
        }
  ...
    /* Set TABLE::fts_doc_id_field for tables with FT KEY */
    if ((key_info->flags & HA_FULLTEXT))
      outparam->fts_doc_id_field = fts_doc_id_field;
  ...
  // Check default value expressions against table's storage engine
  if (share->gen_def_field_count && outparam->file &&
      (!(outparam->file->ha_table_flags() & HA_SUPPORTS_DEFAULT_EXPRESSION))) {
    my_error(ER_UNSUPPORTED_ACTION_ON_DEFAULT_VAL_GENERATED, ...);
  ...
  // Unpack generated default value expressions
  outparam->gen_def_fields_ptr = nullptr;
  if (share->gen_def_field_count) {
    Field **gen_def_field =
        root->ArrayAlloc<Field *>(share->gen_def_field_count + 1);
```

★ `hidden_field_count` 与 `gen_def_field_count` 是**一对容易混淆的"偏移"**：前者只在内部临时表上非 0，表示"表头有几个隐藏列（hash 去重列、集合计数器等）"，消费方式是 `visible_field_ptr() = field + hidden_field_count` / `visible_field_count() = s->fields - hidden_field_count`；后者来自 SHARE（`s->gen_def_field_count`），表示"有几个列的默认值是表达式生成的"，消费方式是 `TABLE::gen_def_fields_ptr[]` 数组（最后一个元素为 `nullptr` 作哨兵）。
`copy_blobs` 是"存 BLOB 时必须深拷一份"的开关（`Field_blob::store` 里 `if (table->copy_blobs || ...) value.copy();`），物化派生表 clone、`LOAD DATA`、hash join 遇到 GEOMETRY 列时都会置真。

#### 2.11 HANDLER 语句与锁位置组

这一组管"`HANDLER t OPEN/READ` 这种跨语句持有表的特殊身份"与"本 TABLE 在 `MYSQL_LOCK` 两个并行数组中的下标"。

```cpp
// [T1] sql/table.h
  /**
    If set, indicate that the table is not replicated by the server.
  */
  bool no_replicate{false};
  /* To signal that the table is associated with a HANDLER statement */
  bool open_by_handler{false};
  ...
  uint lock_position{0};   /* Position in MYSQL_LOCK.table */
  uint lock_data_start{0}; /* Start pos. in MYSQL_LOCK.locks */
  uint lock_count{0};      /* Number of locks */
  uint db_stat{0};         /* mode of file as in handler.h */
  int current_lock{0};     /* Type of lock on table */
```

```cpp
// [T31] sql/lock.cc —— MYSQL_LOCK 是"两个并行数组"：table[] 与 locks[]
  /* Move all write locked tables first */
  TABLE **table = sql_lock->table;
  for (i = found = 0; i < sql_lock->table_count; i++) {
    assert(sql_lock->table[i]->lock_position == i);
    if ((uint)sql_lock->table[i]->reginfo.lock_type > TL_WRITE_ALLOW_WRITE) {
      std::swap(*table, sql_lock->table[i]);
      table++;
  ...
  found = 0;
  for (i = 0; i < sql_lock->table_count; i++) {
    TABLE *tbl = *table;
    tbl->lock_position = (uint)(table - sql_lock->table);
    tbl->lock_data_start = found;
    found += tbl->lock_count;      // 一个表可能占多个 THR_LOCK_DATA（分区/MERGE）
    table++;
  }
```

```cpp
// [T31] sql/lock.cc —— 摘掉一张表后要同时修正两个下标
        /*
          Fix moved table elements.
          lock_position is the index in the 'locked->table' array,
          it must be fixed by one.
          table->lock_data_start is pointer to the lock data for this table
          in the 'locked->locks' array, they must be fixed by 'removed_locks',
          the lock data count of the removed table.
        */
        for (j = i; j < old_tables; j++) {
          tbl = locked->table[j];
          tbl->lock_position--;
          tbl->lock_data_start -= removed_locks;
        }
```

`open_by_handler` 的语义是"这张表是被 `HANDLER` 语句打开的，扫描可以跨语句存在、可以不持锁"，大量 `assert(table->open_by_handler)` 正是靠它豁免常规锁断言（`[T32]`：`assert(inited == NONE || table->open_by_handler);`、`assert(table->open_by_handler || table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);`）。`sql_base.cc` 在 `mark_temp_tables_as_free_for_reuse()` 里显式跳过 `open_by_handler` 的临时表，避免 HANDLER 正在读的表被回收。

#### 2.12 binlog 与杂项组

这一组管"复制开关 / 临时表 DROP 是否记 binlog / 只建键不写行的写路径 / 别名身份 / 调试序号 / 对象失效标记"。

```cpp
// [T1] sql/table.h
  const char *alias{nullptr};  ///< alias or table name
  ...
  bool alias_name_used{false};         /* true if table_name is alias */
  bool get_fields_in_item_tree{false}; /* Signal to fix_field */
```

> ⚠️ `get_fields_in_item_tree` 是 `TABLE` 的 **`bool` 成员**（`/* Signal to fix_field */`），不是方法——这是最容易误读的一个名字。

```cpp
// [T1] sql/table.h —— m_invalid_dict / m_invalid_stats（均为 private，带锁约定）
 private:
  /**
    This TABLE object is invalid and cannot be reused. TABLE object might have
    inconsistent info or handler might not allow some operations.

    For example, TABLE might have inconsistent info about partitioning.
    We also use this flag to avoid calling handler::reset() for partitioned
    InnoDB tables after in-place ALTER TABLE API commit phase and to force
    closing table after REPAIR TABLE has failed during its prepare phase as
    well.

    @note This member can be set only by thread that owns/has opened the
          table and while holding its THD::LOCK_thd_data lock.
          It can be read without locking by this owner thread, or by some other
          thread concurrently after acquiring owner's THD::LOCK_thd_data.

    @note The TABLE will not be reopened under LOCK TABLES in
          close_thread_tables().
  */
  bool m_invalid_dict{false};

  /**
    This TABLE object is invalid and cannot be reused as it has outdated
    rec_per_key and handler stats.

    @note This member is protected from concurrent access to it by lock of
          Table Cache's partition to which this TABLE object belongs,
  */
  bool m_invalid_stats{false};
```

```cpp
// [T1] sql/table.h —— 读写接口（带断言的锁契约）
  inline bool has_invalid_dict() const {
    assert(assert_invalid_dict_is_locked(this));
    return !db_stat || m_invalid_dict;
  }
  /// @note Can be called by thread owning Table_cache::m_lock
  inline bool has_invalid_stats() {
    assert(assert_invalid_stats_is_locked(this));
    return m_invalid_stats;
  }
  void invalidate_dict();
  void invalidate_stats();
```

```cpp
// [T5] sql/table.cc —— 两个失效标记的写者，锁约定在代码里直接体现
void TABLE::invalidate_dict() {
  /*
    m_invalid_dict can be only updated by TABLE owner and while holding its
    LOCK_thd_data lock.
  */
  assert(current_thd == in_use);
  mysql_mutex_lock(&in_use->LOCK_thd_data);
  m_invalid_dict = true;
  mysql_mutex_unlock(&in_use->LOCK_thd_data);
}

void TABLE::invalidate_stats() {
  // m_invalid_stats is protected by Table_cache::m_lock.
  table_cache_manager.assert_owner_all();
  m_invalid_stats = true;
}
```

```cpp
// [T1] sql/table.h —— 内部临时表调试序号（仅 debug 构建存在）
#ifndef NDEBUG
  /**
    Internal tmp table sequential number. Increased in the order of
    creation. Used for debugging purposes when many tmp tables are used
    during execution (e.g several windows with window functions)
  */
  uint tmp_table_seq_id{0};
#endif
```

```cpp
// [T1] sql/table.h —— 临时表 DROP 是否记 binlog（私有，配 setter/getter）
  /**
    This flag decides whether or not we should log the drop temporary table
    command.
  */
  bool should_binlog_drop_if_temp_flag{false};
  ...
  void set_binlog_drop_if_temp(bool should_binlog);
  bool should_binlog_drop_if_temp(void) const;
```

`alias_name_used` 由 `TABLE::init()` 计算（`[T5]`：`alias_name_used = my_strcasecmp(table_alias_charset, s->table_name.str, tl->alias);`，仅当 `thd->lex->need_correct_ident()` 时才比较），供 `Item` 打印时决定是否要输出 `db.tbl` 前缀。
`no_rows` 由 `[T1]` 注释定义为"写操作只写键、不写行"（如 `ALTER TABLE ... ADD INDEX` 建索引路径）。

#### 2.13 小结：TABLE 侧成员的归属判据

★ 上述 7 组 36 个成员清一色是 **per-TABLE、per-语句** 的状态，与 SHARE 侧"终身不变"的清单形成**镜像**：

| 侧 | 判据 | 例子 |
|---|---|---|
| `TABLE_SHARE` | 随表定义终身不变、可跨 TABLE 共享 | 外键双向链、CHECK 模板、键元数据、`base_rec_per_key` |
| `TABLE` | per-TABLE / per-语句，复用前必须擦干净 | `keys_in_use_for_*`、`force_index_*`、`m_partial_update_info`、`sorting_iterator` |

`TABLE::reset()` 逐个清零 `force_index` / `force_index_order` / `force_index_group` / `all_partitions_pruned_away`；`close_thread_tables()` 里 `cleanup_partial_update()`；`filesort_free_buffers()` 里把两个迭代器置 nullptr 并注明 "The TABLE object can be reused"——**这些都是"TABLE 回表缓存前必须擦干净"的清单**。而 `m_invalid_dict` / `m_invalid_stats` 是"擦不干净就别复用"的兜底。

与本文其它章节的衔接：

- **与「表统计信息」**：`m_invalid_stats` 是那一节的**失效开关**——`rec_per_key` / `handler::stats` 过期时置位，该 TABLE 就不再被表缓存复用；而"新的统计值从哪来"正是 1.6 节 `base_rec_per_key` 与 SHARE 侧共享数组的故事。两者合起来才构成完整的"统计信息生命周期"。
- **与「表类型」**：`hidden_field_count` / `m_set_counter` / `m_except` / `m_last_operation_is_distinct` / `m_limit_rows` / `tmp_table_seq_id` 只可能非默认于**内部临时表**（UNION / INTERSECT / EXCEPT 的物化表）；`triggers` / `table_check_constraint_list` 只可能存在于**基表**（临时表不支持触发器）；`open_by_handler` 标记 HANDLER 这种特殊打开方式。⇒ **看到这一组里哪个字段非默认，就能反推出这张表在「表类型」分类里的位置**。

### 3. ★ 三者（SHARE / TABLE / Field）指针关系图

```
   ┌──────────────── TABLE_SHARE  (全实例 1 份) ────────────────┐
   │ table_cache_key "db\0tbl\0"   db   table_name   path        │
   │ fields  reclength  null_bytes  keys  primary_key  db_type() │
   │ keys_in_use / visible_indexes / keys_for_keyread   all_set  │
   │ m_ref_count  m_version  next/**prev  cache_element[]        │
   │  Field **field ──────┐        KEY *key_info ─────┐         │
   │  uchar *default_values ─┐                        │         │
   └──────────────────────┼──┼────────────────────────┼─────────┘
              TABLE::s ────┘  │                        │ memcpy(深拷贝)
                              │                        ▼
   ┌────────────────────────┐ │      ┌──────────────────────────────┐
   │ TABLE  (每会话 1 份)    │ │      │ TABLE::key_info（实例副本）    │
   │ in_use / file /        │ │      │  key_part[].field ──> 实例列   │
   │ pos_in_table_list      │ │      └──────────────────────────────┘
   │ record[0] / record[1]  │ │
   │ read_set/write_set ─>def_*                        │
   │ Field **field ─────────┼─┘                        │
   │ KEY *key_info ─────────┼──────────────────────────┘
   └────────────────────────┘
                              ▼
        SHARE::default_values（模板行 = 默认值的字节镜像）
        ┌───────────┬────────┬────────┬────────┐
        │ null_bytes│ col 0  │ col 1  │ col 2  │
        └───────────┴────────┴────────┴────────┘
             ▲  SHARE::field[i]->ptr（模板 Field）
             │
   move_offset = record[0] - default_values
             │
             ▼
        TABLE::record[0]（实例行缓冲，布局完全相同）
        ┌───────────┬────────┬────────┬────────┐
        │ null_bytes│ col 0  │ col 1  │ col 2  │
        └───────────┴────────┴────────┴────────┘
             ▲  TABLE::field[i]->ptr = 模板 ptr + move_offset（克隆 Field）
```

⇒ **一句话读图**：`TABLE::field[i]` 与 `SHARE::field[i]` **一一对应但不是同一个对象**（前者是 `clone()`，差别只在 `ptr` 落点）；`TABLE::key_info` 与 `SHARE::key_info` **结构相同但不是同一块内存**（`memcpy` + 重绑）。

### 4. ★ 生命周期与缓存分层

#### 4.1 缓存分层全景

```
  dd.tables / dd.columns / dd.indexes                ── 持久化数据字典
        │ open_table_def()  [T4]
        v
  ┌────────────────────────────────────────────────────────────┐
  │ table_def_cache (TDC)                        全局 1 个      │
  │   key = table_cache_key；锁 = LOCK_open + COND_open         │
  │   oldest_unused_share ──> ... ──> end_of_unused_share        │
  │   （ref_count==0 的 SHARE 的 LRU 链）                        │
  └────────────────────────────────────────────────────────────┘
        │ 1 SHARE : N TABLE（SHARE::cache_element[] 是反向索引）
        v
  ┌────────────────────────────────────────────────────────────┐
  │ Table_cache × table_cache_instances（分实例降锁争用）        │
  │   m_cache: key ──> Table_cache_element*                     │
  │   m_unused_tables: 跨表的空闲 TABLE 环形双向链               │
  └────────────────────────────────────────────────────────────┘
        v
  ┌────────────────────────────────────────────────────────────┐
  │ Table_cache_element（1 SHARE 在 1 cache 实例中的投影）       │
  │   used_tables（in_use != nullptr） / free_tables（==nullptr）│
  └────────────────────────────────────────────────────────────┘
```

#### 4.2 SHARE 的诞生：先占坑、再放锁读 DD

```cpp
// [T3]（节选）
  if (!(share = alloc_table_share(db, table_name, key, key_length, open_secondary)))
    return nullptr;
  assign_new_table_id(share);
  table_def_cache->emplace(..., unique_ptr<TABLE_SHARE, Table_share_deleter>(share));
  share->increment_ref_count();      // ★ 先占坑，防中途被删
  share->m_open_in_progress = true;  // 并发首次打开的标记
  mysql_mutex_unlock(&LOCK_open);    // ★ 读 DD 期间放锁，提高并发
  {
    ...
    open_table_err = open_table_def(thd, share, *dynamic_cast<const dd::Table *>(abstract_table));
    if (!open_table_err) update_schema_options(sch, share);
    if (!open_table_err && read_histograms(thd, share, sch, abstract_table))
      open_table_err = true;
  }
  mysql_mutex_lock(&LOCK_open);
  share->m_open_in_progress = false;
  mysql_cond_broadcast(&COND_open);  // 唤醒等待者
```

⇒ 三个关键：① `m_open_in_progress` + `COND_open` 解决"同表并发首次打开"（第一个放锁读 DD，其余等待）；② **先 emplace + 加引用，再放锁**；③ `m_version` 在诞生时快照为当时的全局 `refresh_version`。

#### 4.3 ★ SHARE 的三条死法

```cpp
// [T3] —— 引用归零后：已过期则删，未过期则挂 LRU 尾
void release_table_share(TABLE_SHARE *share) {
  if (share->decrement_ref_count() == 0) {
    if (share->has_old_version() || table_def_shutdown_in_progress)
      table_def_cache->erase(to_string(share->table_cache_key));   // ① 已过期 → 删
    else {
      /* 挂到 oldest_unused_share 链尾（MRU 端）*/
      share->prev = end_of_unused_share.prev;
      *end_of_unused_share.prev = share;
      end_of_unused_share.prev = &share->next;
      share->next = &end_of_unused_share;
      if (table_def_cache->size() > table_def_size)                // ② 容量超限 → 从链头淘汰
        table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
    }
  }
}
```

③ **被 DDL/FLUSH 点名**：`tdc_remove_table()`。若还有引用，**不立即删**，而是 `clear_version()`（置 0 ⇒ `has_old_version()` 为真）+ 收走**空闲** TABLE；等最后一个 TABLE 归还时走 ①自动消失。

★ `free_table_share()` 有个"交棒"细节：若 `m_flush_tickets` 非空（有线程在 `FLUSH TABLES` 等它），销毁动作交给**最后一个被唤醒的等待者**——因为等待者手里还握着 `share` 指针。

#### 4.4 ★ TABLE 的两条归途

```cpp
// [T3] —— 语句结束时：能用就还，不能用才真关
static void release_or_close_table(THD *thd, TABLE *table) {
  Table_cache *tc = table_cache_manager.get_cache(thd);
  tc->lock();
  if (table->s->has_old_version() || table->has_invalid_dict() ||
      table->has_invalid_stats() || table_def_shutdown_in_progress) {
    tc->remove_table(table);
    intern_close_table(table);        /* ★ 销毁：closefrm() → ~TABLE() → my_free() */
  } else {
    tc->release_table(thd, table);    /* ★ 归还（默认路径）*/
  }
  tc->unlock();
}
```

- **归还（默认）**：`release_table()` 把 `in_use` 置空、移入 `free_tables`、挂到 `m_unused_tables` 尾。**不动 `file`、不动 `field`、不动位图**——下次直接复用，避免重建 handler 的高昂代价。
- **销毁（四种时机）**：SHARE 过期 / `invalid_dict`·`invalid_stats` / cache 容量从 LRU 头收 / 关机。
- ★ **淘汰方向**：收的是 `m_unused_tables` **头部**（最久未用），而不是刚归还的那张——注释明说 "We free the least used tables, not the subject table, to keep the LRU order."
- ★ **顺序细节**：`close_thread_table()` 里先解绑 `mdl_ticket`/`pos_in_table_list`，再 `ha_reset()`，最后才进 `LOCK_open` 临界区——**MDL 必须在把表还给缓存之后才释放**，否则别人可能拿到一张 MDL 已放的"半吊子"表。

#### 4.5 ★ 版本号机制（DDL / FLUSH 如何失效）

```cpp
// [T3]
    refresh_version++;                          /* 全局刷新：所有旧 SHARE 一夜过期 */
    table_cache_manager.free_all_unused_tables();
    while (oldest_unused_share->next)
      table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
```

⇒ 两种让 SHARE 过期的方式：
- **全局刷新**（`FLUSH TABLES` 无表名 / 关机）：`refresh_version++` ⇒ 所有旧 SHARE 全部过期；
- **点名刷新**（`ALTER/DROP/RENAME/TRUNCATE/FLUSH TABLES t`）：`tdc_remove_table()` → `share->clear_version()`（置 0）。

★ **关键不变式**："`refresh_version` 自增 + 移除未使用表和 SHARE"必须在 `LOCK_open` 下**原子**完成，否则 TDC 里可能出现"已过期但仍有 TABLE 在用"的 share，让 `wait_for_old_version()` 死等。**正在被使用的表不会被强行拔掉**——只打过期标记，等最后一个引用释放。

#### 4.6 生命周期时间轴

```
  阶段0  dd.tables / dd.columns / dd.indexes（持久化）
    │  open_table_def()  [T4]
    v
  阶段1  TABLE_SHARE 诞生   谁建：get_table_share()/alloc_table_share()
    · 对象+key缓冲+path+cache_element[] 在 SHARE 自己的 MEM_ROOT 上一次分配
    · m_version = refresh_version 快照；emplace 进 TDC；ref_count=1
    · fill_share/columns/indexes/partitioning/fk/check from DD
    · default_values(模板行) / field[](模板列) / key_info(模板键) / all_set
    │  open_table_from_share()  [T2]
    v
  阶段2  TABLE 实例化
    · new TABLE(); s=share; in_use=thd; get_new_handler() → file
    · record[0]/record[1]；share->field[i]->clone()+init()+move_field_offset()
    · key_info = memcpy(share->key_info)，key_part->field 重绑到实例列
    · 8 份位图 + default_column_bitmaps()
    │  解析/优化/执行
    v
  阶段3  TABLE 被"装修"成一次查询
    · mark_column_used() 打 read_set/write_set/read_set_internal
    · range 优化器填 quick_keys / const_key_parts / quick_rows
    · key_read / force_index / const_table / reginfo.join_tab
    │  语句结束 close_thread_tables()
    v
  阶段4  TABLE 归还（默认）  谁收：release_or_close_table() → release_table()
    · 解绑 mdl/pos_in_table_list；ha_reset()；in_use=nullptr
    · used_tables → free_tables → 挂入 m_unused_tables 尾（对象全部保留）
    [销毁路径] remove_table() → intern_close_table()
    │  最后一个 TABLE 消失 ⇒ release_table_share()
    v
  阶段5  SHARE 归零
    · ref_count-- == 0
    · 已过期/关机 → erase → Table_share_deleter → free_table_share() → destroy()
        （destroy 释放 ha_share/插件引用/view_object/tmp_table_def + mem_root.Clear()）
    · 未过期 → 挂 LRU 尾，等容量超限从链头淘汰
```

### 5. 速查表：职责分工

| 关注点 | `TABLE_SHARE`（模板 / 1 份） | `TABLE`（实例 / N 份） |
|---|---|---|
| 名字 | `table_cache_key` / `db` / `table_name` / `path` | `alias`（可由会话改写） |
| 列 | `Field** field`（模板，ptr→`default_values`） | `Field** field`（克隆，ptr→`record[0]`） |
| 行 | `reclength` / `null_bytes` / `default_values` | `record[0/1]` / `null_flags` / `write_row_record` |
| 键 | `key_info`（模板）/ `keys` / `primary_key` / `keys_for_keyread` | `key_info`（**副本**）+ `covering_keys`/`quick_keys`/`merge_keys`（**优化期产物**，`reset()` 清） |
| 位图 | `all_set`（全 1）/ `column_bitmap_size` | `def_*`（本体）+ `read_set`/`write_set`（**指针，可切换**） |
| 引擎 | `db_plugin` → `db_type()` / `db_low_byte_first` / `ha_share` | `file`（`handler*`）/ `db_stat` / `current_lock` |
| 归属 | `m_ref_count` / `m_version` / `next,prev` / `cache_element[]` | `in_use`（THD*）/ `pos_in_table_list` / `next,prev` / `cache_next,prev` |
| 生命周期 | `get_table_share()` 建 → `release_table_share()` → `free_table_share()`/`destroy()` | `open_table_from_share()` 建 → `release_table()` 归还 → `intern_close_table()` 销毁 |

### 6. ★ 三个最容易踩的坑

1. **`TABLE::key_info` 不是共享指针，是深拷贝**——`memcpy` 一份并重绑 `key_part->field` 到实例列，前缀索引还会 `new_field()` 出一个"长度被截断"的专用 `Field`。改一边不影响另一边。
2. **`in_use == nullptr` 才是"空闲"，不是 `file == nullptr`**——归还缓存的 TABLE **仍持有完整的 handler**，只是被 `ha_reset()` 复位。真正析构只有 `intern_close_table()` 一条路。
3. **DDL 不会立刻删除正在使用的 SHARE**——只 `clear_version()`（打过期标记）+ 收走空闲 TABLE；还在被持有的 TABLE 继续工作，直到最后一个引用释放时才销毁。这也是 `LOCK TABLES` 下 `close_thread_tables()` 不会重开表的原因。

---

## 表统计信息：★ 从引擎到优化器的搬运管道

> 源码坐标：`[T3] sql/handler.h`（`ha_statistics` / `info()` / `records_in_range()`）· `[T10] include/my_base.h`（`HA_STATUS_*`）· `[T7] storage/innobase/handler/ha_innodb.cc`（`info_low` / `innodb_rec_per_key` / `records_in_range` / `analyze`）· `[T5] sql/handler.cc`（`ha_analyze` / MRR 调用点）· `[T9] sql/key.h`（`KEY::rec_per_key[_float]`）· `[T6] sql/table.cc`（`fetch_number_of_rows`）· `[T2] sql/sql_base.cc`（失效淘汰）· `[T13] sql/range_optimizer/index_range_scan_plan.cc`（`SKIP_RECORDS_IN_RANGE`）

> **分工导航**：本节讲**统计信息从引擎到优化器的搬运管道**（`ha_statistics` / `info()` / `records_in_range` / rec_per_key / 失效刷新）。**成本模型把 `stats.records` 折算成 cost 的公式**、**直方图**、**`eq_range_index_dive_limit` 调参实践**属「优化器成本模型」，见 [`query/07_optimize/01_cost_model.md`](query/07_optimize/01_cost_model.md)。

### 1. `ha_statistics`：引擎侧统计的落点

★ 8.0 里统计**不在 `TABLE` 上，也不在 `TABLE_SHARE` 上**，而在 handler 里（每个 `TABLE` 有自己的 `handler`，所以是"**每实例**"而非"每 share"——这点决定了后文 B.5 的失效逻辑）：

```cpp
// [T3] sql/handler.h
class ha_statistics {
 public:
  ulonglong data_file_length;     /* Length off data file */
  ulonglong max_data_file_length; /* Length off data file */
  ulonglong index_file_length;
  ulonglong max_index_file_length;
  ulonglong delete_length; /* Free bytes */
  ulonglong auto_increment_value;
  /*
    The number of records in the table.
      0    - means the table has exactly 0 rows
    other  - if (table_flags() & HA_STATS_RECORDS_IS_EXACT)
               the value is the exact number of records in the table
             else
               it is an estimate
  */
  ha_rows records;
  ha_rows deleted;       /* Deleted records */
  ulong mean_rec_length; /* physical reclength */
  /* TODO: create_time should be retrieved from the new DD. Remove this. */
  time_t create_time; /* When table was created */
  ulong check_time;
  ulong update_time;
  uint block_size; /* index block size */

  /*
    number of buffer bytes that native mrr implementation needs,
  */
  uint mrr_length_per_rec;

  /**
    Estimate for how much of the table that is available in a memory
    buffer. Valid range is [0..1]. If it has the special value
    IN_MEMORY_ESTIMATE_UNKNOWN (defined in structs.h), it means that
    the storage engine has not supplied any value for it.
  */
  double table_in_mem_estimate;

  ha_statistics()
      : data_file_length(0), max_data_file_length(0), index_file_length(0),
        delete_length(0), auto_increment_value(0), records(0), deleted(0),
        mean_rec_length(0), create_time(0), check_time(0), update_time(0),
        block_size(0), table_in_mem_estimate(IN_MEMORY_ESTIMATE_UNKNOWN) {}
};
```

```cpp
// [T3] sql/handler.h —— class handler 内
  /** Pointer to duplicate row */
  uchar *dup_ref;

  ha_statistics stats;      // ← 官方读法：table->file->stats.records
```

`TABLE` 上另有一组**优化器自己算的**缓存字段，不要和 `stats` 混：

```cpp
// [T1] sql/table.h
  /*
    For each key that has quick_keys.is_set(key) == true: estimate of #records
    and max #key parts that range access would use.
    —— 每个"可用 range 访问"的索引，range 估算出的行数
  */
  ha_rows quick_rows[MAX_KEY]{0};
  /* Bitmaps of key parts that =const for the entire join. */
  key_part_map const_key_parts[MAX_KEY]{0};
  uint quick_key_parts[MAX_KEY]{0};
  uint quick_n_ranges[MAX_KEY]{0};
```

以及"这份 TABLE 上的统计已过期"标记（保护锁是 Table Cache 分区锁，**不是** `LOCK_open`）：

```cpp
// [T1] sql/table.h
  /**
    This TABLE object is invalid and cannot be reused as it has outdated
    rec_per_key and handler stats.

    @note This member is protected from concurrent access to it by lock of
          Table Cache's partition to which this TABLE object belongs,
  */
  bool m_invalid_stats{false};

  void invalidate_dict();
  void invalidate_stats();
  inline bool has_invalid_stats() {
    assert(assert_invalid_stats_is_locked(this));
    return m_invalid_stats;
  }
```

### 2. `handler::info()` 与 `HA_STATUS_*`

`info()` 是**唯一**的"服务器向引擎批量要统计"的入口，`flag` 决定要哪一段：

```cpp
// [T3] sql/handler.h
  /**
    General method to gather info from handler

    ::info() is used to return information to the optimizer.
    SHOW also makes use of this data Another note, if your handler
    doesn't proved exact record count, you will probably want to
    have the following in your code:
    if (records < 2)
      records = 2;
    The reason is that the server will optimize for cases of only a single
    record. If in a table scan you don't know the number of records
    it will probably be better to set records to two so you can return
    as many records as you need
    ...
      data_file_length
      index_file_length
      delete_length
      check_time
    Take a look at the public variables in handler.h for more information.
    See also my_base.h for a full description.

    @param   flag          Specifies what info is requested
  */
  virtual int info(uint flag) = 0;
```

标志位（`[T10] include/my_base.h`，原注释保留）：

```cpp
/* this one is not used */
#define HA_STATUS_POS 1
/*
  assuming the table keeps shared actual copy of the 'info' and
  local, possibly outdated copy, the following flag means that
  it should not try to get the actual data (locking the shared structure)
  slightly outdated version will suffice
  —— 不要去抢引擎内部锁，用稍微过期的副本就行（优化器热路径常用）
*/
#define HA_STATUS_NO_LOCK 2
/* update the time of the last modification (in handler::update_time) */
#define HA_STATUS_TIME 4
/*
  Recalculate loads of constant variables. MyISAM also sets things
  directly on the table share object.
  ... sortkey, ref_length, block_size, data_file_name, index_file_name.
  handler::table->s->keys_in_use, keys_for_keyread, rec_per_key
  —— "常量"部分：block_size / mean_rec_length / rec_per_key 等
*/
#define HA_STATUS_CONST 8
/*
  update the 'variable' part of the info:
  handler::records, deleted, data_file_length, index_file_length,
  check_time, mean_rec_length
  —— "变量"部分：会随 DML 变的（行数、文件大小）
*/
#define HA_STATUS_VARIABLE 16
/*
  This flag is used to get index number of the unique index that
  reported duplicate key.
  update handler::errkey and handler::dupp_ref
*/
#define HA_STATUS_ERRKEY 32
/*
  update handler::auto_increment_value
*/
#define HA_STATUS_AUTO 64
/*
  Get also delete_length when HA_STATUS_VARIABLE is called. It's ok to set it
  also when only HA_STATUS_VARIABLE but it won't be used.
*/
#define HA_STATUS_VARIABLE_EXTRA 128
```

InnoDB 实现（`info_low()`，两段分别对应 VARIABLE 与 CONST）：

```cpp
// [T7] storage/innobase/handler/ha_innodb.cc
int ha_innobase::info_low(uint flag, bool is_analyze) {
  ...
  if (flag & HA_STATUS_TIME) {
    if (is_analyze || innobase_stats_on_metadata) {
      dict_stats_upd_option_t opt;
      dberr_t ret;
      m_prebuilt->trx->op_info = "updating table statistics";

      if (dict_stats_is_persistent_enabled(ib_table)) {
        if (is_analyze) {
          opt = DICT_STATS_RECALC_PERSISTENT;         // ANALYZE：重算并落盘
        } else {
          /* This is e.g. 'SHOW INDEXES', fetch
          the persistent stats from disk. */
          opt = DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY; // SHOW：不在内存就从磁盘读
        }
      } else {
        opt = DICT_STATS_RECALC_TRANSIENT;            // 非持久化：现算（采样）
      }
      ret = dict_stats_update(ib_table, opt);
      if (ret != DB_SUCCESS) { ... return HA_ERR_GENERIC; }
    }
    stats.update_time = (ulong)std::chrono::system_clock::to_time_t(
        ib_table->update_time.load());
  }

  if (flag & HA_STATUS_VARIABLE) {
    if (!(flag & HA_STATUS_NO_LOCK)) {
      dict_table_stats_lock(ib_table, RW_S_LATCH);     // ★ HA_STATUS_NO_LOCK ⇒ 不抢这把 latch
    }
    n_rows = ib_table->stat_n_rows;                    // ← 行数来自 dict_table_t 的统计
    stat_clustered_index_size = ib_table->stat_clustered_index_size;
    stat_sum_of_other_index_sizes = ib_table->stat_sum_of_other_index_sizes;
    if (!(flag & HA_STATUS_NO_LOCK)) {
      dict_table_stats_unlock(ib_table, RW_S_LATCH);
    }
    ...
    /*
      ... Since SHOW TABLE STATUS seems to call this function with the
      HA_STATUS_TIME flag set, while the left join optimizer does not
      set that flag, we add one to a zero value if the flag is not
      set. That way SHOW TABLE STATUS will show the best estimate,
      while the optimizer never sees the table empty.
      —— ★ 优化器永远不能看到"0 行"，否则会生成"空集"级别的荒谬计划
    */
    if (n_rows == 0 && !(flag & HA_STATUS_TIME) &&
        table_share->table_category != TABLE_CATEGORY_TEMPORARY) {
      n_rows++;
    }
    stats.records = (ha_rows)n_rows;
    stats.deleted = 0;
    calculate_index_size_stats(ib_table, n_rows, stat_clustered_index_size,
                               stat_sum_of_other_index_sizes, &stats);
```

`HA_STATUS_CONST` 段负责 rec_per_key（见第 4 节）：

```cpp
// [T7] storage/innobase/handler/ha_innodb.cc
    if (index == pk) {
      stats.table_in_mem_estimate = pct_cached;      // 聚簇索引缓存比例即"表在内存里的比例"
    }

    if (flag & HA_STATUS_CONST) {
      if (!key->supports_records_per_key()) continue;
      for (j = 0; j < key->actual_key_parts; j++) {
        if ((key->flags & HA_FULLTEXT) || (key->flags & HA_SPATIAL)) {
          /* The record per key does not apply to FTS or Spatial indexes. */
          key->set_records_per_key(j, 1.0f);
          continue;
        }
        ...
        /* innodb_rec_per_key() will use
        index->stat_n_diff_key_vals[] and the value we
        pass index->table->stat_n_rows. Both are
        calculated by ANALYZE and by the background
        stats gathering thread ... */
        const rec_per_key_t rec_per_key =
            innodb_rec_per_key(index, (ulint)j, index->table->stat_n_rows);
        key->set_records_per_key(j, rec_per_key);
        /* The code below is legacy ... */
        ulong rec_per_key_int =
            static_cast<ulong>(innodb_rec_per_key(index, (ulint)j, stats.records));
        /* ★ Since MySQL seems to favor table scans
        too much over index searches, we pretend
        index selectivity is 2 times better than
        our estimate: */
        rec_per_key_int = rec_per_key_int / 2;
        if (rec_per_key_int == 0) rec_per_key_int = 1;
        key->rec_per_key[j] = rec_per_key_int;
      }
    }
```

### 3. ★ `records_in_range()`：优化器问引擎"这个范围大概多少行"

基类默认实现给常量（不实现也能跑，只是估算很糙）：

```cpp
// [T3] sql/handler.h
  /**
    Find number of records in a range.

    Given a starting key, and an ending key estimate the number of rows that
    will exist between the two. max_key may be empty which in case determine
    if start_key matches any rows. Used by optimizer to calculate cost of
    using a particular index.

    @param inx      Index number
    @param min_key  Start of range
    @param max_key  End of range

    @return Number of rows in range.
  */
  virtual ha_rows records_in_range(uint inx [[maybe_unused]],
                                   key_range *min_key [[maybe_unused]],
                                   key_range *max_key [[maybe_unused]]) {
    return (ha_rows)10;
  }
```

真实调用点（`[T5] sql/handler.cc`，MRR 的 const 估算路径；range 优化器正是通过这条路径拿每个 range 的行数）：

```cpp
// [T5] sql/handler.cc —— handler::multi_range_read_info_const()
    /*
      Get the number of rows in the range. This is done by calling
      records_in_range() unless:

        1) The index is unique.
           There cannot be more than one matching row, so 1 is
           assumed. ...
        2) SKIP_RECORDS_IN_RANGE will be set when skip_records_in_range or
           use_index_statistics are true.
           ...
      Note: With SKIP_RECORDS_IN_RANGE, use Index statistics if:
            a) Index statistics is available.
            b) The range is an equality range but the index is either not
               unique or all of the keyparts are not used.
    */
    int keyparts_used = 0;
    if ((range.range_flag & UNIQUE_RANGE) &&   // 1)
        !(range.range_flag & NULL_RANGE))
      rows = 1; /* there can be at most one row */
    else if (range.range_flag & SKIP_RECORDS_IN_RANGE &&  // 2)
             !(range.range_flag & NULL_RANGE)) {
      if ((range.range_flag & EQ_RANGE) &&
          (keyparts_used = my_count_bits(range.start_key.keypart_map)) &&
          table->key_info[keyno].has_records_per_key(keyparts_used - 1)) {
        // 用 rec_per_key 统计代替 index dive
        rows = static_cast<ha_rows>(
            table->key_info[keyno].records_per_key(keyparts_used - 1));
      } else {
        rows = 1;
      }
    } else {
      assert(min_endp || max_endp);
      rows = table->pos_in_table_list->is_derived_unfinished_materialization()
                 ? HA_POS_ERROR
                 : this->records_in_range(keyno, min_endp, max_endp);   // ← 真正的 index dive
      if (rows == HA_POS_ERROR) {
        /* Can't scan one range => can't do MRR scan at all */
        return HA_POS_ERROR;
      }
    }
    total_rows += rows;
```

`SKIP_RECORDS_IN_RANGE` 在 range 树构造时打上（等值 range + 开了 `use_index_statistics`）：

```cpp
// [T13] sql/range_optimizer/index_range_scan_plan.cc
    if (is_eq_range_pred) {
      range->range_flag = EQ_RANGE;
      /*
        Use statistics instead of index dives for estimates of rows in
        this range if the user requested it
      */
      if (param->use_index_statistics)
        range->range_flag |= SKIP_RECORDS_IN_RANGE;
      /*
        An equality range is a unique range (0 or 1 rows in the range)
        if the index is unique (1) and all keyparts are used (2). ...
      */
      if (cur_key_info->flags & HA_NOSAME &&                    // 1)
          (uint)key_tree->part + 1 ==
              cur_key_info->user_defined_key_parts)             // 2)
        range->range_flag |= UNIQUE_RANGE | (cur->min_key_flag & NULL_RANGE);
    }
    ...
  if (seq->skip_records_in_range) range->range_flag |= SKIP_RECORDS_IN_RANGE;
```

InnoDB 实现：把 MySQL 的 `key_range` 翻译成 InnoDB 的 `dtuple_t`，再走 `btr_estimate_n_rows_in_range()`（沿 B-tree 做**有限深度的 dive**，不是真扫）：

```cpp
// [T7] storage/innobase/handler/ha_innodb.cc
ha_rows ha_innobase::records_in_range(
    uint keynr,         /*!< in: index number */
    key_range *min_key, /*!< in: start key value of the range, may also be 0 */
    key_range *max_key) /*!< in: range end key val, may also be 0 */
{
  KEY *key;
  dict_index_t *index;
  dtuple_t *range_start;
  dtuple_t *range_end;
  int64_t n_rows;
  page_cur_mode_t mode1;
  page_cur_mode_t mode2;
  mem_heap_t *heap;

  m_prebuilt->trx->op_info = "estimating records in index range";
  TrxInInnoDB trx_in_innodb(m_prebuilt->trx);

  active_index = keynr;
  key = table->key_info + active_index;
  index = innobase_get_index(keynr);

  if (dict_table_is_discarded(m_prebuilt->table)) { n_rows = HA_POS_ERROR; goto func_exit; }
  if (!index)                      { n_rows = HA_POS_ERROR; goto func_exit; }
  if (index->is_corrupted())       { n_rows = HA_ERR_INDEX_CORRUPT; goto func_exit; }
  if (!index->is_usable(m_prebuilt->trx)) { n_rows = HA_ERR_TABLE_DEF_CHANGED; goto func_exit; }

  heap = mem_heap_create(
      2 * (key->actual_key_parts * sizeof(dfield_t) + sizeof(dtuple_t)),
      UT_LOCATION_HERE);

  range_start = dtuple_create(heap, key->actual_key_parts);
  dict_index_copy_types(range_start, index, key->actual_key_parts);
  range_end = dtuple_create(heap, key->actual_key_parts);
  dict_index_copy_types(range_end, index, key->actual_key_parts);

  // MySQL key buffer → InnoDB tuple（两个 tuple 复用 m_prebuilt 的 srch_key_val1/2）
  row_sel_convert_mysql_key_to_innobase(
      range_start, m_prebuilt->srch_key_val1, m_prebuilt->srch_key_val_len,
      index, (byte *)(min_key ? min_key->key : (const uchar *)nullptr),
      (ulint)(min_key ? min_key->length : 0));
  row_sel_convert_mysql_key_to_innobase(
      range_end, m_prebuilt->srch_key_val2, m_prebuilt->srch_key_val_len, index,
      (byte *)(max_key ? max_key->key : (const uchar *)nullptr),
      (ulint)(max_key ? max_key->length : 0));

  mode1 = convert_search_mode_to_innobase(min_key ? min_key->flag : HA_READ_KEY_EXACT);
  mode2 = convert_search_mode_to_innobase(max_key ? max_key->flag : HA_READ_KEY_EXACT);

  if (mode1 != PAGE_CUR_UNSUPP && mode2 != PAGE_CUR_UNSUPP) {
    if (dict_index_is_spatial(index)) {
      /*Only min_key used in spatial index. */
      n_rows = rtr_estimate_n_rows_in_range(index, range_start, mode1);
    } else {
      n_rows = btr_estimate_n_rows_in_range(index, range_start, mode1,
                                            range_end, mode2);   // ← B-tree 估算
    }
  } else {
    n_rows = HA_POS_ERROR;
  }
  mem_heap_free(heap);

func_exit:
  m_prebuilt->trx->op_info = (char *)"";
  /*
    The MySQL optimizer seems to believe an estimate of 0 rows is
    always accurate and may return the result 'Empty set' based on that.
    ... Add 1 to the value to make sure MySQL does not make the assumption!
  */
  if (n_rows == 0) {
    n_rows = 1;
  }
  return (ha_rows)n_rows;
}
```

### 4. rec_per_key 与优化器拉取入口

"每个键值平均多少行"不在 `ha_statistics` 里，而在 `KEY` 上（8.0 同时保留整型 `rec_per_key` 与浮点 `rec_per_key_float`，**后者优先**）：

```cpp
// [T9] sql/key.h
/**
  Data type for records per key estimates that are stored in the
  KEY::rec_per_key_float[] array.
*/
typedef float rec_per_key_t;

/**
  If an entry for a key part in KEY::rec_per_key_float[] has this value,
  then the storage engine has not provided a value for it and the rec_per_key
  value for this key part is unknown.
*/
#define REC_PER_KEY_UNKNOWN -1.0f

  /**
    Array of AVG(number of records with the same field value) for 1st ... Nth
    key part. 0 means 'not known'. For internally created temporary tables,
    this member can be nullptr.
  */
  ulong *rec_per_key{nullptr};

  /**
    ... This is the same information as stored in the above
    rec_per_key array but using float values instead of integer
    values. If the storage engine has supplied values in this array,
    these will be used. Otherwise the value in rec_per_key will be
    used.
  */
  rec_per_key_t *rec_per_key_float{nullptr};

  bool has_records_per_key(uint key_part_no) const {
    return ((rec_per_key_float &&
             rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN) ||
            (rec_per_key && rec_per_key[key_part_no] != 0));
  }

  rec_per_key_t records_per_key(uint key_part_no) const {
    /*
      If the storage engine has provided rec per key estimates as float
      then use this. If not, use the integer version.
    */
    if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)
      return rec_per_key_float[key_part_no];
    return (rec_per_key[key_part_no] != 0)
               ? static_cast<rec_per_key_t>(rec_per_key[key_part_no])
               : REC_PER_KEY_UNKNOWN;
  }
```

InnoDB 侧的计算（`records / 不同键值数`，可选排除 NULL）：

```cpp
// [T7] storage/innobase/handler/ha_innodb.cc
rec_per_key_t innodb_rec_per_key(const dict_index_t *index, ulint i,
                                 ha_rows records) {
  rec_per_key_t rec_per_key;
  uint64_t n_diff;

  ut_a(index->table->stat_initialized);
  ut_ad(i < dict_index_get_n_unique(index));
  ut_ad(!dict_index_is_spatial(index));

  if (records == 0) {
    /* "Records per key" is meaningless for empty tables.
    Return 1.0 because that is most convenient to the Optimizer. */
    return (1.0);
  }

  n_diff = index->stat_n_diff_key_vals[i];        // 该前缀的不同值个数（来自 ANALYZE 采样）

  if (n_diff == 0) {
    rec_per_key = static_cast<rec_per_key_t>(records);
  } else if (srv_innodb_stats_method == SRV_STATS_NULLS_IGNORED) {
    uint64_t n_null;
    uint64_t n_non_null = index->stat_n_non_null_key_vals[i];
    ...
    /* If the number of NULL values is the same as or
    large than that of the distinct values, we could
    consider that the table consists mostly of NULL value.
    Set rec_per_key to 1. */
    if (n_diff <= n_null) {
      rec_per_key = 1.0;
    } else {
      /* Need to exclude rows with NULL values from
      rec_per_key calculation */
      rec_per_key =
          static_cast<rec_per_key_t>(records - n_null) / (n_diff - n_null);
    }
  } else {
    rec_per_key = static_cast<rec_per_key_t>(records) / n_diff;   // ★ 核心公式
  }

  if (rec_per_key < 1.0) {
    /* Values below 1.0 are meaningless and must be due to the
    stats being imprecise. */
    rec_per_key = 1.0;
  }
  return (rec_per_key);
}
```

★ **拉取入口**：优化器不直接调 `info()`，而是走 `Table_ref::fetch_number_of_rows()`——它把"不同形态的表"统一成一个 `stats.records`：

```cpp
// [T6] sql/table.cc
/**
  @brief Retrieve number of rows in the table

  @details Retrieve number of rows in the table referred by this Table_ref and
  store it in the table's stats.records variable. If this Table_ref refers
  to a materialized derived table/view, then the estimated number of rows of
  the derived table/view is used instead.

  @param fallback_estimate A fallback row estimate to use if the storage engine
  doesn't provide one for us. The old optimizer uses
  PLACEHOLDER_TABLE_ROW_ESTIMATE, which is 2. The hypergraph optimizer uses a
  more pessimistic estimate of 1000 rows.

  @return 0          ok
  @return non zero   error
*/
int Table_ref::fetch_number_of_rows(ha_rows fallback_estimate) {
  if (is_table_function()) {
    // FIXME: open question - there's no estimate for table function.
    table->file->stats.records = fallback_estimate;
  } else if (uses_materialization()) {
    table->file->stats.records = derived->query_result()->estimated_rowcount;  // 物化派生表
  } else if (is_recursive_reference()) {
    /*
      Use the estimated row count of all query blocks before this one, as the
      table will contain, at least, the rows produced by those blocks.
    */
    table->file->stats.records = std::max(
        query_block->master_query_expression()->query_result()->estimated_rowcount,
        fallback_estimate);     // Recursive reference is never a const table
  } else {
    int error = table->file->info(HA_STATUS_VARIABLE | HA_STATUS_NO_LOCK);   // ← 问引擎
    if (error) return error;
    // Some information schema tables have zero as estimate, which can lead
    // to completely wild plans. Add a placeholder to make sure we have
    // _something_ to work with.
    if (schema_table != nullptr && schema_table->fill_table != nullptr &&
        table->file->stats.records == 0) {
      table->file->stats.records = fallback_estimate;
    }
  }
  return 0;
}
```

下游消费者（真实读点，印证"基数估算的唯一真源是 `table->file->stats.records`"）：

```cpp
// sql/sql_planner.cc
      double rowcount = rows2double(tab->table()->file->stats.records);
// sql/sql_select.cc
  ha_rows table_records = table->file->stats.records;
  ...
  table->quick_condition_rows = table->file->stats.records;
// sql/join_optimizer/estimate_selectivity.cc
              1.0 / table->file->stats.records);
// sql/item_subselect.cc
      has_zero_rows = (table->file->stats.records == 0);
```

⇒ `quick_rows[]` 由 range 优化器回填（每个候选索引的 range 估算结果），随后被 `TABLE::quick_condition_rows` 与成本模型消费。

**persistent vs transient（分工导航）**：
- **persistent**：`innodb_stats_persistent=ON`（默认）时，`ANALYZE TABLE` 采样结果写入 `mysql.innodb_table_stats` / `mysql.innodb_index_stats`（DD 表），重启不丢；后台线程按 `innodb_stats_auto_recalc` + 变更比例阈值自动重算；`SHOW INDEX` 等场景用 `DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY` 从磁盘读。
- **transient**：`innodb_stats_persistent=OFF` 时每次 `info_low()` 现算（`DICT_STATS_RECALC_TRANSIENT`），重启即丢。
- 两者的采样参数、后台重算线程、`mysql.innodb_*_stats` 表结构与写入路径属 **InnoDB 统计子系统**；本节只标注它们在 `info_low()` 里的**分叉点**（`dict_stats_is_persistent_enabled()` 分支）。

### 5. 统计何时失效 / 刷新

`ANALYZE TABLE` 的服务器侧路径极短：

```cpp
// [T3] sql/handler.h
  virtual int analyze(THD *, HA_CHECK_OPT *) {
    return HA_ADMIN_NOT_IMPLEMENTED;
  }
// [T5] sql/handler.cc
int handler::ha_analyze(THD *thd, HA_CHECK_OPT *check_opt) {
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);
  mark_trx_read_write();
  return analyze(thd, check_opt);
}
```

InnoDB 直接复用 `info_low()` 并带 `is_analyze=true`：

```cpp
// [T7] storage/innobase/handler/ha_innodb.cc
int ha_innobase::analyze(THD *, HA_CHECK_OPT *) {
  /* Simply call info_low() with all the flags
  and request recalculation of the statistics */
  int ret = info_low(HA_STATUS_TIME | HA_STATUS_CONST | HA_STATUS_VARIABLE,
                     true /* this is ANALYZE */);
  if (ret != 0) { ... }
```

★ **缓存侧失效**：统计存在 handler（**每 `TABLE` 实例**）里，所以 `ANALYZE`/`FLUSH TABLE` 会让**表缓存里的旧 `TABLE` 副本**带着过期统计。8.0 用 `TDC_RT_MARK_FOR_REOPEN` 打标记，之后释放时**直接丢弃而不是归还缓存**：

```cpp
// [T2] sql/sql_base.cc
static void release_or_close_table(THD *thd, TABLE *table) {
  Table_cache *tc = table_cache_manager.get_cache(thd);
  tc->lock();
  if (table->s->has_old_version() || table->has_invalid_dict() ||
      table->has_invalid_stats() || table_def_shutdown_in_progress) {
    tc->remove_table(table);
    mysql_mutex_lock(&LOCK_open);
    intern_close_table(table);       // ★ 统计过期 ⇒ 真关，不回缓存
    mysql_mutex_unlock(&LOCK_open);
  } else
    tc->release_table(thd, table);   // 否则归还，下次复用（含 stats）
}
```

打标记的那一侧：

```cpp
// sql/table_cache.cc
      if (remove_type == TDC_RT_MARK_FOR_REOPEN) {
        Table_cache_element::TABLE_list::Iterator it2(cache_el[i]->used_tables);
        while ((table = it2++)) {
          table->invalidate_stats();
        }
      }
```

### 6. ASCII 图：统计信息流向

```
                ┌─────────────────────────────────────────────────────────┐
                │  持久化 / 采样层（InnoDB）                                │
                │  mysql.innodb_table_stats / innodb_index_stats (DD 表)    │
                │  dict_stats_update(RECALC_PERSISTENT / TRANSIENT / EMPTY) │
                │  后台 recalc 线程（auto_recalc + 变更比例阈值）             │
                │  DML 对 stat_n_rows 的增量调整（非 MVCC、回滚不撤销）       │
                └───────────────┬─────────────────────────────────────────┘
                                │ dict_table_t: stat_n_rows,
                                │  stat_clustered_index_size,
                                │  stat_sum_of_other_index_sizes,
                                │  stat_n_diff_key_vals[], stat_n_non_null_key_vals[]
                                ▼
        ┌────────────────────────────────────────────────────────┐
        │ handler::info(flag)   ← 服务器唯一的批量拉取口            │
        │   HA_STATUS_VARIABLE → stats.records / deleted /        │
        │                        data_file_length / index_file_length │
        │   HA_STATUS_CONST    → KEY::rec_per_key_float[]          │
        │                        block_size / mean_rec_length      │
        │   HA_STATUS_TIME     → stats.update_time（+ 可能触发重算） │
        │   HA_STATUS_NO_LOCK  → 不抢 dict_table_stats_lock(S-latch)│
        │   HA_STATUS_AUTO     → stats.auto_increment_value        │
        └───────────┬──────────────────────────┬─────────────────┘
                    │                          │
                    ▼                          ▼
        handler::stats (ha_statistics)     KEY::rec_per_key[_float]
        table->file->stats.records         table->key_info[k].records_per_key(p)
                    │                          │
                    │  Table_ref::fetch_number_of_rows()
                    │   ├─ 物化派生表 → derived->query_result()->estimated_rowcount
                    │   ├─ 递归 CTE  → 前序 block 的 estimated_rowcount
                    │   ├─ table function → fallback_estimate（引擎无数据）
                    │   └─ 基表 → info(HA_STATUS_VARIABLE | HA_STATUS_NO_LOCK)
                    ▼                          ▼
        ┌────────────────────────────────────────────────────────┐
        │ 优化器                                                   │
        │  · 表扫描基数：stats.records（+ I_S 兜底 fallback）        │
        │  · range 估算：records_in_range(keyno,min,max)  ← index dive│
        │               或 SKIP_RECORDS_IN_RANGE 时用 records_per_key │
        │               → 回填 TABLE::quick_rows[] / quick_condition_rows│
        │  · 等值/连接选择率：rec_per_key、直方图（见成本模型文档）     │
        └────────────────────────────────────────────────────────┘

  刷新 / 失效闭环：
    ANALYZE TABLE → handler::ha_analyze() → ha_innobase::analyze()
                  → info_low(TIME|CONST|VARIABLE, is_analyze=true)
                  → DICT_STATS_RECALC_PERSISTENT（写回 DD 表）
    FLUSH TABLE / TDC_RT_MARK_FOR_REOPEN → TABLE::invalidate_stats()
                  → release_or_close_table() 见 m_invalid_stats ⇒ intern_close_table()
                    （不归还 Table Cache，下次开表重新 info()）
```

---

## 表类型

### 1. DD 层的权威枚举：`dd::enum_table_type`

MySQL 8.0 的"表"在数据字典里只有四种类型：

```cpp
// [T12] sql/dd/types/abstract_table.h
enum class enum_table_type {
  INVALID_TABLE,
  BASE_TABLE,
  USER_VIEW,
  SYSTEM_VIEW
};
```

`Table_ref::required_type` 就是拿它做**开表时的期望校验**（"这个地方只许是真表 / 不许是视图"）：

```cpp
// [T1] sql/table.h
  /**
     Specifies which kind of table should be open for this element
     of table list.
  */
  enum_open_type open_type{OT_TEMPORARY_OR_BASE};
  ...
  dd::enum_table_type required_type{};
```

```cpp
// [T1] sql/table.h —— 先按"临时 or 基表"分，再按 DD 类型校验
/**
   Type of table which can be open for an element of table list.
*/
enum enum_open_type {
  OT_TEMPORARY_OR_BASE = 0,   // 默认：临时表优先，没有再开基表
  OT_TEMPORARY_ONLY,          // 只开临时表（CREATE TEMPORARY TABLE 场景）
  OT_BASE_ONLY                // 只开基表（DDL / TRUNCATE / FLUSH ... 屏蔽同名临时表）
};
```

典型使用（"这张表必须是真表，视图一律拒绝"）：

```cpp
// [T2] sql/sql_base.cc —— open_ltable()
  /* open_ltable can be used only for BASIC TABLEs */
  table_list->required_type = dd::enum_table_type::BASE_TABLE;
// [T2] sql/sql_truncate.cc
  /* Our metadata lock guarantees that no transaction is reading
  or writing into the table. Yet, to open a write cursor we need
  a thr_lock lock. Allow to open base tables only. */
  table_ref->required_type = dd::enum_table_type::BASE_TABLE;
// sql/sql_trigger.cc
  tables->required_type = dd::enum_table_type::BASE_TABLE;
  /* Also prevent DROP TRIGGER from opening temporary table which might
  shadow the subject table ... */
  tables->open_type = OT_BASE_ONLY;
// sql/sql_yacc.yy（FLUSH TABLES ... WITH READ LOCK）
              tables->mdl_request.set_type(MDL_SHARED_NO_WRITE);
              /* Don't try to flush views. */
              tables->required_type= dd::enum_table_type::BASE_TABLE;
              tables->open_type= OT_BASE_ONLY;      /* Ignore temporary tables. */
```

### 2. `Table_ref` 侧的类型判定（真实代码）

DD 只认 4 种，但 SQL 层要区分的形态更多——派生表/CTE/table function 都属于"没有 DD 实体"的内部表，靠 `Table_ref` 自己的指针成员判定：

```cpp
// [T1] sql/table.h —— 全部为 inline，判据就是"某个指针是否非空"
  bool is_view()            const { return view != nullptr; }
  bool is_derived()         const { return derived != nullptr && view == nullptr; }
  bool is_view_or_derived() const { return derived != nullptr; }
  bool is_table_function()  const { return table_function != nullptr; }
  bool is_recursive_reference() const { return m_is_recursive_reference; }
  bool is_base_table() const {
    return !(is_view_or_derived() || is_table_function() ||
             is_recursive_reference());
  }
  bool is_internal() const {
    return is_derived() || is_recursive_reference() || is_table_function();
  }
  bool is_placeholder() const {
    return is_view_or_derived() || is_recursive_reference() ||
           is_table_function() || schema_table || table == nullptr;
  }
  bool is_merged() const { return effective_algorithm == VIEW_ALGORITHM_MERGE; }
  bool uses_materialization() const {
    return effective_algorithm == VIEW_ALGORITHM_TEMPTABLE;
  }
```

### 3. 临时表

临时表没有"独立的 `Table_ref` 形态"——它是**基表 + `TABLE_SHARE::tmp_table != NO_TMP_TABLE`**：

```cpp
// [T1] sql/table.h
enum tmp_table_type {
  NO_TMP_TABLE,                  // 普通持久表
  NON_TRANSACTIONAL_TMP_TABLE,   // CREATE TEMPORARY TABLE（MyISAM 等）
  TRANSACTIONAL_TMP_TABLE,       // CREATE TEMPORARY TABLE（InnoDB）
  INTERNAL_TMP_TABLE,            // 优化器内部临时表（物化派生表/GROUP BY 等）
  SYSTEM_TMP_TABLE
};
```

```cpp
// [T1] sql/table.h
/**
  Check if a Table_ref instance represents a pre-opened temporary table.
*/
inline bool is_temporary_table(const Table_ref *tl) {
  if (tl->is_view() || tl->schema_table) return false;   // ★ 视图和 I_S 表一律不算临时表
  if (!tl->table) return false;
  /*
    NOTE: 'table->s' might be NULL for specially constructed TABLE
    instances. See SHOW TRIGGERS for example.
  */
  if (!tl->table->s) return false;
  return tl->table->s->tmp_table != NO_TMP_TABLE;   // ← 唯一判据
}
```

开表时的临时表旁路（临时表不走表缓存、不走 DD 查找）：

```cpp
// [T2] sql/sql_base.cc —— open_tables()
  if (tables->open_type == OT_TEMPORARY_ONLY || is_temporary_table(tables)) {
    // Already "open", no action required
  } else if (tables->prelocking_placeholder) { ... }
```

创建临时表时，`Table_ref` 甚至把 `db` 留空、`table_name` 直接用 alias（临时表名是一长串全限定路径）：

```cpp
// [T1] sql/table.h
  /**
    This constructor can be used when a Table_ref is needed for an
    existing temporary table. These typically have very long table names, since
    it is a fully qualified path. For this reason, the table is set to the
    alias. The database name is left blank. The lock descriptor is set to
    TL_READ.
  */
  Table_ref(TABLE *table_arg, const char *alias_arg)
      : db(""), table_name(alias_arg), alias(alias_arg), m_map(1),
        table(table_arg), m_lock_descriptor{TL_READ}, db_length(0),
        table_name_length(strlen(alias_arg)) { ... }
```

### 4. schema table（I_S）与 system view

8.0 的 I_S 表分两类：**隐藏系统视图**（DD 上的 system view，走正常优化器/执行器）与**内存填充表**（`ST_SCHEMA_TABLE*` + `fill_table` 回调）：

```cpp
// [T1] sql/table.h
  ST_SCHEMA_TABLE *schema_table{nullptr};   /* Information_schema table */
  /* True when the view field translation table is used to convert
     schema table fields for backwards compatibility with SHOW command. */
  bool schema_table_reformed{false};
  bool schema_table_filled{false};
  // True, If this is a system view
  bool is_system_view{false};
  /*
    Set to 'true' if this is a DD table being opened in the context of a
    dictionary operation. Note that when 'false', this may still be a DD
    table when opened in a non-DD context, e.g. as part of an I_S view query.
  */
  bool is_dd_ctx_table{false};
```

`TABLE_SHARE::table_category` 决定了一些"特权行为"（跳过版本检查、跳过行锁等）：

```cpp
// [T1] sql/table.h
/**
  Category of table found in the table share.
*/
enum enum_table_category {
  TABLE_UNKNOWN_CATEGORY = 0,
  ...
  TABLE_CATEGORY_GTID = 8,
  /**
    A data dictionary table.
    Table's with this category will skip checking the
    TABLE_SHARE versions because these table structures
    are fixed upon server bootstrap.
  */
  TABLE_CATEGORY_DICTIONARY = 9,
  /**
    A ACL metadata table.
    For table in this category we will skip row locks when SQL statement
    reads them.
  */
  TABLE_CATEGORY_ACL_TABLE = 10
};
typedef enum enum_table_category TABLE_CATEGORY;
```

### 5. table function（JSON_TABLE）与 CTE

见上文 `Table_function` / `Table_function_json` 定义。生命周期要点：

- 解析期：`Table_ident::table_function` → `Table_ref::table_function`，`add_table_to_list()` 里 `table_func_count++` 并清空 `derived_key_list`；
- prepare 期：`Table_function::create()` 只**创建**不实例化结果表；列定义来自 `COLUMNS` 子句（`List<Json_table_column>`）；
- 执行期：结果是一张真实临时 `TABLE`（`Table_function::table`），行数无引擎统计 ⇒ 用 `fallback_estimate`；
- 始终 `is_placeholder()` 为真、`is_internal()` 为真（不可见于任何 schema、不做权限检查）。

**CTE**：CTE 引用在解析期**先伪装成派生表**（`Table_ident::sel` 非空 ⇒ `set_derived_query_expression()` 生效），随后被 CTE 机制接管。两种引用形态：

- **非递归引用**：`is_derived() == true`（是派生表），需要物化（**共享物化**，`Common_table_expr::tmp_tables` 里第一个引用真 `create_tmp_table()`，其余 `open_table_from_share()`）；
- **递归引用**：`is_recursive_reference() == true`，`is_derived()` 为 false——解析结束时它退化成"指向已物化临时表的一个引用"，而非一个待物化的派生表。

### 6. 形态总表与判别树

| 形态 | DD 实体 | `Table_ref` 判据 | `table`（`TABLE*`） | 统计来源 |
|---|---|---|---|---|
| base table | `BASE_TABLE` | `is_base_table()` 且 `!is_temporary_table()` | 开表后绑定，可走 Table Cache | `info(HA_STATUS_VARIABLE)` |
| temporary table | 无（会话私有） | `is_temporary_table()`（`s->tmp_table != NO_TMP_TABLE`） | 预开表，不入 Table Cache | 引擎（不走 persistent stats） |
| view | `USER_VIEW` | `is_view()`（`view != nullptr`） | merge ⇒ 始终 `nullptr`；物化 ⇒ 临时表 | 物化时用 `estimated_rowcount` |
| derived table | 无 | `is_derived()`（`derived && !view`） | 物化后指向临时表 | `derived->query_result()->estimated_rowcount` |
| CTE（非递归引用） | 无 | `is_derived() && common_table_expr()` | 共享物化的临时表（clone） | 同上 |
| CTE（递归引用） | 无 | `is_recursive_reference()` | 指向已物化临时表 | 前序 block 的 `estimated_rowcount` |
| schema table（I_S 内存表） | 无 | `schema_table != nullptr` | 伪 TABLE，运行时 `fill_table` 填 | 常为 0 ⇒ 用 `fallback_estimate` 兜底 |
| system view（I_S 视图） | `SYSTEM_VIEW` | `is_system_view == true` | 走视图路径 | 同 view |
| table function（JSON_TABLE） | 无 | `is_table_function()` | 结果临时表 | `fallback_estimate`（无估算） |
| nested join（非表） | 无 | `nested_join != nullptr` | 永远 `nullptr` | — |

```
                     Table_ref 形态判别树（自上而下，命中即止）

  nested_join != nullptr ?  ── yes ──► 【嵌套连接节点】不是表，db/table_name 为空串
        │ no
  view != nullptr ?         ── yes ──► 【视图】
        │                                 ├─ effective_algorithm == MERGE     → merge，table 恒 nullptr
        │ no                              └─ effective_algorithm == TEMPTABLE → 物化成临时表
  derived != nullptr ?      ── yes ──► 【派生表 / CTE 非递归引用】
        │                                 m_common_table_expr ? → CTE（共享物化）
        │ no
  m_is_recursive_reference ?── yes ──► 【递归 CTE 引用】指向已物化临时表
        │ no
  table_function != nullptr ?── yes ──► 【table function，JSON_TABLE】
        │ no
  schema_table != nullptr ? ── yes ──► 【I_S 内存填充表】
        │ no
  is_system_view ?          ── yes ──► 【I_S 系统视图】
        │ no
        └────────────────────────────► 【基表】
                                          ├─ s->tmp_table != NO_TMP_TABLE → 临时表
                                          └─ 否则 → 持久基表（DD: BASE_TABLE）

  正交的两个"约束位"（不是形态，是开表期望）：
    required_type ∈ {BASE_TABLE, USER_VIEW, SYSTEM_VIEW}   ← 与 DD 实体比对
    open_type     ∈ {TEMPORARY_OR_BASE, TEMPORARY_ONLY, BASE_ONLY}
```

### 7. 各形态的深入剖析在别篇（分工导航）

| 类型 | 说明 | 归宿 |
|---|---|---|
| 普通表 | 默认 | 本篇 |
| 临时表 | `CREATE TEMPORARY TABLE`；`NON_TRANSACTIONAL_TMP_TABLE` 不进表锁计数 | [`query/runtime/07_temptable.md`](query/runtime/07_temptable.md) |
| 内部临时表 / intrinsic table | 优化器自建；InnoDB intrinsic 表可关闭锁与 redo | 同上 |
| 分区表 | `ha_innopart`，每分区一个 `dict_table_t` | [`../feat/partitioning.md`](../feat/partitioning.md) |
| 视图 / 派生表（placeholder） | 走 `resolve_placeholder_tables`，可 merge 或物化 | [`query/07_optimize/20_view_resolution.md`](query/07_optimize/20_view_resolution.md) |
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
