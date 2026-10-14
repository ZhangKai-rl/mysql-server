# RCU（Read-Copy-Update）：MySQL 的读多写少并发原语

> **分类**：本篇属 **同步原语**（并发编程基础设施），不是事务锁/语义锁——与 [MDL](../transactional/mdl.md)、[InnoDB 行锁](../transactional/innodb_trx_lock.md) 是两类完全不同的机制。锁的全景与分类见 [`../README.md`](../README.md)。

> 本篇覆盖：RCU 的通用理论与 MySQL 8.0 的具体实现 `MyRcuLock<T>`（`include/my_rcu_lock.h`）——逐行剖析 + 使用场景（SSL acceptor context）+ 内存序取舍 + 受限之处。

## 目录

- [一、RCU 是什么：通用理论](#一rcu-是什么通用理论)
- [二、MyRcuLock：受限版 RCU（逐行剖析）](#二myrcuLock受限版-rcu逐行剖析)
- [三、使用场景：SSL acceptor context](#三使用场景ssl-acceptor-context)
- [四、与读写锁的对比](#四与读写锁的对比)
- [五、受限之处与使用边界](#五受限之处与使用边界)
- [六、单元测试揭示的使用前提](#六单元测试揭示的使用前提)
- [参考](#参考)

---

## 一、RCU 是什么：通用理论

⚠️ **先纠正 8.0.39 的名字（几乎全错）**：

| 常见说法 | 真相 |
|---|---|
| `include/sql/rcu_lock.h` / `sql/rcu_lock.cc` | **都不存在**。真名 **`include/my_rcu_lock.h`**，且**没有任何 .cc**——类是**纯头文件模板**，全部 inline |
| `read_lock()` / `read_unlock()` | 真名 **`rcu_read()` / `rcu_end_read()`** |
| `synchronize()` / `sync()` | 真名 **`wait_for_no_readers()`** |
| `force_read_unlock()` | **不存在**（0 匹配） |
| `no_readers()` / `active_readers()` / `get_readers_count()` | **全不存在**。计数器 `rcu_readers_` 是 `protected`，无任何 accessor |
| `rcu_set_pointer` / `set_pointer` | 真名 **`rcu_write(const T*)`**（底层）与 **`write_wait_and_delete(const T*)`**（高层） |
| `rcu_read_auto_lock` / `RCU_read_lock` / `rcu_read_lock_guard` / `Rcu_lock_guard` | **全 0 匹配**。真名是嵌套类 **`MyRcuLock<T>::ReadLock`** |
| `unittest/gunit/rcu_lock-t.cc` | 真名 **`unittest/gunit/my_rcu_lock-t.cc`** |
| `rcu_dereference` / `rcu_assign_pointer` / `call_rcu` / `rcu_barrier` | **全 0 匹配**——MySQL 是自研极简版，不配套内核那套宏 |

**★ doxygen 原话承认这是"简化版"**（`include/my_rcu_lock.h:33`）：

```cpp
/**
  A class that implements a limited version of the Read-Copy-Update lock pattern
  ...
  The full RCU implementation solves this in a very generic way.
  But we here take a simplification: we assume that there will be frequent times
  when there's not gonna be active readers.          // ★★ 核心设计假设

  If we count atomically the active readers (by calling MyRcuLock::rcu_read()
  ... we can detect these times by simply looping over the atomic count
  (MyRcuLock::rcu_readers_) and waiting for a 0 to come out (this is exactly
  what MyRcuLock::wait_for_no_readers() does). Once we get a zero we know
  that no existing readers will use the old value(s) of the global and new
  readers will get the new value of the global. So if we get that zero we know
  that we can safely dispose of all the old values we hold.
  ...
  @tparam T The class of the global protected pointer
*/
```

> ★★ **结论**：这不是真正的 RCU（没有 grace period / epoch / 回调链表 / `call_rcu`），而是**轮询式 quiescent-state 检测**——靠"读者计数归零"来判定宽限期结束。
>
> ★ 实例化粒度：**每个被保护的全局一个 `MyRcuLock` 实例**（"We instantiate MyRcuLock once for each global we want to protect"），不是全局单例。

**被保护的数据成员与 cache-line padding**（`:216`）：

```cpp
 protected:
  /** the global pointer to protect */
  std::atomic<const T *> rcu_global_;
  /** padding to break the CPU cache lines */
  char rcu_padding_[128];      // ★★ 128 字节填充：把"高频读的指针"与"高频写的计数器"隔开，防 false sharing
  /** the readers counter */
  std::atomic<long> rcu_readers_;
```

> ⚠️ `rcu_padding_` 是 **128 字节**（不是通常的 64）——比一个 cache line 还宽。
> ⚠️ `rcu_readers_` 是 `protected` 且**无任何 getter** —— **无法从外部观测当前读者数**（排查"写者卡住"时这是个盲区）。

**全类只有 5 处原子操作，内存序全表**：

| 位置 | 代码 | 内存序 | 评价 |
|---|---|---|---|
| 构造 | `rcu_global_.store(init)` / `rcu_readers_.store(0)` | 默认 `seq_cst` | 构造期无竞争 |
| `ReadLock::operator const T*` | `return _lock->rcu_global_;` | **隐式 `seq_cst`** | ⚠️ **与 `rcu_read()` 的 relaxed 不一致**——走 RAII 反而更重 |
| `rcu_read()` | `fetch_add(1, relaxed)` | `relaxed` | RMW 本身原子即可，读者间无需 happens-before |
| `rcu_read()` | `rcu_global_.load(relaxed)` | **`relaxed`** | ⚠️ **严格来说偏弱**：缺 acquire，理论上弱序架构（ARM/POWER）有风险 |
| `rcu_end_read()` | `fetch_sub(1, relaxed)` | `relaxed` | 仅需原子性 |
| `rcu_write()` | `rcu_global_.exchange(newT, release)` | **`release`** | ★ 唯一的单向同步点：`newT` 内部构造先于交换可见 |
| `wait_for_no_readers()` | `rcu_readers_.load(relaxed)` | `relaxed` | 只关心最终收敛到 0 |

> ★★ **全类没有一个 `memory_order_acquire`** —— 写者 `release` / 读者 `relaxed`，**release/acquire 并未配对**。正确性实际依赖：①`fetch_add` 在 `load` 之前（程序序）；②x86-TSO 的强内存模型。**在 ARM/POWER 上是可疑的**。

**为什么 epoll / 定时器 / 插件都不用**：全仓 `MyRcuLock` 精确命中只有 **4 个文件**，其中生产代码只有 **2 个**（`sql/ssl_acceptor_context_operator.h`、`sql/ssl_acceptor_context_data.h`）。`sql/` 下搜 `rcu` 命中的 40+ 文件全是 `circular` 的子串误报。`mysqld.cc` 里 "RCU" 只出现在字符串 `"...break infinite loops in circular replication..."`（:9172）。**没有任何全局 RCU 初始化/销毁函数**——生命周期完全由 `Ssl_acceptor_context_container` 掌管。

**RCU = Read-Copy-Update**，面向**读多写少**场景的同步机制（Linux 内核 2.6 引入，Paul McKenney 主导；2006 年后广泛用于内核路由表、链表等）。

名字就是它的三步：

| 步骤 | 做什么 |
|------|--------|
| **Read** | 读者直接读当前指针，**不加锁、不阻塞** |
| **Copy** | 写者**不原地改**，复制一份新版本，在新副本上修改 |
| **Update** | 改完后**原子换指针**，之后的读者看到新版本 |
| （隐含）**延迟回收** | 旧版本不能立刻删——可能还有读者持有旧指针。必须等所有"换指针前开始的读者"结束（这段等待叫 **grace period**），才能安全释放 |

**核心特征**：读者从不阻塞、也绝不被写者阻塞；读写可以**真正并发**（读者读旧版本、写者改新副本）。

**grace period（宽限期）**：从写者换指针那一刻起，到"所有换指针前已开始的读者全部结束"为止的时间段。完整 RCU 的关键工程问题就是**如何高效检测 grace period 结束**——Linux 内核用 **quiescent state**（静默状态：CPU 经历调度/上下文切换/进入用户态等"肯定没在读 RCU 数据"的时刻），读者**无需显式注册**。

## 二、MyRcuLock：受限版 RCU（逐行剖析）

`include/my_rcu_lock.h`，注释（:34）明确："A class that implements a **limited version** of the Read-Copy-Update lock pattern"。

### 2.1 成员布局（:216-222）

```cpp
protected:
  std::atomic<const T *> rcu_global_;   // :218 被保护的全局指针
  char rcu_padding_[128];               // :220 ★ padding 打破 cache line
  std::atomic<long> rcu_readers_;       // :222 活跃读者计数
```

**`rcu_padding_[128]` 是防 false sharing**：读者高频 `fetch_add` 修改 `rcu_readers_`，若它与 `rcu_global_` 在同一 cache line，会令持有该行的所有核的缓存失效，拖慢其他读者读指针。128 字节 padding（> 常见 64 字节 cache line）把两个热原子变量隔开。这是高性能并发代码的标配技巧。

### 2.2 读者：显式计数 + relaxed 读（:135-153）

```cpp
  /**
    Low level API: start reading

    Returns a copy of the global that you can safely use until @ref
    MyRcuLock::rcu_end_read() is called
    @warning Don't forget to call @ref MyRcuLock::rcu_end_read() when done.
    @return the copy of the global
  */
  const T *rcu_read() {
    rcu_readers_.fetch_add(1, std::memory_order_relaxed);   // ①先登记自己为活跃读者
    return rcu_global_.load(std::memory_order_relaxed);     // ②★再取指针
  }

  /**
    Low level API: end reading

    Marks the place where the pointer returned by @ref MyRcuLock::read() is
    not going to be accessed any longer.

    The recommended high level API is @ref MyRcuLock::ReadLock
    @warning Each call to @ref MyRcuLock::rcu_read() must be coupled with a
    call to @ref MyRcuLock::rcu_end_read(). And the value returned by @ref
    MyRcuLock::read() should not be used after calling @ref
    MyRcuLock::rcu_end_read()
  */
  void rcu_end_read() { rcu_readers_.fetch_sub(1, std::memory_order_relaxed); }
```

> ★★ **①②顺序不可颠倒**：必须先 `fetch_add`（登记）再 `load`（取快照）。若反过来，写者可能在登记前就观察到计数为 0 并回收旧对象，读者随后拿到的就是悬垂指针。

**RAII 封装（真名 `MyRcuLock<T>::ReadLock`）**（`:113`）：

```cpp
  /**
    High level read API for readers

    A convenience scope guard class for readers.Use this for all of the RCU
    global readers.
  */
  class ReadLock {
   public:
    /** get the value through the scope guard */
    operator const T *() { return _lock->rcu_global_; }   // ★隐式转换；未指定内存序 → seq_cst
    /** construct a new read lock scope guard */
    ReadLock(MyRcuLock *l) : _lock(l) { _lock->rcu_read(); }   // ★构造即 rcu_read()
    ReadLock(const ReadLock &) = delete;
    ~ReadLock() { _lock->rcu_end_read(); }                     // ★析构即 rcu_end_read()

   protected:
    MyRcuLock operator=(const ReadLock) = delete;   // ⚠️返回类型写成了注入类名 MyRcuLock<T>（本意应是 ReadLock&），且在 protected 段
    MyRcuLock *_lock;
  };
```

**写者 API（真名 `rcu_write` / `write_wait_and_delete`）**（`:172` / `:205`）：

```cpp
  const T *rcu_write(const T *newT) {
    return rcu_global_.exchange(newT, std::memory_order_release);
    //     ★ 原子的 exchange：发布新值 + 取回旧值一步完成
  }

  /**
    A RCU writer API

    Stores a new value into the global, waits for when it's safe to dispose of
    the old one and calls delete on it.

    @param newT The new value to store into the global

    @retval true the function was stopped by the functor. The old value was lost.
    @retval false operation succeeded and the old value was disposed of
    */
  bool write_wait_and_delete(const T *newT) {
    const T *oldT = this->rcu_write(newT);   // ①发布新值，拿回旧值
    if (!oldT) return false;
    if (!wait_for_no_readers()) {            // ②阻塞式宽限期
      delete oldT;                           // ③★真正的回收点：直接 delete，无 call_rcu、无延迟回调链表
      return false;
    }
    // we leak the oldT here               // ⚠️不可达分支
    return true;
  }
```

**析构 = 最后一次回收**（`:105`）：

```cpp
  /**
    Destructor.
    Calls @ref MyRcuLock::write_wait_and_delete to set the global
    to NULL and dispose of the old value
  */
  ~MyRcuLock() { write_wait_and_delete(nullptr); }
  // ★ 销毁 = 置 NULL + 等读者清零 + delete 旧值。这也是唯一会"阻塞在析构里"的地方
```

```cpp
const T *rcu_read() {
  rcu_readers_.fetch_add(1, std::memory_order_relaxed);   // ① 读者数 +1
  return rcu_global_.load(std::memory_order_relaxed);     // ② 读指针
}
void rcu_end_read() {
  rcu_readers_.fetch_sub(1, std::memory_order_relaxed);   // ③ 用完后 -1
}
```

**为什么计数用 relaxed 就够**：RCU 的正确性**不靠内存序**，而靠"读者计数保证旧对象不被提前释放"：

- 读者读到旧指针是**合法**的（RCU 语义允许读者看到旧版本）
- 只要计数不归零，写者就不回收旧对象 → 读者持有的旧指针永远有效
- 对象内容在"发布为全局"之前就已构造完成（写者先完整构造新对象，再换指针）

### 2.3 写者：换指针（:172-174）

```cpp
const T *rcu_write(const T *newT) {
  return rcu_global_.exchange(newT, std::memory_order_release);
}
```

- `exchange` = 原子地"写新值、返回旧值"——一步完成"Update"并拿到待回收的旧版本
- **`release` 的语义**：新对象的初始化（发生在 `rcu_write` 之前的所有写）对**之后**读到新指针的读者可见。读者侧虽然用 relaxed load，但在 x86-TSO 硬件模型下 load 能看到最近 store；release 在这里是防御性/文档性的——核心安全网仍然是读者计数

### 2.4 回收：轮询等待（:187-191）

```cpp
  /**
    Low level API: wait for no active readers

    Call this to wait for a state when there's no active readers.
    Optionally pass a functor to call at each check and stop if instructed.
                        // ⚠️A：文档承诺"可选 functor"，但函数签名根本没有参数 —— 死代码文档

    The high level API to call is @ref MyRcuLock::write_wait_and_delete()

    @retval true if the function was stopped by the functor
                        // ⚠️B：实现恒返回 false，此分支永不成立
    @retval false if an actual moment of 0 readers was detected
  */
  bool wait_for_no_readers() {
    bool stopped = false;
    while (rcu_readers_.load(std::memory_order_relaxed) > 0) my_sleep(10000);
    //                                          ★★ 这就是"宽限期"：忙轮询 + 每次睡 10000 微秒 = 10ms
    return stopped;    // ★ 恒 false → 永不超时、永不放弃
  }
```

```cpp
// include/my_systime.h:59 —— 纯 sleep，无自旋、无 yield
inline void my_sleep(time_t m_seconds) {
  std::this_thread::sleep_for(std::chrono::microseconds{m_seconds});
}
```

> ★★ **两个必须记住的结论**：
> 1. **无超时**：`while` 没有退出条件，`stopped` 恒 false。**一个不退出的读者 = 写者永久阻塞**（活锁风险）。
> 2. **10ms 粒度**：`my_sleep(10000)` 决定了写者最坏情况的回收延迟。不是忙等，但也远不是即时的。
>
> ⚠️ **文档与实现不符**：doxygen 说"可选传一个 functor，被指示时可停止、返回 true"，但**函数签名零参数、实现恒返回 false**——**取消特性从未实现**。`#include <functional>`（第 28 行）**从未使用**，正是这个被砍掉特性的残留。
> ⚠️ 因此 `write_wait_and_delete()` 里的 `// we leak the oldT here` 分支**不可达**——永不泄漏。

```cpp
bool wait_for_no_readers() {
  bool stopped = false;
  while (rcu_readers_.load(std::memory_order_relaxed) > 0) my_sleep(10000);
  return stopped;
}
```

**这就是"受限"的核心**：完整 RCU 用 quiescent state 检测 grace period；MySQL 这里是**显式计数 + 10ms 轮询**。注释（:46-48）交代了简化假设：

> "we assume that there will be **frequent times when there's not gonna be active readers**"

（假设会有频繁的"零活跃读者"时刻。）

### 2.5 高层 API：写→等→删（:205-214）

```cpp
bool write_wait_and_delete(const T *newT) {
  const T *oldT = this->rcu_write(newT);   // ① 换指针，拿到旧版本
  if (!oldT) return false;
  if (!wait_for_no_readers()) {            // ② 等读者归零
    delete oldT;                           // ③ 安全了，回收
    return false;
  }
  // we leak the oldT here                  // ★ 等不到 → 泄漏旧版本（见第五节）
  return true;
}
```

### 2.6 读者 RAII 封装（:113-125）

```cpp
class ReadLock {
 public:
  operator const T *() { return _lock->rcu_global_; }
  ReadLock(MyRcuLock *l) : _lock(l) { _lock->rcu_read(); }   // 构造 = 进入读区
  ~ReadLock() { _lock->rcu_end_read(); }                     // 析构 = 退出读区
};
```

出作用域后不能再使用该指针（RCU 的核心纪律）。

## 三、使用场景：SSL acceptor context

`sql/ssl_acceptor_context_operator.h` —— 教科书级的 RCU 场景：

```cpp
// :47
using Ssl_acceptor_context_data_lock = MyRcuLock<Ssl_acceptor_context_data>;

// :41-54 TLS context 容器
class Ssl_acceptor_context_container {
 protected:
  Ssl_acceptor_context_data_lock *lock_;
  void switch_data(Ssl_acceptor_context_data *new_data);   // 写操作
  ...
};

extern Ssl_acceptor_context_container *mysql_main;    // :56 主连接 channel
extern Ssl_acceptor_context_container *mysql_admin;   // :57 admin 连接 channel
```

**读多写少的极致**：

| 操作 | 频率 | 走哪个 API |
|------|------|-----------|
| TLS 握手（读 SSL context） | **每个新连接一次**，极高 | `Lock_and_access_ssl_acceptor_context`（:105，内部持 `ReadLock`） |
| 重载证书（`ALTER INSTANCE RELOAD TLS`） | 运维手动触发，极低 | `switch_data` → `write_wait_and_delete` |

**读者访问器**（:105-139）用 `operator` 转换直接拿四种形态：

```cpp
class Lock_and_access_ssl_acceptor_context {
 public:
  Lock_and_access_ssl_acceptor_context(Ssl_acceptor_context_container *context)
      : read_lock_(context->lock_) {}                    // ★ 构造即加读计数
  operator const Ssl_acceptor_context_data *() { ... }   // 拿数据
  operator SSL_CTX *() { return c->ssl_acceptor_fd_->ssl_context; }  // 拿 OpenSSL 上下文
  operator SSL *() { ... }
  operator struct st_VioSSLFd *() { ... }
 private:
  Ssl_acceptor_context_data_lock::ReadLock read_lock_;   // 析构自动减计数
};
```

每次握手代码形如：

```cpp
Lock_and_access_ssl_acceptor_context context(mysql_main);
SSL_CTX *ctx = context;   // operator SSL_CTX*()，使用期间旧版本保证不被回收
// ... 用 ctx 完成握手 ...
// 作用域结束，ReadLock 析构，计数 -1
```

**为什么这里不能用读写锁**：每个连接握手都做一次 `rwlock rdlock/unlock`，在高并发（数千连接）下 `rwlock` 的原子操作会让缓存行在核间乒乓（cache-line bouncing），成为吞吐瓶颈。RCU 的读者开销只剩一次 relaxed `fetch_add` + 一次 relaxed `load`。

## 四、与读写锁的对比

| | 互斥锁 | 读写锁 | **RCU** |
|---|---|---|---|
| 读者开销 | 原子操作 + 可能阻塞 | 原子操作 + 可能阻塞 | 计数 +1 读指针（relaxed） |
| 读者会阻塞吗 | 会 | 会（写者持锁时） | **不会** |
| 读写并发 | 否 | 否 | **是**（读旧、写新） |
| 多版本共存 | 否 | 否 | 是（旧版本在 grace period 内存活） |
| 写者代价 | 低 | 低 | 高（复制 + 等读者归零） |
| 适合 | 通用 | 读多写少 | **读极多、写极少、允许短暂读旧版本** |

RCU 的语义代价：**读者可能读到旧版本**（换指针瞬间已开始的读者拿到旧值）。对 SSL context 完全可接受——握手用的是"当前生效的证书"，重载证书后旧连接继续用旧 context 反而是**期望行为**。

## 五、受限之处与使用边界

**唯一生产实例：TLS/SSL acceptor context**（`sql/ssl_acceptor_context_operator.h:40`）：

```cpp
/** TLS context access protector */
class Ssl_acceptor_context_container {
 protected:
  Ssl_acceptor_context_container(Ssl_acceptor_context_data *data);
  ~Ssl_acceptor_context_container();
  void switch_data(Ssl_acceptor_context_data *new_data);

  using Ssl_acceptor_context_data_lock = MyRcuLock<Ssl_acceptor_context_data>;

  Ssl_acceptor_context_data_lock *lock_;   // ★ RCU 锁与被保护数据容器 1:1 绑定

  friend class Lock_and_access_ssl_acceptor_context;
  friend class TLS_channel;
};

extern Ssl_acceptor_context_container *mysql_main;    // 主连接通道
extern Ssl_acceptor_context_container *mysql_admin;   // 管理连接通道
```

```cpp
// ssl_acceptor_context_operator.cc:32
Ssl_acceptor_context_container::Ssl_acceptor_context_container(
    Ssl_acceptor_context_data *data)
    : lock_(nullptr) {
  lock_ = new Ssl_acceptor_context_data_lock(data);   // ★ 此后 data 所有权归 RCU
}

Ssl_acceptor_context_container::~Ssl_acceptor_context_container() {
  if (lock_ != nullptr) delete lock_;   // ★ delete → ~MyRcuLock() → 阻塞等读者清零后 delete 旧 data
  lock_ = nullptr;
}

void Ssl_acceptor_context_container::switch_data(
    Ssl_acceptor_context_data *new_data) {
  if (lock_ != nullptr) lock_->write_wait_and_delete(new_data);   // ★ 唯一写路径
}
```

**写路径触发点：`ALTER INSTANCE RELOAD TLS` / `FLUSH SSL`**（`:100`）：

```cpp
void TLS_channel::singleton_flush(Ssl_acceptor_context_container *container,
                                  std::string channel,
                                  Ssl_init_callback *callbacks,
                                  enum enum_ssl_init_error *error, bool force) {
  if (container == nullptr) return;
  Ssl_acceptor_context_data *news =
      new Ssl_acceptor_context_data(channel, true, callbacks, false, error);
  //  ★★ "Read-Copy" 的 Copy：完整构造好新对象，此刻还没人看得见它
  if (*error != SSL_INITERR_NOERROR && !force) { delete news; return; }
  (void)container->switch_data(news);   // ★★ "Update"：原子换入 + 等宽限期 + 回收旧对象
  return;
}
```

**★ 全局最长的一处读临界区**（`sql/auth/sql_authentication.cc:2887`）——跨越 TLS 握手 + 阻塞网络读：

```cpp
  if (protocol->has_client_capability(CLIENT_SSL)) {
    ...
    /*
      We need to make sure that reference count for
      SSL context is kept till the end of function
    */                                    // ★ 注释直白说明：这就是 RCU 读者计数必须保住的原因
    std::unique_ptr<Lock_and_access_ssl_acceptor_context> context;
    //  ★ 用 unique_ptr 而非栈对象，是为了精确控制 RC 生命周期
    if (thd->is_admin_connection() && g_admin_ssl_configured == true)
      context = std::make_unique<Lock_and_access_ssl_acceptor_context>(mysql_admin);
    else
      context = std::make_unique<Lock_and_access_ssl_acceptor_context>(mysql_main);

    /* Do the SSL layering. */
    if (!context.get()->have_ssl()) return packet_error;
    if (sslaccept(*(context.get()), protocol->get_vio(),
                  protocol->get_net()->read_timeout, &errptr)) {
      return packet_error;               // ★★ 完整 TLS 握手（多轮网络往返）都在读临界区内
    }
    int rc = protocol->read_packet();    // ★★ 阻塞式网络读也在读临界区内
    ...
  }   // ★ 2957 行离开作用域 → unique_ptr 析构 → ReadLock 析构 → rcu_end_read()
```

> ★★ **实战含义**：一个**慢客户端 / 卡住的 TLS 握手会阻塞 `ALTER INSTANCE RELOAD TLS`**，而且写者每次只睡 10ms 地无限轮询——表现为 `ALTER INSTANCE RELOAD TLS` **永久挂住**。这是这个原语最大的使用风险。
>
> ★ 下游读侧共 21 处（`mysqld.cc:5466`、`sql_authentication.cc:1625/1628/2895/2898/2901`、`ssl_acceptor_context_iterator.cc:58` 等）。

**被保护的负载类**（`sql/ssl_acceptor_context_data.h:104`）——`final` + 禁拷贝禁移动：

```cpp
class Ssl_acceptor_context_data final {
 public:
  ...
 protected:
  /* Disable copy/assignment */
  Ssl_acceptor_context_data(const Ssl_acceptor_context_data &) = delete;
  Ssl_acceptor_context_data operator=(const Ssl_acceptor_context_data &) = delete;
  /* Disable move constructs */
  Ssl_acceptor_context_data(Ssl_acceptor_context_data &&) = delete;
  Ssl_acceptor_context_data operator=(Ssl_acceptor_context_data &&) = delete;
 private:
  std::string channel_;
  struct st_VioSSLFd *ssl_acceptor_fd_;   // ★ RCU 真正保护的核心：SSL_CTX 句柄
  SSL *acceptor_;
  ...
};
```

**边界总结（选它必须同时满足）**：

| 前提 | 不满足的后果 |
|---|---|
| 读临界区极短且严格作用域 | 写者永久阻塞（无超时） |
| 写极其稀有 | 每次写都进 10ms 轮询，吞吐崩溃 |
| 新对象必须先完整构造再发布 | 读者看到半初始化对象 |
| `ReadLock` 析构后指针立即失效，绝不缓存 | use-after-free |
| 负载对象可 `delete` 且外部不再引用 | 双重释放 |
| **必须有"无读者"的空档期** | 这是 doxygen 明写的核心假设；持续有读者则永不回收 |

| 维度 | Linux 内核完整 RCU | MySQL `MyRcuLock` |
|------|-------------------|-------------------|
| 读者注册 | 不需要（quiescent state 检测） | **显式计数**（`rcu_read`/`rcu_end_read`，漏配对即灾难） |
| grace period 判定 | 检测所有 CPU 的静默状态 | `while (readers > 0) my_sleep(10000)` 轮询 |
| 持续读负载 | 正常 | **写者一直等**，甚至永远等不到 → 旧对象**泄漏**（:212） |
| 读者 API 纪律 | 无 | 必须 RAII 配对（`ReadLock` 防漏） |

**使用边界**（三条）：

1. **只用于"读负载有间歇"的全局**——SSL context 符合（连接握手之间有空隙）；持续饱和读的全局（如 buffer pool 页指针）绝不能用。
2. **写极低频**——写者要复制整个对象 + 轮询等待；高频写会退化。
3. **读者必须用 `ReadLock` RAII**——漏掉 `rcu_end_read` 会让读者计数永久非零，之后所有写者永久卡死 + 泄漏。

## 六、单元测试揭示的使用前提

**真名 `unittest/gunit/my_rcu_lock-t.cc`**（CMake 注册项 `my_rcu_lock`，`unittest/gunit/CMakeLists.txt:154`）。核心两段：

```cpp
class payload_s {
 public:
  payload_s(const char *a, const char *b, const char *c)
      : d1(a), d2(b), d3(c) {}
  payload_s(const payload_s &other) = default;
  const char *d1;
  const char *d2;
  const char *d3;
};

typedef MyRcuLock<payload_s> MyRcuLockTest;   // ★ 三个 const char*，用来验证"读到的一定是同一代完整对象"

  static void rcu_reader(size_t reps) {
    for (size_t i = 0; i < reps; i++) {
      {
        MyRcuLockTest::ReadLock rl(lock);       // ★ 读临界区只有最内层这个 {}
        const payload_s *ptr = rl;              // 经 operator const T*() 隐式转换取出
        EXPECT_FALSE(ptr->d1[0] != 'a' || ptr->d2[0] != 'b' ||
                     ptr->d3[0] != 'c');        // ★★ 不变式：三个字段必须同代，绝不撕裂
      }                                         // ★★ 离开作用域即 rcu_end_read() —— 写者能等到 0 的前提
      reads.fetch_add(1, std::memory_order_relaxed);
    }
  }

  static void rcu_writer(size_t reps, time_t waitms) {
    for (size_t i = 0; i < reps; i++) {
      payload_s *newp = new payload_s("a", "b", "c");   // Copy：先完全构造好

      bool ret = lock->write_wait_and_delete(newp);     // Update：换入 + 等宽限期 + delete 旧
      EXPECT_EQ(ret, false);                            // ★ 断言必然成功回收（因为 wait_for_no_readers 恒 false）
      /*
        RCU works best with relatively infrequent writes compared to reads.
        Trying to simulate this by spacing out the writes via adding a 100ms
        waits.
        It will work without this too, but will not be testing the code in
        "normal" conditions.
      */                                                // ★★ 测试自陈：写必须低频，否则测的不是"正常工况"
      my_sleep(waitms);
      writes.fetch_add(1, std::memory_order_relaxed);
    }
  }
```

```cpp
TEST_F(my_rcu_lock_test, multi_threaded_run) {
  // Capping this at 300 since a std::system_error will be thrown on
  // i686 when creating more than ~400 threads.
  constexpr size_t NUM_READERS = 300;   // ★★ 读:写 = 300:10
  std::thread readerts[NUM_READERS];
  constexpr size_t NUM_WRITERS = 10;
  std::thread writerts[NUM_WRITERS];
  ...
    for (auto &rt : readerts) rt = std::thread(rcu_reader, 100000);   // 每读者 10 万次读
    for (auto &wt : writerts) wt = std::thread(rcu_writer, 5, 100);   // 每写者只写 5 次，间隔 100ms
  ...
  ASSERT_EQ(reads.load(), 100000 * NUM_READERS);   // ★ 3000 万次读全完成 → 读者从不等待写者
  ASSERT_EQ(writes.load(), NUM_WRITERS * 5);       // ★ 50 次写全完成 → 写者最终总能等到 0
}
```

**九条前提（逐条有源码/测试依据）**：

1. **读临界区必须极短且严格作用域** —— `rcu_reader` 把 `ReadLock` 放在最内层 `{}`。
2. **写必须稀有** —— 注释原文 *"RCU works best with relatively infrequent writes"*；测试读写比 ≈ **60000:1**。
3. **新对象必须完全构造好才能发布** —— `rcu_writer` 先 `new payload_s(...)` 再 `write_wait_and_delete`。
4. **`ReadLock` 析构后指针立即失效** —— 测试从不把 `ptr` 带出作用域。
5. **读侧零阻塞** —— `ASSERT_EQ(reads.load(), 100000 * NUM_READERS)` 证明。
6. **写侧必然成功回收** —— `EXPECT_EQ(ret, false)` 证明（因为 `wait_for_no_readers` 恒 false）。
7. **读者绝不会看到撕裂/半初始化对象** —— `EXPECT_FALSE(ptr->d1[0] != 'a' || ...)` 是唯一的数据正确性断言。
8. **销毁顺序有要求** —— `TearDownTestCase()` 在**所有线程 join 之后**才 `delete lock`，否则析构里的 `write_wait_and_delete(nullptr)` 会死等。
9. **负载对象需可 `delete` 且不可被外部再引用** —— `payload_s` 无虚函数；生产的 `Ssl_acceptor_context_data` 则 `final` + 4 个 `= delete`。

```cpp
  static void SetUpTestCase() {
    lock = new MyRcuLockTest(new payload_s("a", "b", "c"));
    reads = 0; writes = 0;
  }
  static void TearDownTestCase() { delete lock; }   // ★ 此时必须已无读者
```

**测试里的 cache-line padding**（模拟生产布局）：

```cpp
  static MyRcuLockTest *lock;
  static unsigned char p1[128];        // ★ 手动 128 字节 padding
  static std::atomic<long> reads;
  static unsigned char p2[128];
  static std::atomic<long> writes;
  static unsigned char p3[128];
```

`unittest/gunit/my_rcu_lock-t.cc`：

```cpp
// :110-129 多线程压力测试
constexpr size_t NUM_READERS = 300;   // 300 个读者线程
constexpr size_t NUM_WRITERS = 10;    // 10 个写者线程
for (auto &rt : readerts) rt = std::thread(rcu_reader, 100000);   // 各读 10 万次
for (auto &wt : writerts) rt = std::thread(rcu_writer, 5, 100);   // 各写 5 次，写间睡 100ms
```

测试注释（:82-87）点破了 RCU 的使用前提：

> "RCU works best with relatively **infrequent writes** compared to reads. Trying to simulate this by spacing out the writes via adding a **100ms** waits."

**写者之间刻意睡 100ms**——正是为了制造"频繁的无读者时刻"，让 `wait_for_no_readers` 能等到计数归零。这从侧面印证了第五节的使用边界：没有读间隙，写者就卡死。

读者断言（:68-69）验证了 RCU 的核心保证：**读者读到的永远是完整一致的版本**（`"a"/"b"/"c"` 三个字段一致），不会看到"换了一半"的对象——因为写者是"复制新对象完整构造后才换指针"。

---

## 参考

- **McKenney et al.《Read-Copy Update》(2001)** / Linux kernel `Documentation/RCU/` —— RCU 的原始设计与 quiescent state 机制
- **McKenney《Is Parallel Programming Hard, And, If So, What Can You Do About It?》** —— RCU 章节
- MySQL 8.0 源码：`include/my_rcu_lock.h`（实现）、`sql/ssl_acceptor_context_operator.h`（使用）、`unittest/gunit/my_rcu_lock-t.cc`（测试）
