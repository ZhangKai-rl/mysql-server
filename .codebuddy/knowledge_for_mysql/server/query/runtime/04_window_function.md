# 04 窗口函数执行内部：partition / peer / frame（算法级）

> 本篇覆盖：`Window` 对象、`WindowIterator`（流式）与 `BufferingWindowIterator`（缓冲）的执行算法、frame 语义、partition/peer 判定。

## 目录

- [先有个整体印象](#先有个整体印象)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、Window 对象与 setup](#一window-对象与-setup)
- [二、流式 vs 缓冲：两种迭代器](#二流式-vs-缓冲两种迭代器)
- [三、frame 语义与 RANGE 三路比较器](#三frame-语义与-range-三路比较器)
- [四、partition 与 peer 的判定](#四partition-与-peer-的判定)
- [五、窗口函数 Item](#五窗口函数-item)
- [六、执行顺序](#六执行顺序)
- [七、深潜补充：frame buffer 表与缓冲读回三层](#七深潜补充frame-buffer-表与缓冲读回三层)

---

## 先有个整体印象

> ⚠️ **版本事实（8.0.39 实测）**：
> - 窗口函数 Item 类在 **`sql/item_sum.h` / `sql/item_sum.cc`**，不是 `item_windowfunc.h`；`Window` 类没有 `shared_prefix_window`/`m_ordered_by_optimizer`。
> - ★ **两个迭代器在 `sql/iterators/window_iterators.{h,cc}`**，**不是** `sql/window_iterators.cc`（后者不存在）。`sql/` 下只有 `window.cc` / `window.h` / `window_lex.h` / `parse_tree_window.{cc,h}`。
> - ★ **不存在** `Item_sum::do_inverse()`——inversion 是各 Item 在 `val_real()/val_decimal()/val_int()` 里读 `m_window->do_inverse()` 分支实现的（`Window::do_inverse()` 在 `window.h:1267`）。
> - ★ **不存在** `MAX_WINDOWS`——上限是 `constexpr int kMaxWindows = 127`（`window.h:83`）。
> - ★ `process_buffered_windowing_record` / `buffer_windowing_record` / `read_frame_buffer_row` / `bring_back_frame_row` 是 `window_iterators.cc` 里**匿名 namespace 的自由函数**，不是 `Window` 的成员函数；`CreateFramebufferTable` 也是自由函数（`sql_select.cc:4260`，不是 `JOIN::` 成员）。

窗口函数的关键分水岭：**是否需要看"当前行之后的行"**。

- **流式（WindowIterator）**：frame 是 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，只需向前累加，边读边算，**不需要缓冲**。
- **缓冲（BufferingWindowIterator）**：frame 涉及 `FOLLOWING`、`RANGE`、`CUME_DIST`、`NTILE`、`LEAD/LAG` 等，必须**先缓冲一个 partition 再回放**。

```
输入按 partition+order 排好序
  ├─ 流式：check_partition_boundary 触发 WF clear，边读边累加
  └─ 缓冲：buffer_windowing_record 写 frame buffer → process_buffered_windowing_record 判断可输出
```

---

## 设计思想与理论基础

### 窗口函数解决什么问题

聚合（`GROUP BY`）会**塌缩行数**：N 行输入 → G 组输出。窗口函数保留 N 行，同时让每行都能访问"它所在分区的聚合上下文"。

形式上，窗口函数 = **分区（PARTITION BY）+ 排序（ORDER BY）+ 帧（frame）** 三元组，为每行算出一个值。

### 根本约束：是否需要看"当前行之后的行"

这是窗口函数执行器设计的**唯一分水岭**：

| | frame 范围 | 能否流式 | 典型例子 |
|---|---|---|---|
| **流式** | 只到 CURRENT ROW（`UNBOUNDED PRECEDING .. CURRENT ROW`） | ✅ 边读边累加 | `ROW_NUMBER()`、`SUM() OVER (ROWS UNBOUNDED PRECEDING)` |
| **缓冲** | 涉及 `FOLLOWING` / `RANGE` / 整个 partition | ❌ 必须缓冲一个 partition | `LEAD/LAG`、`NTILE`、`CUME_DIST` |

原因很直白：**算当前行需要未来的行，就只能先攒着**。这条约束直接决定了用哪个迭代器（本文第二章）。

### inversion：滑动窗口的增量优化

朴素实现：每个 frame 重新计算 ⇒ O(n × frame_size)。
优化：frame 是滑动的，相邻两个 frame 只差"离开的一行"与"进入的一行" ⇒ **增量维护**：

```
new_value = old_value ⊕ entering_row ⊖ leaving_row
```

对应 `Window` 的 `m_row_optimizable`（ROWS 可用）与 `m_range_optimizable`（RANGE 可用）两个标志。

⚠️ **理论限制**：inversion 要求聚合有**逆运算**。加法有逆（减法），故 `SUM`/`COUNT`/`AVG` 可行；但 **`MAX`/`MIN` 没有逆运算**——移除一个元素后无法得知新的极值（除非改用单调栈等结构），字符串聚合同理。所以 inversion 不是万能的。

#### 判定链：两级条件 + 函数级"一票否决"

```cpp
// sql/window.cc  check_window_functions1()
m_static_aggregates =
    (m_frame->m_from->m_border_type == WBT_UNBOUNDED_PRECEDING &&
     m_frame->m_to->m_border_type == WBT_UNBOUNDED_FOLLOWING);
// If static aggregates, inversion isn't necessary
m_row_optimizable   = (m_frame->m_query_expression == WFU_ROWS)  && !m_static_aggregates;
m_range_optimizable = (m_frame->m_query_expression == WFU_RANGE) && !m_static_aggregates;

for (Item_sum &wf : m_functions) {
  Window_evaluation_requirements reqs;   // 默认 row_optimizable = range_optimizable = true
  if (wf.check_wf_semantics1(thd, select, &reqs)) return true;
  ...
  m_row_optimizable   &= reqs.row_optimizable;    // ★ 一票否决
  m_range_optimizable &= reqs.range_optimizable;
}
```

两级条件：

1. **帧级**：帧单位必须是 ROWS（行号偏移）才谈得上"进出各一行"；RANGE 帧走另一套标志（`range_optimizable`）。此外**静态聚合**（`UNBOUNDED PRECEDING ~ UNBOUNDED FOLLOWING`，整个 partition 就是一个 frame）**不需要** inversion——frame 根本不动，无从增量。
2. **函数级**：每个窗口函数经 `check_wf_semantics1()` 填自己的 `Window_evaluation_requirements`（两个标志默认 `true`），最后逐个 `&=`。源码注释点名了谁能谁不能：

   > Set to true if we can compute a sliding window by a combination of undoing the contribution of rows going out of the frame and adding the contribution of rows coming into the frame. **For example, SUM and AVG allows this, but MAX/MIN do not.** Only applicable if the frame has ROW bounds unit.

#### 失效的代价：退化成 O(n × frame_size)

关闭后退回朴素实现——**每个 frame 重新扫一遍**：

```
ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW   -- frame_size ≈ 1000
  inversion 开：每行 O(1)（加一行、减一行）→ 整窗 O(n)
  inversion 关：每行重扫 1000 行           → 整窗 O(1000n)
```

★ **连坐语义最容易被忽略**：`m_row_optimizable` 是**窗口级**标志，不是函数级。`SUM(x) OVER (...)` 单独跑是 O(n)；只要同一窗口里再加一个 `MAX(y) OVER (...)`，`&=` 会让**两个函数一起退化**——不是"MAX 慢一点"，而是 SUM 也被拖回朴素实现。

#### 怎么判断有没有走到 inversion（这是个观测盲区）

| 手段 | 能看到什么 |
|---|---|
| 源码标志 | `Window::optimizable_row_aggregates()` / `optimizable_range_aggregates()`（注释明说只在 `m_needs_buffering` 为真时有意义） |
| EXPLAIN | **看不出来**——窗口函数不单独成节点，是否启用 inversion 完全不暴露 |
| 实证法 | 加大 `N PRECEDING` 观察耗时曲线：随 N 增长不明显 = inversion 生效；**随 N 线性放大 = 已退化** |

★ 这是窗口函数调优里最"看不见"的一环：EXPLAIN 不暴露、状态变量也没有，只能靠"帧规模 × 函数组合"的经验判断——**同一窗口里混用 SUM 与 MAX/MIN 是最典型的退化写法**。

### 为什么必须先排序

窗口函数的语义建立在 `PARTITION BY + ORDER BY` 之上，执行前输入必须**按 partition + order 排好序**。这个排序（filesort）往往是窗口函数查询的主要代价——`Window::m_sorting_order` 就是二者合并后的物理排序键。

推论：**`RANGE` 帧比 `ROWS` 帧贵**，因为它需要 peer 判定（相同排序键的行属于同一 peer），这由本文第三章的三路比较器承担。

### 他库对比与演进

- PostgreSQL / Oracle 同样采用"排序 + 流式/缓冲"两分法，差异主要在 RANGE 帧的 peer 判定与 inversion 的覆盖范围
- MySQL 的窗口函数是 **8.0 才引入**的，此前只能用自连接或用户变量模拟。这解释了两点：①实现相对"教科书式"（两分法清晰）②优化空间（如并行窗口、更激进的 inversion）尚未展开

### 代价要点

1. **排序代价**（filesort，可能落盘）
2. **缓冲代价**：缓冲模式下整个 partition 需驻留内存或 `frame buffer` 表
3. **RANGE 帧额外代价**：peer 判定

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

`m_partition_items`/`m_order_by_items` 是 `Cached_item` 数组（`Mem_root_array<Cached_item *>`），`Cached_item::cmp()` 语义是"比较当前值并顺手缓存新值"——这是 partition/peer 判定的核心。

**成员的真实源码**（`sql/window.h:105`，稳定态区）：

```cpp
class Window {
 protected:
  Query_block *m_query_block;           ///< The SELECT the window is on
  PT_order_list *const m_partition_by;  ///< <window partition clause>
  PT_order_list *const m_order_by;      ///< <window order clause>
  ORDER *m_sorting_order;               ///< merged partition/order by
  PT_frame *const m_frame;              ///< <window frame clause>
  ...
  /**
    (At least) one window function needs to buffer frame rows for evaluation
    i.e. it cannot be evaluated on the fly just from previous rows seen
  */
  bool m_needs_frame_buffering;   // ★ 不是 m_needs_buffering（后者只在 window.h:1101/1107/1113 的过时注释里）

  /**
    (At least) one window function needs the peer set of the current row to
    evaluate the wf for the current row
  */
  bool m_needs_peerset;

  /**
    (At least) one window function (currently JSON_OBJECTAGG) needs the
    last peer for the current row to evaluate the wf for the current row.
    (This is used only during inversion/optimization)
  */
  bool m_needs_last_peer_in_frame;

  /**
    (At least) one window function needs the cardinality of the partition of
    the current row to evaluate the wf for the current row
  */
  bool m_needs_partition_cardinality;

  /**
    The functions are optimizable with ROW unit. For example SUM is, MAX is
    not always optimizable. Optimized means we can use the optimized evaluation
    path in process_buffered_windowing_record which uses inversion to avoid
    revisiting all frame rows for every row being evaluated.
  */
  bool m_row_optimizable;

  /**
    The functions are optimizable with RANGE unit. ...
  */
  bool m_range_optimizable;

  /**
    The aggregates (SUM, etc) can be evaluated once for a partition, since it
    is static, i.e. all rows will have the same value for the aggregates, e.g.
    ROWS/RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING.
  */
  bool m_static_aggregates;

  bool m_opt_first_row;   ///< Window requires re-evaluation of the first row
  bool m_opt_last_row;    ///< Window requires re-evaluation of the last row
  bool m_last;            ///< The last window to be evaluated at execution time

 public:
  struct st_offset {                    // NTH_VALUE 的偏移量
    int64 m_rowno;
    bool m_from_last;
    ...
  };
  struct st_ll_offset {                 // LEAD/LAG 的偏移量
    int64 m_rowno;  ///< negative values is LEAD
    st_ll_offset() : m_rowno(INT_MIN64 /* uninitialized marker*/) {}
    ...
  };
  struct st_nth { Mem_root_array_YY<st_offset> m_offsets; };
  struct st_lead_lag { Mem_root_array_YY<st_ll_offset> m_offsets; };

 protected:
  st_nth m_opt_nth_row;         // ★ 是结构体，不是 bool
  st_lead_lag m_opt_lead_lag;

  const Window *m_ancestor;     ///< resolved from existing window name
  List<Item_sum> m_functions;   ///< window functions based on 'this'
  Mem_root_array<Cached_item *> m_partition_items;
  Mem_root_array<Cached_item *> m_order_by_items;
```

**上限常量**（`sql/window.h:83`）：

```cpp
/**
  The number of windows is limited to avoid the stack blowing up
  when constructing iterators recursively. There used to be no limit, but
  it would be unsafe since QEP_shared::m_idx of tmp tables assigned for windows
  would exceed the old range of its type plan_idx (int8). It
  has now been widened, so the max number of windows could be increased
  if need be, modulo other problems. We could also make it a system variable.
*/
constexpr int kMaxWindows = 127;
```

```cpp
// sql/window.h:65 —— frame buffer 位置 hint 的"用途"枚举（决定 m_frame_buffer_positions 的槽位）
enum class Window_retrieve_cached_row_reason {
  WONT_UPDATE_HINT = -1,  // special value when using restore_special_row
  FIRST_IN_PARTITION = 0,
  CURRENT = 1,
  FIRST_IN_FRAME = 2,
  LAST_IN_FRAME = 3,
  LAST_IN_PEERSET = 4,
  MISC_POSITIONS = 5      // NTH_VALUE, LEAD/LAG have dynamic indexes 5..N
};
```

### 1.2 setup_windows1 / setup_windows2

- `setup_windows1`（`window.cc:1090`）：解析期装配——解析 PARTITION/ORDER、命名窗口继承 DAG + 环检测、建缓存比较器、`check_window_functions1` 收集需求、`sorting_order()` 合并排序键、`setup_range_expressions` 建 RANGE 比较器。
- `setup_windows2`（`:1308`）：执行期检查（PS 重执行时 border/offset 可能是 `?` 参数）。

**`setup_windows1()` 完整源码**（`sql/window.cc:1090-1273`）：

```cpp
bool Window::setup_windows1(THD *thd, Query_block *select,
                            Ref_item_array ref_item_array, Table_ref *tables,
                            mem_root_deque<Item *> *fields,
                            List<Window> *windows) {
  assert(thd->lex->current_query_block()->first_execution);

  if (windows->elements > kMaxWindows) {
    my_error(ER_TOO_MANY_WINDOWS, MYF(0), windows->elements, kMaxWindows);
    return true;
  }

  /*
    We can encounter aggregate functions in the ORDER BY and PARTITION clauses
    of window function, so make sure we allow it:
  */
  nesting_map save_allow_sum_func = thd->lex->allow_sum_func;
  thd->lex->allow_sum_func |= (nesting_map)1 << select->nest_level;

  for (Window &w : *windows) {
    w.m_query_block = select;
    if (w.m_partition_by != nullptr &&
        w.resolve_window_ordering(thd, ref_item_array, tables, fields,
                                  w.m_partition_by->value.first, true))
      return true;
    if (w.m_order_by != nullptr &&
        w.resolve_window_ordering(thd, ref_item_array, tables, fields,
                                  w.m_order_by->value.first, false))
      return true;
  }

  thd->lex->allow_sum_func = save_allow_sum_func;

  /* Our adjacency list uses std::unordered_set which may throw, so "try" */
  try {
    AdjacencyList adj(windows->size());   // ★ 命名窗口继承关系建成有向图

    /* Resolve inter-window references */
    {
      uint i = 0;
      for (auto wi1 = windows->begin(); wi1 != windows->end(); ++wi1, ++i) {
        Window *w1 = &*wi1;
        if (w1->m_inherit_from != nullptr) {
          bool resolved = false;
          uint j = 0;
          for (auto wi2 = windows->begin(); wi2 != windows->end(); ++wi2, ++j) {
            Window *w2 = &*wi2;
            if (w2->m_name == nullptr) continue;
            String str;
            if (my_strcasecmp(system_charset_info,
                              w1->m_inherit_from->val_str(&str)->ptr(),
                              w2->printable_name()) == 0) {
              w1->set_ancestor(w2);
              resolved = true;
              adj.add(i, j);
              break;
            }
          }
          if (!resolved) {
            String str;
            my_error(ER_WINDOW_NO_SUCH_WINDOW, MYF(0),
                     w1->m_inherit_from->val_str(&str)->ptr());
            return true;
          }
        }
      }
    }

    if (adj.check_circularity()) {       // ★ 环检测：w1 WINDOW w2 + w2 WINDOW w1
      my_error(ER_WINDOW_CIRCULARITY_IN_WINDOW_GRAPH, MYF(0));
      return true;
    }

    /* We now know all references are resolved and they form a DAG */
    for (uint i = 0; i < windows->size(); i++) {
      if (adj.out_degree(i) != 0) {
        /* Only the root can specify partition. SR 10.c) */
        const Window *const non_root = (*windows)[i];
        if (non_root->m_partition_by != nullptr) {
          my_error(ER_WINDOW_NO_CHILD_PARTITIONING, MYF(0));
          return true;
        }
      }

      if (adj.in_degree(i) == 0) {
        /* All windows that nobody depend on (leaves in DAG tree). */
        const Window *const leaf = (*windows)[i];
        const Window *seen_orderer = nullptr;
        /* SR 10.d) No redefines of ORDER BY along inheritance path */
        for (const Window *w3 = leaf; w3 != nullptr; w3 = w3->m_ancestor) {
          if (w3->m_order_by != nullptr) {
            if (seen_orderer != nullptr) {
              my_error(ER_WINDOW_NO_REDEFINE_ORDER_BY, MYF(0),
                       seen_orderer->printable_name(), w3->printable_name());
              return true;
            } else {
              seen_orderer = w3;
            }
          }
        }
      } else {
        /* This window has at least one dependent. SR 10.e */
        const Window *const ancestor = (*windows)[i];
        if (!ancestor->m_frame->m_originally_absent) {
          my_error(ER_WINDOW_NO_INHERIT_FRAME, MYF(0),
                   ancestor->printable_name());
          return true;
        }
      }
    }
  } catch (...) {
    handle_std_exception("setup_windows1");
    return true;
  }

  for (Window &w : *windows) {
    const PT_frame *f = w.frame();
    const PT_order_list *o = w.effective_order_by();

    if (w.m_order_by == nullptr && o != nullptr &&
        w.m_frame->m_originally_absent) {
      /*
        Since we had an empty frame specification, but inherit an ORDER BY (we
        cannot inherit a frame specification), we need to adjust the a priori
        border type now that we know what we inherit (not known before binding
        above).
      */
      assert(w.m_frame->m_query_expression == WFU_RANGE);
      w.m_frame->m_to->m_border_type = WBT_CURRENT_ROW;  // ★ 继承到 ORDER BY 后上界收紧为 CURRENT ROW
    }

    if (w.check_unique_name(*windows)) return true;

    if (w.setup_ordering_cached_items(thd, select, o, false))       return true; // m_order_by_items
    if (w.setup_ordering_cached_items(thd, select,
                                      w.effective_partition_by(), true))
      return true;                                                                // m_partition_items

    if (w.check_window_functions1(thd, select)) return true;   // ★ 需求收集

    /*
      initialize the physical sorting order by merging the partition clause
      and the ordering clause of the window specification.
    */
    (void)w.sorting_order(thd, select->is_implicitly_grouped());

    /* For now, we do not support EXCLUDE */
    if (f->m_exclusion != nullptr) {
      my_error(ER_NOT_SUPPORTED_YET, MYF(0), "EXCLUDE");
      return true;
    }

    /* For now, we do not support GROUPS */
    if (f->m_query_expression == WFU_GROUPS) {
      my_error(ER_NOT_SUPPORTED_YET, MYF(0), "GROUPS");
      return true;
    }

    if (f->m_query_expression == WFU_RANGE && w.setup_range_expressions(thd))
      return true;

    if (w.check_border_sanity1(thd)) return true;
  }

  return false;
}
```

> ★ **易错点**：`setup_windows1` **不会把需求"上推"给 JOIN**。它只往 `Window` 自身字段写；是优化器/执行器反过来**查询** `needs_sorting()` / `needs_buffering()`（查询点：`sql_optimizer.cc:591`、`sql_select.cc:4834/4852`、`opt_explain_json.cc:1210/1225`、`join_optimizer.cc:5734`）。
> ★ `GROUPS` 帧单元与 `EXCLUDE` 在 8.0.39 **枚举里有、运行时直接报 `ER_NOT_SUPPORTED_YET`**。

**建 `Cached_item` 的那一步**（`setup_ordering_cached_items`，`window.cc:644`）：

```cpp
bool Window::setup_ordering_cached_items(THD *thd, Query_block *select,
                                         const PT_order_list *o,
                                         bool partition_order) {
  if (o == nullptr) return false;

  for (ORDER *order = o->value.first; order; order = order->next) {
    if (partition_order) {
      Item_ref *ir =
          new Item_ref(&select->context, order->item, "<window partition by>");
      if (ir == nullptr) return true;
      Cached_item *ci = new_Cached_item(thd, ir);
      if (ci == nullptr) return true;
      m_partition_items.push_back(ci);
    } else {
      Item_ref *ir =
          new Item_ref(&select->context, order->item, "<window order by>");
      if (ir == nullptr) return true;
      Cached_item *ci = new_Cached_item(thd, ir);
      if (ci == nullptr) return true;
      m_order_by_items.push_back(ci);
    }
  }
  return false;
}
```

**`setup_windows2()` 完整源码**（`:1308`）+ 它真正干活的 `check_window_functions2()`（`:1275`）：

```cpp
bool Window::setup_windows2(THD *thd, List<Window> *windows) {
  for (Window &w : *windows) {
    /*
      In execution of PS we need to check again in case ? parameters are used
      for window borders, or for offsets in window functions..
    */
    if (w.check_border_sanity2(thd) || w.check_window_functions2(thd))
      return true;
  }
  return false;
}
```

```cpp
bool Window::check_window_functions2(THD *thd) {
  m_opt_nth_row.m_offsets.clear();
  m_opt_lead_lag.m_offsets.clear();
  m_opt_nth_row.m_offsets.init(thd->mem_root);
  m_opt_lead_lag.m_offsets.init(thd->mem_root);

  for (Item_sum &wf : m_functions) {
    Window_evaluation_requirements reqs;
    if (wf.check_wf_semantics2(&reqs)) return true;
    if (reqs.opt_nth_row.m_rowno > 0)
      m_opt_nth_row.m_offsets.push_back(reqs.opt_nth_row);
    /*
      INT_MIN64 can't be specified due 2's complement range.
      Offset is always given as a positive value; lead converted to negative
      but can't get to INT_MIN64. So, if we see this value, this window
      function isn't LEAD or LAG.
    */
    if (reqs.opt_ll_row.m_rowno != INT_MIN64)
      m_opt_lead_lag.m_offsets.push_back(reqs.opt_ll_row);
  }

  /*
    We do not allow FROM_LAST yet, so sorting guarantees sequential traversal
    of the frame buffer under evaluation of several NTH_VALUE functions invoked
    on a window, which is important for the optimized wf eval strategy
  */
  std::sort(m_opt_nth_row.m_offsets.begin(), m_opt_nth_row.m_offsets.end());
  std::sort(m_opt_lead_lag.m_offsets.begin(), m_opt_lead_lag.m_offsets.end());
  m_is_last_row_in_frame = !m_needs_frame_buffering;

  return false;
}
```

> ★ offsets **排序**的意义：`read_frame_buffer_row()` 的位置 hint 只能**向前跳**，所以 NTH_VALUE/LEAD-LAG 的多次取行必须按行号升序访问。

`sorting_order()`（`:393`）：把有效 PARTITION BY 克隆后追加 ORDER BY，得到"先分区再排序"的单一排序键。执行时输入流按它排序后，才能顺序判定 partition/peer。

```cpp
ORDER *Window::sorting_order(THD *thd, bool implicitly_grouped) {
  if (thd == nullptr) return m_sorting_order;   // ★ 无参调用 = 只取缓存值
  if (implicitly_grouped) {
    m_sorting_order = nullptr;                  // ★ 隐式分组（结果只有一行）无需排序
    return nullptr;
  }

  ORDER *part = effective_partition_by() ? effective_partition_by()->value.first
                                         : nullptr;
  ORDER *ord =
      effective_order_by() ? effective_order_by()->value.first : nullptr;

  /*
    1. Copy both lists
    2. Append the ORDER BY list to the partition list.

    This ensures that all columns are present in the resulting sort ordering
    and that all ORDER BY expressions are at the end.
    The resulting sort can the be used to detect partition change and also
    satisfy the window ordering.
  */
  if (ord == nullptr)
    m_sorting_order = part;
  else if (part == nullptr)
    m_sorting_order = ord;
  else {
    ORDER *sorting = clone(thd, part);
    ORDER *ob = clone(thd, ord);
    append_to_back(&sorting, ob);
    m_sorting_order = sorting;
  }
  return m_sorting_order;
}
```

**多窗口共享一次排序**（`reorder_and_eliminate_sorts`，`:752`；★ 它的调用者是 `eliminate_unused_objects()`（`window.cc:1084`），**不在** `setup_windows1` 里）：

```cpp
bool Window::equal_sort(Window *w1, Window *w2) {
  ORDER *o1 = w1->sorting_order();
  ORDER *o2 = w2->sorting_order();
  if (o1 == nullptr || o2 == nullptr) return false;
  while (o1 != nullptr && o2 != nullptr) {
    if (o1->direction != o2->direction || !(*o1->item)->eq(*o2->item, false))
      return false;
    o1 = o1->next;
    o2 = o2->next;
  }
  return o1 == nullptr && o2 == nullptr;  // equal so far, now also same length
}

void Window::reorder_and_eliminate_sorts(List<Window> *windows) {
  const size_t n = windows->size();
  std::vector<bool> redundant(n, false);
  for (uint i = 0; i < n - 1; i++) {
    for (uint j = i + 1; j < n; j++) {
      if (equal_sort((*windows)[i], (*windows)[j])) {
        if (j > i + 1) {
          // move up to right after window[i], so we can share sort
          windows->swap_elts(i + 1, j);      // ★ 物理搬到相邻位置
        }
        redundant[i + 1] = true;
        break;
      }
    }
  }

  for (uint i = 0; i < n; i++)
    if (redundant[i]) (*windows)[i]->m_sorting_order = nullptr;  // ★ 置空 ⇒ 该步不再插入排序
}
```

---

## 二、流式 vs 缓冲：两种迭代器

### 2.1 流式 WindowIterator（`sql/iterators/window_iterators.cc:1535`）

何时流式：`needs_buffering() == false`（`check_wf_semantics1` 里 `needs_buffer = !(ROWS && UNBOUNDED PRECEDING && CURRENT ROW)`）。

**完整源码**（构造函数里有 `assert(!m_window->needs_buffering())`——二者是一对一的）：

```cpp
// sql/iterators/window_iterators.cc:1510
WindowIterator::WindowIterator(THD *thd,
                               unique_ptr_destroy_only<RowIterator> source,
                               Temp_table_param *temp_table_param, JOIN *join,
                               int output_slice)
    : RowIterator(thd),
      m_source(std::move(source)),
      m_temp_table_param(temp_table_param),
      m_window(temp_table_param->m_window),
      m_join(join),
      m_output_slice(output_slice) {
  assert(!m_window->needs_buffering());
}

bool WindowIterator::Init() {
  if (m_source->Init()) {
    return true;
  }
  m_window->reset_round();

  // Store which slice we will be reading from.
  m_input_slice = m_join->get_ref_item_slice();

  return false;
}
```

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

**完整源码**（`sql/iterators/window_iterators.cc:1558`）：

```cpp
BufferingWindowIterator::BufferingWindowIterator(
    THD *thd, unique_ptr_destroy_only<RowIterator> source,
    Temp_table_param *temp_table_param, JOIN *join, int output_slice)
    : RowIterator(thd),
      m_source(std::move(source)),
      m_temp_table_param(temp_table_param),
      m_window(temp_table_param->m_window),
      m_join(join),
      m_output_slice(output_slice) {
  assert(m_window->needs_buffering());
}

bool BufferingWindowIterator::Init() {
  if (m_source->Init()) {
    return true;
  }
  m_window->reset_round();
  m_possibly_buffered_rows = false;
  m_last_input_row_started_new_partition = false;
  m_eof = false;

  // Store which slice we will be reading from.
  m_input_slice = m_join->get_ref_item_slice();
  assert(m_input_slice >= 0);

  return false;
}
```

```cpp
// sql/iterators/window_iterators.cc:1586
int BufferingWindowIterator::Read() {
  SwitchSlice(m_join, m_output_slice);

  if (m_eof) {
    return ReadBufferedRow(/*new_partition_or_eof=*/true);
  }

  // The previous call to Read() may have caused multiple rows to be ready
  // for output, but could only return one of them. See if there are more
  // to be output.
  if (m_possibly_buffered_rows) {
    int err = ReadBufferedRow(m_last_input_row_started_new_partition);
    if (err != -1) {
      return err;
    }
  }

  for (;;) {
    if (m_last_input_row_started_new_partition) {
      /*
        We didn't really buffer this row yet since, we found a partition
        change so we had to finalize the previous partition first.
        Bring back saved row for next partition.
      */
      if (bring_back_frame_row(
              thd(), m_window, Window::FBC_FIRST_IN_NEXT_PARTITION,
              Window_retrieve_cached_row_reason::WONT_UPDATE_HINT)) {
        return 1;
      }

      /*
        copy_funcs(CFT_HAS_NO_WF) is not necessary: a non-WF function was
        calculated and saved in OUT, then this OUT column was copied to
        special row, then restored to OUT column.
      */

      m_window->reset_partition_state();
      if (buffer_windowing_record(thd(), m_temp_table_param,
                                  nullptr /* first in new partition */)) {
        return 1;
      }

      m_last_input_row_started_new_partition = false;
    } else {
      // Read a new input row, if it exists. This needs to be done under
      // the input slice, so that any expressions in sub-iterators are
      // evaluated correctly.
      int err;
      {
        Switch_ref_item_slice slice_switch(m_join, m_input_slice);
        err = m_source->Read();
      }
      if (err == 1) {
        return 1;  // Error.
      }
      if (err == -1) {
        // EOF. Read any pending buffered rows, and then that's it.
        m_eof = true;
        return ReadBufferedRow(/*new_partition_or_eof=*/true);
      }

      /*
        This saves the values of non-WF functions for the row. For
        example, 1+t.a. But also 1+LEAD. Even though at this point we lack
        data to compute LEAD; the saved value is thus incorrect; later,
        when the row is fully computable, we will re-evaluate the
        CFT_NON_WF to get a correct value for 1+LEAD. We haven't copied fields
        yet, so use input file slice: referenced fields are present in input
        file record.
      */
      {
        Switch_ref_item_slice slice_switch(m_join, m_input_slice);
        if (copy_funcs(m_temp_table_param, thd(), CFT_HAS_NO_WF)) return 1;
      }

      bool new_partition = false;
      if (buffer_windowing_record(thd(), m_temp_table_param, &new_partition)) {
        return 1;
      }
      m_last_input_row_started_new_partition = new_partition;
    }

    int err = ReadBufferedRow(m_last_input_row_started_new_partition);
    if (err == 1) {
      return 1;
    }

    if (err == 0) {
      return 0;
    }

    // This input row didn't generate an output row right now, so we'll just
    // continue the loop.
  }
}
```

```cpp
// sql/iterators/window_iterators.cc:1682
int BufferingWindowIterator::ReadBufferedRow(bool new_partition_or_eof) {
  bool output_row_ready;
  if (process_buffered_windowing_record(
          thd(), m_temp_table_param, new_partition_or_eof, &output_row_ready)) {
    return 1;
  }
  if (thd()->killed) {
    thd()->send_kill_message();
    return 1;
  }
  if (output_row_ready) {
    // Return the buffered row, and there are possibly more.
    // These will be checked on the next call to Read().
    m_possibly_buffered_rows = true;
    return 0;
  } else {
    // No more buffered rows.
    m_possibly_buffered_rows = false;
    return -1;
  }
}
```

> ★ `m_possibly_buffered_rows` 的**唯一写入点**就是这里（1695/1699）；`Read()` 只在 1596 读它。返回 `-1` = "缓冲里没有可出行了"，`Read()` 据此继续向源读更多行。

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

**完整源码（按连续段落切块，`window_iterators.cc:654-1506`）**：

**块 1：三条路径互斥 + 帧边界代数计算**

```cpp
  const bool static_aggregate = w.static_aggregates();
  const bool row_optimizable = w.optimizable_row_aggregates();
  const bool range_optimizable = w.optimizable_range_aggregates();

  // These three strategies are mutually exclusive:
  assert((static_aggregate + row_optimizable + range_optimizable) <= 1);
  // 【注】可以三者全 0 —— 那就是 naive 全量扫描

  ...
  /**
    Row_number of the first row in the frame. Invariant: lower_limit >= 1
    after initialization.
  */
  int64 lower_limit = 1;
  /**
    Row_number of the logically last row to be computed in the frame, may be
    higher than the number of rows in the partition. ...
  */
  int64 upper_limit = 0;
```

```cpp
// window_iterators.cc:771 —— 计算 lower_limit / upper_limit
  if (f->m_query_expression == WFU_RANGE) {
    lower_limit = w.first_rowno_in_range_frame();
    /*
      For RANGE frame, we first buffer all the rows in the partition due to the
      need to find last peer before first can be processed. This can be
      optimized, FIXME.
    */
    upper_limit = INT64_MAX;                  // ★ RANGE 必须读完整个 partition 才能定上界
  } else {
    assert(f->m_query_expression == WFU_ROWS);
    bool lower_within_limits = true;
    // Determine lower border, handle wraparound for unsigned value:
    int64 border =
        f->m_from->border() != nullptr ? f->m_from->border()->val_int() : 0;
    if (border < 0) {
      border = INT64_MAX;
    }
    switch (f->m_from->m_border_type) {
      case WBT_CURRENT_ROW:
        lower_limit = current_row;
        break;
      case WBT_VALUE_PRECEDING:
        /*
          Example: 1 PRECEDING and current row== 2 => 1
                                   current row== 1 => 1
                                   current row== 3 => 2
        */
        lower_limit = std::max<int64>(current_row - border, 1);
        break;
      case WBT_VALUE_FOLLOWING:
        if (border <= (std::numeric_limits<int64>::max() - current_row))
          lower_limit = current_row + border;
        else {
          lower_within_limits = false;
          lower_limit = INT64_MAX;
        }
        break;
      case WBT_UNBOUNDED_PRECEDING:
        lower_limit = 1;
        break;
      case WBT_UNBOUNDED_FOLLOWING:
        assert(false);
        break;
    }

    // Determine upper border, handle wraparound for unsigned value:
    border = f->m_to->border() != nullptr ? f->m_to->border()->val_int() : 0;
    if (border < 0) {
      border = INT64_MAX;
    }
    {
      switch (f->m_to->m_border_type) {
        case WBT_CURRENT_ROW:
          upper_limit = current_row;   // we always have enough cache
          break;
        case WBT_VALUE_PRECEDING:
          upper_limit = current_row - border;
          break;
        case WBT_VALUE_FOLLOWING:
          if (border <= (std::numeric_limits<longlong>::max() - current_row))
            upper_limit = current_row + border;
          else {
            upper_limit = INT64_MAX;
            /*
              If both the border specifications are beyond numeric limits,
              the window frame is empty.
            */
            if (f->m_from->m_border_type == WBT_VALUE_FOLLOWING &&
                !lower_within_limits) {
              lower_limit = INT64_MAX;
              upper_limit = INT64_MAX - 1;   // ★★ "空帧"的编码：让循环一次也不进
            }
          }
          break;
        case WBT_UNBOUNDED_FOLLOWING:
          unbounded_following = true;
          upper_limit = INT64_MAX;  // need whole partition
          break;
        case WBT_UNBOUNDED_PRECEDING:
          assert(false);
          break;
      }
    }
  }
```

**块 2：节流阀——"帧凑齐了吗？"**

```cpp
// window_iterators.cc:891
  /*
    Determine if, given our current read and buffering state, we have enough
    buffered rows to compute an output row.

    Example: ROWS BETWEEN 1 PRECEDING and 3 FOLLOWING
    ...
    We can now process row 1 since both lower (1-1=0) and upper (1+3=4) are less
    than or equal to 4, the last row in the cache so far.
    We can not process row 2 since: !(4 >= 2 + 3) and we haven't seen the last
    row in partition which means that the frame may not be full yet.
    ...
  */
  if (!((lower_limit <= last_rowno_in_cache &&
         upper_limit <= last_rowno_in_cache &&
         !w.needs_partition_cardinality()) || /* we have cached enough rows */
        new_partition_or_eof /* we have cached all rows */))
    return false;  // We haven't read enough rows yet, so return
```

> ★ 这一行是缓冲模式的**核心节流阀**：帧没装满、或需要分区基数（NTILE / LEAD-LAG / CUME_DIST / PERCENT_RANK）而分区未读完 ⇒ 直接返回 false，让 `Read()` 再读一行。

**块 3：naive 全量扫描主循环**

```cpp
  if ((static_aggregate && current_row == 1) ||   // skip for row > 1
      (optimizable && !w.aggregates_primed()) ||  // skip for 2..N in frame
      (!static_aggregate && !optimizable))        // normal: no skip
  {
    int64 rowno;        ///< iterates over rows in a frame
    int64 skipped = 0;  ///< RANGE: # of visited rows seen before the frame

    for (rowno = lower_limit; rowno <= upper; rowno++) {
      if (optimizable) optimizable_primed = true;

      /*
        Set window frame state before computing framing window function.
        'n' is the number of row #rowno relative to the beginning of the
        frame, 1-based.
      */
      const int64 n = rowno - lower_limit + 1 - skipped;
      w.set_rowno_in_frame(n);

      const Window_retrieve_cached_row_reason reason =
          (n == 1 ? Window_retrieve_cached_row_reason::FIRST_IN_FRAME
                  : Window_retrieve_cached_row_reason::LAST_IN_FRAME);
      /*
        Hint maintenance: we will normally read past last row in frame, so
        prepare to resurrect that hint once we do.
      */
      w.save_pos(reason);

      /* Set up the non-wf fields for aggregating to the output row. */
      if (bring_back_frame_row(thd, &w, rowno, reason)) return true;

      if (range_frame) {
        if (w.before_frame()) {
          skipped++;
          continue;
        }
        if (w.after_frame()) {
          w.set_last_rowno_in_range_frame(rowno - 1);
          if (!first_row_in_range_frame_seen)
            // empty frame, optimize starting point for next row
            w.set_first_rowno_in_range_frame(rowno);
          w.restore_pos(reason);
          break;
        }  // else: row is within range, process

        if (!first_row_in_range_frame_seen) {
          /*
            Optimize starting point for next row: monotonic increase in frame
            bounds
          */
          first_row_in_range_frame_seen = true;
          w.set_first_rowno_in_range_frame(rowno);
        }
      }

      /*
        Compute framing WFs. For ROWS frame, "upper" is exactly the frame's
        last row; but for the case of RANGE
        we can't be sure that this is indeed the last row, but we must make a
        pessimistic assumption. ...
      */
      const bool setstate =
          (rowno == upper || range_frame || have_last_value);
      if (setstate)
        w.set_is_last_row_in_frame(true);  // temporary state for next call

      // Accumulate frame's row into WF's value for current_row:
      if (copy_funcs(param, thd, CFT_WF_FRAMING)) return true;

      if (setstate) w.set_is_last_row_in_frame(false);  // undo temporary state
    }
    ...
  }
```

**块 4：ROWS inversion（滑动窗口的四行核心）**

```cpp
  } else if (row_optimizable && w.aggregates_primed()) {
    /*
      Rows 2..N in partition: we still have state from previous current row's
      frame computation, now adjust by subtracting row 1 in frame (lower_limit)
      and adding new, if any, final frame row
    */
    const bool remove_previous_first_row =
        (lower_limit > 1 && lower_limit - 1 <= last_rowno_in_cache);
    const bool new_last_row =
        (upper_limit <= upper &&
         !unbounded_following /* all added when primed */);
    const int64 rows_in_frame = upper - lower_limit + 1;
    w.set_first_rowno_in_rows_frame(lower_limit);

    /* possibly subtract: early in partition there may not be any */
    if (remove_previous_first_row) {
      ...
      if (bring_back_frame_row(
              thd, &w, lower_limit - 1,
              Window_retrieve_cached_row_reason::FIRST_IN_FRAME))
        return true;                       // ① 取回"即将滑出帧"的那一行

      w.set_inverse(true);                 // ② 打开逆运算开关
      if (!new_last_row) {
        w.set_rowno_in_frame(current_row - lower_limit + 1);
        if (rows_in_frame > 0)
          w.set_is_last_row_in_frame(true); // do final comp., e.g. div in AVG

        if (copy_funcs(param, thd, CFT_WF_FRAMING)) return true;  // ③ 各 Item 做减法

        w.set_is_last_row_in_frame(false);
      } else {
        if (copy_funcs(param, thd, CFT_WF_FRAMING)) return true;
      }

      w.set_is_last_row_in_peerset_within_frame(false);
      w.set_inverse(false);                // ④ 关掉
    }

    if (have_first_value && (lower_limit <= last_rowno_in_cache)) {
      // We have seen first row of frame, FIRST_VALUE can be computed:
      if (bring_back_frame_row(
              thd, &w, lower_limit,
              Window_retrieve_cached_row_reason::FIRST_IN_FRAME))
        return true;
      w.set_rowno_in_frame(1);
      /*
        Framing WFs which accumulate (SUM, COUNT, AVG) shouldn't accumulate
        this row again as they have done so already. Evaluate only
        X_VALUE/MIN/MAX.
      */
      if (copy_funcs(param, thd, CFT_WF_USES_ONLY_ONE_ROW)) return true;
    }
    ...
    if (new_last_row)  // Add new last row to framing WF's value
    {
      if (bring_back_frame_row(
              thd, &w, upper, Window_retrieve_cached_row_reason::LAST_IN_FRAME))
        return true;

      w.set_rowno_in_frame(upper - lower_limit + 1)
          .set_is_last_row_in_frame(true);

      if (copy_funcs(param, thd, CFT_WF_FRAMING)) return true;   // ⑤ 加进新行

      w.set_is_last_row_in_frame(false);
    }
  }
```

> ★★ **inversion 的四行本质**：`bring_back_frame_row(lower_limit - 1)` → `set_inverse(true)` → `copy_funcs(CFT_WF_FRAMING)`（各 Item 在自己的 `val_*()` 里看到 `m_window->do_inverse()` 就做减法）→ `set_inverse(false)`。再加一行"新进帧"的 `copy_funcs`。总代价 **O(1)** 而非 O(frame_size)。

**块 5：RANGE inversion 最甜的短路**

```cpp
    /*
      As an optimization, if:
      - RANGE frame specification ends at CURRENT ROW and
      - current_row belongs to frame of previous row,
      then both rows are peers, so have the same frame: nothing changes.
    */
    if (range_to_current_row && current_row >= prev_first_rowno_in_frame &&
        current_row <= prev_last_rowno_in_frame) {
      // Peer set should already have been determined:
      assert(w.last_rowno_in_peerset() >= current_row);
      have_peers_current_row = true;
    } else {
      ...  /* 逆掉滑出的一整段 peer + 正向加入新进帧的行 */
    }
```

> ★ 这是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`（即"有 ORDER BY 时的默认帧"）独有的优化：**当前行是上一行的 peer ⇒ 帧完全相同 ⇒ 一行都不用重算**。

**块 6：收尾——peerset 探测、primed 置位、恢复当前行**

```cpp
  /* We need the peer of the current row to evaluate the row. */
  if (needs_peerset && !have_peers_current_row) {
    int64 first = current_row;
    if (current_row != 1) first = w.last_rowno_in_peerset() + 1;

    if (current_row >= first) {
      int64 rowno;
      for (rowno = current_row; rowno <= last_rowno_in_cache; rowno++) {
        if (bring_back_frame_row(
                thd, &w, rowno,
                Window_retrieve_cached_row_reason::LAST_IN_PEERSET))
          return true;

        if (rowno == current_row) {
          /* establish current row as base-line for peer set */
          w.reset_order_by_peer_set();
          w.set_last_rowno_in_peerset(current_row);
        } else if (w.in_new_order_by_peer_set()) {
          w.set_last_rowno_in_peerset(rowno - 1);
          break;  // we have accumulated all rows in the peer set
        }
      }
      if (rowno > last_rowno_in_cache)
        w.set_last_rowno_in_peerset(last_rowno_in_cache);
    }
  }

  if (optimizable && optimizable_primed) w.set_aggregates_primed(true);

  if (bring_back_frame_row(thd, &w, current_row,
                           Window_retrieve_cached_row_reason::CURRENT))
    return true;

  /* NTILE and other non-framing wfs */
  if (w.needs_partition_cardinality()) {
    if (process_wfs_needing_partition_cardinality(
            thd, param, have_nth_value, have_lead_lag, current_row, &w,
            Window_retrieve_cached_row_reason::CURRENT))
      return true;
  }

  if (w.is_last() && copy_funcs(param, thd, CFT_HAS_WF)) return true;
  *output_row_ready = true;
  w.set_last_row_output(current_row);
  return false;
}
```

> ★ `m_aggregates_primed` 是"第一帧已被完整算过一次"的开关——只有它为真，下一行才走 inversion。

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

### 3.2.1 `setup_range_expressions()` 完整源码（`window.cc:240-391`）

先看清**比较器长什么样**（`window.h:564` 处成员前的设计注释，这是理解 RANGE 的钥匙）：

```cpp
  /*
    For RANGE bounds, we need to be able to check if a given row is
    before the lower bound, or after the upper bound, as we don't know
    ahead of time how many rows to skip (unlike for ROW bounds).
    ...
    We solve this by having an array of comparators of the type

       value ⋛ <cache>(ref_value)

    where ⋛ is a three-way comparator, and <cache>(ref_value) is an
    Item_cache where we can store the value of the reference row
    before we invoke the comparison. For instance, if we have a
    row bound such as

        ... OVER (ORDER BY a,b) FROM t1

     we will have an array with the two comparators

        t1.a ⋛ <cache>(a)
        t1.b ⋛ <cache>(b)

     ... As this is a lexicographic comparison, we only need to evaluate the
     second comparator if the first one returns equality.

     The expressions on the right-hand side can be somewhat more
     complicated; say that we have a window specification like

        ... OVER (ORDER BY a RANGE BETWEEN 2 PRECEDING AND 3 FOLLOWING)

     In this case, the lower bound and upper bound are different
     (both arrays will always contain only a single element):

        Lower: t1.a ⋛ <cache>(a) - 2
        Upper: t1.a ⋛ <cache>(a) + 3

     ie., there is an Item_func_plus or Item_func_minus with some
     constant expression as the second argument.

     The comparators for the lower bound is stored in [0], and for
     the upper bound in [1].
   */
  Bounds_checked_array<Arg_comparator> m_comparators[2];
```

```cpp
bool Window::setup_range_expressions(THD *thd) {
  assert(m_frame->m_query_expression == WFU_RANGE);
  const PT_order_list *o = effective_order_by();

  if (o == nullptr) {
    /*
      Without ORDER BY, all rows are peers, so in a RANGE frame CURRENT ROW
      extends to infinity, which we rewrite accordingly.
      We do not touch other border types (e.g. N PRECEDING) as they must be
      checked in more detail later.
    */
    {
      // ★★ 硬约束①：无 ORDER BY ⇒ 全表互为 peer ⇒ CURRENT ROW 改写为 UNBOUNDED
      if (m_frame->m_from->m_border_type == WBT_CURRENT_ROW)
        m_frame->m_from->m_border_type = WBT_UNBOUNDED_PRECEDING;
      if (m_frame->m_to->m_border_type == WBT_CURRENT_ROW)
        m_frame->m_to->m_border_type = WBT_UNBOUNDED_FOLLOWING;
    }
  }

  for (PT_border *border : {m_frame->m_from, m_frame->m_to}) {
    enum_window_border_type border_type = border->m_border_type;
    switch (border_type) {
      case WBT_UNBOUNDED_PRECEDING:
      case WBT_UNBOUNDED_FOLLOWING:
        /* no computation required */
        break;
      case WBT_VALUE_PRECEDING:
      case WBT_VALUE_FOLLOWING: {
        /*
          Frame uses RANGE <value>, require ORDER BY with one column
          cf. SQL 2014 7.15 <window clause>, SR 13.a.ii
        */
        if (!o || o->value.size() != 1) {   // ★★ 硬约束②-a：ORDER BY 恰好 1 列
          my_error(ER_WINDOW_RANGE_FRAME_ORDER_TYPE, MYF(0), printable_name());
          return true;
        }

        /* check the ORDER BY type */
        Item *order_expr = *(o->value.first->item);
        switch (order_expr->result_type()) {
          case INT_RESULT:
          case REAL_RESULT:
          case DECIMAL_RESULT:
            goto ok;
          case STRING_RESULT:
            if (order_expr->is_temporal()) goto ok;  // ★ 字符串只允许 temporal
          default:;
        }
        my_error(ER_WINDOW_RANGE_FRAME_ORDER_TYPE, MYF(0), printable_name());
        return true;
      ok:;
      }
        [[fallthrough]];
      case WBT_CURRENT_ROW: {
        auto comparators = Bounds_checked_array<Arg_comparator>::Alloc(
            thd->mem_root, o->value.size());
        for (size_t i = 0; i < o->value.size(); ++i) {
          bool asc = elt(o->value, i)->direction == ORDER_ASC;
          Item *nr = m_order_by_items[i]->get_item();   // 左端：候选行的 ORDER BY 表达式

          /*
            Below, "value" is the value of ORDER BY expr at current row ...
            "nr" is the value of the ORDER BY expr at another row in partition
            which we want to determine whether resided in the specified RANGE.
            We poke in the actual value of expr of the current row (cached) into
            value in reset_order_by_peer_set().
          */
          Item_cache *value = make_result_item(nr);
          if (value == nullptr) return true;

          Item *cmp_arg;
          if (border_type == WBT_VALUE_PRECEDING ||
              border_type == WBT_VALUE_FOLLOWING) {
            assert(i == 0);  // only one expr allowed with WBT_VALUE_*
            cmp_arg = border->build_addop(
                value, border_type == WBT_VALUE_PRECEDING, asc, this);
            if (cmp_arg == nullptr) return true;
          } else {
            cmp_arg = value;
          }

          Item **left_args = new (thd->mem_root) Item *(nr);
          Item **right_args = new (thd->mem_root) Item *(cmp_arg);
          if (!cmp_arg->fixed) {
            if (cmp_arg->fix_fields(thd, right_args)) return true;
          }
          ...
          comparators[i] = Arg_comparator(left_args, right_args);
          ...
          // ENUM/SET 类型按数值比较（特殊处理）
          if (nr->real_item()->type() == Item::FIELD_ITEM) {
            Item_field *field = down_cast<Item_field *>(nr->real_item());
            if (field->field->real_type() == MYSQL_TYPE_ENUM ||
                field->field->real_type() == MYSQL_TYPE_SET) {
              if (comparators[i].set_cmp_func(nullptr, left_args, right_args,
                                              true, INT_RESULT))
                return true;
              compare_func_set = true;
            }
          }
          if (!compare_func_set) {
            if (comparators[i].set_cmp_func(nullptr, left_args, right_args,
                                            /*set_null_arg=*/true))
              return true;
          }
        }
        m_comparators[border == m_frame->m_to] = comparators;  // ★ from→[0]、to→[1]
        break;
      }
    }
  }
  return false;
}
```

### 3.2.2 `before_or_after_frame()` 完整源码（`window.cc:540-627`）

```cpp
bool Window::before_or_after_frame(bool before) {
  PT_border *border;
  enum_window_border_type infinity;  // the extreme bound of the border
  if (before) {
    border = frame()->m_from;
    infinity = WBT_UNBOUNDED_PRECEDING;
  } else {
    border = frame()->m_to;
    infinity = WBT_UNBOUNDED_FOLLOWING;
  }

  enum enum_window_border_type border_type = border->m_border_type;

  if (border_type == infinity) return false;  // all rows included

  assert(
      border_type == WBT_CURRENT_ROW ||
      (m_order_by_items.size() == 1 && (border_type == WBT_VALUE_PRECEDING ||
                                        border_type == WBT_VALUE_FOLLOWING)));

  uint i = 0;
  Bounds_checked_array<Arg_comparator> comparators = m_comparators[!before];
  // ★ 判"在下界之前"用 [0]；判"在上界之后"用 [1]

  ORDER *o_expr = effective_order_by()->value.first;

  for (auto it = m_order_by_items.begin(); it != m_order_by_items.end();
       ++it, o_expr = o_expr->next, ++i) {
    Cached_item *cur_row = *it;

    /*
      'cur_row' represents the value of the current row's windowing ORDER BY
      expression, and 'candidate' represents the same expression in the
      candidate row. ...
      First, as the candidate row has just been copied back from the frame
      buffer, we must update the item's null_value
    */
    Item *candidate = cur_row->get_item();
    if (candidate->update_null_value()) return true;

    const bool asc = o_expr->direction == ORDER_ASC;

    const bool nulls_at_infinity =  // true if NULLs stick to 'infinity'
        before ? asc : !asc;        // ★ ASC 时 NULL 排在最前 ⇒ 贴 PRECEDING 侧

    if (cur_row->null_value)  // Current row is NULL
    {
      /*
        Per the standard, if current row is NULL,
        <numeric value> PRECEDING/FOLLOWING is a bound which is positioned at
        "the NULLs" (=peers). So is CURRENT ROW. So, for example, in NULLS
        FIRST ordering, BETWEEN 2 FOLLOWING AND 3 FOLLOWING yields only the
        NULLs, while BETWEEN 2 FOLLOWING AND UNBOUNDED FOLLOWING yields the
        whole partition.
      */
      if (candidate->null_value)
        continue;  // peer, so can't be before or after
      else
        return !nulls_at_infinity;
    }

    if (candidate->null_value) return nulls_at_infinity;

    /*
      NOTE: The reference value has already been set in
      reset_order_by_peer_set().
    */
    int val = comparators[i].compare();   // ★ 三路比较（±N 已折进右端表达式）
    if (val != 0) {
      if (!asc) val = -val;               // ★ DESC 翻转，让"大"依旧代表"靠后"
      return before ? (val < 0) : (val > 0);
    }

    // Compared equal, so move to the next in the lexicographic comparison
  }
  return false;   // ★ 所有 ORDER BY 列都相等 ⇒ 候选在框内
}
```

```cpp
// sql/window.h:925 —— public 包装（真正实现是 private 的 before_or_after_frame）
  bool before_frame() { return before_or_after_frame(true); }
  bool after_frame() { return before_or_after_frame(false); }
```

> ★ **NULL 的四种组合**一次说清（设 ASC）：
> | 当前行 | 候选行 | 结论 |
> |---|---|---|
> | NULL | NULL | `continue` → 本列无法区分，看下一个 ORDER BY 列 |
> | NULL | 非 NULL | `return !nulls_at_infinity` = `!true` = false ⇒ **在框内**（ASC 时 NULL 在最前，非 NULL 都在其后） |
> | 非 NULL | NULL | `return nulls_at_infinity` = true ⇒ **在框外** |
> | 非 NULL | 非 NULL | 走三路比较 |

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

### 4.3 三个函数的完整源码

**`check_partition_boundary()`**（`window.cc:456`，★ 比上面的抽象多一句 `m_first_rowno_in_range_frame = 1`）：

```cpp
void Window::check_partition_boundary() {
  DBUG_TRACE;
  bool anything_changed = false;

  if (m_part_row_number == 0)  // first row in first partition
  {
    anything_changed = true;
  }

  /**
    If we have partitioning and any one of the partitioning columns have
    changed since last row, we have a new partition.
  */
  for (Cached_item *item : m_partition_items) {
    anything_changed |= item->cmp();
    // ★ cmp() 有副作用：把当前值写进 cache，下次比较就"和上一行比"
  }

  m_partition_border = anything_changed;

  if (m_partition_border) {
    m_part_row_number = 1;
    m_first_rowno_in_range_frame = 1;   // ★ RANGE 帧的单调下界也在此复位
  } else {
    m_part_row_number++;
  }
}
```

**`FindCacheInComparator()` + `reset_order_by_peer_set()`**（`window.cc:483` / `:507`）：

```cpp
/*
  For a comparator from m_comparators, locate the Item_cache to update
  with a new reference value (see Window::m_comparators).

  The comparator is one of

    candidate {<, >} current row
    candidate {<, >} current row {-,+} constant

  The second form is used when the the RANGE frame boundary is
  WBT_VALUE_PRECEDING/WBT_VALUE_FOLLOWING, "constant" above being the
  value specified in the query, cf. the setup in
  Window::setup_range_expressions.
*/
static Item_cache *FindCacheInComparator(const Arg_comparator &cmp) {
  Item *to_update;
  if (cmp.get_right()->type() == Item::CACHE_ITEM) {
    to_update = cmp.get_right();
  } else {
    to_update = down_cast<Item_func *>(cmp.get_right())->get_arg(0);
    // ★ 右端是 Item_func_plus/minus 时，取它的 arg0 —— 也就是那个 Item_cache
  }
  return down_cast<Item_cache *>(to_update);
}

void Window::reset_order_by_peer_set() {
  DBUG_TRACE;

  for (Cached_item *item : m_order_by_items) {
    /*
      A side-effect of doing this comparison, is to update the cache, so that
      when we compare the new value to itself later, it is in its peer set.
    */
    (void)item->cmp();      // ① 刷新 Cached_item（用于 partition/peer 边界判定）
  }

  // Update the reference value for ORDER BY elements as used by
  // before_or_after_frame(). These are the actual items used in the three-way
  // comparisons, whereas cached_item is used in in_new_order_by_peer_set().
  for (int i = 0; i < 2; ++i) {
    for (Arg_comparator &cmp : m_comparators[i]) {
      FindCacheInComparator(cmp)->cache_value();   // ② 冻结"当前行"的值为基准
    }
  }
}
```

**`in_new_order_by_peer_set()`**（`window.cc:528`；★ 默认参数是 `true`，即比较**全部** ORDER BY 表达式）：

```cpp
bool Window::in_new_order_by_peer_set(bool compare_all_order_by_items) {
  DBUG_TRACE;
  bool anything_changed = false;

  for (Cached_item *item : m_order_by_items) {
    anything_changed |= item->cmp();
    if (!compare_all_order_by_items) break;
    // ★ 传 false 时只看第一个 ORDER BY 表达式 —— 用于"提前退出"的快路径
  }

  return anything_changed;
}
```

```cpp
// sql/window.h:915
  bool in_new_order_by_peer_set(bool compare_all_order_by_items = true);
```

> ★ **两套"当前行基准"并存**，这是最容易混淆的地方：
> - `m_order_by_items` 的 `Cached_item` → 服务于 **peer 边界判定**（`in_new_order_by_peer_set`）
> - `m_comparators[i]` 里的 `Item_cache` → 服务于 **RANGE 帧边界判定**（`before_or_after_frame`）
>
> 两者都在 `reset_order_by_peer_set()` 里被刷新——所以它必须**每次换 current_row 时调一次**，这正是 `process_buffered_windowing_record` 在 `if (range_frame)` 分支里调它的原因。

`reset_partition_state()` 是 `window.h:1411` 的一行转发，真正实现 `reset_execution_state()`（`window.cc:1371`）：

```cpp
void Window::reset_execution_state(Reset_level level) {
  switch (level) {
    case RL_ROUND:
      if (m_frame_buffer != nullptr) (void)m_frame_buffer->empty_result_table();
      m_frame_buffer_total_rows = 0;
      m_frame_buffer_partition_offset = 0;
      m_part_row_number = 0;
      [[fallthrough]];
    case RL_PARTITION:
      /*
        Forget positions in the frame buffer: they won't be valid in a new
        partition.
      */
      if (!m_frame_buffer_positions.empty()) {
        for (Frame_buffer_position &it : m_frame_buffer_positions) {
          it.m_rowno = -1;   // ★ 使所有位置 hint 失效（read_frame_buffer_row 会跳过 -1）
        }
      }  // else not allocated, empty result set

      m_tmp_pos.m_rowno = -1;
      /*
        w.frame_buffer()->file->ha_reset();
        We could truncate the file here if it is not too expensive..? FIXME
      */
      break;
  }

  m_last_row_output = 0;
  m_last_rowno_in_cache = 0;
  m_aggregates_primed = false;
  m_first_rowno_in_range_frame = 1;
  m_last_rowno_in_range_frame = 0;
  m_first_rowno_in_rows_frame = 1;
  m_row_has_fields_in_out_table = 0;
}
```

> ⚠️ 注意：**frame buffer 临时表本身并不截断**（源码留了 FIXME），只是逻辑上重新计数 + hint 失效。所以"每个 partition 都清空 buffer"是**逻辑**清空，不是物理清空。

---

## 五、窗口函数 Item

| 类 | 位置（`sql/item_sum.cc`） | 算法要点 |
|----|------|---------|
| `Item_row_number` | `:4735`（`val_int`） | **没有 `add()`**；`m_ctr++` 一行；分区边界 `clear()` |
| `Item_rank`（RANK/DENSE_RANK 同一类，`m_dense` 区分） | `:4807` | 每个 ORDER BY 列一个 `Cached_item`，变化时 `m_rank_ctr += 1 + (dense ? 0 : m_duplicates)` |
| `Item_percent_rank` | `:4933`（独立类，非 `Item_rank`） | `needs_buffer` + `needs_peerset`（借 peerset 探测算 rank） |
| `Item_cume_dist` | `:4880` | `last_rowno_in_peerset() / last_rowno_in_cache()` |
| `Item_ntile` | `:5008` | `needs_partition_cardinality`，`full_rounds = N/buckets`、`modulus = N%buckets`，前 modulus 桶多一行 |
| `Item_lead_lag` | `:5668`（`compute()`） | LEAD 归一成负 LAG，`our_offset` 时从 frame buffer 取 `current_row - m_n` |
| `Item_sum_sum` / `Item_sum_count` / `Item_sum_avg` | `:1984` / `:2181` / `:2306` | ★ **没有 `do_inverse()` 方法**——在 `val_*()` 里判 `m_window->do_inverse()` 分支 |

> ⚠️ **勘误**：`Item_sum_sum::do_inverse()` 不存在。全仓库只有 `Window::do_inverse()`（`window.h:1267`）。而且 **`Item_sum_sum::add()` 第一行就是 `assert(!m_is_window_function)`**——窗口函数路径**根本不走 `add()`**，累加与逆运算全在 `val_real()`/`val_decimal()` 里发生（因为 `copy_funcs()` 是通过**求值 Item** 触发的）。这是最容易被写错的一点。

**统一入口 `Item_sum::wf_common_init()`**（`item_sum.cc:909`）——每个 WF 的 `val_*()` 第一行都调它：

```cpp
bool Item_sum::wf_common_init() {
  if (m_window->do_copy_null()) {
    assert(m_window->needs_buffering());
    null_value = is_nullable();
    return true;                      // ★ 空帧 ⇒ 写 NULL 并提前返回
  }
  if (m_window->at_partition_border() && !m_window->needs_buffering()) {
    clear();                          // ★ 流式路径：分区首行在这里自清
  }
  return false;
}
```

**`Item_sum::check_wf_semantics1()` 基类默认实现**（`item_sum.cc:404`）——"是否需要 frame buffer"的**唯一判据来源**：

```cpp
bool Item_sum::check_wf_semantics1(THD *, Query_block *,
                                   Window_evaluation_requirements *r) {
  const PT_frame *frame = m_window->frame();

  /*
    If we have ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW, we can just
    accumulate as we see rows, never need to invert old rows or to look at
    future rows, so don't need a frame buffer.
  */
  r->needs_buffer = !(frame->m_query_expression == WFU_ROWS &&
                      frame->m_from->m_border_type == WBT_UNBOUNDED_PRECEDING &&
                      frame->m_to->m_border_type == WBT_CURRENT_ROW);

  if (with_distinct) {
    my_error(ER_NOT_SUPPORTED_YET, MYF(0), "<window function>(DISTINCT ..)");
    return true;
  }
  return false;
}
```

> ★ 注意它**不看 ORDER BY**：`ROWS UNBOUNDED PRECEDING AND CURRENT ROW` 即使有 ORDER BY 也不需要缓冲。

**`Window_evaluation_requirements`**（`sql/window.h:1538`，★ 定义在 `window.h`，`item_sum.h:87` 只有前向声明）：

```cpp
struct Window_evaluation_requirements {
  bool needs_buffer;
  bool needs_peerset;
  bool needs_last_peer_in_frame;
  bool opt_first_row;
  bool opt_last_row;
  Window::st_offset opt_nth_row;    ///< Used if we have NTH_VALUE
  Window::st_ll_offset opt_ll_row;  ///< Used if we have LEAD or LAG
  /**
    Set to true if we can compute a sliding window by a combination of
    undoing the contribution of rows going out of the frame and adding the
    contribution of rows coming into the frame.  For example, SUM and AVG
    allows this, but MAX/MIN do not. Only applicable if the frame has ROW
    bounds unit.
  */
  bool row_optimizable;
  bool range_optimizable;

  Window_evaluation_requirements()
      : needs_buffer(false),
        needs_peerset(false),
        needs_last_peer_in_frame(false),
        opt_first_row(false),
        opt_last_row(false),
        row_optimizable(true),     // ★ 两个 optimizable 默认 true
        range_optimizable(true) {} //   ⇒ 由具体 wf 覆写成 false 来"一票否决"
};
```

**`Item_row_number`**（`:4735`）：

```cpp
longlong Item_row_number::val_int() {
  DBUG_TRACE;

  if (m_window->at_partition_border() && !m_window->needs_buffering()) {
    clear();      // ★ 只在"流式"路径自清；缓冲路径由 reset_non_framing_wf_state() 统一清
  }

  m_ctr++;

  DBUG_PRINT("enter", ("Item_row_number::val_int  at border: %d ctr: %llu",
                       m_window->at_partition_border(), m_ctr));
  return m_ctr;
}

void Item_row_number::clear() { m_ctr = 0; }
// val_real / val_str / val_decimal 全部转调 val_int()
```

**`Item_rank`**（`:4807`，RANK 与 DENSE_RANK 共用，`m_dense` 区分）：

```cpp
longlong Item_rank::val_int() {
  DBUG_TRACE;
  if (m_window->at_partition_border() && !m_window->needs_buffering()) {
    clear();
  }

  bool change = false;
  if (m_window->has_windowing_steps()) {
    /*
      Check if any of the ORDER BY expressions have changed. If so, we
      need to update the rank, considering any duplicates.
    */
    for (Cached_item *item : m_previous) {
      change |= item->cmp();
    }
  }
  // if no windowing steps, no comparison needed.

  if (change) {
    m_rank_ctr += 1 + (m_dense ? 0 : m_duplicates);   // ★★ 一行区分两种语义
    m_duplicates = 0;
  } else {
    m_duplicates++;
  }

  return m_rank_ctr;
}

void Item_rank::clear() {
  m_rank_ctr = 1;
  m_duplicates = -1;   // ★ 分区首行也会走 else { m_duplicates++; }，从 -1 变 0

  // Reset comparator
  if (m_window->has_windowing_steps()) {
    for (Cached_item *item : m_previous) {
      item->cmp();  // set baseline
    }
  }
}
```

`m_previous` 的建立（`check_wf_semantics1`，`:4765`）——**每个 ORDER BY 表达式一个 `Cached_item`**，与 `Window::m_order_by_items` 是**两套独立的缓存**（RANK 自己维护一份，因为在流式路径下也可能被求值）：

```cpp
bool Item_rank::check_wf_semantics1(THD *thd, Query_block *select,
                                    Window_evaluation_requirements *) {
  const PT_order_list *order = m_window->effective_order_by();
  // SQL2015 6.10 <window function> SR 6.a: require ORDER BY; we don't.
  if (!order) return false;  // all rows in partition are peers
  for (ORDER *o = order->value.first; o != nullptr; o = o->next) {
    /*
      We need to access the value of the ORDER expression when evaluating
      RANK to determine equality or not, so we need a handle.
    */
    Item_ref *ir = new Item_ref(&select->context, o->item, "<partition order>");
    if (ir == nullptr) return true;

    m_previous.push_back(new_Cached_item(thd, ir));
  }
  return false;
}
```

**`Item_cume_dist` / `Item_percent_rank`**（`:4870` / `:4909` 的 `check_wf_semantics1`，`:4880` / `:4933` 的 `val_real`）——peerset 需求的典型：

```cpp
bool Item_cume_dist::check_wf_semantics1(THD *, Query_block *,
                                         Window_evaluation_requirements *r) {
  // we need to know partition cardinality, so two passes
  r->needs_buffer = true;
  // Before we can compute for the current row we need the count of its peers
  r->needs_peerset = true;
  // SQL2015 6.10 <window function> SR 6.h: don't require ORDER BY.
  return false;
}

double Item_cume_dist::val_real() {
  if (!m_window->has_windowing_steps())
    return 1.0;  // degenerate case, no real windowing

  double cume_dist = (double)m_window->last_rowno_in_peerset() /
                     m_window->last_rowno_in_cache();
  return cume_dist;
}
```

```cpp
bool Item_percent_rank::check_wf_semantics1(THD *, Query_block *,
                                            Window_evaluation_requirements *r) {
  // we need to know partition cardinality, so two passes
  r->needs_buffer = true;
  /*
    The family of RANK functions doesn't need the peer set: ... However, this family
    needs detection of when the current row leaves the current peer set (to
    increase the rank counter):
    - RANK and DENSE_RANK do so internally with row comparison;
    - but PERCENT_RANK, as it needs partition cardinality, requires buffering,
    so it can simply pretend it needs_peerset() and then the buffering code will
    detect the peer set's end and provide it in last_rowno_in_peerset().
  */
  r->needs_peerset = true;
  ...
}

double Item_percent_rank::val_real() {
  if (!m_window->has_windowing_steps())
    return 0.0;  // degenerate case, no real windowing

  if (m_window->rowno_being_visited() == m_window->rowno_in_partition()) {
    if (m_last_peer_visited) {
      m_rank_ctr += m_peers;
      m_peers = 0;
      m_last_peer_visited = false;
    }
    m_peers++;
    if (m_window->rowno_being_visited() == m_window->last_rowno_in_peerset())
      m_last_peer_visited = true;
    if (m_rank_ctr == 1) return 0;
  }

  double percent_rank =
      (double)(m_rank_ctr - 1) / (m_window->last_rowno_in_cache() - 1);
  return percent_rank;
}
```

**`Item_ntile::val_int()`**（`:5008`）：

```cpp
longlong Item_ntile::val_int() {
  if (m_window->rowno_being_visited() == m_window->rowno_in_partition()) {
    // ★ 只在"访问当前行"时算；访问别的行时返回缓存的 m_value
    if (args[0]->is_null()) {
      null_value = true;
      return 0;
    }

    longlong buckets = args[0]->val_int();
    if (buckets == 0) {
      my_error(ER_WRONG_ARGUMENTS, MYF(0), func_name());
      return error_int();
    }

    /*
      Should not be evaluated until we have read all rows in partition
      notwithstanding any frames, so last_rowno_in_cache should be cardinality
      of partition.
    */
    int64 full_rounds = m_window->last_rowno_in_cache() / buckets;
    int64 modulus = m_window->last_rowno_in_cache() % buckets;
    int64 r;

    /*
      Rows might not distribute evenly, if modulus!=0. In that case, add
      extras at the beginning as per SQL 2011 section 6.10 <window function>
      GR 1a, ii, 3): the first 'modulus' buckets contain 'full_rounds + 1'
      rows, the other buckets contain 'full_rounds' rows.
     */
    if (modulus == 0 && full_rounds == 0) {
      r = 1;  // degenerate case; no real windowing
    } else {
      // Using convention "row 0 is first row" for those two variables:
      int64 rowno = m_window->rowno_in_partition() - 1,
            // the first rowno of smaller buckets
          first_of_small = modulus * (full_rounds + 1);
      if (rowno >= first_of_small)  // row goes into small buckets
      {
        r = (rowno - first_of_small) / full_rounds + 1 + modulus;
      } else  // row goes into big buckets
      {
        r = rowno / (full_rounds + 1) + 1;
      }
    }
    m_value = r;
  }

  return m_value;
}

bool Item_ntile::check_wf_semantics1(THD *, Query_block *,
                                     Window_evaluation_requirements *r) {
  r->needs_buffer =
      true;  // we need to know partition cardinality, so two passes
  return false;
}

bool Item_ntile::check_wf_semantics2(Window_evaluation_requirements *) {
  Item *arg = args[0];
  /*
    Semantic check of the argument. Should be a positive constant
    integer larger than zero ... NULL literal is not allowed. Dynamic
    parameter is allowed, and may not be NULL.
  */
  if (!arg->const_for_execution() || arg->result_type() != INT_RESULT ||
      arg->val_int() <= 0 || arg->is_null()) {
    my_error(ER_WRONG_ARGUMENTS, MYF(0), func_name());
    return true;
  }
  return false;
}
```

**`Item_lead_lag`**：LEAD 归一化 + 真正的求值核心（`compute()`，`:5668`）：

```cpp
bool Item_lead_lag::check_wf_semantics2(Window_evaluation_requirements *r) {
  /*
    Semantic check of the offset argument. Should be a integral constant,
    non-negative.
  */
  if (arg_count >= 2) {
    Item *arg = args[1];
    if (!arg->const_for_execution() || arg->result_type() != INT_RESULT ||
        ((m_n = arg->val_int()) < 0 || arg->is_null())) {
      my_error(ER_WRONG_ARGUMENTS, MYF(0), func_name());
      return true;
    }
  } else {
    m_n = 1;
  }

  /*
    Canonicalize LEAD to negative LAG so we can order all sequentially around
    current row: positive value are LAG, i.e. addresses a row earlier than
    the current row in the result set.
  */
  if (m_is_lead) {
    m_n = -m_n;      // ★★ LEAD(2) → m_n = -2；LAG(2) → m_n = 2
  }
  r->opt_ll_row.m_rowno = m_n;
  return false;
}

bool Item_lead_lag::compute() {
  if (m_window->do_inverse()) {
    // nothing, not relevant for LEAD/LAG
  } else {
    if (m_window->rowno_being_visited() == m_window->rowno_in_partition()) {
      /*
        Setup default value if present: it needs to be evaluated on the
        current row, not at the lead/lag row, cf. GR 1.b.i, SQL 2011
      */
      if (arg_count == 3) m_default->cache_value();  // ★ default 必须在当前行求值
      null_value = true;  // a priori for current row
    }

    if (!m_window->has_windowing_steps()) {
      // empty FROM: we have exactly one constant row
      ...
    }

    bool our_offset = (m_window->rowno_being_visited() ==
                       m_window->rowno_in_partition() - m_n);
    // ★ LAG(2): 目标 = 当前 - 2；LEAD(2): 目标 = 当前 - (-2) = 当前 + 2

    if (our_offset) {
      if ((m_window->rowno_being_visited()) < 1 ||
          (m_window->rowno_being_visited() > m_window->last_rowno_in_cache())) {
        /*
          The row is outside the partition set; use default value if any
          provided else use NULL
        */
        if (arg_count == 3) {
          null_value = m_default->null_value;
          m_use_default = true;
        }
      } else {
        m_value->cache_value();
        null_value = m_value->null_value;
      }
      m_has_value = true;
    } else {
      // Visiting another function; return NULL or result we have.
      if (!m_has_value) null_value = true;
    }
  }
  return null_value || current_thd->is_error();
}
```

> ★ `val_int/real/decimal/str/get_date` 全都是同一个三连：`if (wf_common_init()) ...; if (compute()) ...; return m_use_default ? m_default->val_X() : m_value->val_X();`
> ★ `process_wfs_needing_partition_cardinality()` 会为每个 LEAD/LAG 偏移做一次 `bring_back_frame_row(rowno_to_visit)` + `copy_funcs(CFT_WF_NEEDS_PARTITION_CARDINALITY)`；目标行越界时**不取行**，由 `our_offset` 分支里的越界判断给 default/NULL。

**inversion 的真实落点**（`Item_sum_sum::val_real()`，`:1984`）：

```cpp
double Item_sum_sum::val_real() {
  DBUG_TRACE;
  assert(fixed == 1);
  if (m_is_window_function) {
    if (wf_common_init()) return 0.0;

    if (hybrid_type == DECIMAL_RESULT) { ... } else {
      double d = args[0]->val_real();

      if (!args[0]->null_value) {
        if (m_window->do_inverse()) {
          assert(m_count > 0 && m_count > m_frame_null_count);
          sum -= d;                 // ★★ 逆运算：减法
          m_count--;
        } else {
          sum += d;
          m_count++;
        }
      } else {
        if (m_window->do_inverse()) {
          assert(m_count >= m_frame_null_count && m_frame_null_count > 0);
          m_count--;
          m_frame_null_count--;
        } else {
          m_count++;
          m_frame_null_count++;
        }
      }
      null_value = (m_count == m_frame_null_count);
    }
    return sum;
  } else {
    if (aggr) aggr->endup();
    ...
  }
}
```

```cpp
// Item_sum_count::val_int() —— item_sum.cc:2181
    if (m_window->do_inverse()) {
      if (count > 0) count--;
    } else {
      count++;
    }
```

```cpp
// Item_sum_avg::val_real() —— item_sum.cc:2283
    double sum = Item_sum_sum::val_real();

    if (m_window->is_last_row_in_frame()) {   // ★★ 除法只在"帧最后一行"做一次
      const int64 divisor = m_count - m_frame_null_count;
      if (divisor > 0) sum = sum / ulonglong2double(divisor);
    }
    m_avg = sum;  // save
    return sum;
```

> ★★ **`Item_sum_avg` 那个 `is_last_row_in_frame()` 判断，解释了 `process_buffered_windowing_record` 里为什么到处 `set_is_last_row_in_frame(true)`**：AVG 的除法只能做一次，而"哪一行是帧的最后一行"在 RANGE 帧下无法提前知道，所以代码采用 **pessimistic assumption**——每读一行都临时置 `is_last_row_in_frame(true)` 做一次除法并存进 `m_avg`/`m_avg_dec`；若后面发现还不是最后一行，下一个 peer 会重算覆盖它。这是"用重复计算换代码简洁"的典型取舍。

`Item_sum_sum::add()`（`:1939`）第一行的断言，再次确认窗口路径不走 `add()`：

```cpp
bool Item_sum_sum::add() {
  assert(!m_is_window_function);
  ...
}
```

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

**完整源码**（`sql/sql_select.cc:4260`；★ 是**自由函数**，不是 `JOIN::` 成员，声明在 `sql_select.h:1019`）：

```cpp
bool CreateFramebufferTable(
    THD *thd, const Temp_table_param &tmp_table_param,
    const Query_block &query_block, const mem_root_deque<Item *> &source_fields,
    const mem_root_deque<Item *> &window_output_fields,
    Func_ptr_array *mapping_from_source_to_window_output, Window *window) {
  /*
    Create the window frame buffer tmp table.  We create a
    temporary table with same contents as the output tmp table
    in the windowing pipeline (columns defined by
    curr_all_fields), but used for intermediate storage, saving
    the window's frame buffer now that we know the window needs
    buffering.
  */
  Temp_table_param *par =
      new (thd->mem_root) Temp_table_param(thd->mem_root, tmp_table_param);
  par->m_window_frame_buffer = true;      // ① 标记这是 frame buffer

  // Don't include temporary fields that originally came from
  // a window function (or an expression containing a window function).
  // Window functions are not relevant to store in the framebuffer,
  // and in fact, trying to restore them would often overwrite
  // good data we shouldn't.
  //
  // Not that the regular filtering in create_tmp_table() cannot do this
  // for us, as it only sees the Item_field, not where it came from.
  mem_root_deque<Item *> fb_fields(window_output_fields);
  for (size_t i = 0; i < fb_fields.size(); ++i) {
    Item *orig_item = source_fields[i];
    if (orig_item->has_wf()) {
      fb_fields[i] = nullptr;             // ② 剔除 WF 列
    }
  }
  fb_fields.erase(std::remove(fb_fields.begin(), fb_fields.end(), nullptr),
                  fb_fields.end());
  count_field_types(&query_block, par, fb_fields, false, false);

  TABLE *table =
      create_tmp_table(thd, par, fb_fields, nullptr, false, false,
                       query_block.active_options(), HA_POS_ERROR, "");
  if (table == nullptr) return true;      // ③

  window->set_frame_buffer_param(par);
  window->set_frame_buffer(table);

  // For window function expressions we are to evaluate after
  // framebuffering, we need to replace their arguments to point to the
  // output table instead of the input table (we could probably also have
  // used the framebuffer if we wanted). E.g., if our input is t1 and our
  // output is <temporary>, we need to rewrite 1 + SUM(t1.x) OVER w into
  // 1 + SUM(<temporary>.x) OVER w.
  for (Func_ptr &ptr : *mapping_from_source_to_window_output) {
    if (ptr.func()->has_wf()) {
      ReplaceMaterializedItems(thd, ptr.func(),
                               *mapping_from_source_to_window_output,
                               /*need_exact_match=*/false);   // ④
    }
  }
  return false;
}
```

> ★ 调用点：`sql_select.cc:4853`（旧优化器，`if (m_windows[wno]->needs_buffering())` 时）；hypergraph 在 `join_optimizer/finalize_plan.cc:557`。
> ★ 步骤④的语义很关键：frame buffering 阶段 WF 读的是 **OUT 表的记录**（`bring_back_frame_row` 反向拷回的目标就是 OUT），所以 WF 的参数必须从"输入表字段"改指向"输出表字段"。

### 7.2 缓冲读回的三层函数（`sql/iterators/window_iterators.cc`）

| 函数 | 位置 | 做什么 |
|------|------|--------|
| `buffer_windowing_record` | `:255-279` | 写入 buffer。新分区时 `save_special_row(FBC_FIRST_IN_NEXT_PARTITION)` 并**提前 return** |
| `read_frame_buffer_row` | `:285-352` | 读取。从 `m_frame_buffer_positions` 挑"**≤目标且最近**"的 hint（`:297-306`）→ `ha_rnd_pos` → 最多再 1 次 `ha_rnd_next`（`:317-349`，`assert cnt <= 1`） |
| `bring_back_frame_row` | `:404-480` | 恢复。特殊行走 `restore_special_row`；否则记录 `position()`（`:441-445`），最后 `copy_fields(fb_info, thd, true)` **反向拷回**（`:469`） |

> hint 机制的意义：frame 经常要往回读若干行（如 `ROWS BETWEEN 5 PRECEDING`），每次都从头扫太慢——用"最近的一个已知位置"作为起点，最多再前进一次即可。

**① `buffer_windowing_record()` 完整源码**（`:255-279`）：

```cpp
/**
  If we cannot evaluate all window functions for a window on the fly, buffer the
  current row for later processing by process_buffered_windowing_record.

  @param[in,out] new_partition If input is not nullptr:
                            sets the bool pointed to to true if a new partition
                            was found and there was a previous partition; if
                            so the buffering of the first row in new
                            partition isn't done and must be repeated
                            later: we save away the row as rowno
                            FBC_FIRST_IN_NEXT_PARTITION, then fetch it back
                            later, cf. end_write_wf.
                            If input is nullptr, this is the "later" call to
                            buffer the first row of the new partition:
                            buffer the row.
  @return true if error.
*/
bool buffer_windowing_record(THD *thd, Temp_table_param *param,
                             bool *new_partition) {
  DBUG_TRACE;
  Window *w = param->m_window;

  if (copy_fields(w->frame_buffer_param(), thd)) return true;
  // ★ OUT → FB 的正向拷贝，且**在判分区之前**做：因为要保存的正是已拷进 FB record[0] 的这行

  if (new_partition != nullptr) {
    const bool first_partition = w->partition_rowno() == 0;
    w->check_partition_boundary();

    if (!first_partition && w->partition_rowno() == 1) {
      *new_partition = true;
      w->save_special_row(Window::FBC_FIRST_IN_NEXT_PARTITION,
                          w->frame_buffer());
      return false;      // ★★ 提前 return：本行**不写入** frame buffer
    }
  }

  if (buffer_record_somewhere(thd, w, w->partition_rowno())) return true;

  w->set_last_rowno_in_cache(w->partition_rowno());   // ★ 缓冲水位的唯一推进点

  return false;
}
```

**② `read_frame_buffer_row()` 完整源码**（`:285-352`）——hint 选择的全部逻辑：

```cpp
/**
  Read row rowno from frame buffer tmp file using cached row positions to
  minimize positioning work.
*/
bool read_frame_buffer_row(int64 rowno, Window *w, bool for_nth_value) {
  int use_idx = 0;  // closest prior position found, a priori 0 (row 1)
  int diff = w->last_rowno_in_cache();  // maximum a priori
  TABLE *t = w->frame_buffer();

  // Find the saved position closest to where we want to go
  for (int i = w->m_frame_buffer_positions.size() - 1; i >= 0; i--) {
    Window::Frame_buffer_position cand = w->m_frame_buffer_positions[i];
    if (cand.m_rowno == -1 || cand.m_rowno > rowno) continue;
    // ★ 只接受"已初始化(m_rowno != -1) 且 ≤ 目标"的 hint

    if (rowno - cand.m_rowno < diff) {
      /* closest so far */
      diff = rowno - cand.m_rowno;
      use_idx = i;
    }
  }

  Window::Frame_buffer_position *cand = &w->m_frame_buffer_positions[use_idx];

  int error =
      t->file->ha_rnd_pos(w->frame_buffer()->record[0], cand->m_position);
  if (error) {
    t->file->print_error(error, MYF(0));
    return true;
  }

  if (rowno > cand->m_rowno) {
    /*
      The saved position didn't correspond exactly to where we want to go, but
      is located one or more rows further out on the file, so read next to move
      forward to desired row.
    */
    const int64 cnt = rowno - cand->m_rowno;

    /*
      We should have enough location hints to normally need only one extra read.
      If we have just switched to INNODB due to MEM overflow, a rescan is
      required, so skip assert if we have INNODB.
    */
    assert(w->frame_buffer()->s->db_type()->db_type == DB_TYPE_INNODB ||
           cnt <= 1 ||
           (w->last_row_output() == 0 &&
            w->frame()->m_from->m_border_type == WBT_VALUE_FOLLOWING) ||
           for_nth_value);    // ★★ 正常情况下 cnt <= 1

    for (int i = 0; i < cnt; i++) {
      error = t->file->ha_rnd_next(t->record[0]);
      if (error) {
        t->file->print_error(error, MYF(0));
        return true;
      }
    }
  }

  return false;
}
```

> ★ **hint 只能向前跳、不能后退**——这就是帧边界必须**单调递增**的原因（RANGE 帧的 `first_rowno_in_range_frame` 单调下界也正是为此而维护）。

**③ `bring_back_frame_row()` 完整源码**（`:404-480`）：

```cpp
bool bring_back_frame_row(THD *thd, Window *w, int64 rowno,
                          Window_retrieve_cached_row_reason reason,
                          int fno = 0) {
  DBUG_TRACE;
  assert(reason == Window_retrieve_cached_row_reason::MISC_POSITIONS ||
         fno == 0);
  w->set_rowno_being_visited(rowno);
  uchar *fb_rec = w->frame_buffer()->record[0];

  assert(rowno != 0);

  /*
    If requested row is the last we fetched from FB and copied to OUT, we
    don't need to fetch and copy again.
    Because "reason", "fno" may differ from the last call which fetched the
    row, we still do the updates of w.m_frame_buffer_positions even if
    do_fetch=false.
  */
  bool do_fetch;

  if (rowno == Window::FBC_FIRST_IN_NEXT_PARTITION) {
    do_fetch = true;
    w->restore_special_row(rowno, fb_rec);      // ★ 从内存缓存恢复，不碰文件
  } else {
    assert(reason != Window_retrieve_cached_row_reason::WONT_UPDATE_HINT);
    do_fetch = w->row_has_fields_in_out_table() != rowno;  // ★ OUT 里已是这行 ⇒ 跳过
    // 【注】这是帧扫描中大量重复访问的关键优化

    if (do_fetch &&
        read_frame_buffer_row(
            rowno, w,
            reason == Window_retrieve_cached_row_reason::MISC_POSITIONS))
      return true;

    /* Got row rowno in record[0], remember position */
    const TABLE *const t = w->frame_buffer();
    t->file->position(fb_rec);
    std::memcpy(
        w->m_frame_buffer_positions[static_cast<int>(reason) + fno].m_position,
        t->file->ref, t->file->ref_length);
    w->m_frame_buffer_positions[static_cast<int>(reason) + fno].m_rowno = rowno;
  }

  if (!do_fetch) return false;

  Temp_table_param *const fb_info = w->frame_buffer_param();
  ...
  /*
    Do the inverse of copy_fields to get the row's fields back to the input
    table from the frame buffer.
  */
  bool rc = copy_fields(fb_info, thd, true);   // ★★ reverse=true：FB → OUT
  ...
  if (!rc) {
    // fields are in OUT
    if (rowno >= 1) w->set_row_has_fields_in_out_table(rowno);
  }
  return rc;
}
```

**特殊行的存/取**（`Window::save_special_row` / `restore_special_row`，`:488` / `:506`）：

```cpp
void Window::save_special_row(uint64 special_rowno, TABLE *t) {
  size_t l = t->s->reclength;
  assert(m_special_rows_cache_max_length >= l);  // check room.
  // From negative enum, get proper array index:
  int idx = FBC_FIRST_KEY - special_rowno;      // ★ -1 → 0
  m_special_rows_cache_length[idx] = l;
  std::memcpy(m_special_rows_cache + idx * m_special_rows_cache_max_length,
              t->record[0], l);
}

void Window::restore_special_row(uint64 special_rowno, uchar *record) {
  int idx = FBC_FIRST_KEY - special_rowno;
  size_t l = m_special_rows_cache_length[idx];
  std::memcpy(record,
              m_special_rows_cache + idx * m_special_rows_cache_max_length, l);
  // Sometimes, "record" points to IN record
  set_row_has_fields_in_out_table(0);   // ★ 置 0 ⇒ 强制下一次真正去取
}
```

```cpp
// sql/window.h:364 —— 真实枚举名是 Window::Special_keys（不是 Frame_buffer_key）
  enum Special_keys {
    /**
      We read an incoming row. We notice it is the start of a new
      partition. We must thus process the just-finished partition, but that
      processing uses this row's buffer; so, we save this row first, process
     the partition, and restore it later.
    */
    FBC_FIRST_IN_NEXT_PARTITION = -1,
    // Insert new values here.
    // And keep the ones below up to date.
    FBC_FIRST_KEY = FBC_FIRST_IN_NEXT_PARTITION,
    FBC_LAST_KEY = FBC_FIRST_IN_NEXT_PARTITION,
  };
```

> ★ 源码自己留了 TODO：*"We have only one special key. Do we need the enum?"* —— 目前只有一个特殊 key。
> ★ **为什么行号取负值**：注释说 "so that they will never collide with actual row numbers in the frame"，这样 `bring_back_frame_row()` 可以把"普通行"和"特殊行"用同一个 `int64 rowno` 参数统一处理，不需要额外的标志位。

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

