# InnoDB 行读取：row_search_mvcc 与游标推进

> 基于 MySQL 8.0.39 源码。承接 `server/handler.md` 的 handler 接口，讲 **InnoDB 侧**如何用持久游标（pcur）定位与推进、逐行返回记录。
>
> **边界**：SQL 层如何调 handler 见 [`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)；handler 接口语义（`position`/`ref`/`rnd_pos`）见 [`../server/handler.md`](../server/handler.md)；可见性判断（read view、版本回溯）见 [`mvcc.md`](mvcc.md)。

## 目录

- [概述](#概述)
- [从 handler 到 row_search_mvcc](#从-handler-到-row_search_mvcc)
- [direction 与游标推进](#direction-与游标推进)
- [need_to_process](#need_to_process)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

SQL 层每次 `iterator->Read()` 穿过 handler 进入 `row_search_mvcc`，后者靠**持久游标（pcur）**定位 + 推进，每次返回一行。

### 与 mvcc.md 的边界

- `mvcc.md`：讲**可见性**——read view、版本回溯、读哪个版本
- 本文：讲**执行流程**——怎么走到游标、怎么推进、位置失效怎么恢复

---

## 从 handler 到 row_search_mvcc

```
（SQL 层）TableScanIterator::Read
  → table->file->ha_rnd_next(m_record)      handler.cc:2969     ← server/InnoDB 分界面
    → ha_innobase::rnd_next                 ha_innodb.cc:10833
      → general_fetch(buf, ROW_SEL_NEXT, 0) ha_innodb.cc:10530
        → row_search_mvcc(...)              ha_innodb.cc:10558
```

索引扫描同理，`ha_rnd_next` 换成 `ha_index_next`（`index_next` → `general_fetch`）。

---

## direction 与游标推进

### direction 是什么

`row_search_mvcc` 的参数，表示**本次调用**是"打开游标(0)"还是"基于已有游标续读(`ROW_SEL_NEXT`=1 / `ROW_SEL_PREV`=-1)"。

它是**调用级**的，不是"行级"的，也**不是每定位一行就置 0**：

- **打开游标**：`direction=0`，只在扫描的第一次（`index_first`/`index_read`）出现，`pcur->open` 定位。`index_read` 里 `row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)`（`ha_innodb.cc:10304`）
- **续读**：之后每次 `rnd_next`/`index_next` 都传 `ROW_SEL_NEXT`（`general_fetch`，`ha_innodb.cc:10619`/`10853`）

`rnd_next` 内部用 `m_start_of_scan` 标志区分：首次 `index_first`，之后 `general_fetch(ROW_SEL_NEXT)`（`ha_innodb.cc:10844-10854`）。

### direction 与锁定读/半一致性读无关

它只回答"是否打开游标"。半一致性读的"第一次读"和"第二次重读"，direction 取决于该行在扫描中的位置（扫描首条则第一次 0、重读 NEXT；扫描中段则两次都 NEXT）。

重读"同一行"靠的**不是 direction**，而是：半一致性读没走 `next_rec`（游标不推进）+ 第二次进来 `row_read_type==DID` 且 `need_to_process==false` 时不 `goto next_rec`。

### 游标推进

`next_rec` 标签（`row0sel.cc:5803`）→ PHASE 5 → `if (moves_up) pcur->move_to_next(&mtr); else pcur->move_to_prev(...)`（`row0sel.cc:5908-5936`）→ `goto rec_loop` 回到循环头处理下一条。

半一致性读路径（`row0sel.cc:5288-5314`）**不经过** `next_rec`，`break` 后落到 `normal_return` 返回，游标停在原记录（`store_position` 已存），故下次读仍在同一行——这是"半一致性读重读同行"的机制基础。

---

## need_to_process

`sel_restore_position_for_mysql`（`row0sel.cc:3406`）的返回值：调 `pcur->restore_position` 恢复之前 `store_position` 保存的位置，返回 true 表示"恢复后需重新处理游标现在指向的记录"（原记录被删、位置失效时，按 `moves_up` 调 `move_to_next` 前移）。

在 `row0sel.cc:4887-4906`：

- `need_to_process == true` 且 `row_read_type == DID`：把 DID 复位成 TRY（记录已删，之前的半一致性读作废）
- `need_to_process == false` 且 `row_read_type == DID`：不 `goto next_rec`，继续处理同一行（半一致性读重读加锁的分支）

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/handler/ha_innodb.cc:10833` | `ha_innobase::rnd_next` |
| `storage/innobase/handler/ha_innodb.cc:10530` | `general_fetch`（传 direction） |
| `storage/innobase/handler/ha_innodb.cc:10304` | `index_read` 里 direction=0 |
| `storage/innobase/handler/ha_innodb.cc:10844` | `m_start_of_scan` 区分首次/续读 |
| `storage/innobase/row/row0sel.cc:3406` | `sel_restore_position_for_mysql`（need_to_process） |
| `storage/innobase/row/row0sel.cc:4887` | need_to_process 与 DID 的分支 |
| `storage/innobase/row/row0sel.cc:5288` | 半一致性读路径（不经 next_rec） |
| `storage/innobase/row/row0sel.cc:5803` | `next_rec` 标签（游标推进入口） |
| `storage/innobase/row/row0sel.cc:5908` | `pcur->move_to_next` / `move_to_prev` |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Locking and Transaction Model*（半一致性读）
- *MySQL 8.0 Reference Manual → Consistent Nonlocking Reads*

**相关文档**
- 可见性判断见 [`mvcc.md`](mvcc.md)
- handler 接口见 [`../server/handler.md`](../server/handler.md)
