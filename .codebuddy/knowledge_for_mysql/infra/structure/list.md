# MySQL 链表全盘点：一种结构，五种实现

> 基于 MySQL 8.0.39 源码，盘点 MySQL 的**全部自研链表实现**并糅合对照：InnoDB 的 `ut_list_base`（侵入式双向 + **成员指针偏移编译期固化**）、server 的 `SQL_I_List`（侵入式单向 + `T **next` 二级指针）、`base_list`/`List`（非侵入式单向 + `**last` 尾指针 + `end_of_list` 哨兵）、`I_List`/`ilink`（侵入式双向 + sentinel 伪装 + `unlink` 自解链）、C 的 `LIST`（非侵入式双向 + `void *data`）。五种实现回答同一个问题——"元素怎么串成链"——**寻址协议**是它们的第一分野。核心结论：**侵入性越强，运行时开销越低，但类型纪律越严**。
>
> **边界**：本篇讲链表容器本身（跨层数据结构，不属锁原语——分类见 [`../lock/README.md`](../lock/README.md)）。`ut_list_base` 的用户（trx_sys / buf_pool / lock）在各自模块详写：事务链表见 [`../../innodb/trx.md`](../../innodb/trx.md)，buffer pool 各链表见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)，锁链表见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)；`SQL_I_List`/`List` 的用法散见 [`../../server/query/`](../../server/query/)（item 链、执行计划链）。

## 目录

- [总览：一种结构，五种实现](#总览一种结构五种实现)
- [结构组织：节点怎么挂（寻址协议是第一分野）](#结构组织节点怎么挂寻址协议是第一分野)
- [为什么要用偏移量：ut_list_base 的核心论证](#为什么要用偏移量ut_list_base-的核心论证)
- [侵入 vs 非侵入、双向 vs 单向](#侵入-vs-非侵入双向-vs-单向)
- [用户全景](#用户全景)
- [核心调用栈](#核心调用栈)
- [他库对比](#他库对比)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [选择指南](#选择指南)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [坑与易混淆概念汇总](#坑与易混淆概念汇总)
- [参考](#参考)

---

## 总览：一种结构，五种实现

### 全量盘点：MySQL 8.0.39 的自研链表

| 实现 | 位置 | 侵入性 | 方向 | 寻址协议 | 用户 |
|---|---|---|---|---|---|
| `ut_list_base` | `storage/innobase/include/ut0lst.h` | 侵入式 | 双向 | **成员指针偏移**（编译期固化） | trx_sys 三链表、buf_pool 七链表、lock、ReadView 等 |
| `SQL_I_List<T>` | `sql/sql_list.h` | 侵入式 | 单向 | `T **next` 二级指针 | item 链、prepared statement 等 |
| `base_list` / `List<T>` | `sql/sql_list.h` | 非侵入式 | 单向 | `**last` 尾指针 + `end_of_list` 哨兵 | item_buff、执行计划链、大量 server 层容器 |
| `I_List<T>` / `ilink<T>` | `sql/sql_list.h` | 侵入式 | 双向 | `T **prev` 二级指针 + sentinel 伪装 | `i_string`/`i_string_pair`（mysqld 参数、复制 rewrite-db） |
| `LIST` | `include/my_list.h` | 非侵入式 | 双向 | 裸 `prev`/`next` + `void *data` | C 客户端库（`list_add`/`list_cons` 等） |

### 一张光谱

```
            侵入式 ──────────────────────────────────► 非侵入式
        ut_list_base   SQL_I_List   I_List/ilink   base_list/List   C LIST
          双向           单向           双向            单向          双向
     成员指针偏移     T**next      T**prev+sentinel   **last+哨兵   prev/next+data
      零运行时开销    间接指针        自解链          尾插 O(1)      C 兼容
```

### 版本演进

| 版本 | 变化 |
|---|---|
| 1995 | Heikki Tuuri 创建 `ut0lst.h`，最初是宏体系（`UT_LIST_*`），node 为 `ib_list_node_struct` |
| 2000 | `sql_list.h` 创建（base_list / List / SQL_I_List，server 层 item 链的历史底座） |
| 2011.12 | Sunny Bains 把 ut0lst.h 重写为 C++ 模板：`ut_list_node<Type>` + `ut_list_base<Type, NodeGetter>` |
| 8.0 前后 | ut_list_base 新增 `Removable` 迭代器（遍历时可删当前项）、`UT_LIST_BASE_NODE_T_EXTERN`（前向声明场景）、原子化 `count` |

> 老宏（`UT_LIST_ADD_FIRST` 等）仍保留在 `ut0lst.h` 作为兼容层，直接转发到模板函数。

### 阅读路径

本篇以 `ut_list_base` 的"成员指针偏移"为主线（它是五种实现里设计最精巧的），其余四种按"寻址协议差异"并排对照。只想读懂某一个实现：按目录找它的 `###` 小节。

---

## 结构组织：节点怎么挂（寻址协议是第一分野）

链表的物理操作（插、删、遍历）操作的是 node，但链表头只存元素指针。**"给定元素指针，怎么找到它的 node"**——这个寻址协议是五种实现的第一分野。

### ut_list_base：成员指针偏移（编译期固化）

元素内嵌 `ut_list_node<Type>`：

```cpp
template <typename Type>
struct ut_list_node {
  Type *prev;   /*!< pointer to the previous node, NULL if start of list */
  Type *next;   /*!< pointer to next node, NULL if end of list */
};
```

链表头只存三个运行时字段（`first_element`/`last_element`/`count`），**不存偏移量**——偏移量作为成员指针 `&t::m` 固化在类型里（详见「为什么要用偏移量」）：

```cpp
template <typename Type, ut_list_node<Type> Type::*node_ptr>
struct ut_list_base_explicit_getter {
  static const ut_list_node<Type> &get_node(const Type &element) {
    return element.*node_ptr;   // 编译期偏移 → mov rax, [elem + 0x1A8]
  }
};
#define UT_LIST_BASE_NODE_T(t, m) \
  ut_list_base<t, ut_list_base_explicit_getter<t, &t::m>>
```

配套：`UT_LIST_BASE_NODE_T_EXTERN(t, m)` + `UT_LIST_NODE_GETTER_DEFINITION(t, m)` 用于前向声明场景（典型例子：`lock0types.h` 用 `UT_LIST_BASE_NODE_T_EXTERN(lock_t, trx_locks)`，因为 `trx_locks` 成员定义在更晚的 `lock0priv.h`）。

#### 迭代器：正向极简 + Removable 的 O(1) 删除

正向迭代器极简（只持一个元素指针，`end()` 是 `nullptr`）：

```cpp
template <typename E>
class base_iterator {
 private:
  E *m_elem;
 public:
  base_iterator(E *elem) : m_elem(elem) {}
  E *operator*() const { return m_elem; }
  base_iterator &operator++() {
    m_elem = next(*m_elem);   // 经 NodeGetter::get_node 取 .next
    return *this;
  }
};
```

**反向迭代器不存在**（`rbegin`/`rend` 全仓 0 命中）——反向遍历只能手工从 `last_element` 沿 `prev` 走（`ut_list_validate` 内部就是这么干的）。

**`Removable` 迭代器是本实现最精巧的部分**——支持"遍历时删除当前项（甚至当前项之后若干项）仍保持 O(1)"：

```cpp
class iterator {
 private:
  ut_list_base &m_list;
  elem_type *m_elem;       // 当前项
  elem_type *m_prev_elem;  // 锚点：当前项的前驱
 public:
  iterator(ut_list_base &list, elem_type *elem)
      : m_list{list}, m_elem{elem},
        m_prev_elem{elem ? prev(*elem) : nullptr} {
    ut_ad(m_prev_elem == nullptr);   // 只测试过从头开始的情况
  }
  iterator &operator++() {
    ut_ad(!m_prev_elem || next(*m_prev_elem) || m_list.last_element == m_prev_elem);
    /* The reason this is so complicated is that we want to support cases in
    which the body of the loop removed not only the current element, but
    also some elements even further after it. */
    auto here = m_prev_elem == nullptr ? m_list.first_element : next(*m_prev_elem);
    if (here != m_elem) {
      m_elem = here;         // 当前项（及之后项）被删：从锚点重定位
    } else {
      m_prev_elem = m_elem;  // 当前项还在：正常前进
      m_elem = next(*m_elem);
    }
    return *this;
  }
};
```

**O(1) 的机制**：`operator++` **从不读 `m_elem` 自身的 `next`**（因为 `ut_list_remove` 摘链后会把节点的 prev/next 置空，见下），而是从"锚点"`m_prev_elem` 的 `next`（或表头 `first_element`）重新定位。若循环体删除了当前项及之后若干项，`here = next(*m_prev_elem)` 恰好落在删除后紧跟在锚点之后的元素上，**一步到位，无需回溯**。契约（`removable()` 注释）：不允许删除 prev 元素（否则锚点失效）；插入语义四条——插在当前项之后 → 会被处理；插在 prev 之前 → 不会被处理；插在当前项正前 → 禁止（可能死循环）；重插当前项（如 move to front）→ 安全且不会重复处理。

#### 操作函数：插入的指针序列与删除后的置空

`ut_list_insert`（elem1 之后插 elem2，注意指针修改顺序——先挂 elem2 两翼、再修邻居、最后改 elem1->next）：

```cpp
elem2_node.prev = elem1;
elem2_node.next = elem1_node.next;
ut_ad((elem2_node.next == nullptr) == (list.last_element == elem1));
if (elem2_node.next != nullptr) {
  List::get_node(*elem2_node.next).prev = elem2;
} else {
  list.last_element = elem2;
}
elem1_node.next = elem2;
list.update_length(1);
```

`ut_list_remove` 摘链后**明确置空** `node.next/node.prev`——这正是 `Removable` 迭代器不能依赖被删节点 next 的原因：

```cpp
auto &node = List::get_node(*elem);
if (node.next != nullptr) {
  List::get_node(*node.next).prev = node.prev;
} else {
  list.last_element = node.prev;
}
if (node.prev != nullptr) {
  List::get_node(*node.prev).next = node.next;
} else {
  list.first_element = node.next;
}
node.next = nullptr;
node.prev = nullptr;
list.update_length(-1);
```

`update_length` 是**"读-改-写"而非原子自增**：

```cpp
void update_length(int diff) {
  ut_ad(diff > 0 || static_cast<size_t>(-diff) <= get_length());
  count.store(get_length() + diff, std::memory_order_release);
}
```

与 `count` 字段注释自洽："It is atomic to allow unprotected reads. **Writes must be protected by some external latch.**"——release-store 发布写、acquire-load 无锁读；写路径不是原子 RMW，因此必须由外部 latch 串行化。

其余操作：`ut_list_reverse` 逐节点 swap prev/next（前进用 `List::prev(*elem)`，因为箭头已被翻转）最后交换 first/last；`ut_list_move_to_front` 先 `ut_ad(ut_list_exists)` 再 remove + prepend（remove 清空节点指针后 prepend 重挂）；`ut_list_validate`（debug）先正向遍历 `ut_a(count == get_length())` 再反向从 `last_element` 沿 prev 计数再次校验——**双向链接可走通 + 两个方向节点数与原子 count 一致**。

#### 老宏体系与初始化

`UT_LIST_*` 全是模板函数的**薄转发宏**（`UT_LIST_ADD_FIRST(LIST, ELEM) ut_list_prepend(LIST, ELEM)` 等）；`UT_LIST_GET_LEN` 读 `get_length()`（无锁 acquire-load）。`UT_LIST_INIT` 用 **placement new 重跑构造**（同时恢复 UNIV_DEBUG 下的 `0xCAFE` 初始化魔数——检测 malloc 出来未跑构造的裸对象）：

```cpp
#define UT_LIST_INIT(b)                                            \
  {                                                                \
    auto &list_ref = (b);                                          \
    new (&list_ref) std::remove_reference_t<decltype(list_ref)>(); \
  }
```

### SQL_I_List：T **next 二级指针

```cpp
template <typename T>
class SQL_I_List {
 public:
  uint elements;
  T *first;        /* The first element in the list. */
  T **next;        /* A reference to the next element in the list. */
```

`next` 是**二级指针**——指向"最后一个元素身上 next 字段的位置"：

```cpp
  inline void link_in_list(T *element, T **next_ptr) {
    elements++;
    (*next) = element;   // 尾节点的 next 字段 ← 新元素
    next = next_ptr;     // 二级指针推进到新尾的 next 字段位置
    *next = nullptr;
  }
```

尾插 O(1) 的关键：**不需要走链找尾**，`next` 永远指着"尾节点的 next 字段"，直接 `*next = element` 就挂上了。

**初始化与 O(1) 链拼接**（完整 API 只有 `link_in_list`/`clear`/`save_and_clear`/`push_front`/`push_back`——**没有删除、没有反转**，单向链表彻底放弃这两件事）：

```cpp
inline void clear() {
  elements = 0;
  first = nullptr;
  next = &first;    // 空表：next 指向自己的 first 字段（拷贝构造特判的呼应）
}

inline void save_and_clear(SQL_I_List<T> *save) {
  *save = *this;    // 浅拷贝（所有权转移语义，见下）
  clear();
}

inline void push_front(SQL_I_List<T> *save) {
  /* link current list last */
  *save->next = first;   // 当前链头挂到 save 链尾——O(1) 链拼接
  first = save->first;
  elements += save->elements;
}

inline void push_back(SQL_I_List<T> *save) {
  if (save->first) {
    *next = save->first;  // 接到当前链尾
    next = save->next;    // 二级指针搬移到 save 的尾槽
    elements += save->elements;
  }
}
```

`save_and_clear` + `push_front`/`push_back` 是**prepared statement 重用机制的核心工具**：语句重执行时把 `order_list`/`group_list` 整体摘下来（`save_and_clear`），解析完新语句后再 `push_front`/`push_back` 挂回去——全部是 O(1) 指针操作，ORDER 对象一个不搬（这就是 `order_list_ptrs`/`group_list_ptrs` 备份 next 指针存在的意义）。

**拷贝构造是"所有权转移"语义**（配合 `save_and_clear` 的 `*save = *this; clear();` 用法）：

```cpp
SQL_I_List(const SQL_I_List &tmp)
    : elements(tmp.elements),
      first(tmp.first),
      next(elements ? tmp.next : &first) {}   // 空表特判：next 指向自己的 first
```

浅拷贝共享节点链与 `next` 二级指针；**空表特判 `next = &first` 避免拷贝出悬垂二级指针**（若空表还拷贝别人的 next，就会指向别人表内的槽位）。

**用户是全仓最"贴身"的**（元素自带 `T *next`，链直接长在对象里）：`Query_block::m_table_list`（FROM 子句表链，`SQL_I_List<Table_ref>`，注释"use Table_ref::next_local to traverse"）、`order_list`/`group_list`（ORDER BY/GROUP BY，prepared statement 需备份 next 指针——`order_list_ptrs`/`group_list_ptrs`）、`sp_instr::m_trig_field_list`（触发器 OLD/NEW 字段链，还有 **嵌套列表** `SQL_I_List<SQL_I_List<Item_trigger_field>>`——每个 sp 指令一条子链）、`PT_delete::delete_tables`（DELETE 目标表）、`HA_CREATE_INFO::merge_list`（MERGE 表子表）、`Query_block::sroutines_list`（语句涉及的存储例程）。

### base_list / List：**last 尾指针 + end_of_list 哨兵（非侵入式）

非侵入式：节点 `list_node` 由链表自己分配（`new (*THR_MALLOC)` 或 MEM_ROOT），元素只是 `void *info`：

```cpp
struct list_node {
  list_node *next;
  void *info;
  list_node(void *info_par, list_node *next_par) : next(next_par), info(info_par) {}
  list_node() /* For end_of_list */ {
    info = nullptr;
    next = this;   // 哨兵：next 指向自己
  }
};
extern MYSQL_PLUGIN_IMPORT list_node end_of_list;

class base_list {
 protected:
  list_node *first, **last;   // last 也是二级指针：指向"尾节点的 next 字段"
```

两个设计点（文件头注释原文）：

> All list ends with a pointer to the 'end_of_list' element, which data pointer is a null pointer and the next pointer points to itself. This makes it very fast to traverse lists as we don't have to test for a special end condition for list that can't contain a null pointer.

- **`end_of_list` 哨兵**：全局唯一节点，`next` 指向自己——遍历终止条件就是 `node == &end_of_list`，不用判 NULL；
- **`**last` 二级指针**：与 SQL_I_List 的 `next` 同构，尾插 O(1)：`*last = new list_node(info, &end_of_list); last = &(*last)->next;`。

`List<T>` 是 base_list 的类型化包装，额外提供：`operator[]`（下标访问，线性走链）、`replace`/`swap_elts`（换 info 不换节点）、`sort`（**交换排序**——换 `info` 不换节点，注释明说"list iterators that are initialized before sort could be safely used after sort"；代价是 O(n²)，"the list to be sorted is supposed to be short"）、`delete_elements`/`destroy_elements`（清空 + 析构元素）、C++11 range-for（`List_STL_Iterator`，ForwardIterator 完整五成员）。

**迭代器的 O(1) 删除机制**（`base_list_iterator` 的双二级指针设计）：

```cpp
inline void init(base_list &list_par) {
  list = &list_par;
  el = &list_par.first;   // el：下一个待访问节点的"槽位"（&first 或 &某节点->next）
  prev = nullptr;         // prev：存放 current 的那个槽位
  current = nullptr;
}
inline void *next(void) {
  prev = el;              // prev ← 上一个 el（即"指向 current 的指针的地址"）
  current = *el;
  el = &current->next;
  return current->info;
}
inline void remove(void) {
  list->remove(prev);     // *prev 就是指向被删节点的链接指针，直接改它 = O(1) 摘链
  el = prev;              // 回退到被删节点原本的槽位，下次 next() 取到后继继续
  current = nullptr;      // Safeguard
}
```

`base_list::remove(list_node **prev)` 里 `*prev = node` 一步完成摘链（同时修正 `last`），**不需要从头遍历找前驱**——这就是"单向链表 O(1) 删除"的秘密：迭代器持有 prev 槽位。`next_fast()` 只做 `tmp = *el; el = &tmp->next;`，**不维护 prev/current**，所以 `List_iterator_fast` 把 `remove/replace/after/ref` 全部禁用，换取更快的遍历。

**深拷贝构造一次分配全部节点**（sql_list.cc）：`mem_root->Alloc(sizeof(list_node) * rhs.elements)` 连续分配，注释："It's okay to allocate an array of nodes at once: we never call a destructor for list_node objects anyway."——节点是 POD，无析构，整块分配 + 逐个填 next 即可。

**典型用户**：`Item_cond::list`（`List<Item>`，AND/OR 条件子节点表，优化器大量 `args->concat/disjoin` 重组）、`List<TABLE> sj_tmp_tables`（半连接物化临时表）、`List<TABLE> m_unupdated_check_opt_tables`（UPDATE 不需更新的 check 表）、`Query_block::purge_value_list`/`kill_value_list`（PURGE/KILL 参数）、`ftfunc_list_alloc`（全文检索函数收集）。澄清：头注释的 "Used for item and item_buffs" 是历史遗留——`item_buff` 类型 8.0.39 已不存在。

### I_List / ilink：T **prev 双向 + sentinel 伪装（侵入式双向）

```cpp
template <typename T>
class ilink {
  T **prev, *next;
 public:
  void unlink() {
    /* Extra tests because element doesn't have to be linked */
    if (prev) *prev = next;
    if (next) next->prev = prev;
    prev = nullptr;
    next = nullptr;
  }
};
```

三个关键设计：

1. **`prev` 是 `T **` 二级指针**（不是 `T *`）：指向"前一个节点的 next 字段"——摘链时 `*prev = next` 直接改前驱的 next，不需要知道前驱节点是谁；
2. **`unlink()` 自解链**：元素自己知道怎么把自己摘下来，且**不需要链表头**（`prev` 已指向前驱的 next 字段位置）——这是 O(1) 删除且"无头可用"场景的关键；
3. **sentinel 伪装成 T**：链表尾哨兵是 `ilink<T> sentinel` 成员，被 `static_cast<T *>` 强转后当元素用（`first = static_cast<T *>(&sentinel)`，`SUPPRESS_UBSAN`）——头注释明说 "this inherently unsafe, since we rely on <T> to have the same layout as ilink<T>"，靠 `static_assert(!std::is_polymorphic<T>::value)` 挡掉多态类。

**哨兵初始化与 O(1) 双向插入**（`clear`/`push_front`/`push_back` 的指针序列）：

```cpp
void clear() SUPPRESS_UBSAN {
  first = static_cast<T *>(&sentinel);  // 空表：first 指向哨兵自己
  sentinel.prev = &first;               // 哨兵的 prev 指向 first 槽位
}

void push_front(T *a) {
  first->prev = &a->next;   // 原头（或哨兵）的 prev 指向新节点的 next 槽
  a->next = first;
  a->prev = &first;         // 新头的 prev 指向表头槽位
  first = a;
}

void push_back(T *a) {      // "in front of the sentinel"——插在哨兵之前
  *sentinel.prev = a;       // 哨兵 prev 指向的槽位（原尾的 next 或 first）挂新元素
  a->next = static_cast<T *>(&sentinel);
  a->prev = sentinel.prev;
  sentinel.prev = &a->next; // 哨兵 prev 推进到新尾的 next 槽
}
```

哨兵的 `prev` 是二级指针，作用与 SQL_I_List 的 `next`、base_list 的 `last` 同构：**永远指向"尾节点的 next 字段"**——所以 `push_back` O(1) 且无需判空（空表时指向 `first` 槽位，代码零分支）。

**配套操作**：`get()`（弹头：`first->unlink(); return first`——头即元素，unlink 靠 prev 自解链）、`move_elements_to(new_owner)`（所有权转移，`assert(new_owner->is_empty())`，整链指针搬移 + 原主 clear）、`unlink()` 注释原文 "Extra tests because element doesn't have to be linked"——**元素可以不挂链**，unlink 是幂等的。迭代器 `base_ilist_iterator::next()` 注释 "coded to allow push_back() while iterating"——el 推进到 `&current->next` 槽位，遍历中尾插不影响正在进行的迭代。

`I_List<T>` 是 base_ilist 的 public 包装（私有继承 + using 导出；拷贝构造/赋值声明不定义——"two list heads containing the same elements"）。**典型用户**：`i_string`（`const char *ptr`）——`--plugin-load`/`--early-plugin-load` 选项链（`opt_plugin_load_list`）、协议层 `store(Protocol*, I_List<i_string>*)` 把字符串链序列化给客户端；`i_string_pair`（key/val）——`Rpl_filter::rewrite_db`（replicate-rewrite-db 规则）；`Item_change_list`（`I_List<Item_change_record>`，查询改写回滚记录）；`I_List<COND_CMP>`（条件传播保存等价比较）；`NAMED_ILIST`（`I_List<NAMED_ILINK>`，按名管理 keycache）。

### C LIST：prev/next 裸指针（非侵入式，C 兼容）

```c
typedef struct LIST {
  struct LIST *prev, *next;
  void *data;
} LIST;
```

**无表头对象、无计数**：root 就是一个节点指针，空链即 NULL。实现全在 `mysys/list.cc`（8.0 里是 .cc 不是 .c），逐函数看：

**`list_add`**（头插，且支持"任意节点之前"插入——传入非根节点即在链中间插）：

```cpp
LIST *list_add(LIST *root, LIST *element) {
  if (root) {
    if (root->prev) /* If add in mid of list */
      root->prev->next = element;
    element->prev = root->prev;
    root->prev = element;
  } else
    element->prev = nullptr;
  element->next = root;
  return element; /* New root */
}
```

**返回新根**——这是 C 无表头链表的惯例（root 会变，调用方必须接返回值）。

**`list_cons`**（Lisp 风格头插——分配节点 + list_add；变量名 `new_charset` 是历史遗迹）：

```cpp
LIST *list_cons(void *data, LIST *list) {
  LIST *new_charset = (LIST *)my_malloc(key_memory_LIST, sizeof(LIST),
                                        MYF(MY_FAE | MY_ZEROFILL));
  if (!new_charset) return nullptr;
  new_charset->data = data;
  return list_add(list, new_charset);
}
```

（8.0.39 中 `list_cons` **无任何调用点**，仅声明与定义——存留的历史 API。）

**`list_delete`**（标准双向摘链；**不清 element 的指针、不释放内存**；删根则返回新根）：

```cpp
LIST *list_delete(LIST *root, LIST *element) {
  if (element->prev)
    element->prev->next = element->next;
  else
    root = element->next;
  if (element->next) element->next->prev = element->prev;
  return root;
}
```

**`list_reverse`**（逐节点交换 prev/next，返回新根即原尾）、**`list_free`**（可选 `free_data` 释放 data 再释放全部节点）、**`list_length`**（每次 O(n) 现数，无缓存）、**`list_walk`**（对每节点调 `action(data, argument)`，返回非零即中止）。

**实际调用点**（grep `list_add`）：server 侧 myisam `myisam_open_list`（已打开表注册表）、heap `heap_open_list`、myisammrg `myrg_open_list`、mysys `thr_lock_thread_list`、sql `dynamic_variables_allocs`（动态插件变量内存追踪）；客户端库 libmysql `mysql->stmts = list_add(mysql->stmts, &stmt->list)`（**MYSQL_STMT 挂到连接上**）、sql-common 的 mysql_options 扩展链。**它是连接库与存储引擎老代码的通用工具**，server 核心已不用。

---

## 为什么要用偏移量：ut_list_base 的核心论证

### 前提一：InnoDB 必须用侵入式链表

链表的"侵入式 vs 非侵入式"不是风格选择，而是被 InnoDB 对象的性质逼出来的：

- **对象不可拷贝/移动**：`trx_t`、`buf_page_t`、`lock_t` 这些对象被指针跨结构引用（事务引用锁、锁引用事务、页被多个队列引用），拷贝意味着要重挂所有外部指针，根本不现实。而 `std::list` 式非侵入链表恰恰要求元素可拷贝/移动进节点
- **零内存分配**：插入/删除只改指针，不 `malloc`。InnoDB 里这些操作发生在持锁/热路径上（buffer pool 每页状态迁移、事务提交），分配内存意味着额外的锁与失败处理
- **缓存局部性**：node 就嵌在元素里，`elem->next` 和 `elem->data` 在同一 cache line 内，一次缓存行加载同时拿到数据和链接

侵入式 = node 内嵌在元素里、链表头只持元素指针。**这正是偏移量问题的来源。**

### 前提二：侵入式必然面临"元素指针 → node"的转换

链表操作（插、删、遍历）操作的物理实体是 node（prev/next），但链表头只存了 `elem_type *`。所以每次操作都要回答同一个问题：

> 给定一个元素指针，它的 `ut_list_node` 成员在这个元素内的哪里？

这就是偏移量问题的本质——**它是侵入式链表的"寻址协议"**。

### 三种候选方案对比

| 方案 | 做法 | 代价 |
|------|------|------|
| A. 运行时存偏移 | 链表头存 `size_t node_offset`，访问时 `(char*)elem + node_offset` | 每实例 +8 字节；每次访问多一次内存 load + 加法；且偏移是运行期值，编译器无法内联 |
| B. 非侵入式 | 放弃侵入，node 独立分配 | 回到前提一的三个不可接受点 |
| C. 编译期偏移（InnoDB 选） | 偏移作为模板非类型参数 `&t::m` | 零空间、零时间（见下） |

### 为什么偏移量是编译期常量而非运行时字段

核心洞察：**元素类型一旦定义，node 成员的偏移就是固定的**。`&t::m` 是编译期可知的常量，没有理由让它变成运行时数据。把它固化成模板参数后：

- **零空间**：偏移量不占链表头任何字节，同一类型的链表共享一个编译期常量
- **零时间**：编译器把 `element.*node_ptr` 内联成一条带立即数偏移的访问指令 `mov rax, [elem + 0x1A8]`，连加法都没有——`0x1A8` 是编码进指令里的立即数。如果运行时存偏移，每次都要先读偏移再做一次加法，且读偏移本身是内存访问
- **类型安全**：`ut_list_node<Type> Type::*node_ptr` 的类型约束保证这个偏移只能指向 `ut_list_node<Type>` 类型的成员——把偏移写错（指向别的成员）会在编译期报错，而不是运行时读到脏数据

这正是"为什么用偏移量"的完整答案：**侵入式必须解决寻址，而寻址信息在编译期就确定，所以用编译期常量偏移；只有这样才能做到零运行时开销。**

### 实现思路：成员指针 + 模板，把"寻址"变成"类型的一部分"

普通成员指针 `Type::*` 的底层就是一个偏移量（非多态、非多重继承时）。InnoDB 没有在链表头里塞一个运行时偏移值，而是把偏移"装进类型"：

1. `UT_LIST_BASE_NODE_T(t, m)` 把用户写的 `t, m` 展开为 `ut_list_base<t, ut_list_base_explicit_getter<t, &t::m>>`
2. `&t::m` 作为非类型模板参数，编译期就是一个偏移常量
3. `get_node(element)` 返回 `element.*node_ptr`，编译器用常量偏移直接生成寻址指令
4. 因为 getter 是 `static inline`，所有 `List::get_node(...)` 调用点都被内联展开，整个链表操作变成"直接操作元素内嵌的 node"（`UT_LIST_NODE_GETTER_DEFINITION` 注释强调 "can be fully inlined by the compiler"）

设计本质：**把原本是"链表头的运行时属性"（node 在哪）提升为"链表类型的编译期属性"**。这与 Linux 内核 `list_head` + `container_of`（用 `offsetof` 宏取偏移）思路同源，都是"编译期偏移"，只是 C++ 用成员指针更类型安全；而 Server 层用二级指针（`T **next`/`**last`）则是另一种寻址协议。

### 成员指针（pointer-to-member）语言机制剖析

前面论证了"为什么用偏移量"，这一节把 C++ 语言机制本身拆开——`ut_list_node<Type> Type::*node_ptr` 到底是个什么东西。

#### 语法拆解：这个声明怎么读

```cpp
template <typename Type, ut_list_node<Type> Type::*node_ptr>
//                        └────┬────┘└─┬──┘└─┬────┘
//                     成员的类型  |  参数名
//                       "Type 的"  |
//                              "成员指针"声明符
```

完整读法：`node_ptr` 是"**指向 `Type` 的 `ut_list_node<Type>` 类型成员的成员指针**"（pointer to member of `Type` of type `ut_list_node<Type>`）。配套的三个语法点：

- **取成员地址** `&t::m`（宏 `UT_LIST_BASE_NODE_T(t, m)` 展开出 `&t::m`）：这个 `&` **不是取内存地址**，是取"成员在类型内的相对位置"——产物是成员指针，不是普通指针
- **解引用** `element.*node_ptr`（`.*` 运算符）与 `p->*node_ptr`（`->*` 运算符）：与 `*`/`->` 平行，是 C++ 专门给成员指针配的两个解引用运算符
- **类型系统**：`ut_list_node<Type> Type::*` 与 `Type *` 是**完全不同、互不兼容**的类型——不能互相赋值、不能互相 cast

#### 成员指针是什么：与普通指针的三点本质差异

1. **不是运行时地址**。普通指针是"内存里某个字节的地址"；成员指针是"相对某类对象基址的**相对位置**"的抽象——它自己单独没有意义，必须配一个对象基址才能定位到具体内存
2. **不能与 `void *` 互转**。`reinterpret_cast` 禁止成员指针与对象指针互转（标准未定义行为）。这也意味着它**无法放进"通用回调"之类的类型擦除槽位**
3. **大小实现定义，且可能大于 `void *`**。单继承时通常 8 字节；多继承/虚继承时更大（Itanium ABI 下是 `{偏移, this调整}` 或 `{虚基类表索引, 偏移}` 结构体）

#### 底层表示：三种继承形态

| 继承形态 | 底层表示 | 说明 |
|---|---|---|
| 无继承 / 单继承（非多态） | 纯字节偏移 | 最简单：就是 `offsetof` 的值 |
| 多继承 / 多态 | 偏移 + this 调整 | 基类子对象不在对象头部，需先调整 `this` |
| 虚继承 | 虚基类表索引 + 偏移 | 虚基类位置运行期才定，需查表 |

**InnoDB 的场景恰好是最简单的一种**：`trx_t`、`buf_page_t`、`lock_t` 等元素都是非多态、无虚继承的纯数据类，所以 `&t::m` 的底层就是"成员相对对象基址的字节偏移"一个数——这才能被编译成 `mov rax, [elem + 0x1A8]` 里的立即数 `0x1A8`。**换做多继承对象，成员指针携带的 this 调整信息会让"零运行时开销"不成立**（解引用时要先加调整量），ut_list_base 能"零开销"依赖了这个前提。

#### 为什么能做模板非类型参数（NTTP）

C++ 非类型模板参数允许的类型自古就包括**成员指针**（整型、枚举、指针、引用、成员指针五类）。`&t::m` 在类模板实例化时是已知的编译期常量，所以可以：

```cpp
template <typename Type, ut_list_node<Type> Type::*node_ptr>   // NTTP
struct ut_list_base_explicit_getter { ... };
```

**这个"可作 NTTP"是整条设计链的机制前提**：如果成员指针只能当函数参数传，它在运行时就是普通变量（依赖 inline + 常量传播才能优化掉，不可靠）；正因为能固化进类型，编译器才保证每个实例化拿到确定的常量偏移并内联。

#### 为什么选成员指针而不是 `offsetof` 宏

两条路都能拿到"偏移量"，InnoDB 选成员指针是**类型安全**胜出：

| | `offsetof(Type, member)` | `&Type::member` |
|---|---|---|
| 形态 | 宏，展开成字节运算 | 语言原生表达式 |
| 类型 | `size_t`（裸数值，无类型信息） | `ut_list_node<Type> Type::*`（带完整类型） |
| 指错成员 | 不报错（`offsetof(trx_t, mysql_trx_list)` 照样算出个数） | **编译期报错**（类型不匹配 `ut_list_node<Type>` 的直接拒收） |
| 作 NTTP | 需先包装成常量类 | 直接支持 |

Linux 内核的 `container_of` 用 `offsetof` 是因为 C 没有成员指针；InnoDB 在 C++ 里用成员指针把"偏移"变成了**类型系统可见的实体**——指针指错成员、getter 类型对不上，都在编译期拦下。

#### `NodeGetter` 抽象层的作用

`get_node` 不是直接写在 `ut_list_base` 里，而是抽成 `NodeGetter` 模板参数——这是一个**策略点**：`ut_list_base_explicit_getter`（成员指针）只是默认策略，同一份链表代码可以换其他 getter（比如运行期通过函数指针/条件逻辑取 node 的场景）。成员指针被装进 getter 而不是散落在链表代码里，正是为了把"寻址"这一变化维度隔离在单一类型之后。

### 五种实现的寻址协议对照

| 实现 | 寻址协议 | 尾插 | 删除 | 运行时开销 |
|---|---|---|---|---|
| `ut_list_base` | 成员指针偏移（编译期） | O(1)（`last_element`） | O(1)（prev 指针） | **零**（立即数偏移） |
| `SQL_I_List` | `T **next` 二级指针 | O(1)（间接指针直挂） | 不可（单向） | 一次间接 |
| `base_list`/`List` | `**last` 二级指针 + 哨兵 | O(1) | O(n)（找前驱） | 一次间接 |
| `I_List`/`ilink` | `T **prev` 二级指针 + sentinel | O(1) | O(1)（unlink 自解链） | 一次间接 |
| C `LIST` | 裸 `prev`/`next` | O(1) | O(1) | 无抽象（C 直接） |

---

## 侵入 vs 非侵入、双向 vs 单向

### 侵入 vs 非侵入

| | 侵入式（ut_list_base / SQL_I_List / I_List） | 非侵入式（base_list/List / C LIST） |
|---|---|---|
| 节点 | 元素自带（`ut_list_node` / `next` 成员 / `ilink` 基类） | 链表分配 `list_node`（含 `void *info`） |
| 分配 | 插入零分配 | 每次插入分配一个节点（THR_MALLOC / MEM_ROOT） |
| 元素可拷贝？ | 不要求（元素不搬） | 只存指针，也不要求（但节点分配有代价） |
| 一个元素挂多链 | 可以（多个 node 成员） | 可以（同一指针塞进多条链） |
| 适用 | 内核/引擎热路径、不可拷贝对象 | server 层通用、生命周期统一管理 |

### 双向 vs 单向

- **双向**（ut_list_base / I_List / C LIST）：O(1) 删除（有 prev）、可反向遍历；代价是元素多一个指针字段。
- **单向**（SQL_I_List / base_list）：省一个指针；代价是删除要 O(n) 找前驱（SQL_I_List 干脆不提供删除，base_list 的 `remove` 靠迭代器持 prev 变 O(1)）。

---

## 用户全景

### ut_list_base（InnoDB 内部）

- 事务系统：`rw_trx_list`（活跃读写事务）、`mysql_trx_list`、`serialisation_list`（提交序列）
- Buffer pool：`flush_list`（脏页）、`LRU`、`free`、`unzip_LRU`、`zip_clean`、`zip_free`
- 锁系统：`trx->trx_locks`（一个事务持有的全部锁）、`rw_lock_list_t`
- 存储：`mem_block_t` 内存块链表、`dyn0buf.h` 动态缓冲区
- 其他：`ReadView` 链表、`fil_node_t` 的 `LRU`、`Token` 链表、`recv_t` 恢复链表、`que_thr_t` 队列等

### server 层四种

各实现的具体用户已在「结构组织」章各小节详列（含实例化点与用途）：`SQL_I_List`（FROM 表链 / ORDER BY / 触发器字段链 / DELETE 目标表等）、`base_list`/`List`（Item_cond 条件树 / 半连接临时表 / PURGE/KILL 参数等）、`I_List`（plugin-load 选项 / replicate-rewrite-db / Item_change_record / keycache 命名链）、C `LIST`（myisam/heap 打开表注册表 / thr_lock 线程链 / 客户端库 MYSQL_STMT 挂连接）。

共性判断：`List<T>` 是**历史底座**（头注释建议 "prefer std::vector<T> or std::list<T> to List<T> wherever possible"——新代码优先 STL）；C `LIST` 是连接库与存储引擎老代码的工具，server 核心已不用。

---

## 核心调用栈

以"插入元素到 rw_trx_list"为例（事务创建时）：

```
trx_start_low (trx0trx.cc)
  → trx_sys->rw_trx_list.push_back(trx) / ut_list_append (ut0lst.h)
    → List::get_node(*elem)   // 成员指针偏移，编译期常量
      → element.*node_ptr     // 编译后: mov rax, [elem + 0x1A8]
    → elem_node.prev/next 指针链接
    → list.update_length(1)   // count.store
```

以"遍历 rw_trx_list 输出 SHOW ENGINE INNODB STATUS 的 TRX LIST"为例：

```
srv_innodb_monitor (srv0srv.cc)
  → lock_trx_print / trx_list_traversal
    → ut_list_base::begin()/operator++   // 通过 NodeGetter 的 next() 推进
```

以"List 尾插"为例（base_list::push_back）：

```
List<Item>::push_back(item)
  → base_list::push_back(info)
    → *last = new (THR_MALLOC) list_node(info, &end_of_list)  // 二级指针直挂
    → last = &(*last)->next   // 推进到新尾的 next 字段位置
```

以"SQL_I_List 挂 ORDER BY 子句"为例（sql_parse.cc）：

```
Query_block::add_to_list (sql_parse.cc)
  → list.link_in_list(order, &order->next)
    → *next = order       // 尾节点的 next 字段 ← 新 ORDER
    → next = &order->next // 二级指针推进到新尾
```

以"I_List 挂 --plugin-load 选项"为例（sql_plugin.cc）：

```
process_plugin_load_list → opt_plugin_load_list.push_back(new i_string(ptr))
  → base_ilist::push_back(a)
    → *sentinel.prev = a           // 哨兵 prev 指向的槽位挂新元素
    → sentinel.prev = &a->next     // 哨兵 prev 推进到新尾
```

---

## 他库对比

- **Linux kernel `list_head`**：`struct list_head { next, prev }` 内嵌进元素，通过 `container_of` 宏（`offsetof` + 指针算术）从 node 反推元素地址。思路与 ut_list_base 相同（侵入式 + 编译期偏移），但 Linux 用宏 + `offsetof`，InnoDB 用 C++ 成员指针模板；Linux 的 `hlist` 用 `pprev` 二级指针——与 SQL_I_List 的 `T **next`、ilink 的 `T **prev` 同源
- **Boost.Intrusive / folly::IntrusiveList**：提供更全的侵入式容器族，同样基于成员指针（`boost::intrusive::member_hook`）——ut_list_base 是"只做一个双向链表"的最小侵入实现
- **`std::list`**：非侵入式，元素拷贝进独立节点，语义安全但多一次分配、缓存局部性差——InnoDB 对象不可拷贝所以不可用

---

## 相关的系统变量/状态变量

链表本身无系统变量。间接相关：

| 状态变量/输出 | 关联 |
|---------------|------|
| `SHOW ENGINE INNODB STATUS` 的 `TRANSACTIONS` 段 | 遍历 `trx_sys->rw_trx_list` / `mysql_trx_list` 输出事务 |
| `TRX LIST` / `LOCK WAIT` 段 | 遍历 `trx->trx_locks`、锁等待链表 |
| `BUFFER POOL AND MEMORY` 段 | 遍历 `buf_pool->flush_list` / `LRU` 统计页数 |
| `Innodb_buffer_pool_pages_dirty` | `flush_list` 长度 |

---

## 选择指南

- **引擎热路径、对象不可拷贝、要 O(1) 删除、要挂多链** → `ut_list_base`（成员指针偏移，零运行时开销）
- **server 层、元素自带 next 指针、只头尾插入不需要删除** → `SQL_I_List`
- **server 层通用容器、元素生命周期统一管理、要下标访问/排序** → `List<T>`（但新代码优先 STL）
- **侵入式双向 + 元素要能自解链（无链表头场景）** → `I_List`/`ilink`
- **C 代码** → `LIST`

---

## ★ 本机制里的工程实现技法

### 一、寻址协议三族：偏移量 / 二级指针 / 哨兵

五种实现收敛为三种寻址思路：**编译期偏移**（ut_list_base，零开销但类型纪律严）、**二级指针**（SQL_I_List 的 `T **next`、base_list 的 `**last`、ilink 的 `T **prev`——指向"某节点的指针字段"实现 O(1) 尾插/自解链）、**哨兵节点**（`end_of_list` 的 next 指向自己免 NULL 判断；base_ilist 的 sentinel 伪装成 T 让首尾操作统一）。

### 二、二级指针是"尾插 O(1) + 自解链"的通用解

`T **next` 指向"尾节点 next 字段的位置"，挂新元素 = `*next = elem`；`T **prev` 指向"前驱 next 字段的位置"，摘链 = `*prev = next`——**不需要知道邻居节点是谁，只需要知道改哪个指针字段**。这是 C 链表"漂亮且快"的经典手法（与 Linux 内核 `hlist` 的 `pprev` 同源）。

### 三、哨兵把边界情况折叠进普通路径

`end_of_list` 的 `next` 指向自己 → 遍历终止判 `node == &end_of_list`（无需判 NULL）；base_ilist 的 sentinel 被强转成 T → 空链/首尾插入都走同一套代码。**哨兵的代价是"它假装是元素"**——base_ilist 头注释直接承认 "inherently unsafe"，靠 `static_assert(!std::is_polymorphic)` 兜底。

### 四、排序换 info 不换节点

`List::sort` 用交换排序**只 swap `info` 指针**——节点顺序不变，排序前初始化的迭代器排序后依然安全（注释原文）。代价是 O(n²)（"the list to be sorted is supposed to be short"）。

---

## 坑与易混淆概念汇总

### 坑

1. **base_list 的浅拷贝构造是历史陷阱**：`base_list(const base_list &tmp)` 是浅拷贝且**所有权隐式转移**（注释原文："the old instance is not updated, so both objects end up sharing the same nodes... please do not use it in any new code"）。
2. **base_ilist 的 sentinel 伪装是 UB 边缘**：`static_cast<T *>(&sentinel)` + `SUPPRESS_UBSAN`，靠"T 与 ilink<T> 布局相同"的假设——多态类被 static_assert 挡掉，但其他布局差异靠纪律。
3. **SQL_I_List 无删除操作**：单向 + 无 prev，只能通过 `save_and_clear` 整体搬走再过滤。
4. **`I_List` 拷贝被禁止**：base_ilist 声明但不定义拷贝构造/赋值——"two list heads containing the same elements"。
5. **ut_list_base 的 count 是原子但写需外部 latch**（注释原文）：读可无锁（陈旧可容忍），写必须持锁——`update_length` 是 `store(load()+diff)` 非原子 RMW，并发写必丢更新。
6. **Removable 迭代器的契约**：不允许删除 prev 元素（锚点失效）；插在当前项正前会死循环；只测试过"从头开始"的路径（构造断言 `ut_ad(m_prev_elem == nullptr)`）。
7. **C LIST 无计数**：`list_length` 每次 O(n) 现数；`list_add`/`list_delete`/`list_reverse` 都会改变 root，**调用方必须接返回值**（忘接 = 链丢失）。
8. **`list_delete` 不清指针不释放**：被删节点的 prev/next 仍是旧值、内存不释放——与 `ut_list_remove`（置空节点指针）形成对比。
9. **成员指针"零开销"依赖非多态单继承**：多继承/虚继承对象上成员指针携带 this 调整/虚基类表索引，解引用不再是纯立即数偏移——`mov rax, [elem + 0x1A8]` 只在 InnoDB 这类纯数据元素上成立。
10. **`nullptr` 成员指针解引用是 UB**：`get_node` 从不检查——依赖"元素类型必有该 `ut_list_node` 成员"的纪律（模板类型系统保证，除非用 EXTERN 变体绕过）。

### 易混淆

- **`ut_list_node<Type>`（元素内的链接字段）≠ `ut_list_base<Type, NodeGetter>`（链表头）**：前者每元素一份，后者每条链表一份。
- **`UT_LIST_BASE_NODE_T` ≠ `UT_LIST_BASE_NODE_T_EXTERN`**：前者要求类型完整（能取 `&t::m`），后者用于前向声明场景，配合 `UT_LIST_NODE_GETTER_DEFINITION` 补 getter。
- **`SQL_I_List`（侵入式，元素自带 next）≠ `List`（非侵入式，链表分配节点）**：都叫 List 且都在 sql_list.h，但一个是"元素是链的一部分"一个是"节点包着元素指针"。
- **`I_List`（侵入式双向）≠ `List`（非侵入式单向）**：名字像，实现完全相反。
- **`base_list_iterator::remove` 靠迭代器持 prev 变 O(1)**：`List::remove` 本身无此便利（要走链找前驱）。
- **`List_STL_Iterator`（C++11 range-for）≠ `List_iterator`（老式 ++ 迭代器）**：后者可 replace/remove/after，前者只有前进。
- **老宏 `UT_LIST_*` 与新模板等价**：`ut0lst.h` 里宏是兼容层，直接转发到模板函数；新代码可用模板，存量代码仍走宏。`UT_LIST_INIT` 是 placement new 重跑构造（恢复 0xCAFE 魔数），不是 memset。
- **ut_list_base 无反向迭代器**：`rbegin`/`rend`/`reverse_iterator` 全仓 0 命中——反向遍历手工从 `last_element` 沿 prev 走。
- **`item_buff` 类型 8.0.39 已不存在**：sql_list.h 头注释 "Used for item and item_buffs" 是历史遗留。
- **C `LIST` 的节点≠数据**：节点是链表分配的 `LIST` 结构（prev/next/data 指针），`data` 指向外部元素——所以是**非侵入式**（元素不内嵌链接字段），不要与"节点即元素"的侵入式混淆。
- **`List_iterator_fast` 的 `++` 是 `next_fast` 不是 `next`**：不维护 prev/current，所以删除/替换/插入全禁用——用错迭代器（想删元素却用 fast 版）是静默错误。

---

## 参考

**经典算法**
- Linux 内核 `list_head` + `container_of`（`offsetof` 宏取偏移，与 ut_list_base 的成员指针同源）与 `hlist` 的 `pprev` 二级指针（与 SQL_I_List 的 `T **next` 同源）——源码无引用注释，为从实现反推的对应关系。

**相关文档**
- `ut_list_base` 的用户视角：事务链表见 [`../../innodb/trx.md`](../../innodb/trx.md)；buffer pool 各链表（flush_list/LRU 等）见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)；锁链表见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
- 数据结构的兄弟篇：哈希三实现见 [`hash.md`](hash.md)，队列两实现见 [`queue.md`](queue.md)
- 数据结构归属与全量清单见 [`../README.md`](../README.md)
