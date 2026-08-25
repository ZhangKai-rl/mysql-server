# InnoDB Undo Log 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 undo log 格式与记录类型、undo segment 与回滚段、undo 生成路径、事务回滚、MVCC 旧版本重建、purge 机制（边界、buffer pool watch、执行流程）、undo 表空间管理、DDL 与 undo（为什么 DDL 不记数据 undo，原子 DDL/DDL log 完整机制见 ddl.md）、GTID 持久化与 undo。其中 purge 边界、buffer pool watch 章节为源码级研究沉淀，其余章节为框架占位待补充。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [Undo log 格式与记录类型](#undo-log-格式与记录类型)
- [Undo segment 与回滚段](#undo-segment-与回滚段)
- [Undo 生成路径](#undo-生成路径)
- [Undo 与事务回滚](#undo-与事务回滚)
- [Undo 与 MVCC](#undo-与-mvcc)
- [Purge 机制](#purge-机制)
  - [Purge 边界](#purge-边界)
  - [Buffer Pool Watch 与 purge](#buffer-pool-watch-与-purge)
  - [Purge 执行流程](#purge-执行流程)
- [Undo 表空间管理](#undo-表空间管理)
- [DDL 与 Undo](#ddl-与-undo)
- [GTID 持久化与 Undo](#gtid-持久化与-undo)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

InnoDB 的回滚日志，记录事务对数据修改的"逆操作"信息。每条 undo log record 挂在 undo log segment 中，通过 `DB_ROLL_PTR`（7 字节 roll pointer）串联成记录的旧版本链。它是事务原子性（A，回滚）与隔离性（I，MVCC 一致性读）的载体。

### 用途

- 事务回滚：未提交事务失败时，按 undo log 逆向执行逆操作，恢复数据到事务前状态
- MVCC 一致性读：活跃 read view 通过 `DB_ROLL_PTR` 沿 undo 版本链重建历史版本，实现非锁定读
- Purge：已提交事务产生的 delete-marked 记录与过期 undo log，由 purge 线程异步清理
- 崩溃恢复：未提交事务的 undo 在恢复阶段回滚（与 redo 重放配合）
- GTID 持久化：commit 阶段将 GTID 写入 undo header，随 redo 持久化，后台线程异步刷 `mysql.gtid_executed` 表

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 回滚段固定在系统表空间；undo log 与数据页混在 ibdata |
| 5.7 | 引入独立 undo tablespace（`innodb_undo_tablespaces`），undo 可与数据分离 |
| 8.0 | 默认独立 undo tablespace；`innodb_undo_log_truncate` 自动 truncate undo；原子 DDL 引入 `mysql.innodb_ddl_log` 表 |
| 8.0.30 | undo tablespace 重新设计为固定 2 个隐式表空间（`#innodb_redo` 风格命名），truncate 机制优化 |

---

## 理论基础

### 设计模式

- **undo/redo 分离**：undo 记录"如何撤销"，redo 记录"如何重做"。InnoDB 对页面修改同时产生 redo（前向）与 undo（逆向），符合 ARIES 的 do-undo-redo 模型
- **版本链（version chain）**：每条聚簇索引记录的 `DB_ROLL_PTR` 指向最近一次修改的 undo record，undo record 内嵌旧 `DB_ROLL_PTR`，形成链表，沿链可重建任意历史版本
- **生产者-消费者（purge）**：事务 commit 产生 undo（生产者），purge 线程异步清理（消费者），靠 history list 与 purge view 衔接

### 相关论文

- **ARIES**（Mohan et al., 1992）：do-undo-redo 恢复模型。undo log 用于事务回滚与崩溃恢复回滚，redo log 用于前滚。InnoDB 遵循此模型但做了工程化简化

【待补充】undo 相关的其他学术参考

### 算法与数据结构

- **Undo log segment**：undo 日志的物理存储单元，位于 undo tablespace 或系统表空间
- **History list**：已提交事务的 undo log 按 `trx->no` 排序的链表，purge 按序消费
- **Roll pointer（DB_ROLL_PTR）**：7 字节，编码 (is_insert, rseg_id, page_no, offset)，定位 undo record
- **Purge view**：purge 线程克隆的最老 MVCC read view，决定 undo log 能否被清理

【待补充】undo record 内部格式、rseg 分配算法的复杂度分析

### 类似实现对比

【待补充】与 PostgreSQL / Oracle / SQL Server 的 undo 机制对比

### 历史背景

【待补充】undo 从系统表空间到独立表空间的演进动机，8.0 undo truncate 的设计权衡

---

## Undo log 格式与记录类型

【待补充】本章应覆盖：

- undo log page 布局（undo page header、segment header）
- undo record 二进制格式（type、undo_no、table_id、cmpl_info、字段旧值）
- 三种 record 类型：`TRX_UNDO_INSERT_REC`（插入回滚）、`TRX_UNDO_UPD_EXIST_REC`（更新已有记录）、`TRX_UNDO_DEL_MARK_REC`（删除标记）
- insert undo vs update undo 的区别（insert undo 事务提交后即可 purge，无需 MVCC；update undo 需等 purge view）
- `TRX_UNDO_UPD_DEL_REC`、extern storage 等特殊场景
- undo record 如何编码"逆操作"所需信息（旧字段值、旧 roll ptr）

---

## Undo segment 与回滚段

【待补充】本章应覆盖：

- `trx_rseg_t`（rollback segment）结构：rseg array、slot 分配
- undo tablespace 与 system tablespace 中的 rseg 分布
- `trx_undo_t`（undo log）结构：undo slot、insert undo vs update undo
- rseg mutex、undo slot 分配与释放
- history list 的组织（按 rseg 维护，purge 聚合）
- `trx_sys->rsegs` / `tmp_rsegs` / undo::spaces 的关系

---

## Undo 生成路径

【待补充】本章应覆盖：

- DML（insert/update/delete）如何产生 undo record
- `trx_undo_report_row` / `trx_undo_page_report_insert` / `trx_undo_page_report_modify`
- undo 写入与 redo 的关系（undo 页修改本身受 redo 保护）
- mtr 在 undo 写入中的角色
- `DB_ROLL_PTR` 的回填时机
- undo 大小估算与 page 分配

---

## Undo 与事务回滚

【待补充】本章应覆盖：

- 事务回滚入口（`row_undo` / `trx_rollback`）
- savepoint 机制
- 回滚的执行顺序（逆序应用 undo record）
- 崩溃恢复时的回滚（`trx_rollback_or_clean_recovered`）
- 大事务回滚的性能考量

---

## Undo 与 MVCC

【待补充】本章应覆盖：

- 一致性读如何沿 `DB_ROLL_PTR` 版本链重建旧版本（`row_vers_build_for_consistent_read`）
- read view 的四个字段（`m_low_limit_id` / `m_up_limit_id` / `m_creator_trx_id` / `m_ids`）与可见性判断（`changes_visible`）
- `trx->no`（提交序号）vs `trx->id`（开始序号）在 MVCC 中的不同作用
- AC-NL-RO 事务的 view 生命周期（closed 但不 free）
- read view 与 purge 的协同（view_list 按 low_limit_no 降序）

---

## Purge 机制

Purge 是 InnoDB 异步清理已提交事务遗留的 undo log 与 delete-marked 记录的后台机制。事务 commit 后，其 undo log 进入 history list，purge 线程按提交序号（`trx->no`）顺序清理——只要 undo log 不再被任何活跃 read view 需要，就可被 purge。

### Purge 边界

Purge 的边界是 `purge_sys->view.low_limit_no()`，即 purge view 的 `m_low_limit_no` 字段。边界检查点在 `trx_purge_fetch_next_rec`：

```
2276:2280:storage/innobase/trx/trx0purge.cc
  // XXXXXXXXX: purge停的位置
  if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) {
    return nullptr;
  }
```

即当 purge 迭代器遇到的 undo log 其 `trx_no >= low_limit_no()` 时停止 purge。

**来源链**：`trx_purge`（trx0purge.cc:2486）入口执行 `trx_sys->mvcc->clone_oldest_view(&purge_sys->view)`，purge 克隆 MVCC 中最老的活跃 read view 作为自己的 purge view。`clone_oldest_view`（read0read.cc:718）从 `m_views` 链表尾部取第一个未关闭的 view（链表按 `m_low_limit_no` 降序，尾部最老），若无活跃 view 则 `view->prepare(0)` 新建。

`m_low_limit_no` 的赋值在 `ReadView::prepare`（read0read.cc:448）：

```
453:455:storage/innobase/read/read0read.cc
  m_low_limit_no = trx_get_serialisation_min_trx_no();
  m_low_limit_id = trx_sys_get_next_trx_id_or_no();
```

`trx_get_serialisation_min_trx_no`（trx0sys.ic:270）返回 `trx_sys->serialisation_min_trx_no`。这个原子变量代表 serialisation_list（已提交待 purge 的事务链表）中最小的 `trx->no`，维护逻辑在 `trx_add_to_serialisation_list` / `trx_erase_from_serialisation_list_low`：

```
1514:1538:storage/innobase/trx/trx0trx.cc
  UT_LIST_ADD_LAST(trx_sys->serialisation_list, trx);

  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) == 1) {
    trx_sys->serialisation_min_trx_no.store(trx->no);
  }
  ...
  UT_LIST_REMOVE(trx_sys->serialisation_list, trx);
  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) > 0) {
    trx_sys->serialisation_min_trx_no.store(
        UT_LIST_GET_FIRST(trx_sys->serialisation_list)->no);
  } else {
    trx_sys->serialisation_min_trx_no.store(trx_sys_get_next_trx_id_or_no());
  }
```

事务 commit 阶段分配 `trx->no` 并加入 serialisation_list 尾部（list 按 `trx->no` 单调递增，因为 `trx_sys_allocate_trx_no` 在 serialisation_mutex 下单调递增分配）。list 非空时 `serialisation_min_trx_no` 是首元素（最小）的 `trx->no`；list 空时设为 `next_trx_id_or_no`（无待 purge 事务，所有已提交事务都可清理）。

**正确性论证**（read0read.cc:128 FACT C）：view_list 按 `low_limit_no` 降序排列（newest first），purge 克隆最老 view，故任何活跃 view 的 `low_limit_no` 都 >= purge view 的。`m_low_limit_no` 的语义是"view 不需要 `trx_no < 该值` 事务的 undo log"（这些事务在 view 创建前已完成 serialisation 收尾）。因此 `trx_no < purge_view.low_limit_no()` 的事务其 undo log 无任何活跃 view 需要，purge 安全。

**为何用 `m_low_limit_no`（trx->no 提交序）而非 `m_low_limit_id`（trx->id 开始序）**：purge 清理已提交事务的 undo log 必须按提交序判断。`trx->id` 是事务开始时分配的（写行 DB_TRX_ID），`trx->no` 是 commit 阶段才分配的提交序号。一个 `trx->id` 很小但很晚提交的事务，其 undo log 不能被早 purge。

**为何取"最老 view"而非"当前 serialisation_min"**：长期活跃的老 view 创建时 serialisation_min 还很小，仍可能需要 [其创建时 serialisation_min, 当前 serialisation_min) 区间内事务的旧版本。无活跃 view 时 `prepare(0)` 新建取当前 serialisation_min，所有已 serialise 事务都可 purge。

**GTID 持久化压低边界**：`clone_oldest_view` 末尾（read0read.cc:743）调 `view->reduce_low_limit(gtid_oldest_trxno)`，若 GTID 未刷盘的最老 trx_no < 当前 m_low_limit_no 则压低边界，避免 purge 删 undo log 破坏 GTID 持久化依赖的 undo header 读取。

### Buffer Pool Watch 与 purge

Watch 哨兵机制是 purge 线程清理 secondary index 上 delete-marked 记录时的关键协作手段，核心目的是让 purge 不必为了操作一个尚未读入 buffer pool 的页而阻塞 OLTP 路径。

完整链路从 purge worker 处理 secondary index 删除开始。当 purge 要删除二级索引记录时走 `BTR_DELETE_OP` 分支，fetch 模式设为 `Page_fetch::IF_IN_POOL_OR_WATCH`：

```
1118:1120:storage/innobase/btr/btr0cur.cc
      // notre: 对于undo purge, purge sec index时会用ibuf缓存del op, 并在page_hash设置watch
      case BTR_DELETE_OP:
        ut_ad(fetch == Page_fetch::IF_IN_POOL_OR_WATCH);
```

`IF_IN_POOL_OR_WATCH` 的语义是：页在 buffer pool 就直接用；不在就放一个 watch 哨兵占位。这个动作由 `Buf_fetch::is_on_watch`（buf0buf.cc:3797）完成，它调用 `buf_pool_watch_set`（buf0buf.cc:2926）。`buf_pool_watch_set` 从 `buf_pool->watch[]` 数组取空闲哨兵，把状态从 `BUF_BLOCK_POOL_WATCH` 改成 `BUF_BLOCK_ZIP_PAGE`，设置 page_id、`buf_fix_count=1`，插入 page_hash。注释明确说"this function will be called only by the purge thread"。

哨兵放好后 purge 不读盘，而是把删除操作缓冲到 change buffer：

```
1124:1141:storage/innobase/btr/btr0cur.cc
        if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
          cursor->flag = BTR_CUR_DELETE_REF;
        } else if (ibuf_insert(IBUF_OP_DELETE, tuple, index, page_id, page_size,
                               cursor->thr)) {
          cursor->flag = BTR_CUR_DELETE_IBUF;
        } else {
          buf_pool_watch_unset(page_id);
          break;
        }

        buf_pool_watch_unset(page_id);
        goto func_exit;
```

先 `row_purge_poss_sec` 判断能否安全删除，可删则 `ibuf_insert(IBUF_OP_DELETE)` 缓冲到 change buffer，成功就 `watch_unset` 解除哨兵；缓冲失败也 `watch_unset` 然后 break 走实际读盘。后续页被读入时 change buffer merge 删除，`buf_pool_watch_remove` 替换哨兵。

**哨兵状态机**：`buf_pool->watch[]` 是预分配的 `buf_page_t` 数组（`BUF_POOL_WATCH_SIZE` 个，等于最大 purge 线程数）。`BUF_BLOCK_POOL_WATCH`（空闲态）——不在 page_hash、无 page_id、`zip.data==nullptr`、`buf_fix_count==0`。`BUF_BLOCK_ZIP_PAGE`（激活态）——在 page_hash 占位、有 page_id、`buf_fix_count>=1`，但 `zip.data==nullptr`（伪装压缩页无数据）。转换在 `buf_pool_watch_set`（POOL_WATCH→ZIP_PAGE 入 hash）与 `buf_pool_watch_remove`（buf0buf.cc:3028，ZIP_PAGE→POOL_WATCH 出 hash）。

**`buf0buf.ic:350` 的 `BUF_BLOCK_POOL_WATCH` 必须 `ut_error`**：`buf_page_in_file`（buf0buf.ic:346）判断一个 buf_page_t 是否代表真实文件页缓存。空闲哨兵不在任何 hash/list，正常路径拿不到，流到这里是 use-after-free 类 bug，`ut_error` 防御暴露；它无 frame/zip.data，当真实页操作必崩；而激活态 ZIP_PAGE 故意能通过 `buf_page_in_file`（在 hash 会被查到），区分激活哨兵 vs 真实压缩页靠专门的 `buf_pool_watch_is_sentinel`（buf0buf.cc:2891，检查地址在 watch[] 范围 + zip.data==null）。

### Purge 执行流程

【待补充】本章应覆盖：

- purge 线程模型（`srv_purge_coordinator` + worker threads，`innodb_purge_threads`）
- `trx_purge` 主循环（trx0purge.cc:2466）：clone_oldest_view → attach_undo_recs → worker 并行 purge
- `purge_iter_t`（iter/limit/done 三个指针，生产者消费者模型）
- `trx_purge_attach_undo_recs`（批次读取 undo rec 分配给 worker）
- `row_purge`（row0purge.cc:1167）单条 undo record 的 purge 入口
- `row_purge_record_func`：parse → remove_clust / remove_sec / upd_exist_or_extern
- `row_purge_poss_sec`：判断二级索引记录能否安全删除（`row_vers_old_has_index_entry`）
- history list truncate（`trx_purge_truncate`）
- purge 滞后（`innodb_max_purge_lag` / `innodb_max_purge_lag_delay`）

---

## Undo 表空间管理

【待补充】本章应覆盖：

- undo tablespace 的物理组织（8.0.30 隐式 2 个 + 用户可建）
- `undo::spaces` 与 `Truncate` 机制（`innodb_undo_log_truncate`）
- undo tablespace truncate 流程（`trx_purge_truncate_undo_spaces`）
- undo 大小增长与收缩
- `undo::ddl_mutex` 与 truncate 同步

---

## DDL 与 Undo

DROP TABLE 和 TRUNCATE TABLE **不记录用户数据的 undo log**——它们不像 `DELETE FROM t` 那样为每一行产生 undo record。它们保证原子性和可恢复性靠的是 DDL log 和数据字典元数据的 undo。

### 为什么不记数据 undo

DDL 是隐式提交的，DDL 语句本身是一个原子事务，用户层面不可回滚。DROP/TRUNCATE 是物理操作（删表空间文件、free btree root），不是逐行 delete，没有逐行 undo 的产生点。被删或被清空的表数据不需要 MVCC 旧版本——表都没了或被重建了，没有 read view 会访问旧数据。

### DDL log 机制（8.0 原子 DDL）

InnoDB 8.0 引入 `mysql.innodb_ddl_log` 表和 `Log_DDL` 类（log0ddl.cc），配合 `HTON_SUPPORTS_ATOMIC_DDL` 实现原子 DDL。DROP TABLE 在 `row_drop_table_for_mysql` 中写入三类 DDL log：

```
4028:4127:storage/innobase/row/row0mysql.cc
  if (!table->is_temporary() && !file_per_table) {
      err = log_ddl->write_free_tree_log(trx, index, true);    // 释放索引 btree（共享表空间的表）
  }
  ...
  if (!is_temp) {
    log_ddl->write_drop_log(trx, table_id);                     // 记录 drop（table_id，清理 dict cache）
  }
  ...
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);  // 删除 .ibd 文件
```

DDL log 通过 `DDL_Log_Table::insert` 向 `mysql.innodb_ddl_log` 表 insert 记录。`mysql.innodb_ddl_log` 是普通 InnoDB 表，insert 产生 undo（insert 的 undo，记录如何 delete 这条 ddl_log 记录）+ redo。注意这个 undo 不是为了"回滚 DROP TABLE 的数据"，而是为了 DDL 事务失败时回滚对 `innodb_ddl_log` 表本身的修改——这是原子 DDL 的元数据层面保证。

除了 DDL log，DROP/TRUNCATE 还修改 8.0 的数据字典（`dd::Table` 等存储在 InnoDB 系统表 `mysql.tables` 中）。这些元数据修改同样通过普通事务执行，产生 undo + redo。

### DDL log 生命周期

> DDL log 生命周期（执行期只写日志 / 提交后 `post_ddl` replay / 崩溃恢复 `recover→replay_all`）与原子 DDL 本质，详见 [ddl.md](ddl.md)。

### TRUNCATE TABLE：rename + drop + create

> TRUNCATE = rename + drop + create（`innobase_truncate::truncate` ha_innodb.cc:14558），完整机制详见 [ddl.md](ddl.md)。

---

## GTID 持久化与 Undo

InnoDB 的 GTID 持久化与 undo log 紧密耦合，有三条持久化路径：binlog 文件（Gtid_log_event）/ binlog rotate 刷表 / InnoDB undo log 异步刷表。

**InnoDB GTID 持久化**：事务 commit 阶段同时将 GTID 写入 undo log header（随 redo 持久化，保证崩溃不丢）+ 写入内存 list（后台线程异步刷 `mysql.gtid_executed` 表）。后台线程从内存 list 读取 GTID 刷表，不从 undo log 解析；undo log 只在崩溃恢复时解析（恢复未刷表的 GTID）。

**Clone_persist_gtid 双 buffer**：`m_gtids[2]` 奇偶切换，事务写 active list，后台线程读 flush list，避免读写冲突。

**purge 与 GTID 的协同**：purge 边界会被 GTID 持久化进度压低。`clone_oldest_view` 末尾调用 `view->reduce_low_limit(gtid_persistor.get_oldest_trx_no())`，若还有事务的 GTID 未持久化到 `mysql.gtid_executed` 表，它的 trx_no 不能被 purge 越过——因为 purge 会删 undo log，而 GTID 持久化依赖从 undo log header 读取 GTID。

【待补充】GTID 持久化的完整源码路径（`Clone_persist_gtid`、`Gtid_set`、刷表线程模型）

---

## Misc

### purge 与 commit order 的关系

事务 commit 阶段分配 `trx->no`（`trx_add_to_serialisation_list`，trx0trx.cc:1501 调 `trx_sys_allocate_trx_no`，在 serialisation_mutex 下单调递增），决定 purge 顺序与 MVCC read view 可见性。`binlog_order_commits=ON` 时 `trx->no` 分配序 == binlog 序；OFF 时各线程抢 mutex 分配顺序不保证。Clone/一致性快照依赖 `trx->no` 序 == binlog 序取一致 snapshot。

### purge lag 限流

`innodb_max_purge_lag` 控制 history list 过长时对 DML 延迟注入（`trx_purge_dml_delay`），`innodb_max_purge_lag_delay` 限制单次最大延迟。

【待补充】其他 undo 相关的杂项知识点

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `trx0purge.cc:2466` | `trx_purge`：purge 批次主流程入口 |
| `trx0purge.cc:2276` | purge 边界检查 `iter.trx_no >= view.low_limit_no()` |
| `trx0purge.h:1041-1091` | `purge_iter_t`（trx_no/undo_no）+ iter/limit 指针 |
| `read0read.cc:718` | `MVCC::clone_oldest_view`：purge 克隆最老 read view |
| `read0read.cc:448` | `ReadView::prepare`：`m_low_limit_no = serialisation_min_trx_no` |
| `read0read.cc:128` | FACT C 证明：purge view 最老 → 边界安全 |
| `read0types.h:307` | `m_low_limit_no` 语义注释 |
| `trx0sys.ic:270` | `trx_get_serialisation_min_trx_no` |
| `trx0sys.h:524` | `serialisation_min_trx_no` 原子变量定义 |
| `trx0trx.cc:1501` | `trx_add_to_serialisation_list`：分配 trx->no 入 serialisation_list |
| `trx0trx.cc:1527` | `trx_erase_from_serialisation_list_low`：维护 serialisation_min_trx_no |
| `buf0buf.ic:346` | `buf_page_in_file`：BUF_BLOCK_POOL_WATCH → ut_error |
| `buf0buf.cc:2926` | `buf_pool_watch_set`：purge 插哨兵（POOL_WATCH→ZIP_PAGE） |
| `buf0buf.cc:3028` | `buf_pool_watch_remove`：哨兵替换（ZIP_PAGE→POOL_WATCH） |
| `buf0buf.cc:2891` | `buf_pool_watch_is_sentinel`：区分激活哨兵 vs 真实压缩页 |
| `buf0buf.cc:3797` | `Buf_fetch::is_on_watch`：IF_IN_POOL_OR_WATCH 模式调 watch_set |
| `btr0cur.cc:1118` | `BTR_DELETE_OP` + `IF_IN_POOL_OR_WATCH`：purge 删 sec index 走 watch |
| `btr0cur.cc:1124` | `row_purge_poss_sec` + `ibuf_insert(IBUF_OP_DELETE)` |
| `row0purge.cc:1167` | `row_purge`：单条 undo record purge 入口 |
| `row0mysql.cc:3771` | `row_drop_table_for_mysql`：DROP TABLE InnoDB 侧实现（只写 DDL log，详见 ddl.md） |
| — | 原子 DDL / DDL log / TRUNCATE / btr_free 相关条目全部移至 [ddl.md](ddl.md) |
