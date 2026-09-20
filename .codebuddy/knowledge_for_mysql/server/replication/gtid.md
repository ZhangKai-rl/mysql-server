# MySQL GTID 机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 SID/SIDNO/GNO 定义、Gtid_set 结构、GNO 分配算法、GTID 生命周期完整流程、三条持久化路径、Clone 双 buffer 机制。

## 目录

- [核心概念：SID / SIDNO / GNO](#核心概念sid--sidno--gno)
- [GTID 核心类结构与 Gtid_set 区间算法](#gtid-核心类结构与-gtid_set-区间算法)
  - [Gtid 类与 Gtid_specification](#gtid-类与-gtid_specification)
  - [Sid_map：SID ↔ SIDNO 双向映射](#sid_mapsid--sidno-双向映射)
  - [Gtid_set 内部实现](#gtid_set-内部实现)
  - [Gtid_set 的 C++ 实现深水区](#gtid_set-的-c-实现深水区)
  - [Owned_gtids](#owned_gtids)
  - [Gtid_state 锁体系](#gtid_state-锁体系)
- [GTID 相关系统变量：读写语义](#gtid-相关系统变量读写语义)
  - [gtid_mode](#gtid_mode)
  - [enforce_gtid_consistency](#enforce_gtid_consistency)
  - [gtid_next](#gtid_next)
  - [gtid_purged](#gtid_purged)
  - [GTID 复制协议：exclude 集合（COM_BINLOG_DUMP_GTID）](#gtid-复制协议exclude-集合com_binlog_dump_gtid)
  - [gtid_executed / gtid_owned](#gtid_executed--gtid_owned)
  - [binlog_gtid_simple_recovery / session_track_gtids / gtid_executed_compression_period](#binlog_gtid_simple_recovery--session_track_gtids--gtid_executed_compression_period)
- [GNO 分配算法](#gno-分配算法)
- [GTID 生命周期](#gtid-生命周期)
  - [服务器启动与初始化](#服务器启动与初始化)
  - [事务执行前——GTID 一致性检查](#事务执行前gtid-一致性检查)
  - [Flush Stage——GTID 分配与写入 binlog](#flush-stagegtid-分配与写入-binlog)
  - [Sync Stage——fsync 持久化](#sync-stagefsync-持久化)
  - [Commit Stage——GTID 外部化](#commit-stagegtid-外部化)
  - [InnoDB GTID 持久化（异步刷表）](#innodb-gtid-持久化异步刷表)
  - [Binlog Rotate 时的持久化](#binlog-rotate-时的持久化)
  - [崩溃恢复：源码级展开](#崩溃恢复源码级展开)
  - [Rollback 时的 GTID 处理](#rollback-时的-gtid-处理)
- [GTID 持久化：三条路径（概览）](#gtid-持久化三条路径概览)
- [Clone_persist_gtid：InnoDB 侧 GTID 持久化（源码逐段）](#clone_persist_gtidinnodb-侧-gtid-持久化源码逐段)

---

## 核心概念：SID / SIDNO / GNO

### SID（Source ID）

128 位的 UUID，标识一个 MySQL 实例（准确说是标识一个 GTID 来源）。每个 server 在启动时生成或读取 `auto.cnf` 中的 UUID 作为自己的 SID。**SID 不是 server_id**（server_id 是 32 位整数，用于 binlog event 头和复制区分，与 GTID 体系无关）。

### SIDNO（Source ID Number）

SID 在本实例内的整数索引。`Sid_map` 维护一个 `SID` 到 `SIDNO` 的映射表，每个 SID 在本地被编号为 1, 2, 3...。SIDNO 是**本地编号**，非全局唯一——不同实例的 Sid_map 可能给同一个 SID 分配不同的 SIDNO。

### GNO（Global Transaction Number）

int64 序列号，与 SID 组合形成完整的 GTID（`SID:GNO`，如 `8e75c5ea-2403-11f0-b1bb-34210bd0f65d:17`）。

**GNO 与 `trx_id` / `trx_no` 完全无关**。`trx_id` 是 InnoDB 内部事务标识（写进行记录的 `DB_TRX_ID`），`trx_no` 是提交时分配的序列化号（决定 purge 和 MVCC），GNO 是复制层面的全局事务标识。三者属于不同层面。

### GTID 的表示

一个 GTID 由 `(SIDNO, GNO)` 二元组唯一确定。在 binlog 中以 `Gtid_log_event` 携带，在内存中以 `Gtid` 结构体表示。

---

## GTID 核心类结构与 Gtid_set 区间算法

> 命名核实（8.0.39 事实）：区间合并函数名为 `Gtid_set::add_gno_interval`（**不存在** `add_gtid_interval`）；单点增删接口名为 `_add_gtid` / `_remove_gtid`（下划线前缀，调用者须持锁）；`Sid_map` 查询接口名为 `sid_to_sidno` / `sidno_to_sid`（**不存在** `get_sidno_for_uuid` / `get_sid_for_sidno`）；**不存在** `Sid_map::Lock` RAII 类（RAII 封装是 `Checkable_rwlock::Guard`）。

### 基本类型

```cpp
typedef int rpl_sidno;                              // SIDNO 是 int
using rpl_gno = binary_log::gtids::gno_t;           // GNO 是 int64
const rpl_gno GNO_END = INT64_MAX;                  // "越界哨兵"，不是合法 GNO
const rpl_gno GNO_WARNING_THRESHOLD = (GNO_END / 100) * 99;  // 超过 99% 报警
```

### Gtid 类与 Gtid_specification

`Gtid` 是 POD 结构（含在 `THD::variables` 里）：

```cpp
struct Gtid {
  rpl_sidno sidno;
  rpl_gno gno;
  void set(rpl_sidno sidno_arg, rpl_gno gno_arg) { ... }
  bool is_empty() const { ... return sidno == 0; }   // 空 = sidno==0（gno 必须同步为 0）
  bool equals(const Gtid &other) const { return sidno == other.sidno && gno == other.gno; }
  int to_string(const rpl_sid &sid, char *buf) const;
  enum_return_status parse(Sid_map *sid_map, const char *text);
};
```

- **`Gtid` 没有 `operator<`/`operator==`，只有 `equals()`**——有序性由 `Gtid_set` 的区间迭代器在链表层保证
- `parse()`：吃 36 字节 UUID → `sid_map->add_sid(sid)` 取 sidno → 必须紧跟 `:` → 解析 GNO（必须 >0）→ 必须以 `\0` 结尾。GNO 为 0/负/垃圾一律报 `ER_MALFORMED_GTID_SPECIFICATION`

`enum_gtid_type` 共 **6 个枚举值**（不是常见资料说的 4 个）：

```cpp
enum enum_gtid_type {
  AUTOMATIC_GTID = 0,   // GTID_NEXT='AUTOMATIC'；==0 保证 memset 后默认值正确
  ASSIGNED_GTID,        // GTID_NEXT='UUID:NUMBER'；gtid 字段有效
  ANONYMOUS_GTID,       // GTID_NEXT='ANONYMOUS'
  UNDEFINED_GTID,       // ASSIGNED 提交后、下一个事务开始前（防非事务表跨事务拆分）
  NOT_YET_DETERMINED_GTID, // 从库读到旧格式 relay log 开头、尚未见 Gtid_log_event 时
  PRE_GENERATE_GTID     // applier 处理 Anonymous_gtid_log_event 时的瞬时内部状态
};
```

`Gtid_specification` = `{enum_gtid_type type; Gtid gtid;}`。解析规则完整：

```cpp
enum_return_status Gtid_specification::parse(Sid_map *sid_map, const char *text) {
  if (my_strcasecmp(&my_charset_latin1, text, "AUTOMATIC") == 0) {
    type = AUTOMATIC_GTID;  gtid.sidno = 0;  gtid.gno = 0;
  } else if (my_strcasecmp(&my_charset_latin1, text, "ANONYMOUS") == 0) {
    type = ANONYMOUS_GTID;  gtid.sidno = 0;  gtid.gno = 0;
  } else {
    PROPAGATE_REPORTED_ERROR(gtid.parse(sid_map, text));  // 解析失败在此报错
    type = ASSIGNED_GTID;
  }
  RETURN_OK;
}
```

即 `gtid_next` 三种取值："AUTOMATIC"/"ANONYMOUS" 大小写不敏感；其余任意文本按 `SID:GNO` 解析。`to_string()` 特例：`UNDEFINED_GTID` 按 `ASSIGNED_GTID` 打印。

### Sid_map：SID ↔ SIDNO 双向映射

```
              _sidno_to_sid: Prealloced_array<Node*,8>       _sorted: 按 SID 字节序排序
              （按下标 sidno-1 索引）
 sidno=1 ──────────► Node{1, uuid_A} ◄────────────┐
 sidno=2 ──────────► Node{2, uuid_B} ◄──────┐     │
 sidno=3 ──────────► Node{3, uuid_C} ◄──┐   │     │
                                        │   │     │
              _sid_to_sidno: malloc_unordered_map<rpl_sid, Node*>
                uuid_A ──► Node{1,A}   uuid_B ──► Node{2,B}   uuid_C ──► Node{3,C}
              （hash 与数组指向同一个 Node，Node 归 hash 所有）
```

★ `add_sid()` 的"读锁→写锁升级"（这是最精妙的一段）：

```cpp
rpl_sidno Sid_map::add_sid(const rpl_sid &sid) {
  if (sid_lock) sid_lock->assert_some_lock();       // 调用者必须持 rdlock 或 wrlock
  auto it = _sid_to_sidno.find(sid);
  if (it != _sid_to_sidno.end())                    // 快路径：纯读锁下命中即返回
    return it->second->sidno;

  bool is_wrlock = false;
  if (sid_lock) {
    is_wrlock = sid_lock->is_wrlock();
    if (!is_wrlock) { sid_lock->unlock(); sid_lock->wrlock(); }  // ★ 升级
  }
  rpl_sidno sidno;
  it = _sid_to_sidno.find(sid);                     // ★ 二次查找：升级窗口期可能被并发插入
  if (it != _sid_to_sidno.end())
    sidno = it->second->sidno;
  else {
    sidno = get_max_sidno() + 1;                    // ★ sidno 只增不复用，永远追加在末尾
    if (add_node(sidno, sid) != RETURN_STATUS_OK) sidno = -1;
  }
  if (sid_lock && !is_wrlock) { sid_lock->unlock(); sid_lock->rdlock(); }  // 降级回读锁
  return sidno;
}
```

`add_node()` 把同一个 `Node*` push 进 `_sidno_to_sid`、`_sorted`、`_sid_to_sidno` 三处；失败回滚前两处。★ 关键回调（这就是 `ensure_sidno()` 必须存在的理由）：写锁下回调 `gtid_state->ensure_sidno()`——所有按 sidno 作数组下标的容器（`executed_gtids`/`lost_gtids`/`gtids_only_in_table`/`previous_gtids_logged` 的 `m_intervals`、`owned_gtids` 的 `sidno_to_hash`、`sid_locks` 的 `Mutex_cond_array`）必须同步扩容到 N，否则新 sidno 索引越界。

### Gtid_set 内部实现

#### Interval：**[start, end) 半开区间**

```cpp
  struct Interval {
    rpl_gno start;   /// The first GNO of this interval.
    rpl_gno end;     /// The first GNO after this interval.  ← 即开区间端点
    Interval *next;  // 同一 sidno 内单向链表，按 GNO 升序
  };
```

★ **end 是开区间端点**：区间覆盖 `{start, ..., end-1}`。证据：`_add_gtid(sidno, gno)` 调 `add_gno_interval(&ivit, gno, gno + 1, ...)`（单点是 `[gno, gno+1)`）；`to_string` 打印 `end - 1`；`get_last_gno` 返回 `end - 1`。**半开区间使合并/分裂/交集判断全程无 ±1 边界调整**。

#### m_intervals 稀疏数组 + free_intervals 空闲链表

```cpp
  Prealloced_array<Interval *, 8> m_intervals;  // 下标 sidno-1 → 该 sidno 区间链表头
  Interval *free_intervals;                     // 空闲 Interval 单链表（栈式）
  Interval_chunk *chunks;                       // 按块分配：每块 CHUNK_GROW_SIZE=8 个 Interval
```

`get_free_interval()`：free 链表非空弹出；空则 `create_new_chunk(8)`（OOM 重试 `MAX_NEW_CHUNK_ALLOCATE_TRIES` 次仍失败直接 `_exit`）整块挂入再弹。`put_free_interval()`：头插回收。**删除的 Interval 不归还 OS，只回收入池**，集合清空后可复用。

#### add_gno_interval：区间合并（核心算法，完整）

```cpp
void Gtid_set::add_gno_interval(Interval_iterator *ivitp, rpl_gno start,
                                rpl_gno end, Free_intervals_lock *lock) {
  assert(start > 0);  assert(start < end);
  Interval *iv;
  Interval_iterator ivit = *ivitp;
  has_cached_string_length = false;            // 字符串长度缓存失效

  while ((iv = ivit.get()) != nullptr) {
    if (iv->end >= start) {
      if (iv->start > end)
        // (start,end) 严格在当前区间之前：无交集，跳出到下方"插入新节点"
        break;
      // 到这里必有交集或相邻
      if (iv->start < start) start = iv->start;      // 向前扩
      while (iv->next && end >= iv->next->start) {   // ★ 吞并后续所有相交/相邻区间
        lock->lock_if_not_locked();
        ivit.remove(this);                            // 被吞区间回收进 free 链表
        iv = ivit.get();
      }
      iv->start = start;                              // 就地扩写当前区间
      if (iv->end < end) iv->end = end;               // 向后扩
      *ivitp = ivit;
      return;
    }
    ivit.next();                                      // 完全在左侧，继续找
  }
  // 与任何已有区间均不相交：从 free 链表取新节点，插到当前迭代位置之前
  Interval *new_iv;
  lock->lock_if_not_locked();
  get_free_interval(&new_iv);
  new_iv->start = start;
  new_iv->end = end;
  ivit.insert(new_iv);
  *ivitp = ivit;
}
```

迭代器 `Interval_iterator` 的 `p` 持有**"上一个节点的 next 指针的地址"**，因此 `insert/remove` 都是 O(1) 指针改写。注意合并条件是 `>=`：**相邻区间（无缝隙）也合并**。

```
add_gno_interval 合并示意：链表 [1,3) -> [5,6) -> [9,11)，add [6,9)：
  [5,6): end=6 >= start=6 ✓ → start 不变
         next=[9,11): end=9 >= 9 ✓ → remove [9,11)
         改写 iv=[5,11)
  结果：[1,3) -> [5,11)      （四点 + 两处相邻合并成一段）
```

#### remove_gno_interval：差集删除与区间分裂（完整）

```cpp
void Gtid_set::remove_gno_interval(Interval_iterator *ivitp, rpl_gno start,
                                   rpl_gno end, Free_intervals_lock *lock) {
  assert(start < end);
  Interval_iterator ivit = *ivitp;
  Interval *iv;
  has_cached_string_length = false;

  while (true) {                                  // 跳过完全在左侧的区间
    iv = ivit.get();
    if (iv == nullptr) goto ok;
    if (iv->end > start) break;
    ivit.next();
  }
  if (iv->start < start) {
    if (iv->end > end) {
      // ★ 删除区间落在 iv 中部：一分为二（唯一会分配新节点的删除路径）
      Interval *new_iv;
      lock->lock_if_not_locked();
      get_free_interval(&new_iv);
      new_iv->start = end;  new_iv->end = iv->end;
      iv->end = start;
      ivit.next();
      ivit.insert(new_iv);
      goto ok;
    }
    iv->end = start;                              // 只削掉尾巴，继续
    ivit.next();
    iv = ivit.get();
    if (iv == nullptr) goto ok;
  }
  while (iv->end <= end) {                        // 完全被覆盖的整段删除（回收）
    lock->lock_if_not_locked();
    ivit.remove(this);
    iv = ivit.get();
    if (iv == nullptr) goto ok;
  }
  if (iv->start < end) iv->start = end;           // 只削掉头部
ok:
  *ivitp = ivit;
}
```

```
分裂示意：remove [4,6) 于 [1,8)：  [1,8) → [1,4) -> [6,8)
```

#### 集合运算

- `add_gtid_set(other)`（并集）：逐区间调 `add_gno_interval`；两集合 Sid_map 不同时按 `sidno_to_sid` 翻译
- `remove_gtid_set(other)`：逐区间调 `remove_gno_interval`
- ★ `is_subset` 的 `is_interval_subset`（双指针 O(n+m)）：

```cpp
static bool is_interval_subset(Const_interval_iterator *sub,
                               Const_interval_iterator *super) {
  const Interval *super_iv = super->get();
  const Interval *sub_iv = sub->get();
  do {
    if (super_iv == nullptr) return false;
    while (sub_iv->start > super_iv->end) {        // 跳过 super 中完全在左侧的区间
      super->next();
      super_iv = super->get();
      if (super_iv == nullptr) return false;
    }
    if (sub_iv->start < super_iv->start || sub_iv->end > super_iv->end)
      return false;                                // 越过边界即不是子集
    sub->next();
    sub_iv = sub->get();
  } while (sub_iv != nullptr);
  return true;
}
```

- ★ `intersection(other, result)` 的实现是**集合恒等式** `A∩B == A - (A - B)`，而非逐区间求交——代码注释自认"简单、比直接求交略慢"，换取复用成熟的 remove 算法

#### to_string 与 add_gtid_text

- `to_string` 按 **UUID 字典序**（`_sorted`）而非 sidno 序遍历——确定性输出；区间间 `:` 分隔；**只有 `iv->end > iv->start + 1` 才输出 `-` 与 `end - 1`**（半开转闭区间文本，单点 GTID 不打连字符）
- ★ `String_format` 提供八项分隔符配置（`sid_gno_separator`/`gno_start_end_separator`/`gno_gno_separator`/`empty_set_string`…），预置 `default_string_format` / `sql_string_format`（单引号+换行）/ `commented_string_format`（`# ` 前缀）；配合 `has_cached_string_length` 三元组缓存**字符串长度**——**相同 String_format 重复序列化零开销**（任何 add/remove 都会失效缓存）
- `add_gtid_text(text)`：允许前导 `+`（配合 `@@GLOBAL.GTID_PURGED` 的追加语义）；首次调用先数冒号个数**一次性预分配 chunk**；解析 36 字节 UUID → `add_sid` → `ensure_sidno` → 逐 `:` 项解析 GNO（支持 0x 进制），`-` 后解析 end 并 **`end++` 转半开**；`end <= start` 的区间静默丢弃；利用"迭代器惯性"（当前位置已在 start 之前就不重头扫）减少重复扫描

#### ensure_sidno / 无 lossy filter（核实）

`ensure_sidno(sidno)`：`m_intervals.size() < sidno` 时同样"读锁→写锁升级→复查→补齐 nullptr→降级"，并 assert `sidno <= sid_map->get_max_sidno()`。**★ 核实结论：8.0.39 的 `Gtid_set` 没有 lossy filter 机制**——不存在 `m_filters` 成员（全仓库搜索无此符号）；区间数量由可用内存硬性支撑，OOM 路径是 `create_new_chunk` 重试后 `_exit`。

### Gtid_set 的 C++ 实现深水区

> 上一节是算法；这一节是**实现技艺**——为什么这么写、每处 C++ 技巧服务于什么目标。★ 源码位置勘误：8.0.39 中**不存在 `sql/rpl_gtid_set.h`**——`Gtid_set` 全部声明（含嵌套类）在 `sql/rpl_gtid.h`，实现在 `sql/rpl_gtid_set.cc`；`Prealloced_array` 在 `include/prealloced_array.h`，`malloc_unordered_map` 在 `include/map_helpers.h`（底层 allocator 在 `sql/malloc_allocator.h`）。**不存在**的符号：`is_forward()`、`_contains_gtid`、`get_last_sidno`、`Gtid_set::operator=`（从未显式声明）。

#### Interval_iterator_base：二级指针 `p` 的设计

```cpp
  template <typename Gtid_set_p, typename Interval_p>
  class Interval_iterator_base {
   public:
    Interval_iterator_base(Gtid_set_p gtid_set, rpl_sidno sidno) {
      assert(sidno >= 1 && sidno <= gtid_set->get_max_sidno());
      init(gtid_set, sidno);
    }
    /// Construct a new iterator over the free intervals of a Gtid_set.
    Interval_iterator_base(Gtid_set_p gtid_set) {
      p = const_cast<Interval_p *>(&gtid_set->free_intervals);
    }
    /// Reset this iterator.
    inline void init(Gtid_set_p gtid_set, rpl_sidno sidno) {
      p = const_cast<Interval_p *>(&gtid_set->m_intervals[sidno - 1]);
    }
    /// Advance current_elem one step.
    inline void next() {
      assert(*p != nullptr);
      p = const_cast<Interval_p *>(&(*p)->next);
    }
    /// Return current_elem.
    inline Interval_p get() const { return *p; }

   protected:
    /**
      Holds the address of the 'next' pointer of the previous element,
      or the address of the initial pointer into the list, if the
      current element is the first element.
    */
    Interval_p *p;
  };
```

- **`p` 是"指向 next 指针的指针"（link-pointer 迭代器）**。`init` 让 `p` 指向 `m_intervals[sidno-1]`——即**链表头指针本身的地址**。此时 `*p` 就是链头
- ★ **`next()` 的指针体操**：`p = &(*p)->next`。它不记"当前节点"，而记"当前节点的 next 字段的地址"——因此迭代器可以随时改写所在位置的链接关系，这是 `insert/remove` 能安全工作的根基
- **`const_cast` 的双重职责**：对 `Const_interval_iterator`，把 `Interval * const *`（const 对象成员的地址）转成 `const Interval **`——顶层剥 const 对象的 const（C++"相似类型"允许各层 cv 变化），内层把 `Interval *` 重绑定为 `const Interval *`（保证只读）
- **free-interval 构造**：直接让 `p` 指向 `free_intervals` 链头——**同一个迭代器类同时服务"区间链表"和"空闲链表"两种语义**

##### set / insert / remove 的指针体操与访问控制

```cpp
  class Interval_iterator
      : public Interval_iterator_base<Gtid_set *, Interval *> {
   private:
    inline void set(Interval *iv) { *p = iv; }
    /// Insert the given element before current_elem.
    inline void insert(Interval *iv) {
      iv->next = *p;
      set(iv);
    }
    /// Remove current_elem.
    inline void remove(Gtid_set *gtid_set) {
      assert(get() != nullptr);
      Interval *next = (*p)->next;
      gtid_set->put_free_interval(*p);
      set(next);
    }
    /**
      Only Gtid_set is allowed to use set/insert/remove.

      They are not safe to use from other code because: (1) very easy
      to make a mistakes (2) they don't clear cached_string_format or
      cached_string_length.
    */
    friend class Gtid_set;
  };
```

- ★ `insert(iv)`：`iv->next = *p; *p = iv;`——把新节点接到当前位置**之前**。因为 `*p` 是"指向当前位置的链接槽"，改它即可完成插入，**不需要知道前驱节点是谁**。这正是二级指针相对"前驱指针+当前指针"方案的胜利：**链表头与中间节点的插入统一成同一句 `*p = iv`**
- ★ `remove(gtid_set)`：记 `next = (*p)->next` → `put_free_interval(*p)` 把被删节点挂回 free list → `set(next)`。**之所以要传 `Gtid_set*`：删除的节点不能泄漏，要还给该集合的复用池**——它是唯一既动链表又动空闲池的地方
- `set/insert/remove` 是 `private` + `friend class Gtid_set`，注释给的两条理由：极易误用；**它们不刷新 `cached_string_format`/`cached_string_length`**，外部直接调用会让字符串长度缓存脏掉

##### Const_interval_iterator 与 Interval_iterator 的关系

**不是继承关系，而是同一个类模板的两个实例化**：`Interval_iterator_base<const Gtid_set *, const Interval *>` 与 `Interval_iterator_base<Gtid_set *, Interval *>`。const 版本只能 `get()/next()/init()`；非 const 版本多了私有 `set/insert/remove`。const 边界由模板参数 + 基类内 const_cast 统一处理。

##### 迭代器失效规则

- 基类注释："The iterator always points to an interval pointer..."——**插入/删除发生在迭代器当前位置时，迭代器不失效**（`*p` 槽位仍有效）
- ★ `Interval_iterator` 是**值类型**：`add_gno_interval` 接收 `Interval_iterator *ivitp`，内部复制一份推进，最后 `*ivitp = ivit` 把最新位置**写回调用方**——这就是"惰性推进"的惯性语义
- 乱序输入时必须 `ivit.init(this, sidno)` 从头再来，否则会插错位置

#### Interval_chunk：C 柔性数组的 C++ 变体

```cpp
  struct Interval_chunk {
    Interval_chunk *next;
    Interval intervals[1];
  };
  static const int CHUNK_GROW_SIZE = 8;
```

```cpp
void Gtid_set::create_new_chunk(int size) {
  int i = 0;
  Interval_chunk *new_chunk = nullptr;
  assert_free_intervals_locked();
  while (i < MAX_NEW_CHUNK_ALLOCATE_TRIES) {
    /* one element is already pre-allocated, so we only add size-1
       elements to the size of the struct. */
    new_chunk = (Interval_chunk *)my_malloc(
        key_memory_Gtid_set_Interval_chunk,
        sizeof(Interval_chunk) + sizeof(Interval) * (size - 1), MYF(MY_WME));
    if (new_chunk != nullptr) break;
    my_sleep(1);                       /* 每次 1 微秒，躲瞬时内存压力 */
    i++;
  }
  if (MAX_NEW_CHUNK_ALLOCATE_TRIES == i || DBUG_EVALUATE_IF(...)) {
    my_safe_printf_stderr("%s", "[Fatal] Out of memory while allocating "
                          "a new chunk of intervals for storing GTIDs.\n");
    _exit(MYSQLD_FAILURE_EXIT);        /* GTID 集不能静默丢数据 */
  }
  new_chunk->next = chunks;
  chunks = new_chunk;
  add_interval_memory_lock_taken(size, new_chunk->intervals);
}
```

- ★ **尺寸计算**：`intervals[1]` 已含 1 个元素，故分配 `sizeof(Interval_chunk) + sizeof(Interval) * (size - 1)`——C89 柔性数组技巧的 C++ 变体（C++ 不允许 `intervals[]` 空柔性数组，`[1]` 是事实约定）
- **为什么不是 `new Interval_chunk` 或 `std::vector`**：`new` 只能按编译期大小分配；`std::vector` 对象构造/析构有开销、空间分散。这里要的是一次 `my_malloc` 拿 size 个**连续 POD Interval 裸对象**（`Interval` 无构造函数），且带 PSI key 可被 Performance Schema 记账
- **OOM 语义**：重试 10 次后 `_exit(MYSQLD_FAILURE_EXIT)`——不能带病运行
- `chunks` 单链表头插，唯一用途是析构释放（`~Gtid_set` 遍历 `my_free`）
- 块内 intervals 通过 `add_interval_memory_lock_taken` 链成 list 挂到 `free_intervals`；`get_free_interval`（头删）/`put_free_interval`（头插）都是 O(1)——★ **稳态下每次 add/remove 一个新 Interval 零 malloc**，只有 free list 空时才按 8 个一批真正分配

#### Free_intervals_lock：懒加锁 RAII

```cpp
  class Free_intervals_lock {
   public:
    /// Create a new lock, but do not acquire it.
    Free_intervals_lock(Gtid_set *_gtid_set) : gtid_set(_gtid_set), locked(false) {}
    void lock_if_not_locked() {
      if (gtid_set->sid_lock && !locked) {
        mysql_mutex_lock(&gtid_set->free_intervals_mutex);
        locked = true;
      }
    }
    void unlock_if_locked() {
      if (gtid_set->sid_lock && locked) {
        mysql_mutex_unlock(&gtid_set->free_intervals_mutex);
        locked = false;
      }
    }
    ~Free_intervals_lock() { unlock_if_locked(); }
   private:
    Gtid_set *gtid_set;
    bool locked;
  };
```

- **状态机**：构造时 `locked=false` **不取锁**；顶层函数栈上创建并一路传指针到底层；只有真正要动 free list 的 `get_free_interval`/`put_free_interval` 才调 `lock_if_not_locked()`——**首次调用才真正加锁**，之后 no-op；析构时条件解锁。纯只读扫描（两区间恰好可合并、无需分配）**全程不碰 mutex**
- ★ **`sid_lock == nullptr` 的零锁路径**：`lock_if_not_locked` 第一个条件直接短路。此时连 `free_intervals_mutex` 都没初始化（`init()` 里 `if (sid_lock) mysql_mutex_init(...)`）。语义：**sid_lock 为空的 Gtid_set 被约定为线程局部**（头文件注释原文 "such Gtid_sets are assumed to be thread-local"）。典型场景：`save_gtids_of_last_binlog_into_table` 里 `Sid_map sid_map(nullptr); Gtid_set logged_gtids_last_binlog(&sid_map, nullptr);`——rotate 落表的局部集合，还预先 `add_interval_memory(64, iv)` 用栈上 64 个 Interval 免分配
- 设计意图：把锁的开销从"每次调用"变成"只在分配发生时"，RAII 保证异常/多出口路径不泄漏锁——一个对象同时实现 lazy acquire 与 scope-bound release

#### Prealloced_array：GTID 代码的"小对象数组"

```cpp
template <typename Element_type, size_t Prealloc>
class Prealloced_array {
  static constexpr bool Has_trivial_destructor =
      std::is_trivially_destructible<Element_type>::value;
  bool using_inline_buffer() const { return m_inline_size >= 0; }
  Element_type *buffer() {
    return using_inline_buffer() ? m_buff : m_ext.m_array_ptr;
  }
```

- **模板参数**：`Element_type` + `Prealloc`（内联预分配个数）。GTID 三处使用：`m_intervals` 是 `Prealloced_array<Interval *, 8>`（**最多 8 个 SID 时链头数组零堆分配**）、`Owned_gtids::sidno_to_hash`、`Sid_map::_sorted`
- **存储布局**：`int m_inline_size` + union{外部 {ptr,size,cap} / 内联 `m_buff[Prealloc]`}。`m_inline_size >= 0` = 用内联缓冲；`-1` = 已切堆。`sizeof(Prealloced_array<void *, 4>) <= 40` 有 static_assert 兜底
- ★ `reserve`：`my_malloc` 新数组 → **placement-new 移动构造**每个旧元素（不是 memcpy，类对象语义正确）→ 非平凡析构才逐元素析构 → 释放旧堆。`emplace_back`：满时**2 倍扩容**，`adjust_size(1)` 只动 size 字段
- ★ `clear()` 只析构元素、`set_size(0)`，**不释放 capacity**——这正是 `Gtid_set::clear()` 语义的底层支撑："不归还任何内存，下次 add 复用"
- **为什么不用 `std::vector`**：① 内联预分配（≤8 SID 零堆分配）② `my_malloc`/PSI key 内存记账（`std::vector` 走全局 `operator new`，P_S 不可见）③ ★ `push_back`/`reserve` **返回 bool（OOM 而非异常）**——MySQL 服务器不用异常，`ensure_sidno` 靠返回值路径报 `ER_OUT_OF_RESOURCES` ④ `Has_trivial_destructor` 模板分支让 `Interval *` 类元素跳过逐元素析构

#### malloc_unordered_map 与 Malloc_allocator

```cpp
template <class Key, class Value, class Hash = std::hash<Key>,
          class KeyEqual = std::equal_to<Key>>
class malloc_unordered_map
    : public std::unordered_map<Key, Value, Hash, KeyEqual,
                                Malloc_allocator<std::pair<const Key, Value>>> {
 public:
  malloc_unordered_map(PSI_memory_key psi_key)
      : std::unordered_map<...>(/*bucket_count=*/10, Hash(), KeyEqual(),
                                Malloc_allocator<>(psi_key)) {}
};
```

- **不是新容器，是换 allocator 的继承包装**：唯一行为差异是分配源（`Malloc_allocator` 核心两行：`my_malloc(...)` / `my_free(p)`，失败抛 `std::bad_alloc` 满足 STL 契约）与 P_S 记账；哈希策略、rehash、迭代器语义与 `std::unordered_map` 完全相同
- `unique_ptr_my_free` = `std::unique_ptr<T, My_free_deleter>`——让 `my_malloc` 出的裸指针获得 RAII，避免节点析构时混用 `delete`

#### add_gtid_text 解析状态机逐行剖析

```cpp
rpl_gno parse_gno(const char **s) {
  char *endp;
  long long ret = my_strtoll(*s, &endp, 0);   // ★ base=0 自动探测：0x 十六进制、前导 0 八进制、否则十进制
  if (ret < 0 || ret >= GNO_END) return -1;
  *s = endp;
  return static_cast<rpl_gno>(ret);
}
```

主状态机核心（`add_gtid_text`）：

```cpp
  // Allocate space for all intervals at once, if nothing is allocated.
  if (chunks == nullptr) {
    // compute number of intervals in text: it is equal to the number of colons
    int n_intervals = 0;
    text = s;
    for (; *s; s++)
      if (*s == ':') n_intervals++;
    lock.lock_if_not_locked();
    create_new_chunk(n_intervals);   // ★ 一次性分配恰好够用的 Interval
    lock.unlock_if_locked();
    s = text;
  }

  while (true) {
    while (*s == ',') { s++; SKIP_WHITESPACE(); }        // 吞逗号（允许空段）
    if (*s == 0) RETURN_OK;                              // 纯逗号串也是合法空集

    if (anonymous != nullptr && strncmp(s, "ANONYMOUS", 9) == 0) {
      *anonymous = true;  s += 9;
    } else {
      rpl_sid sid;
      if (sid.parse(s, binary_log::Uuid::TEXT_LENGTH) != 0) goto parse_error;
      s += binary_log::Uuid::TEXT_LENGTH;                // 严格消费 36 字节
      rpl_sidno sidno = sid_map->add_sid(sid);
      PROPAGATE_REPORTED_ERROR(ensure_sidno(sidno));
      SKIP_WHITESPACE();

      Interval_iterator ivit(this, sidno);
      while (*s == ':') {                                // 内层：区间序列
        s++;
        rpl_gno start = parse_gno(&s);
        if (start <= 0) goto parse_error;                // start 必须 > 0
        SKIP_WHITESPACE();
        rpl_gno end;
        if (*s == '-') {
          s++;
          end = parse_gno(&s);
          if (end < 0) goto parse_error;
          end++;                                         // ★ 闭区间转半开
          SKIP_WHITESPACE();
        } else
          end = start + 1;
        if (end > start) {
          // Use the existing iterator position if the current interval does
          // not begin before it. Otherwise iterate from the beginning.
          Interval *current = ivit.get();
          if (current == nullptr || start < current->start)
            ivit.init(this, sidno);                      // 乱序才回卷
          add_gno_interval(&ivit, start, end, &lock);
        }                                               // end <= start 静默丢弃
      }
    }
    if (*s != ',' && *s != 0) goto parse_error;          // 段落边界校验
  }
  assert(0);

parse_error:
  BINLOG_ERROR(("Malformed Gtid_set specification '%.200s'.", text),
               (ER_MALFORMED_GTID_SET_SPECIFICATION, MYF(0), text));
```

逐段要点：

1. ★ **"先数冒号一次性预分配"就在这里**：`chunks == nullptr` 时先扫全文统计 `':'` 个数（区间数与冒号数相等）→ `create_new_chunk(n_intervals)` **一次大分配替代 N 次按需小分配**，解析过程中 free list 永远不空
2. 外层循环 = SID 序列，内层循环 = 区间序列；`ANONYMOUS` 仅当调用方传了出参才允许
3. `parse_gno` 的 `base=0` 自动探测（`0x` 十六进制 / 前导 `0` 八进制 / 十进制，头文件注释原文 "NUMBER is a decimal, 0xhex, or 0oct number"）
4. ★ `end++` 把文本闭区间 `start-end` 转成内部半开 `[start, end)`；`end <= start` 退化区间**静默丢弃**
5. ★ **迭代器惯性**：`current == nullptr || start < current->start` 才回卷——文本通常升序，`ivit` 贴尾推进，每个 `add_gno_interval` 摊还 O(1)；乱序输入退化为反复从头扫
6. 所有错误统一 `goto parse_error` 报 `ER_MALFORMED_GTID_SET_SPECIFICATION`；`assert(0)` 表明 while(true) 只有 RETURN_OK/goto 两种出路

#### Gtid_set 的拷贝 / 赋值 / 清空语义

- ★ **无拷贝构造、无 `operator=`**（从未声明；隐式浅拷贝会复制裸指针与 `mysql_mutex_t`，属 UB，代码规范是**一律指针传递**）
- 构造函数对空参数的降级：`sid_map == nullptr` 允许（用于"只要集合运算、不要文本"场景），但 `to_string`/`add_gtid_text` 会 `assert(sid_map != nullptr)`；`sid_lock == nullptr` 时**不初始化 free_intervals_mutex**，走线程局部零锁路径
- `clear()`：把每个 SIDNO 的区间链表**尾接**回 free list（先走到 free list 尾再 `set(iv)`，保持分配顺序）→ 链头置空——**零释放、零 malloc，全量复用**
- ★ `clear_set_and_sid_map()` 的注释警告**顺序不可颠倒**：必须先 `clear()` 再 `m_intervals.clear()` 再 `sid_map->clear()`，否则出现 `Gtid_set::get_max_sidno() > Sid_map::get_max_sidno()` 的不变量破坏
- ★ `equals()` 走**结构比较而非字符串**：快路径共享 Sid_map 时逐 SIDNO 双迭代器比对 `start/end`；不同 Sid_map 时按 `get_sorted_sidno` 归并游标逐 UUID 比对——全程不生成任何字符串

#### contains_gtid / get_gtid_count / get_last_gno

```cpp
bool Gtid_set::contains_gtid(rpl_sidno sidno, rpl_gno gno) const {
  if (sidno > get_max_sidno()) return false;
  Const_interval_iterator ivit(this, sidno);
  const Interval *iv;
  while ((iv = ivit.get()) != nullptr) {
    if (gno < iv->start)
      return false;                    // 区间升序，后面的只会更大
    else if (gno < iv->end)
      return true;                     // 半开语义
    ivit.next();
  }
  return false;
}
```

**线性遍历 + 提前剪枝**（区间按 start 升序）。`get_gtid_count` 逐区间累加 `end - start`，`get_last_gno` 也是**整链遍历**（不是 O(1) 取尾），均无缓存字段——高频调用应避免。★ 不存在 `get_last_sidno`；"最大下标"是 `get_max_sidno()`（`m_intervals.size()`），语义是"已开辟空间的 sidno 上限"而非"最后一个有内容的 sidno"。

#### 摊还复杂度分析

- ★ **"每事务 add 一个 GNO"近似 O(1) 的成立条件**：顺序提交的常态下该 SIDNO 只有一个区间且新 GNO 落在尾部——扫描 1 步即与末区间合并（`iv->end++`），**不分配、不锁 free list，O(1) 且无 mutex**
- **迭代器惯性的作用域**：批处理路径（`add_gtid_text`/`add_gtid_set`）升序输入下每个区间定位摊还 O(1)；回卷只在乱序时触发
- **内存侧摊还**：free list 弹出/归还 O(1) 头操作；真实 malloc 只在 free list 空时按 8 个一批发生；`clear()` 后全量复用——稳态"每事务零 malloc"
- **退化场景**：① 单 SIDNO 区间数 k 很大（乱序提交、purge 打洞、clone 恢复碎片）时每事务扫描 O(k)；② `add_gtid_text` 的冒号预分配只在 `chunks == nullptr` 时生效，重复追加退回小 chunk；③ `get_last_gno`/`get_gtid_count`/`contains_gtid` 无缓存

**设计意图总结**：这套实现的一切技艺（link-pointer 迭代器、POD 块分配、懒锁、Prealloced_array）服务于同一个目标——**让热路径（顺序提交事务时的 `executed_gtids` 维护）做到零堆分配、零 mutex、摊还 O(1)**，同时保持 P_S 内存可观测与线程安全断言完备。

### Owned_gtids

```cpp
class Owned_gtids {
  struct Node { rpl_gno gno; my_thread_id owner; };   // 一个 (gno → owner) 记录
  Prealloced_array<malloc_unordered_multimap<rpl_gno, unique_ptr_my_free<Node>> *, 8>
      sidno_to_hash;   // 下标 sidno-1 → 该 sidno 的 gno→owner 哈希
```

★ 结构是**按 sidno 分桶的 gno→owner multimap**（不是常见资料说的"Gtid_hash_map"）。multimap 允许同一 GNO 挂多个 owner——为 **Group Replication 场景**（同一 GTID 可被额外线程"借有"）。

```cpp
enum_return_status Owned_gtids::add_gtid_owner(const Gtid &gtid, my_thread_id owner) {
  Node *n = (Node *)my_malloc(key_memory_Sid_map_Node, sizeof(Node), MYF(MY_WME));
  n->gno = gtid.gno;  n->owner = owner;
  get_hash(gtid.sidno)->emplace(gtid.gno, unique_ptr_my_free<Node>(n));
  RETURN_OK;
}
void Owned_gtids::remove_gtid(const Gtid &gtid, const my_thread_id owner) {
  auto it_range = get_hash(gtid.sidno)->equal_range(gtid.gno);
  for (auto it = it_range.first; it != it_range.second; ++it)
    if (it->second->owner == owner) { get_hash(gtid.sidno)->erase(it); return; }
}
```

`THD::OWNED_SIDNO_ANONYMOUS == -2`、`OWNED_SIDNO_GTID_SET == -1` 是 `thd->owned_gtid.sidno` 的**保留哨兵值**（真实 sidno ≥1）：`-2` 表示线程正"拥有"一个匿名事务（只递增原子计数，**不进** Owned_gtids）；`-1` 为 `gtid_next_list` 多 GTID 预留（8.0.39 中对应代码被 `#ifdef HAVE_GTID_NEXT_LIST` 摘除，实际不使用）。`sidno==0` 表示不拥有任何事务。

### Gtid_state 锁体系

```cpp
class Gtid_state {
  mutable Checkable_rwlock *sid_lock;      // 即 global_sid_lock：保护 SID 集合的大小变化
  mutable Sid_map *sid_map;
  Mutex_cond_array sid_locks;              // 每 sidno 一对 {mutex, cond}
  Gtid_set lost_gtids, executed_gtids, gtids_only_in_table, previous_gtids_logged;
  Owned_gtids owned_gtids;
  rpl_sidno server_sidno;  rpl_gno next_free_gno;
  Prealloced_array<bool, 8> commit_group_sidnos;      // 组提交 sidno 锁位图
  std::atomic<int32> atomic_anonymous_gtid_count;
  ...
};
```

- **`sid_lock`**：保护"SID 集合大小变化"——sidno 扩容、Sid_map 增删、以及需要全量一致视图的操作（`to_string`/`equals`/`add_gtid_set` 要求 wrlock）。读锁 + 单个 sidno mutex 即可安全读写**某一个 SID** 的数据；多把 sidno 锁必须**按 sidno 升序**获取（防死锁）
- **`sid_locks`（`Mutex_cond_array`）**：`WAIT_FOR_EXECUTED_GTID_SET` 就等在 per-sidno 的 cond 上。`wait_for_sidno` 语义：调用者持读锁 + sidno mutex，函数内部**先释放 global_sid_lock 再 cond_wait**（返回时两者都不持有）——避免持读锁睡眠阻塞 `RESET MASTER`
- ★ 等待协议防 TOCTOU：`wait_for_gtid_set` 外层 `while (!verified)` 反复拷贝待等集合，对每个 sidno `lock_sidno → remove_intervals_for_sidno(&executed_gtids)`，仍未清空才 `wait_for_sidno`；**只要发生过一次等待就整轮重跑**——防御等待期间 `RESET MASTER` 清掉 executed 导致的假完成
- `update_commit_group(first_thd)` 沿 `thd->next_to_commit` 链遍历，先按升序收集并锁住本组涉及的全部 sidno（`commit_group_sidnos` 位图防重复），逐线程 remove + `executed_gtids._add_gtid`，最后统一 broadcast + unlock——**一次性锁住整个组**避免逐线程加解锁

| 决策 | 实现方式 | 动机 / 代价 |
|---|---|---|
| 区间用半开 `[start,end)` | `end` 存"第一个不在区间内的 GNO" | 合并/分裂/交集无 ±1 边界调整 |
| Interval 按块分配 + 空闲链表 | 8 个/块 + `get/put_free_interval` | 避免每事务 new/delete；OOM 直接 `_exit`（无 lossy 降级） |
| sidno 只增不复用 | `get_max_sidno()+1` | sidno 是稳定"指针"，引用永不悬空；代价是长期运行单调增长 |
| `add_sid` 读锁→写锁升级 | 升级后二次 hash 查找 | 常见路径（SID 已存在）零写锁开销 |
| 扩容联动 | `add_node` 写锁内回调 `ensure_sidno()` | 所有 sidno 索引容器原子同步扩容 |
| 输出确定性 | `_sorted` 按 UUID 字节序 + `String_format` | 可复现的文本/哈希 |
| `intersection` 用 `A-(A-B)` | 复用 remove 算法 | 代码量最小；注释自认比直接求交略慢 |
| owned 用 multimap 允许多 owner | `equal_range` 按 owner 精确删除 | 兼容 Group Replication 共享所有权 |
| 等待协议 | wait 前释放 sid_lock；等待即整轮重验 | 避免持读锁睡眠 + 消除 TOCTOU 假完成 |
| 组提交一次锁全部 sidno | 位图 + 升序加锁 | 摊销整组加解锁；升序避免死锁 |

### gtids_only_in_table

`gtids_only_in_table` 表示只在 `mysql.gtid_executed` 表中、不在 binlog 中的 GTID。这种情况出现在从库开启 binlog 但关闭 `log_replica_updates` 时——事务通过引擎提交刷入 GTID 表，但不写 binlog，所以 binlog 中没有对应的 `Gtid_log_event`。

---

## GTID 相关系统变量：读写语义

> 变量描述符框架（`sys_var` 类树、SET 三趟、来源追踪）见 [`../infra/variables.md`](../infra/variables.md)。本节只讲这几个变量的**业务语义**：谁在何时能改、check/update 钩子做什么、背后维护什么状态。变量名（如 `fix_gtid_mode`）均已 grep 核实——8.0.39 中不存在的会明确标注。

类树总览：

```
sys_var
├── Sys_var_typelib → Sys_var_enum → Sys_var_gtid_mode            gtid_mode（GLOBAL）
├── Sys_var_multi_enum → Sys_var_enforce_gtid_consistency         enforce_gtid_consistency（GLOBAL）
├── Sys_var_gtid_next                                             gtid_next（SESSION_ONLY，直接继承 sys_var）
├── Sys_var_gtid_purged                                           gtid_purged（GLOBAL，非 READ_ONLY！）
└── Sys_var_charptr_func（构造即 READ_ONLY NON_PERSIST）
    ├── Sys_var_gtid_executed                                     gtid_executed（GLOBAL 只读）
    └── Sys_var_gtid_owned                                        gtid_owned（SESSION 只读）
```

### gtid_mode

**只有 4 个取值，不是 8 个**（`Gtid_mode::value_type`）：`OFF=0 / OFF_PERMISSIVE=1 / ON_PERMISSIVE=2 / ON=3`，默认 OFF。名字数组是 `Gtid_mode::names[]`（`rpl_gtid_mode.cc`）——**不存在** `gtid_mode_typelib`/`gtid_mode_names` 符号。

运行时修改的钩子是 `Sys_var_gtid_mode::global_update`（**不存在** `fix_gtid_mode`/`update_gtid_mode`，那是 5.7 的历史名字）。它的核心是**四把锁 + 系列约束检查 + 落地**：

```cpp
// 锁序：Gtid_mode::lock（trywrlock，抢不到直接报错不阻塞）
//      → channel_map.wrlock → mysql_bin_log.get_log_lock → global_sid_lock->wrlock
if (mysqld_server_started && abs((int)new_gtid_mode - (int)old_gtid_mode) > 1) {
  my_error(ER_GTID_MODE_CAN_ONLY_CHANGE_ONE_STEP_AT_A_TIME, MYF(0));  // 一次只能走一步
}
...
if (new_gtid_mode == Gtid_mode::ON && get_gtid_consistency_mode() != GTID_CONSISTENCY_MODE_ON) {
  my_error(ER_CANT_SET_GTID_MODE, MYF(0), "ON", "ENFORCE_GTID_CONSISTENCY is not ON");
}
...
// 落地
global_var(ulong) = new_gtid_mode;      // 写背板 Gtid_mode::sysvar_mode（供 SHOW/持久化）
global_gtid_mode.set(new_gtid_mode);    // 写原子值（全服务器其他代码读这个）
LogErr(SYSTEM_LEVEL, ER_CHANGED_GTID_MODE, ...);
mysql_bin_log.rotate(true, &dont_care); // 强制轮转 binlog，让新 Previous_gtids 反映新状态
```

约束清单：① **一次一步**（OFF→ON 必须经 OFF_PERMISSIVE→ON_PERMISSIVE，启动期 `mysqld_server_started==false` 免检，所以配置文件可直接设）；② ON 前要求 `enforce_gtid_consistency=ON`（双向联动，另一方向见下节）；③ ON 前要求无进行中的匿名事务（`get_anonymous_ownership_count()==0`）与无 AUTOMATIC 的 GTID-violating 事务；④ 设 OFF 前要求 `owned_gtids` 为空、无 AUTO_POSITION 通道、无 `WAIT_FOR_EXECUTED_GTID_SET` 等待者；⑤ 从 ON 往下改时无 `ASSIGN_GTIDS_TO_ANONYMOUS_TRANSACTIONS=LOCAL/UUID`、`GTID_ONLY`、`source_connection_auto_failover` 通道；⑥ GR 运行中禁止改非 ON。

启动路径不走钩子：`gtid_server_init()` 直接 `global_gtid_mode.set((value_type)Gtid_mode::sysvar_mode)`。

### enforce_gtid_consistency

值域 `OFF/ON/WARN`（别名 `FALSE/TRUE`），**8.0 默认 ON**。修改钩子 `Sys_var_enforce_gtid_consistency::global_update`（**不存在** `check_enforce_gtid_consistency`/`assert_enforce_gtid_consistency`，也**不扫描 binlog**）：

```cpp
global_sid_lock->wrlock();
// 与 gtid_mode 联动：gtid_mode==ON 时禁改非 ON
if (new_mode != GTID_CONSISTENCY_MODE_ON && gtid_mode == Gtid_mode::ON) {
  my_error(ER_GTID_MODE_ON_REQUIRES_ENFORCE_GTID_CONSISTENCY_ON, MYF(0)); goto err;
}
// 有进行中的 GTID-violating 事务（automatic + anonymous 两个计数）时：
//   目标是 ON → 报错 ER_CANT_ENFORCE_GTID_CONSISTENCY_WITH_ONGOING_GTID_VIOLATING_TX
//   OFF→WARN → 只警告
global_var(ulong) = new_mode;   // 写 _gtid_consistency_mode
LogErr(INFORMATION_LEVEL, ER_CHANGED_ENFORCE_GTID_CONSISTENCY, ...);
```

注意落地**没有** binlog rotate（与 gtid_mode 不同）。变量本身只是阈值状态——语句执行时由 `binlog.cc` 的检查点读 `get_gtid_consistency_mode()` 决定报错或警告（消费端）。

### gtid_next

SESSION-only、`NO_CMD_LINE`、默认 `"AUTOMATIC"`。三种用户可见输入形态（内部 `enum_gtid_type` 另有 `UNDEFINED/NOT_YET_DETERMINED/PRE_GENERATE` 三个内部态）：

| 输入 | 内部 | set_gtid_next 做什么 |
|---|---|---|
| `AUTOMATIC` | `AUTOMATIC_GTID=0` | 仅 `set_automatic()`，不获取任何所有权；提交时按 gtid_mode 决定生成 GTID 或匿名 |
| `ANONYMOUS` | `ANONYMOUS_GTID` | 要求 `gtid_mode != ON`；置 `owned_gtid.sidno = OWNED_SIDNO_ANONYMOUS(-2)` + `acquire_anonymous_ownership()` |
| `UUID:N` | `ASSIGNED_GTID` | 要求 `gtid_mode != OFF`；已执行则直接接受（语句稍后被跳过）；被占则 `wait_for_gtid` 阻塞；否则 `gtid_state->acquire_ownership()` 写 Owned_gtids + `thd->owned_gtid` |

`check_gtid_next`（真实存在）三重校验：存储函数/触发器内禁止（`ER_VARIABLE_NOT_SETTABLE_IN_SF_OR_TRIGGER`）、**多语句事务进行中禁止**（`ER_VARIABLE_NOT_SETTABLE_IN_TRANSACTION`，XA PREPARED 例外）、权限 `SESSION_VARIABLES_ADMIN`/`SYSTEM_VARIABLES_ADMIN`/`SUPER`/`REPLICATION_APPLIER`。

**关键语义：提交后 gtid_next 不是重置回 AUTOMATIC，而是进入 `UNDEFINED_GTID`**（`update_gtids_impl_own_gtid` 对 ASSIGNED 类型 `set_undefined()`），下一条语句被 `gtid_pre_statement_checks` 报 `ER_GTID_NEXT_TYPE_UNDEFINED_GTID` 强制"一个显式 GTID 只用于一个事务"；回滚/连接关闭的兜底才 `set_automatic()`（`sql_base.cc`，保证 DROP TEMPORARY TABLE 能生成自己的 GTID）。

### gtid_purged

**不是 READ_ONLY**——注册 flag 是 `NON_PERSIST GLOBAL_VAR(gtid_purged)`，无 READ_ONLY 位。所以运行时 `SET @@GLOBAL.gtid_purged` 能通过 resolve 的只读检查（权限仍要 SUPER/SYSTEM_VARIABLES_ADMIN），"只读型语义"由业务检查表达：

```cpp
// Gtid_state::add_lost_gtids —— 三个集合约束
if (!starts_with_plus) {
  if (!lost_gtids->is_subset(gtid_set))
    my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "the new value must be a superset of the old value");
  gtid_set->remove_gtid_set(lost_gtids);       // 先剥掉已 purged 再查交集
}
if (executed_gtids.is_intersection_nonempty(gtid_set))
  my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "must not overlap with @@GLOBAL.GTID_EXECUTED");
if (owned_gtids.is_intersection_nonempty(gtid_set))
  my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "must not overlap with @@GLOBAL.GTID_OWNED");
// 落地：写 mysql.gtid_executed 表 → gtids_only_in_table/lost_gtids/executed_gtids 三集合都加
//      → broadcast_sidnos 唤醒 wait_for_gtid 等待者
```

即"binlog 必须未开/executed 为空"的直觉说法，源码里是**三集合交集检查**：新增部分与 executed（去掉 lost）不相交、与 owned 不相交、且新值必须是旧值超集（只增不减）。`check_gtid_purged` 另拒 GR 运行中、拒 `SET DEFAULT`（该变量无默认值，`ER_NO_DEFAULT`）。启动初始化**不走**这个 sys_var——`mysql_bin_log.init_gtid_sets()` 直接算好集合改 `Gtid_state::lost_gtids`（mysqldump 的 `--set-gtid-purged` 走运行时 SET 路径）。

#### gtid_purged 在复制中的角色：与 exclude 集合的关系

`gtid_purged` 不是孤立的运维工具，它**直接决定从库能否重连**。从库用 GTID auto-positioning 重连主库时，主库 dump 线程做两道校验（rpl_binlog_sender.cc:877-937）：

1. **从库不能比主库"知道得更多"**：从库的 exclude 集合（它宣称已执行的 GTID）必须是主库 `executed_gtids ∪ owned_gtids` 的子集，否则报 `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`（从库执行了主库没有的事务，数据分叉）

2. **主库 purge 掉的必须是从库已有的**（★ 这就是 gtid_purged 的复制语义）：

```cpp
// rpl_binlog_sender.cc:925 —— 主库 lost_gtids（= gtid_purged）必须 ⊆ 从库 exclude 集合
if (!gtid_state->get_lost_gtids()->is_subset(m_exclude_gtid)) {
    mysql_bin_log.report_missing_purged_gtids(m_exclude_gtid, errmsg);
    // → "Cannot replicate because the master purged required binary logs.
    //    Replicate the missing transactions from elsewhere, or provision a new
    //    replica from backup."（ER_RPL_SOURCE_HAS_PURGED_REQUIRED_GTIDS 家族）
}
```

**含义**：如果主库 purge 掉了某个 GTID 的 binlog，而从库还没执行过这个 GTID，那么**从库永远无法补上这个事务**——主库已经没有它的 binlog 了。此时复制只能断开，唯一的出路是"从别的实例补数据，或用备份重建从库"。这是生产上 gtid_purged 最危险的场景：**purge binlog 前必须确认所有从库都已执行到 purge 点之后**。

#### gtid_purged 的典型使用场景

| 场景 | 操作 | 语义 |
|---|---|---|
| mysqldump 备份 | `mysqldump --set-gtid-purged=AUTO`（默认） | 备份文件里带 `SET @@GLOBAL.GTID_PURGED='...'`，从备份恢复的实例"认领"备份时刻的 executed 集合 |
| 搭建新从库（备份+binlog 定位） | 恢复备份后手动 `SET GTID_PURGED` | 告诉新从库"备份里已包含这些 GTID 的数据"，之后的复制从 purge 点继续 |
| 修复 executed 集合 | 手动设置 purged | 当 executed 集合因 binlog 丢失而不完整时，把丢失部分标记为 purged（等价于"我承认这些事务的数据已存在/不需要了"） |
| 清空复制状态 | `RESET MASTER` | 清空 binlog + gtid_purged + gtid_executed（从库上等价 `RESET REPLICA` 清通道） |

### GTID 复制协议：exclude 集合（COM_BINLOG_DUMP_GTID）

> 上节从 gtid_purged 的视角引出了 exclude 集合。本节从 dump 协议视角完整讲它——这是"从库怎么告诉主库该发什么"的机制，也是 GTID auto-positioning 的核心。源码都在 `sql/rpl_binlog_sender.cc`。

#### exclude 集合是什么

从库 IO thread 用 `COM_BINLOG_DUMP_GTID` 命令连接主库时，把**自己已执行的 GTID 集合**作为 exclude 集合发给主库，语义是"**这些 GTID 我都有了，跳过它们**"。主库 dump 线程把它存进 `Binlog_sender::m_exclude_gtid`（构造参数 `exclude_gtids`，rpl_binlog_sender.cc:235）：

```cpp
m_using_gtid_protocol(exclude_gtids != nullptr),   // 非空 = 走 GTID 协议
m_check_previous_gtid_event(exclude_gtids != nullptr),
```

从库发的是什么集合？取决于场景：

- 常规重连：`Executed_Gtid_Set`（从库已执行的）
- IO thread 接收中断后恢复：`Retrieved_Gtid_Set`（已收到但可能还没执行完的）∪ executed——防止"已收到未执行"的事务被主库重发

#### 主库侧：skip_event 跳过已执行事务

`Binlog_sender::send_events` 主循环（rpl_binlog_sender.cc:618）对每个事件：

```cpp
if (m_exclude_gtid &&
    (in_exclude_group = skip_event(event_ptr, in_exclude_group))) {
    // 跳过：定期补发 heartbeat 让从库推进 master_log_pos
    ...
} else {
    send_packet();   // 发送
}
```

`skip_event`（rpl_binlog_sender.cc:741）的状态机：

```cpp
switch (event_type) {
  case GTID_LOG_EVENT:
    // 解析 GTID → contains_gtid 判断是否在 exclude 集合中
    gtid.sidno = gtid_ev.get_sidno(m_exclude_gtid->get_sid_map());
    gtid.gno = gtid_ev.get_gno();
    return m_exclude_gtid->contains_gtid(gtid);   // 在集合中 → 整个事务跳过
  case ROTATE_EVENT:
    return false;                                  // rotate 从不跳过
}
return in_exclude_group;   // 事务中间的事件跟随 GTID 事件的判定
```

**跳过是"整事务"粒度**：一个 GTID 在 exclude 集合中，则该事务的 GTID event 到 Xid/Commit event 之间的全部事件（包括 Table_map、Rows）都被跳过（`in_exclude_group` 状态传播）。跳过的窗口里定期发 heartbeat（`exclude_group_end_pos`），让从库知道"主库跳过到了哪个位置"，否则从库的 `master_log_pos` 会停滞。

#### 发送起点的选择：find_first_log_not_in_gtid_set

主库不会从头扫所有 binlog，而是找**第一个"Previous_gtids 不被 exclude 完全覆盖"的文件**作为起点（rpl_binlog_sender.cc:933）：

```cpp
if (mysql_bin_log.find_first_log_not_in_gtid_set(
        index_entry_name, m_exclude_gtid, &first_gtid, errmsg)) { ... }
```

逻辑：binlog 文件的 `Previous_gtids_log_event` 记录"本文件之前所有事务的 GTID"。若某文件的 Previous_gtids ⊆ exclude（从库全都有），则该文件里所有事务从库都已执行，**整个文件跳过**；找到第一个 Previous_gtids ⊄ exclude 的文件，从那里开始逐事件 skip。这样主库只需读 index 文件 + 各文件头部，不必扫描全部 binlog 内容。

#### 三道校验（check_sender_capabilities 路径）

连接建立后（rpl_binlog_sender.cc:877-937）：

1. **从库 ⊆ 主库 executed ∪ owned**：`m_exclude_gtid->is_subset_for_sid(executed_and_owned)` 不满足 → `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`。从库执行了主库没有的 GTID = 数据分叉（可能从库曾被提升过或手动注入过事务）

2. **主库 purged ⊆ 从库 exclude**：见上节。不满足 → "master purged required binary logs" 错误。从库缺的事务主库已经物理删除，复制无解

3. **发送起点确定**：`find_first_log_not_in_gtid_set` 定位起点文件后 `m_check_previous_gtid_event = false`（该文件必有 Previous_gtids，不必再检查）

若跳过的第一个事务正好是某 binlog 文件的第一个事务，FD event 的 `created` 字段要清 0（rpl_binlog_sender.cc:946-953）——避免从库误以为新连接而清理临时表。

#### 四个集合的角色对比

| 集合 | 维护方 | 语义 | 与 exclude 的关系 |
|---|---|---|---|
| `executed_gtids` | 主库 | 主库已执行的 | exclude 必须是它的子集（校验 1） |
| `owned_gtids` | 主库 | 主库正在执行的（未提交） | 并入 executed 一起校验 |
| `lost_gtids`（=gtid_purged） | 主库 | 主库已执行但 binlog 已删的 | 必须是 exclude 的子集（校验 2） |
| **exclude 集合** | **从库** | **从库宣称已执行的** | 主库据此跳过事件、选择发送起点 |

#### 与 gtid_executed 的区别

- `gtid_executed`（从库自己的）：从库本地已执行的事务——它是**被动的**（从库回放后自然增长）
- exclude 集合：从库**主动发给主库**的声明——本质内容就是 executed（或 retrieved ∪ executed），但角色是"请求过滤器"而非"状态记录"

一句话：**exclude 集合是从库把自己的 executed 集合"广播"给主库，换取主库只发增量**；gtid_purged 是主库单方面删除 binlog 的声明，两者在重连时的子集校验关系决定了复制能否继续。

### gtid_executed / gtid_owned

真 READ_ONLY（`Sys_var_charptr_func` 构造带 `READ_ONLY NON_PERSIST`），SET 在 resolve 阶段直接被拒。读源：global 读 `Gtid_state::executed_gtids`/`owned_gtids`（持 `global_sid_lock` wrlock 后 to_string）；session 读 `thd->owned_gtid`（`sidno==0` 显示空、`-2` 显示 "ANONYMOUS"、`-1` 读 `thd->owned_gtid_set`）。

**澄清：`gtid_current_pos` 在 MySQL 8.0.39 中不存在**（全仓库 grep 0 匹配）——它是 MariaDB 的变量，别当 MySQL 特性。

### binlog_gtid_simple_recovery / session_track_gtids / gtid_executed_compression_period

- **`binlog_gtid_simple_recovery` 默认是 true（ON），不是 false**；READ_ONLY（只能命令行/配置文件）。控制 `MYSQL_BIN_LOG::init_gtid_sets` 两处提前终止：反向扫描若最新 binlog 无任何 GTID 事件（`NO_GTIDS`）直接断定 executed/purged 为空；正向扫描只读第一个 binlog 的 `Previous_gtids_log_event` 就确定 purged。代价（注释明说）：旧 5.7.5 前 binlog + 混用 gtid_mode 的场景可能算出错误集合且不会自愈。
- **`session_track_gtids`**（8.0.26+）：`OFF/OWN_GTID/ALL_GTIDS` 三值，SESSION 作用域，默认 OFF。`on_update` 钩子调 `Session_gtids_tracker::update` 注册/注销 `Session_consistency_gtids_ctx` 监听；OWN_GTID 收集本会话刚提交的 `thd->owned_gtid`，ALL_GTIDS 提交后快照整个 `executed_gtids`；经 OK 包的 `SESSION_TRACK_GTIDS` 类型实体上报客户端。
- **`gtid_executed_compression_period`**：默认 0（8.0.23 起，之前 1000），控制 mysql.gtid_executed 表压缩线程 `compress_gtid_table` 的触发周期。**8.0.39 实测：该变量除注册/定义外没有任何代码读取**（`m_atomic_count` 无递增点），注释所称"按 period 计数触发"的路径已退化——压缩实际只由 binlog rotate 路径触发。变量仍在纯为向后兼容。

---

## GNO 分配算法

GNO 的自动分配由 `get_automatic_gno`（rpl_gtid_state.cc:413）完成。算法在 `executed_gtids`（已执行 GTID 集合）的间隙和 `owned_gtids`（已被占有但未提交的 GTID 集合）中寻找空闲的 GNO：

1. 从 `next_free_gno` 开始扫描
2. 跳过 `executed_gtids` 中已有的 GNO（已执行的不能重复分配）
3. 跳过 `owned_gtids` 中已有的 GNO（已被其他事务占有但未提交的不能分配）
4. 找到第一个空闲的 GNO，分配给当前事务

分配前事务需要获取 GNO 的 ownership（通过 `gtid_state->generate_automatic_gtid`），分配后事务持有该 GTID 的所有权直到 commit 或 rollback。

---

## GTID 生命周期

### 服务器启动与初始化

```
Gtid_state::init()
  → sid_map->add_sid(server_uuid)       // 注册本server UUID，得到server_sidno
  → next_free_gno = 1
  → read_gtid_executed_from_table()     // 从mysql.gtid_executed表加载executed_gtids
```

同时 InnoDB 从 undo log 恢复未刷表的 GTID：

```
trx_rseg_persist_gtid()
  → 遍历 rollback segment history list
  → 对 trx_no >= gtid_trx_no 的 undo log:
    → trx_undo_gtid_read_and_persist()  // 从undo header提取GTID
    → Clone_persist_gtid::add()         // 加入内存list
  → Clone_persist_gtid::flush_gtids()   // 恢复阶段首次刷表
  → m_thread_active = true              // 后台线程就绪
```

### 事务执行前——GTID 一致性检查

`gtid_pre_statement_checks` 检查：

- `gtid_next.type == AUTOMATIC` 时，当前语句是否违反 GTID 一致性
- `gtid_next.type == ASSIGNED` 时，GTID 是否已执行（executed_gtids）或被拥有（owned_gtids）

### Flush Stage——GTID 分配与写入 binlog

在 `process_flush_stage_queue`（binlog.cc:8471）中，leader 遍历 group 中的每个 THD：

```
assign_automatic_gtids_to_flush_group(first_seen)
  → 对每个 THD:
    → generate_automatic_gtid(head, ...)
      → sid_lock->rdlock()              // 全局读锁
      → lock_sidno(server_sidno)        // per-sidno锁
      → get_automatic_gno(sidno)        // 找空闲GNO
        → 从 next_free_gno 开始
        → 在 executed_gtids 区间间隙中找
        → 检查 owned_gtids 是否已被占用
      → acquire_ownership(thd, gtid)
        → owned_gtids.add_gtid_owner(gtid, thread_id)
        → thd->owned_gtid = {sidno, gno}
      → next_free_gno = gno + 1
```

随后 `Gtid_log_event` 写入 binlog：

```
flush_thread_caches(head)
  → binlog_cache_mngr::flush
    → binlog_cache_data::flush
      → Transaction_dependency_tracker::step()     // 分配 sequence_number
      → Binlog_event_writer 构造
      → MYSQL_BIN_LOG::write_transaction
        → Transaction_dependency_tracker::get_dependency()  // 确定last_committed
        → Gtid_log_event 构造
          → spec.set(thd->owned_gtid)    // GTID = (sidno, gno)
          → 携带 last_committed, sequence_number, transaction_length
        → MYSQL_BIN_LOG::write_cache     // 写入binlog文件
```

### Sync Stage——fsync 持久化

```
sync_binlog_file(false)
  → m_binlog_file->sync()   // fsync(binlog_fd)
  → GTID随binlog文件持久化到磁盘
```

### Commit Stage——GTID 外部化

```
process_commit_stage_queue(thd, commit_queue)
  → ha_commit_low(head)                    // 引擎层提交
  → gtid_state->update_commit_group(first)
    → global_sid_lock->rdlock()
    → update_gtids_impl_lock_sidnos(first)
    → 对每个 THD:
      → update_gtids_impl_own_gtid(thd, is_commit=true)
        → owned_gtids.remove_gtid(thd->owned_gtid)   // 释放所有权
        → executed_gtids._add_gtid(thd->owned_gtid)  // 加入已执行集合
        → thd->clear_owned_gtids()                   // 清理THD
        → gtid_next.set_undefined()                  // 重置

  → signal_done(final_queue)               // 唤醒所有follower
```

### InnoDB GTID 持久化（异步刷表）

```
事务InnoDB commit时:
  → Clone_persist_gtid::get_gtid_info(trx, gtid_desc)
    → 从 thd->owned_gtid 提取GTID字符串
  → Clone_persist_gtid::add(gtid_desc)
    → 加入 active_list (内存)

后台线程 periodic_write():
  → sleep(1秒)
  → flush_gtids()
    → switch_active_list()           // 切换双buffer
    → write_to_table()
      → 构造 Gtid_set
      → gtid_table_persistor->save()  // 写mysql.gtid_executed表
    → update_gtid_trx_no(oldest_trx_no)
      → trx_sys_persist_gtid_num()   // 持久化到系统表空间
    → check_compress() → compress()  // 定期压缩表
```

### Binlog Rotate 时的持久化

```
save_gtids_of_last_binlog_into_table()
  → logged_gtids_last_binlog = executed_gtids
                                - previous_gtids_logged
                                - gtids_only_in_table
  → previous_gtids_logged += logged_gtids_last_binlog
  → save(logged_gtids_last_binlog)   // 写gtid_executed表

新binlog文件开头:
  → Previous_gtids_log_event(executed_gtids)
    → 编码executed_gtids为二进制写入
```

### 崩溃恢复：源码级展开

> binlog.md「GTID 与 binlog 的持久化交互」章有收敛流程图；本节给**源码级**展开（两趟扫描的方向与判据、表读取的两档错误、purged 的只能增长约束）。InnoDB 侧（undo 扫描 + `trx_rseg_persist_gtid`）见本文「Clone_persist_gtid」节。

**恢复时序总览**：

```
mysqld 启动
  ├─ gtid_state->init()                    ① server_uuid → sidno（先于一切）
  ├─ gtid_state->read_gtid_executed_from_table()   ② 全表扫描 mysql.gtid_executed
  │    └─ fetch_gtids(): ha_rnd_next 逐行 → encode "uuid:s-e" → add_gtid_text
  │        → executed_gtids（表不存在返回 1 容忍；-1 才 abort）
  └─ mysql_bin_log.init_gtid_sets(&gtids_in_binlog, &purged_gtids_from_binlog, …)
        ├─ 剔除启动时刚新建的空 binlog（is_server_starting → pop_back）
        ├─ 第一趟【反向】：从最新往最旧扫，找"最后一个含 PREVIOUS_GTIDS 的文件"
        │    → 其 PREVIOUS_GTIDS + 全部 Gtid_log_event 并入 gtids_in_binlog；
        │      仅当一直扫到最旧文件才把该文件 PREV 喂给 purged_gtids_from_binlog
        └─ 第二趟【正向】（仅当没扫到最旧文件）：找"第一个同时含
             PREVIOUS_GTIDS + Gtid_log_event 的文件"，其 PREV 并入 purged
             （5.6 升级场景：SET GTID_PURGED 靠轮转 binlog 生效，藏在旧文件头部）
  └─ 集合收敛（global_sid_lock->wrlock 下）：
       gtids_in_binlog_not_in_table = gtids_in_binlog − executed_gtids → 补写表
       executed_gtids ∪= 差集；gtids_only_in_table = executed − gtids_in_binlog
       lost_gtids = gtids_only_in_table ∪ purged_gtids_from_binlog
       → 补写 Previous_gtids_log_event 到当前空 binlog 头部并 sync
```

**真实签名核实**：输出参数只有两个 GTID 集合，**不存在 `first_gtid`/`last_gtid` 输出参数**（`first_gtid` 是函数内部局部变量）：

```cpp
bool MYSQL_BIN_LOG::init_gtid_sets(Gtid_set *all_gtids, Gtid_set *lost_gtids,
                                   bool verify_checksum, bool need_lock,
                                   Transaction_boundary_parser *trx_parser,
                                   Gtid_monitoring_info *partial_trx,
                                   bool is_server_starting)
```

`trx_parser`/`partial_trx` 只在恢复 **relay log**（IO 线程重建 `Retrieved_Gtid_Set`）时使用，启动 binlog 场景传 `nullptr`。

#### 第一趟：反向遍历（找最新 GTID 状态）

```cpp
rit = filename_list.rbegin();
bool can_stop_reading = false;
reached_first_file = (rit == filename_list.rend());
while (!can_stop_reading && !reached_first_file) {
    const char *filename = rit->c_str();  rit++;
    reached_first_file = (rit == filename_list.rend());
    switch (read_gtids_from_binlog(
        filename, all_gtids,
        reached_first_file ? lost_gtids : nullptr,   // ★ 只有最旧文件才喂 lost_gtids
        nullptr /* first_gtid */, sid_map, verify_checksum, is_relay_log)) {
      case ERROR:            error = 1; goto end;
      case GOT_GTIDS:        can_stop_reading = true; break;
      case GOT_PREVIOUS_GTIDS:
        if (!is_relay_log) can_stop_reading = true;  // binlog 见 PREV 即可停
        break;
      case NO_GTIDS:
        /* simple_recovery 且启动时：最后一个文件无 GTID → 直接收工 */
        if (binlog_gtid_simple_recovery && is_server_starting && !is_relay_log) {
            assert(all_gtids->is_empty()); assert(lost_gtids->is_empty());
            goto end;
        }
        [[fallthrough]];
      case TRUNCATED: break;
    }
}
```

★ 为什么"只有最旧文件"的 PREVIOUS_GTIDS 才进 `lost_gtids`：PREVIOUS_GTIDS 语义是"本文件之前全部 binlog 的 GTID 集"，现存最旧文件的 PREVIOUS_GTIDS 恰等于"**已被 purge 掉的那些 binlog 的 GTID**"。若反向在中间文件就停（没扫到最旧），`purged_gtids_from_binlog` 为空，必须靠第二趟补齐。

#### 第二趟：正向遍历（仅 binlog，且未扫到最旧文件时）

```cpp
if (lost_gtids != nullptr && !reached_first_file) {
    /* 5.6 通过轮转 binlog 设置 GTID_PURGED：
       N 无 PREV、N+1 空 PREV+ROTATE、N+2 PREV=purged集+GTID；
       反向会在 N+2 停而拿不到 purged 集，故正向找第一个
       "PREVIOUS_GTIDS + Gtid_log_event"的文件 */
    for (it = filename_list.begin(); it != filename_list.end(); it++) {
        Gtid first_gtid = {0, 0};
        switch (read_gtids_from_binlog(
            filename, nullptr, lost_gtids,
            binlog_gtid_simple_recovery ? nullptr : &first_gtid,  // ★ 关键差异
            sid_map, verify_checksum, is_relay_log)) {
          case ERROR:     error = 1; [[fallthrough]];
          case GOT_GTIDS: goto end;          // 找到目标文件，其 PREV 已入 lost
          case NO_GTIDS:
          case GOT_PREVIOUS_GTIDS:
            if (binlog_gtid_simple_recovery) goto end;  // 只读第一个文件就停
            [[fallthrough]];                            // 否则继续扫下一个
          case TRUNCATED: break;
        }
    }
}
```

★ **`binlog_gtid_simple_recovery` 开/关的路径差异**（默认 ON，READ_ONLY）：

- **ON**：传 `first_gtid=nullptr`，函数读完 PREVIOUS_GTIDS 即返回 `GOT_PREVIOUS_GTIDS`，外层直接 `goto end`——**全程最多读两个文件的头部**
- **OFF**：必须传 `&first_gtid`，否则函数读完 PREV 就提前返回、永远发现不了后面的 GTID 事件；传了之后继续读到第一个 `Gtid_log_event` 才返回。没有 GTID 的老文件（5.7.5 之前）逐个返回 `NO_GTIDS`，外层 `fallthrough` 继续扫下一个——**这就是"关了会正扫全部 binlog"的原因**：必须找到历史上第一个含 GTID 的文件才能定 purged 集。sys_vars 注释同时警告：ON 时两种 5.6 老场景下 purged/executed 可能算错且"错一次就永远错"

#### read_gtids_from_binlog：单文件扫描

返回 `GOT_GTIDS`（含 PREV+GTID）/ `GOT_PREVIOUS_GTIDS`（仅 PREV）/ `NO_GTIDS` / `ERROR` / `TRUNCATED`。核心判据：

```cpp
      case binary_log::GTID_LOG_EVENT: {
        if (ret != GOT_GTIDS) {
          if (ret != GOT_PREVIOUS_GTIDS) {
            /* 见 GTID 却未见 PREV → 逻辑损坏（ER_BINLOG_LOGICAL_CORRUPTION） */
            my_printf_error(ER_BINLOG_LOGICAL_CORRUPTION, msg_fmt, MYF(0), filename,
                "The first global transaction identifier was read, but no other "
                "information regarding identifiers existing on the previous log "
                "files was found.");
            ret = ERROR, done = true; break;
          } else ret = GOT_GTIDS;
        }
        ...
      default:
        /* 见到普通事件却还没见过 PREV → 本文件后续不可能再有 GTID */
        if (ret != GOT_GTIDS && ret != GOT_PREVIOUS_GTIDS) done = true;
        break;
```

★ **Gtid_log_event 先于 Previous_gtids_log_event 出现 = binlog 逻辑损坏**（`ERROR`）。另有优化：`!is_relay_log && prev_gtids != nullptr && all_gtids == nullptr && first_gtid == nullptr` 时读完头部 PREV 即 `done=true`——simple_recovery 第二趟就吃这个。

#### read_gtid_executed_from_table 与 fetch_gtids

```cpp
int Gtid_table_persistor::fetch_gtids(Gtid_set *gtid_set) {
  int ret = 0;  int err = 0;
  TABLE *table = nullptr;
  Gtid_table_access_context table_access_ctx;
  THD *thd = current_thd;

  if (table_access_ctx.init(&thd, &table, false)) {   // 打不开表（含表不存在）
    ret = 1;  goto end;                               // → 返回 1，容忍
  }
  if ((err = table->file->ha_rnd_init(true))) { ret = -1; goto end; }

  while (!(err = table->file->ha_rnd_next(table->record[0]))) {  // 全表扫
    /* 源码 @todo：未来改为 rdlock + 每 sidno 加锁；现在逐行 wrlock */
    global_sid_lock->wrlock();
    if (gtid_set->add_gtid_text(encode_gtid_text(table).c_str()) != RETURN_STATUS_OK) {
      global_sid_lock->unlock();  break;
    }
    global_sid_lock->unlock();
  }
  table->file->ha_rnd_end();
  if (err != HA_ERR_END_OF_FILE) ret = -1;           // 真 IO 错误
end:
  table_access_ctx.deinit(thd, table, 0 != ret, true);
  return ret;
}
```

★ **返回语义两档**：`1` = 表打不开（`OPEN_IF_EXISTS` 策略下表不存在即此路径，如首次启动/`--initialize` 后）→ **容忍**；`-1` = 全表扫描中途真错误 → `unireg_abort(1)` **拒绝启动**。`encode_gtid_text()` 把一行三列拼成 `uuid:start-end` 文本（field[0] + `:` + field[1] + `-` + field[2]）喂给 `add_gtid_text`。

#### gtid_purged 的计算与"只能增长"约束

恢复期推导（`global_sid_lock->wrlock()` 下）：

```cpp
if (!gtids_in_binlog.is_empty() && !gtids_in_binlog.is_subset(executed_gtids)) {
  gtids_in_binlog_not_in_table.add_gtid_set(&gtids_in_binlog);
  if (!executed_gtids->is_empty())
    gtids_in_binlog_not_in_table.remove_gtid_set(executed_gtids);
  /* 注释列了四种补表场景：5.7 升级、从备份配的 slave、
     RESET MASTER 后未轮转即崩溃、崩溃时末个 binlog 的 GTID 未落表 */
  if (gtid_state->save(&gtids_in_binlog_not_in_table) == -1)
    unireg_abort(MYSQLD_ABORT_EXIT);
  executed_gtids->add_gtid_set(&gtids_in_binlog_not_in_table);
}
gtids_only_in_table->add_gtid_set(executed_gtids);
gtids_only_in_table->remove_gtid_set(&gtids_in_binlog);
lost_gtids->add_gtid_set(gtids_only_in_table);
lost_gtids->add_gtid_set(&purged_gtids_from_binlog);
```

★ **表与 binlog 有交集冲突时不判损坏，取并集**：binlog 有而表没有的（崩溃窗口内未落表）补写表；表有而 binlog 没有的进 `gtids_only_in_table` 归入 `lost_gtids`；交集自然保留。唯一 abort 是补写表失败。

`SET @@GLOBAL.GTID_PURGED` 的约束（`Gtid_state::add_lost_gtids`）：

```cpp
  if (!starts_with_plus) {
    /* ★ "只能增长"约束：不带 '+' 的新值必须是旧值的超集 */
    if (!gtid_state->get_lost_gtids()->is_subset(gtid_set)) {
      my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, MYF(0),
               "the new value must be a superset of the old value");
      RETURN_REPORTED_ERROR;
    }
    gtid_set->remove_gtid_set(gtid_state->get_lost_gtids());  // 先减旧 purged
  }
  if (executed_gtids.is_intersection_nonempty(gtid_set)) {    // 不得与已执行重叠
    my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, MYF(0),
             "the added gtid set must not overlap with @@GLOBAL.GTID_EXECUTED");
    RETURN_REPORTED_ERROR;
  }
  if (owned_gtids.is_intersection_nonempty(gtid_set)) {       // 不得与持有中重叠
    my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, MYF(0),
             "the added gtid set must not overlap with @@GLOBAL.GTID_OWNED");
    RETURN_REPORTED_ERROR;
  }
  if (save(gtid_set)) RETURN_REPORTED_ERROR;    // 先落表
  gtids_only_in_table.add_gtid_set(gtid_set); lost_gtids.add_gtid_set(gtid_set);
  executed_gtids.add_gtid_set(gtid_set);
  lock_sidnos(gtid_set); broadcast_sidnos(gtid_set); unlock_sidnos(gtid_set);
```

即：不带 `+` = 替换式（`old_purged ⊆ S` 且 `S ∩ (executed−old_purged) = ∅`，**只能增长不能缩小**）；带 `+` = 追加式（跳过子集检查，但仍不得与 executed/owned 相交）。GTID_PURGED 也写进表，下次恢复时成为 `gtids_only_in_table` 的一部分。

#### mysql.gtid_executed 表结构与压缩线程

```sql
CREATE TABLE IF NOT EXISTS gtid_executed (
    source_uuid CHAR(36) NOT NULL,
    interval_start BIGINT NOT NULL,
    interval_end BIGINT NOT NULL,
    PRIMARY KEY(source_uuid, interval_start)) ENGINE=INNODB TABLESPACE=mysql
```

★ 设计意图：一行存一个 **GNO 区间**而非逐 GNO 一行；主键 `(source_uuid, interval_start)` 使同一 UUID 的区间按起点**天然有序**，压缩算法沿主键顺序扫一遍即可合并相邻区间（`ha_index_first/ha_index_next`，`gno_end+1 == next.start` 即删后行改首行），一轮一个事务。

★ **压缩线程在 8.0.39 仍存在**：`compress_gtid_table()` + `create_compress_gtid_table_thread()`，启动时**无条件压缩一次**，`Gtid_table_persistor::save(..., compress=true)` 成功后 signal。线程 THD 带 `set_skip_readonly_check()`。`gtid_executed_compression_period` 默认 0，sys_var 注释建议设 0：GTID 已改由 InnoDB 随事务持久化，前台大事务量时再唤压缩线程反而拖慢。

#### 恢复后的收尾

1. `previous_gtids_logged->add_gtid_set(&gtids_in_binlog)`——供后续 rotate 时计算补表差集。★ 注意 8.0.39 中 `Gtid_state::update_prev_gtids()` **仍有定义但已无调用点**（历史遗留）；`update_on_flush` **不存在**（5.7 旧名）
2. ★ **启动时补写 `Previous_gtids_log_event`**：`open_binlog()` 创建启动空 binlog 时只能写空 PREVIOUS_GTIDS；恢复完成后用算出的 `gtids_in_binlog` 重建事件 `write_event_to_binlog_and_sync()`（注释原文："Write the previous set of gtids at this point because during the creation of the binary log this is not done"）——不补写则下一个轮转文件的 PREVIOUS_GTIDS 链会断
3. `gtid_state->init()`：`server_sid.parse(server_uuid)` → `sid_map->add_sid` → `server_sidno = sidno; next_free_gno = 1`

| 决策 | 实现 | 代价 / 理由 |
|---|---|---|
| executed 恢复取并集而非判损坏 | 表 ∪ binlog，差集补表 | 崩溃窗口内落表滞后是常态，不可视为损坏 |
| 反向只需一个文件（simple ON） | 找最后一个含 PREV 的文件即停 | 省冷盘全扫；5.6 老 binlog 链下可能算错（注释明示） |
| purged 需第二趟正向扫描 | 找第一个"PREV+GTID"文件 | 兼容 5.6 `SET GTID_PURGED` 轮转语义；OFF 时正扫全部 |
| 表读取失败分两档 | 不存在=1 容忍；IO 错=-1 abort | 区分"首次启动"与"损坏" |
| purged 只能增长 | `is_subset` + 与 executed/owned 交集校验 | 防把已执行事务伪报为丢失 |
| 区间行 + (uuid,start) 主键 | 一行多 GNO；主键序即合并序 | 存储压缩 + O(n) 单事务压缩 |
| PREVIOUS_GTIDS 启动补写 | 恢复完重建事件再 sync | open_binlog 先于恢复，只能事后修正头部 |

### Rollback 时的 GTID 处理

```
update_gtids_impl_own_gtid(thd, is_commit=false)
  → owned_gtids.remove_gtid(thd->owned_gtid)  // 释放所有权
  → 不加入executed_gtids                       // 回滚不记录
  → if (sidno == server_sidno && next_free_gno > gno)
      next_free_gno = gno                       // 回退GNO，填补空洞
  → thd->clear_owned_gtids()
```

Rollback 后该 GNO 可被后续事务重新分配，避免 GNO 空洞永久浪费。

---

## GTID 持久化：三条路径（概览）

GTID 的持久化有三条路径，各自适用于不同场景：

### 路径一：binlog 文件（Gtid_log_event）

主库正常提交时，每个事务在 binlog 中写一个 `Gtid_log_event`，携带完整的 `SID:GNO`。从库回放时读取该 event 获知 GTID。这是最直接的持久化方式。

### 路径二：binlog rotate 刷表

当 binlog 文件发生轮转时，MySQL 将已执行的 GTID 批量写入 `mysql.gtid_executed` 表。这是一种**批量补刷**机制，避免每个事务都写表带来的性能开销。详见上文 [Binlog Rotate 时的持久化](#binlog-rotate-时的持久化)。

### 路径三：InnoDB undo log 异步刷表

对于不开 binlog 或 `log_replica_updates=OFF` 的实例（如从库），没有 `Gtid_log_event` 可依赖，GTID 通过 InnoDB undo log 持久化。详见上文 [InnoDB GTID 持久化（异步刷表）](#innodb-gtid-持久化异步刷表)。

---

## Clone_persist_gtid：InnoDB 侧 GTID 持久化（源码逐段）

> **事实校正（8.0.39 源码核实）**
> - 实现在 `storage/innobase/clone/clone0repl.cc` + `include/clone0repl.h`，**不是** `plugin/clone` 的 `clone_plugin.cc`（那个插件壳里没有这个类）
> - **不存在** `Clone_persist_gtid::instance()`；唯一全局入口是 `clone_sys->get_gtid_persistor()`
> - 成员中**不存在** `m_gtid_state`、`m_gtid_mutex`；同步靠 `trx_sys_serialisation_mutex` + `os_event_t m_event`
> - **不存在** `write_one_gtid()`；写单条的接口是 `add()` + `write_to_table()`
> - `Clone_handler::XA_Operation` / `XA_Block` 定义在 `sql/clone_handler.h`，**不在** `sql/xa.h`

### 类结构：成员、常量、锁

| 成员 | 类型 / 语义 |
|---|---|
| `m_gtids[2]` | `Gitd_info_list`（源码拼写如此）= `std::vector<Gtid_info>`，`Gtid_info` 是 64 字节定长 `std::array`；`GTID_INFO_SIZE == TRX_UNDO_LOG_GTID_LEN`（`static_assert` 保证与 undo 里存的格式一致） |
| `m_active_number` | `atomic<uint64_t>` 当前 active 列表编号 |
| `m_flush_number` | `atomic<uint64_t>` 已刷盘编号；`check_flushed()` 判 `m_flush_number >= req` |
| `m_gtid_trx_no` | `atomic<uint64_t>` ★ **GTID 尚未落表的最小 trx_no**，purge 的硬水位 |
| `m_num_gtid_mem` | `atomic<int>` active list 堆积条数 |
| `m_flush_in_progress` | `atomic<bool>` 落表进行中（影响 `get_oldest_trx_no()`） |
| `m_explicit_request` / `m_flush_request_number` | 强制 flush+压缩请求及其编号 |
| `m_compression_counter` / `m_compression_gtid_counter` | 压缩计数（按写次数 / 按 GTID 条数） |
| `m_event` / `m_close_thread` / `m_thread_active` / `m_active` | 唤醒 / 退出请求 / 线程存活 / 是否接收 GTID |

常量：`GTID_INFO_SIZE=64`、`s_time_threshold=100ms`、`s_gtid_threshold=1024`、`s_compression_threshold=50`、`s_max_gtid_threshold=1024*1024`。

### 双 buffer 的精确语义：编号奇偶取模 + 自增

不是"两个指针互换"，而是**编号奇偶取模** + **active 编号自增**：

```cpp
  Gitd_info_list &get_list(uint64_t list_number) {
    int list_index = (list_number & static_cast<uint64_t>(1));
    return (m_gtids[list_index]);
  }
  Gitd_info_list &get_active_list() {          /* 只在 serialisation mutex 下调用 */
    ut_ad(trx_sys_serialisation_mutex_own());
    return (get_list(m_active_number));
  }
  uint64_t switch_active_list() {
    ut_ad(trx_sys_serialisation_mutex_own());  /* flip 必须持此锁 */
    uint64_t flush_number = m_active_number;
    ++m_active_number;                         /* active 自然翻到另一个 buffer */
    m_compression_gtid_counter += m_num_gtid_mem;
    m_num_gtid_mem.store(0);
    ut_ad(get_active_list().size() == 0);      /* 新 active 必须为空 */
    return (flush_number);                     /* 旧 active 成为本次 flush list */
  }
```

构造时 `m_flush_number=0`、`m_active_number=1`（初始 active 是 `m_gtids[1]`）；`write_to_table()` 末尾 `ut_ad((m_flush_number + 1) == flush_list_number)` 保证每次 flush 恰好 +1。

**事务侧入队 `add()`**：

```cpp
void Clone_persist_gtid::add(const Gtid_desc &gtid_desc) {
  if (!gtid_desc.m_is_set) return;                     /* 无 GTID 直接跳过 */
  if (!is_active() || gtid_table_persistor == nullptr) return;
  ut_ad(trx_sys_serialisation_mutex_own());            /* 调用者持锁 */

  /* 反压：堆积到硬上限，提交线程自己等后台刷完 */
  if (check_max_gtid_threshold() && is_thread_active()) {
    trx_sys_serialisation_mutex_exit();
    wait_flush(false, false, nullptr);
    trx_sys_serialisation_mutex_enter();
  }
  auto &current_gtids = get_active_list();             /* 写入 active，不碰 flush list */
  current_gtids.push_back(gtid_desc.m_info);           /* 64 字节定长拷贝 */
  int current_value = ++m_num_gtid_mem;
  if (current_value == s_gtid_threshold) os_event_set(m_event);  /* 攒够 1024 条唤醒后台 */
}
```

★ **两级阈值**：1024 条 → 唤醒后台提前 flush（软）；1024×1024 条 → 提交线程在 `add()` 里**同步反压** `wait_flush()`（硬）。因为 GTID 会被压缩成区间一起写，实践中硬上限触不到。

强制 flush 的请求编号有一段易错逻辑——**active 为空时，要等的是上一批**：

```cpp
  uint64_t request_immediate_flush(bool compress) {
    trx_sys_serialisation_mutex_enter();
    uint64_t request_number = m_active_number.load();
    if (m_num_gtid_mem.load() == 0) { ut_a(request_number > 0); --request_number; }
    m_flush_request_number = request_number;
    trx_sys_serialisation_mutex_exit();
    if (compress) m_explicit_request.store(true);
    return (request_number);
  }
```

### 上侧：`request_persist_gtid_by_se()` 如何走到 InnoDB

★ **没有 `innobase_request_persist_gtid_by_se()` 这个 handler 回调**。InnoDB 不是被"通知"，而是**主动查询 THD 上的标志位**：

```cpp
  /* sql/sql_class.h —— 只是打两个 bit */
  void request_persist_gtid_by_se() {
    m_se_gtid_flags.set(SE_GTID_PERSIST_EXPLICIT);
    m_se_gtid_flags.set(SE_GTID_PERSIST);
  }
  void set_gtid_persisted_by_se()   { m_se_gtid_flags.set(SE_GTID_PERSIST); }
```

server 侧的分岔口在 `commit_owned_gtids()`——**只有 SE 声明不接管时才由 server 直接写表**：

```cpp
std::pair<int, bool> commit_owned_gtids(THD *thd, bool all) {
  if (is_ha_commit_low_invoking_commit_order(thd, all)) {
    ...
    /* GTID 不由 SE 持久化时，才走 server 的老路径：直接写 mysql.gtid_executed */
    if (thd->owned_gtid.sidno > 0 && !thd->se_persists_gtid()) error = gtid_state->save(thd);
  }
}
```

InnoDB 在 **undo 段首次分配**时（`trx_undo_assign_undo()` 的 `is_first` 分支）调 `set_persist_gtid(trx, true)`，再 `thd->set_gtid_persisted_by_se()`；`persists_gtid()` 决定存几份：

```cpp
trx_undo_t::Gtid_storage Clone_persist_gtid::persists_gtid(const trx_t *trx) {
  auto thd = trx->mysql_thd;
  if (thd == nullptr) thd = thd_get_current_thd();
  if (thd == nullptr || !thd->se_persists_gtid()) return trx_undo_t::Gtid_storage::NONE;
  if (thd->is_extrenal_xa())  return trx_undo_t::Gtid_storage::PREPARE_AND_COMMIT;
  return trx_undo_t::Gtid_storage::COMMIT;   /* 普通事务只存 commit GTID */
}
```

**提交时把 GTID 交给 SE 的确切位置**是 `trx_release_impl_and_expl_locks()`，注释本身就是正确性证明：

```cpp
  if (serialised) {
    trx_sys_serialisation_mutex_enter();
    /* 1. 必须在 undo 中标记 committed 之后：否则 GTID 可能先于事务落盘
       2. 必须在移出 serialisation_list 之前：否则 undo 可能先于 GTID 被 purge */
    if (gtid_desc.m_is_set) {
      auto &gtid_persistor = clone_sys->get_gtid_persistor();
      gtid_persistor.add(gtid_desc);        /* 可能临时释放并重获该 mutex，故必须排在 [2] 之前 */
    }
    trx_erase_from_serialisation_list_low(trx);
    trx_sys_serialisation_mutex_exit();
  }
```

XA PREPARE 走另一条：`trx_set_prepared_in_tc()` 里 `get_gtid_info()` → `add()`；XA ROLLBACK 与恢复走 `trx_undo_gtid_read_and_persist()`。

### 下侧：`write_to_table()` 与两条持久化路径的接缝

```cpp
int Clone_persist_gtid::write_to_table(uint64_t flush_list_number,
                                       Gtid_set &table_gtid_set, Sid_map &sid_map) {
  Gtid_set write_gtid_set(&sid_map, nullptr);
  static const int PREALLOCATED_INTERVAL_COUNT = 64;      /* 栈上预分配区间，少 malloc */
  Gtid_set::Interval iv[PREALLOCATED_INTERVAL_COUNT];
  write_gtid_set.add_interval_memory(PREALLOCATED_INTERVAL_COUNT, iv);

  auto &flush_list = get_list(flush_list_number);         /* 后台线程独占访问 */
  for (auto &gtid_info : flush_list) {                    /* 64 字节文本 → Gtid_set */
    auto gtid_str = reinterpret_cast<const char *>(&gtid_info[0]);
    if (write_gtid_set.add_gtid_text(gtid_str) != RETURN_STATUS_OK) return ER_INTERNAL_ERROR;
  }
  bool is_recovery = !m_thread_active.load();
  if (is_recovery) {
    /* 恢复期：与表里已有 GTID 去重，避免重复插入 */
    write_gtid_set.remove_gtid_set(&table_gtid_set);
    table_gtid_set.add_gtid_set(&write_gtid_set);
  } else {
    /* 运行期：binlog 打开时可能有别的线程并发写表，用全局 prev_gtids 去重 */
    gtid_state->update_prev_gtids(&write_gtid_set);
  }
  if (!write_gtid_set.is_empty()) {
    ++m_compression_counter;
    err = gtid_table_persistor->save(&write_gtid_set, false);   /* compress=false！ */
  }
  flush_list.clear();
  ut_ad((m_flush_number + 1) == flush_list_number);
  m_flush_number.store(flush_list_number);                       /* 推进已刷编号 */
  return (err);
}
```

★ 去重语义（`Gtid_state::update_prev_gtids`）：`if (!opt_bin_log) return;`（关 binlog 不做这层去重）→ `write_gtid_set->remove_gtid_set(&previous_gtids_logged)` 剔掉别的线程已写过的 → `previous_gtids_logged.add_gtid_set(write_gtid_set)` 登记防重写。

`compress=false` 的原因：压缩不在这里做，而是 `flush_gtids()` 尾部按阈值单独调 `gtid_table_persistor->compress(thd)`。

### purge 的相互牵制（两个生命周期的耦合点）

**为什么必须卡住 purge——根因链**（本节最重要的"为什么"）：

1. **GTID 的唯一天然载体是 undo header**。redo 没有事务边界（redo 只描述页修改，不承载"这个事务的 GTID 是什么"）；而 binlog 在"没开 binlog 的从库 / `log_replica_updates=OFF`"场景下**不存在**。所以 8.0 把 GTID 写进 undo header 的专用 GTID 区（`TRX_UNDO_FLAG_GTID` / `TRX_UNDO_FLAG_XA_PREPARE_GTID` 两个标志位），**随 redo 持久化**——这正是它跨崩溃存活的依据（undo 写入机制见 [`../../innodb/undo_log.md`](../../innodb/undo_log.md)「GTID 持久化与 Undo」节）
2. **落表是异步的**。GTID 进 `mysql.gtid_executed` 表要等后台线程 flip + 批量写，存在一个天然窗口：**"undo 里已有 GTID，表里还没有"**
3. **purge 是唯一会物理销毁 undo 的机制**。于是矛盾出现：GTID 靠 undo 保命，purge 却以回收 undo 为天职——**两者必须用一个水位铰接起来，否则窗口期内 purge 把 undo 删了，GTID 就彻底没了**

**若 purge 越过了水位，后果链**：

```
purge 回收了"GTID 尚未落表"的 undo
   → 该 GTID 的三个载体（undo / 表 / binlog）全部没有 → 从 gtid_executed 永久消失
   → 该实例未来被提升为主库、或崩溃后从库用 GTID auto_position 重连时：
       "已执行集合"比实际少一个 GTID → 从库认为没执行过 → 重放对应 binlog
   → 主键冲突 / 数据错乱 / 主从不一致（且无任何报错，静默发生）
```

这与 redo_log.md 崩溃恢复章的教训同构：**GTID 集合的完整性 = 复制正确性的前提**，任何"静默少一个 GTID"都等价于数据破坏。

```cpp
  trx_id_t get_oldest_trx_no() {
    trx_id_t ret_no = m_gtid_trx_no.load();
    if (ret_no == TRX_ID_MAX) {              /* 特殊值：尚无 GTID 需要落表 */
      ut_ad(!is_thread_active());
      ut_ad(m_num_gtid_mem.load() == 0);
    } else if (m_num_gtid_mem.load() == 0) {
      /* 内存里没有积压、且没有 flush 在进行 → 已全部落表，purge 无需等待 */
      if (!m_flush_in_progress.load()) ret_no = TRX_ID_MAX;
    }
    return (ret_no);
  }
```

注意 `m_flush_in_progress` 这个"充分不必要"的**保守判断**：宁可让 purge 多等，也不能让它越过正在写的批次。

水位如何扣到 purge view 上（`MVCC::clone_oldest_view()` 尾部，purge 每次取视图时生效）：

```cpp
  /* Update view to block purging transaction till GTID is persisted. */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  view->reduce_low_limit(gtid_persistor.get_oldest_trx_no());
  /*  void reduce_low_limit(trx_id_t trx_no) {
        if (trx_no < m_low_limit_no) { m_low_limit_no = trx_no; }   只向下压，不抬高
      } */
```

purge 的推进判据是 `purge_sys->iter.trx_no < purge_sys->view.low_limit_no()`，undo truncate 也用 `view->low_limit_no()` 兜顶——**所以 GTID 没落表，undo 就一个字节都不能回收**。

落表后如何放开：

```cpp
void Clone_persist_gtid::update_gtid_trx_no(trx_id_t new_gtid_trx_no) {
  auto trx_no = m_gtid_trx_no.load();
  if (trx_no != TRX_ID_MAX && trx_no >= new_gtid_trx_no) return;
  m_gtid_trx_no.store(new_gtid_trx_no);      /* 内存水位前进 */
  trx_sys_persist_gtid_num(new_gtid_trx_no); /* 写进 TRX_SYS 页，崩溃恢复后仍有效 */
  srv_purge_wakeup();                        /* 唤醒 purge 立刻重新取视图 */
}
```

`new_gtid_trx_no` 取自 `trx_sys_oldest_trx_no()`（serialisation_list 队首的 `no`），**在持 serialisation mutex 时、切表之前采样**。因为"入 GTID list"排在"移出 serialisation list"之前，凡 `no < oldest_trx_no` 的事务，其 GTID 必已在刚切出的 flush list 里——**这就是水位安全的证明**。

**水位为什么要落盘（`trx_sys_persist_gtid_num`）**——崩溃安全性的最后一块拼图：

```cpp
/* storage/innobase/trx/trx0sys.cc —— 写 TRX_SYS 页，产生 redo */
void trx_sys_persist_gtid_num(trx_id_t gtid_trx_no) {
  mtr_t mtr;
  mtr.start();
  auto sys_header = trx_sysf_get(&mtr);
  /* Update GTID transaction number. All transactions with lower
  transaction number are no longer processed for GTID. */
  mlog_write_ull(page + TRX_SYS_TRX_NUM_GTID, gtid_trx_no, &mtr);
  mtr.commit();
}
```

`TRX_SYS_TRX_NUM_GTID` 是 TRX_SYS 页上预留的 8 字节（`TRX_SYS_MYSQL_LOG_INFO + 名字长度`之后），语义即注释：**"低于此 trx_no 的事务不再需要处理 GTID"**。若不落盘，崩溃重启后：

- 内存 `m_gtid_trx_no` 归零 ⇒ purge 不知道哪些 undo 已安全，**最坏会提前回收 → 丢 GTID**（正是上面的后果链）
- 恢复扫描 `trx_rseg_persist_gtid()` 从 TRX_SYS 页读出水位（`mach_read_from_8(page + TRX_SYS_TRX_NUM_GTID)`），只扫 `trx_no >= 水位` 的 history 节点，水位以下**已确认落表**直接跳过——不重复、不漏扫
- 注意它走 `mlog_write_ull` + `mtr.commit()`：**水位更新本身产生 redo**，所以它要么和 GTID 落表一起崩溃前生效、要么一起回滚，不存在"表已写、水位没落盘"的中间态

由此，整个恢复期的时序是：`srv_start` 恢复阶段先 `set_oldest_trx_no_recovery(水位)` 恢复内存水位 → `trx_rseg_persist_gtid` 从水位接着扫 undo 补 GTID 进 active list → 后台线程 `periodic_write` 首刷 `flush_gtids()`（is_recovery 分支：先去重再写表）→ 水位继续前进。**"undo 里的 GTID"与"表里的 GTID"在崩溃后依然收敛到同一集合**。

反方向（为什么必须先落表）：GTID 存在 undo header 里，恢复时靠扫 undo 找回。`trx_rseg_persist_gtid()` 从 history list 头（最新端）往尾（最旧端）扫：

```cpp
static void trx_rseg_persist_gtid(trx_rseg_t *rseg, trx_id_t gtid_trx_no) {
  if (gtid_trx_no == 0) return;                       /* 老版本未启用 GTID 持久化 */
  mtr_t mtr; mtr_start(&mtr);                         /* 全程只读，不产生 redo */
  auto rseg_max_trx_no = mach_read_from_8(rseg_header + TRX_RSEG_MAX_TRX_NO);
  if (rseg_max_trx_no < gtid_trx_no) { mtr_commit(&mtr); return; }
  fil_addr_t node_addr = flst_get_first(rseg_header + TRX_RSEG_HISTORY, &mtr);
  mtr_commit(&mtr);
  while (node_addr.page != FIL_NULL) {
    mtr_start(&mtr);
    auto undo_trx_no = mach_read_from_8(undo_log + TRX_UNDO_TRX_NO);
    if (undo_trx_no < gtid_trx_no) { mtr_commit(&mtr); break; }  /* 已落表，后面更旧，停 */
    trx_undo_gtid_read_and_persist(undo_log);                    /* → gtid_persistor.add() */
    node_addr = flst_get_next_addr(node, &mtr);
    mtr_commit(&mtr);
  }
}
```

**耦合闭环**：undo 回收需要 `trx_no < purge 水位`；purge 水位被 `m_gtid_trx_no` 压住；`m_gtid_trx_no` 前进依赖 GTID 落表；而落表的数据源（恢复期）正是那些还没被回收的 undo header。**若 purge 先跑、undo 被 truncate，GTID 就永久丢失** → 从库 `gtid_executed` 出现空洞。交叉印证：`trx_purge_run()`（恢复 purge）第一行就是 `gtid_persistor.wait_flush(false,false,nullptr)`，主动把 GTID 挤下去换 purge 立刻能推进。

```
  提交线程 (N 并发)                  GTID 后台线程                     purge 线程
  ─────────────────                ──────────────                    ────────────
  trx_release_impl_and_expl_locks
   ├─ serialisation_mutex_enter
   ├─ add() → m_gtids[active&1] ──┐
   └─ erase_from_serialisation    │
                                  │        periodic_write 循环
                                  │         ├─ mutex_enter
                                  │         ├─ oldest = trx_sys_oldest_trx_no()  ← 采样水位
                                  │         ├─ switch_active_list()  ← flip（持锁，O(1)）
                                  └─────────┤   active 编号 +1，旧 list 交出
                                            ├─ mutex_exit          ← 写表期间不持锁！
                                            ├─ write_to_table() → Gtid_table_persistor::save()
                                            ├─ m_flush_number++（独占，故不会回写同一 list）
                                            └─ update_gtid_trx_no(oldest)
                                                    │
                                                    ├─ trx_sys_persist_gtid_num()（落 TRX_SYS 页）
                                                    └─ srv_purge_wakeup() ──────► 重取读视图
                                                                                  reduce_low_limit(
                                                                                    get_oldest_trx_no())
                                                                                  → 只能 purge
                                                                                    trx_no < 水位
```

### XA 与 clone 的交叉

`Clone_handler::XA_Block`（`sql/clone_handler.h`）：`begin_xa_operation()` 先无锁快路径 `if (!s_xa_block_op.load()) return;`，被挡住才退到 `s_xa_mutex` 上 10ms 轮询；`block_xa_operation()` 置位后等 `s_xa_counter` 归零，60 秒超时返回失败。

clone 在 `Clone_Snapshot::synchronize_binlog_gtid()` 前加这道闸，源码注释就是原因：

```cpp
  /* Block external XA operations. XA prepare commit and rollback operations
  are first logged to binlog and added to global gtid_executed before doing
  operation in SE. Without blocking, we might persist such GTIDs from global
  gtid_executed before the operations are persisted in Innodb. */
  Clone_handler::XA_Block xa_block_guard(thd);
```

即：**外部 XA 的 GTID 会先进入全局 `gtid_executed` 再落 SE**，若不挡住，clone 可能把"SE 里还没提交"的 GTID 当成已持久化写进快照。

`write_other_gtids()` 补的是另一个洞——**非 InnoDB 事务的 GTID**（其他引擎、或纯 binlog 产生的），InnoDB 的 undo 里根本没有它们：

```cpp
int Clone_persist_gtid::write_other_gtids() {
  if (opt_bin_log) return gtid_state->save_gtids_of_last_binlog_into_table();
  return 0;
}
```

★ 它**只在 `flush_gtids()` 里、压缩之前**被调用一次——因为压缩会把多行区间合并成一行，**漏掉的区间压缩后就再也找不回来了**。

### flush 触发时机全表

| 触发方式 | 入口 | 说明 |
|---|---|---|
| 周期 | `periodic_write()` → `os_event_wait_time(m_event, 100ms)` 超时即 flush | 基础节奏 100ms |
| 攒够 | `add()` 中 `m_num_gtid_mem == 1024` → `os_event_set` | 高负载下不必等满 100ms |
| 硬反压 | `add()` 中 `>= s_max_gtid_threshold` → 提交线程自己 `wait_flush()` | 提交线程阻塞换内存不涨 |
| 强制 | `wait_flush(compress, early_timeout, cbk)` | clone 快照前、`innobase_flush_logs()` 非 group-commit 分支（即 `FLUSH LOGS`）、`trx_purge_run()`、`RESET MASTER` |
| 关闭/启动 | `periodic_write()` 首次进入先 `flush_gtids()`；退出循环后慢关闭再 flush 一次 | 启动那次用于消费恢复期从 undo 扫出的 GTID |
| 压缩阈值 | `check_compress()` | **仅 `!opt_bin_log` 时**看 `gtid_executed_compression_period`；另看本地 `s_compression_threshold=50` 次写 |

### 后台线程

`Clone_persist_gtid::start()` 用 `os_thread_create` 建线程，句柄存 `srv_threads.m_gtid_persister`；调用点在 `srv_start()` 中**恢复完成之后、`srv_start_purge_threads()` 之前**（注释 "Start and consume all GTIDs for recovered transactions"）。

```cpp
void Clone_persist_gtid::periodic_write() {
  auto thd = create_internal_thd();
  thd->set_skip_readonly_check();        /* 只读实例也要能写 gtid_executed */

  flush_gtids(thd);                      /* ① 先消费恢复期从 undo 扫出的 GTID */
  m_thread_active.store(true);           /* ② 才放开 start() 的等待者 */

  for (;;) {
    auto is_shutdown = (srv_shutdown_state.load() >= SRV_SHUTDOWN_CLEANUP);
    if (is_shutdown || m_close_thread.load()) { m_active.store(false); break; }
    if (!flush_immediate()) os_event_wait_time(m_event, s_time_threshold);
    os_event_reset(m_event);
    flush_gtids(thd);                    /* 无论超时还是被唤醒，都先 flush 一次 */
  }
  /* 慢关闭：把最后一批刷掉，让 undo 能被 purge；刷不干净则告警 */
  if (m_num_gtid_mem.load() > 0 && srv_fast_shutdown < 2) {
    flush_gtids(thd);
    if (m_num_gtid_mem.load() > 0) ib::warn(ER_IB_MSG_GTID_FLUSH_AT_SHUTDOWN);
  }
  m_active.store(false);
  destroy_internal_thd(thd);
  m_thread_active.store(false);
}
```

① 的顺序不可换：恢复期 `m_thread_active==false`，`flush_gtids()` / `write_to_table()` 走 "is_recovery" 分支（先从表里 `fetch_gtids()` 去重、写完强制压缩、并 `clone_update_gtid_status()` 收尾）。

`flush_gtids()` 里唯一消费者就是这个后台线程——**这保证了 `write_to_table()` 在放锁期间独占 flush list**，不会有两个 flush 同时摸同一个 `m_gtids[]`，也就不需要额外的 buffer 锁。

### 设计决策点

| 决策 | 选择 | 代价 / 理由 |
|---|---|---|
| 缓冲结构 | 编号奇偶取模的双 list，而非指针交换或环形队列 | 切换是 O(1) 自增整数；`m_flush_number+1==n` 断言让"刷到哪"可线性比较 |
| 切换时机 | 只在 `flush_gtids()` 持 serialisation mutex 时切 | 提交线程与 flip 互斥，但写表本身放锁，不把 IO 延迟灌进提交路径 |
| 写表不持锁 | `write_to_table()` 在放锁后执行 | 靠"单一消费者"保证独占，换提交路径不被磁盘 IO 阻塞 |
| 同步原语 | 复用 `trx_sys_serialisation_mutex` + `os_event`，无自有 mutex | 少一把全局锁；代价是 GTID 入队与事务串行化共用临界区 |
| 水位采样 | 切表前取 `trx_sys_oldest_trx_no()` | 与"先入 GTID list、后出 serialisation list"的顺序共同构成不丢的证明 |
| purge 耦合 | `reduce_low_limit(get_oldest_trx_no())` 压低 purge view | 以"purge 偶尔多等"换"GTID 绝不丢、undo 绝不提前回收" |
| 上限处理 | 1024 唤醒 / 1M 同步反压 | 软阈值摊薄写表次数，硬阈值是内存兜底 |
| 压缩时机 | 由 GTID 线程代管，`save(..., compress=false)` | 避免 server 侧压缩线程与 InnoDB 落表竞争 |
| XA 处理 | clone 期 `XA_Block` 挡住全部外部 XA | XA 的 GTID 先入全局集合再落 SE，不挡会克隆出"超前"的 GTID |
| 非 InnoDB GTID | 压缩前 `write_other_gtids()` 补一次 | 压缩不可逆，漏掉的区间再也无法从 undo 找回 |

