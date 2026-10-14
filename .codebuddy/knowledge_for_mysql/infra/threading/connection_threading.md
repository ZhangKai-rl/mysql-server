# 连接线程模型深度解析

> 基于 MySQL 8.0.39 源码，剖析**一条客户端连接从被 accept 到线程回收**的完整生命周期：`Connection_acceptor` 接纳循环、`Channel_info` 来源抽象、`Per_thread_connection_handler` 的**线程缓存三件套**、`block_until_new_connection()` 的无超时等待、`max_connections` 的**两段式校验**，以及它与认证子系统的接缝。
>
> **边界**：本篇讲 **server 层**的**线程与连接管理**（谁接纳、谁执行、如何复用、如何回收）——即全栈线程模型里的"闸门 0 与 0.5"；连接建立后跑的**握手协议与插件协商**见 [../auth/authentication.md](../../server/auth/authentication.md)；**网络 IO 抽象**见 [vio.md](../../server/infra/vio.md)；**mysqld 启动期如何初始化这些对象**见 [../lifecycle/02_startup.md](../../server/lifecycle/02_startup.md)；权限判定见 [../auth/authorization.md](../../server/auth/authorization.md)。
>
> ★ **跨层全景**：线程不止 server 层——InnoDB 有 26 种后台线程，且线程与并发控制、PFS 三向耦合（一条 SQL 卡住时"挂"在 MDL_wait / `srv_slot` / `sync_array` 三种不同结构上）。**全栈线程清单 + 耦合关系 + `thread/innodb/*` 观测名见 [../../infra/threading_architecture.md](threading_architecture.md)**；本篇只聚焦连接线程本身。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 社区版默认采用 **one-thread-per-connection**（每连接一线程）模型：一条客户端连接 = 一个 `THD` 对象 = 一条 OS 线程。连接断开后线程**不销毁**，而是进入线程缓存等待复用。

管理这套模型的源码集中在 `sql/conn_handler/`（约 4300 行）：

| 文件 | 职责 |
|---|---|
| `connection_acceptor.h` | 接纳主循环（模板类，参数化 listener 类型） |
| `socket_connection.cc/.h` | `Mysqld_socket_listener`：poll/select + accept |
| `channel_info.h/.cc` | 连接来源的抽象基类 |
| `connection_handler_manager.h/.cc` | 单例管理器：连接计数、handler 分派、插件装载 |
| `connection_handler_per_thread.cc` | **默认实现**：线程缓存 + 复用 |
| `connection_handler_one_thread.cc` | 调试用的单线程模式 |
| `plugin_connection_handler.h` | 供外部插件（如线程池）接管的包装器 |
| `named_pipe_connection.cc` / `shared_memory_connection.cc` | Windows 专属来源 |

### 用途

回答三类实际问题：

1. **`Threads_cached` / `Threads_created` 为什么会长？** → 线程缓存的命中与回退
2. **`max_connections` 打满时为什么 root 还能连进来？** → 两段式校验 + 管理接口
3. **认证失败的连接会不会拖垮线程？** → 不会，线程回池（见[接缝](#六与认证子系统的接缝)）

### 版本演进

| 版本 | 变化 |
|---|---|
| 早期～至今 | `one-thread-per-connection` 一直是社区版默认；`no-threads` 供调试 |
| 5.x | 引入 `Channel_info` 抽象，把 TCP / UDS / named pipe / shared memory 统一 |
| 8.0 | 引入**管理接口**（`admin_address` / `admin_port`），配合 `SERVICE_CONNECTION_ADMIN` 形成"打满也能进"的第二通道 |
| 8.0 | `Connection_handler_manager` 独立成单例，暴露 `my_thread_scheduler_service` 供线程池插件接管 |

---

## 理论基础

### 设计思想与权衡

**核心矛盾**：线程创建的代价 vs 并发连接的成本。

**选定方案**：**预先创建 + 缓存复用**——线程服务完一条连接后**不退出**，阻塞在条件变量上等待下一次分配；新连接来了优先唤醒缓存线程，只有缓存空了才真的 `pthread_create`。

**为什么是"每连接一线程"而不是线程池**（这是 8.0 社区版最容易被误解的一点）：

| 方案 | 优点 | 代价 |
|---|---|---|
| **每连接一线程**（社区版默认） | 实现简单；一条连接阻塞（锁等待、大查询）**不影响**其他连接；`THD` 可以放线程局部 | 连接数多时线程数线性增长；上下文切换开销大 |
| **线程池**（企业版插件） | 线程数可控，高并发下上下文切换少 | 一条慢查询可能占满线程导致其他连接饿死；需要精细的调度与超时策略 |

★ **重要事实**：MySQL 8.0.39 社区版源码树里**没有任何线程池实现**，只有**完整的插件接入机制**：

- 社区版 `plugin/` 目录下无 `thread_pool`；
- `Connection_handler_manager::init()` 的 switch **只有两个 case**（`SCHEDULER_ONE_THREAD_PER_CONNECTION` / `SCHEDULER_NO_THREADS`），第三分支是 `assert(false)`；
- 但接口一应俱全：`my_thread_scheduler_service`（`sql/sql_plugin_services.h`）、`struct Connection_handler_functions`（`include/mysql/thread_pool_priv.h`）、以及 `thread_handling_names` 里预留的第三个显示值 `"loaded-dynamically"`。

即：**位置留好了，实现留给外部/企业版插件**。

**为什么要有线程缓存**：`pthread_create` 的代价在"短连接风暴"下会主导延迟。缓存把"创建/销毁"摊成"阻塞/唤醒"。代价是一批常驻线程占内存（每条线程栈由 `my_thread_stack_size` 决定）。

**为什么线程必须是 DETACHED**：见[线程属性](#线程创建detached-与-psi-key)——因为线程可能被缓存复用，生命周期与连接完全解耦，**没有 join 点**。这也决定了缓存必须靠条件变量（而非 join）实现。

### 理论溯源

`Connection_handler` 是**策略模式**（`add_connection` / `get_max_threads` 两个纯虚函数），`Connection_handler_manager` 持有基类指针做分派；`Channel_info` 是**工厂方法**（基类 `create_thd()` 调用纯虚 `create_and_init_vio()`，子类提供不同的 Vio 构造）；`Connection_acceptor` 是**模板方法 + 策略参数化**（同一套循环代码服务三种 listener）。

### 算法与数据结构

**线程缓存的三个状态量**（这是全篇最核心的数据结构）：

| 名字 | 类型 | 语义 |
|---|---|---|
| `blocked_pthread_count` | `static ulong` | 当前阻塞在 `COND_thread_cache` 上、**可复用**的线程**数量**（= 状态变量 `Threads_cached`） |
| `waiting_channel_info_list` | `static std::list<Channel_info *> *` | **待处理的连接队列**：找不到空闲线程时把 `Channel_info*` 压进去，由阻塞线程醒来后 `pop_front()` 取走 |
| `wake_pthread` | 文件内 `static uint` | **唤醒令牌**："已发出 signal 但尚未被消费"的计数，保证"队列里有几个就唤醒几个"，避免惊群 |

★ **注意这里没有任何"线程对象链表"**——常见资料里说的 `blocked_pthread_list` / `waiting_pthread_list` **在 8.0 里都不存在**。缓存是"计数 + 队列 + 令牌"三件套，线程本身只是阻塞在条件变量上，**不被任何容器持有**。

配套同步原语：`LOCK_thread_cache`（保护以上全部）、`COND_thread_cache`（阻塞线程等在这里）、`COND_flush_thread_cache`（收缩缓存时等"被赶走的线程真的走了"）。

---

## 核心实现

### 主链路

```
mysqld_main()                                    [sql/mysqld.cc]
 ├─ network_init()
 │    ├─ new Mysqld_socket_listener(...)          [socket_connection.cc]
 │    ├─ new Connection_acceptor<Mysqld_socket_listener>(listener)
 │    └─ init_connection_acceptor() → Mysqld_socket_listener::setup_listener()
 ├─ Connection_handler_manager::init()            [connection_handler_manager.cc]
 └─ mysqld_socket_acceptor->connection_event_loop()   ← 主线程（非 Windows），永不返回
      └─ Mysqld_socket_listener::listen_for_connection_event()
           ├─ poll() / select()
           ├─ get_listen_socket()      （admin socket 优先检查）
           ├─ accept_connection() → mysql_socket_accept()
           └─ new Channel_info_local_socket / Channel_info_tcpip_socket
      └─ Connection_handler_manager::process_new_connection(channel_info)
           ├─ check_and_incr_conn_count(channel_info->is_admin_connection())   ← 第一道闸
           └─ m_connection_handler->add_connection(channel_info)
                    ↓
      Per_thread_connection_handler::add_connection()
           ├─ check_idle_thread_and_enqueue_connection()   ← 命中缓存则到此为止
           └─ mysql_thread_create(key_thread_one_connection, ..., handle_connection, arg)
                    ↓ （新线程）
      handle_connection(void *arg)                [connection_handler_per_thread.cc]
           ├─ my_thread_init()                    （只在线程启动时调一次）
           └─ for (;;) {
                ├─ init_new_thd(channel_info)     ← THD 在**新线程内部**创建
                ├─ thd_manager->add_thd(thd)
                ├─ thd_prepare_connection(thd) ──► 认证（见 authentication.md）
                ├─ while (thd_connection_alive(thd)) do_command(thd)
                ├─ end_connection(thd) / close_connection(...)
                ├─ remove_thd / dec_connection_count / delete thd
                └─ block_until_new_connection()   ← 回池等待，拿不到则退出
              }
           └─ my_thread_end(); my_thread_exit()
```

> **Windows 差异**：非 Windows 是**主线程**直接跑接纳循环；Windows 下 `setup_conn_event_handler_threads()` 为每个 acceptor 各起一个线程（socket / named pipe / shared memory 三条）。

### 一、连接接纳：`Connection_acceptor` 与 listener

`Connection_acceptor` 是个模板类，存在的唯一意义是**让同一套循环代码服务不同的 listener**：

```cpp
// sql/conn_handler/connection_acceptor.h
template <typename Listener>
class Connection_acceptor {
  Listener *m_listener;
 public:
  Connection_acceptor(Listener *listener) : m_listener(listener) {}

  bool init_connection_acceptor() { return m_listener->setup_listener(); }

  void connection_event_loop() {
    Connection_handler_manager *mgr = Connection_handler_manager::get_instance();
    while (!connection_events_loop_aborted()) {
      Channel_info *channel_info = m_listener->listen_for_connection_event();
      if (channel_info != nullptr) mgr->process_new_connection(channel_info);
    }
  }
  ...
};
```

**循环结构**：单线程 `while` + 阻塞式 `listen_for_connection_event()` + **同步** `process_new_connection()`。注意接纳是**串行**的——accept 一个、处理完、再 accept 下一个（处理指"分配线程"，不是"跑完整个连接"）。

**★ 多路复用用的是 `poll`，fallback `select`，没有 epoll/kqueue**：

```cpp
// sql/conn_handler/socket_connection.h
#ifdef HAVE_POLL
  struct poll_info_t {
    std::vector<struct pollfd> m_fds;
    std::vector<MYSQL_SOCKET> m_pfs_fds;
  };
  poll_info_t m_poll_info;
#else
  struct select_info_t {
    fd_set m_read_fds, m_client_fds;
    my_socket m_max_used_connection;
    ...
  };
  select_info_t m_select_info;
#endif
```

```cpp
Channel_info *Mysqld_socket_listener::listen_for_connection_event() {
#ifdef HAVE_POLL
  int retval = poll(&m_poll_info.m_fds[0], m_socket_vector.size(), -1);
#else
  m_select_info.m_read_fds = m_select_info.m_client_fds;
  int retval = select((int)m_select_info.m_max_used_connection,
                      &m_select_info.m_read_fds, 0, 0, 0);
#endif
  ...
  const Listen_socket *listen_socket = get_listen_socket();
  ...
  if (accept_connection(listen_socket->m_socket, &connect_sock)) { ...; return nullptr; }
  ...
  if (listen_socket->m_socket_type == Socket_type::UNIX_SOCKET)
    channel_info = new (std::nothrow) Channel_info_local_socket(connect_sock);
  else
    channel_info = new (std::nothrow) Channel_info_tcpip_socket(
        connect_sock, (listen_socket->m_socket_interface ==
                       Socket_interface_type::ADMIN_INTERFACE));
  return channel_info;
}
```

要点：
- **超时是 `-1`（无限阻塞）**，select 分支同样是无限等待；
- `get_listen_socket()` 扫 `revents & POLLIN`，**admin 接口 socket 被优先检查**；
- `accept_connection()` 内部重试 `MAX_ACCEPT_RETRY` 次，遇到 `EMFILE/ENFILE` 会 `sleep(1)` 让出（防止描述符耗尽时打满 CPU）。

### 二、`Channel_info`：连接来源的抽象

基类把"accept 拿到的那个平台相关句柄"与"如何从它造出一个能用的 `THD`"封装起来，让上层完全不感知来源：

```cpp
// sql/conn_handler/channel_info.h
/**
  This abstract base class represents connection channel information
  about a new connection. Its subclasses encapsulate differences
  between different connection channel types.

  Currently we support local and TCP/IP sockets (all platforms),
  named pipes and shared memory (Windows only).
*/
class Channel_info {
  ulonglong prior_thr_create_utime;
 protected:
  virtual Vio *create_and_init_vio() const = 0;
  Channel_info() : prior_thr_create_utime(0) {}
 public:
  virtual ~Channel_info() = default;
  virtual THD *create_thd();
  virtual void send_error_and_close_channel(uint errorcode, int error, bool senderror);
  ulonglong get_prior_thr_create_utime() const { return prior_thr_create_utime; }
  void set_prior_thr_create_utime() { prior_thr_create_utime = my_micro_time(); }
  virtual bool is_admin_connection() const { return false; }
};
```

★ **四个子类都不在 `channel_info.h` 里**，而是各自定义在自己 listener 的 `.cc` 中（文件内部可见）：

| 子类 | 文件 |
|---|---|
| `Channel_info_local_socket` | `socket_connection.cc` |
| `Channel_info_tcpip_socket` | `socket_connection.cc` |
| `Channel_info_named_pipe` | `named_pipe_connection.cc`（Windows） |
| `Channel_info_shared_mem` | `shared_memory_connection.cc`（Windows） |

基类的 `create_thd()` 用到了工厂方法的经典形态——基类定流程、子类补差异：

```cpp
THD *Channel_info::create_thd() {
  DBUG_EXECUTE_IF("simulate_resource_failure", return nullptr;);
  Vio *vio_tmp = create_and_init_vio();        // ← 子类实现
  if (vio_tmp == nullptr) return nullptr;
  THD *thd = new (std::nothrow) THD;
  if (thd == nullptr) { vio_delete(vio_tmp); return nullptr; }
  thd->get_protocol_classic()->init_net(vio_tmp);
  return thd;
}
```

`Channel_info_local_socket` 的 override 里能看到"本地 socket 的 host 被预设成 `my_localhost`"：

```cpp
  THD *create_thd() override {
    THD *thd = Channel_info::create_thd();
    if (thd != nullptr) {
      init_net_server_extension(thd);
      thd->security_context()->set_host_ptr(my_localhost, strlen(my_localhost));
    }
    return thd;
  }
```

`Channel_info_tcpip_socket` 额外带 `m_is_admin_conn`，override `is_admin_connection()`——**这正是后面 `max_connections` 判定能区分管理连接的原因**。`prior_thr_create_utime` 则用于统计 `Slow_launch_threads`。

### 三、`Per_thread_connection_handler`：线程缓存三件套

#### 类定义

```cpp
// sql/conn_handler/connection_handler_impl.h
class Per_thread_connection_handler : public Connection_handler {
  bool check_idle_thread_and_enqueue_connection(Channel_info *channel_info);

  static std::list<Channel_info *> *waiting_channel_info_list;
  static mysql_mutex_t LOCK_thread_cache;
  static mysql_cond_t COND_thread_cache;
  static mysql_cond_t COND_flush_thread_cache;

 public:
  static ulong blocked_pthread_count;   // Protected by LOCK_thread_cache.
  static ulong slow_launch_threads;
  static bool shrink_cache;
  static ulong max_blocked_pthreads;    // ← thread_cache_size 的绑定目标

  static void init();
  static void destroy();
  static void kill_blocked_pthreads();
  static Channel_info *block_until_new_connection();
  static void modify_thread_cache_size(const ulong thread_cache_size);

 protected:
  bool add_connection(Channel_info *channel_info) override;
  uint get_max_threads() const override;   // return max_connections;
};
```

#### "复用还是新建"：一个严格大于号

```cpp
bool Per_thread_connection_handler::add_connection(Channel_info *channel_info) {
  int error = 0;
  my_thread_handle id;

  if (!check_idle_thread_and_enqueue_connection(channel_info)) return false;

  /*
    There are no idle threads available to take up the new
    connection. Create a new thread to handle the connection
  */
  channel_info->set_prior_thr_create_utime();
  error = mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                              handle_connection, (void *)channel_info);
  if (error) {
    connection_errors_internal++;
    if (!create_thd_err_log_throttle.log())
      LogErr(ERROR_LEVEL, ER_CONN_PER_THREAD_NO_THREAD, error);
    channel_info->send_error_and_close_channel(ER_CANT_CREATE_THREAD, error, true);
    Connection_handler_manager::dec_connection_count();
    return true;
  }
  Global_THD_manager::get_instance()->inc_thread_created();
  return false;
}

bool Per_thread_connection_handler::check_idle_thread_and_enqueue_connection(
    Channel_info *channel_info) {
  bool res = true;
  mysql_mutex_lock(&LOCK_thread_cache);
  if (Per_thread_connection_handler::blocked_pthread_count > wake_pthread) {
    waiting_channel_info_list->push_back(channel_info);
    wake_pthread++;
    mysql_cond_signal(&COND_thread_cache);
    res = false;                       // ← 已交给空闲线程，不需要新建
  }
  mysql_mutex_unlock(&LOCK_thread_cache);
  return res;
}
```

★ 判定式是 `blocked_pthread_count > wake_pthread`（**严格大于**）——意思是"存在**尚未被预约**的空闲线程"。`wake_pthread` 是"已 signal 但还没人消费"的令牌，两者之差才是真正可用的空闲线程数。

另一个细节：这里只 `mysql_cond_signal()`（唤醒**一个**），不是 broadcast——与令牌机制配合，正好"队列里有 N 个就唤醒 N 次"。

#### 线程创建：DETACHED 与 PSI key

```cpp
error = mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                            handle_connection, (void *)channel_info);
```

- 函数名是 **`mysql_thread_create`**（PSI 宏，内部才退化到 `my_thread_create`），不是 `my_thread_create`；
- 线程属性 `connection_attrib` 在 `mysqld.cc` 里初始化，关键是 **`MY_THREAD_CREATE_DETACHED`**：
  ```cpp
  (void)my_thread_attr_init(&connection_attrib);
  my_thread_attr_setdetachstate(&connection_attrib, MY_THREAD_CREATE_DETACHED);
  #ifndef _WIN32
  pthread_attr_setscope(&connection_attrib, PTHREAD_SCOPE_SYSTEM);
  #endif
  ```
  ★ **DETACHED 是理解整个模型的钥匙**：线程退出即自动回收，**没有任何 join 点**。所以"线程复用"在技术上只能靠条件变量实现——线程自己不退出，循环着等下一个 `Channel_info`。
- PSI key `key_thread_one_connection` 注册为 `{"one_connection", "connection", ...}`，所以 `performance_schema.threads` 里连接线程的 NAME 是 **`thread/sql/one_connection`**。

### 四、`block_until_new_connection()`：无超时的等待

```cpp
/**
  Block the current pthread for reuse by new connections.

  @retval NULL   Too many pthreads blocked already or shutdown in progress.
  @retval !NULL  Pointer to Channel_info object representing the new connection
                 to be served by this pthread.
*/
Channel_info *Per_thread_connection_handler::block_until_new_connection() {
  Channel_info *new_conn = nullptr;
  mysql_mutex_lock(&LOCK_thread_cache);
  if (blocked_pthread_count < max_blocked_pthreads && !shrink_cache) {
    /* Don't kill the pthread, just block it for reuse */
    DBUG_PRINT("info", ("Blocking pthread for reuse"));

    /*
      mysys_var is bound to the physical thread,
      so make sure mysys_var->dbug is reset to a clean state
      before picking another session in the thread cache.
    */
    DBUG_POP();
    assert(!_db_is_pushed_());

    blocked_pthread_count++;
    while (!connection_events_loop_aborted() && !wake_pthread && !shrink_cache)
      mysql_cond_wait(&COND_thread_cache, &LOCK_thread_cache);
    blocked_pthread_count--;

    if (shrink_cache && blocked_pthread_count <= max_blocked_pthreads) {
      mysql_cond_signal(&COND_flush_thread_cache);
    }

    if (wake_pthread) {
      wake_pthread--;
      if (!waiting_channel_info_list->empty()) {
        new_conn = waiting_channel_info_list->front();
        waiting_channel_info_list->pop_front();
      } else {
        assert(0);  // We should not get here.
      }
    }
  }
  mysql_mutex_unlock(&LOCK_thread_cache);
  return new_conn;
}
```

★ **没有超时**：用的是 `mysql_cond_wait`（不是 `timedwait`）。退出只有三个条件——`connection_events_loop_aborted()`（shutdown）、`wake_pthread != 0`（分到新连接）、`shrink_cache`（缓存被收缩）。

**入池条件**是 `blocked_pthread_count < max_blocked_pthreads && !shrink_cache`：缓存已满或正在收缩时**直接返回 nullptr，线程真的退出**。这就是"线程缓存满了就销毁线程"的落点。

那段 `DBUG_POP()` 与注释值得注意：`mysys_var` 绑定的是**物理线程**而不是 THD，所以复用前必须手动清 DBUG 栈——这是"线程复用"留下的一个必须手工处理的副作用。

#### 缓存收缩

```cpp
void Per_thread_connection_handler::modify_thread_cache_size(
    const ulong thread_cache_size) {
  mysql_mutex_lock(&LOCK_thread_cache);
  if (thread_cache_size >= blocked_pthread_count) {
    mysql_mutex_unlock(&LOCK_thread_cache);
    return;
  }
  shrink_cache = true;
  if (thread_cache_size == 0) {
    mysql_cond_broadcast(&COND_thread_cache);
  } else {
    ulong num_threads = blocked_pthread_count - thread_cache_size;
    for (ulong i = 0; i < num_threads; i++)
      mysql_cond_signal(&COND_thread_cache);
  }
  // Wait until threads have been unblocked from thread cache.
  while (blocked_pthread_count > thread_cache_size)
    mysql_cond_wait(&COND_flush_thread_cache, &LOCK_thread_cache);
  shrink_cache = false;
  mysql_mutex_unlock(&LOCK_thread_cache);
}

void Per_thread_connection_handler::kill_blocked_pthreads() {
  modify_thread_cache_size(0);
}
```

即：调小 `thread_cache_size` 时**精确唤醒多出来的那几个**让它们退出，并在 `COND_flush_thread_cache` 上等它们真走了才返回。`kill_blocked_pthreads()` 用于 shutdown。

### 五、`handle_connection()`：复用循环的全貌

```cpp
/**
  Thread handler for a connection

  @param arg   Connection object (Channel_info)

  This function (normally) does the following:
  - Initialize thread
  - Initialize THD to be used with this thread
  - Authenticate user
  - Execute all queries sent on the connection
  - Take connection down
  - End thread  / Handle next connection using thread from thread cache
*/
extern "C" {
static void *handle_connection(void *arg) {
  Global_THD_manager *thd_manager = Global_THD_manager::get_instance();
  Connection_handler_manager *handler_manager =
      Connection_handler_manager::get_instance();
  Channel_info *channel_info = static_cast<Channel_info *>(arg);
  bool pthread_reused [[maybe_unused]] = false;

  if (my_thread_init()) {
    connection_errors_internal++;
    channel_info->send_error_and_close_channel(ER_OUT_OF_RESOURCES, 0, false);
    handler_manager->inc_aborted_connects();
    Connection_handler_manager::dec_connection_count();
    delete channel_info;
    my_thread_exit(nullptr);
    return nullptr;
  }

  for (;;) {
    THD *thd = init_new_thd(channel_info);
    if (thd == nullptr) {
      connection_errors_internal++;
      handler_manager->inc_aborted_connects();
      Connection_handler_manager::dec_connection_count();
      break;  // We are out of resources, no sense in continuing.
    }
    ...
    thd_manager->add_thd(thd);

    if (thd_prepare_connection(thd))
      handler_manager->inc_aborted_connects();
    else {
      while (thd_connection_alive(thd)) {
        if (do_command(thd)) break;
      }
      end_connection(thd);
    }
    close_connection(thd, 0, false, false);

    thd->get_stmt_da()->reset_diagnostics_area();
    thd->release_resources();
    thd_manager->remove_thd(thd);
    Connection_handler_manager::dec_connection_count();
    ...
    delete thd;
    ...
    // Server is shutting down so end the pthread.
    if (connection_events_loop_aborted()) break;

    channel_info = Per_thread_connection_handler::block_until_new_connection();
    if (channel_info == nullptr) break;
    pthread_reused = true;
    ...
  }

  my_thread_end();
  my_thread_exit(nullptr);
  return nullptr;
}
}  // extern "C"
```

**三个关键观察**：

1. **`my_thread_init()` 在 `for(;;)` 外面只调一次** ⇒ 复用线程不重复初始化 mysys per-thread 状态。原因是 mysqld 的线程是裸 pthread，mysys 的 per-thread 状态（`st_my_thread_var`：错误缓冲、dbug 状态等）必须显式初始化，否则 `my_error()` / `my_malloc()` 会踩空。
2. **THD 在循环内 new / delete** ⇒ **THD 生命周期严格短于线程**。一条线程可以服务 N 个连接，每个连接一个全新 THD。这解释了很多 `THD` 相关的"为什么每轮都要重建"的设计。
3. `handle_connection` 是 `extern "C"` 的 static 函数（pthread 入口 ABI 要求），所以不能写成成员函数。

> 函数真名是 `handle_connection`。`handle_one_connection` **不存在**——它只出现在过时注释和 `sp_head.cc` 的栈样本文本里，容易被误当真名引用。

#### `init_new_thd()`：THD 诞生于新线程内部

```cpp
static THD *init_new_thd(Channel_info *channel_info) {
  THD *thd = channel_info->create_thd();
  if (thd == nullptr) {
    channel_info->send_error_and_close_channel(ER_OUT_OF_RESOURCES, 0, false);
    delete channel_info;
    return nullptr;
  }
  thd->set_new_thread_id();

  if (channel_info->get_prior_thr_create_utime() != 0) {
    /* A pthread was created to handle this connection: increment
       slow_launch_threads counter if it took more than slow_launch_time
       seconds to create the pthread. */
    ulonglong launch_time = thd->start_utime - channel_info->get_prior_thr_create_utime();
    if (launch_time >= slow_launch_time * 1000000ULL)
      Per_thread_connection_handler::slow_launch_threads++;
  }
  delete channel_info;
  ...
  thd_set_thread_stack(thd, (char *)&thd);   // 栈起点 = 第一个局部变量的地址
  thd->store_globals();
  return thd;
}
```

★ **THD 不是由接纳线程创建的**，而是在**服务线程内部**创建 —— 接纳线程只把 `Channel_info*` 传进来。这样 `store_globals()` 才能把 THD 写进正确的线程局部。

`thd_set_thread_stack(thd, (char *)&thd)` 是个巧妙的技巧：用 `handle_connection` **第一个局部变量**（就是 `thd` 自己）的地址作为栈起点，供栈溢出检测用。

### 六、`Connection_handler_manager`：连接计数与两段式校验

#### 单例与分派

```cpp
class Connection_handler_manager {
  static Connection_handler_manager *m_instance;
  static mysql_mutex_t LOCK_connection_count;
  static mysql_cond_t COND_connection_count;
  Connection_handler *m_connection_handler;
  Connection_handler *m_saved_connection_handler;   // 插件接管时保存原 handler
  ulong m_saved_thread_handling;
  ulong m_aborted_connects;
  ulong m_connection_errors_max_connection;
  ...
 public:
  static Connection_handler_manager *get_instance() {
    assert(m_instance != nullptr);
    return m_instance;
  }
  ...
};
```

访问器是 **`get_instance()`**（小 g）；保存 handler 的成员是 **`m_saved_connection_handler`**（不是 `m_saved_handler`）。

基类 `Connection_handler` 只有两个纯虚函数，且是 `protected` + `friend class Connection_handler_manager`——**只有 manager 能调**：

```cpp
class Connection_handler {
 protected:
  friend class Connection_handler_manager;
  virtual bool add_connection(Channel_info *channel_info) = 0;
  virtual uint get_max_threads() const = 0;
};
```

**所有权契约**（`process_new_connection()`）：`add_connection()` 返回 `false` ⇒ `Channel_info` 所有权转移给 handler（由新线程 `delete`）；返回 `true` ⇒ manager 负责 `delete` 并 `inc_aborted_connects()`。

#### 第一道闸：接纳时

```cpp
void Connection_handler_manager::process_new_connection(Channel_info *channel_info) {
  if (connection_events_loop_aborted() ||
      !check_and_incr_conn_count(channel_info->is_admin_connection())) {
    channel_info->send_error_and_close_channel(ER_CON_COUNT_ERROR, 0, true);
    delete channel_info;
    return;
  }
  if (m_connection_handler->add_connection(channel_info)) {
    inc_aborted_connects();
    delete channel_info;
  }
}

bool Connection_handler_manager::check_and_incr_conn_count(bool is_admin_connection) {
  bool connection_accepted = true;
  mysql_mutex_lock(&LOCK_connection_count);
  /*
    Here we allow max_connections + 1 clients to connect
    (by checking before we increment by 1).

    The last connection is reserved for SUPER users. This is
    checked later during authentication where valid_connection_count()
    is called for non-SUPER users only.
  */
  if (connection_count > max_connections && !is_admin_connection) {
    connection_accepted = false;
    m_connection_errors_max_connection++;
  } else {
    ++connection_count;
    if (connection_count > max_used_connections) {
      max_used_connections = connection_count;
      max_used_connections_time = time(nullptr);
    }
  }
  mysql_mutex_unlock(&LOCK_connection_count);
  return connection_accepted;
}
```

#### 第二道闸：认证时

```cpp
// sql/auth/sql_authentication.cc
static inline bool check_restrictions_for_com_connect_command(THD *thd) {
  if (thd->is_admin_connection() &&
      !thd->m_main_security_ctx
           .has_global_grant(STRING_WITH_LEN("SERVICE_CONNECTION_ADMIN")).first) {
    my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "SERVICE_CONNECTION_ADMIN");
    return true;
  }
  if (!(thd->m_main_security_ctx.check_access(SUPER_ACL) ||
        thd->m_main_security_ctx
            .has_global_grant(STRING_WITH_LEN("CONNECTION_ADMIN")).first ||
        thd->m_main_security_ctx
            .has_global_grant(STRING_WITH_LEN("SERVICE_CONNECTION_ADMIN")).first)) {
    if (!Connection_handler_manager::get_instance()->valid_connection_count()) {
      my_error(ER_CON_COUNT_ERROR, MYF(0));   // too many connections
      return true;
    }
  }
  return false;
}
```

★ **这就是"打满了 root 还能连进来"的完整答案**，它是**两段式**的：

| 阶段 | 位置 | 规则 |
|---|---|---|
| 接纳时 | `check_and_incr_conn_count()` | 只要 `is_admin_connection()`（走 `admin_address`/`admin_port` 管理接口）就**允许突破** `max_connections` |
| 认证时 | `check_restrictions_for_com_connect_command()` | 没有 `SUPER` / `CONNECTION_ADMIN` / `SERVICE_CONNECTION_ADMIN` 任一的用户，**二次校验** `valid_connection_count()`，超限则拒 |

即：普通用户即便在接纳阶段"蒙混过关"（计数恰好等于上限时 +1 成功），也会在认证阶段被挡掉。**预留的那个连接是给有管理权限的账户的，不是给任何人的。**

> ★ 一个命名不一致的小坑：头文件里参数名写作 `ignore_max_connection_count`，`.cc` 定义与调用处用的是 `is_admin_connection`。

> ★ `extra_max_connections` **全仓库 0 命中**——那是 Percona/MariaDB 的变量，8.0 社区版用的是上面这套 admin interface + 动态权限。

### 七、与认证子系统的接缝

```
handle_connection()
  ├─ init_new_thd()                    → THD + set_new_thread_id() + store_globals()
  ├─ thd_manager->add_thd(thd)
  ├─ thd_prepare_connection(thd)  ────►  [sql/sql_connect.cc]
  │     ├─ lex_start(thd)
  │     ├─ login_connection(thd)
  │     │     ├─ 把读写超时临时切成 connect_timeout
  │     │     ├─ check_connection(thd)
  │     │     │     ├─ vio_peer_addr() → 反解 IP/host → host_cache
  │     │     │     ├─ acl_check_host()
  │     │     │     └─ acl_authenticate()  ──► 见 authentication.md
  │     │     │            └─ check_restrictions_for_com_connect_command()
  │     │     │                 └─ valid_connection_count()   ← 上面那道第二闸
  │     │     └─ 认证成功后把超时切回 net_read/write_timeout
  │     └─ prepare_new_connection_state(thd)
  ├─ while (thd_connection_alive(thd)) do_command(thd)
  └─ end_connection() / close_connection()
```

`login_connection()` 里有一段很关键的注释与逻辑：

```cpp
static bool login_connection(THD *thd) {
  int error;
  /* Use "connect_timeout" value during connection phase */
  thd->get_protocol_classic()->set_read_timeout(connect_timeout, true);
  thd->get_protocol_classic()->set_write_timeout(connect_timeout);

  error = check_connection(thd);
  thd->send_statement_status();

  if (error) {   // Wrong permissions
    ...
    return true;
  }
  /* Connect completed, set read/write timeouts back to default */
  thd->get_protocol_classic()->set_read_timeout(thd->variables.net_read_timeout);
  thd->get_protocol_classic()->set_write_timeout(thd->variables.net_write_timeout);
  return false;
}
```

★ **`connect_timeout` 只覆盖握手阶段**，认证通过后立刻切回 `net_read_timeout` / `net_write_timeout`。这是"为什么大查询不受 `connect_timeout` 影响"的答案。

#### ★ 认证失败时线程怎么处理：不终止，回池

看 `handle_connection()` 的那段：

```cpp
    if (thd_prepare_connection(thd))
      handler_manager->inc_aborted_connects();
    else {
      while (thd_connection_alive(thd)) { if (do_command(thd)) break; }
      end_connection(thd);
    }
    close_connection(thd, 0, false, false);
```

认证失败（返回 `true`）时：

1. `inc_aborted_connects()` ⇒ **`Aborted_connects++`**；
2. **跳过** `do_command` 循环**和 `end_connection()`** ⇒ 所以 **`Aborted_clients` 不会因为认证失败而增加**（它只在 `end_connection()` 里按 `thd->killed || net->error` 累加）——这是两篇缝起来时最容易写错的一点；
3. 照常执行 `close_connection()` / `release_resources()` / `remove_thd()` / `dec_connection_count()` / `delete thd`；
4. **然后 `for(;;)` 继续** → `block_until_new_connection()` → 拿到新连接就复用。

**结论**：认证失败**不会杀死线程**。完整清理后线程归还缓存，等待服务下一个连接。`Threads_cached` / `Threads_created` 都不会因认证失败而变化。

### 八、THD 与线程的解绑：`Global_THD_manager`

`THD` 集合按 `thread_id` **分区**存储以降低全局锁竞争：

```cpp
void Global_THD_manager::add_thd(THD *thd) {
  assert(thd->thread_id() != reserved_thread_id);
  const int partition = thd_partition(thd->thread_id());
  MUTEX_LOCK(lock_list, &LOCK_thd_list[partition]);
  std::pair<THD_array::iterator, bool> insert_result =
      thd_list[partition].insert_unique(thd);
  if (insert_result.second) ++atomic_global_thd_count;
  assert(insert_result.second);
}

void Global_THD_manager::remove_thd(THD *thd) {
  const int partition = thd_partition(thd->thread_id());
  MUTEX_LOCK(lock_remove, &LOCK_thd_remove[partition]);
  MUTEX_LOCK(lock_list, &LOCK_thd_list[partition]);
  assert(unit_test || thd->release_resources_done());
  ...
  const size_t num_erased = thd_list[partition].erase_unique(thd);
  if (num_erased == 1) --atomic_global_thd_count;
  assert(1 == num_erased);
  mysql_cond_broadcast(&COND_thd_list[partition]);
}
```

`remove_thd()` 末尾的 broadcast 是给 `SHOW PROCESSLIST` / `KILL` 这类遍历者的。`thread_id` 由 `get_new_thread_id()` 在锁内自增分配。

> ★ `Per_thread_connection_handler::remove_thd()` **不存在**——THD 的登记/注销是 `Global_THD_manager` 的职责，与 handler 无关。handler 只管三件事：`add_connection()`（进入）、`get_max_threads()`（上限）、线程缓存。

`end_connection()` 里那句注释点出了"线程复用"是设计前提：

```cpp
  /*
    The thread may returned back to the pool and assigned to a user
    that doesn't have a limit. Ensure the user is not using resources
    of someone else.
  */
  release_user_connection(thd);
```

#### ★ 跨线程唤醒：`THD::awake()` —— KILL 如何打断另一个线程

这是本篇唯一"一个线程要改变另一个线程状态"的地方。`KILL` 语句的执行者**不是**被杀的那个线程，所以全部靠 `THD::awake()`（`sql/sql_class.cc`）异步设置标志 + 逐个踢醒等待点。

```cpp
void THD::awake(THD::killed_state state_to_set) {
  mysql_mutex_assert_owner(&LOCK_thd_data);        // ★ 调用者必须先持这把锁
  shutdown_clone_vio();
  ...
```

**五个动作，顺序即逻辑**：

| # | 动作 | 作用 | 备注 |
|---|---|---|---|
| ① | `kill_immunizer` 判定 | ★ **DD 操作期间免疫**：只把 kill 状态暂存，退出免疫区后再补一次 `awake()` | 保证新 DD 表上的操作不被中途打断 |
| ② | 置 `killed` 标志 | ★ 例外：`m_server_idle && KILL_QUERY` 时**不置位** | 注释原文：*"so that it doesn't affect the next command incorrectly"*——空闲连接收到 KILL QUERY 不该影响下一条命令 |
| ③ | `shutdown_active_vio()` | ★ **直接关 socket**，把阻塞在 `read()` 的线程踢出来 | 仅当 `this != current_thd`（杀别人）时做 |
| ④ | `ha_kill_connection(this)` | 打断**存储引擎内部**的等待 | InnoDB 侧对应 `trx_is_interrupted()` |
| ⑤ | broadcast `current_cond` / `current_mutex` | 踢醒正在等**条件变量**的线程 | 见下方长注释 |

★ **③ 有一个官方承认的竞态**（注释原文，值得原样记住）：

> *"without closing the socket we have a race condition. If the target thread passes the check of thd->killed, and then the current thread runs through THD::awake() ... and then the target thread runs into read(), it will block on the socket. As a result of the discussions around Bug#37780, it has been decided that we **accept the race condition**. A second KILL awakes the target from read()."*

⇒ **一次 KILL 可能杀不掉，需要第二次**——这是运维上"KILL 了没反应就再 KILL 一次"的源码出处。

★ **⑤ 也承认会失败**（注释原文）：若受害者已锁 `current_mutex` 但还没 `enter_cond()`（此时 `current_cond`/`current_mutex` 都是 0），就收不到信号，可能"wait forever"。缓解手段是 `enter_cond()` 之后再查一次 killed。

另外还有一步 **`post_kill_notification`**（`Connection_handler_manager::event_functions`），用于通知调度器/线程池插件（社区版是 no-op，企业版线程池要靠它）。

#### ★ 线程退出时的清理顺序

`THD::release_resources()` 的关键尾部（★ 顺序不能乱）：

```cpp
void THD::release_resources() {
  ...
  mysql_mutex_lock(&LOCK_thd_data);   ... mysql_mutex_unlock(&LOCK_thd_data);
  mysql_mutex_lock(&LOCK_thd_query);  ... mysql_mutex_unlock(&LOCK_thd_query);
  ...
  ha_close_connection(this);          // ① 先通知存储引擎（InnoDB 在此销毁 trx）
  ...
  /* Debug sync system must be closed after ha_close_connection, because ... */
  ...
  if (current_thd == this) restore_globals();   // ★ ② 最后才解绑线程局部量
```

★ `restore_globals()` 把 `current_thd` / `THR_MALLOC` 置回 `nullptr`——**必须在 `ha_close_connection()` 之后**，因为 InnoDB 销毁 trx 时还要进 InnoDB（需要 `current_thd`）。这也正是 InnoDB 侧那条断言要留一个后门的原因：

```cpp
bool trx_can_be_handled_by_current_thread(const trx_t *trx) {
  return trx->mysql_thd == nullptr || trx->mysql_thd == current_thd ||
         /* THD::restore_globals() set current_thd to nullptr and it is called
         in Table_access::~Table_access() before THD::release_resources(). ...
         Until Table_access::~Table_access() is fixed, we disable this
         assertion when current_thd == nullptr. */
         current_thd == nullptr;
}
```

（注：`current_thd` 是 C++11 `thread_local`，定义在 `sql/current_thd.cc`；★ **8.0.39 已无 `THR_THD` 这个 pthread key**，只剩两处 `#include` 注释提到它。）

### 九、`no-threads` 调试模式与插件接口

```cpp
bool One_thread_connection_handler::add_connection(Channel_info *channel_info) {
  if (my_thread_init()) { ... }
  THD *thd = channel_info->create_thd();
  ...
  thd_manager->add_thd(thd);
  bool error = false;
  if (thd_prepare_connection(thd))
    error = true;
  else {
    delete channel_info;
    while (thd_connection_alive(thd)) { if (do_command(thd)) break; }
    end_connection(thd);
  }
  close_connection(thd, 0, false, false);
  thd->release_resources();
  thd_manager->remove_thd(thd);
  Connection_handler_manager::dec_connection_count();
  delete thd;
  return error;
}
```

★ 它**同步地把整个连接生命周期跑完**，跑在**接纳线程里**，所以一次只能服务一个连接——纯调试用途（gdb 单步时非常方便，因为没有线程切换）。

**变量名与取值**（容易记错）：

- 变量：**`thread_handling`**，且是 **READ_ONLY**（只能启动时设）
- 枚举：`Connection_handler_manager::scheduler_types`（小写开头，不是 `Scheduler_types`），值为 `SCHEDULER_ONE_THREAD_PER_CONNECTION = 0`、`SCHEDULER_NO_THREADS`、`SCHEDULER_TYPES_COUNT`
- 取值字符串：`"one-thread-per-connection"` / **`"no-threads"`** / `"loaded-dynamically"`

**插件接管**（`Plugin_connection_handler`）：

```cpp
/**
   This is a wrapper class around global free functions implemented
   by connection handler plugins (e.g. thread pool). So instead of
   plugins implementing a Connection_handler subclass, they supply
   a set of function pointers to my_connection_handler_set() which
   instantiates Plugin_connection_handler.
*/
class Plugin_connection_handler : public Connection_handler {
  Connection_handler_functions *m_functions;
 protected:
  bool add_connection(Channel_info *channel_info) override {
    return m_functions->add_connection(channel_info);
  }
  uint get_max_threads() const override { return m_functions->max_threads; }
};
```

它不是"私有接口"，而是**官方暴露的插件服务** `my_thread_scheduler_service`（`include/mysql/service_thread_scheduler.h`），插件填 `struct Connection_handlerFunctions { max_threads; add_connection; end; }`（`include/mysql/thread_pool_priv.h`）。同时配套的 `THD_event_functions`（`thd_wait_begin/end`、`post_kill_notification`）让线程池能感知"THD 要睡了/醒了/被 kill 了"——这是做调度所必需的。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `max_connections` | **151**（`MAX_CONNECTIONS_DEFAULT`，`sys_vars.h`） | 范围 1–100000；`PARSE_EARLY`（PFS 用它做 sizing hint） |
| `thread_cache_size` | 静态 `DEFAULT(0)`，但**运行期被改写** | 见下方说明 |
| `thread_handling` | `one-thread-per-connection`（枚举值 0） | **READ_ONLY**，只能启动时设 |
| `slow_launch_time` | — | 线程创建超过它就累加 `slow_launch_threads` |

★ **`thread_cache_size` 的默认值有个陷阱**：`sys_vars.cc` 里静态默认是 `DEFAULT(0)`，但 `mysqld.cc` 的 `init_common_variables()` 会把它动态改成 **`8 + max_connections / 100`，上限 100**：

```cpp
  /* Calculate and update default value for thread_cache_size. */
  if ((default_value = 8 + max_connections / 100) > 100) default_value = 100;
  var = find_static_system_variable("thread_cache_size");
  var->update_default(default_value);
```

所以默认 `max_connections = 151` ⇒ **实际默认 `thread_cache_size = 9`**（不是 0）。范围是 0–16384。运行期改它会触发 `ON_UPDATE(modify_thread_cache_size)` 去收缩缓存。

### 状态变量

| 状态变量 | 背后对象 | 自增点 / 语义 |
|---|---|---|
| `Threads_cached` | `Per_thread_connection_handler::blocked_pthread_count` | `block_until_new_connection()` 里的 `++` / `--`。用 `SHOW_LONG_NOFLUSH`（`FLUSH STATUS` 不清零） |
| `Threads_connected` | `Connection_handler_manager::connection_count` | `check_and_incr_conn_count()` / `dec_connection_count()` |
| `Threads_created` | `Global_THD_manager::atomic_thread_created` | **唯一自增点**是 `Per_thread_connection_handler::add_connection()` 末尾。★ `no-threads` 模式不计数，缓存复用也不计数 |
| `Threads_running` | `Global_THD_manager::atomic_num_thread_running` | `sql_parse.cc` 的 `do_command()` 里 `inc/dec_thread_running()`。**语义是"正在执行 command 的连接数"，不是 OS 线程数** |
| `Connections` | `Global_THD_manager::thread_id_counter - 1` | 统计**累计分配过的线程 id 数**（含内部线程，故减 1），不是仅客户端连接 |
| `Aborted_connects` | `Connection_handler_manager::m_aborted_connects` | `my_thread_init` 失败 / `init_new_thd` 失败 / `thd_prepare_connection` 失败 / handler 拒绝 |
| `Aborted_clients` | `aborted_threads` | `end_connection()` 里按 `thd->killed \|\| (net->error && net->vio)` 累加。★ **认证失败不计入** |
| `Max_used_connections` | `Connection_handler_manager::max_used_connections` | `check_and_incr_conn_count()` 刷新，`FLUSH STATUS` 可重置 |
| `Max_used_connections_time` | 同上 `max_used_connections_time` | 到达峰值的时间戳 |

### shutdown 时怎么收尾

`close_connections()`（`sql/mysqld.cc`）的顺序很讲究：

```
Per_thread_connection_handler::kill_blocked_pthreads();   ← 先把缓存线程全部赶走
  → close_listener()
  → thd_manager->wait_till_no_thd()
  → Connection_handler_manager::wait_till_no_connection()  ← 等 COND_connection_count
```

而 `wait_till_no_connection()` 等的信号，正是 `dec_connection_count()` 在 `connection_count == 0` 时发出的 `mysql_cond_signal(&COND_connection_count)`。

---

## 可观测性

**一眼判断线程模型的健康度**：

| 现象 | 说明 |
|---|---|
| `Threads_created` 持续增长 | 线程缓存**没起作用**：要么 `thread_cache_size = 0`，要么连接来得比缓存能承接的快 |
| `Threads_cached` 长期为 0 且连接频繁 | 同上，考虑调大 `thread_cache_size` |
| `Threads_cached` 接近 `thread_cache_size` | 缓存已满，多余的线程会被真的销毁 |
| `Threads_running` 高但 `Threads_connected` 更高 | 正常——大部分连接空闲等待命令 |
| `Aborted_connects` 涨而 `Aborted_clients` 不动 | 典型的**认证失败**（密码错、host 不允许、连接数超限），而非客户端异常断开 |

**PFS 里怎么看**：连接线程的 `NAME` 是 `thread/sql/one_connection`（来自 `key_thread_one_connection`）。

**错误码**：

| 错误 | 含义 |
|---|---|
| `ER_CON_COUNT_ERROR` | Too many connections（两道闸都可能抛） |
| `ER_CANT_CREATE_THREAD` | `mysql_thread_create` 失败 |
| `ER_OUT_OF_RESOURCES` | `my_thread_init()` 或 `create_thd()` 失败 |
| `ER_SERVER_SHUTDOWN` | 复用线程拿到连接时发现正在 shutdown |
| `ER_SPECIFIC_ACCESS_DENIED_ERROR` | 管理连接但缺 `SERVICE_CONNECTION_ADMIN` |

---

## Misc

### 易混淆概念

- **`Threads_cached` vs `thread_cache_size`**：前者是**当前**缓存了多少条线程（状态），后者是**上限**（配置）。前者 ≤ 后者。
- **`Threads_created` vs `Connections`**：前者只统计 per-thread 模式下**真正新建**的线程；后者统计累计分配过的 **thread id**（含内部线程、含 `no-threads` 模式）。
- **`Threads_running` ≠ OS 线程数**：它是"正在执行 command 的连接数"，空闲连接在 `Threads_connected` 里但不在 `Threads_running` 里。
- **`Aborted_connects` vs `Aborted_clients`**：前者是**建立**失败（认证/资源/超限），后者是**已建立**后异常断开。认证失败只加前者。

### ★ 本篇核实出的"流传但不存在"的名字

写文档或排查时踩过这些坑，逐条列出：

| 常见写法 | 真实情况 |
|---|---|
| `blocked_pthread_list` / `waiting_pthread_list` | **都不存在**。真实是 `blocked_pthread_count`（计数）+ `waiting_channel_info_list`（队列）+ `wake_pthread`（令牌） |
| `extra_max_connections` | **全仓 0 命中**（Percona/MariaDB 概念）。8.0 用 admin interface + 动态权限做两段式校验 |
| `Per_thread_connection_handler::remove_thd()` | 不存在，是 `Global_THD_manager::remove_thd()` |
| `handle_one_connection` | **函数不存在**，真名 `handle_connection`（前者只在过时注释里） |
| `Connection_handler_manager::getInstance()` | 是 `get_instance()`（小写 g） |
| `m_saved_handler` | 是 `m_saved_connection_handler`（还有配套的 `m_saved_thread_handling`） |
| `Scheduler_types` | 是 `Connection_handler_manager::scheduler_types`（小写开头） |
| 调试模式取值 `"one-thread"` | 实际是 **`"no-threads"`** |
| 用 epoll | **没有**，`poll()` fallback `select()` |
| 社区版有线程池 | **没有实现**，只有 `my_thread_scheduler_service` 接口 |
| `check_and_incr_conn_count` 在 `sql_connect.cc` | 在 `sql/conn_handler/connection_handler_manager.cc` |

---

## 参考

**源码**

- `sql/conn_handler/`（约 4300 行）：本篇的全部一手材料
- `sql/mysqld.cc`：`network_init()`、`connection_attrib`、`close_connections()`、状态变量表 `status_vars[]`
- `sql/mysqld_thd_manager.cc`：`Global_THD_manager` 的分区登记
- `sql/sql_connect.cc`：`thd_prepare_connection()` / `login_connection()` / `check_connection()`
- `sql/auth/sql_authentication.cc`：`check_restrictions_for_com_connect_command()`

**相关文档**

- ★ **连接线程如何调用进 InnoDB、两族线程如何咬合**（THD↔trx 绑定、后台线程的 internal THD、并发闸门、关机握手）：[`../../infra/threading_architecture.md`](threading_architecture.md)「七、server ↔ InnoDB 的交互边界」
- InnoDB 后台线程全景与并发控制：同上（[一~六章](threading_architecture.md)）
- 握手与认证：[../auth/authentication.md](../../server/auth/authentication.md)
- 网络 IO 抽象：[vio.md](../../server/infra/vio.md)
- 启动期初始化：[../lifecycle/02_startup.md](../../server/lifecycle/02_startup.md)
- 权限判定：[../auth/authorization.md](../../server/auth/authorization.md)
- PFS 与 `key_thread_one_connection` 的注册：[pfs.md](../observability/pfs/pfs.md)
