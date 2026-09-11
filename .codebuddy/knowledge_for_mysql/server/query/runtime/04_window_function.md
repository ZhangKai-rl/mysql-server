# 04 窗口函数执行内部：partition / peer / frame（算法级）

> 本篇覆盖：`Window` 对象、`WindowIterator`（流式）与 `BufferingWindowIterator`（缓冲）的执行算法、frame 语义、partition/peer 判定。

## 目录

- [先有个整体印象](#先有个整体印象)
- [一、Window 对象与 setup](#一window-对象与-setup)
- [二、流式 vs 缓冲：两种迭代器](#二流式-vs-缓冲两种迭代器)
- [三、frame 语义与 RANGE 三路比较器](#三frame-语义与-range-三路比较器)
- [四、partition 与 peer 的判定](#四partition-与-peer-的判定)
- [五、窗口函数 Item](#五窗口函数-item)
- [六、执行顺序](#六执行顺序)
- [七、深潜补充：frame buffer 表与缓冲读回三层](#七深潜补充frame-buffer-表与缓冲读回三层)

---

## 先有个整体印象

> ⚠️ **版本事实**：窗口函数 Item 类在 **`sql/item_sum.h`**（`item_sum.cc`），不是 `item_windowfunc.h`；`Window` 类没有 `shared_prefix_window`/`m_ordered_by_optimizer`（那是更新版本）。

窗口函数的关键分水岭：**是否需要看"当前行之后的行"**。

- **流式（WindowIterator）**：frame 是 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，只需向前累加，边读边算，**不需要缓冲**。
- **缓冲（BufferingWindowIterator）**：frame 涉及 `FOLLOWING`、`RANGE`、`CUME_DIST`、`NTILE`、`LEAD/LAG` 等，必须**先缓冲一个 partition 再回放**。

```
输入按 partition+order 排好序
  ├─ 流式：check_partition_boundary 触发 WF clear，边读边累加
  └─ 缓冲：buffer_windowing_record 写 frame buffer → process_buffered_windowing_record 判断可输出
```

---

## 一、Window 对象与 setup

### 1.1 关键成员（`sql/window.h:105`）

```cpp
class Window {
  ORDER *m_sorting_order;          // partition+order 合并后的物理排序键
  bool m_needs_frame_buffering;    // 至少一个 WF 需要缓冲
  bool m_needs_peerset;            // 需要当前行 peerset（CUME_DIST）
  bool m_needs_partition_cardinality; // 需要整个 partition 行数（NTILE/LEAD/LAG）
  bool m_row_optimizable;          // ROWS 可用 inversion 优化
  bool m_range_optimizable;        // RANGE 可用 inversion 优化
  bool m_static_aggregates;        // 帧 = UNBOUNDED PRECEDING..UNBOUNDED FOLLOWING
  Mem_root_array<Cached_item *> m_partition_items; // PARTITION BY 缓存比较器
  Mem_root_array<Cached_item *> m_order_by_items;  // ORDER BY 缓存比较器
};
```

`m_partition_items`/`m_order_by_items` 是 `Cached_item` 数组，`Cached_item::cmp()` 语义是"比较当前值并顺手缓存新值"——这是 partition/peer 判定的核心。

### 1.2 setup_windows1 / setup_windows2

- `setup_windows1`（`window.cc:1090`）：解析期装配——解析 PARTITION/ORDER、命名窗口继承 DAG + 环检测、建缓存比较器、`check_window_functions1` 收集需求、`sorting_order()` 合并排序键、`setup_range_expressions` 建 RANGE 比较器。
- `setup_windows2`（`:1308`）：执行期检查（PS 重执行时 border/offset 可能是 `?` 参数）。

`sorting_order()`（`:393`）：把有效 PARTITION BY 克隆后追加 ORDER BY，得到"先分区再排序"的单一排序键。执行时输入流按它排序后，才能顺序判定 partition/peer。

---

## 二、流式 vs 缓冲：两种迭代器

### 2.1 流式 WindowIterator（`window_iterators.cc:1535`）

何时流式：`needs_buffering() == false`（`check_wf_semantics1` 里 `needs_buffer = !(ROWS && UNBOUNDED PRECEDING && CURRENT ROW)`）。

```cpp
int WindowIterator::Read() {
  SwitchSlice(m_join, m_input_slice);
  int err = m_source->Read();                    // 读下一输入行
  SwitchSlice(m_join, m_output_slice);
  if (err != 0) return err;

  copy_funcs(m_temp_table_param, thd(), CFT_HAS_NO_WF);  // 1) 先复制非 WF 表达式
  m_window->check_partition_boundary();                  // 2) 判定 partition 边界
  copy_funcs(m_temp_table_param, thd(), CFT_WF);         // 3) 求值本 window 的 WF
  if (m_window->is_last() && copy_funcs(..., CFT_HAS_WF)) return 1;  // 4) 最后才算依赖 WF 的表达式
  return 0;
}
```

流式下没有 frame buffer，靠"输入已排序 + `check_partition_boundary()` 在边界处触发各 WF 的 `clear()`"。frame 内化在 WF 的累加状态里（SUM 的 sum/count、FIRST/LAST 缓存值）。

### 2.2 缓冲 BufferingWindowIterator

**缓冲什么**：把输出临时表中已算好的、被 WF 引用到的字段整行写进 frame buffer 临时文件。

**跨 partition 的一行多读问题**：要确认某行是 partition 最后一行，必须读到下一 partition 的第一行。此时先把这第一行保存到 `m_special_rows_cache`（key `FBC_FIRST_IN_NEXT_PARTITION=-1`），等旧 partition 处理完再恢复。

`Read()` 状态机（`window_iterators.cc:1586`）：

1. 先排空上次已算好但没返回的行（`m_possibly_buffered_rows`）。
2. 若上一行跨了 partition：恢复特殊行 → `reset_partition_state()` → 缓冲该行。
3. 否则读输入：读一行，复制非 WF 函数，缓冲；读到 partition 边界则先藏起来。
4. 每次缓冲后调 `process_buffered_windowing_record`，判断当前帧是否凑齐、可输出一行。
5. **何时输出一行**：`current_row` 的 `lower_limit`/`upper_limit` 都 ≤ `last_rowno_in_cache`（或已 EOF/新 partition）。

### 2.3 process_buffered_windowing_record（`window_iterators.cc:654`）—— 核心求值器

**(a) 算帧边界**（ROWS 直接代数计算）：

```cpp
case WBT_CURRENT_ROW:       lower_limit = current_row; break;
case WBT_VALUE_PRECEDING:   lower_limit = std::max<int64>(current_row - border, 1); break;
case WBT_VALUE_FOLLOWING:   lower_limit = current_row + border; break;
case WBT_UNBOUNDED_PRECEDING: lower_limit = 1; break;
```

RANGE 则 `lower_limit = w.first_rowno_in_range_frame()`（单调递增起点 hint），`upper_limit = INT64_MAX`（须读完 partition 才能定 peer 末行）。

**(b) 判断是否可输出**：`lower_limit <= last_rowno_in_cache && upper_limit <= last_rowno_in_cache` 或已缓存整个 partition。

**(c) 三种执行路径**（互斥）：

1. **naive 全量扫描**（非 static、非 optimizable）：每个 current_row 把 frame 内每行取回 `copy_funcs(CFT_WF_FRAMING)` 累加。
2. **static aggregates**：整 partition 的 SUM/AVG/FIRST/LAST 相同，只在 partition 第一行算一次，后续复用。
3. **inversion 优化**（`row_optimizable`/`range_optimizable`）：滑动帧移动时 `set_inverse(true)` 让聚合函数"减掉"离开帧的行、"加上"进入帧的行，避免每行重扫。

---

## 三、frame 语义与 RANGE 三路比较器

### 3.1 默认 frame（`sql_yacc.yy:11371`）

- 有 ORDER BY：默认 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- 无 ORDER BY：默认 `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`

### 3.2 RANGE 的 peer 语义

RANGE 的难点是"不知道该跳过多少行"（要看 ORDER BY 值的差）。实现：为每个 ORDER BY 表达式造**三路比较器**（`setup_range_expressions`，`window.cc:240`），实际判定在 `before_or_after_frame`（`:540`）：

```cpp
for (auto it = m_order_by_items.begin(); ...) {
  Cached_item *cur_row = *it;
  Item *candidate = cur_row->get_item();
  ...
  int val = comparators[i].compare();   // 三路比较
  if (val != 0) {
    if (!asc) val = -val;
    return before ? (val < 0) : (val > 0);  // 字典序，一旦不等即可定论
  }
  // 相等，继续比较下一个 ORDER BY 表达式（字典序）
}
```

**RANGE peer 语义**：ORDER BY 值相等的行属于同一 peer，靠"候选行与当前行 ORDER BY 值的字典序比较"判断是否在帧内，而非像 ROWS 那样数行号。NULL 按标准彼此是 peer（`nulls_at_infinity`）。

---

## 四、partition 与 peer 的判定

### 4.1 partition 边界：check_partition_boundary（`window.cc:456`）

```cpp
void Window::check_partition_boundary() {
  bool anything_changed = (m_part_row_number == 0);
  for (Cached_item *item : m_partition_items) {
    anything_changed |= item->cmp();   // 任一 PARTITION BY 列变化 → 新分区
  }
  m_partition_border = anything_changed;
  if (m_partition_border) { m_part_row_number = 1; ... }
  else m_part_row_number++;
}
```

`Cached_item::cmp()` 返回"是否与缓存上次值不同"并缓存新值。输入按 `m_sorting_order`（partition 列在前）排序后，partition 变化 = 任一 PARTITION BY 列值与上一行不同。

### 4.2 peer 判定：in_new_order_by_peer_set（`:528`）

同样用 `Cached_item::cmp()` 判断 ORDER BY 值是否变化。peerset 长度在 `process_buffered_windowing_record` 里从 current_row 向后扫描，直到遇到第一个 ORDER BY 值不相等的行。

---

## 五、窗口函数 Item

| 类 | 位置 | 算法要点 |
|----|------|---------|
| `Item_row_number` | `item_sum.cc:4735` | 流式 partition 边界 `clear()`，`m_ctr++` |
| `Item_rank`（RANK/DENSE_RANK） | `:4807` | 每个 ORDER BY 列一个 `Cached_item`，变化时 `m_rank_ctr += 1 + (dense ? 0 : m_duplicates)` |
| `Item_ntile` | `:5008` | `needs_partition_cardinality`，缓冲完 partition 才算：`full_rounds = N/buckets`、`modulus = N%buckets`，前 modulus 桶多一行 |
| `Item_lead_lag` | `:5604` | LEAD 变负 LAG 统一寻址，`our_offset` 时从 frame buffer 取 `current_row - offset` |
| `Item_sum_sum`（SUM OVER） | `:1984` | `do_inverse()` 决定 add 还是 subtract（inversion 优化） |

`Item_sum::check_wf_semantics1`（`item_sum.cc:404`）是"语义检查 + 需求收集"入口，把 `needs_buffer/needs_peerset/needs_partition_cardinality/opt_*` 汇总进 window。

---

## 六、执行顺序

SQL 逻辑顺序：**FROM/JOIN → WHERE → GROUP BY → HAVING → WINDOW → ORDER BY → LIMIT**。

执行器在 `make_tmp_tables_info` 体现：

1. 先建 GROUP BY/聚合临时表，HAVING 挂在 group 表上（HAVING 在窗口**之前**求值）。
2. 有窗口时**推迟最终 ORDER BY**（`sql_select.cc:4712`：注释明确"too early to do ORDER BY if we have windowing"）。
3. 按 `m_windows` 顺序插入 windowing 临时表步骤（`OT_WINDOWING_FUNCTION`）。
4. windowing 全部完成后，最终 ORDER BY 才在 filesort/流式发送阶段发生。

---


## 七、深潜补充：frame buffer 表与缓冲读回三层

### 7.1 frame buffer 表如何创建（`CreateFramebufferTable`，`sql_select.cc:4260-4318`）

缓冲模式需要一个"回看窗口"的临时表，建表要点：

1. `m_window_frame_buffer = true`（`:4275`）——标记这是 frame buffer 而非常规临时表
2. **剔除 `orig_item->has_wf()` 的列**（`:4285-4293`）——窗口函数列本身不入 buffer（它们是要被计算的，不是要被回看的）
3. `create_tmp_table`（`:4296-4298`）
4. `ReplaceMaterializedItems`（`:4310-4316`）——把表达式里对原列的引用换成对临时表列的引用

### 7.2 缓冲读回的三层函数（`window_iterators.cc`）

| 函数 | 位置 | 做什么 |
|------|------|--------|
| `buffer_windowing_record` | `:255-279` | 写入 buffer。新分区时 `save_special_row(FBC_FIRST_IN_NEXT_PARTITION)` 并**提前 return** |
| `read_frame_buffer_row` | `:285-352` | 读取。从 `m_frame_buffer_positions` 挑"**≤目标且最近**"的 hint（`:297-306`）→ `ha_rnd_pos` → 最多再 1 次 `ha_rnd_next`（`:317-349`，`assert cnt <= 1`） |
| `bring_back_frame_row` | `:404-480` | 恢复。特殊行走 `restore_special_row`；否则记录 `position()`（`:441-445`），最后 `copy_fields(fb_info, thd, true)` **反向拷回**（`:469`） |

> hint 机制的意义：frame 经常要往回读若干行（如 `ROWS BETWEEN 5 PRECEDING`），每次都从头扫太慢——用"最近的一个已知位置"作为起点，最多再前进一次即可。

### 7.3 RANGE 的 NULL 语义（`window.cc:588-607`）

```cpp
nulls_at_infinity = before ? asc : !asc;   // NULL 被视为"无穷远"的方向取决于排序方向
```

当前行是 NULL 时，**只有 NULL 候选才算 peer**（NULL 与任何值都不等，只与 NULL 相等）。

### 7.4 setup_range_expressions 的两条硬约束（`:240-291`）

1. **无 ORDER BY 时 `CURRENT ROW` 被改写为 UNBOUNDED**（`:252-255`）——没有序，"当前行"没有意义
2. **`RANGE N PRECEDING/FOLLOWING` 要求 ORDER BY 恰好 1 列且是数值/时间类型**（`:272-289`）——只有可加减的类型才能算"偏移 N"

### 7.5 参照值如何刷新（`reset_order_by_peer_set`，`:507-526`）

做两件事：**既**刷新 `Cached_item`（用于判定 partition/peer 边界），**又**经 `FindCacheInComparator`（`:497-505`）调 `cache_value()` **刷新三路比较器的右值**——因为 RANGE frame 的比较是拿"当前行的值 ± N"去比，右值每次都要更新。

### 7.6 多窗口与优化位

- `kMaxWindows = 127`（`window.h:83`）
- 多窗口复用排序：`reorder_and_eliminate_sorts`（`window.cc:752`）+ `equal_sort`（`:736`）——相同排序要求的窗口共享一次排序
- 优化位：`m_needs_last_peer_in_frame`（`window.h:148`，`JSON_OBJECTAGG` 需要）、`m_opt_first_row`/`m_opt_last_row`（`:183`/`:189`）、`m_opt_nth_row`/`m_opt_lead_lag`（`:233-234`）

### 7.7 输出节奏的两状态

`BufferingWindowIterator::Read`（`:1586-1680`）与 `ReadBufferedRow`（`:1682-1702`）靠两个状态协作：`m_possibly_buffered_rows`、`m_last_input_row_started_new_partition`。

注意：`CFT_HAS_NO_WF` 的条件**必须在 input slice 下求值**（`:1656-1659`）——它引用的是输入列，若在 output slice 求值会拿到错误的值。

## 参考

**标准 / 官方文档**
- **SQL:2011 标准**（窗口函数的 `PARTITION BY` / `ORDER BY` / frame 语义定义）
- *MySQL 8.0 Reference Manual → Window Functions*
- *MySQL 8.0 Reference Manual → Window Function Frame Specification*（`ROWS` / `RANGE`）

