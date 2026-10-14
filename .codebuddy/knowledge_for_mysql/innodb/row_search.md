# InnoDB 行读取：row_search_mvcc 与游标推进

> 基于 MySQL 8.0.39 源码。承接 `server/handler.md` 的 handler 接口，讲 **InnoDB 侧**如何用持久游标（pcur）定位与推进、逐行返回记录。
>
> **边界**：SQL 层如何调 handler 见 [`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)；handler 接口语义（`position`/`ref`/`rnd_pos`）见 [`../server/handler.md`](../server/handler.md)；可见性判断（read view、版本回溯）见 [`mvcc.md`](mvcc.md)。本篇只讲 **InnoDB 侧**：拿到 handler 调用后如何定位、推进、转换、返回。

- [★ 怎么读：跨篇学习路径（先读哪篇、卡住跳哪篇）](#怎么读跨篇学习路径)
- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [主线总览：一次查询怎么走完 PHASE 1–5](#主线总览一次查询怎么走完-phase-1-5)
    - [从 handler 到 row_search_mvcc：首读与续读两条路](#从-handler-到-row_search_mvcc首读与续读两条路)
    - [行格式：rec_t 磁盘行与 record[0] 的转换](#行格式rec_t-磁盘行与-record0-的转换)
  - 行为与游标
    - [不同隔离级别下的表现](#不同隔离级别下的表现)
    - [direction 与游标推进](#direction-与游标推进)
    - [need_to_process](#need_to_process)
  - 读路径
    - [MVCC 可见性：快照读与二级索引回表](#mvcc-可见性快照读与二级索引回表)
    - [B-tree 定位：btr_cur_search_to_nth_level](#b-tree-定位btr_cur_search_to_nth_level)
    - [持久游标：store_position / restore_position](#持久游标store_position--restore_position)
  - 加锁
    - [加锁：sel_set_rec_lock 与隐式锁转显式](#加锁sel_set_rec_lock-与隐式锁转显式)
- [参考](#参考)

---

## 怎么读：跨篇学习路径

`row_search_mvcc` 横跨多篇，**单读本篇不够**。★ **建议先从 `index/btr.md` 入手**（理由见下），再回本篇：

```
⓪ 最小前置    data_repr.md「四、InnoDB 物理层：rec_t 与 offsets」
                 ⇒ btr 操作的就是 rec_t / page，先认识它长什么样
                       ↓
① B-tree 概念  btr.md「是什么 / 用途 / 设计思想与权衡」+「主链路与整体组织」
                       ↓
② 游标体系 ★   btr.md「游标体系：btr_cur_t 与 btr_pcur_t」
③ 搜索 ★      btr.md「搜索 btr_cur_search_to_nth_level」+「页内二分」
                       ↓  ⇒ 到这一步，本篇 PHASE 3（定位）/ 5（推进）已通
④ 本体骨架    本篇「主线总览 PHASE 1–5」+「函数签名与入参」
                 → 配源码 row0sel.cc 的 row_search_mvcc 逐 PHASE 读
                       ↓
                 ┌── 读到哪个 PHASE 卡住就跳哪篇（见下表），补完回来 ──┐
                       ↓
⑤ 回归        重读本篇「主线总览」，此时应能看懂图上每一条分支
```

**为什么先学 btr 而不是先啃本篇**：`row_search_mvcc` 是个**交叉点**——同时牵扯格式（4 种表示）、锁、MVCC、server 分界，天然不适合当入口。B-tree 则**自足**：只关心"记录怎么组织、怎么找到"，不依赖格式转换与 handler 分界 ⇒ 从它入手明显更轻松。

**★ btr.md 有一半不该先学**（3155 行的分布）：

| 部分 | 体量 | 先学？ |
|---|---|---|
| 概述 + 理论基础 + 主链路 | ~310 行 | ✅ |
| 游标体系 ★ | ~497 行 | ✅ 核心 |
| 搜索 `btr_cur_search_to_nth_level` ★ | ~433 行 | ✅ 核心 |
| 中间页 / 页内二分 | ~218 行 | ✅ |
| 索引锁 / AHI / 行数估算 | ~370 行 | ⚠️ 按需 |
| **插入与分裂 / 删除与合并 / 更新 / 批量构建 / 生命周期** | **~1194 行** | ❌ **别先学**（比本篇更难） |

⇒ 学完 ①②③ 再回本篇，PHASE 3 / 5 就通了；DML 那部分留到真正需要时再看。

**第 ④ 步的跳转表**：

| 卡在 | 跳去读 |
|---|---|
| PHASE 2 AHI 捷径 | [`index/ahi.md`](../index/ahi.md) |
| PHASE 3 打开 / 恢复游标 | [`index/btr.md`](../index/btr.md)（`open_no_init`、`move_to_*`、`restore_position`） |
| PHASE 4 可见性 / 半一致读 | [`mvcc.md`](mvcc.md) |
| PHASE 4 回表 | 本篇「Row_sel_get_clust_rec_for_mysql」 |
| PHASE 4 加锁 | 本篇「sel_set_rec_lock 与隐式锁转显式」 |
| PHASE 4 字段转换 | 本篇「row_sel_field_store_in_mysql_format_func」+ [`data_repr.md`](../infra/structure/data_repr.md) 写方向（对照看） |

**★ 游标契约：本篇的前提（第 ② 步唯一必须先拿下的东西）**

`btr_pcur_t`（持久游标）必须懂——本篇的 PHASE 3（打开 vs 恢复游标）与 PHASE 5（推进）全程在用（`open_no_init` / `store_position` / `restore_position` / `move_to_next_user_rec`）：

- `btr_cur_t` = **mtr 生命周期内**的树游标（mtr commit 即失效）
- `btr_pcur_t` = **跨 mtr 的持久游标**（靠 `store_position` 存下位置，下次 `restore_position` 恢复）

⇒ 不搞懂这个区分，"为什么游标要 store/restore"、"为什么跨 mtr 后位置会失效"就完全无法理解。它也是"持久"二字的本义——`row_search_mvcc` 每次调用各持一个 mtr，靠它才能在多次调用间保持扫描位置。

⇒ 三个提醒：

1. **第 ① 步别跳过**——本篇一半代码在做格式转换，不先建立四种表示（rec_t / dtuple / record[0] / 引擎键）的概念，后面读不动。
2. **字段转换必须两个方向对照看**——读方向那个最隐晦的 `len & 1` 补半个空格，根因在写方向的 `col_len &= ~1`。只看一边永远想不通。
3. **第 ②⁵ 步别跳过也别贪多**——只读游标体系那一节，不要试图先读完 btr.md。

## 概述

### 是什么

SQL 层每次 `iterator->Read()` 穿过 handler 进入 `row_search_mvcc`，后者靠**持久游标（pcur）**定位 + 推进，每次返回一行。

### 与 mvcc.md 的边界

- `mvcc.md`：讲**可见性**——read view、版本回溯、读哪个版本
- 本文：讲**执行流程**——怎么走到游标、怎么推进、位置失效怎么恢复

---

## 理论基础

### row_search_mvcc：为什么是"怪兽函数"

`row_search_mvcc` 是 InnoDB 里最复杂的函数之一。`row0sel.h` 里三个并列函数的注释本身就是"复杂度分层"的最强证据：

- `row_search_no_mvcc`：*"Function is for temporary tables that are not shared across connections and so lot of complexity is reduced especially locking and transaction related."* —— 临时表/内部表走这条路，**砍掉了锁和事务复杂度**
- `row_search_mvcc`：*"mainly used for tables that are shared accorss connection and so it employs technique that can help re-construct the rows that transaction is suppose to see."* —— 共享表要走这条路

同一个函数体内，按 PHASE 注释处理至少这些模式：

| 模式 | 判定 |
|---|---|
| 唯一等值点查 | `unique_search` 标志 |
| 快照一致读（LOCK_NONE） | `select_lock_type == LOCK_NONE` → `trx_assign_read_view` |
| 加锁读 FOR UPDATE | `sel_set_rec_lock` |
| 二级索引扫描 + 回表 | `requires_clust_rec` 标签 |
| 全表扫 | `open_at_side(PAGE_CUR_G)` |
| 半一致读 | `use_semi_consistent` |
| 空间索引 R-tree | `dict_index_is_spatial` |
| AHI 捷径 | `row_sel_try_search_shortcut_for_mysql` |
| ICP | `row_search_idx_cond_check` |
| 行预取 | `can_prefetch_records` |

**为什么挤在一个函数里**：

1. **刻意分层**——"共享表 vs 临时表"已拆成 `row_search_mvcc` vs `row_search_no_mvcc`。共享表把 MVCC、行锁、二级索引回表、外存 BLOB、空间索引、预取、AHI 全堆在一起，才成了怪兽函数
2. **历史包袱**——函数头注释保留着 1997 年 Heikki Tuuri 的原味（"accorss" 拼写错误），主循环是 `goto` 标签串起的状态机（`rec_loop` / `next_rec` / `requires_clust_rec` / `lock_wait_or_error`），是长期增量演化的产物而非一次性设计
3. **为什么不能拆成多个小函数**——主循环的状态（`pcur` 位置、`mtr` 持有的 page latch、`did_semi_consistent_read`、`offsets` 归属的 rec 是聚簇还是二级索引、`moves_up` 方向）相互纠缠，尤其"恢复游标后是否要重处理当前记录"（`need_to_process`）跨越整个"取行→判定→加锁/回溯→返回→续读"流程，拆函数必然要引入新的状态对象

**★ 从 handler 入口看：`row_search_mvcc` 只有两个调用点，两者用 `direction` 区分**（这是理解整个函数的第一刀）：

```cpp
    // ① 首读：ha_innobase::index_read()（ha_innodb.cc:10293）
    ret = row_search_mvcc(buf, mode /* 具体值 */, m_prebuilt, match_mode, 0);
    //                                                                    ↑ direction == 0

    // ② 续读：ha_innobase::general_fetch()（ha_innodb.cc:10558）
    ret = row_search_mvcc(buf, PAGE_CUR_UNSUPP, m_prebuilt, match_mode, direction);
    //                         ↑ 恒为 UNSUPP        ↑ ROW_SEL_NEXT(1) / ROW_SEL_PREV(2)
```

> ★★ **`general_fetch()` 硬传 `PAGE_CUR_UNSUPP`**，真正的 mode 是在 `row_search_mvcc` 里从持久游标**回读**的：
>
> ```cpp
>     mode = pcur->m_search_mode;    // row0sel.cc:4625
> ```
>
> 而 `m_search_mode` 是在 `btr_pcur_t::restore_position()` 末尾被还原的（`m_search_mode = old_mode;`）。**这是"续读不需要再传搜索条件"的实现机制**——位置信息全在 `pcur` 里。

**★ 全部 handler 入口的映射表**（这是本文少有的"一图看全"）：

| server 接口 | InnoDB 实现 | 传的 mode | direction | match_mode |
|---|---|---|---|---|
| `index_read(key, HA_READ_KEY_EXACT)` | `index_read` | `PAGE_CUR_GE` | 0 | `ROW_SEL_EXACT` |
| `index_read(key, HA_READ_KEY_OR_NEXT)` | `index_read` | `PAGE_CUR_GE` | 0 | 0 |
| `index_read(key, HA_READ_AFTER_KEY)` | `index_read` | `PAGE_CUR_G` | 0 | 0 |
| `index_read(key, HA_READ_BEFORE_KEY)` | `index_read` | `PAGE_CUR_L` | 0 | 0 |
| `index_read(key, HA_READ_KEY_OR_PREV)` | `index_read` | `PAGE_CUR_LE` | 0 | 0 |
| `index_read_last(key)` | → `index_read(..., HA_READ_PREFIX_LAST)` | `PAGE_CUR_LE` | 0 | `ROW_SEL_EXACT_PREFIX` |
| `index_first()` | → `index_read(nullptr, 0, HA_READ_AFTER_KEY)` | `PAGE_CUR_G` | 0 | 0 |
| `index_last()` | → `index_read(nullptr, 0, HA_READ_BEFORE_KEY)` | `PAGE_CUR_L` | 0 | 0 |
| `rnd_next()` 首行 | → `index_first()` | `PAGE_CUR_G` | 0 | 0 |
| `rnd_next()` 后续 | → `general_fetch(ROW_SEL_NEXT)` | UNSUPP | 1 | 0 |
| `rnd_pos(pos)` | → `index_read(pos, ref_length, HA_READ_KEY_EXACT)` | `PAGE_CUR_GE` | 0 | `ROW_SEL_EXACT` |
| `index_next()` | → `general_fetch(ROW_SEL_NEXT, 0)` | UNSUPP | 1 | 0 |
| `index_next_same()` | → `general_fetch(ROW_SEL_NEXT, m_last_match_mode)` | UNSUPP | 1 | **复用首读的** |
| `index_prev()` | → `general_fetch(ROW_SEL_PREV, 0)` | UNSUPP | 2 | 0 |

```cpp
/** Converts a search mode flag understood by MySQL to a flag understood
 by InnoDB. */
page_cur_mode_t convert_search_mode_to_innobase(ha_rkey_function find_flag) {
  switch (find_flag) {
    case HA_READ_KEY_EXACT:
      /* this does not require the index to be UNIQUE */
    case HA_READ_KEY_OR_NEXT:
      return (PAGE_CUR_GE);          // ★ 等值点查和 ">=" 都落到 GE
    case HA_READ_AFTER_KEY:
      return (PAGE_CUR_G);           // ★ index_first() 用的就是这个
    case HA_READ_BEFORE_KEY:
      return (PAGE_CUR_L);           // ★ index_last() 用的就是这个
    case HA_READ_KEY_OR_PREV:
    case HA_READ_PREFIX_LAST:
    case HA_READ_PREFIX_LAST_OR_PREV:
      return (PAGE_CUR_LE);
    case HA_READ_MBR_CONTAIN:   return (PAGE_CUR_CONTAIN);
    case HA_READ_MBR_INTERSECT: return (PAGE_CUR_INTERSECT);
    case HA_READ_MBR_WITHIN:    return (PAGE_CUR_WITHIN);
    case HA_READ_MBR_DISJOINT:  return (PAGE_CUR_DISJOINT);
    case HA_READ_MBR_EQUAL:     return (PAGE_CUR_MBR_EQUAL);
    case HA_READ_PREFIX:        return (PAGE_CUR_UNSUPP);
    case HA_READ_INVALID:       return (PAGE_CUR_UNSUPP);
      /* do not use "default:" in order to produce a gcc warning ... */
  }
  my_error(ER_CHECK_NOT_IMPLEMENTED, MYF(0), "this functionality");
  return (PAGE_CUR_UNSUPP);
}
```

> ⚠️ **不存在 `convert_search_mode_to_page_cur_mode()`**——真名 `convert_search_mode_to_innobase()`。
> ★★ **`HA_READ_KEY_EXACT` 没有独立的"精确模式"**——换算结果是 `PAGE_CUR_GE`，"精确"靠 `match_mode = ROW_SEL_EXACT` + 后续 `cmp_dtuple_rec()` 二次校验实现。同理 `HA_READ_PREFIX_LAST` = `PAGE_CUR_LE` + `ROW_SEL_EXACT_PREFIX`。

**★ 全表扫描本质就是聚簇索引的正向扫描**（三条证据）：

```cpp
int ha_innobase::rnd_init(bool scan) {
  int err = change_active_index(table_share->primary_key);   // ★① 切到主键索引
  if (!scan) {
    m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;         // ★ rnd_pos 场景禁用 semi-consistent
  }
  m_start_of_scan = true;
  return err;
}

int ha_innobase::rnd_next(uchar *buf) {
  if (m_start_of_scan) {
    error = index_first(buf);            // ★② 首行 = PAGE_CUR_G 定位
    if (error == HA_ERR_KEY_NOT_FOUND) error = HA_ERR_END_OF_FILE;
    m_start_of_scan = false;
  } else {
    error = general_fetch(buf, ROW_SEL_NEXT, 0);   // ★③ 后续 = ROW_SEL_NEXT 推进
  }
  return error;
}

int ha_innobase::rnd_pos(uchar *buf, uchar *pos) {
  int error = index_read(buf, pos, ref_length, HA_READ_KEY_EXACT);  // ★ 按主键点查
  ...
}
```

> ⚠️ **`index_end()` 的注释还写着 "Currently does nothing"，但代码已经不是了**：现在会释放 `index->last_sel_cur`、`active_index = MAX_KEY`、`in_range_check_pushed_down = false`、`m_ds_mrr.dsmrr_close()`。
> ★ `index_init(keynr)` **只有一句** `return change_active_index(keynr)`——真正的绑定与模板编译都在后者。

**★ 错误码翻译的首读/续读差异**（常被忽略）：

```cpp
  // index_read()：定位阶段越界也报 KEY_NOT_FOUND
    case DB_RECORD_NOT_FOUND:  error = HA_ERR_KEY_NOT_FOUND; break;
    case DB_END_OF_INDEX:      error = HA_ERR_KEY_NOT_FOUND; break;

  // general_fetch()：一律 EOF
    case DB_RECORD_NOT_FOUND:  error = HA_ERR_END_OF_FILE; break;
    case DB_END_OF_INDEX:      error = HA_ERR_END_OF_FILE; break;
```

而 `index_first()` / `index_last()` 再把它翻译成 EOF：

```cpp
  int error = index_read(buf, nullptr, 0, HA_READ_AFTER_KEY);
  /* MySQL does not seem to allow this to return HA_ERR_KEY_NOT_FOUND */
  if (error == HA_ERR_KEY_NOT_FOUND) {
    error = HA_ERR_END_OF_FILE;
  }
```

> ★ **PHASE 5 的推进也区分两种越界**：`match_mode != 0` → `DB_RECORD_NOT_FOUND`；否则 → `DB_END_OF_INDEX`。对应"等值/前缀扫描越界"与"范围/全扫描越界"两种语义。

### 主线总览：一次查询怎么走完 PHASE 1-5

上面讲了"为什么挤在一个函数里"，这里先给骨架——**看清主线再看代码**（各 PHASE 在下面分节剖析）：

```
        【入口】首读: index_read / index_first  → mode = PAGE_CUR_GE/G/L
               续读: general_fetch             → mode = PAGE_CUR_UNSUPP
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ row_search_mvcc(buf, mode, prebuilt, match_mode, direction) │
  └─────────────────────────────────────────────────────────────┘
     │
     ├─ PHASE 1  记录缓冲 / 预取缓存    命中缓存就直接返回给 server
     │           row_sel_dequeue_cached_row_for_mysql
     │
     ├─ PHASE 2  AHI 自适应哈希捷径     唯一键 + 聚簇 + 一致性读才走
     │           row_sel_try_search_shortcut_for_mysql（三态）
     │
     ├─ PHASE 3  ★ 打开 vs 恢复游标     ★ 由入口传进来的 mode 决定
     │           mode 具体   → open（open_no_init 从 root 下降）
     │           mode UNSUPP → restore（sel_restore_position_for_mysql）
     │
     ├─ PHASE 4  rec_loop 主循环（核心）
     │           取一条记录 →
     │             ├─ ICP 判定 row_search_idx_cond_check
     │             │    不满足 → next_rec（★ 省掉一次回表）
     │             ├─ 当前读：sel_set_rec_lock 加锁
     │             │  快照读：lock_*_cons_read_sees 判可见性
     │             ├─ 二级索引 → 回表 Row_sel_get_clust_rec_for_mysql
     │             ├─ 不可见 → undo 回溯 row_vers_build_for_consistent_read
     │             └─ 转换 row_sel_store_mysql_rec
     │                     → row_sel_field_store_in_mysql_format_func
     │
     ├─ PHASE 5  next_rec 推进
     │           move_to_next_user_rec（页内 / 跨页）→ 回到 PHASE 4
     │
     └─ 返回 DB_SUCCESS / DB_RECORD_NOT_FOUND / DB_END_OF_INDEX
              → general_fetch 映射为 HA_ERR_END_OF_FILE
```

两个关键接缝：

1. **入口的 `mode` 决定 PHASE 3 走 open 还是 restore** —— 见上文「首读与续读」。
2. **PHASE 4 里 ICP 不满足就跳 `next_rec`** —— 这是整个函数"省一次回表"的地方，也是 ICP 收益的来源。

### 函数签名与入参（逐参数）

```cpp
dberr_t row_search_mvcc(byte *buf, page_cur_mode_t mode,
                        row_prebuilt_t *prebuilt, ulint match_mode,
                        const ulint direction)
```

| 参数 | 含义与取值 |
|---|---|
| `buf` | out：MySQL 行格式（server 层 record 布局）的输出缓冲 |
| `mode` | B-tree 搜索模式：`PAGE_CUR_G`(>) / `GE`(>=) / `L`(<) / `LE`(<=) / `CONTAIN`/`WITHIN`(R-tree) |
| `prebuilt` | in/out：总上下文。关键字段：`index`（本次索引）、`search_tuple`（定位键）、`pcur`（持久游标）、`select_lock_type`（LOCK_NONE 快照读 / LOCK_S / LOCK_X）、`mysql_template`（行转换模板）、`fetch_cache`（预取缓存） |
| `match_mode` | `0`=范围扫描；`ROW_SEL_EXACT`=等值点查；`ROW_SEL_EXACT_PREFIX`=前缀匹配 |
| `direction` | `0`=打开/定位游标（首次）；`ROW_SEL_NEXT`=取下一行；`ROW_SEL_PREV`=取上一行。`!=0` 时要求 `pcur` 已 `store_position` 过 |

### PHASE 1：记录缓冲 / 预取缓存（逐行）

```cpp
if (direction == 0) {
  trx->op_info = "starting index read";
  prebuilt->n_rows_fetched = 0;
  prebuilt->n_fetch_cached = 0;
  prebuilt->fetch_cache_first = 0;
  prebuilt->m_end_range = false;
  if (record_buffer != nullptr) record_buffer->reset();
  if (prebuilt->sel_graph == nullptr) row_prebuild_sel_graph(prebuilt);
} else {
  trx->op_info = "fetching rows";
  ...
  if (prebuilt->n_fetch_cached > 0) {
    row_sel_dequeue_cached_row_for_mysql(buf, prebuilt);   // ★ 缓存命中：直接弹出返回
    prebuilt->n_rows_fetched++;
    err = DB_SUCCESS;
    goto func_exit;
  } else if (prebuilt->m_end_range) {
    err = DB_RECORD_NOT_FOUND;
    goto func_exit;
  }
  ...
  mode = pcur->m_search_mode;                              // 沿用游标保存的搜索模式
}
```

- `direction == 0`（首次）：清零计数/缓存/结束标志；`row_prebuild_sel_graph` 构建哑 select 查询图——不是为了执行 Query Graph，而是为了拿到 `que_thr_t` 线程上下文（后续 `lock_table`、`row_mysql_handle_errors` 都要用）
- `direction != 0`（续读）：**缓存命中时直接 `row_sel_dequeue_cached_row_for_mysql` 弹出一行返回，完全不碰 B-tree**——这是预取优化的命中路径
- **为什么要缓存**：主循环每取一行都要「定位 B-tree 页 → 加 page latch → 转行 → 提交 mtr 释放 latch」。若一次 mtr 内能连续转多行放进缓存，之后若干次调用直接从内存弹行，省掉反复 latch/unlatch。`MYSQL_FETCH_CACHE_THRESHOLD = 4`（只有 `n_rows_fetched >= 4` 确认为长范围扫描才启用）、`MYSQL_FETCH_CACHE_SIZE = 8`（缓存上限）

### unique_search 判定（进入 PHASE 2 前）

```cpp
if (match_mode == ROW_SEL_EXACT && dict_index_is_unique(index) &&
    dtuple_get_n_fields(search_tuple) == dict_index_get_n_unique(index) &&
    (index->is_clustered() || !dtuple_contains_null(search_tuple))) {
  unique_search = true;
  ...
}
```

四个条件：等值点查 + 索引唯一 + 键字段数等于唯一列数（给了完整唯一键）+（聚簇索引 或 二级键不含 NULL）。二级唯一索引可能含多个 NULL（NULL != NULL），所以要求键中无 NULL。

### PHASE 2：AHI 自适应哈希捷径

```cpp
if (direction == 0 && unique_search && btr_search_enabled &&
    index->is_clustered() && !prebuilt->templ_contains_blob &&
    !prebuilt->used_in_HANDLER &&
    (prebuilt->mysql_row_len < UNIV_PAGE_SIZE / 8) && !prebuilt->innodb_api) {
  mode = PAGE_CUR_GE;
  if (trx->mysql_n_tables_locked == 0 && !prebuilt->ins_sel_stmt &&
      prebuilt->select_lock_type == LOCK_NONE &&
      trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
      MVCC::is_view_active(trx->read_view)) {
    ...
    switch (row_sel_try_search_shortcut_for_mysql(&rec, ...)) {
      case SEL_FOUND:    row_sel_store_mysql_rec(...); goto func_exit;   // 命中
      case SEL_EXHAUSTED: ... DB_RECORD_NOT_FOUND; goto func_exit;        // 确定无此行
      case SEL_RETRY:    break;                                           // 回退正常流程
    }
  }
}
```

- **外层 8 条件**：首次定位 + 唯一点查 + AHI 开 + 聚簇索引 + 无 BLOB（外存字段需释放 search latch，与 AHI 冲突）+ 非 HANDLER 滚动游标 + 行足够短 + 非 innodb_api
- **内层 5 条件**（快照读前提）：非 INSERT...SELECT + 非内部 select + LOCK_NONE + 隔离级别 > RU + read view 活跃
- `row_sel_try_search_shortcut_for_mysql` 三态：`SEL_FOUND`（AHI 命中，可见且非 delete-mark，直接转换返回）；`SEL_EXHAUSTED`（`up_match < n_fields` 键不存在，或 delete-mark，确定无此行）；`SEL_RETRY`（不可见需要 undo 回溯，或定位到非 user record，回退正常流程）
- ★ **捷径路径不 `store_position`**——AHI 快照读是一次性的，不需要给后续 NEXT 保存位置

### PHASE 3：打开 vs 恢复游标（核心分支）

```cpp
  if (direction != 0) {
    auto need_to_process = sel_restore_position_for_mysql(
        &same_user_rec, BTR_SEARCH_LEAF, pcur, moves_up, &mtr);
    if (need_to_process) {
      /* 恢复后游标已校正到新记录，直接 rec_loop 处理当前记录 */
    } else if (row_read_type != ROW_READ_DID_SEMI_CONSISTENT) {
      goto next_rec;   /* 游标停在"上次已返回的那条"上，先前进 */
    }
  } else if (dtuple_get_n_fields(search_tuple) > 0) {
    pcur->open_no_init(index, search_tuple, mode, BTR_SEARCH_LEAF, 0, &mtr, ...);
    rec = pcur->get_rec();
    /* 降序扫描 + gap 锁：对逻辑后继加 LOCK_GAP（防止 DESC 幻读） */
    ...
  } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_L) {
    pcur->open_at_side(mode == PAGE_CUR_G, index, BTR_SEARCH_LEAF, false, 0, &mtr);
  }
```

- **`direction != 0`（恢复游标）**：`sel_restore_position_for_mysql` 返回 `need_to_process`——true = 恢复后游标已前进到"下一条待处理记录"（原记录被删/换页，函数内部已校正），直接落进 `rec_loop`；false = 游标停在"上次已返回的那条"上，必须 `goto next_rec` 先前进（除非是 DID_SEMI_CONSISTENT 需要悲观重读同一行）
- **`direction == 0 && search_tuple 有字段`（按键定位）**：`open_no_init` 从根下到叶子按 mode 定位；降序扫描（`!moves_up`，定位到 range 上界）时，幻影会从定位点"左边"插入，所以对逻辑后继加 `LOCK_GAP`
- **`direction == 0 && mode == G/L`（全表扫）**：`open_at_side` 打开到索引最左（G）/最右（L）叶子
- **`moves_up` 推导**：`direction == 0` 时 `GE/G/CONTAIN` → true（升序）；`direction == ROW_SEL_NEXT` → true；其余（PREV、或 0 且 L/LE 降序）→ false

**`sql_stat_start` 三态**（在打开游标前）：

```cpp
  if (!prebuilt->sql_stat_start) {
    /* 语句已开始：一致读必须有 read view（防御断言） */
  } else if (prebuilt->select_lock_type == LOCK_NONE) {
    trx_assign_read_view(trx);      // 快照读：分配 read view
    prebuilt->sql_stat_start = false;
  } else {
    err = lock_table(0, index->table, LOCK_S ? LOCK_IS : LOCK_IX, thr);   // 锁定读：加意向锁
    ...
  }
```

### PHASE 4：rec_loop 主循环（逐行，最重要）

**循环入口与中断检查**：

```cpp
rec_loop:
  if (trx_is_interrupted(trx)) {
    if (!spatial_search) pcur->store_position(&mtr);
    err = DB_INTERRUPTED;
    goto normal_return;
  }
  rec = pcur->get_rec();          // ★ 取游标当前指向的记录，本轮的处理对象
```

**infimum / supremum 检查**：

```cpp
  if (page_rec_is_infimum(rec)) { prev_rec = nullptr; goto next_rec; }
  if (page_rec_is_supremum(rec)) {
    // ① end_range 优化：用上一页最后一条 prev_rec 提前判断是否越界（end_loop>=100 节流）
    // ② supremum 加 gap 锁：LOCK_ORDINARY 锁住"页末尾之后的间隙"
    if (set_also_gap_locks && !trx->skip_gap_locks() && select_lock_type != LOCK_NONE && ...)
      sel_set_rec_lock(pcur, rec, index, offsets, ..., LOCK_ORDINARY, thr, &mtr);
    prev_rec = nullptr;
    goto next_rec;                 // supremum 永不进结果
  }
```

- infimum（页内最小哨兵）永不进结果、永不加锁，直接跳过
- supremum（页末尾哨兵）两步：① **end_range 优化**——扫描跨页且 server 下推了 `end_range` 时，用 `prev_rec` 提前判越界避免扫完整棵树；② **加 gap 锁**——对 supremum 加 `LOCK_ORDINARY`（next-key）锁住"页内最后一条之后"的间隙，阻止其他事务在页末尾插入

**页损坏检查**：`rec_get_next_offs` 读记录头里的 next 偏移，落在 `[PAGE_NEW_SUPREMUM, UNIV_PAGE_SIZE - PAGE_DIR)` 之外即页损坏。`srv_force_recovery > 0` 且升序时跳过本页剩余（抢救数据），否则报 `DB_CORRUPTION`。

**offsets 与 match_mode 过滤**：

```cpp
  offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, ...);
  ...
  if (match_mode == ROW_SEL_EXACT) {
    if (0 != cmp_dtuple_rec(search_tuple, rec, index, offsets)) {
      /* 不匹配：锁定读先加 LOCK_GAP（锁住匹配键应当落下的间隙）*/
      sel_set_rec_lock(..., LOCK_GAP, ...);
      pcur->store_position(&mtr);
      pcur->m_rel_pos = BTR_PCUR_BEFORE;    // 这条不匹配记录将来作 index_next 起点
      err = DB_RECORD_NOT_FOUND;
      goto normal_return;
    }
  } else if (match_mode == ROW_SEL_EXACT_PREFIX) { ... /* 前缀同理 */ }
```

- ★ 注释强调：**此处不能信任游标的 `up_match`**（游标可能已 move 过绕回），必须重新 `cmp_dtuple_rec`
- 不匹配时 `m_rel_pos = BTR_PCUR_BEFORE`——这条记录将来可作 `index_next` 的起始，游标语义是"位于它之前"

**锁定读分支**（`select_lock_type != LOCK_NONE`）：

```cpp
    auto row_to_range_relation = row_compare_row_to_range(
        set_also_gap_locks, trx, unique_search, index, clust_index, rec, ...);
    ulint lock_type;
    if (row_can_be_in_range) {
      lock_type = gap_can_intersect_range ? LOCK_ORDINARY : LOCK_REC_NOT_GAP;
    } else {
      lock_type = gap_can_intersect_range ? LOCK_GAP : (DB_RECORD_NOT_FOUND);
    }

    const bool use_semi_consistent =
        row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
        !unique_search && index == clust_index && !trx_is_high_priority(trx);
    err = sel_set_rec_lock(pcur, rec, index, offsets,
                           use_semi_consistent ? SELECT_SKIP_LOCKED : select_mode,
                           select_lock_type, lock_type, thr, &mtr);
```

- **锁类型三选一**（`row_compare_row_to_range` 算出记录与范围的关系）：记录在范围内 + 间隙相交 → `LOCK_ORDINARY`（next-key，默认 RR）；记录在范围内 + 间隙不相交（唯一点查）→ `LOCK_REC_NOT_GAP`；记录不在范围 + 间隙相交 → `LOCK_GAP`
- `use_semi_consistent` 四条件：TRY_SEMI + 非唯一点查 + 聚簇索引 + 非高优先级。启用时用 `SELECT_SKIP_LOCKED`——**遇到被锁记录返回 `DB_SKIP_LOCKED` 而不是阻塞建 WAITING 锁**
- `DB_SKIP_LOCKED` 两分支：显式 `SKIP LOCKED` → `goto next_rec`；半一致读 → `row_sel_build_committed_vers_for_mysql` 用 undo 构建**最新已提交版本**，`old_vers == nullptr`（未提交新插入）→ `next_rec`，否则 `rec = old_vers` 返回已提交版本
- `DB_LOCK_WAIT` → 清空 `new_rec_lock`，`goto lock_wait_or_error`

**快照读分支**（`select_lock_type == LOCK_NONE`）：

```cpp
  } else {
    if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
      /* 读未提交：直接读最新版本，什么都不做 */
    } else if (index == clust_index) {
      if (!lock_clust_rec_cons_read_sees(rec, index, offsets, trx_get_read_view(trx))) {
        row_sel_build_prev_vers_for_mysql(...);   // 不可见 → undo 回溯
        if (old_vers == nullptr) goto next_rec;    // read view 中还不存在
        rec = old_vers;
      }
    } else {
      /* 二级索引：记录不含完整版本链，不可信必须回表 */
      if (!lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH: goto next_rec;          // ICP 不满足，省一次回表
          case ICP_OUT_OF_RANGE: ... goto idx_cond_failed;
          case ICP_MATCH: goto requires_clust_rec;
        }
      }
    }
  }
```

**delete-mark 跳过**：

```cpp
  if (rec_get_deleted_flag(rec, comp)) {
    prebuilt->try_unlock(true);     // 低隔离级别下释放刚加的锁
    if (index == clust_index && unique_search && !used_in_HANDLER) {
      err = DB_RECORD_NOT_FOUND;    // 唯一键下不可能有"第二条同键记录"
      goto normal_return;
    }
    goto next_rec;
  }
```

**ICP 三态**：`ICP_NO_MATCH`（索引上就不满足，跳过）/ `ICP_OUT_OF_RANGE`（越界，`goto idx_cond_failed`）/ `ICP_MATCH`（继续）。

**二级回表**（`requires_clust_rec` 标签）：

```cpp
  if (index != clust_index && prebuilt->need_to_access_clustered) {
  requires_clust_rec:
    mtr_has_extra_clust_latch = true;      // 回表在同一 mtr 里又 latch 聚簇页
    err = row_sel_get_clust_rec_for_mysql(prebuilt, index, rec, thr,
                                          &clust_rec, &offsets, &heap, ...);
    ...
    result_rec = clust_rec;                // 最终返回的是聚簇记录（或其旧版本）
  } else {
    result_rec = rec;
  }
```

- `mtr_has_extra_clust_latch`：回表在**同一个 mtr 里又 latch 聚簇页**，推进前必须 commit mtr 再 restart（避免持有二级页 latch 时 latch 聚簇页破坏 latch 顺序）
- `row_sel_get_clust_rec_for_mysql`：① 用二级记录的主键列构建 `clust_ref` ② 定位聚簇记录 ③ 锁定读加 `LOCK_REC_NOT_GAP`（等值回表不需 gap）④ 快照读不可见则 undo 回溯 + `row_sel_sec_rec_is_for_clust_rec` 校验旧版本是否真对应原二级记录

**转换 + 预取缓存 + 保存位置**：

```cpp
  row_sel_store_mysql_rec(buf, prebuilt, result_rec, vrow, ...);   // rec_t → record[0]

  if (record_buffer != nullptr ||
      ((match_mode == ROW_SEL_EXACT || n_rows_fetched >= MYSQL_FETCH_CACHE_THRESHOLD) &&
       prebuilt->can_prefetch_records())) {
    /* 缓存路径：入队，n_fetch_cached < max 则 goto next_rec 继续填 */
  } else {
    /* 直接返回单行 */
  }

  err = DB_SUCCESS;
  if (!unique_search || !index->is_clustered() || direction != 0 ||
      select_lock_type != LOCK_NONE || used_in_HANDLER || innodb_api) {
    pcur->store_position(&mtr);     // ★ 何时存位置
  }
  goto normal_return;
```

- ★ **缓存路径的关键行为**：函数不会立即返回，而是继续在**同一个 mtr/page latch 内**多抓几行填满缓存，直到缓存满才返回首行
- ★ **何时存位置（核心优化）**：唯一聚簇点查 + 一致读 + `direction == 0` + 非 HANDLER + 非 innodb_api 时**不存**——这种查询 fetch next/prev 必然 EOF，存了也白存

### PHASE 5：next_rec 推进

```cpp
next_rec:
  end_loop++;
  if (row_read_type == ROW_READ_DID_SEMI_CONSISTENT)
    row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;   // 半一致读标志是一次性的
  did_semi_consistent_read = false;
  prebuilt->new_rec_lock.reset();
  vrow = nullptr;

  if (mtr_has_extra_clust_latch || spatial_search) {
    /* 回表 latch 过聚簇页：必须 store → commit mtr → restart → restore */
    pcur->store_position(&mtr);
    mtr_commit(&mtr);
    mtr_has_extra_clust_latch = false;
    mtr_start(&mtr);
    if (sel_restore_position_for_mysql(&same_user_rec, BTR_SEARCH_LEAF, pcur,
                                       moves_up, &mtr)) {
      prev_rec = nullptr;
      goto rec_loop;               // 恢复后已校正到新记录，直接处理
    }
  }

  if (moves_up) {
    if (!pcur->move_to_next(&mtr)) {       // 已到索引尽头
      pcur->store_position(&mtr);
      err = (match_mode != 0) ? DB_RECORD_NOT_FOUND : DB_END_OF_INDEX;
      goto normal_return;
    }
  } else {
    if (!pcur->move_to_prev(&mtr)) goto not_moved;
  }
  goto rec_loop;                   // 新一轮处理下一条
```

- ★ **mtr 重启**：回表后（`mtr_has_extra_clust_latch`）推进前必须 `store_position → mtr_commit → mtr_start → restore_position`——持有二级页 latch 时访问聚簇页会破坏 latch 顺序
- **状态复位**：半一致读标志是一次性的、`new_rec_lock` 复位、`vrow` 置空
- 推进失败（索引尽头）：点查/前缀查 → `DB_RECORD_NOT_FOUND`；纯范围扫描 → `DB_END_OF_INDEX`

### lock_wait_or_error（锁等待/错误汇聚）

任何加锁失败/锁等待都汇聚到这里。先 `pcur->store_position`（**锁等待前必须保存位置**，唤醒后才能 restore 回冲突记录重试），`mtr_commit` 释放所有页 latch（不能带着 latch 去 sleep），`row_mysql_handle_errors` 返回 true（锁等待结束）则 restore 位置后 `goto rec_loop` 重试；返回 false（死锁/超时/被 kill）则 `goto func_exit`。

### 主循环状态流转图

```
                         ┌──────────────────────────────────────────┐
                         │          row_search_mvcc 入口            │
                         └──────────────────────────────────────────┘
                                         │
             ┌───────────────────────────┼──────────────────────────────┐
             │ direction==0              │ direction!=0                 │
             ▼                           ▼                              │
    [PHASE1 清零/建 sel_graph]   [PHASE1 尝试弹缓存行]──命中──► func_exit │
             │                           │ 未命中                        │
             ▼                           ▼                              │
    [unique_search 判定]        [mtr_start]                             │
             │                           │                              │
             ▼                           ▼                              │
    [PHASE2 AHI 捷径] ──FOUND/EXHAUSTED──► func_exit(成功/未找到)        │
             │ RETRY                                                   │
             ▼                                                          │
    [PHASE3 open_no_init / open_at_side / restore_position]             │
             │                                                          │
             ▼                                                          │
    ╔═══════════════════════════════════════════════════════════════════╪══╗
    ║                      rec_loop 主循环                              │  ║
    ║  中断?──►store_position──►normal_return                           │  ║
    ║   rec=get_rec()                                                  │  ║
    ║   ├ infimum ─────────────────────────► next_rec                  │  ║
    ║   ├ supremum ──加gap锁──► next_rec                               │  ║
    ║   ├ match_mode 过滤(不等→加gap锁→NOT_FOUND)                      │  ║
    ║   ├ 锁定读: sel_set_rec_lock                                     │  ║
    ║   │    ├ SUCCESS ───────────────► 继续                           │  ║
    ║   │    ├ SKIP_LOCKED(显式) ─────► next_rec                       │  ║
    ║   │    ├ SKIP_LOCKED(半一致)──► build_committed_vers             │  ║
    ║   │    └ LOCK_WAIT ─────────────► lock_wait_or_error             │  ║
    ║   ├ 快照读: 可见性判断                                          │  ║
    ║   │    ├ 聚簇不可见 → build_prev_vers: null→next_rec else替换rec│  ║
    ║   │    └ 二级不可见 → ICP过滤 → requires_clust_rec               │  ║
    ║   ├ delete-mark? ──►(唯一点查→NOT_FOUND) else next_rec           │  ║
    ║   ├ ICP: NO_MATCH→next_rec / OUT_OF_RANGE→idx_cond_failed        │  ║
    ║   ├ 二级回表: row_sel_get_clust_rec_for_mysql                    │  ║
    ║   ├ row_sel_store_mysql_rec 转换                                 │  ║
    ║   ├ 预取缓存? ──入队→缓存未满→next_rec                           │  ║
    ║   └ store_position → goto normal_return                          │  ║
    ╚═══════════════════════════════════════════════════════════════════╪══╝
                                                                      │
    next_rec: 状态复位 / mtr重启(回表后) / move_to_next(prev)          │
    │ 失败→not_moved→store_position→NOT_FOUND/END_OF_INDEX             │
    └──► goto rec_loop ◄───────────────────────────────────────────────┘

    lock_wait_or_error: store_position→mtr_commit→handle_errors
         ├ 锁等待结束→(表锁→wait_table_again / 行锁→restore→rec_loop)
         └ 不可重试→func_exit
```

| goto 标签 | 含义 | 去向 |
|---|---|---|
| `rec_loop` | 主循环入口 | `next_rec` / `normal_return` / `lock_wait_or_error` |
| `next_rec` | 推进到下一记录 | 推进后 `goto rec_loop` |
| `requires_clust_rec` | 二级回表 | 回表后继续或 `next_rec` |
| `idx_cond_failed` | ICP 越界收尾 | 存位置 → `normal_return` |
| `normal_return` | 正常返回 | `func_exit` 返回 err |
| `lock_wait_or_error` | 加锁失败/等待 | 恢复→`rec_loop` 或 `func_exit` |

---

### 快照读 vs 当前读：一刀切成两条互斥路径

同一个函数里用 `prebuilt->select_lock_type == LOCK_NONE` 一刀切开：

- **快照读**（LOCK_NONE）：read view + undo 回溯，**不碰锁**——读不阻塞写、写不阻塞读（MVCC 非锁定一致读）
- **当前读**（LOCK_S / LOCK_X）：`sel_set_rec_lock` 加锁，**不回溯版本**，只看最新版本——保证"读最新 + 防止幻读"

这比"单一行读取函数"合理，因为两者共享同一套"游标定位/推进/回表/行格式转换"基础设施，只有"拿哪一版 + 加不加锁"这一步分叉。

### 与 MyISAM 的对比

MyISAM 无 MVCC、无行锁、无 undo，读就是读：`ha_myisam::index_next` 直接调 `mi_rnext`，内部 `flag = SEARCH_BIGGER`，按文件偏移 `info->lastpos` 取下一条。**没有可见性判断、没有版本回溯、没有回表、没有持久游标（只有文件偏移 `lastpos`）。** 这是"读一行"复杂度差距的根源——InnoDB 用复杂度换来了并发。

---

## 核心实现

### 从 handler 到 row_search_mvcc：首读与续读两条路

进入 `row_search_mvcc` 只有两条路，**区别就落在传给它的 `mode` 参数上**：

```
SQL 语句
  │
  └─ 执行器算子：TableScanIterator（全表）/ IndexScanIterator（索引）/ ...
       │
       └─ handler 分界面（server ↔ InnoDB）：ha_rnd_next / ha_index_read / ha_index_next ...
            │
            ├─【首读】第一次定位到某条记录
            │    索引路径：ha_innobase::index_read()  → row_search_mvcc(mode = PAGE_CUR_GE/G/L)
            │    全表路径：rnd_init + index_first     → row_search_mvcc(mode = PAGE_CUR_G)
            │
            └─【续读】游标已定位，接着取下一条
                 ha_innobase::general_fetch(buf, direction, match_mode)
                   → row_search_mvcc(buf, PAGE_CUR_UNSUPP, ...)
                                         ↑
                             ★ UNSUPP = "不需要再定位"
```

| | 首读 | 续读 |
|---|---|---|
| 传的 mode | `PAGE_CUR_GE` / `G` / `L`（要定位） | `PAGE_CUR_UNSUPP`（已定位） |
| `row_search_mvcc` 走 | **open**（PHASE 3 的打开分支） | **restore**（PHASE 3 的恢复分支） |
| 入口方法 | `index_read` / `index_first` / `rnd_next`（首次） | `general_fetch` |
| 映射的结束错误 | `HA_ERR_KEY_NOT_FOUND`（定位未命中） | `HA_ERR_END_OF_FILE`（续读到头） |

⇒ ★ **这就是 PHASE 3「打开 vs 恢复游标」的由来**：调用方传的 `mode` 不同，函数内部就必须分两路。**入口的选择**与**内部的分支**是同一件事的两端——看懂这一层，PHASE 3 那个核心分支就不再是"凭什么分"，而是"被谁决定的"。

各入口方法在下面逐节剖析：`rnd_init`（全表初始化）→ `change_active_index`（切索引 / 建模板）→ `index_read`（定位读第一行）→ `rnd_next` / `index_next|prev|first|last`（推进）→ `general_fetch`（续读统一入口）。

### rnd_init：全表扫描初始化（逐行）

```cpp
int ha_innobase::rnd_init(bool scan) {
  assert(table_share->is_missing_primary_key() ==
         (bool)m_prebuilt->clust_index_was_generated);

  int err = change_active_index(table_share->primary_key);

  /* Don't use semi-consistent read in random row reads (by position). */
  if (!scan) {
    m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;
  }

  m_start_of_scan = true;
  return err;
}
```

逐行：

- **断言**：表缺主键 ⇔ InnoDB 侧聚簇索引是内部生成的。全表扫描一定走聚簇索引，这个断言保证后面选的主键就是那颗聚簇索引
- **`change_active_index(table_share->primary_key)`**：把活跃索引切到主键。用户没定义主键时 `primary_key` 是 `MAX_KEY`，`innobase_get_index(MAX_KEY)` 返回 `first_index()`（聚簇索引）。⇒ **全表扫描本质是"聚簇索引的顺序扫描"**
- **`if (!scan)`**：`scan=false` 是"按位置随机读"（`rnd_pos` 场景）。半一致读只在顺序扫描判定 WHERE 时需要，随机读直接禁用，回退 `ROW_READ_WITH_LOCKS`
- **`m_start_of_scan = true`**：这是 `rnd_next` 区分"首次定位"与"续读"的标志

**为什么 rnd_init 之后 rnd_next 能一行行取**：`change_active_index` 做了三件事——① `active_index`/`prebuilt->index` 指向聚簇索引；② `init_search_tuples_types()` 初始化 `search_tuple`/`m_stop_tuple` 字段类型；③ `build_template(false)` 建立 MySQL↔InnoDB 行转换模板。这些是取行的"基础设施"。

### index 选取：change_active_index

`index_init` 整个函数就是 `change_active_index(keynr)`（第二个参数 `sorted` 被忽略，InnoDB 不关心结果是否要求有序）。核心：

```cpp
active_index = keynr;                            // server 层选的索引号
m_prebuilt->index = innobase_get_index(keynr);   // keynr → dict_index_t*
...
m_prebuilt->init_search_tuples_types();          // 按新索引重设 search_tuple 类型
build_template(false);                           // 重建行转换模板
```

- `innobase_get_index(keynr)`：`keynr != MAX_KEY` 时按名字在 dict cache 找；`keynr == MAX_KEY` 时返回 `first_index()`（聚簇索引）
- **`active_index` 是 `handler` 基类字段**，但对 InnoDB 而言唯一赋值点在 `change_active_index`（`index_init` 覆盖了基类实现且没调基类）

**覆盖索引 vs 回表在 prebuilt 里的体现**：

| 字段 | 含义 |
|---|---|
| `read_just_key` | `HA_EXTRA_KEYREAD` 置 1 = 覆盖索引，不回表 |
| `need_to_access_clustered` | 二级索引但需要的列不在索引里 = 1，必须回表 |

server 层判定"只需索引列"时调 `HA_EXTRA_KEYREAD` → `read_just_key=1`，随后 `build_template` 只构建索引列映射，`need_to_access_clustered` 为 0，`row_search_mvcc` 就不回表。

### index_read：定位 + 读第一行

`index_read` 自己**不再选索引**（索引是 `index_init`/`rnd_init` 已选好的，它用 `m_prebuilt->index`）。关键点：

- **`key_ptr != nullptr`**：把 MySQL key 转成 InnoDB 的 `search_tuple`
- **`key_ptr == nullptr`**（`index_first`/`index_last`）：`search_tuple` 置 0 字段 = "定位到索引最左/最右"，方向由 mode 决定
- **`find_flag` → mode + match_mode**：`HA_READ_KEY_EXACT` → `ROW_SEL_EXACT`；`HA_READ_PREFIX_LAST` → `ROW_SEL_EXACT_PREFIX`；`m_last_match_mode` 记住它供 `index_next_same` 用
- ★ **`row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)` 的第 5 个参数 `direction=0` 是"首次定位"**——这就是 `index_read` 和 `index_next` 的本质区别：一个传 `0`，一个传 `ROW_SEL_NEXT/PREV`

### rnd_next：全表扫描下一行（逐行）

```cpp
int ha_innobase::rnd_next(uchar *buf) {
  if (m_user_thd->transaction_rollback_request) return HA_ERR_GENERIC;
  ha_statistic_increment(&System_status_var::ha_read_rnd_next_count);

  if (m_start_of_scan) {
    error = index_first(buf);                 // 首次：定位到聚簇索引第一条
    if (error == HA_ERR_KEY_NOT_FOUND) {
      error = HA_ERR_END_OF_FILE;             // 空表：rnd 接口约定用 END_OF_FILE
    }
    m_start_of_scan = false;
  } else {
    error = general_fetch(buf, ROW_SEL_NEXT, 0);   // 续读
  }
  return error;
}
```

逐行：

- **回滚检查**：server 层已请求回滚（被 kill），直接返回
- **`m_start_of_scan` 是首次/续读的分水岭**（在 `rnd_init` 被置 true）
- 首次 = `index_first`（内部 `index_read(buf, nullptr, 0, HA_READ_AFTER_KEY)` → `PAGE_CUR_G` 定位最左），空表把 `HA_ERR_KEY_NOT_FOUND` 映射成 `HA_ERR_END_OF_FILE`
- 续读 = `general_fetch(buf, ROW_SEL_NEXT, 0)`：restore 游标后向后移动一条

**与 `index_next` 的差异**：`rnd_next` 自己处理"首次要 `index_first`"，且扫描对象是聚簇索引；`index_next` 无首次/续读区分（假定 `index_read` 已定位），可在任意活跃索引上走。

### index_next / index_prev / index_first / index_last

```cpp
int ha_innobase::index_next(uchar *buf) {
  ha_statistic_increment(&System_status_var::ha_read_next_count);
  return general_fetch(buf, ROW_SEL_NEXT, 0);
}
```

- `index_next_same`：只多传 `m_last_match_mode`（`ROW_SEL_EXACT/PREFIX`），要求"下一条也必须匹配同一 key"，用于 ref/eq_ref 的多行同 key 扫描
- `index_prev`：传 `ROW_SEL_PREV`，用于 `ORDER BY ... DESC` 逆序扫描
- `index_first` = `index_read(buf, nullptr, 0, HA_READ_AFTER_KEY)`（定位最左）；`index_last` = `HA_READ_BEFORE_KEY`（定位最右，配合 `index_prev` 逆序）

### general_fetch：续读的统一入口（逐行）

```cpp
int ha_innobase::general_fetch(uchar *buf, uint direction, uint match_mode) {
  const trx_t *trx = m_prebuilt->trx;
  ...
  if (!intrinsic && TrxInInnoDB::is_aborted(trx)) {
    innobase_rollback(ht, m_user_thd, false);
    return convert_error_code_to_mysql(DB_FORCED_ABORT, 0, m_user_thd);
  }
  auto ret = innobase_srv_conc_enter_innodb(m_prebuilt);   // 进入并发控制
  ...
  if (!intrinsic) {
    ret = row_search_mvcc(buf, PAGE_CUR_UNSUPP, m_prebuilt, match_mode, direction);
  } else {
    ret = row_search_no_mvcc(...);                          // intrinsic 临时表
  }
  innobase_srv_conc_exit_innodb(m_prebuilt);
  switch (ret) {
    case DB_SUCCESS: error = 0; break;
    case DB_RECORD_NOT_FOUND: error = HA_ERR_END_OF_FILE; break;
    case DB_END_OF_INDEX: error = HA_ERR_END_OF_FILE; break;
    ...
  }
  return error;
}
```

逐行：

- **`mode = PAGE_CUR_UNSUPP`**：续读时游标已定位，不再需要 `PAGE_CUR_GE/G/L` 这类定位 mode；`direction` 才是要传的（`row_search_mvcc` 见 `PAGE_CUR_UNSUPP` 就走 restore 而非 open）
- **`DB_RECORD_NOT_FOUND` 和 `DB_END_OF_INDEX` 都映射 `HA_ERR_END_OF_FILE`**——与 `index_read` 映射 `HA_ERR_KEY_NOT_FOUND` 不同，这是"续读到头" vs "定位未命中"的语义区别
- 首读由 `index_read`（索引路径）或 `rnd_init`+`index_first`（全表路径）完成；`general_fetch` 是"已定位之后"的通用续读

---

## 不同隔离级别下的表现

> 本节含四个子节：`三把开关`（半一致性读的触发条件）→ `四个隔离级别逐一` → `半一致读的四条件` → `全景对照表`——从开关到结论逐层收口。

### 三把开关

隔离级别在取行路径上只通过**三把开关**起作用，其余逻辑完全共享：

1. **`select_lock_type`**（`LOCK_NONE` 快照读 vs `LOCK_S`/`LOCK_X` 当前读）
2. **`trx->read_view` 的 active/closed 状态**（RR 复用 vs RC 重建 vs RU 不用）
3. **`skip_gap_locks()` / `allow_semi_consistent()` / `releases_non_matching_rows()`**——三者其实是同一件事：RU/RC 为 true，RR/SE 为 false

```cpp
bool skip_gap_locks() const {
  switch (isolation_level) {
    case READ_UNCOMMITTED:
    case READ_COMMITTED:  return (true);
    case REPEATABLE_READ:
    case SERIALIZABLE:    return (false);
  }
}
bool allow_semi_consistent() const { return (skip_gap_locks()); }
```

### 四个隔离级别逐一

**READ UNCOMMITTED**（脏读）

非锁定读**直接跳过可见性检查**：

```cpp
} else {
  /* This is a non-locking consistent read: if necessary, fetch
  a previous version of the record */
  if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
    /* Do nothing: we let a non-locking SELECT read the
    latest version of the record */
  } else if (index == clust_index) { ... }
```

RU 直接返回当前记录（可能是未提交的最新版本）。AHI 捷径也要求 `isolation_level > TRX_ISO_READ_UNCOMMITTED`（RU 无 view 不走捷径）。`skip_gap_locks()==true`，锁定读不加 gap。

**READ COMMITTED**（read view 每语句重建 + 半一致读）

read view **每条语句重建**：语句结束时 `external_lock` 里 `<= RC` 则 `view_close`；下一条语句 `trx_assign_read_view` 看到 `!MVCC::is_view_active` → `view_open` 新建。

**半一致读只在 RC 生效**：`allow_semi_consistent() == skip_gap_locks()`，RU/RC 为 true。半一致读 = UPDATE/DELETE 扫描时目标行被别的事务锁住，先读**最近已提交版本**判断 WHERE 是否匹配，不匹配就跳过而不用等待：

```cpp
case DB_SKIP_LOCKED:
  ut_a(use_semi_consistent);
  ut_a(trx->allow_semi_consistent());
  row_sel_build_committed_vers_for_mysql(...);   // 读最新已提交版本，而非 read view 版本
  if (old_vers == nullptr) goto next_rec;         // 未提交的新插入
  did_semi_consistent_read = true;
  rec = old_vers;
```

**REPEATABLE READ（默认）**（快照固定）

read view **事务内复用**：语句结束**不**关闭（关闭条件 `<= RC` 不成立），`is_view_active` 一直 true → `trx_assign_read_view` 不重建 → 整个事务用同一快照：

```cpp
ReadView *trx_assign_read_view(trx_t *trx) {
  if (srv_read_only_mode) return (nullptr);
  else if (!MVCC::is_view_active(trx->read_view))
    trx_sys->mvcc->view_open(trx->read_view, trx);   // 只有 view 不活跃才重建
  return (trx->read_view);
}
```

`skip_gap_locks()==false`，锁定读加 next-key/gap 锁防幻读。

**SERIALIZABLE**（普通 SELECT 转当前读）

普通 SELECT 在**显式事务内**被提升为 `LOCK_S` 当前读：

```cpp
if (lock_type == F_RDLCK) {
  ...
  } else if (trx->isolation_level == TRX_ISO_SERIALIZABLE &&
             m_prebuilt->select_lock_type == LOCK_NONE &&
             thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
    m_prebuilt->select_lock_type = LOCK_S;    // ★ 普通 SELECT → 当前读
    m_stored_select_lock_type = LOCK_S;
  }
}
```

注意 AUTOCOMMIT=1 的一致读仍是 `LOCK_NONE`（只读事务无需串行化锁也能串行化）。`skip_gap_locks()==false`，加 gap 锁。

### 半一致读的四条件

```cpp
const bool use_semi_consistent =
    prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
    !unique_search && index == clust_index && !trx_is_high_priority(trx);
```

1. `row_read_type == TRY`——前提 `allow_semi_consistent()`（= RC/RU）
2. `!unique_search`——唯一点查锁冲突应直接等待/报错，不适合跳过
3. `index == clust_index`——半一致读只对聚簇记录生效
4. `!trx_is_high_priority`——高优先级事务避免被饿死

### 全景对照表

| 维度 | RU | RC | RR（默认） | SERIALIZABLE |
|---|---|---|---|---|
| 普通 SELECT | LOCK_NONE（读最新） | LOCK_NONE | LOCK_NONE | 事务内 LOCK_S；autocommit LOCK_NONE |
| read view | 不建（可见性跳过） | 每语句重建 | 事务内复用（快照固定） | 同 RR（但普通 SELECT 转当前读） |
| 可见性检查 | 跳过（脏读） | 检查 + 回表回溯 | 检查 + 回表回溯 | 当前读，不查 view |
| 加行锁（普通 SELECT） | 否 | 否 | 否 | 事务内加 LOCK_S |
| gap 锁（锁定读） | 不加 | 不加 | 加 next-key/gap | 加 next-key/gap |
| 半一致读 | 允许（实践主要在 RC） | 生效 | 不允许 | 不允许 |

---

## direction 与游标推进

> `direction` 参数（`ROW_SEL_NEXT` / `ROW_SEL_PREV`）决定游标推进方向。下面三节分别讲：它的语义与取值、它与锁定读/半一致性读的关系（常见误区）、以及 `next_rec` 的状态推进。

### direction 是什么

`row_search_mvcc` 的参数，表示**本次调用**是"打开游标(0)"还是"基于已有游标续读(`ROW_SEL_NEXT`=1 / `ROW_SEL_PREV`=-1)"。

它是**调用级**的，不是"行级"的，也**不是每定位一行就置 0**：

- **打开游标**：`direction=0`，只在扫描的第一次（`index_first`/`index_read`）出现，`pcur->open` 定位。`index_read` 里 `row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)`
- **续读**：之后每次 `rnd_next`/`index_next` 都传 `ROW_SEL_NEXT`（走 `general_fetch`）

`rnd_next` 内部用 `m_start_of_scan` 标志区分：首次 `index_first`，之后 `general_fetch(ROW_SEL_NEXT)`。

**★ `direction` 在源码里是 `const` 入参——函数内永不改写**：

```cpp
dberr_t row_search_mvcc(byte *buf, page_cur_mode_t mode,
                        row_prebuilt_t *prebuilt, ulint match_mode,
                        const ulint direction) {    // ★ const
```

```cpp
@param[in]      direction       0 or ROW_SEL_NEXT or ROW_SEL_PREV;
                                Note: if this is != 0, then prebuilt must has a
                                pcur with stored position! In opening of a
                                cursor 'direction' should be 0.
```

**★★ `moves_up` 是 `direction` 的唯一推导产物**（`row0sel.cc:4819`）：

```cpp
  /* Note that if the search mode was GE or G, then the cursor
  naturally moves upward (in fetch next) in alphabetical order,
  otherwise downward */

  if (direction == 0) {
    if (mode == PAGE_CUR_GE || mode == PAGE_CUR_G || mode >= PAGE_CUR_CONTAIN) {
      moves_up = true;        // ★ 定位阶段：GE / G / 空间索引 ⇒ 后续正向
    }
  } else if (direction == ROW_SEL_NEXT) {
    moves_up = true;          // ★ 续读阶段：ROW_SEL_NEXT ⇒ 正向
  }
  //  ★ 隐含：direction == ROW_SEL_PREV（或 direction==0 且 mode 为 L/LE）⇒ moves_up 保持 false（反向）
```

> ★★ **`moves_up` 默认是 `false`**——这是个容易看漏的点：反向扫描是"默认"，正向才需要显式置位。

**★ `direction` 的 7 个使用点**（全仓就这么几处，逐一列出更有说服力）：

| # | 行号 | 作用 |
|---|---|---|
| 1 | 4543 | PHASE 1：`direction == 0` → 复位 `n_rows_fetched` / 缓存 / `m_end_range`，并 `row_prebuild_sel_graph()`；`!= 0` → 命中缓存直接出队返回 |
| 2 | 4637 | `unique_search && direction != 0 && !used_in_HANDLER` → 直接 `DB_RECORD_NOT_FOUND` **省掉一次 B 树搜索** |
| 3 | 4679 | AHI 捷径只在 `direction == 0 && unique_search` 时启用 |
| 4 | 4819 | `moves_up` 推导（上表） |
| 5 | 4878 | **PHASE 3 的分派**：`!= 0` → `sel_restore_position_for_mysql`；`== 0` → `open_no_init` / `open_at_side` |
| 6 | 5772 | `store_position` 的条件之一（唯一键聚簇一致性读 + `direction == 0` 时可省） |
| 7 | 5806+ | PHASE 5：`moves_up ? move_to_next : move_to_prev` |

```cpp
  // 使用点 2：唯一键点查 + 续读 ⇒ 直接"没有下一行"
    if (UNIV_UNLIKELY(direction != 0 && !prebuilt->used_in_HANDLER)) {
      err = DB_RECORD_NOT_FOUND;
      goto func_exit;
    }
```

```cpp
  // 使用点 6：不必存位置的优化条件
  if (!unique_search || !index->is_clustered() || direction != 0 ||
      prebuilt->select_lock_type != LOCK_NONE || prebuilt->used_in_HANDLER ||
      prebuilt->innodb_api) {
    if (!spatial_search) {
      pcur->store_position(&mtr);
    }
    ...
  }
```

### direction 与锁定读/半一致性读无关

它只回答"是否打开游标"。半一致性读的"第一次读"和"第二次重读"，direction 取决于该行在扫描中的位置（扫描首条则第一次 0、重读 NEXT；扫描中段则两次都 NEXT）。

重读"同一行"靠的**不是 direction**，而是：半一致性读没走 `next_rec`（游标不推进）+ 第二次进来 `row_read_type==DID` 且 `need_to_process==false` 时不 `goto next_rec`。

### 游标推进

`next_rec` 标签 → PHASE 5 → `if (moves_up) pcur->move_to_next(&mtr); else pcur->move_to_prev(...)` → `goto rec_loop` 回到循环头处理下一条。

半一致性读路径**不经过** `next_rec`，`break` 后落到 `normal_return` 返回，游标停在原记录（`store_position` 已存），故下次读仍在同一行——这是"半一致性读重读同行"的机制基础。

**★ PHASE 5 的真正推进代码**（`row0sel.cc:5836`，含一条重要注释）：

```cpp
  /* NOTE: For moves_up==false, the mini-transaction will be
  committed and restarted every time when switching b-tree
  pages. For moves_up==true in index condition pushdown, we can
  scan an entire secondary index tree within a single
  mini-transaction. As long as the prebuilt->idx_cond does not
  match, we do not need to consult the clustered index or
  return records to MySQL, and thus we can avoid repositioning
  the cursor. What prevents us from buffer-fixing all leaf pages
  within the mini-transaction is the btr_leaf_page_release()
  call in btr_pcur::move_to_next_page(). Only the leaf page where
  the cursor is positioned will remain buffer-fixed.
  For R-tree spatial search, we also commit the mtr each time */

  if (moves_up) {
    bool move;
    if (spatial_search) {
      move = rtr_pcur_move_to_next(search_tuple, mode, prebuilt->select_mode,
                                   pcur, 0, &mtr);
    } else {
      move = pcur->move_to_next(&mtr);          // ★ 正向
    }
    if (!move) {
    not_moved:
      if (!spatial_search) pcur->store_position(&mtr);
      if (match_mode != 0) err = DB_RECORD_NOT_FOUND;   // ★ 等值/前缀扫描越界
      else                 err = DB_END_OF_INDEX;       // ★ 范围/全扫描越界
      goto normal_return;
    }
  } else {
    if (UNIV_UNLIKELY(!pcur->move_to_prev(&mtr))) {     // ★ 反向
      goto not_moved;
    }
  }

  goto rec_loop;
```

> ★★ **上面的注释解释了 ICP 的一个隐藏收益**：正向扫描 + ICP 不匹配时，**可以在单个 mtr 内扫完整个二级索引树**（因为不需要回表、不需要重定位游标），这是 ICP 收益被低估的部分。

**★ 正向跨页 vs 反向跨页的代价不对称**（`btr0pcur.cc`）：

```cpp
void btr_pcur_t::move_to_next_page(mtr_t *mtr) {
  ...
  auto next_block = btr_block_get(page_id_t(block->page.id.space(), next_page_no),
                                  block->page.size, mode, ...);
  btr_leaf_page_release(get_block(), mode, mtr);   // ★ 释放当前页 latch
  page_cur_set_before_first(next_block, get_page_cur());
  ...
}
```

```cpp
void btr_pcur_t::move_backward_from_page(mtr_t *mtr) {
  ...
  store_position(mtr);
  mtr_commit(mtr);                    // ★★ 反向必须提交 mtr 重开
  mtr_start(mtr);
  restore_position(latch_mode2, mtr, UT_LOCATION_HERE);
  ...
}
```

> ★★ **原因**：InnoDB 的 B-tree 加锁顺序是 **left-to-right**。反向扫描要拿到"左页 + 当前页"两把 latch，必须先提交 mtr 重开，用 `BTR_SEARCH_PREV` / `BTR_MODIFY_PREV` 这种专门的前向模式恢复。这是**反向扫描（ORDER BY DESC）比正向慢**的源码级原因。
> ⚠️ **不存在 `btr_pcur_t::move_to_prev_page()`**——真名 `move_backward_from_page()`。存在的只有 `move_to_next_page()`。

```cpp
inline bool btr_pcur_t::move_to_next(mtr_t *mtr) {
  if (is_after_last_on_page()) {
    if (is_after_last_in_tree(mtr)) return (false);   // ★ 树尾 ⇒ not_moved
    move_to_next_page(mtr);
    return (true);
  }
  move_to_next_on_page();
  return (true);
}

bool btr_pcur_t::move_to_prev(mtr_t *mtr) {
  if (is_before_first_on_page()) {
    if (is_before_first_in_tree(mtr)) return (false);
    move_backward_from_page(mtr);       // ★ 反向跨页
    return (true);
  }
  move_to_prev_on_page();
  return (true);
}
```

> ★ 另有一个 `row_search_traverse()` 把 `moves_up` → next/prev 封装起来，供 `row_search_no_mvcc` 用（返回值同样是 `match_mode != 0 ? NOT_FOUND : END_OF_INDEX`）。

---

### need_to_process

`sel_restore_position_for_mysql` 的返回值：调 `pcur->restore_position` 恢复之前 `store_position` 保存的位置，返回 true 表示"恢复后需重新处理游标现在指向的记录"（原记录被删、位置失效时，按 `moves_up` 调 `move_to_next` 前移）。

在 need_to_process 与 DID 的分支里：

- `need_to_process == true` 且 `row_read_type == DID`：把 DID 复位成 TRY（记录已删，之前的半一致性读作废）
- `need_to_process == false` 且 `row_read_type == DID`：不 `goto next_rec`，继续处理同一行（半一致性读重读加锁的分支）

**★ `sel_restore_position_for_mysql()` 完整源码**（`row0sel.cc:3401`，`need_to_process` 的唯一产出者）：

```cpp
static bool sel_restore_position_for_mysql(
    bool *same_user_rec,   /*!< out: true if we were able to restore
                            the cursor on a user record with the
                            same ordering prefix in the B-tree index */
    ulint latch_mode, btr_pcur_t *pcur, bool moves_up, mtr_t *mtr) {
  auto success = pcur->restore_position(latch_mode, mtr, UT_LOCATION_HERE);

  *same_user_rec = success;      // ★★ 恢复成功（仍在同一条用户记录）才为 true

  ut_ad(!success || pcur->m_rel_pos == BTR_PCUR_ON);

  switch (pcur->m_rel_pos) {
    case BTR_PCUR_UNSET:
      ut_d(ut_error);
      ut_o(return (true));
    case BTR_PCUR_ON:
      if (!success && moves_up) {
      next:
        pcur->move_to_next(mtr);   // ★ 正向 + 没恢复到原记录 ⇒ 先推进一条
        return true;
      }
      return (!success);
    case BTR_PCUR_AFTER_LAST_IN_TREE:
    case BTR_PCUR_BEFORE_FIRST_IN_TREE:
      return true;
    case BTR_PCUR_AFTER:
      pcur->m_pos_state = BTR_PCUR_IS_POSITIONED;
    prev:
      if (pcur->is_on_user_rec() && !moves_up) {
        pcur->move_to_prev(mtr);   // ★ 反向 + 原记录在"下一条"之后 ⇒ 回退一条
      }
      return true;
    case BTR_PCUR_BEFORE:
      switch (pcur->m_pos_state) {
        case BTR_PCUR_IS_POSITIONED_OPTIMISTIC:
          pcur->m_pos_state = BTR_PCUR_IS_POSITIONED;
          if (pcur->m_search_mode == PAGE_CUR_GE) {
            /* Positioned during Greater or Equal search with
            BTR_PCUR_BEFORE. Optimistic restore to the same record.
            If scanning for lower then we must move to previous record.
            This can happen with:
            HANDLER READ idx a = (const);
            HANDLER READ idx PREV; */
            goto prev;
          }
          return true;
        case BTR_PCUR_IS_POSITIONED:
          if (moves_up && pcur->is_on_user_rec()) goto next;
          return true;
        case BTR_PCUR_WAS_POSITIONED:
        case BTR_PCUR_NOT_POSITIONED:
          break;
      }
  }
  ut_d(ut_error);
  ut_o(return (true));
}
```

> ★★ **`need_to_process` 全仓只有 2 处**（`:4888` 定义、`:4893` 唯一使用点），且只在 `direction != 0` 的 PHASE 3 分支里。它和 `same_user_rec` 是 `sel_restore_position_for_mysql` 的两个出参，语义不同：
> - `need_to_process`：**是否要重新处理游标现在指向的记录**（返回 true 说明位置有偏移）
> - `same_user_rec`：**是否成功恢复到同一条用户记录**（在 PHASE 5 和 `lock_wait_or_error` 里另有 3 处使用）

```cpp
    // 唯一使用点（row0sel.cc:4888-4903）
    auto need_to_process = sel_restore_position_for_mysql(
        &same_user_rec, BTR_SEARCH_LEAF, pcur, moves_up, &mtr);

    ut_ad(prev_rec == nullptr);

    if (UNIV_UNLIKELY(need_to_process)) {
      if (UNIV_UNLIKELY(prebuilt->row_read_type == ROW_READ_DID_SEMI_CONSISTENT)) {
        /* We did a semi-consistent read, but the record was removed in
        the meantime. */
        prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;
      }
    } else if (UNIV_LIKELY(prebuilt->row_read_type != ROW_READ_DID_SEMI_CONSISTENT)) {
      goto next_rec;     // ★ 恢复成功且落在上次返回的那条 ⇒ 直接推进
    }
```

> ⚠️ 注意 `same_user_rec` 的另一个消费点（`row0sel.cc:5987`）：`if (!same_user_rec && trx->releases_non_matching_rows()) prebuilt->new_rec_lock.reset();`——**没能恢复到同一条就放弃"mini-rollback"解锁优化**。

**★ 持久游标的三级恢复机制**（`btr0pcur.h:368` 的注释给出了完整对照表）：

```cpp
void btr_pcur_t::store_position(mtr_t *mtr) {
  ...
  if (page_rec_is_supremum_low(offs)) {
    rec = page_rec_get_prev(rec);
    m_rel_pos = BTR_PCUR_AFTER;         // ★ 落在 supremum ⇒ "最后一条用户记录之后"
  } else if (page_rec_is_infimum_low(offs)) {
    rec = page_rec_get_next(rec);
    m_rel_pos = BTR_PCUR_BEFORE;        // ★ 落在 infimum ⇒ "第一条用户记录之前"
  } else {
    m_rel_pos = BTR_PCUR_ON;            // ★ 落在真实用户记录
  }

  m_old_stored = true;
  m_old_rec = dict_index_copy_rec_order_prefix(index, rec, &m_old_n_fields,
                                               &m_old_rec_buf, &m_buf_size);
  m_block_when_stored.store(block);     // ★ 乐观恢复用：直接拿这个 block 指针
  m_modify_clock = block->get_modify_clock(...);  // ★ 乐观恢复的"页面未被改"判据
}
```

| 保存字段 | 恢复时用途 |
|---|---|
| `m_block_when_stored` | **乐观恢复**：直接访问这个 block 指针，不走 B-tree 搜索 |
| `m_modify_clock` | **乐观判断**：clock 没变说明页面未被修改，直接用 offset 定位 |
| `m_old_rec`（排序前缀副本） | **悲观恢复**：clock 变了就用 key 从 root 重新搜索 |
| `m_rel_pos` | **微调**：BEFORE/AFTER 需要 `move_to_next`/`prev` 调整 |

```cpp
  // 悲观恢复：按 m_rel_pos 选搜索模式
    switch (m_rel_pos) {
      case BTR_PCUR_ON:     mode = PAGE_CUR_LE; break;   // ★ 精确落回同一条
      case BTR_PCUR_AFTER:  mode = PAGE_CUR_G;  break;
      case BTR_PCUR_BEFORE: mode = PAGE_CUR_L;  break;
      default: ut_error;
    }
    open_no_init(index, tuple, mode, latch_mode, 0, mtr, location);   // ★ 从 root 重新下降
    m_search_mode = old_mode;    // ★★ 还原 —— 这正是 general_fetch 后续回读 mode 的来源
```

> ★ 空树特例：`BTR_PCUR_AFTER_LAST_IN_TREE` / `BEFORE_FIRST_IN_TREE` 直接 `btr_cur_open_at_index_side()`，且 `return (false)`（一定"没恢复到同一条"）。

---

### row_sel_convert_mysql_key_to_innobase：server 查询键 → InnoDB dtuple

这是**查询侧的输入转换**——`index_read` 拿到 server 的引擎键格式（`key_copy` 产出），要先转成 InnoDB 的 `dtuple_t` 才能交给 B-tree 搜索：

```cpp
void row_sel_convert_mysql_key_to_innobase(dtuple_t *tuple, byte *buf,
                                           ulint buf_len, dict_index_t *index,
                                           const byte *key_ptr, ulint key_len) {
  /* For documentation of the key value storage format in MySQL, see
  ha_innobase::store_key_val_for_row() in ha_innodb.cc. */
  key_end = key_ptr + key_len;

  /* Permit us to access any field in the tuple (ULINT_MAX): */
  dtuple_set_n_fields(tuple, ULINT_MAX);   // ★ 先放开，解析完再定实际字段数

  // ★ 特例：无主键表的生成聚簇索引，唯一排序列是 ROW_ID（6 字节），直接拷完返回
  if (UNIV_UNLIKELY(dfield_get_type(dfield)->mtype == DATA_SYS)) {
    ut_a(key_len == DATA_ROW_ID_LEN);
    dfield_set_data(dfield, key_ptr, DATA_ROW_ID_LEN);
    dtuple_set_n_fields(tuple, 1);
    return;
  }

  while (key_ptr < key_end) {
    data_offset = 0;
    is_null = false;
    // ★ 与写入侧 key_copy 呼应：可空列的第一字节是 NULL 标记
    if (!(dfield_get_type(dfield)->prtype & DATA_NOT_NULL)) {
      /* The first byte in the field tells if this is an SQL NULL value */
      data_offset = 1;
      if (*key_ptr != 0) {
        dfield_set_null(dfield);
        is_null = true;
      }
    }

    if (DATA_LARGE_MTYPE(type) || DATA_GEOMETRY_MTYPE(type)) {
      if (dict_index_is_spatial(index)) {
        data_len = key_len;                // R-tree：wkb 总大小
      } else {
        ut_a(field->prefix_len > 0 || DATA_POINT_MTYPE(type));
        // ★ 小端 2 字节长度头（低位在前）：data_len = 低字节 + 256*高字节
        data_len = key_ptr[data_offset] + 256 * key_ptr[data_offset + 1];
        // ★ 占位用的是 prefix_len（前缀长度），不是实际 data_len
        //   注释解释：MySQL 在 key buffer 里为该字段预留 prefix_len 字节，
        //   即使实际值只占 data_len —— 所以键格式里变长字段是"定长占位"
        data_field_len =
            data_offset + 2 +
            (type == DATA_POINT ? DATA_POINT_LEN : field->prefix_len);
        data_offset += 2;
      }
    }
    ...
  }
}
```

⇒ 两个容易漏的点：

1. **无主键表走 ROW_ID 特例**（`DATA_SYS`），直接拷 6 字节就返回，**不进通用循环**。
2. **变长字段在键格式里按 `prefix_len` 定长占位**，实际长度由 2 字节小端长度头给出——这正是"键格式恒 2 字节长度头"的另一半证据（写入侧见 `key_copy`，格式图谱见 handler.md）。

## 行格式：rec_t 磁盘行与 record[0] 的转换

> 这是读路径的"格式交接"段：从 `rec_t` 的物理布局讲到字段寻址（`rec_get_offsets`）、NULL 语义、以及 instant 列在偏移层的投影——与 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md) 的"行写方向编码"互为镜像。

### rec_t 的物理布局

COMPACT/DYNAMIC 记录（new-style）的位域布局，`rem0rec.ic` 的权威注释：

```
(1) byte offset downward from origin:
    1-2: 8+8 bits relative offset of next record
    3:   3 bits status (000=conventional / 001=node pointer / 010=infimum / 011=supremum)
         5 bits heap number
    4:   8 bits heap number
    5:   4 bits n_owned + 4 bits info bits
```

固定头 5 字节（`REC_N_NEW_EXTRA_BYTES = 5`）。`rec` 的 origin 指向**第一个数据字段**，固定头在 origin 之下（低地址方向）。

COMPACT 记录从低地址到高地址的完整布局：

```
[变长字段长度列表 lens][NULL bitmap][(instant 版本位 1B)][5B 固定头][字段数据...]
                                                       ^ rec origin 指向这里
```

info bits 四位的含义（`rem0rec.ic`）：`MIN_REC`（min rec 标志）/ `DELETED`（delete mark，`rec_get_deleted_flag` 读它）/ `INSTANT`（instant ADD COLUMN）/ `VERSION`（row version）。

```cpp
/* Info bit denoting the predefined minimum record ... */
constexpr uint32_t REC_INFO_MIN_REC_FLAG   = 0x10UL;
/** The deleted flag in info bits; when bit is set to 1, it means the record has
 been delete marked */
constexpr uint32_t REC_INFO_DELETED_FLAG   = 0x20UL;
/* Use this bit to indicate record has version */
constexpr uint32_t REC_INFO_VERSION_FLAG   = 0x40UL;
/** The instant ADD COLUMN flag. When it is set to 1, it means this record
was inserted/updated after an instant ADD COLUMN. */
constexpr uint32_t REC_INFO_INSTANT_FLAG   = 0x80UL;
```

```cpp
/* Number of extra bytes in an old-style record ... */
constexpr uint32_t REC_N_OLD_EXTRA_BYTES = 6;   // REDUNDANT
/* Number of extra bytes in a new-style record ... */
constexpr int32_t  REC_N_NEW_EXTRA_BYTES = 5;   // COMPACT/DYNAMIC（★ int32_t！）
/* Number of extra bytes in a new-style temporary record ...
This is used only after instant ADD COLUMN. */
constexpr uint32_t REC_N_TMP_EXTRA_BYTES = 1;

/* Record status values */
constexpr uint32_t REC_STATUS_ORDINARY  = 0;
constexpr uint32_t REC_STATUS_NODE_PTR  = 1;
constexpr uint32_t REC_STATUS_INFIMUM   = 2;
constexpr uint32_t REC_STATUS_SUPREMUM  = 3;
constexpr uint32_t REC_NODE_PTR_SIZE    = 4;
```

> ⚠️ **常量不在 `include/rem0rec.h` / `include/rem/rec.h`**——真身在 **`storage/innobase/rem/rec.h`**（8.0.39 目录重排后，`include/rem0rec.h:45` 用 `#include "rem/rec.h"` 引入）。
> ⚠️ `REC_N_NEW_EXTRA_BYTES` 的类型是 **`constexpr int32_t`**，其余是 `uint32_t`——混用时容易触发符号比较警告。

### 字段寻址：rec_get_offsets

`row_search_mvcc` 每取一条记录都先：

```c
offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, ...);
```

得到 `offsets` 数组——每个元素是"该字段结束偏移 + 标志位"（`REC_OFFS_SQL_NULL / REC_OFFS_DEFAULT / REC_OFFS_DROP / REC_OFFS_EXTERNAL`）。COMPACT 字段寻址的主循环（`rec_init_offsets_comp_ordinary`）：

```c
do {
  if (nullable) {                     /* 读 NULL bitmap */
    if (*nulls & null_mask) { len = offs | REC_OFFS_SQL_NULL; goto resolved; }
  }
  if (!field->fixed_len) {            /* 变长：读 lens 列表 */
    len = *lens--;
    if (DATA_BIG_COL(col)) {          /* 可能外部存储 / 两字节长度 */
      if (len & 0x80) { len <<= 8; len |= *lens--; offs += len & 0x3fff; len = offs; goto resolved; }
    }
    len = offs += len;
  } else {
    len = offs += field->fixed_len;   /* 定长 */
  }
resolved:
  rec_offs_base(offsets)[i + 1] = len;
} while (++i < rec_offs_n_fields(offsets));
```

**★ offsets 数组的头部布局**（`rem/rec.h:516` 的权威注释，以及"错位一格"的设计）：

```cpp
/* Compact flag ORed to the extra size returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_COMPACT   = 1U << 31;   // 0x80000000  ★只用 offsets_base[0]
/* SQL NULL flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_SQL_NULL  = 1U << 31;   // 0x80000000  用 offsets_base[1+i]
/* External flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_EXTERNAL  = 1 << 30;    // 0x40000000
/* Default value flag in offsets ... */
constexpr uint32_t REC_OFFS_DEFAULT   = 1 << 29;    // 0x20000000  INSTANT ADD 默认值
/* An INSTANT DROP flag in offsets ... */
constexpr uint32_t REC_OFFS_DROP      = 1 << 28;    // 0x10000000  INSTANT DROP
/* Mask for offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_MASK      = REC_OFFS_DROP - 1;   // 0x0FFFFFFF
```

```cpp
#ifdef UNIV_DEBUG
constexpr uint32_t REC_OFFS_HEADER_SIZE = 4;   // ★ debug：offsets[2]=rec, offsets[3]=index
#else
constexpr uint32_t REC_OFFS_HEADER_SIZE = 2;   // ★ release：offsets[0]=n_alloc, offsets[1]=n_fields
#endif
constexpr uint32_t REC_OFFS_NORMAL_SIZE = 100; // 栈上默认大小
constexpr uint32_t REC_OFFS_SMALL_SIZE  = 10;
```

> ★★ **`REC_OFFS_HEADER_SIZE` 不固定**：release 是 2，UNIV_DEBUG 是 4。多出来的 `offsets[2]`/`[3]` 是 debug "magic"（见 `rec_offs_make_valid`）。
> ★ **offsets[0] 是 `n_alloc`（数组容量）不是 magic**，offsets[1] 是 `n_fields`。`rec_offs_init(offsets)` 宏就是`rec_offs_set_n_alloc(offsets, (sizeof offsets) / sizeof *offsets)`。

```
  ┌──────────┬──────────┬─(debug)──┬───────────┬─────────────┬──────────────┬─────┐
  │ offsets[0]│ offsets[1]│ [2] rec │ [3] index │ base[0]     │ base[1..n]   │ ... │
  │ n_alloc  │ n_fields │ (magic)  │ (magic)   │ extra+flags │ 各字段结束偏移│     │
  └──────────┴──────────┴──────────┴───────────┴─────────────┴──────────────┴─────┘
                                                ↑ rec_offs_base(offsets) 从这里开始
  base[1+i] = 字段 i 的【结束】偏移（不是起始！）+ 标志位
  字段 i 的起始偏移 = (i == 0) ? 0 : (base[i] & REC_OFFS_MASK)
  len(i) = base[1+i] & MASK - base[i] & MASK
```

```cpp
static inline ulint rec_get_nth_field_offs_low(const ulint *offsets, ulint n,
                                               ulint *len) {
  if (n == 0) {
    offs = 0;
  } else {
    offs = rec_offs_base(offsets)[n] & REC_OFFS_MASK;   // ★ 前一个字段的"结束" = 本字段起始
  }

  length = rec_offs_base(offsets)[1 + n];               // ★ 本字段的"结束"槽位

  if (length & REC_OFFS_SQL_NULL)        length = UNIV_SQL_NULL;
  else if (length & REC_OFFS_DEFAULT)    length = UNIV_SQL_ADD_COL_DEFAULT;
  else if (length & REC_OFFS_DROP)       length = UNIV_SQL_INSTANT_DROP_COL;
  else { length &= REC_OFFS_MASK; length -= offs; }

  *len = length;
  return (offs);
}
```

> ★★ **`rec_offs_base(offsets)[1+i]` 存的是"字段 i 的结束偏移"，`[i]` 是"字段 i-1 的结束偏移"**——这个"错位一格"是理解全部 `rec_offs_*` 的关键。
> ⚠️ `rec_offs_*` 是**四层**封装：`rem/rec.h`（base / n_alloc / n_fields / validate）→ `rem0rec.ic`（comp / any_extern / data_size）→ `rem0lrec.h`（`_low` 版，只用物理列号）→ `rem0wrec.h/.cc`（带 `index` 的包装，**先做 `get_field_off_pos()` 逻辑列号→物理列号翻译**）。INSTANT ADD/DROP 的翻译就发生在最后一层：
>
> ```cpp
> ulint rec_offs_nth_extern(const dict_index_t *index, const ulint *offsets, ulint n) {
>   if (index && index->has_row_versions()) {
>     n = index->get_field_off_pos(n);      // ★ 逻辑列号 → 物理列号
>   }
>   validate_rec_offset(index, offsets, n, UT_LOCATION_HERE);
>   return (rec_offs_nth_extern_low(offsets, n));
> }
> ```

**★ `rec_get_offsets()` 里 `n` 的决定方式**（不是"记录里存了几个字段"这么简单）：

```cpp
  if (dict_table_is_comp(index->table)) {
    switch (UNIV_EXPECT(rec_get_status(rec), REC_STATUS_ORDINARY)) {
      case REC_STATUS_ORDINARY:  n = dict_index_get_n_fields(index); break;
      case REC_STATUS_NODE_PTR:  n = dict_index_get_n_unique_in_tree_nonleaf(index) + 1; break;
      case REC_STATUS_INFIMUM:
      case REC_STATUS_SUPREMUM:  n = 1; break;
      default: ut_error;
    }
  } else {
    n = rec_get_n_fields_old(rec, index);     // REDUNDANT：读记录头里的 n_fields
  }
  if (UNIV_UNLIKELY(n_fields < n)) n = n_fields;

  ulint size = n + (1 + REC_OFFS_HEADER_SIZE);   // ★ 末尾多 1 个给"最后一个字段的结束偏移"（= data_size）
  if (UNIV_UNLIKELY(!offsets) || rec_offs_get_n_alloc(offsets) < size) {
    offsets = static_cast<ulint *>(mem_heap_alloc(*heap, size * sizeof(ulint)));
    rec_offs_set_n_alloc(offsets, size);
  }
  rec_offs_set_n_fields(offsets, n);
  rec_init_offsets(rec, index, offsets);
  return (offsets);
```

> ⚠️ **`rec_get_offsets()` 定义在 `storage/innobase/rem/rec.cc:364`，不在 `rem0rec.cc`**（后者只有调用点）。
> ★ `rec_offs_data_size()` = `base[n_fields] & MASK`（正好用了那个多出来的槽位）；`rec_offs_extra_size()` = `base[0] & ~(COMPACT|EXTERNAL)`——**注意它没抹 DEFAULT/DROP**，因为那两个只出现在 `[1+i]`。

### ★ row_sel_store_mysql_rec：取行总入口（接口级要点）

> **完整逐行剖析**见下文 `clust_templ_for_sec` 一节（含 `template_col` 映射、`sec_field_no` 反查、三种分发、`row_sel_store_mysql_field` 的借出-归还与三条取数路径）。本节只留**接口级要点**。

头注释（`row0sel.h`）：

> "Convert a row in the Innobase format to a row in the MySQL format. Note that the template in prebuilt may advise us to copy only a few columns to mysql_rec, other columns are left blank."

主循环**只遍历 `prebuilt->mysql_template`（本次查询需要的列），不遍历索引全部字段**：

```c
for (ulint i = 0; i < prebuilt->n_template; i++) {
  const auto templ = &prebuilt->mysql_template[i];
  ulint field_no = rec_clust ? templ->clust_rec_field_no : templ->rec_field_no;
  row_sel_store_mysql_field(mysql_rec, prebuilt, rec, rec_index, prebuilt_index,
                            offsets, field_no, templ, sec_field_no, lob_undo, blob_heap);
}
```

⇒ **`DB_ROW_ID` / `DB_TRX_ID` / `DB_ROLL_PTR` 三个隐藏系统列默认不拷给 server**——除非查询显式 `SELECT _rowid`，模板里才会有对应列。MVCC 取 `DB_TRX_ID` 走的是另一条路（直接从 rec 读 `trx_id_offset`），不靠模板。

`row_sel_store_mysql_field` 做四件事：

**(1) NULL**：置 server 层 NULL bitmap 位，并把字段数据区拷成 default 值：

```c
if (len == UNIV_SQL_NULL) {
  /* MySQL assumes that the field for an SQL NULL value is set to the default value. */
  mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;
  memcpy(mysql_rec + templ->mysql_col_offset,
         (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
         templ->mysql_col_len);
  return true;
}
```

**`mysql_null_byte_offset == 0` 的列，其 NULL bit 就落在 `record[0]` 第 0 字节**——这就是 server 层"`record[0]` 第一个字节是 NULL bitmap（前 8 列）"约定的来源。InnoDB 不自己造 NULL bitmap，而是把位写进 `mysql_template[]` 事先算好的偏移/掩码。

**(2) 变长字段**：真 VARCHAR 写 1/2 字节长度前缀（`row_mysql_store_true_var_len`），CHAR 则 pad 尾部空格。

**(3) 字节序**与 **(4) BLOB 的三条取数路径**（外部溢出页经 `lob::btr_rec_copy_externally_stored_field` / 页内本地大字段必须 `mem_heap_dup` 到 `blob_heap` / 普通字段 memcpy）——完整代码与逐行解释见下文 `clust_templ_for_sec` 节的「`row_sel_store_mysql_field`：单字段落盘」。

### ★ row_sel_field_store_in_mysql_format_func：字段级转换的 switch 全貌

上一节剖析的是"怎么把每个字段交给转换"（NULL 判定、BLOB 外存拷贝、字段号切换），本节剖析**转换本身**——单个非 NULL 字段从 InnoDB 格式到 MySQL 格式的最后一步。这是 rec_t → record[0] 转换链的最底层，`row_sel_store_mysql_field` 的两处调用都收口到这里。

**定位**：它的逆操作（counterpart）是 INSERT 方向的 `row_mysql_store_col_in_innobase_format`，两者头注释互相指认。也就是说这张 switch 与"写入时的编码"是一一镜像的关系，读懂了写方向的编码，读方向的解码就是它的逆过程。

**完整源码（就近注释）**：

```cpp
void row_sel_field_store_in_mysql_format_func(
    byte *dest, const mysql_row_templ_t *templ, const dict_index_t *index,
    IF_DEBUG(ulint field_no, ) const byte *data,
    ulint len IF_DEBUG(, ulint sec_field)) {
  byte *ptr;
#ifdef UNIV_DEBUG
  const dict_field_t *field =
      templ->is_virtual ? nullptr : index->get_field(field_no);
  bool clust_templ_for_sec = (sec_field != ULINT_UNDEFINED);   // 仅 assert 用
#endif /* UNIV_DEBUG */

  if (templ->is_multi_val) {              // ★ 多值列绝不该走单值转换：出现即 bug
    ib::fatal(UT_LOCATION_HERE, ER_CONVERT_MULTI_VALUE)
        << "Table name: " << index->table->name
        << " Index name: " << index->name;
  }

  auto const mysql_col_len = templ->mysql_col_len;   // server 侧该列宽度

  ut_ad(rec_field_not_null_not_add_col_def(len));    // len 不是 NULL / ADD_COL_DEFAULT 哨兵
  UNIV_MEM_ASSERT_RW(data, len);
  UNIV_MEM_ASSERT_W(dest, mysql_col_len);
  UNIV_MEM_INVALID(dest, mysql_col_len);             // debug：标记 dest 未初始化

  switch (templ->type) {
    const byte *field_end;    // 在 switch 块首声明：作用域覆盖所有 case
    byte *pad;

    case DATA_INT: {
      /* Convert integer data from Innobase to a little-endian
      format, sign bit restored to normal */
      ptr = dest + len;
      for (;;) {                    // ★ 从高地址往低地址倒写 = 整段字节反转
        ptr--;
        *ptr = *data;
        if (ptr == dest) break;
        data++;
      }
      if (!templ->is_unsigned) {    // ★ 仅 signed：写方向 XOR 过 0x80，这里还原
        dest[len - 1] = (byte)(dest[len - 1] ^ 128);
      }
      ut_ad(mysql_col_len == len);  // 定长类型：server 宽度 == InnoDB 长度
      break;
    }

    case DATA_VARCHAR:
    case DATA_VARMYSQL:
    case DATA_BINARY:
      field_end = dest + mysql_col_len;      // 补空格的上界

      if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR) {
        /* This is a >= 5.0.3 type true VARCHAR. Store the length of the
        data to the first byte or the first two bytes of dest. */
        dest = row_mysql_store_true_var_len(dest, len, templ->mysql_length_bytes);
        /* Copy the actual data. Leave the rest of the buffer uninitialized. */
        memcpy(dest, data, len);             // ★ 长度头（1/2B）+ 数据；其余不初始化
        break;
      }

      /* Copy the actual data */
      ut_memcpy(dest, data, len);            // 老式 CHAR：先拷数据

      /* Pad with trailing spaces. */
      pad = dest + len;
      ut_ad(templ->mbminlen <= templ->mbmaxlen);

      /* We treat some Unicode charset strings specially. */
      switch (templ->mbminlen) {
        case 4:                              // UTF-32
          /* InnoDB should never have stripped partial UTF-32 characters. */
          ut_a(!(len & 3));
          break;
        case 2:                              // UCS2 / UTF-16：空格 = 0x0020 两字节
          /* A space char is two bytes, 0x0020 in UCS2 and UTF-16 */
          if (UNIV_UNLIKELY(len & 1)) {      // ★ 奇数长度 = 写方向剥掉了「半个空格」
            /* A 0x20 has been stripped from the column. Pad it back. */
            if (pad < field_end) {
              *pad++ = 0x20;                 // ★ 先手动补回那一字节
            }
          }
      }
      row_mysql_pad_col(templ->mbminlen, pad, field_end - pad);  // 其余按 mbminlen 填满
      break;

    case DATA_BLOB:
      /* Store a pointer to the BLOB buffer to dest: the BLOB was
      already copied to the buffer in row_sel_store_mysql_rec */
      row_mysql_store_blob_ref(dest, mysql_col_len, data, len);  // 长度 + 8 字节指针
      break;

    case DATA_POINT:
    case DATA_VAR_POINT:
    case DATA_GEOMETRY:
      /* We store all geometry data as BLOB data at server layer. */
      row_mysql_store_geometry(dest, mysql_col_len, data, len);  // 与 BLOB 同构
      break;

    case DATA_MYSQL:
      memcpy(dest, data, len);
      ut_ad(mysql_col_len >= len);
      ut_ad(templ->mbmaxlen >= templ->mbminlen);

      /* If field_no equals to templ->icp_rec_field_no ... The length should
      still be equal, unless the field pointed by icp_rec_field_no has a
      prefix or this is a virtual column. For end range condition check ...
      The length check assert is relaxed for clust_templ_for_sec. */
      ut_ad(templ->is_virtual || templ->mbmaxlen > templ->mbminlen ||
            mysql_col_len == len || clust_templ_for_sec ||
            (field_no == templ->icp_rec_field_no && field->prefix_len > 0));

      /* The following assertion would fail for old tables
      containing UTF-8 ENUM columns due to Bug #9526. */
      ut_ad(!templ->mbmaxlen || !(mysql_col_len % templ->mbmaxlen));
      ut_ad(templ->is_virtual || clust_templ_for_sec ||
            len * templ->mbmaxlen >= mysql_col_len ||
            index->has_row_versions() ||
            (field_no == templ->icp_rec_field_no && field->prefix_len > 0));
      ut_ad(templ->is_virtual || !(field->prefix_len % templ->mbmaxlen));

      /* Pad with spaces. This undoes the stripping done in row0mysql.cc,
      function row_mysql_store_col_in_innobase_format(). */
      if ((templ->mbminlen == 1 && templ->mbmaxlen != 1) ||       // ① 多字节字符集 CHAR
          (templ->is_virtual && mysql_col_len > len)) {           // ② 前缀虚拟列（见下注）
        /* NOTE（第二条件）：前缀虚拟索引取不到完整值（完整虚拟列目前只能在
        server 层算出），而 server 假定返回值恒有尾空格，故补 0x20；
        更优方案是 server 不依赖尾空格判断 CHAR 终结。 */
        memset(dest + len, 0x20, mysql_col_len - len);
      }
      break;

    default:
#ifdef UNIV_DEBUG
    case DATA_SYS_CHILD:
    case DATA_SYS:
      /* These column types should never be shipped to MySQL. */
      ut_d(ut_error);                   // ★ debug：系统列送到 server 即 bug
      [[fallthrough]];

    case DATA_CHAR:
    case DATA_FIXBINARY:
    case DATA_FLOAT:
    case DATA_DOUBLE:
    case DATA_DECIMAL:
      /* Above are the valid column types for MySQL data. */
#endif /* UNIV_DEBUG */
      /* If sec_field value is present then mapping of secondary index records
      to clustered index template happens for end range comparison. So length
      can vary according to secondary index record length. */
      ut_ad((templ->is_virtual && !field) ||
            (field && field->prefix_len
                 ? field->prefix_len == len
                 : clust_templ_for_sec ? 1 : mysql_col_len == len));
      memcpy(dest, data, len);          // 定长类型：无端序问题，直接拷贝
  }
}
```

> ★ **release 下的真实形状**：`#ifdef UNIV_DEBUG` 的 case 标签被摘除后，`DATA_CHAR/FIXBINARY/FLOAT/DOUBLE/DECIMAL/SYS/SYS_CHILD` **全部落到 `default`** 统一 `memcpy`；而 `field`/`clust_templ_for_sec` 两个局部变量和 `IF_DEBUG(field_no, sec_field)` 两个参数**在 release 下整个消失**——所以那些断言在热路径上零成本。

- `dest` 已是 `mysql_rec + templ->mysql_col_offset`（上层算好的列偏移）
- `templ` 是本列的 `mysql_row_templ_t`，转换所需的全部元信息（`type`/`is_unsigned`/`mysql_col_len`/`mbminlen`/`mbmaxlen`/`mysql_length_bytes`）都在里面——**转换函数对 server 层行格式的全部知识都来自模板，不来自字段本身**
- `IF_DEBUG(field_no / sec_field)` 是只用于 assert 的调试参数（inline 包装 `row_sel_field_store_in_mysql_format` 在 release 下把它们整个编译掉）

前置检查：`templ->is_multi_val` 直接 `ib::fatal`——多值索引虚拟列是数组，永远不可能走单值转换，出现即 bug。

switch 逐分支：

**`DATA_INT`：大端 → 小端 + 符号位还原**

```cpp
ptr = dest + len;
for (;;) {
  ptr--;
  *ptr = *data;        // 从高地址往低地址倒着写 = 逐字节倒序
  if (ptr == dest) break;
  data++;
}
if (!templ->is_unsigned) {
  dest[len - 1] = (byte)(dest[len - 1] ^ 128);   // 还原符号位
}
```

两件事：①InnoDB 内部整数是**大端 + 符号位翻转**（offset binary，最高字节 XOR 0x80 存符号），server 层 record[0] 要**小端 + 正常补码**。逐字节倒序实现大小端转换，末字节 `^ 128` 还原符号位——正是写入方向 `row_mysql_store_col_in_innobase_format` 的逆变换。②`ut_ad(mysql_col_len == len)`：定长类型长度必然一致。

**`DATA_VARCHAR` / `DATA_VARMYSQL` / `DATA_BINARY`：长度头与补空格**

```cpp
if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR) {
  dest = row_mysql_store_true_var_len(dest, len, templ->mysql_length_bytes);
  memcpy(dest, data, len);            // 真 VARCHAR：长度头 + 数据，后面不 pad
  break;
}
ut_memcpy(dest, data, len);           // CHAR 类：先拷数据

/* mbminlen switch: Unicode 特例 */
switch (templ->mbminlen) {
  case 4:  ut_a(!(len & 3));  break;   // UTF-32：不允许剥出半个字符
  case 2:
    if (UNIV_UNLIKELY(len & 1)) {
      if (pad < field_end) *pad++ = 0x20;   // UCS2/UTF-16：补回被剥掉的半个空格
    }
}
row_mysql_pad_col(templ->mbminlen, pad, field_end - pad);   // 尾部补空格
```

- **真 VARCHAR**（`mysql_type == DATA_MYSQL_TRUE_VARCHAR`）：先写 1 或 2 字节长度头（`row_mysql_store_true_var_len`，小端 `mach_write_to_1/2_little_endian`；1 还是 2 字节由 `templ->mysql_length_bytes` 决定——注意这**只在行格式里**用，键格式恒 2 字节），再 memcpy 数据，**不补空格**（VARCHAR 长度头已明确告诉 server 实际长度，多余空格会改变值语义）。
- **CHAR 类**：memcpy 后 `row_mysql_pad_col` 按 `mbminlen` 补空格——1 字节字符集补 `0x20`，UCS2/UTF-16 补 `0x00 0x20`，UTF-32 补 4 字节。这是写入方向"剥尾部空格"（`row_mysql_store_col_in_innobase_format`）的逆操作：InnoDB 行里 CHAR 不存尾空格，读回时补回。
- **mbminlen==2 的奇偶特例**（最隐晦）：UCS2/UTF-16 里空格是两字节 `0x00 0x20`，写入方向按字符剥空格时可能只剥掉一个字节，留下奇数长度。这里 `len & 1` 检测到这个半字符，手动补一个 `0x20` 把它凑成完整的空格——之后 `row_mysql_pad_col` 再按两字节对齐补，才不会把 `0x00` 补在字符串中间。

**`DATA_BLOB`：只存"长度 + 指针"，不拷数据**

```cpp
row_mysql_store_blob_ref(dest, mysql_col_len, data, len);
```

`row_mysql_store_blob_ref` 把 dest 整个 `memset` 成 0，然后写**两样东西**：前 `col_len - 8` 字节是 BLOB 长度（小端，1–4 字节由列宽决定），后 8 字节是**指向数据的指针**。这印证了上一节"本地小 BLOB 也必须先拷进 blob_heap"的原因——record[0] 里的 BLOB 字段从来不是数据本身，而是"长度 + 外置指针"。

**`DATA_POINT` / `DATA_VAR_POINT` / `DATA_GEOMETRY`**：`row_mysql_store_geometry`，与 BLOB 同构（长度 + 指针）——几何数据在 server 层一律按 BLOB 列处理。

**`DATA_MYSQL`：多字节字符集的定长 CHAR，memcpy + 断言群 + 补空格**

```cpp
memcpy(dest, data, len);
/* 6 个 ut_ad 断言（详见下）+ 补空格 */
if ((templ->mbminlen == 1 && templ->mbmaxlen != 1) ||
    (templ->is_virtual && mysql_col_len > len)) {
  memset(dest + len, 0x20, mysql_col_len - len);
}
```

`DATA_MYSQL` 的准确语义是 **"any charset fixed length char"**（`data0type.h:86-90`）——即**非 latin1 字符集的定长 CHAR**（如 utf8mb4 的 `CHAR(N)`）：InnoDB 按定长存，server 层语义是字符类。

⚠️ **一处常见误解要纠正**：它与 ENUM/SET **无关**——ENUM/SET 在 InnoDB 侧走 `DATA_INT`（`ha_innodb.cc:8050-8054` 原文注释 "ENUM and SET ... return (DATA_INT)"，按整数存）。这恰好解释了**键读节里那个 `type != DATA_INT` 的排除检查**（`row0sel.cc:2390-2391`）——ENUM/SET 在键里的表示与整数相同，不能被 true VARCHAR 的 `+2` 长度头逻辑误伤。

该支共 **6 个 `ut_ad` 断言**（`:2627`、`:2628`、`:2642-2644`、`:2648`、`:2652-2655`、`:2656`），每个都在**放松**一个"长度必然相等"的假设：

- 允许 `mysql_col_len == len` 不成立的情形：①`clust_templ_for_sec`（二级索引存列前缀，长度天然更短，end-range 比较只需要前缀）②ICP 检查的前缀列（`field_no == templ->icp_rec_field_no && field->prefix_len > 0`）③虚拟列
- 补空格的第一个条件 `mbminlen == 1 && mbmaxlen != 1`：多字节字符集（如 utf8mb4）的 CHAR 需要尾部空格填充；第二个条件 `is_virtual && mysql_col_len > len`：前缀虚拟索引取不到完整值，但 server 层按约定**期待值恒有尾空格**，所以把不完整部分一律填 `0x20`（注释明说这是 fixup，更优的方案是让 server 别依赖尾空格判断字符串终结，但现状如此）

**`default`：定长二进制类型直接 memcpy**

```cpp
ut_ad((templ->is_virtual && !field) ||
      (field && field->prefix_len ? field->prefix_len == len
                                  : clust_templ_for_sec ? 1 : mysql_col_len == len));
memcpy(dest, data, len);
```

覆盖 `DATA_CHAR`/`DATA_FIXBINARY`/`DATA_FLOAT`/`DATA_DOUBLE`/`DATA_DECIMAL`——定长类型无大小端问题（FLOAT/DOUBLE 在 InnoDB 里本就是按 IEEE-754 字节序存的），直接搬字节。而 `DATA_SYS`/`DATA_SYS_CHILD`（`DB_ROW_ID` 等系统列）在 debug 下 `ut_error`：**它们永远不会被送给 server**。

### row_mysql_convert_row_to_innobase：INSERT 方向的 record[0] → dtuple

与上面 SELECT 方向（`row_sel_store_mysql_rec` → `row_sel_field_store_in_mysql_format_func`）**互为镜像**——这条链的另一半。99 行，按模板逐列转换：

```cpp
static void row_mysql_convert_row_to_innobase(
    dtuple_t *row,            /* in/out: 字段类型信息已拷好 */
    row_prebuilt_t *prebuilt, /* in: template_type 必须是 ROW_MYSQL_WHOLE_ROW */
    const byte *mysql_rec,    /* in: MySQL 行的格式；注意 row 可能是指向它的指针 */
    mem_heap_t **heap) {      /* in/out: 用于深拷贝 multi-value / blob 数据 */
  ut_ad(prebuilt->template_type == ROW_MYSQL_WHOLE_ROW);
  ut_ad(prebuilt->mysql_template);

  for (i = 0; i < prebuilt->n_template; i++) {
    bool is_multi_val = false;
    templ = prebuilt->mysql_template + i;

    // ★ 虚拟列与普通列在 dtuple 里是**分开的两组**，各有独立计数器
    if (templ->is_virtual) {
      dfield = dtuple_get_nth_v_field(row, n_v_col);   // 虚拟列组
      n_v_col++;
      if (dfield_is_multi_value(dfield)) {
        is_multi_val = true;
        n_m_v_col++;
      }
    } else {
      dfield = dtuple_get_nth_field(row, n_col);       // 普通列组
      n_col++;
    }

    // ★ NULL 从 **MySQL 行的位图**读（不是 InnoDB 的 null bitmap）
    if (templ->mysql_null_bit_mask != 0) {
      if (mysql_rec[templ->mysql_null_byte_offset] &
          (byte)(templ->mysql_null_bit_mask)) {
        dfield_set_null(dfield);
        continue;
      }
    }

    if (is_multi_val) {
      innobase_get_multi_value(prebuilt->m_mysql_table, v_col->m_col.ind,
                               dfield, &prebuilt->mv_data[n_m_v_col - 1], 0,
                               dict_table_is_comp(prebuilt->table),
                               prebuilt->heap);
      /* For multi-value data, the deep copy may cost too much. ...
      However, once more virtual column data needs to be calculated later,
      for example, insert by modify, server will overwrite the memory used
      here. So the safest way is a deep copy. */
      // ★ 必须深拷贝：insert by modify 时 server 会重算虚拟列并**覆盖这块内存**
      if (*heap == nullptr) *heap = mem_heap_create(128, UT_LOCATION_HERE);
      dfield_multi_value_dup(dfield, *heap);

    } else {
      // ★ 每列的真正编码 —— 读方向 row_sel_field_store_in_mysql_format_func 的逆操作
      row_mysql_store_col_in_innobase_format(
          dfield, prebuilt->ins_upd_rec_buff + templ->mysql_col_offset,
          true,                              /* MySQL row format data */
          mysql_rec + templ->mysql_col_offset, templ->mysql_col_len,
          dict_table_is_comp(prebuilt->table));

      /* server has issue regarding handling BLOB virtual fields,
      and we need to duplicate it with our own memory here */
      // ★ BLOB 虚拟列也要深拷贝：注释直说 server 在这块"有问题"
      if (templ->is_virtual &&
          DATA_LARGE_MTYPE(dfield_get_type(dfield)->mtype)) {
        if (*heap == nullptr) {
          *heap = mem_heap_create(dfield->len, UT_LOCATION_HERE);
        }
        dfield_dup(dfield, *heap);
      }
    }
  }

  // ★ FTS doc id：若列不是用户提供的（由 server 生成），在这里补分配
  if (prebuilt->table->fts) {
    ut_a(prebuilt->table->fts->doc_col != ULINT_UNDEFINED);
    fts_create_doc_id(prebuilt->table, row, prebuilt->heap);
  }
}
```

四个要点：

1. **虚拟列与普通列在 dtuple 里是两组**（`nth_field` / `nth_v_field` 分开取，两个独立计数器），不是混在一个数组里——这解释了为什么行格式里虚拟列处处是特例。
2. **NULL 从 MySQL 行的位图读**（`mysql_null_byte_offset` / `mysql_null_bit_mask`），不是 InnoDB 的 null bitmap。
3. ★ **只有两处"必须深拷贝"**：多值索引（server 在 insert-by-modify 时会覆盖内存）和 BLOB 虚拟列（注释直说 server 处理有问题）。**其余列只存指针**——所以参数注释强调 `mysql_rec` 在 row 使用期间不能释放。
4. **每列编码委托给 `row_mysql_store_col_in_innobase_format`**，正是读方向 `row_sel_field_store_in_mysql_format_func` 的逆操作（两者头注释互相指认，见上文「定位」）——**写方向的 190 行逐分支剖析**（`DATA_INT` 的"从尾向前 + `^= 128`"、CHAR 剥空格、true VARCHAR 的 1/2 字节长度头、BLOB 引用）见 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md)「转换桥梁」章，本篇只给调用点、不重复展开。

⇒ 两个方向合起来看：**读方向是"逐列解码 + 补空格 + 写长度头"，写方向是"逐列编码 + 剥空格 + 写长度头"**，共用同一套 `mysql_row_templ_t` 模板——**转换的全部知识都在模板里**，两个方向各自取用。

### ★ clust_templ_for_sec：扫描二级索引、但模板基于主键

`row_search_mvcc` 与 `row_sel_store_mysql_rec` 里各有一个同名局部标志，语义一致：

```cpp
/* True if we are scanning a secondary index, but the template is based
on the primary index. */
bool clust_templ_for_sec;
```

**赋值**：

- `row_search_mvcc`（`:4839`）：`clust_templ_for_sec = index != clust_index && prebuilt->need_to_access_clustered;`
- `row_sel_store_mysql_rec`（`:2735`）：`clust_templ_for_sec = (sec_field_no != ULINT_UNDEFINED);`——只要该列在二级索引记录里有对应字段号就算

**取列时的字段号切换**（`:2750`，在 `row_sel_field_store_in_mysql_format` 路径上）：

```cpp
if (clust_templ_for_sec) {
  clust_field_no = field_no;      // 存下模板（基于 PK）的字段号
  field_no = sec_field_no;        // 暂时换成二级索引记录里的字段号
}
/* ...用 sec_field_no 从二级索引记录取数据... */
if (clust_templ_for_sec) {
  field_no = clust_field_no;      // 还原为聚簇索引的字段号
}
```

★★ **为什么能这么做**——因为 **InnoDB 的二级索引记录里隐含主键列**。于是"模板基于 PK 的那些列"中，有一部分是**可以直接从二级索引记录里读出来的**，根本不用回表。这正是"二级索引记录包含 PK"这一物理事实带来的红利，`clust_templ_for_sec` 就是把这件事显式化的标志。

配套处理还有几处，都体现"两套坐标共存"的复杂性：

| 位置 | 处理 |
|---|---|
| `:2637-2644` | end-range 检查的**长度断言放松**：索引可能只存列前缀，`len` 可能小于 `mysql_col_len`，故断言里加 `\|\| clust_templ_for_sec` |
| `:2745` | `rec_offs_validate` 用 `prebuilt_index`（模板所属索引）而非 `rec_index`（记录所属索引） |
| `:2916-2925` | 需要"存储二级索引记录中的所有聚簇列"时，遍历 `prebuilt_index` 的字段并映射回 `rec_index` 的字段号 |
| `:5021` | end_range_cache 构造时 `key_index = clust_index`（模板基于 PK） |

**为什么值得单独理解它**：这是"覆盖索引 / 回表"二分之外的**第三种形态**——不是纯覆盖（全在二级索引里），也不是普通回表（全去聚簇取），而是"**模板按 PK 建、但尽量从二级索引记录里直接抠出 PK 列，剩下的才回表**"。它就是 `row_sel_store_mysql_rec` 里最难的那段坐标换算。

**`sec_field_no` 是怎么算出来的（唯一生产者，`:3006-3034`）**：

★ 先记住一点：上层传给 `row_sel_field_store_in_mysql_format_func` 的 `sec_field` 形参本身**只在 debug 构建存在**（`IF_DEBUG(, ulint sec_field)` 包裹，release 下整个参数消失），它只用来放宽断言。它的值这样构造：

```cpp
// row0sel.cc:3006-3030
ulint field_no = rec_clust ? templ->clust_rec_field_no : templ->rec_field_no;
ulint sec_field_no = ULINT_UNDEFINED;                    // :3009 默认"不是这个场景"

ut_ad(rec_index->get_field(field_no)->prefix_len == 0);  // :3014 常规路径禁止前缀列

if (clust_templ_for_sec) {                               // :3016 仅 end-range 场景进入
  std::vector<const dict_col_t *>::iterator it;
  const dict_field_t *field = rec_index->get_field(field_no);
  const dict_col_t *col = field->col;

  it = std::find(template_col.begin(), template_col.end(), col);   // :3021 反查该列在二级索引里的序号
  if (it == template_col.end()) {
    continue;                                            // :3024 该列不在二级索引 record 里 → 跳过不填
  }
  ut_ad(templ->rec_field_no == templ->clust_rec_field_no);   // :3027 此场景两者必然相等
  sec_field_no = it - template_col.begin();              // :3029 ★ 得到"二级索引字段号"
}
```

其中 `template_col` 是一张**"二级索引字段号 → 表列"映射表**（`:2916-2931`）：

```cpp
if (clust_templ_for_sec) {
  /* Store all clustered index column of secondary index record. */
  for (ulint i = 0; i < dict_index_get_n_fields(prebuilt_index); i++) {   // 遍历二级索引字段
    auto sec_field = dict_index_get_nth_field_pos(rec_index, prebuilt_index, i);  // :2919
    if (sec_field == ULINT_UNDEFINED) {                  // :2922 该字段在 rec_index 里找不到
      template_col.push_back(nullptr);
      continue;
    }
    const auto field = rec_index->get_field(sec_field);
    template_col.push_back(field->col);                  // :2929 记下"表列"
  }
}
```

★ `dict_index_get_nth_field_pos(index, index2, n)` 的语义：**第 n 个 `index2` 字段在 `index` 中的位置**（找不到返回 `ULINT_UNDEFINED`，`dict0dict.h:739-748`）——所以 `template_col` 的**下标是二级索引字段号**、值是表列，`sec_field_no` 正好可配"按二级索引算出的 `offsets`"去取数据。★ 这也是 `:2745` 为何要写 `ut_ad(rec_offs_validate(rec, clust_templ_for_sec ? prebuilt_index : rec_index, offsets))`——**offsets 属于被扫描的二级索引、模板属于聚簇索引**，两套坐标必须分开校验。★ 两个不豁免的例外：`DATA_INT` 分支**不用** `clust_templ_for_sec`（整型定长、索引里不存在"整型前缀"，`len` 必等于 `mysql_col_len`，且符号位翻转依赖精确长度）；外部大字段路径（`:2824-2826`）硬编码传 `ULINT_UNDEFINED`（那条分支永远按 `clust_rec_field_no` 走）。

#### 完整链路：三个函数接力

上面讲了 `sec_field_no` 的构造，下面把它放回完整链路——`clust_templ_for_sec` 串起三个函数：

```
row_search_mvcc                    判定：index != clust_index && need_to_access_clustered（:4837-4840）
  └─ row_sel_store_mysql_rec       建 template_col 映射、反查 sec_field_no（:2896-3070）
       └─ row_sel_store_mysql_field    借出/归还字段号，取数据（:2704-2894）
            └─ row_sel_field_store_in_mysql_format_func   落 MySQL 格式 + 放宽断言（:2505-2716，见上节 switch 全貌）
```

#### `row_sel_store_mysql_rec`：取行总入口（`row0sel.cc:2896-3070`）

**头注释与签名**（参数语义是理解全链路的钥匙）：

```cpp
/** Convert a row in the Innobase format to a row in the MySQL format.
Note that the template in prebuilt may advise us to copy only a few
columns to mysql_rec, other columns are left blank. ...
@param[in]      rec_clust           true if rec is in the clustered index
                                    instead of index which could belong to prebuilt->index
@param[in]      rec_index           index of rec
@param[in]      prebuilt_index      prebuilt->index
@param[in]      clust_templ_for_sec true if rec belongs to secondary index
                                    but the prebuilt->template is in
                                    clustered index format and it
                                    is used only for end range comparison
@param[in,out]  blob_heap           If not null then use this heap for BLOBs
@return true on success, false if not all columns could be retrieved */
bool row_sel_store_mysql_rec(byte *mysql_rec, row_prebuilt_t *prebuilt,
                             const rec_t *rec, const dtuple_t *vrow,
                             bool rec_clust, const dict_index_t *rec_index,
                             const dict_index_t *prebuilt_index,
                             const ulint *offsets, bool clust_templ_for_sec,
                             lob::undo_vers_t *lob_undo, mem_heap_t *&blob_heap) {
  std::vector<const dict_col_t *> template_col;      // :2903 映射表（构造见上）
  DBUG_TRACE;

  ut_ad(rec_clust || rec_index == prebuilt_index);   // :2907 非聚簇记录时 rec_index 必为 prebuilt->index
  ut_ad(!rec_clust || rec_index->is_clustered());    // :2908

  /* 若 blob_heap 就是 prebuilt 自己的堆 → 可安全清空复用；否则清空责任在调用方 */
  if (blob_heap != nullptr && blob_heap == prebuilt->blob_heap) {
    mem_heap_empty(blob_heap);                       // :2913 ★ 每次行转换清空（只装一行的 BLOB）
  }
```

★ 参数三元组 `rec` / `rec_index` / `prebuilt_index` 是全场最易混的：**`rec` 是实际被扫描的记录**（end-range 场景下在二级索引上）、**`rec_index` 是它所属的索引**、**`prebuilt_index` 是模板所描述的索引**（`prebuilt->index`），`offsets` 与 `prebuilt_index` 同源。★ `blob_heap` 是**引用参数**（`mem_heap_t *&`）——函数内惰性创建的堆会**回写调用方**，供后续 end-range 检查继续复用。

**主循环的三种分发（`:2933-3037`）**：

```cpp
for (ulint i = 0; i < prebuilt->n_template; i++) {   // :2933 只遍历模板（本次需交付的列）
  const auto templ = &prebuilt->mysql_template[i];

  if (templ->is_multi_val) {                         // :2946 continue 之一：多值列
    ut_ad(templ->is_virtual);
    continue;
  }

  if (templ->is_virtual && rec_index->is_clustered()) {   // :2952 虚拟列（聚簇索引上）
    /* Skip virtual columns if it is not a covered search or virtual key read is not requested. */
    if ((prebuilt_index != nullptr && !dict_index_has_virtual(prebuilt_index)) ||
        (!prebuilt->read_just_key && !prebuilt->m_read_virtual_key) || !rec_clust) {
      continue;                                      // :2959 continue 之二：非覆盖/非虚拟键读
    }

    dict_v_col_t *col = dict_table_get_nth_v_col(rec_index->table,
                                                 templ->clust_rec_field_no);   // :2964
    ut_ad(vrow);
    const auto dfield = dtuple_get_nth_v_field(vrow, col->v_pos);   // :2968 ★ 值来自 vrow
    if (dfield_get_type(dfield)->mtype == DATA_MISSING) {   // :2975 未物化虚拟列
      ut_ad(prebuilt->m_read_virtual_key);
      /* If it is part of index key the data should have been materialized. */
      ut_ad(prebuilt_index->get_col_pos(col->v_pos, false, true) == ULINT_UNDEFINED);
      continue;                                      // :2983
    }

    if (dfield->len == UNIV_SQL_NULL) {              // 虚拟列 NULL → 置位 + 拷默认值
      mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;
      memcpy(mysql_rec + templ->mysql_col_offset,
             (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
             templ->mysql_col_len);
    } else {
      row_sel_field_store_in_mysql_format(           // :2993 ★ 数据来自 dfield，不碰记录
          mysql_rec + templ->mysql_col_offset, templ, rec_index,
          templ->clust_rec_field_no, (const byte *)dfield->data, dfield->len,
          ULINT_UNDEFINED);
      if (templ->mysql_null_bit_mask) {
        mysql_rec[templ->mysql_null_byte_offset] &= ~(byte)templ->mysql_null_bit_mask;
      }
    }
    continue;                                        // :3003
  }

  /* ★ 普通列：字段号依 rec_clust 选择 */
  ulint field_no = rec_clust ? templ->clust_rec_field_no : templ->rec_field_no;  // :3006-3007
  ulint sec_field_no = ULINT_UNDEFINED;              // :3009 默认不走 sec 借位

  ut_ad(rec_index->get_field(field_no)->prefix_len == 0);  // :3014 常规路径禁止前缀列

  if (clust_templ_for_sec) {                         // :3016 ★ 反查（详见上段）
    std::vector<const dict_col_t *>::iterator it;
    const dict_field_t *field = rec_index->get_field(field_no);
    const dict_col_t *col = field->col;
    it = std::find(template_col.begin(), template_col.end(), col);   // :3021
    if (it == template_col.end()) continue;          // :3024 continue 之三：该列不在索引 record 里
    ut_ad(templ->rec_field_no == templ->clust_rec_field_no);   // :3027
    sec_field_no = it - template_col.begin();        // :3029
  }

  if (!row_sel_store_mysql_field(mysql_rec, prebuilt, rec, rec_index,
                                 prebuilt_index, offsets, field_no, templ,
                                 sec_field_no, lob_undo, blob_heap)) {
    return false;                                    // :3035 ★ 一列失败 → 整行失败（调用方 skip 该行）
  }
}
```

★ `std::find` 按**列指针**匹配：`nullptr` 占位不会误配，但**同一列在索引里重复出现时只返回首个**——这依赖 `template_col` 按聚簇字段序插入的顺序保证正确性。

**收尾：FTS doc_id 与返回值（`:3039-3052`）**：

```cpp
if (dict_table_has_fts_index(rec_index->table)) {    // :3044
  /* FIXME: We only need to read the doc_id if an FTS indexed column is being updated. */
  if ((rec_index->is_clustered() && !clust_templ_for_sec) ||
      prebuilt->fts_doc_id_in_read_set) {
    prebuilt->fts_doc_id =
        fts_get_doc_id_from_rec(rec_index->table, rec, rec_index, nullptr);
  }
}
return true;                                         // :3052 全列成功
```

★ `clust_templ_for_sec` 为真时 `rec_index` 是二级索引 → 第一个条件为假，只有读集合**真声明**需要 `FTS_DOC_ID` 才取。

#### `row_sel_store_mysql_field`：单字段落盘（`row0sel.cc:2704-2894`）

```cpp
/** Convert a field in the Innobase format to a field in the MySQL format.
@param[in]      field_no        templ->rec_field_no or templ->clust_rec_field_no
                                or templ->icp_rec_field_no or sec field no if
                                clust_templ_for_sec is true
@param[in]      sec_field_no    Secondary index field no if the secondary index
                                record but the prebuilt template is in
                                clustered index format and used only for end
                                range comparison. */
// :2725 [[nodiscard]]：调用方必须消费返回值（false = 该行不可用）
[[nodiscard]] static bool row_sel_store_mysql_field(
    byte *mysql_rec, row_prebuilt_t *prebuilt, const rec_t *rec,
    const dict_index_t *rec_index, const dict_index_t *prebuilt_index,
    const ulint *offsets, ulint field_no, const mysql_row_templ_t *templ,
    ulint sec_field_no, lob::undo_vers_t *lob_undo, mem_heap_t *&blob_heap) {
  DBUG_TRACE;
  const byte *data;
  ulint len;
  ulint clust_field_no = 0;                            // :2734 借出时的暂存位
  bool clust_templ_for_sec = (sec_field_no != ULINT_UNDEFINED);   // :2735 与上层同名的局部副本

  ut_ad(templ); ut_ad(prebuilt->default_rec);
  ut_ad(templ >= prebuilt->mysql_template);
  ut_ad(templ < &prebuilt->mysql_template[prebuilt->n_template]);
  ut_ad(clust_templ_for_sec || field_no == templ->clust_rec_field_no ||    // :2742 ★ 三种合法来源
        field_no == templ->rec_field_no || field_no == templ->icp_rec_field_no);
  ut_ad(rec_offs_validate(
      rec, clust_templ_for_sec == true ? prebuilt_index : rec_index, offsets));  // :2745

  /* :2748 ★ 借出端：备份聚簇字段号，把 field_no 换成二级索引字段号 */
  if (clust_templ_for_sec) {
    clust_field_no = field_no;                         // :2751
    field_no = sec_field_no;                           // :2752
  }

  if (rec_offs_nth_extern(rec_index, offsets, field_no)) {   // :2755 ★ 路径一：外部存储字段
    /* Copy an externally stored field to a temporary heap */
    ut_a(!prebuilt->trx->has_search_latch);            // :2758 读 LOB 期间不得持 search latch
    ut_ad(field_no == templ->clust_rec_field_no);      // :2759
    ut_ad(templ->type != DATA_POINT);                  // :2760

    mem_heap_t *heap;
    if (DATA_LARGE_MTYPE(templ->type)) {               // :2764 大字段 → 复用 prebuilt->blob_heap
      if (blob_heap == nullptr) {
        blob_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);   // :2765 惰性创建
      }
      heap = blob_heap;
    } else {
      heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);   // :2771 否则用临时堆
    }

    dict_index_t *clust_index = rec_index->table->first_index();     // :2778 LOB 归属聚簇索引
    const page_size_t page_size = dict_table_page_size(rec_index->table);
    size_t lob_version = 0;

    data = lob::btr_rec_copy_externally_stored_field(  // :2784 ★ 真正从溢出页读 LOB
        prebuilt->trx, clust_index, rec, offsets, page_size, field_no, &len,
        &lob_version, dict_index_is_sdi(rec_index), heap);

    if (data == nullptr) {                             // :2788 外部字段尚未写完的极端情形
      /* ... just return false ... only seen by recv_recovery_rollback_active()
      or any TRX_ISO_READ_UNCOMMITTED transactions. */
      if (heap != blob_heap) mem_heap_free(heap);
      ut_a((!prebuilt->idx_cond &&
            prebuilt->m_mysql_handler->end_range != nullptr) ||
           (prebuilt->trx->isolation_level == TRX_ISO_READ_UNCOMMITTED));  // :2804
      return false;                                    // :2807 调用方 skip 该行
    }

    if (lob_undo != nullptr) {                         // :2810 登记 LOB undo（可回滚）
      ulint local_len;
      const byte *field_data =
          rec_get_nth_field_instant(rec, offsets, field_no, rec_index, &local_len);
      const byte *field_ref = field_data + local_len - BTR_EXTERN_FIELD_REF_SIZE;  // :2815 20B 外存引用
      lob::ref_t ref(const_cast<byte *>(field_ref));
      lob_undo->apply(clust_index, field_no, const_cast<byte *>(data), len,
                      lob_version, ref.page_no());     // :2818
    }

    ut_a(rec_field_not_null_not_add_col_def(len));
    row_sel_field_store_in_mysql_format(mysql_rec + templ->mysql_col_offset,
                                        templ, rec_index, field_no, data, len,
                                        ULINT_UNDEFINED);   // :2824 ★ 此路径不传 sec
    if (heap != blob_heap) mem_heap_free(heap);        // :2828
  } else {
    /* :2832 路径二/三：字段在行内 */
    data = rec_get_nth_field_instant(rec, offsets, field_no, rec_index, &len);  // :2834

    if (len == UNIV_SQL_NULL) {                        // :2836 ★ NULL 分支
      /* MySQL assumes that the field for an SQL NULL value is set to the default value. */
      ut_ad(templ->mysql_null_bit_mask);
      mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;  // :2843
      memcpy(mysql_rec + templ->mysql_col_offset,
             (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
             templ->mysql_col_len);                    // :2845
      return true;                                     // :2848 ★ NULL 也算"成功交付"
    }

    if (DATA_LARGE_MTYPE(templ->type) || DATA_GEOMETRY_MTYPE(templ->type)) {   // :2851 ★ 路径三
      /* It is a BLOB field locally stored in the InnoDB record: we MUST copy its
      contents to prebuilt->blob_heap here because row_sel_field_store_in_mysql_format()
      stores a pointer to the data, and the data passed to us will be invalid as soon
      as the mini-transaction is committed and the page latch ... is released.
      For DATA_POINT, it's stored like CHAR in InnoDB, but it should be a BLOB field
      in MySQL layer. So we still treated it as BLOB here. */
      mem_heap_t *heap{};
      if (blob_heap == nullptr) blob_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);
      heap = blob_heap;
      data = static_cast<byte *>(mem_heap_dup(heap, data, len));   // :2872 ★ 深拷贝到 blob_heap
    }

    /* :2875 ★ 归还端：把 field_no 还原回聚簇字段号 */
    if (clust_templ_for_sec) field_no = clust_field_no;   // :2877

    row_sel_field_store_in_mysql_format(mysql_rec + templ->mysql_col_offset,
                                        templ, rec_index, field_no, data, len,
                                        sec_field_no);    // :2880 ★ sec 原样透传给底层（仅 debug 用）
  }

  ut_ad(rec_field_not_null_not_add_col_def(len));      // :2885

  if (templ->mysql_null_bit_mask) {                    // :2887 可空列且值非 NULL → 清 NULL 位
    mysql_rec[templ->mysql_null_byte_offset] &= ~(byte)templ->mysql_null_bit_mask;
  }
  return true;                                         // :2893
}
```

★ **三条取数路径的触发条件**：① `rec_offs_nth_extern()` 为真（字段溢出到外部页）→ `btr_rec_copy_externally_stored_field` 真读 LOB（大字段复用 `blob_heap`、非大字段用临时堆）；② 行内且 `DATA_LARGE_MTYPE`/`DATA_GEOMETRY_MTYPE`（未溢出的 BLOB/TEXT/GEOMETRY/POINT）→ **必须 `mem_heap_dup` 到 `blob_heap`**（因为落盘只写"指向数据的指针"，mtr 提交/页闩释放后原地址失效）；③ 普通字段 → 底层函数内直接 memcpy。★ **借出只在入口一次（`:2752`）、归还只在末尾一次（`:2877`）**——中间所有 `rec_get_nth_field_instant(..., field_no, ...)` 都用二级索引字段号；注意 `:2824` 的外部字段路径用的是"仍未归还"的字段号（该路径本就不属于 end-range 的典型场景）。

#### end-range 两处使用点

**① 页尾预比较（`:5003-5050`）**——扫到页的 supremum 且空转 100 轮后，用页内最后一条记录与 end range 比对：

```cpp
if (page_rec_is_supremum(rec)) {                       // :5003
  if (prev_rec != nullptr && !prebuilt->innodb_api &&
      prebuilt->m_mysql_handler->end_range != nullptr &&
      prebuilt->idx_cond == false && end_loop >= 100) {   // :5010
    dict_index_t *key_index = prebuilt->index;         // :5014 默认用 prebuilt 索引
    if (end_range_cache == nullptr) {                  // :5016 惰性分配缓存行（MySQL 行格式）
      end_range_cache = static_cast<byte *>(ut::malloc_withkey(
          UT_NEW_THIS_FILE_PSI_KEY, prebuilt->mysql_row_len));
    }
    if (clust_templ_for_sec) {
      /** Secondary index record but the template based on PK. */
      key_index = clust_index;                         // :5024 ★ 模板按 PK
    }
    /** Create offsets based on prebuilt index. */
    offsets = rec_get_offsets(prev_rec, prebuilt->index, offsets,
                              ULINT_UNDEFINED, UT_LOCATION_HERE, &heap);   // :5028 offsets 按被扫索引
    if (row_sel_store_mysql_rec(end_range_cache, prebuilt, prev_rec, prev_vrow,
                                clust_templ_for_sec, key_index, prebuilt->index,
                                offsets, clust_templ_for_sec,               // :5031
                                prebuilt->get_lob_undo(), prebuilt->blob_heap)) {
      if (row_search_end_range_check(end_range_cache, prev_rec, prebuilt,
                                     clust_templ_for_sec, offsets, record_buffer)) {
        if (next_buf != nullptr) prebuilt->m_end_range = true;   // :5042
        err = DB_RECORD_NOT_FOUND;                     // :5045 越界 → 终止扫描
        goto normal_return;
      }
    }
  }
}
```

★ 三个实参组合正是"记录在二级索引、模板在聚簇"的**精确表达**：第 5 参 `rec_clust` 传 `clust_templ_for_sec`（记录确实在二级索引上 → false）、第 6 参 `rec_index = key_index`（此时 = `clust_index`）、第 9 参开关 = `clust_templ_for_sec`。

**② 正常取行后的即时比较（`:5660-5711`，节选）**：

```cpp
if (!row_sel_store_mysql_rec(next_buf, prebuilt, result_rec, vrow,
                             result_rec != rec,                    // :5662 rec_clust
                             result_rec != rec ? clust_index : index,
                             prebuilt->index, offsets,
                             false,                                // :5664 ★ 此处开关传 false
                             nullptr, prebuilt->blob_heap)) { ... goto next_rec; }

if (record_buffer != nullptr && prebuilt->m_mysql_handler->end_range != nullptr) {   // :5679
  /* 若 end range 引用虚拟列且正在读聚簇索引，next_buf 缺虚拟列值，
     需按二级索引取 offsets 以便从索引里读出虚拟列 */
  if (clust_templ_for_sec && prebuilt->m_mysql_handler->m_virt_gcol_in_end_range) {  // :5684
    if (sec_offsets == nullptr) { rec_offs_init(sec_offsets_); sec_offsets = sec_offsets_; }
    sec_offsets = rec_get_offsets(rec, index, sec_offsets, ULINT_UNDEFINED,
                                  UT_LOCATION_HERE, &heap);    // :5696 ★ 按二级索引生成
  }
  if (row_search_end_range_check(next_buf, rec, prebuilt, clust_templ_for_sec,
                                 sec_offsets, record_buffer)) {   // :5701
    if (next_buf != buf) record_buffer->remove_last();   // :5705 回滚最后一条
    next_buf = prev_buf;
    err = DB_RECORD_NOT_FOUND;                           // :5708
    goto normal_return;
  }
}
```

★ 与①的差别：常规落盘传 `clust_templ_for_sec = false`（`rec_clust` 已由 `result_rec != rec` 表达），该开关只在后面的 end-range 检查里**作为标志**使用；`sec_offsets` 特意按**二级索引**生成，专服务"end-range 引用虚拟列、需从二级索引取虚拟列值"这一情形。

#### `row_search_end_range_check`：消费比较结果（`:3870-3917`）

```cpp
// @param[in]  clust_templ_for_sec  true if rec 属于二级索引但模板基于聚簇索引
// @retval true 越界 / false 仍在范围内
static bool row_search_end_range_check(byte *mysql_rec, const rec_t *rec,
                                       row_prebuilt_t *prebuilt,
                                       bool clust_templ_for_sec,
                                       const ulint *offsets,
                                       Record_buffer *record_buffer) {
  const auto handler = prebuilt->m_mysql_handler;
  ut_ad(handler->end_range != nullptr);                // :3886 前提

  /* 非覆盖二级索引扫描时，mysql_rec 在 handler 调 update_generated_read_fields()
  之前没有虚拟列值。若 end-range 条件引用虚拟列，需先从二级索引把虚拟列值补进 mysql_rec 再比较。 */
  if (clust_templ_for_sec && handler->m_virt_gcol_in_end_range) {   // :3893
    ut_ad(offsets != nullptr);
    for (ulint i = 0; i < prebuilt->n_template; ++i) {
      const auto &templ = prebuilt->mysql_template[i];
      if (templ.is_virtual && templ.icp_rec_field_no != ULINT_UNDEFINED &&
          !row_sel_store_mysql_field(mysql_rec, prebuilt, rec, prebuilt->index,
                                     prebuilt->index, offsets,
                                     templ.icp_rec_field_no, &templ,        // :3901 ★ 用 icp 字段号
                                     ULINT_UNDEFINED, nullptr,
                                     prebuilt->blob_heap)) {
        return (false);
      }
    }
  }

  if (handler->compare_key_in_buffer(mysql_rec) > 0) {   // :3908 ★ 优化器注入的比较回调
    if (record_buffer != nullptr) record_buffer->set_out_of_range(true);   // :3910 标记越界
    return (true);                                       // :3913 越界
  }
  return (false);                                        // :3916 在范围内
}
```

★ **为什么 end-range 只需前缀即可**：`compare_key_in_buffer` 逐列比较的是优化器注入的**前缀 key**（`end_range`）——所以即便 `mysql_rec` 是用聚簇模板从二级索引记录"部分映射"填出来的，只要**前缀列**被正确落盘，比较即成立。`clust_templ_for_sec` 的作用是保证"前缀列集合能正确映射"（含必要时从二级索引补虚拟列），**而非要求整行完备**——这正是那些长度断言能"点名赦免"的正当性来源。★ 注意 `:3901` 补虚拟列时用的是 `templ.icp_rec_field_no`（**ICP 字段号**）且 `rec_index = prebuilt_index = prebuilt->index`、`sec_field_no = ULINT_UNDEFINED`——即这条路径按"普通模板列"取数，而非 `clust_templ_for_sec` 的借位方式。

#### 与 ICP 的 `icp_rec_field_no` 是同源问题

| | `clust_templ_for_sec`（`sec_field_no`） | `icp_rec_field_no`（`set_templ_icp`） |
|---|---|---|
| 触发场景 | 二级索引扫描 + 回表、**模板基于 PK**（`index != clust_index && need_to_access_clustered`） | ICP 下推时 **扫描索引 ≠ 建模板索引**（`scan_index != index`） |
| 反查方式 | `dict_index_get_nth_field_pos` + `template_col` + `std::find`（**按列身份**） | `scan_index->get_col_pos(col_position, ...)`（**按列位置**，带前缀回退） |
| 源码 | `row0sel.cc:3016-3030` | `ha_innodb.cc:8360-8368`（见 [`../server/handler.md`](../server/handler.md) 的 `set_templ_icp`） |

★ **同源**：都是"**模板字段号 ≠ 记录所在索引的字段号**"导致的取数错位，都靠"拿列身份到目标索引里反查字段位置"修正——**证据**是两者最终汇入**同一个断言豁免集合**（`row0sel.cc:2742-2743` 的 `clust_rec_field_no || rec_field_no || icp_rec_field_no`），即三种"字段号来源"本质是同一种"借位"问题。★ **不同**：`icp_rec_field_no` 处理的是"扫描索引 ≠ 模板索引"，用**列位置**反查并允许前缀回退；`clust_templ_for_sec` 处理的是"二级索引记录 + 聚簇模板"这一更具体形态，用**列身份**反查。

**★ `mysql_row_templ_t` 就是承载这三种"字段号来源"的结构**（⚠️ **它在 `row0mysql.h:491`，不在 `row0sel.h`**）：

```cpp
struct mysql_row_templ_t {
  ulint col_no;                  /*!< column number of the column */
  ulint rec_field_no;            /*!< field number of the column in an
                                 Innobase record in the current index */
  ulint clust_rec_field_no;      /*!< field number of the column in an
                                 Innobase record in the clustered index */
  ulint icp_rec_field_no;        /*!< field number of the column in an
                                 Innobase record in the current index;
                                 only defined for columns that could be
                                 used to evaluate a pushed down index
                                 condition and/or end-range condition */
  ulint mysql_col_offset;        /*!< offset of the column in the MySQL row format */
  ulint mysql_col_len;           /*!< length of the column in the MySQL row format */
  ulint mysql_mvidx_len;         /*!< index length on multi-value array */
  ulint mysql_null_byte_offset;  /*!< MySQL NULL bit byte offset */
  ulint mysql_null_bit_mask;     /*!< bit mask to get the NULL bit, zero if not nullable */
  ulint type;                    /*!< column type in Innobase mtype numbers DATA_CHAR... */
  ulint mysql_type;              /*!< MySQL type code; this is always < 256 */
  ulint mysql_length_bytes;      /*!< TRUE_VARCHAR 用 1 还是 2 字节存长度 */
  ulint charset;                 /*!< MySQL charset-collation code */
  ulint mbminlen;                /*!< minimum length of a char, in bytes */
  ulint mbmaxlen;                /*!< maximum length of a char, in bytes */
  ulint is_unsigned;             /*!< != 0 ⇒ unsigned integer */
  ulint is_virtual;              /*!< != 0 ⇒ virtual column */
  ulint is_multi_val;            /*!< != 0 ⇒ Multi-Value Array virtual column */
};
```

> ★★ **"MySQL 侧位置 ↔ InnoDB 侧位置"的映射核心是 6 个字段**：MySQL 侧 `mysql_col_offset` / `mysql_col_len` / `mysql_null_byte_offset` / `mysql_null_bit_mask`；InnoDB 侧 `rec_field_no`（当前扫描索引）/ `clust_rec_field_no`（聚簇）/ `icp_rec_field_no`（ICP 与 end-range）。
> ★ `row_sel_store_mysql_rec` 的那句 `rec_clust ? templ->clust_rec_field_no : templ->rec_field_no` 就是靠前两个字段切换的。

```cpp
  unsigned n_template : 10;                /*!< number of elements in the template */
  unsigned null_bitmap_len : 10;           /*!< SQL NULL bitmap 字节数 */
  unsigned need_to_access_clustered : 1;   /*!< 二级索引取不到某列 ⇒ 必须回表 */
  unsigned templ_contains_blob : 1;        /*!< 模板含 DATA_LARGE_MTYPE 列 */
  unsigned templ_contains_fixed_point : 1; /*!< 模板含 DATA_POINT */
```

> ★ `n_template` 是 **10 bit 位域** → 上限 1023（MySQL 单表 1017 列刚好装得下）。
> ⚠️ `templ_contains_blob` 看的是 **MySQL 侧类型** `DATA_LARGE_MTYPE()`，与 InnoDB 是否真 off-page **无关**（VARCHAR 也可 off-page 但不置此位）——这正是为什么还要一个 `templ_contains_fixed_point`（DATA_POINT 在 InnoDB 里不是 BLOB 类型）。

**★★ `clust_templ_for_sec` 的推导不在 `ha_innodb.cc`**（该文件全文 0 处该标识符）：

```cpp
  // row0sel.cc:4837
  clust_index = index->table->first_index();

  clust_templ_for_sec =
      index != clust_index && prebuilt->need_to_access_clustered;
```

> ★ `need_to_access_clustered` 的上游是 `build_template_needs_field()`（`ha_innodb.cc:8165`）：**当前索引不含某列（`index_contains == false`）但查询又要读它** → 返回非 null 并置位。所以 `clust_templ_for_sec` 的完整语义是"扫的是二级索引 **且** 模板里有二级索引取不到的列"。

**★ `row_search_end_range_check()` 完整源码**（`row0sel.cc:3864`）：

```cpp
static bool row_search_end_range_check(byte *mysql_rec, const rec_t *rec,
                                       row_prebuilt_t *prebuilt,
                                       bool clust_templ_for_sec,
                                       const ulint *offsets,
                                       Record_buffer *record_buffer) {
  const auto handler = prebuilt->m_mysql_handler;
  ut_ad(handler->end_range != nullptr);

  /* When reading from non-covering secondary indexes, mysql_rec won't
  have the values of virtual columns until the handler has called
  update_generated_read_fields(). If the end-range condition refers to a
  virtual column, we may have to copy its value from the secondary index
  before evaluating the condition. */
  if (clust_templ_for_sec && handler->m_virt_gcol_in_end_range) {
    ut_ad(offsets != nullptr);
    for (ulint i = 0; i < prebuilt->n_template; ++i) {
      const auto &templ = prebuilt->mysql_template[i];
      if (templ.is_virtual && templ.icp_rec_field_no != ULINT_UNDEFINED &&
          !row_sel_store_mysql_field(mysql_rec, prebuilt, rec, prebuilt->index,
                                     prebuilt->index, offsets,
                                     templ.icp_rec_field_no, &templ,
                                     ULINT_UNDEFINED, nullptr,
                                     prebuilt->blob_heap)) {
        return (false);
      }
    }
  }

  if (handler->compare_key_in_buffer(mysql_rec) > 0) {   // ★ server 层的 range 上界比较
    if (record_buffer != nullptr) {
      record_buffer->set_out_of_range(true);             // ★ 让预取缓存停止填充
    }
    return (true);
  }
  return (false);
}
```

> ⚠️ **注意这条路径补虚拟列时用的是 `templ.icp_rec_field_no`**，且 `rec_index = prebuilt_index = prebuilt->index`、`sec_field_no = ULINT_UNDEFINED`——即**按"普通模板列"取数**，而不是 `clust_templ_for_sec` 的借位方式。

**★ 预取缓存有两条来源，不是 InnoDB 自己 malloc 的数组**：

```cpp
static Record_buffer *row_sel_get_record_buffer(const row_prebuilt_t *prebuilt) {
  if (prebuilt->m_mysql_handler == nullptr) return nullptr;
  return prebuilt->m_mysql_handler->ha_get_record_buffer();
  //  ★ 优先用 server 层提供的 Record_buffer
}
```

```cpp
  cached_rec = record_buffer
                   ? record_buffer->record(prebuilt->fetch_cache_first)
                   : prebuilt->fetch_cache[prebuilt->fetch_cache_first];
  //  ★ 没有才用 InnoDB 自己的 fetch_cache[8]（MYSQL_FETCH_CACHE_SIZE = 8）
```

```cpp
static inline void row_sel_prefetch_cache_init(row_prebuilt_t *prebuilt) {
  ut_ad(row_sel_get_record_buffer(prebuilt) == nullptr);
  sz = UT_ARR_SIZE(prebuilt->fetch_cache) * (prebuilt->mysql_row_len + 8);
  ptr = static_cast<byte *>(ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sz));
  for (i = 0; i < UT_ARR_SIZE(prebuilt->fetch_cache); i++) {
    /* A user has reported memory corruption in these buffers in Linux.
    Put magic numbers there to help to track a possible bug. */
    mach_write_to_4(ptr, ROW_PREBUILT_FETCH_MAGIC_N);
    ptr += 4;
    prebuilt->fetch_cache[i] = ptr;
    ptr += prebuilt->mysql_row_len;
    mach_write_to_4(ptr, ROW_PREBUILT_FETCH_MAGIC_N);
    ptr += 4;
  }
}
```

> ★ 每个 slot **前后各 4 字节 magic**（`ROW_PREBUILT_FETCH_MAGIC_N`）——注释明说是为排查一例 Linux 上的内存损坏报告加的。

---

## MVCC 可见性：快照读与二级索引回表

> `row_search_mvcc` 的可见性判定段：快照读（聚簇索引）→ 二级索引回表确认 → 旧版本构建与 undo 回溯。理解它的前提是 [`mvcc.md`](mvcc.md) 的 Read View 部分。

### 快照读（聚簇索引）

```c
if (index == clust_index) {
  if (!lock_clust_rec_cons_read_sees(rec, index, offsets, trx_get_read_view(trx))) {
    row_sel_build_prev_vers_for_mysql(...);   /* 不可见 → 沿 undo 回溯 */
    if (old_vers == nullptr) goto next_rec;    /* 该行在 read view 创建前还不存在 */
    rec = old_vers;
  }
}
```

`lock_clust_rec_cons_read_sees` 从聚簇记录读 `DB_TRX_ID`，交给 read view 的 `changes_visible` 判定。

### ★ 二级索引不能直接信，必须回表确认

`lock_sec_rec_cons_read_sees` 的注释是核心证据：

> "NOTE that a non-clustered index page contains so little information on its modifications that also in the case false, the present version of rec may be the right, but we must check this from the clustered index record."

原因：**二级索引记录本身不携带 `DB_TRX_ID` / `DB_ROLL_PTR`**（它们只存在于聚簇记录），二级索引只能拿到"页级"的 `PAGE_MAX_TRX_ID` 粗粒度提示：

```c
trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));
return (view->sees(max_trx_id));   /* sees(id) == id < m_up_limit_id */
```

- 返回 true（`max_trx_id < m_up_limit_id`）→ 肯定可见，可信
- 返回 false → **不确定**，`goto requires_clust_rec` 回表，用聚簇记录的 `DB_TRX_ID` 做精确判断，必要时再沿聚簇 undo 回溯

### ★ `row_search_idx_cond_check`：ICP 在引擎侧的执行

ICP（Index Condition Pushdown）是**优化器决定下推、引擎执行**：server 把"能用索引字段判断的那部分 WHERE"交给 InnoDB，在**回表之前**先过滤，省掉不必要的回表。主循环里 `switch (row_search_idx_cond_check(...))` 就是执行入口。

```cpp
static ICP_RESULT row_search_idx_cond_check(
    byte *mysql_rec,          /* out: MySQL 格式的行（仅当返回 ICP_MATCH 才有效） */
    row_prebuilt_t *prebuilt,
    const rec_t *rec,         /* in: InnoDB 记录 */
    const ulint *offsets) {
  ICP_RESULT result;
  ulint i;
  ut_ad(rec_offs_validate(rec, prebuilt->index, offsets));

  // ★ 没有下推条件 ⇒ 直接算"匹配"，一次都不进下面的循环
  //   ⇒ ICP 关闭时这个函数退化成一个布尔判断，这是"零开销"的落地
  if (!prebuilt->idx_cond) {
    return (ICP_MATCH);
  }
  MONITOR_INC(MONITOR_ICP_ATTEMPTS);

  if (prebuilt->blob_heap != nullptr) {
    mem_heap_empty(prebuilt->blob_heap);   // 每次评估前清空，避免跨行累积
  }

  // ★ 只转换"ICP 需要的那几列"，不是整行
  for (i = 0; i < prebuilt->idx_cond_n_cols; i++) {
    const mysql_row_templ_t *templ = &prebuilt->mysql_template[i];
    /* Skip virtual columns */
    if (templ->is_virtual) {
      continue;                            // 虚拟列不参与 ICP
    }
    if (!row_sel_store_mysql_field(
            mysql_rec, prebuilt, rec, prebuilt->index, prebuilt->index, offsets,
            templ->icp_rec_field_no, templ, ULINT_UNDEFINED, nullptr,
            prebuilt->blob_heap)) {
      return (ICP_NO_MATCH);               // 转换失败视为不匹配
    }
  }
```

转换完这少数几列，就**回调 server 层求值**：

```cpp
  /* We assume that the index conditions on
  case-insensitive columns are case-insensitive. The
  case of such columns may be wrong in a secondary
  index, if the case of the column has been updated in the past,
  or a record has been deleted and a record inserted in a different case. */
  // ★ ICP 的本质：引擎拿着刚转好的字段，回调 server 用真正的 Item 表达式树求值
  //   ⇒ 判定逻辑在 server（复用优化器那套），引擎只负责"准备数据 + 拿结果"
  // ⚠️ 陷阱：大小写不敏感列的二级索引里，大小写可能是"历史遗留的错值"
  //    （列被 update 过大小写，或 delete 后 insert 了不同大小写）
  //    ⇒ 所以这里假定"下推到 case-insensitive 列的条件本身是 case-insensitive 的"
  result = innobase_index_cond(prebuilt->m_mysql_handler);
```

三种结果的走向：

```cpp
  switch (result) {
    case ICP_MATCH:
      /* Convert the remaining fields to MySQL format.
      If this is a secondary index record, we must defer
      this until we have fetched the clustered index record. */
      // ★ 匹配才补转"剩下的列"；但二级索引只带部分列，必须等回表拿到
      //   聚簇记录后才能转全 ⇒ 这里直接跳过（need_to_access_clustered 为真时）
      if (!prebuilt->need_to_access_clustered ||
          prebuilt->index->is_clustered()) {
        if (!row_sel_store_mysql_rec(mysql_rec, prebuilt, rec, nullptr, false,
                                     prebuilt->index, prebuilt->index, offsets,
                                     false, nullptr, prebuilt->blob_heap)) {
          ut_ad(prebuilt->index->is_clustered());
          return (ICP_NO_MATCH);
        }
      }
      MONITOR_INC(MONITOR_ICP_MATCH);
      return (result);

    case ICP_NO_MATCH:
      MONITOR_INC(MONITOR_ICP_NO_MATCH);
      return (result);         // ⇒ 主循环 goto next_rec，省掉一次回表

    case ICP_OUT_OF_RANGE:
      MONITOR_INC(MONITOR_ICP_OUT_OF_RANGE);
      // ★ 越界不只是"这条不匹配"——意味着后面也不会再匹配了
      //   ⇒ 通知预取缓存停止，否则会继续预取注定被丢弃的行
      const auto record_buffer = row_sel_get_record_buffer(prebuilt);
      if (record_buffer) {
        record_buffer->set_out_of_range(true);
      }
      return (result);         // ⇒ 主循环 goto idx_cond_failed，结束扫描
  }
  ut_error;
}
```

⇒ **ICP 的收益就在 `ICP_NO_MATCH` 这一支**：不回表。而 `ICP_OUT_OF_RANGE` 更进一步——连扫描都提前结束。三种结果都有 `MONITOR_INC` 计数（对应 `Innodb_icp_attempts` / `_match` / `_no_match` / `_out_of_range` 状态变量）。

### ★ 预取缓存的入队/出队 与 range 边界检查

主循环里"命中一行"后并不立即返回，而是先**入预取缓存**凑一批，server 下次取行时再**出队**。

**入队**：★ 只在 ICP 开启时才真正拷贝

```cpp
static inline void row_sel_enqueue_cache_row_for_mysql(
    byte *mysql_rec, row_prebuilt_t *prebuilt) {
  /* For non ICP code path the row should already exist in the
  next fetch cache slot. */
  // ★ 非 ICP 路径下，主循环里 row_sel_store_mysql_rec 已经把行直接写进了
  //   下一个缓存槽位 ⇒ 这里不需要再拷一次，只递增计数
  //   ICP 路径下才要把 mysql_rec 拷进最后一个槽
  if (prebuilt->idx_cond) {
    byte *dest = row_sel_fetch_last_buf(prebuilt);
    ut_memcpy(dest, mysql_rec, prebuilt->mysql_prefix_len);
  }
  ++prebuilt->n_fetch_cached;
}
```

**出队**：三条拷贝路径，取完 rewind

```cpp
static inline void row_sel_dequeue_cached_row_for_mysql(
    byte *buf, row_prebuilt_t *prebuilt) {
  /* The row is cached in the server-provided buffer, if there
  is one. If not, get it from our own prefetch cache.*/
  // server 提供的 record_buffer 优先，否则用 InnoDB 自己的 fetch_cache
  const auto record_buffer = row_sel_get_record_buffer(prebuilt);
  cached_rec = record_buffer
                   ? record_buffer->record(prebuilt->fetch_cache_first)
                   : prebuilt->fetch_cache[prebuilt->fetch_cache_first];

  if (UNIV_UNLIKELY(prebuilt->keep_other_fields_on_keyread)) {
    row_sel_copy_cached_fields_for_mysql(buf, cached_rec, prebuilt);

  } else if (prebuilt->mysql_prefix_len > 63) {
    /* The record is long. Copy it field by field, in case
    there are some long VARCHAR column of which only a
    small length is being used. */
    // ★ 长记录逐字段拷：长 VARCHAR 声明很长但实际只用一小段，
    //   整块拷 mysql_prefix_len 会把大量没用的字节也拷过去
    UNIV_MEM_INVALID(buf, prebuilt->mysql_prefix_len);
    /* First copy the NULL bits. */
    ut_memcpy(buf, cached_rec, prebuilt->null_bitmap_len);   // 先拷 NULL 位图
    /* Then copy the requested fields. */
    for (i = 0; i < prebuilt->n_template; i++) {
      templ = prebuilt->mysql_template + i;
      /* Skip virtual columns */
      if (templ->is_virtual) { ... continue; }
      ut_a(!templ->is_multi_val);   // 多值列一定是虚拟列
      row_sel_copy_cached_field_for_mysql(buf, cached_rec, templ);
    }
  } else {
    ut_memcpy(buf, cached_rec, prebuilt->mysql_prefix_len);  // 短记录整块拷
  }

  prebuilt->n_fetch_cached--;
  prebuilt->fetch_cache_first++;

  if (prebuilt->n_fetch_cached == 0) {
    /* All the prefetched records have been returned.
    Rewind so that we can insert records at the beginning
    of the prefetch cache or record buffer. */
    // ★ 全取完就 rewind 到 0 并清空 ⇒ 缓存是"环形复用"，不是消费完重新分配
    prebuilt->fetch_cache_first = 0;
    if (record_buffer != nullptr) {
      record_buffer->clear();
    }
  }
}
```

**range 边界检查**：与 ICP 同构——判定也在 server 侧

```cpp
static bool row_search_end_range_check(byte *mysql_rec, const rec_t *rec,
                                       row_prebuilt_t *prebuilt,
                                       bool clust_templ_for_sec,
                                       const ulint *offsets,
                                       Record_buffer *record_buffer) {
  const auto handler = prebuilt->m_mysql_handler;
  ut_ad(handler->end_range != nullptr);

  /* When reading from non-covering secondary indexes, mysql_rec won't
  have the values of virtual columns until the handler has called
  update_generated_read_fields(). If the end-range condition refers to a
  virtual column, we may have to copy its value from the secondary index
  before evaluating the condition. */
  // ★ 坑：非覆盖二级索引时虚拟列的值还没填（要等 server 调 update_generated_read_fields）
  //   ⇒ 若 end_range 条件引用了虚拟列，必须先从二级索引把值拷进来再判定
  if (clust_templ_for_sec && handler->m_virt_gcol_in_end_range) {
    ut_ad(offsets != nullptr);
    for (ulint i = 0; i < prebuilt->n_template; ++i) {
      const auto &templ = prebuilt->mysql_template[i];
      if (templ.is_virtual && templ.icp_rec_field_no != ULINT_UNDEFINED &&
          !row_sel_store_mysql_field(
              mysql_rec, prebuilt, rec, prebuilt->index, prebuilt->index,
              offsets, templ.icp_rec_field_no, &templ, ULINT_UNDEFINED, nullptr,
              prebuilt->blob_heap)) {
        return (false);
      }
    }
  }

  // ★ 与 ICP 一样是"引擎准备数据 + 回调 server 判定"
  if (handler->compare_key_in_buffer(mysql_rec) > 0) {
    // 越界 ⇒ 通知预取缓存停止（与 ICP_OUT_OF_RANGE 的处理一致）
    if (record_buffer != nullptr) {
      record_buffer->set_out_of_range(true);
    }
    return (true);
  }
  return (false);
}
```

返回 true = 已超出 `end_range`，主循环据此结束扫描。

### ★ `Row_sel_get_clust_rec_for_mysql`：回表的完整实现

上一节讲了"为什么必须回表"，落到实现上就是它。★ 注意它是 **functor 类而不是普通函数**——8.0 把它重构成了带缓存的对象：

```cpp
class Row_sel_get_clust_rec_for_mysql {
  const rec_t *cached_clust_rec;   // ★ 上一次回表的聚簇记录
  rec_t *cached_old_vers;          // ★ 上一次回溯出的旧版本

 public:
  Row_sel_get_clust_rec_for_mysql()
      : cached_clust_rec(nullptr), cached_old_vers(nullptr) {}

  dberr_t operator()(row_prebuilt_t *prebuilt, dict_index_t *sec_index,
                     const rec_t *rec, que_thr_t *thr, const rec_t **out_rec,
                     ulint **offsets, mem_heap_t **offset_heap,
                     const dtuple_t **vrow, mtr_t *mtr,
                     lob::undo_vers_t *lob_undo);
};
```

**为什么必须是对象而不是函数**：回表一次要沿聚簇 undo 回溯，代价高。范围扫描时**相邻二级记录常常指向同一条聚簇记录**，缓存住 (聚簇记录 → 旧版本) 后，下次撞到同一条记录可直接复用 `cached_old_vers`，跳过整条回溯链。⇒ 普通函数无状态，做不到这件事。

`operator()` 共 263 行，按五步剖析（省略参数校验与空间索引特例）：

**第一步：用二级记录构造主键元组并定位**

```cpp
  *out_rec = nullptr;
  trx = thr_get_trx(thr);

  // 用二级记录里的主键列，填出聚簇索引的搜索元组 clust_ref
  row_build_row_ref_in_tuple(prebuilt->clust_ref, rec, sec_index, *offsets);

  clust_index = sec_index->table->first_index();   // 聚簇索引 = 表的第一个索引

  // pcur 定位到 sec rec 对应的 clust rec
  prebuilt->clust_pcur->open_no_init(clust_index, prebuilt->clust_ref,
                                     PAGE_CUR_LE, BTR_SEARCH_LEAF, 0, mtr,
                                     UT_LOCATION_HERE);

  clust_rec = prebuilt->clust_pcur->get_rec();
```

**第二步：★ 命中判定——定位成功 ≠ 找到，必须查 `low_match`**

```cpp
  /* Note: only if the search ends up on a non-infimum record is the
  low_match value the real match to the search tuple */
  if (!page_rec_is_user_rec(clust_rec) ||
      prebuilt->clust_pcur->get_low_match() <   /* 回表 search_mode == page_cur_le */
          dict_index_get_n_unique(clust_index)) {
    // 回表失败，没找到 sec 对应 clust rec
```

`PAGE_CUR_LE` 是"小于等于"，**即使主键不存在也会停在前一条记录上**。⇒ 判据是**匹配到的字段数**：`low_match < 聚簇索引唯一键数` 说明主键没完全对上 ⇒ 这条聚簇记录其实不存在（二级记录是残留或 delete-mark 的）。★ 这是回表特有的坑：**拿到 `clust_rec` 不等于找到了对应行**。

**第三步：重算 offsets；然后分锁定读 / 快照读两路**

```cpp
  // 进来的 offsets 是二级索引的，现在换成聚簇记录的
  *offsets = rec_get_offsets(clust_rec, clust_index, *offsets, ULINT_UNDEFINED,
                             UT_LOCATION_HERE, offset_heap);

  if (prebuilt->select_lock_type != LOCK_NONE) {
    /* we are searching the clust rec with a unique condition,
    hence we set a LOCK_REC_NOT_GAP type lock */
    // ★ 等值回表 ⇒ 只锁记录不锁间隙（不可能再插入第二条同主键的记录）
    err = lock_clust_rec_read_check_and_lock(
        lock_duration_t::REGULAR, prebuilt->clust_pcur->get_block(), clust_rec,
        clust_index, *offsets, prebuilt->select_mode,
        static_cast<lock_mode>(prebuilt->select_lock_type), LOCK_REC_NOT_GAP,
        thr);
```

快照读这一路才是缓存真正生效的地方：

```cpp
    /* If the isolation level allows reading of uncommitted data,
    then we never look for an earlier version */
    if (trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
        !lock_clust_rec_cons_read_sees(clust_rec, clust_index, *offsets,
                                       trx_get_read_view(trx))) {
      if (clust_rec != cached_clust_rec) {
        /* The following call returns 'offsets' associated with 'old_vers' */
        err = row_sel_build_prev_vers_for_mysql(
            trx->read_view, clust_index, prebuilt, clust_rec, offsets,
            offset_heap, &old_vers, vrow, mtr, lob_undo);
        ...
        cached_clust_rec = clust_rec;      // ★ 回填缓存，供下一条二级记录复用
        cached_old_vers = old_vers;
      } else {
        err = DB_SUCCESS;
        old_vers = cached_old_vers;        // ★ 命中缓存：直接复用，跳过整条 undo 回溯

        if (old_vers != nullptr) {
          /* The offsets need not be same for the latest version of
          clust_rec and its old version old_vers.  Re-calculate the offsets
          for old_vers. */
          // ★ 同一条记录的最新版与旧版字段布局可能不同 ⇒ offsets 必须重算，不能沿用
          *offsets =
              rec_get_offsets(old_vers, clust_index, *offsets, ULINT_UNDEFINED,
                              UT_LOCATION_HERE, offset_heap);
        }
      }

      if (old_vers == nullptr) {
        goto err_exit;
      }

      clust_rec = old_vers;
    }
```

**第四步：★ 拿到旧版本后，还要确认"它真的对应原二级记录"**

```cpp
    /* If we had to go to an earlier version of row or the
    secondary index record is delete marked, then it may be that
    the secondary index record corresponding to clust_rec
    (or old_vers) is not rec; in that case we must ignore
    such row because in our snapshot rec would not have existed. */
    if (clust_rec &&
        (old_vers || trx->isolation_level <= TRX_ISO_READ_UNCOMMITTED ||
         dict_index_is_spatial(sec_index) ||
         rec_get_deleted_flag(rec, dict_table_is_comp(sec_index->table)))) {
      bool rec_equal;

      err = row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec,
                                             clust_index, thr, rec_equal);
      if (err != DB_SUCCESS) {
        goto err_exit;
      } else if (!rec_equal) {
        clust_rec = nullptr;      // 不对应 ⇒ 这行在快照里本不该存在，丢弃
      }
    }
```

（判定算法见下文「★ row_sel_sec_rec_is_for_clust_rec」）

**第五步：只有锁定读才 `store_position`**

```cpp
func_exit:
  *out_rec = clust_rec;

  /* Store the current position if select_lock_type is not
  LOCK_NONE or if we are scanning using InnoDB APIs */
  // ★ 快照读不需要（后续 NEXT 会重新定位）；锁定读需要，
  //   因为 update / unlock_row() 还要用这个游标找回这一行
  if (prebuilt->select_lock_type != LOCK_NONE || prebuilt->innodb_api) {
    prebuilt->clust_pcur->store_position(mtr);
  }
```

### 回表后的旧版本校验

回表拿到旧版本后，还要 `row_sel_sec_rec_is_for_clust_rec` 验证"这个旧版本聚簇记录是否真的对应原二级索引记录 rec"——**防止通过二级索引访问到快照里本不该存在的行**（比如二级记录是 delete-mark、或回表结果来自更早版本）。

### undo 回溯：row_vers_build_for_consistent_read（外层循环）

```c
version = rec;
for (;;) {
  bool purge_sees = trx_undo_prev_version_build(rec, mtr, version, index, *offsets,
                                                heap, &prev_version, nullptr, vrow, 0, lob_undo);
  if (prev_version == nullptr) { *old_vers = nullptr; break; }   // 回溯到 INSERT undo：快照前不存在
  *offsets = rec_get_offsets(prev_version, index, ...);
  trx_id = row_get_rec_trx_id(prev_version, index, *offsets);
  if (view->changes_visible(trx_id, ...)) {   // ★ 每步可见性判断在外层做
    *old_vers = rec_copy(buf, prev_version, *offsets);   // 可见 → 拷贝到 in_heap 返回
    break;
  }
  version = prev_version;                      // 不可见 → 继续回溯
}
```

- ★ **每步可见性判断在外层循环做**，内层 `trx_undo_prev_version_build` 只负责"单步还原"
- `prev_version == nullptr` → 回溯到 INSERT undo（第一次插入），记录在快照前不存在
- `purge_sees == false` → undo 日志已被 purge 清除，返回 `DB_MISSING_HISTORY`

### 单步回溯：trx_undo_prev_version_build

```c
roll_ptr = row_get_rec_roll_ptr(rec, index, offsets);
if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
  return true;   // INSERT undo：*old_vers 保持 nullptr，调用方判定"快照前不存在"
}
...
trx_undo_get_undo_rec(roll_ptr, rec_trx_id, heap, is_temp, ..., &undo_rec);   // 读 undo 记录（可能已被 purge）
ptr = trx_undo_rec_get_pars(undo_rec, &type, &cmpl_info, ..., &undo_no, &table_id, ...);
ptr = trx_undo_update_rec_get_sys_cols(ptr, &trx_id, &roll_ptr, &info_bits);  // 旧版本的 trx_id/roll_ptr/info_bits
ptr = trx_undo_update_rec_get_update(ptr, index, type, ..., &update, ...);    // 解析 update vector
```

按 undo 类型决定动作：

- **INSERT undo**：`trx_undo_roll_ptr_is_insert` 为真 → 这是该记录第一次插入，返回 `nullptr`
- **DELETE undo**（delete-mark）：undo 记录"本次做了 delete-mark"，update vector 携带**旧 info_bits**，应用后"补回被删除前的状态"（去掉 delete mark）
- **UPDATE undo**：update vector 是被改字段的**旧值**，应用后覆盖当前记录得更新前版本

旧版本的两种构造方式：

```c
if (row_upd_changes_field_size_or_external(index, offsets, update)) {
  // 字段长度或外部存储(BLOB)变化：必须整体重建记录
  entry = row_rec_to_index_entry(rec, index, offsets, heap);
  row_upd_index_replace_new_col_vals(entry, index, update, heap);
  *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);
} else {
  // 字段定长：拷贝当前记录后原地 patch（快路径）
  *old_vers = rec_copy(buf, rec, offsets);
  row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr);
}
```

- 定长字段走**原地 patch**（`rec_copy` + `row_upd_rec_in_place`），只覆盖被 update vector 命中的字段，效率高
- 变长或 BLOB 外存：记录物理布局会变，必须拆成 dtuple → 用旧值替换 → 重新编码成 rec

### 二级旧版本校验：row_sel_sec_rec_is_for_clust_rec（算法）

回表拿到旧版本后，逐字段比"二级记录的键字段"与"聚簇旧版本对应列"：

```c
if (rec_get_deleted_flag(clust_rec, ...)) { is_equal = false; return DB_SUCCESS; }  // 聚簇已 delete-mark
n = dict_index_get_n_ordering_defined_by_user(sec_index);   // 只比用户定义的排序列
for (i = 0; i < n; i++) {
  // 虚拟列：从聚簇行重新计算；普通列：dict_col_get_clust_pos 定位 + rec_get_nth_field_instant 取
  ...
  if (0 != cmp_data_data(col->mtype, col->prtype, true, clust_field, len, sec_field, sec_len)) {
    is_equal = false;   // 不匹配
    goto func_exit;
  }
}
```

**为什么需要**：二级索引不版本化但聚簇版本化。典型失配场景——事务 A 把列 c 从 5 改成 6（二级记录 c=6 插入、c=5 delete-mark），另一个事务的 read view 在 A 提交前创建，回表构造出 c=5 的旧聚簇版本，它显然不对应二级记录 c=6。

- 只遍历**用户定义的排序列**（`n_ordering_defined_by_user`），避免把二级索引内部追加的主键列重复比较
- 前缀索引、外部 BLOB（`row_sel_sec_rec_is_for_blob`）、空间 MBR、多值列各有专门比较分支
- `rec_equal == false` → 这条二级记录在 read view 中不可见或失配，跳过

---

### B-tree 定位：btr_cur_search_to_nth_level

> 本节从「取行」视角概述搜索；B-tree 搜索算法 / 分裂 / 合并 / AHI 的完整机制（latch_mode 全表、8.0 SMO 锁预测裁剪、意图升级重搜等）详见 [`btr.md`](../index/btr.md)。

`pcur->open_no_init` / `open_at_side` 内部真正干活的函数——从根下到叶子定位一条记录。这是"取行"的地基。

### 搜索模式换算：叶子语义 → 非叶语义（经典坑）

node_ptr 的 key 是其子页的**最小记录前缀**，每个 `(key, child_page_no)` 指向 `[key, 下一个 node_ptr 的 key)` 区间：

```cpp
switch (mode) {
  case PAGE_CUR_GE: page_mode = PAGE_CUR_L; break;   // leaf 找 >= X，非叶找 < X 的最大 node_ptr
  case PAGE_CUR_G:  page_mode = PAGE_CUR_LE; break;  // leaf 找 > X，非叶找 <= X 的最大 node_ptr
  default: page_mode = mode; break;
}
```

原因：若非叶也用 `PAGE_CUR_GE`，搜索 15 会命中 node_ptr=20，而 15 在 `[10,20)` 区间，会漏掉目标；用 `PAGE_CUR_L`（严格小于）命中 node_ptr=10，正确定位到 `[10-19]`。

### 主循环：逐层下降（latch coupling）

```cpp
search_loop:
  // 非叶层用 upper_rw_latch（继承 index lock 对应的页锁），叶层用 latch_mode
  ...
  block = buf_page_get_gen(page_id, page_size, rw_latch, root_guess, fetch, ..., mtr);

  if (height == ULINT_UNDEFINED) {
    height = btr_page_get_level(page);    // 第一次到 root，读树高
    index->search_info->root_guess = block;  // 缓存 root
  }

  if (height == 0) {
    // 到叶子：latch coupling 释放沿途所有上层 block
    for (; n_releases < n_blocks; n_releases++)
      mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases], tree_blocks[n_releases]);
    page_mode = mode;                     // 恢复原始搜索模式
  }

  // 页内二分定位（page_cur_search_with_match）
  ...
  if (level != height) {
    // 非叶：读 node_ptr 的 child page no，下降到下一层
    page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets));
    n_blocks++;
    goto search_loop;
  }
```

- **latch coupling**：`tree_blocks[]`/`tree_savepoints[]` 追踪沿途页面。每层拿到子页号后，判断"是否会触发 SMO（分裂/合并）"——不会则释放父页锁（只保留 root），会则保留父页锁直到目标层（因为 SMO 会修改父页）
- **`root_guess`**：缓存的 root block 指针，传给 `buf_page_get_gen` 作 hint

### 页内二分：page_cur_search_with_match（两级二分）

```cpp
// 1) 先对 page directory（slot 数组）二分，直到上下 slot 距离为 1
low = 0;
up = page_dir_get_n_slots(page) - 1;
while (up - low > 1) {
  mid = (low + up) / 2;
  mid_rec = page_dir_slot_get_rec(page_dir_get_nth_slot(page, mid));
  cur_matched_fields = min(low_matched_fields, up_matched_fields);
  cmp = tuple->compare(mid_rec, index, offsets, &cur_matched_fields);
  if (cmp > 0) { low = mid; low_matched_fields = cur_matched_fields; }
  else if (cmp) { up = mid; up_matched_fields = cur_matched_fields; }
  ...
}

// 2) 再对 slot 内 owned records 线性查找，直到 low_rec 与 up_rec 相邻
// 3) 定位：mode <= PAGE_CUR_GE 则 position 到 up_rec，否则 low_rec
```

- **`up_match`/`low_match` 跨层传递公共前缀**：`cur_matched_fields = min(low, up)` 作为下一轮比较的起始匹配数——父子层之间的公共前缀无需重比，这是 B-tree 搜索的经典优化
- `low_match` = 搜索键与左记录匹配的字段数，`up_match` = 与右记录匹配的字段数

### AHI 快速路径（在下降前）

```cpp
if (rw_lock_get_writer(btr_get_search_latch(index)) == RW_LOCK_NOT_LOCKED &&
    latch_mode <= BTR_MODIFY_LEAF && index->search_info->last_hash_succ &&
    !index->disable_ahi && !estimate && !dict_index_is_spatial(index) &&
    UNIV_LIKELY(btr_search_enabled) && !modify_external &&
    btr_search_guess_on_hash(tuple, mode, latch_mode, cursor, has_search_latch, mtr)) {
  btr_cur_n_sea++;
  return;
}
```

`btr_search_guess_on_hash`：`dtuple_hash` 算哈希 → AHI 哈希表取 rec 指针 → `buf_block_from_ahi` 反推 block → `buf_page_get_known_nowait` **nowait 方式上 latch（失败即回退，绝不在 AHI 路径阻塞）** → `btr_search_check_guess` 校验 index id/space/key 匹配。AHI 是"猜测"，每次先 `last_hash_succ = false`，校验通过才置 true。

### 返回后游标状态

`cursor->flag = BTR_CUR_BINARY`（非 AHI 路径）；`cursor->low_match/up_match` 写回（对 `PAGE_CUR_LE` 有意义，用于判断唯一性/回表命中）；叶页的 latch 由 mtr 持有。

---

## 加锁：sel_set_rec_lock 与隐式锁转显式

> 读路径的加锁时机与转换：下面三节讲 `sel_set_rec_lock` 的锁类型分派、隐式锁 → 显式锁的转换（`lock_rec_convert_impl_to_expl`）、以及二级索引隐式锁的反推判定。

### sel_set_rec_lock 的分派

```cpp
if (index->is_clustered()) {
  err = lock_clust_rec_read_check_and_lock(...);   // 聚簇
} else if (dict_index_is_spatial(index)) {
  err = sel_set_rtr_rec_lock(...);                 // 空间索引：predicate lock
} else {
  err = lock_sec_rec_read_check_and_lock(...);     // 二级
}
```

`type` 参数：`LOCK_ORDINARY`（next-key）/ `LOCK_GAP` / `LOCK_REC_NOT_GAP`；`mode`：`LOCK_S`/`LOCK_X`；`sel_mode`：`SELECT_ORDINARY`/`SKIP_LOCKED`/`NOWAIT`。

### 隐式锁 → 显式锁：lock_rec_convert_impl_to_expl

**为什么要转换**：InnoDB 里事务修改聚簇记录时，默认**不创建显式记录锁**，而是把自己的 `trx_id` 写进记录的 `DB_TRX_ID` 字段、undo 地址写进 `DB_ROLL_PTR`——这就是**隐式锁**。任何后来者都能从记录上的 `trx_id` 推断"这条记录最新版本属于哪个事务"。

但隐式锁不是一个 `lock_t` 对象，**不在哈希表的锁队列里**，无法作为 wait-for 图节点，也无法被 `lock_rec_has_to_wait` 检测冲突。所以另一个事务要加锁时，必须先把隐式锁"物化"成显式 `LOCK_X | LOCK_REC_NOT_GAP` 的 `lock_t`：

```cpp
void lock_rec_convert_impl_to_expl(const buf_block_t *block, const rec_t *rec, ...) {
  if (index->is_clustered()) {
    trx_id_t trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);
    trx = trx_rw_is_active(trx_id, true);           // 读 DB_TRX_ID，找活跃事务
  } else {
    trx = lock_sec_rec_some_has_impl(rec, index, offsets);   // 二级：回表反推作者事务
  }
  if (trx != nullptr) {
    lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx, heap_no);
  }
}
```

### lock_rec_lock：快路径 / 慢路径

```cpp
static dberr_t lock_rec_lock(bool impl, select_mode sel_mode, ulint mode, ...) {
  switch (lock_rec_lock_fast(impl, mode, block, heap_no, index, thr)) {
    case LOCK_REC_SUCCESS:         return DB_SUCCESS;
    case LOCK_REC_SUCCESS_CREATED: return DB_SUCCESS_LOCKED_REC;
    case LOCK_REC_FAIL:
      return lock_rec_lock_slow(impl, sel_mode, mode, block, heap_no, index, thr);
  }
}
```

慢路径核心：

```cpp
if (held_lock != nullptr) return DB_SUCCESS;   // 已有足够强的锁

const auto conflicting = lock_rec_other_has_conflicting(mode, block, heap_no, trx);
if (conflicting.wait_for != nullptr) {
  switch (sel_mode) {
    case SELECT_SKIP_LOCKED: return DB_SKIP_LOCKED;
    case SELECT_NOWAIT:      return DB_LOCK_NOWAIT;
    case SELECT_ORDINARY: {
      RecLock rec_lock(thr, index, block, heap_no, mode);
      return rec_lock.add_to_waitq(conflicting.wait_for);   // 挂入等待队列，触发死锁检测
    }
  }
}
lock_rec_add_to_queue(LOCK_REC | mode, block, heap_no, index, trx);
return DB_SUCCESS_LOCKED_REC;
```

- **`lock_t` 结构**：`type_mode` 位图（`LOCK_REC=32`/`LOCK_WAIT=256`/`LOCK_GAP=512`/`LOCK_REC_NOT_GAP=1024`/`LOCK_INSERT_INTENTION=2048`，低 4 位是 S/X 模式）；`lock_rec_t` 含 `page_id` + **位图**（每 heap_no 一位）——同一事务同页同类型多锁合并成一个 `lock_t`
- **冲突检测 `lock_rec_has_to_wait`** 有几条经典规则：纯 gap 锁之间不冲突；记录锁不等待 gap 锁；gap 锁不等待 record-not-gap 锁；**没人需要等待 insert intention 锁**
- `impl=true`（如 UPDATE 的 `lock_clust_rec_modify_check_and_lock`）无冲突时**保持隐式锁**，不创建显式锁；`sel_set_rec_lock` 走 `impl=false`，总是创建显式记录锁

### 二级索引隐式锁的反推：row_vers_impl_x_locked

二级记录**没有 `DB_TRX_ID`**，所以 `lock_sec_rec_some_has_impl` 不能直接读 trx_id，而是先看页头的 `PAGE_MAX_TRX_ID` 粗筛，落在活跃区间才回表：

```cpp
max_trx_id = page_get_max_trx_id(page);
if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id)) {
  trx = nullptr;                                       // 快判断：绝无隐式锁
} else {
  trx = row_vers_impl_x_locked(rec, index, offsets);   // 回表聚簇推断
}
```

`row_vers_impl_x_locked_low` 的核心思想（源码注释有形式化推导）：二级索引不版本化，但"聚簇先改、二级后同步"，因此一条二级记录 S 要么与聚簇当前版本同步、要么与上一版本同步。算法沿聚簇版本链回溯，找 S 的"作者事务"（`row_vers_find_matching` 比较各版本字段值/delete-mark 状态）。

---

### 持久游标：store_position / restore_position

> 本节从「取行」视角讲恢复协议；`btr_pcur_t` 结构全字段、乐观/悲观恢复算法与跨页推进的锁序细节详见 [`btr.md`](../index/btr.md)。

`btr_pcur_t` 的"持久"含义：**释放 page latch、甚至页面被写出 buffer pool / 发生页分裂合并之后，游标依然能靠"记录的排序 key 前缀 + 相对位置"重新定位。**

保存字段 → 恢复用途：

| 保存 | 恢复 |
|---|---|
| `m_block_when_stored` | 乐观恢复：直接用 block 指针，不走 B-tree 搜索 |
| `m_modify_clock` | 乐观判断：clock 没变 ⇒ 页面未改，直接 offset 定位 |
| `m_old_rec`（key 副本） | 悲观恢复：clock 变了（页面被改/淘汰），用 key 从 root 重新搜索 |
| `m_rel_pos` | 恢复后微调：BEFORE/AFTER 需 move_to_next/prev |

这就是 `need_to_process` 的来由——`restore_position` 之后，原记录可能已被删除，需要决定是否重新处理当前记录。

### rel_pos 状态机

```cpp
auto success = pcur->restore_position(latch_mode, mtr, ...);
*same_user_rec = success;

switch (pcur->m_rel_pos) {
  case BTR_PCUR_ON:    // 游标原来正停在一条 user record 上
    if (!success && moves_up) { pcur->move_to_next(mtr); return true; }
    return !success;
  case BTR_PCUR_BEFORE:   // 停在记录前（PAGE_CUR_G/L 定位结果）
    if (moves_up && is_on_user_rec()) { pcur->move_to_next(mtr); return true; }
    return true;
  case BTR_PCUR_AFTER:    // 停在记录后（PAGE_CUR_LE 定位结果）
    if (is_on_user_rec() && !moves_up) pcur->move_to_prev(mtr);
    return true;
  case BTR_PCUR_BEFORE_FIRST_IN_TREE:
  case BTR_PCUR_AFTER_LAST_IN_TREE:
    return true;   // 树首/树尾：由调用方处理边界
}
```

`rel_pos` 各状态含义：

| rel_pos | 含义 |
|---|---|
| `BTR_PCUR_ON` | 游标正停在一条 user record 上 |
| `BTR_PCUR_BEFORE` | 停在该记录**前**（`PAGE_CUR_G`/`L` 定位结果） |
| `BTR_PCUR_AFTER` | 停在该记录**后**（`PAGE_CUR_LE` 定位结果） |
| `BTR_PCUR_BEFORE_FIRST_IN_TREE` | 全树第一条之前 |
| `BTR_PCUR_AFTER_LAST_IN_TREE` | 全树最后一条之后 |

`m_pos_state` 区分恢复方式：`IS_POSITIONED`（精确重新定位）vs `IS_POSITIONED_OPTIMISTIC`（乐观恢复到了同一物理记录，但 rel_pos 是 BEFORE/AFTER，位置还需按方向校正）。

### 乐观 vs 悲观恢复

```cpp
// 乐观恢复：仅当锁模式允许且非 intrinsic 表
if ((latch_mode == BTR_SEARCH_LEAF || ...) && !is_intrinsic()) {
  if (m_block_when_stored.run_with_hint([&](buf_block_t *hint) {
        return hint != nullptr &&
               btr_cur_optimistic_latch_leaves(hint, m_modify_clock, &latch_mode, &m_btr_cur, ...);
      })) {
    if (m_rel_pos == BTR_PCUR_ON) return true;   // 同一物理记录，直接成功
    ...
  }
}

// 悲观恢复：用保存的 key 从 root 重新 search
tuple = dict_index_build_data_tuple(index, m_old_rec, m_old_n_fields, heap);
switch (m_rel_pos) {
  case BTR_PCUR_ON:    mode = PAGE_CUR_LE; break;
  case BTR_PCUR_AFTER: mode = PAGE_CUR_G;  break;
  case BTR_PCUR_BEFORE: mode = PAGE_CUR_L; break;
}
open_no_init(index, tuple, mode, latch_mode, 0, mtr, ...);   // 内部即 btr_cur_search_to_nth_level
```

- **乐观恢复**：`buf_page_optimistic_get` 直接对保存的 block 上 latch，检查 `m_modify_clock` 是否与保存值一致——**clock 没变 ⇒ 页面未被修改 ⇒ 记录位置仍有效**，无需从 root 重走
- **悲观恢复**：clock 变了（页面被改/淘汰）→ 用 `m_old_rec` 重建搜索 tuple，按 rel_pos 映射搜索模式（`ON→LE` / `AFTER→G` / `BEFORE→L`），`open_no_init` 从 root 完整执行 `btr_cur_search_to_nth_level`
- **`same_user_rec`（out）** = 恢复在"排序前缀完全相同"的 user record 上（只有乐观 ON 或悲观 ON 且 `cmp_dtuple_rec==0` 才 true）；**`need_to_process`（返回值）** = 恢复/校正后游标是否停在需要立即处理的位置

## 真名勘误与结构性事实

**★★ 真名勘误全表**：

| 常见写法 | 8.0.39 真相 |
|---|---|
| `convert_search_mode_to_page_cur_mode()` | **不存在**。真名 `convert_search_mode_to_innobase()`（`ha_innodb.cc:10087`） |
| `ha_innobase::index_read_last()` 不存在 | **存在**（`:10370`），只 1 行，转发 `index_read(..., HA_READ_PREFIX_LAST)` |
| `index_init()` 自己建模板 | **错**。只有一句 `return change_active_index(keynr)`；绑定 + `init_search_tuples_types()` + `build_template()` 全在后者 |
| `index_end()` "currently does nothing" | 注释过时，**代码已经做了 4 件事**（释放 `last_sel_cur`、`active_index = MAX_KEY`、`in_range_check_pushed_down = false`、`m_ds_mrr.dsmrr_close()`） |
| 续读时 `general_fetch()` 会重传原 `mode` | **错**。硬传 `PAGE_CUR_UNSUPP`；真正的 mode 由 `mode = pcur->m_search_mode;` 从持久游标**回读** |
| `HA_READ_KEY_EXACT` 有独立"精确模式" | **错**。换算为 `PAGE_CUR_GE`；"精确"靠 `match_mode = ROW_SEL_EXACT` + `cmp_dtuple_rec()` 二次校验 |
| `HA_READ_PREFIX_LAST` 有专门 page mode | **错**。换算为 `PAGE_CUR_LE` + `ROW_SEL_EXACT_PREFIX` |
| `records_in_range()` 在 InnoDB 没实现 | **存在且完整**（`:16697`）；**结果为 0 时强制改成 1**，防优化器直接判空集 |
| 全表扫描不走索引 | **错**。`rnd_init()` 第一件事就是 `change_active_index(primary_key)`；`rnd_next()` 就是聚簇索引正向扫描 |
| `rnd_pos()` 用特殊定位路径 | **错**。就是 `index_read(pos, ref_length, HA_READ_KEY_EXACT)` |
| `index_read()` 里调 `dtuple_set_n_fields_cmp()` | **没有**。调的是 `dtuple_set_n_fields()`，后者**同时设置** `n_fields` 与 `n_fields_cmp` |
| `btr_pcur_t::move_to_prev_page()` | **不存在**。真名 `move_backward_from_page()`；存在的只有 `move_to_next_page()` |
| `need_to_process` 是多处通用的循环变量 | **只有 2 处**（`:4888` 定义、`:4893` 唯一使用点），且只在 `direction != 0` 分支 |
| `m_stop_tuple` 每次 `index_read` 都生效 | 有条件：必须 `end_range != nullptr && is_reading_range()`，后者只在 `read_range_first/next()` 的 RAII 守卫内为 true |
| 预取缓存是 InnoDB 自己 malloc 的数组 | **两条路**：优先 server 的 `Record_buffer`，没有才用 `fetch_cache[8]` |
| 记录常量在 `include/rem0rec.h` / `include/rem/rec.h` | 真身在 **`storage/innobase/rem/rec.h`** |
| `rec_get_offsets()` 在 `rem0rec.cc` | 在 **`storage/innobase/rem/rec.cc:364`** |
| `mysql_row_templ_t` / `mysql_template` / `n_template` 在 `row0sel.h` | 全在 **`row0mysql.h`**（491 / 640 / 603） |
| `dtuple_set_types_binary()` 在 `data0data.h` | 定义在 **`data0data.ic:581`**；`.h` 只是声明 |
| `cmp_dtuple_rec_with_match()` | **不存在**（8.0.39 已删，只剩注释里的 `@see`）。实际是 `cmp_dtuple_rec_with_match_low()` |
| `row_sel_store_mysql_field()` 是公开函数 | 是 **`static` + `[[nodiscard]]`** |
| `row_sel_fill_vrow()` 是公开 | **`static void`**（`row0sel.cc:4227`） |
| `clust_templ_for_sec` 在 `ha_innodb.cc` 推导 | **不在**（0 处）。推导在 `row0sel.cc:4839` |
| BLOB 在 `..._format_func` 里拷贝 | **错**。数据在外层 `row_sel_store_mysql_field` 已拷进 `blob_heap`，`..._format_func` 只写"长度 + 8 字节指针"的 ref |
| `REC_OFFS_HEADER_SIZE` 固定 | **2（release）/ 4（UNIV_DEBUG）** |
| offsets[0] 是 magic 字段 | offsets[0] = **n_alloc**，offsets[1] = **n_fields**；真正的 magic 是 debug 下的 offsets[2]/[3] |
| `ROW_SEL_PREV` = -1 | **= 2**（`ROW_SEL_NEXT` = 1）。`moves_up` 靠 `direction == ROW_SEL_NEXT` 判定，不是 `< 0` |

**★ 三个"结构性事实"**：

1. **`direction` 只有一个下游产物：`moves_up`**。而 `moves_up` 的默认值是 `false`——**反向是默认，正向才需显式置位**。整个函数里"方向"只影响三件事：PHASE 5 的 `move_to_next` vs `move_to_prev`、`sel_restore_position_for_mysql` 的校正方向、以及 `row_compare_row_to_range` 的 gap 判定。

2. **`general_fetch` 不传搜索条件**——它传 `PAGE_CUR_UNSUPP`，靠 `mode = pcur->m_search_mode` 回读。而 `m_search_mode` 是在 `btr_pcur_t::restore_position()` 末尾用 `m_search_mode = old_mode;` 还原的。**这条链是"持久游标能跨 mtr 续读"的实现基础**。

3. **`rec_offs_base(offsets)[1+i]` 是"字段 i 的结束偏移"而非起始**——这个"错位一格"设计贯穿全部 `rec_offs_*`，也是 `rec_offs_data_size()` = `base[n_fields] & MASK` 能成立的理由（数组末尾多分配的那 1 个槽位就是给它的）。

**★ 一个容易被忽略的性能事实**：PHASE 5 的注释明写——**正向扫描 + ICP 不匹配时，可以在单个 mtr 内扫完整个二级索引树**（不需要回表、不需要重定位游标）。这是 ICP 收益中被低估的部分。反向扫描则相反：因为 InnoDB 的 B-tree 加锁顺序是 left-to-right，反向跨页**必须 `mtr_commit` + `mtr_start` + `restore_position`**，这是 `ORDER BY DESC` 比正向慢的源码级原因。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Locking and Transaction Model*（半一致性读）
- *MySQL 8.0 Reference Manual → Consistent Nonlocking Reads*

**相关文档**
- 可见性判断见 [`mvcc.md`](mvcc.md)
- handler 接口见 [`../server/handler.md`](../server/handler.md)
