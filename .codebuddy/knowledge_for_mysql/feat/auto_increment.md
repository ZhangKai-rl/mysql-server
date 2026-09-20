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

---

## 理论基础

### 为什么自增需要专门的锁

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

---

## 核心实现

### 锁获取：innobase_lock_autoinc（`ha_innodb.cc:8684`）

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

`release_auto_increment`（:19674）做收尾：把 `trx->n_autoinc_rows` 清零（批量插入预估行数的记账）。

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

另：`TRUNCATE` 会重置计数器（InnoDB 实现是重建表，见 [`../server/table.md`](../server/table.md)）。

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
