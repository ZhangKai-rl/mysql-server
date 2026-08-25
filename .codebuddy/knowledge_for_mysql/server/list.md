# MySQL 链表（List）基础设施深度解析

> 基于 MySQL 8.0.39 源码，涵盖 InnoDB 的 `ut_list_base`（侵入式 + 成员指针偏移）、Server 层的 `SQL_I_List` / `my_list.h` C 链表。本章先记录 InnoDB 实现，Server 实现后续补充。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [InnoDB: ut_list_base（侵入式 + 成员指针偏移）](#innodb-ut_list_base侵入式--成员指针偏移)
- [Server: SQL_I_List（待补）](#server-sql_i_list待补)
- [Server: my_list.h 的 C LIST（待补）](#server-my_listh-的-c-list待补)
- [三种实现对比（待补）](#三种实现对比待补)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

链表是 MySQL 最基础的数据结构之一，代码库中存在多套实现，按侵入性分为两类：

| 实现 | 位置 | 形态 | 侵入性 |
|------|------|------|--------|
| `ut_list_base` | `storage/innobase/include/ut0lst.h` | 模板双向链表，node 嵌在元素内 | 侵入式 |
| `SQL_I_List<T>` / `List<T>` | `sql/sql_list.h` | 模板链表 | 侵入式（间接指针）/ 非侵入式 |
| `LIST` | `include/my_list.h` | C 语言双向链表，`void *data` | 非侵入式 |

本章聚焦 InnoDB 的 `ut_list_base`（由老宏 `UT_LIST_BASE_NODE_T` 体系演变而来）。

### 用途

作为通用基础设施，InnoDB 内部几乎所有"按链组织、需 O(1) 插入删除"的场景都在用它，典型包括：

- 事务系统：`rw_trx_list`（活跃读写事务）、`mysql_trx_list`、`serialisation_list`（提交序列）
- Buffer pool：`flush_list`（脏页）、`LRU`、`free`、`unzip_LRU`、`zip_clean`、`zip_free`
- 锁系统：`trx->trx_locks`（一个事务持有的全部锁）、`rw_lock_list_t`
- 存储：`mem_block_t` 内存块链表、`dyn0buf.h` 动态缓冲区
- 其他：`ReadView` 链表、`fil_node_t` 的 `LRU`、`Token` 链表、`recv_t` 恢复链表、`que_thr_t` 队列等

### 版本演进

| 版本 | 变化 |
|------|------|
| 1995 | Heikki Tuuri 创建 `ut0lst.h`，最初是宏体系（`UT_LIST_*`），node 为 `ib_list_node_struct` |
| 2011.12 | Sunny Bains 重写为 C++ 模板：`ut_list_node<Type>` + `ut_list_base<Type, NodeGetter>` |
| 8.0 前后 | 新增 `Removable` 迭代器（遍历时可删当前项）、`UT_LIST_BASE_NODE_T_EXTERN`（前向声明场景）、原子化 `count` |

> 老宏（`UT_LIST_ADD_FIRST` 等）仍保留在 `ut0lst.h:326-456` 作为兼容层，直接转发到模板函数。

---

## 理论基础

### 设计模式

**侵入式容器（Intrusive Container）**：链表节点是元素对象自己的成员（`ut_list_node<Type>` 嵌在元素里），链表头只持有元素指针，不单独分配节点。对比非侵入式 `std::list`（节点由链表分配、元素被拷贝/移动进节点）。

**指针到成员（Pointer-to-Member）作非类型模板参数**：偏移量在编译期固化进类型，而非存在实例中——这是本实现零运行时开销的根基（详见正文）。

### 算法与数据结构

双向链表：已知元素指针时插入/删除均 O(1)；`first`/`last` 双端访问 O(1)；`count` 维护长度 O(1)。链表头含 `first_element`/`last_element` 两个端点 + 原子 `count`。

### 类似实现对比

- **Linux kernel `list_head`**：`struct list_head { next, prev }` 内嵌进元素，通过 `container_of` 宏（`offsetof` + 指针算术）从 node 反推元素地址。思路与 InnoDB 相同（侵入式 + 编译期偏移），但 Linux 用宏 + `offsetof`，InnoDB 用 C++ 成员指针模板
- **Boost.Intrusive / folly::IntrusiveList**：提供更全的侵入式容器族，同样基于成员指针（`boost::intrusive::member_hook`）
- **`std::list`**：非侵入式，元素拷贝进独立节点，语义安全但多一次分配、缓存局部性差

### 历史背景

1995 年 MySQL/InnoDB 代码合并早期，Heikki Tuuri 用纯 C 宏实现链表，便于 C 代码使用。2011 年 Sunny Bains 重写时引入模板化与类型安全，但保留宏作为兼容入口。设计演进动机：在"类型安全 + 零开销"和"历史代码兼容"之间做平衡。

---

## InnoDB: ut_list_base（侵入式 + 成员指针偏移）

### 3.1 两层结构：node 在元素内，base 是链表头

**元素内嵌的节点**（`ut0lst.h:49-61`）：

```cpp
template <typename Type>
struct ut_list_node {
  Type *prev;   /*!< pointer to the previous node, NULL if start of list */
  Type *next;   /*!< pointer to next node, NULL if end of list */
};
```

**链表头**（`ut0lst.h:80-139`）：

```cpp
template <typename Type, typename NodeGetter>
struct ut_list_base {
  elem_type *first_element{nullptr};
  elem_type *last_element{nullptr};
  std::atomic<size_t> count{0};   // 原子计数：读可不加锁，写需外部 latch
  ...
};
```

关键点：**链表头只存三个运行时字段（两个端点指针 + 计数），不存偏移量**。元素间通过 `ut_list_node` 的 `prev`/`next` 直接互链。

### 3.2 偏移量在哪：编译期模板参数，不在实例中

"链表怎么知道元素的 node 成员在哪"由 `NodeGetter` 回答。核心是 `ut_list_base_explicit_getter`（`ut0lst.h:245-251`）：

```cpp
template <typename Type, ut_list_node<Type> Type::*node_ptr>
struct ut_list_base_explicit_getter {
  static const ut_list_node<Type> &get_node(const Type &element) {
    return element.*node_ptr;   // 用成员指针访问元素内的 node 字段
  }
};
```

`ut_list_node<Type> Type::*node_ptr` 是 C++ **成员指针（pointer-to-member）**，作为**非类型模板参数**。对非多态类、非多重继承，成员指针底层就是一个常量偏移量（元素内 node 成员相对对象头的字节偏移）。

宏 `UT_LIST_BASE_NODE_T`（`ut0lst.h:257-258`）把"元素类型 + 成员名"展开成完整链表类型：

```cpp
#define UT_LIST_BASE_NODE_T(t, m) \
  ut_list_base<t, ut_list_base_explicit_getter<t, &t::m>>
```

`&t::m` 就是那个成员指针。于是：

| 信息 | 存在哪 | 何时确定 |
|------|--------|----------|
| `first_element`/`last_element`/`count` | base 实例的内存 | 运行时 |
| 成员偏移量 `&t::m` | 编译期固化进类型（模板非类型参数） | 编译期 |

同一类型的链表共享同一个偏移量常量，**实例不携带偏移**，没有运行时成本。

### 3.3 零运行时开销的机理

`get_node` 是 `static inline`，`element.*node_ptr` 中 `node_ptr` 是编译期常量，所以编译器直接把它优化成"固定偏移的内存访问"：

```asm
mov rax, [rdi + 0x1A8]   ; rdi = 元素指针，0x1A8 = 编译期算出的偏移
```

生成的机器码与手写 `elem.list_member` 完全一样。`UT_LIST_NODE_GETTER_DEFINITION`（`ut0lst.h:274-276`）注释明确强调 "can be fully inlined by the compiler"——遍历代码全部内联成直接偏移访问，因此**侵入式链表能像手写 C 链表一样快**。

### 3.4 两个宏与"定义在前向声明场景"的处理

- `UT_LIST_BASE_NODE_T(t, m)`：要求 `t` 已完整定义（能取 `&t::m`）
- `UT_LIST_BASE_NODE_T_EXTERN(t, m)`（`ut0lst.h:284-285`）：当 `t::m` 尚未在作用域内（比如 header 中只有前向声明）时使用，配合 `UT_LIST_NODE_GETTER_DEFINITION(t, m)`（`ut0lst.h:274-276`）在 `t` 定义完成后补上 getter 定义

典型例子：`lock0types.h:88` 用 `UT_LIST_BASE_NODE_T_EXTERN(lock_t, trx_locks)`，因为 `trx_locks` 成员定义在更晚的 `lock0priv.h`。

### 3.5 为什么要用偏移量：从问题到方案的推演

#### 前提一：InnoDB 必须用侵入式链表

链表的"侵入式 vs 非侵入式"不是风格选择，而是被 InnoDB 对象的性质逼出来的：

- **对象不可拷贝/移动**：`trx_t`、`buf_page_t`、`lock_t` 这些对象被指针跨结构引用（事务引用锁、锁引用事务、页被多个队列引用），拷贝意味着要重挂所有外部指针，根本不现实。而 `std::list` 式非侵入链表恰恰要求元素可拷贝/移动进节点
- **零内存分配**：插入/删除只改指针，不 `malloc`。InnoDB 里这些操作发生在持锁/热路径上（buffer pool 每页状态迁移、事务提交），分配内存意味着额外的锁与失败处理
- **缓存局部性**：node 就嵌在元素里，`elem->next` 和 `elem->data` 在同一 cache line 内，一次缓存行加载同时拿到数据和链接

侵入式 = node 内嵌在元素里、链表头只持元素指针。**这正是偏移量问题的来源。**

#### 前提二：侵入式必然面临"元素指针 → node"的转换

链表操作（插、删、遍历）操作的物理实体是 node（prev/next），但链表头只存了 `elem_type *`。所以每次操作都要回答同一个问题：

> 给定一个元素指针，它的 `ut_list_node` 成员在这个元素内的哪里？

这就是偏移量问题的本质——**它是侵入式链表的"寻址协议"**。

#### 三种候选方案对比

| 方案 | 做法 | 代价 |
|------|------|------|
| A. 运行时存偏移 | 链表头存 `size_t node_offset`，访问时 `(char*)elem + node_offset` | 每实例 +8 字节；每次访问多一次内存 load + 加法；且偏移是运行期值，编译器无法内联 |
| B. 非侵入式 | 放弃侵入，node 独立分配 | 回到前提一的三个不可接受点 |
| C. 编译期偏移（InnoDB 选） | 偏移作为模板非类型参数 `&t::m` | 零空间、零时间（见下） |

#### 为什么偏移量是编译期常量而非运行时字段

核心洞察：**元素类型一旦定义，node 成员的偏移就是固定的**。`&t::m` 是编译期可知的常量，没有理由让它变成运行时数据。把它固化成模板参数后：

- **零空间**：偏移量不占链表头任何字节，同一类型的链表共享一个编译期常量
- **零时间**：编译器把 `element.*node_ptr` 内联成一条带立即数偏移的访问指令 `mov rax, [elem + 0x1A8]`，连加法都没有——`0x1A8` 是编码进指令里的立即数。如果运行时存偏移，每次都要先读偏移再做一次加法，且读偏移本身是内存访问
- **类型安全**：`ut_list_node<Type> Type::*node_ptr` 的类型约束保证这个偏移只能指向 `ut_list_node<Type>` 类型的成员——把偏移写错（指向别的成员）会在编译期报错，而不是运行时读到脏数据

这正是"为什么用偏移量"的完整答案：**侵入式必须解决寻址，而寻址信息在编译期就确定，所以用编译期常量偏移；只有这样才能做到零运行时开销。**

#### 实现思路：成员指针 + 模板，把"寻址"变成"类型的一部分"

普通成员指针 `Type::*` 的底层就是一个偏移量（非多态、非多重继承时）。InnoDB 没有在链表头里塞一个运行时偏移值，而是把偏移"装进类型"：

1. `UT_LIST_BASE_NODE_T(t, m)` 把用户写的 `t, m` 展开为 `ut_list_base<t, ut_list_base_explicit_getter<t, &t::m>>`（`ut0lst.h:257-258`）
2. `&t::m` 作为非类型模板参数，编译期就是一个偏移常量
3. `get_node(element)` 返回 `element.*node_ptr`（`ut0lst.h:247-250`），编译器用常量偏移直接生成寻址指令
4. 因为 getter 是 `static inline`，所有 `List::get_node(...)` 调用点都被内联展开，整个链表操作变成"直接操作元素内嵌的 node"

设计本质：**把原本是"链表头的运行时属性"（node 在哪）提升为"链表类型的编译期属性"**。这与 Linux 内核 `list_head` + `container_of`（用 `offsetof` 宏取偏移）思路同源，都是"编译期偏移"，只是 C++ 用成员指针更类型安全；而 Server 层 `SQL_I_List` 用 `T **next` 间接指针则是另一种寻址协议（见对比章节）。

### 3.6 与 Server 层实现的本质差异（预告）

InnoDB 的偏移量固化在类型中；而 Server 层 `SQL_I_List<T>`（`sql/sql_list.h:45-100`）用 `T **next`（指向"下一个元素的 next 字段位置"的指针）做间接链接，是**指针间接而非偏移量**方案。两者差异详见后续章节。

---

## Server: SQL_I_List（待补）

> 占位。`sql/sql_list.h:45-100`：侵入式单向链表，`T *first` + `T **next` 间接指针，用于 item 链、prepared statement 等。

---

## Server: my_list.h 的 C LIST（待补）

> 占位。`include/my_list.h:36-52`：C 语言非侵入式双向链表，`void *data`，C 客户端库用。

---

## 三种实现对比（待补）

> 占位。对比维度：侵入性 / 偏移量 vs 间接指针 / 类型安全 / 缓存局部性 / 适用场景。

---

## 核心调用栈

以"插入元素到 rw_trx_list"为例（事务创建时）：

```
trx_start_low (trx0trx.cc)
  → trx_sys->rw_trx_list.push_back(trx) / ut_list_append (ut0lst.h:333)
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

## Misc

### 易混淆：`ut_list_node` vs `ut_list_base`

- `ut_list_node<Type>`：**元素内部**的链接字段（prev/next），每个元素一份
- `ut_list_base<Type, NodeGetter>`：**链表头**（first/last/count），每条链表一份

### 易混淆：`UT_LIST_BASE_NODE_T` vs `UT_LIST_BASE_NODE_T_EXTERN`

- 前者要求元素类型已完整定义（直接取 `&t::m`）
- 后者用于"定义尚未进入作用域"的场景，配合 `UT_LIST_NODE_GETTER_DEFINITION` 前向声明 getter，避免循环 include

### 历史残留

`UT_LIST_*` 宏（`ut0lst.h:326-456`）是老接口的兼容层，新代码可直接用模板函数；但大量存量代码仍走宏，两者等价。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/include/ut0lst.h:49-61` | `ut_list_node<Type>`：元素内嵌的 prev/next 节点 |
| `ut0lst.h:80-243` | `ut_list_base`：链表头 + 迭代器 + Removable |
| `ut0lst.h:245-251` | `ut_list_base_explicit_getter`：成员指针非类型模板参数，偏移编译期固化 |
| `ut0lst.h:257-258` | `UT_LIST_BASE_NODE_T(t, m)` 宏 |
| `ut0lst.h:274-276` | `UT_LIST_NODE_GETTER_DEFINITION(t, m)` 宏 |
| `ut0lst.h:284-285` | `UT_LIST_BASE_NODE_T_EXTERN(t, m)` 宏 |
| `ut0lst.h:296-547` | 操作函数族（prepend/append/insert/remove/reverse/move_to_front/validate） |
| `storage/innobase/include/trx0sys.h:513-551` | 事务系统三链表（serialisation/rw_trx/mysql_trx） |
| `storage/innobase/include/buf0buf.h:2398-2503` | buffer pool 各链表（flush_list/LRU/free 等） |
| `storage/innobase/include/lock0types.h:88` | `trx_lock_list_t`：事务持有的锁链表 |
| `sql/sql_list.h:45-100` | `SQL_I_List<T>`：Server 侵入式链表（待补章节） |
| `include/my_list.h:36-52` | C 链表 `LIST`（待补章节） |
