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
