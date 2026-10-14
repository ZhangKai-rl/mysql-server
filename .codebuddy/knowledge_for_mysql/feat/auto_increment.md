# 自增列（AUTO_INCREMENT）

> **定位**：自增从 SQL 解析到 InnoDB 落地的完整链路——server 层自增字段描述、handler 区间分配接口、InnoDB 的 AUTOINC 三种锁模式、计数器持久化、以及"空洞/重启回退/上限"这些常见疑问的源码答案。
> **边界**：自增列的元数据字段（`TABLE_SHARE` 四处定位信息、`TABLE` 运行时字段）见 [`../server/table.md`](../server/table.md)；AUTOINC 表锁在锁系统中的位置见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)；复制格式与 SBR 安全性见 [`../server/replication/binlog.md`](../server/replication/binlog.md)。

---

## 概述

### 是什么

自增不是"一个计数器加一"，而是**三层协作 + 一个持久化问题**：

| 层 | 职责 | 关键对象 |
|---|---|---|
| SQL 层 | 识别自增列、决定要不要生成值 | `TABLE_SHARE` 的自增四字段、`TABLE::autoinc_field_has_explicit_non_null_value` |
| handler 接口 | **批量**申请一段区间 | `ha_innobase::get_auto_increment(offset, increment, nb_desired_values, ...)` |
| InnoDB | 分配、加锁、持久化 | `dict_table_t::autoinc`、AUTOINC mutex/表锁、`dict_table_autoinc_log` |

★★ **自增的核心矛盾**：值必须**单调不重复**，但生成它的事务**可能回滚**。InnoDB 的答案是——**自增值一旦分配就永不回收**（回滚不归还），这直接解释了"自增列为什么有空洞"。

**⚠️ 三层里最容易搞混的两个 `next_number_field`**：

```cpp
// TABLE_SHARE（table.h:748，open 后固定）
Field **found_next_number_field{nullptr};   // ★ 类型是 Field**（指向 field 数组元素）
uint next_number_index{0};                  // 自增列所在索引编号
uint next_number_key_offset{0};             // 在该 KEY 中的字节偏移
uint next_number_keypart{0};                // 是索引第几列；==0 表示是首列

// TABLE（table.h:1501-1502，★ 每语句激活/去激活）
Field *next_number_field{nullptr};          /* Set if next_number is activated */
Field *found_next_number_field{nullptr};    /* Set on open */
```

> ★ **不存在 `TABLE_SHARE::next_number_field`**——`next_number_field` 只在 `TABLE` 上，且**非 nullptr 时 SE 的 `write_row()` 才去生成自增值**。激活点有三个：`mysql_insert()`（sql_insert.cc:530）、`Query_result_insert::start_execution()`（:2290）、LOAD DATA（sql_load.cc:554）；去激活在 `mysql_insert()` 末尾（:733）。

```cpp
  insert_table->next_number_field = insert_table->found_next_number_field;   // ① 激活
  ...
  insert_table->next_number_field = nullptr;                                 // ② 去激活
```

**自增列的唯一判定依据**（`dd_table_share.cc:1049`）：

```cpp
  if ((auto_flags & Field::NEXT_NUMBER) != 0)
    share->found_next_number_field = &share->field[field_nr];
```

**★ `autoinc_field_has_explicit_non_null_value`**——自增列 NOT NULL，所以**无法从值上区分"用户显式写了 0"还是"NULL 被转成 0"**，必须靠这个独立标志位：

```cpp
  /**
    To indicate that value of the auto_increment field was provided
    explicitly by the user or from some other source (e.g. in case of
    INSERT ... SELECT, ALTER TABLE or LOAD DATA) and not as default
    or result of conversion from NULL value.
    @note Since auto_increment fields are always non-NULL we can't find
          out using methods of Field class if 0 value stored in such field
          was provided explicitly or is result of applying default/conversion
          from NULL value. ... Hence the need for this flag.
    @note Used only in the MODE_NO_AUTO_VALUE_ON_ZERO mode and only
          by handler::write_row().
  */
  bool autoinc_field_has_explicit_non_null_value{false};
```

> ★ 它有 **18 个设置/清除点**（fill_record 两处置 true、LOAD DATA 四种格式各一处、分区 handler 置 true 后恢复、binlog Rows_log_event 三处、`TABLE::reset()` 置 false、RAII guard 析构置 false……）。`Autoinc_field_has_explicit_non_null_value_reset_guard` 是每行处理完自动清位的 RAII 类。

**★ `ha_write_row()` 里没有自增逻辑**——server 层唯一的 `update_auto_increment()` 调用点是**分区 handler**：

```cpp
int handler::ha_write_row(uchar *buf) {
  ...
  MYSQL_TABLE_IO_WAIT(PSI_TABLE_WRITE_ROW, MAX_KEY, error,
                      { error = write_row(buf); })   // ★ 转虚函数，自增由 SE 内部处理
  if (unlikely(error)) return error;
  if (unlikely((error = binlog_log_row(table, nullptr, buf, log_func))))
    return error;
  return 0;
}
```

```cpp
// InnoDB（ha_innodb.cc:9051）—— 在 write_row 内部调基类方法
  if (table->next_number_field && record == table->record[0]) {
    m_prebuilt->autoinc_error = DB_SUCCESS;
    error_result = update_auto_increment();      // ★ 基类 handler 方法，内部再回调 get_auto_increment()
    if (error_result) { ... }
    auto_inc_used = true;
  }
  ...
  error = row_insert_for_mysql((byte *)record, m_prebuilt);   // ★ 值已写进 record[0]
```

> ⚠️ **`update_auto_increment()` 不是虚函数**（handler.h:4931，在 `protected:` 段）。真正可覆盖的是 `get_auto_increment()` / `release_auto_increment()`。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.1 | 引入 `innodb_autoinc_lock_mode`（0/1/2）三档锁模式 |
| 5.7 及以前 | 计数器**只存内存**，重启后 `SELECT MAX(col)+1` 重算 → 可能**回退**（删除过的最大值会被重新使用） |
| **8.0** | 计数器**持久化**（redo + DDTableBuffer），重启不回退；默认值改为 `2` |
| 8.0 | 默认 `binlog_format=ROW` + `innodb_autoinc_lock_mode=2`，两者配套 |

### ★ 持久化方案的历史演进（Bug #199 及其各路解法）

"重启丢自增"是**困扰了 15 年的经典问题**（[Bug #199](https://bugs.mysql.com/bug.php?id=199)，2003 年报告），各家解法不同：

| 方案 | 提出者 | 做法 | 代价 |
|---|---|---|---|
| `SELECT MAX+1` | MySQL 官方（5.7 前默认） | 重启后扫描 PK 取最大 | ★ **回退风险**：删除过的最大值会被重新使用 |
| **写 PK root 页 `PAGE_MAX_TRX_ID`** | **AliSQL**（[commit 02a5207](https://github.com/alibaba/AliSQL/commit/02a52074a637303a6b298b5f452b7673830f3ad8)），贡献给 **MariaDB**（[MDEV-6076](https://jira.mariadb.org/browse/MDEV-6076)）；**PolarDB / CynosDB 同源** | 把 autoinc 写在 PK root 页一个**原本不用的位置**（`PAGE_MAX_TRX_ID` 槽） | 改动随页写入走 redo，天然持久；但依赖"二级索引根页字段复用"，且只在特定版本可用 |
| ★ **redo + DDTableBuffer** | MySQL 8.0 官方 | 计数器变化写 `MLOG_TABLE_DYNAMIC_META` redo，checkpoint 时快照进 `mysql.innodb_dynamic_metadata`（见 [`innodb_dict.md`](../server/dd/innodb_dict.md) 动态元数据节） | 通用机制（autoinc / corrupted index / 更新时间共用），但要理解三层（内存三态 + redo + buffer 表） |

★ **三种方案的分野**：AliSQL 方案是"借一个物理槽位"（改动最小、但依赖页布局细节），8.0 方案是"建一条通用动态元数据通道"（改动大、但 autoinc 只是它第一个乘客）。8.0 之所以没采用 PAGE_MAX_TRX_ID 方案，很可能是因为它要服务的不止 autoinc 一个需求（corrupt 标记、update_time 都要持久化），专门为 autoinc 复用页内槽位太窄。

**★ 8.0 方案的两位乘客**（`persistent_type_t`，`dict0mem.h:2752`）——autoinc 只是其一：

```cpp
enum persistent_type_t {
  PM_SMALLEST_TYPE = 0,
  /** Persistent Metadata type for corrupted indexes */
  PM_INDEX_CORRUPTED = 1,
  /** Persistent Metadata type for autoinc counter */
  PM_TABLE_AUTO_INC = 2,          // ★ 自增 = 2
  /* TODO: Will add following types
  PM_TABLE_UPDATE_TIME = 3,
  PM_TABLESPACE_SIZE = 4,
  PM_TABLESPACE_MAX_TRX_ID = 5, */
  PM_BIGGEST_TYPE = 3
};
```

**⚠️ 三个文件都不存在**：`dict0persist.cc` / `dict0persist.h` / `persist0persist.h` 全库 0 命中。8.0 后期已合并——`Persister` / `PersistentTableMetadata` / `Persisters` 在 **`include/dict0mem.h:2752-3022`**，实现在 `dict/dict0dict.cc`。

**★ redo 记录的二进制布局**（`mtr0types.h:236`，源码注释原文）：

```
 * auto-inc redo :
 *      1B (类型)     |  1 ~ 11B （auto inc 值） |
 * PM_TABLE_AUTO_INC |   auto inc compressed    |
```

```cpp
  /** log for some persistent dynamic metadata change */
  MLOG_TABLE_DYNAMIC_META = 62,       // ★ redo 类型 62
```

```cpp
ulint AutoIncPersister::write(const PersistentTableMetadata &metadata,
                              byte *buffer, ulint size) const {
  ulint length = 0;
  uint64_t autoinc = metadata.get_autoinc();
  mach_write_to_1(buffer, static_cast<byte>(PM_TABLE_AUTO_INC));   // ★ 1 字节类型
  ++length; ++buffer;
  ulint len = mach_u64_write_much_compressed(buffer, autoinc);     // ★ 1~11 字节变长压缩
  length += len;
  return (length);
}
```

> ★ 每条自增 redo 只有 **2~12 字节**（1B 类型 + 变长压缩整数 + 23B 记录头）。写之前不 flush（`/* No need to flush due to performance reason */`），靠崩溃恢复重放。

**★ 崩溃恢复不是某个 `dict_table_autoinc_recover()`**（该函数不存在），而是三段：

```
recv_parse_log_rec()  MLOG_TABLE_DYNAMIC_META 分支（log0recv.cc:2867）
  → MetadataRecover::parseMetadataLog()   // 解析 + 按 version 合并
  → MetadataRecover::store()              // 写回 mysql.innodb_dynamic_metadata
  → dict_table_load_dynamic_metadata() / dict_table_apply_dynamic_metadata()
```

```cpp
static bool dict_table_apply_dynamic_metadata(
    dict_table_t *table, const PersistentTableMetadata *metadata) {
  ...
  /* If the metadata version is bigger than the one in table.
  it could be that an ALTER TABLE has been rolled back, so metadata
  in new version should be ignored too. */
  if (table->version != metadata->get_version()) {   // ★★ version 必须严格相等才应用
    return (get_dirty);
  }
  uint64_t autoinc = metadata->get_autoinc();
  if (autoinc > table->autoinc_persisted) {
    table->autoinc = autoinc;                        // ★★ 自增在此被读回
    table->autoinc_persisted = autoinc;
    get_dirty = true;
  }
```

> ★★ **version 机制是理解自增持久化的关键**：每次 DDL 让 version++，于是旧 redo / 旧 DDTableBuffer 行自动作废（这是 `dd_set_autoinc()` 里 `version++` 的目的）。

**★ 多条 redo 的合并规则**（`AutoIncPersister::aggregate`）：

```cpp
void AutoIncPersister::aggregate(
    PersistentTableMetadata &metadata,
    const PersistentTableMetadata &new_entry) const {
  if (new_entry.get_version() > metadata.get_version()) {
    metadata.set_autoinc(new_entry.get_autoinc());          // 版本更新 → 覆盖
    metadata.set_version(new_entry.get_version());
  } else if (new_entry.get_version() == metadata.get_version()) {
    metadata.set_autoinc_if_bigger(new_entry.get_autoinc()); // 同版本 → 取大者
  }
}
```

**★ 落盘节奏：没有独立后台线程**，由 **log checkpointer 线程**在 `log_consider_checkpoint()` 里调 `dict_persist_to_dd_table_buffer()`，周期 = `innodb_log_checkpoint_every`（默认 1000 ms）。关键约束：

```cpp
  /* As soon as we release the dict_persist->mutex, new dynamic
  metadata changes could happen. They would be not persisted
  until next call to dict_persist_to_dd_table_buffer.
  We must not remove redo which could allow to deduce them.
  Therefore the maximum allowed lsn for checkpoint is the
  current lsn. */
  log_set_dict_max_allowed_checkpoint_lsn(*log_sys, persisted_lsn);
```

> ★★ **checkpoint 不能越过"最后一次写回 DDTableBuffer 的 LSN"**——否则 redo 被回收后元数据就丢了。这就是 redo 与 DDTableBuffer 的契约。

---

## 理论基础

### 为什么自增需要专门的锁

> **视角声明**：本篇从**跨层特性**视角讲三档模式的语义、默认值与对复制的影响，以及 handler 侧的获取路径。**AUTOINC 作为锁机制本身**（`lock_t` 的第五种表锁模式、兼容矩阵、为什么必须是语句级释放、与 `autoinc_mutex` 的分工）的权威剖析见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)「AUTOINC 锁」——两处互指，不重复详写。

自增值在**引擎内部**生成，不走事务的行锁，因此需要单独的机制保证并发下不重复。三种模式是"**并发度 vs 语句级确定性**"的权衡：

| 模式 | 常量 | 行为 | 适用场景 |
|---|---|---|---|
| 0 traditional | `AUTOINC_OLD_STYLE_LOCKING` | **所有** INSERT 类语句都拿表级 `LOCK_AUTO_INC`，**语句结束**才释放 | SBR 下安全（语句串行） |
| 1 consecutive | `AUTOINC_NEW_STYLE_LOCKING` | 简单 INSERT（行数已知）只拿 mutex 分配区间；`INSERT...SELECT`/`LOAD`（行数未知）仍拿表锁 | 默认 SBR 安全的折中 |
| 2 interleaved | `AUTOINC_NO_LOCKING` | 任何语句都**只拿 mutex**，表锁完全不用 | 并发最高，但**并发插入的自增值会交错**，SBR 下不安全 |

```cpp
static const long AUTOINC_OLD_STYLE_LOCKING = 0;
static const long AUTOINC_NEW_STYLE_LOCKING = 1;
static const long AUTOINC_NO_LOCKING = 2;
```

**默认值**（`ha_innodb.cc:22903`）：

```cpp
MYSQL_SYSVAR_LONG(autoinc_lock_mode, innobase_autoinc_lock_mode,
                  PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,   // ★ 只读，需重启修改
                  "... 2 => No AUTOINC locking (unsafe for SBR)",
                  nullptr, nullptr, AUTOINC_NO_LOCKING,        // ★ 默认 2
                  AUTOINC_OLD_STYLE_LOCKING, AUTOINC_NO_LOCKING, 0);
```

★ 8.0 把默认从 1 改成 **2**，前提是默认 `binlog_format=ROW`——RBR 记录的是行镜像，不依赖自增值的生成顺序，所以"交错"无害。若用 SBR 必须改回 1。

**★ `LOCK_AUTO_INC` 是独立的第五种表锁模式**（`lock0types.h:51`，值 4）——不是表级 X 锁：

```cpp
enum lock_mode {
  LOCK_IS = 0,      /* intention shared */
  LOCK_IX,          /* intention exclusive */
  LOCK_S,           /* shared */
  LOCK_X,           /* exclusive */
  LOCK_AUTO_INC,    /* locks the auto-inc counter of a table
                    in an exclusive mode */        // ★ 枚举值 = 4
  LOCK_NONE,        /* this is used elsewhere to note consistent read */
  LOCK_NUM = LOCK_NONE,
  LOCK_NONE_UNSET = 255
};
```

**★★ 兼容矩阵与强度矩阵**（`lock0priv.h:592`，源码注释原文）：

```cpp
/*  LOCK COMPATIBILITY MATRIX
    IS IX S  X  AI
 IS +    +  +  -  +
 IX +    +  -  -  +
 S  +    -  +  -  -
 X  -    -  -  -  -
 AI +    +  -  -  -          ★ AI 行：与 IS/IX 兼容，与 S/X/AI 都不兼容
*/
static const byte lock_compatibility_matrix[5][5] = {
    /* IS */ {true,  true,  true,  false, true },
    /* IX */ {true,  true,  false, false, true },
    /* S  */ {true,  false, true,  false, false},
    /* X  */ {false, false, false, false, false},
    /* AI */ {true,  true,  false, false, false}};

/*  STRONGER-OR-EQUAL RELATION (mode1=row, mode2=column)
    IS IX S  X  AI
 IS +  -  -  -  -
 IX +  +  -  -  -
 S  +  -  +  -  -
 X  +  +  +  +  +
 AI -  -  -  -  +          ★ AI 只 ≥ AI
*/
static const byte lock_strength_matrix[5][5] = {
    /* IS */ {true,  false, false, false, false},
    /* IX */ {true,  true,  false, false, false},
    /* S  */ {true,  false, true,  false, false},
    /* X  */ {true,  true,  true,  true,  true },
    /* AI */ {false, false, false, false, true }};
```

> ★★ **两条结论**：① AI 与 AI 不兼容 ⇒ **任一时刻只有一个持有者**；② 强度矩阵里 AI 只 ≥ 自己 ⇒ **IX 不能顶替 AI**，所以 `lock_table_has()` 不会复用 IX 去"覆盖" AI 请求。
> ⚠️ **`LOCK_AUTO_INC` 不会把事务升级为 RW**（`lock0lock.cc` 里 `if ((mode == LOCK_IX || mode == LOCK_X) ...)` 才调 `trx_set_rw_mode()`）。

**★ 释放路径：`row_unlock_table_autoinc_for_mysql()` 不存在**——8.0 是 `lock_unlock_table_autoinc()`，由**语句结束**触发：

```cpp
void lock_unlock_table_autoinc(trx_t *trx) {
  ...
  trx_mutex_enter(trx);
  ut_ad(!trx->lock.wait_lock);
  bool might_have_autoinc_locks = lock_trx_holds_autoinc_locks(trx);  // ★ 先廉价探测
  trx_mutex_exit(trx);

  if (might_have_autoinc_locks) {
    /* lock_release_autoinc_locks() requires exclusive global latch as the
    AUTOINC locks might be on tables from different shards. */
    locksys::Global_exclusive_latch_guard guard{UT_LOCATION_HERE};
    trx_mutex_enter(trx);
    lock_release_autoinc_locks(trx);    // ★ 逆序释放
    trx_mutex_exit(trx);
  }
}
```

> ★★ **这就是"AUTOINC 锁是语句级、不是事务级"的源码依据**——它挂在 `trx->lock.autoinc_locks` 向量上，语句结束就逆序（LIFO）全部放掉，**不跟事务到 commit**。

**⚠️ AI 锁的 `lock_t` 分配有两种路径**（`lock_table_create`，`lock0lock.cc:3436`）：

```cpp
  ++table->count_by_mode[type_mode & LOCK_MODE_MASK];
  /* For AUTOINC locking we reuse the lock instance only if
  there is no wait involved else we allocate the waiting lock
  from the transaction lock heap. */
  if (type_mode == LOCK_AUTO_INC) {
    lock = table->autoinc_lock;            // ★ 立即授予 → 复用表上预分配的 lock_t
    ut_ad(table->autoinc_trx == nullptr);
    table->autoinc_trx = trx;              // ★ 登记持有者
    ib_vector_push(trx->lock.autoinc_locks, &lock);
  } else if (...) { ... }
```

> ★ `dict_table_t::autoinc_lock` 是表上**预分配**的缓冲——注释明说：*"otherwise the lock heap would grow rapidly if we do a large insert from a select"*。只有**需要等待时**才从 trx lock heap 分配。
> ⚠️ **`n_waiting_or_granted_auto_inc_locks` 不存在**——真名 `ulong count_by_mode[LOCK_NUM]`（`dict0mem.h:2438`），`count_by_mode[LOCK_AUTO_INC]` 就是"持有者 + 等待者"总数。`innobase_lock_autoinc()` 里 mode 1 的降级判据读的就是它（可免 latch 偷看）。

---

## 核心实现

### handler 侧获取：innobase_lock_autoinc（跨层调用路径）

> 下面这个 switch 的**锁语义**（每种模式到底拿什么锁、为什么 mode 2 不拿表锁）见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)「AUTOINC 锁」；本篇关注**调用路径与分支条件**。

```cpp
dberr_t ha_innobase::innobase_lock_autoinc(void) {
  long lock_mode = innobase_autoinc_lock_mode;

  /* intrinsic 表（会话私有）或显式要求不加锁 → 一律不加 */
  if (m_prebuilt->table->is_intrinsic() || m_prebuilt->no_autoinc_locking) {
    lock_mode = AUTOINC_NO_LOCKING;
  }

  switch (lock_mode) {
    case AUTOINC_NO_LOCKING:
      dict_table_autoinc_lock(m_prebuilt->table);      // ① 只拿 mutex
      break;

    case AUTOINC_NEW_STYLE_LOCKING:
      /* 只有简单 INSERT / REPLACE 能走"轻量路径" */
      if (thd_sql_command(m_user_thd) == SQLCOM_INSERT ||
          thd_sql_command(m_user_thd) == SQLCOM_REPLACE) {
        dict_table_t *ib_table = m_prebuilt->table;
        dict_table_autoinc_lock(ib_table);             // 先拿 mutex

        if (ib_table->count_by_mode[LOCK_AUTO_INC]) {
          /* 已有别人持有表级 AUTO_INC 锁（说明有 LOAD/INSERT..SELECT 在跑） */
          dict_table_autoinc_unlock(ib_table);         // ② 先放 mutex 再降级，避免死锁
        } else {
          break;                                        // 轻量路径成立
        }
      }
      [[fallthrough]];                                  // ③ 否则落到 old style

    case AUTOINC_OLD_STYLE_LOCKING:
      error = row_lock_table_autoinc_for_mysql(m_prebuilt);   // 拿表级 LOCK_AUTO_INC
      if (error == DB_SUCCESS) {
        dict_table_autoinc_lock(m_prebuilt->table);           // 再拿 mutex
      }
      break;
  }
  return error;
}
```

三个关键点：

1. **①② 的"先放再降级"**：mode 1 下如果检测到已有事务持有表级 `LOCK_AUTO_INC`（`count_by_mode[LOCK_AUTO_INC]` 非空，说明有 `INSERT...SELECT`/`LOAD` 正在跑），就**先释放 mutex 再走 old style**——直接拿两把锁会与持有顺序相反的线程死锁。
2. **③ `[[fallthrough]]`**：mode 1 对非简单 INSERT（或上述降级场景）**直接复用 mode 0 的实现**——所以 mode 1 是"能轻则轻、不能轻则重"的自适应。
3. **mutex 无论如何都要拿**：最终推进计数器都要在 `dict_table_autoinc_lock` 保护下进行，表锁只是额外保证"语句期间不被别的语句插入"。

### 区间分配：get_auto_increment

handler 接口一次要**一段**值，不是一个个要：

```cpp
void ha_innobase::get_auto_increment(ulonglong offset, ulonglong increment,
                                     ulonglong nb_desired_values,
                                     ulonglong *first_value,
                                     ulonglong *nb_reserved_values);
```

读取入口 `innobase_get_autoinc`（:19653）：

```cpp
dberr_t ha_innobase::innobase_get_autoinc(ulonglong *value) {
  *value = 0;
  m_prebuilt->autoinc_error = innobase_lock_autoinc();        // 按模式加锁
  if (m_prebuilt->autoinc_error == DB_SUCCESS) {
    *value = dict_table_autoinc_read(m_prebuilt->table);       // 读当前计数器
    if (*value == 0) {                                          // 未初始化 → 报错
      m_prebuilt->autoinc_error = DB_UNSUPPORTED;
      dict_table_autoinc_unlock(m_prebuilt->table);
    }
  }
  return m_prebuilt->autoinc_error;
}
```

★ 源码里有个坦白的注释（:19715）：`nb_desired_values` **只对多行 INSERT 的第一次调用准确**，对 `LOAD` 等语句"meaningless"——所以 InnoDB 只在第一次调用时记录它，后续调用不再依赖。未知行数（如 `INSERT...SELECT`）时按 1 个一个申请，这就是 mode 1 下这类语句仍要拿表锁的原因（否则每次只拿 mutex 会导致值交错、破坏连续性）。

**★ `nb_desired_values` 的三个来源**（这是"批量预分配"的完整答案）。注意**三个来源都在 server 层 `handler::update_auto_increment()`，不在 InnoDB**：

```cpp
      if ((auto_inc_intervals_count == 0) && (estimation_rows_to_insert > 0))
        nb_desired_values = estimation_rows_to_insert;   // ① ha_start_bulk_insert(rows) 给的估计
      else if ((auto_inc_intervals_count == 0) &&
               (thd->lex->bulk_insert_row_cnt > 0)) {
        nb_desired_values = thd->lex->bulk_insert_row_cnt;  // ② 多行 INSERT ... VALUES 的行数
      } else /* go with the increasing defaults */ {
        if (auto_inc_intervals_count <= AUTO_INC_DEFAULT_NB_MAX_BITS) {
          nb_desired_values =
              AUTO_INC_DEFAULT_NB_ROWS * (1 << auto_inc_intervals_count);  // ③ 指数增长
          nb_desired_values = std::min(nb_desired_values,
                                       ulonglong(AUTO_INC_DEFAULT_NB_MAX));
        } else nb_desired_values = AUTO_INC_DEFAULT_NB_MAX;
      }
```

```cpp
#define AUTO_INC_DEFAULT_NB_ROWS 1     // ★ 8.0 实测默认只预留 1 个（注释："Some prefer 1024 here"）
#define AUTO_INC_DEFAULT_NB_MAX_BITS 16
#define AUTO_INC_DEFAULT_NB_MAX ((1 << AUTO_INC_DEFAULT_NB_MAX_BITS) - 1)   // 65535
```

| 语句 | `ha_start_bulk_insert()` 参数 | 走哪个来源 |
|---|---|---|
| 单行/多行 `INSERT ... VALUES` | `insert_many_values.size()`（>0） | ①（失败退 ②） |
| `INSERT ... SELECT` | `(ha_rows)0`（**未知**） | ③ 指数增长：1,2,4,8,...,65535 |
| `CREATE ... SELECT` | `(ha_rows)0` | ③ |
| `LOAD DATA` | `(ha_rows)0` | ③ |

> ⚠️ **`rows == 0` 表示"很多行"，不是"0 行"**（`ha_start_bulk_insert` 注释原文：*"rows == 0 means we will probably insert many rows"*）。

**InnoDB 侧只采信第一次**（`get_auto_increment`，`ha_innodb.cc:19688`）：

```cpp
  /* Called for the first time ? */
  if (trx->n_autoinc_rows == 0) {
    trx->n_autoinc_rows = (ulint)nb_desired_values;
    /* It's possible for nb_desired_values to be 0:
    e.g., INSERT INTO T1(C) SELECT C FROM T2; */
    if (nb_desired_values == 0) {
      trx->n_autoinc_rows = 1;              // ★ 未知行数 → 退化成每次 1 个
    }
    *first_value = std::max(*first_value, autoinc);
  } else if (...)
  *nb_reserved_values = trx->n_autoinc_rows;
```

**★ 传统模式不预分配**（这是 mode 0 与其它模式的本质区别之一）：

```cpp
  if (innobase_autoinc_lock_mode != AUTOINC_OLD_STYLE_LOCKING ||
      m_prebuilt->no_autoinc_locking) {
    next_value = innobase_next_autoinc(current, *nb_reserved_values, increment,
                                       offset, col_max_value);
    m_prebuilt->autoinc_last_value = next_value;
    if (m_prebuilt->autoinc_last_value < *first_value) {
      *first_value = (~(ulonglong)0);       // 溢出信号
    } else {
      dict_table_autoinc_update_if_greater(m_prebuilt->table,
                                           m_prebuilt->autoinc_last_value);  // ★ 预分配：跳到区间末端
    }
  } else {
    /* This will force write_row() into attempting an update
    of the table's AUTOINC counter. */
    m_prebuilt->autoinc_last_value = 0;     // ★★ 传统模式：置 0 强制逐行回写
  }
```

> ★ `autoinc_last_value == 0` 是"未预分配"的哨兵值。`write_row()` 里 `if (auto_inc >= m_prebuilt->autoinc_last_value)` 命中就回写计数器——传统模式下恒真，所以**每一行都回写**。

**★ 区间算法 `innobase_next_autoinc()`**（`ha_innodb.cc:2671`）——这是自增数学的核心：

```cpp
ulonglong innobase_next_autoinc(
    ulonglong current,   /* 当前值 */
    ulonglong need,      /* 需要的个数 */
    ulonglong step,      /* auto_increment_increment */
    ulonglong offset,    /* auto_increment_offset */
    ulonglong max_value) /* 列类型上限 */
{
  ulonglong next_value;
  ulonglong block = need * step;      // ★ 本次要跨过的跨度 = 个数 × 步长

  /* According to MySQL documentation, if the offset is greater than
  the step then the offset is ignored. */
  if (offset > block) {               // ★ offset > 总跨度时忽略 offset（多主复制场景）
    offset = 0;
  }

  /* Check for overflow. */
  if (block >= max_value || offset > max_value || current >= max_value ||
      max_value - offset <= offset) {
    next_value = max_value;           // 哨兵 1：撞顶
  } else {
    ulonglong free = max_value - current;
    if (free < offset || free - offset <= block) {
      next_value = max_value;         // 哨兵 2：剩余空间放不下一个 block
    } else {
      next_value = 0;                 // 哨兵 3：还有空间
    }
  }

  if (next_value == 0) {
    ulonglong next;
    if (current > offset) next = (current - offset) / step;
    else                  next = (offset - current) / step;
    next_value = next * step;
    /* Check for overflow */
    if (max_value - next_value >= block) {
      next_value += block;            // ★ 推进一个整 block（批量预留在此发生）
      if (max_value - next_value >= offset) {
        next_value += offset;         // ★ 再叠加 offset
      } else {
        next_value = max_value;
      }
    } else {
      next_value = max_value;
    }
  }
  ut_a(next_value != 0);
  ut_a(next_value <= max_value);
  return (next_value);                // ★ 返回"区间之后的上界"
}
```

> ★★ **返回值语义**：不是"下一个可用值"，而是**已用区间的上界**。所以 `get_auto_increment()` 拿到后直接 `dict_table_autoinc_update_if_greater(table, last_value)` —— 计数器一步跳到区间末端。

`release_auto_increment`（:19674）做收尾：把 `trx->n_autoinc_rows` 清零（批量插入预估行数的记账）。

```cpp
void ha_innobase::release_auto_increment() {
  trx_t *trx = m_prebuilt->trx;
  TrxInInnoDB trx_in_innodb(trx);
  /* Clear the trx's autoinc_rows as the operation requiring auto increment
  values has been completed. */
  if (trx->n_autoinc_rows > 0) {
    trx->n_autoinc_rows = 0;      // ★ 语句级计数器清零，下一语句重新采信 nb_desired_values
  }
}
```

> ⚠️ **`m_prebuilt->autoinc_no_reserved` 不存在**——预留行数记在 `trx_t::n_autoinc_rows`（`trx0trx.h:1046`）。它在 `write_row()` 里**每写一行 `--`**（倒计时），并在 commit / rollback / `end_stmt` 三处重置。

### 计数器推进与持久化：dict_table_autoinc_log

8.0 的关键改进——**计数器写 redo**（`dict0dict.cc:758`）：

```cpp
bool dict_table_autoinc_log(dict_table_t *table, uint64_t value, mtr_t *mtr) {
  bool log = false;
  mutex_enter(table->autoinc_persisted_mutex);

  if (table->autoinc_persisted < value) {            // ① 单调递增才更新
    dict_table_autoinc_persisted_update(table, value);

    /* 与 checkpoint 的并发协调（此处省略十余行注释） */
    if (table->dirty_status.load() == METADATA_DIRTY) {
      ut_ad(table->in_dirty_dict_tables_list);
    } else {
      dict_table_mark_dirty(table);                   // ② 标记表为 dirty，待回写 DDTableBuffer
    }
    log = true;
  }
  mutex_exit(table->autoinc_persisted_mutex);

  if (log) {
    PersistentTableMetadata metadata(table->id, table->version);
    metadata.set_autoinc(value);
    Persister *persister = dict_persist->persisters->get(PM_TABLE_AUTO_INC);
    persister->write_log(table->id, metadata, mtr);    // ③ 写 redo
    /* No need to flush due to performance reason */   // ④ 不强刷
  }
  ...
}
```

1. ① 只在更大时更新——**计数器永不回退**的保证；
2. ② `dirty_status` 是为了把值最终回写到 `mysql.innodb_dynamic_metadata`（DDTableBuffer），redo 负责崩溃窗口、DDTableBuffer 负责长期持久化；
3. ③ redo 类型 `PM_TABLE_AUTO_INC`；
4. ④ ★ 注释明说**不强刷 redo**——自增计数器丢一点无所谓（重启后最多是"比实际最大值小"，而下面的 SELECT MAX 兜底会修正），所以为了性能不刷盘。

调用点：插入时 `row0ins.cc:2451`、更新时 `row0upd.cc:2829`。

```cpp
// 插入侧（row0ins.cc:2437）—— ★ 在拿到 index 锁之后、任何其它资源之前写，防死锁
  bool persist_autoinc = false;
  if (!index->table->is_temporary() &&                // ★ 临时表不写 redo
      dict_table_has_autoinc_col(index->table)) {
    uint64_t counter =
        row_get_autoinc_counter(entry, index->table->autoinc_field_no);
    if (counter != 0) {
      /* Always log the counter change first, so it won't
      be affected by any follow-up failure. */
      persist_autoinc = dict_table_autoinc_log(index->table, counter, &mtr);
    }
  }
  ...
  /* Persist auto increment value to DD buffer table if requested. Do it after
  closing the mini transaction and releasing latches. */
  if (persist_autoinc) {
    dict_table_persist_to_dd_table_buffer(index->table);
  }
```

**★★ 双计数器：8.0 的关键设计**（`dict0mem.h:2328`，注释原文）：

```cpp
  /** Mutex protecting the autoincrement counter. */
  ib_mutex_t *autoinc_mutex;                          // ★ 锁 A

  /** Autoinc counter value to give to the next inserted row. */
  uint64_t autoinc;                                   // ★ 内存计数器（可领先）

  /** Mutex protecting the persisted autoincrement counter. */
  ib_mutex_t *autoinc_persisted_mutex;                // ★ 锁 B（与 A 分离）

  /** Autoinc counter value that has been persisted in redo logs or
  DDTableBuffer. ...
  This is different from the 'autoinc' above, which could be bigger
  than this one, because 'autoinc' will get updated right after
  some counters are allocated, but we will write the counter to redo
  logs and update this counter later. ...
  We want this counter because when we need to write the counter back
  to DDTableBuffer, we had better keep it consistency with the counter
  that has been written to redo logs. Besides, we can't read the 'autoinc'
  directly easily, because the autoinc_lock is required and there could
  be a deadlock. */
  uint64_t autoinc_persisted;                         // ★ 已落 redo/DDTableBuffer 的值，恒 <= autoinc

  /** The position of autoinc counter field in clustered index. */
  ulint autoinc_field_no;                             // ★ ULINT_UNDEFINED = 无自增列

  /** The transaction that currently holds the AUTOINC lock on this table. */
  std::atomic<const trx_t *> autoinc_trx;             // ★ 可免 latch 偷看
```

> ★★ **不变式：`autoinc_persisted <= autoinc`**。`autoinc` 在分配时立刻推进，`autoinc_persisted` 在写 redo 后才推进。崩溃时最多丢"两者之差"，而 `SELECT MAX` 兜底会向上修正——**方向永远安全**（只可能偏大）。
> ⚠️ 两把 mutex 是**分开的**（`autoinc_mutex` vs `autoinc_persisted_mutex`），且都是**懒创建**（`os_once::do_or_wait_for_done`，`dict_table_autoinc_alloc`）——不是每表都付成本。

**`dirty_status` 三态状态机**（`enum table_dirty_status`，`dict0mem.h:1900`）：

```
METADATA_CLEAN ──[autoinc_log 首次变更]──► METADATA_DIRTY ──[checkpoint 回写]──► METADATA_BUFFERED
                                                  ▲                                      │
                                                  └──────[再次变更]─────────────────────┘
```

```cpp
enum table_dirty_status {
  METADATA_DIRTY = 0,     // 内存脏了，待写回 DDTableBuffer
  METADATA_BUFFERED,      // 已写进 DDTableBuffer，待写回 DD
  METADATA_CLEAN          // 全部最新，DDTableBuffer 里没这一行
};
```

```cpp
void dict_table_mark_dirty(dict_table_t *table) {
  mutex_enter(&dict_persist->mutex);
  switch (table->dirty_status.load()) {
    case METADATA_DIRTY:  break;                              // 已是 DIRTY
    case METADATA_CLEAN:
      UT_LIST_ADD_LAST(dict_persist->dirty_dict_tables, table);
      [[fallthrough]];
    case METADATA_BUFFERED:
      table->dirty_status.store(METADATA_DIRTY);              // BUFFERED→DIRTY 不再重复入链
      ++dict_persist->num_dirty_tables;
      dict_persist_update_log_margin();                       // ★ 预留 redo margin
  }
  mutex_exit(&dict_persist->mutex);
}
```

> ★ `dict_persist_update_log_margin()` 按脏表数估算 redo 预留（`log_margin_per_table_no_split = 80` 字节/表 + 页分裂 margin），并写进 `log_set_dict_persist_margin()`——**自增持久化会真实占用 redo 空间预算**。
> ⚠️ `in_dirty_dict_tables_list` **只在 `UNIV_DEBUG` 下存在**（`dict0mem.h:2177`），release 构建没有这个字段。

### 开表时的初始化：SELECT MAX 只在异常场景

★ **常见误解澄清**：很多人以为 8.0 重启后仍会 `SELECT MAX(col)`。实际只有"读出来是 0"时才走（`ha_innodb.cc:7090`）：

```cpp
read_auto_inc = dict_table_autoinc_read(m_prebuilt->table);   // ① 先读持久化的值

if (read_auto_inc == 0) {
  index = innobase_get_index(table->s->next_number_index);
  /* Execute SELECT MAX(col_name) FROM TABLE;
     This is necessary when an imported tablespace doesn't have a correct
     cfg file so autoinc has not been initialized, or the table is empty. */
  err = row_search_max_autoinc(index, col_name, &read_auto_inc);   // ② 兜底
  if (read_auto_inc > 0) {
    ib::warn(ER_IB_MSG_553) << "Reading max(auto_inc_col) = " << read_auto_inc
                            << " for table " << index->table->name
                            << ", because there was an IMPORT without cfg file.";
  }
} else {
  err = DB_SUCCESS;                                            // ③ 正常路径：直接用
}
```

- ① 有持久化值 → ③ 直接用，**不做 SELECT MAX**（这是 8.0 与 5.7 的本质差异）；
- ② 只有 IMPORT 无 cfg 文件、或表尚无值时才执行 `SELECT MAX`，并打 warning；
- 空表（`DB_RECORD_NOT_FOUND`）时 `auto_inc = 0` → **禁用自增生成但让 open 成功**（注释：读应成功、写应失败，让用户可以 `ALTER TABLE` 修正，见 :7150-7156）；
- 正常路径还要过 `innobase_next_autoinc(read_auto_inc, 1, 1, 0, col_max_value)`——按 increment=1 算下一个值，并**检查是否已达列类型上限**。

---

## DDL 交互

`CREATE TABLE ... AUTO_INCREMENT = x` / `ALTER TABLE ... AUTO_INCREMENT = x` / `OPTIMIZE` / `CREATE INDEX`（后者会走表重建）都要**保留原计数器**（`ha_innodb.cc:13628`）：

```cpp
if (m_create_info->auto_increment_value > 0 &&
    ((m_create_info->used_fields & HA_CREATE_USED_AUTO) ||
     cmd == SQLCOM_ALTER_TABLE || cmd == SQLCOM_OPTIMIZE || cmd == SQLCOM_CREATE_INDEX)) {
  dict_table_autoinc_lock(innobase_table);
  dict_table_autoinc_initialize(innobase_table, auto_inc_value);
  dict_table_autoinc_unlock(innobase_table);
}
```

即：**建表/改表时若显式指定 `AUTO_INCREMENT=x`，就用它初始化计数器**；否则重建后的表会继承旧值（否则重建表会导致自增值回退、产生重复键）。

**INPLACE ALTER 的最终计数器由 `commit_get_autoinc()` 决定**（`handler0alter.cc:6581`）——未指定时**直接沿用旧计数器**：

```cpp
  } else {
    /* An AUTO_INCREMENT value was not specified.
    Read the old counter value from the table. */
    ut_ad(old_table->found_next_number_field);
    dict_table_autoinc_lock(ctx->old_table);
    ctx->max_autoinc = ctx->old_table->autoinc;      // ★★ 沿用旧值
    dict_table_autoinc_unlock(ctx->old_table);
  }
```

> ★★ 但**显式指定 `AUTO_INCREMENT=x` 时有个反直觉的分支**：如果 `x` **小于等于**当前 `autoinc_persisted`，还要再扫一次索引求真实最大值。源码注释给了完整例子：表里有 (1),(2),(100),(200),(1000)，`DELETE WHERE a>200` 后执行 `ALTER TABLE t1 AUTO_INCREMENT=150`——期望下一个值是 **201 而不是 150**：

```cpp
    /* Here is an example:
    Let's say we have a table t1 with one AUTOINC column, existing
    rows (1), (2), (100), (200), (1000), after following SQLs:
    DELETE FROM t1 WHERE a > 200;
    ALTER TABLE t1 AUTO_INCREMENT = 150;
    we expect the next value allocated from 201, but not 150. */
    if (ctx->max_autoinc <= max_value_table) {
      err = row_search_max_autoinc(index, autoinc_field->field_name,
                                   &max_value_table);
      ...
```

```cpp
  /* 提交时写入新表（handler0alter.cc:7831） */
      dict_table_autoinc_lock(t);
      dict_table_autoinc_initialize(t, ctx->max_autoinc);
      t->autoinc_persisted = ctx->max_autoinc - 1;   // ★ 同步更新"已持久化"值
      dict_table_autoinc_set_col_pos(t, field->field_index());
      dict_table_autoinc_unlock(t);
```

**★ `TRUNCATE` 不一定重置计数器**——取决于 `m_keep_autoinc`：

```cpp
int innobase_truncate<Table>::truncate() {
  ...
  if (m_keep_autoinc) {                              // ★ 分区/部分 truncate：先存起来
    autoinc_persisted = m_table->autoinc_persisted;
    autoinc = m_table->autoinc;
  }
  ...                                                 // delete + create 重建
  if (m_keep_autoinc) {                              // ★ 再回填
    m_table->autoinc_persisted = autoinc_persisted;
    m_table->autoinc = autoinc;
  }
```

> ⚠️ **全表 `TRUNCATE` 时 `m_keep_autoinc = false`**（`ha_innodb.cc:14553`），计数器随新表重建归零；**分区 TRUNCATE / 部分 truncate 会保留计数器**。这是"TRUNCATE 一定把自增归 1"这个说法的反例。

**★ DD 里存的是"上一个已用值"，不是"下一个"**——所以读的时候要 `+1`，写的时候要 `-1`：

```cpp
// 读（dict0dd.cc:5129）
  m_table->version = version;
  dict_table_autoinc_lock(m_table);
  dict_table_autoinc_initialize(m_table, autoinc + 1);   // ★★ +1
  dict_table_autoinc_unlock(m_table);
  m_table->autoinc_persisted = autoinc;

// 写（dict0dd.cc:1613）
void dd_set_autoinc(dd::Properties &se_private_data, uint64_t autoinc) {
  if (autoinc > 0) {
    /* InnoDB persists the "previous" AUTO_INCREMENT value. */
    autoinc--;                                           // ★★ -1
  }
  ...
    if (!se_private_data.get(dd_table_key_strings[DD_TABLE_VERSION], &version)) {
      version++;      // ★ version++ 让 DDTableBuffer / redo 里的旧元数据作废
    }
  se_private_data.set(dd_table_key_strings[DD_TABLE_VERSION], version);
  se_private_data.set(dd_table_key_strings[DD_TABLE_AUTOINC], autoinc);
}
```

另：`TRUNCATE` 走 `ha_innobase::create()` 的旁路（`SQLCOM_TRUNCATE` → `truncate_impl()`），不走 `create_impl()`。

---

## Misc

### 自增列为什么有空洞

三个来源，都由"不回收"设计导致：

1. **事务回滚**：已分配的自增值不归还；
2. **批量预分配丢弃**：`INSERT ... SELECT` 预取一段（如 100 个），实际只用了 60 个，剩 40 个丢弃（`release_auto_increment` 里 `n_autoinc_rows` 清零，不归还）；
3. **DELETE 最大值**：删除最大行不会拉低计数器（单调递增保证）。

### 8.0 重启后自增值会回退吗

不会——这是 8.0 的主要修复。5.7 计数器只在内存，重启后 `SELECT MAX(col)+1`，若最大行被删过，值会**回退并重用**（可能与历史 binlog 冲突）。8.0 通过 `dict_table_autoinc_log` 写 redo + DDTableBuffer 持久化，重启直接读回。

注意：即使持久化丢了一小部分（因为不强刷 redo），SELECT MAX 兜底会向上修正——**方向是安全的**（只可能偏大不会偏小）。

### 达到类型上限

`innobase_next_autoinc` 会比对 `field->get_max_int_value()`；计数器到达上限后，后续插入拿不到新值，报**重复键错误**（InnoDB 行为：不再生成，反复用最大值撞主键）。

### 容易误解的概念

- **`innodb_autoinc_lock_mode` 是只读变量**（`PLUGIN_VAR_READONLY`），改它要重启。
- **mode 2 下 SBR 不安全**不是"会重复"，而是"并发插入的自增值交错，从库重放顺序不同导致主从数据逻辑不一致"（RBR 无此问题）。
- **一个表只能有一个自增列，且必须是某个索引的第一列**（否则报 `ER_WRONG_AUTO_KEY`）——与 `TABLE_SHARE` 里存 `next_number_index/keypart` 的设计对应。

## 参考

- *MySQL 8.0 Reference Manual → AUTO_INCREMENT Handling in InnoDB*
- *MySQL 8.0 Reference Manual → innodb_autoinc_lock_mode*
- [MySQL 自增列 — 豆沙包大的拳头](https://zhuanlan.zhihu.com/p/28383658735)
