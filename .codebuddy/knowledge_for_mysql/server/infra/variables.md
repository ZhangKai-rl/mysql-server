# MySQL 变量体系深度解析（系统变量 / 状态变量 / 用户变量 / 配置文件）

> 基于 MySQL 8.0.39 源码。MySQL 的"变量"有三大类：**系统变量**（`SET GLOBAL/SESSION/PERSIST` 可改的配置项）、**状态变量**（`SHOW STATUS` 的只读计数器）、**用户变量**（`SET @a := 1`）。它们的共同底座是"**描述符与值分离**"，且被 Performance Schema 用同一种 `SHOW_VAR` 中间格式统一处理。本篇把三者 + 配置文件/命令行这条"来源链路"一次讲透。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [A. 系统变量：`sys_var` 类体系与注册](#a-系统变量sys_var-类体系与注册)
  - [B. 配置文件与命令行：来源链路](#b-配置文件与命令行来源链路)
  - [C. GLOBAL / SESSION 双份存储](#c-global--session-双份存储)
  - [D. SET 语句与 `SET PERSIST`](#d-set-语句与-set-persist)
  - [E. 用户变量（@x）](#e-用户变量x)
  - [F. 特殊 SET 分支与 `SET_VAR` hint](#f-特殊-set-分支与-set_var-hint)
  - [G. 状态变量：`SHOW STATUS` → PFS](#g-状态变量show-status--pfs)
- [H. 与 PFS / I_S 系统表的关系](#h-与-pfs--i_s-系统表的关系)
- [变量体系相关的元变量](#变量体系相关的元变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 三类变量总览

| 类别 | 例子 | 谁能改 | 存储在哪 | 查询方式 |
|---|---|---|---|---|
| 系统变量 | `max_connections`、`innodb_buffer_pool_size` | `SET [GLOBAL\|SESSION\|PERSIST]` | `sys_var` 描述符 + 全局/会话两份内存 | `SHOW [GLOBAL\|SESSION] VARIABLES` / `performance_schema.global_variables` |
| 状态变量 | `Com_select`、`Innodb_buffer_pool_reads` | 只能读 | `System_status_var` 计数器块 | `SHOW [GLOBAL\|SESSION] STATUS` / `performance_schema.global_status` |
| 用户变量 | `SET @a := 1` | 任何用户随时 | `THD::user_vars`（per-connection hash） | 直接 `SELECT @a` |

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.0 | `SET GLOBAL`/`SET SESSION` 语义定形；`sys_var` 体系雏形 |
| 5.1 | 插件接口引入 `MYSQL_SYSVAR`/`MYSQL_THDVAR` 宏族 |
| 5.6 | PFS 引入 `status_by_thread/user/host/account` 等表，与 SHOW STATUS 并存 |
| 5.7 | `performance_schema.session_variables`/`global_variables`；I_S 的变量表靠 `mysql_system_variables` 临时表实现 |
| 8.0.0 | **`SET PERSIST`/`SET PERSIST_ONLY`/`RESET PERSIST`**（WL#8688）；`variables_info` 来源追踪（who/when 由 WL#9720 补）；动态权限框架（WL#8131）；**I_S 变量表移除、`SHOW STATUS`/`SHOW VARIABLES` 重写为 PFS 查询** |
| 8.0.3 | `SET_VAR` optimizer hint（WL#681） |
| 8.0.14 | `SESSION_VARIABLES_ADMIN` 动态权限（WL#12217） |
| 8.0.29 | 敏感变量（`SENSITIVE` flag、`persisted_variables_key` 加密、`SENSITIVE_VARIABLES_OBSERVER`）；`persisted_variables` 表 |

> 为什么演进的决策动机见「理论基础 → 他库对比与演进动机」。

### 一张总图

```
                    ┌────────────── 来源（写入路径） ──────────────┐
                    │ 命令行 --max-connections=100                 │
   my.cnf ── load_defaults ──► argv ──► my_getopt ──► setval()     │
   mysqld-auto.cnf ──(重启)──► argv ────────────────┘             │
   SET GLOBAL/PERSIST ──► sql_set_variables ──► global_update      │
   SET SESSION ──► sql_set_variables ──► session_update            │
                    ▼                                              │
        sys_var（描述符：名字/类型/offset/锁/回调，内嵌 my_option） │
                    │ 通过 offset 定位值                            │
        ┌───────────┴────────────┐                                 │
        ▼                        ▼                                 │
   global_system_variables   THD::variables（每连接一份，新连接拷贝）│
   （+ dynamic_variables_ptr 插件动态块）                            │
        ▲                                                          │
        └── performance_schema.variables_info（VARIABLE_SOURCE 溯源）┘

   状态变量（只读，另一条链路）：
       语句执行 ++counters ──► System_status_var（连续 ulonglong 块，per-THD）
                              ├─ 会话结束 ──► global_status_var（add_to_status）
                              └─ PFS 聚合 ──► status_by_thread/user/host/account
       查询：SHOW STATUS ──(语法重写)──► SELECT ... FROM performance_schema.global_status
```

---

## 理论基础

### 设计思想与权衡

#### 1. 描述符与值分离：一个变量 = 一个 `sys_var` 对象 + 一份共享内存

`sys_var` 里**不存值**，只存 `ptrdiff_t offset`。读全局值 = `*(T*)((uchar*)&global_system_variables + offset)`，读会话值 = `*(T*)((uchar*)&thd->variables + offset)`。为什么：

- **一份描述、两条写入路径**：命令行解析（`my_getopt`）与运行时 `SET` 都通过同一 offset 写同一内存，天然保证 `--max_connections=100` 与 `SET GLOBAL max_connections=100` 等价。
- **静态对象免分配**：几百个 `Sys_var_*` 都是 `sys_vars.cc` 里的文件作用域静态对象，C++ 动态初始化阶段即完成构造并挂进 `all_sys_vars` 链，`sys_var_init()` 只把链塞进 hash。
- **代价**：`System_variables` 结构体字段顺序被 offset 绑定；"假偏移"（`GLOBAL_VAR(X)` 用指针差，可为负）与"真偏移"（`SESSION_VAR(X)` 用 `offsetof`，恒正）混在同一字段，靠正负号区分——`Sys_var_integer` 构造里的 `if (offset >= 0) global_var(T) = def_val;` 就是利用这一点判断默认值该不该写进全局。

#### 2. 命令行选项就是系统变量：一个 `my_option`，两套解析

`sys_var` 内嵌一个**值语义**的 `my_option`（不是指针）。构造时 `option.value = (uchar**)global_var_ptr()`、`option.arg_source = &source`。所以 `--innodb-io-capacity=400` 的解析路径是 `my_handle_options2 → findopt → setval` 直接写 `srv_io_capacity`，与 `SET GLOBAL innodb_io_capacity=400` 的 `global_update` 写同一地址。**"命令行选项"和"系统变量"不是两个体系，是同一体系的两张脸。**

被否决的替代方案是 PostgreSQL 那样把配置解析与运行时参数分离（postgresql.conf 是文本、GUC 是运行时结构，靠 `pg_file_settings`/SIGHUP 二次加载沟通）。MySQL 选择合并的理由是简单；代价是 `sys_var_add_options` 要把所有变量**按值拷贝**进 `std::vector<my_option>` 交给 getopt，以及 `PARSE_EARLY`/`PARSE_NORMAL` 两批解析的复杂性。

#### 3. 会话值 = 新连接时一次性整块拷贝 + 插件变量的惰性 COW

`THD::init()` → `plugin_thdvar_init()` 里核心只有一行 `thd->variables = global_system_variables;`。**GLOBAL 改了已存在会话不受影响**不是某一行 if，而是三条代码事实的组合：① 拷贝只在新连接时一次；② `global_update` 只写 `global_system_variables`；③ `session_update` 只写 `thd->variables`。反向也成立：`SET SESSION` 后新连接也看不到（拷的是全局实例）。

插件/组件的 THDVAR（会话级）变量**不在** `System_variables` 结构体里（结构体编译期定死），而是存在动态字节块 `global_system_variables.dynamic_variables_ptr` 上。会话的 `dynamic_variables_ptr` 初始为 null，首次访问才 `alloc_and_copy_thd_dynamic_variables()` **增量 COW**（只拷新增字节，已改写的保持不动）——绝大多数连接永远不碰大多数 THDVAR，这是 COW 的直接动机。

#### 4. 状态变量的连续内存块：把结构体当数组用

`System_status_var` 前半段是**刻意连续的 ulonglong 计数器**，用 `FIRST_STATUS_VAR`/`LAST_STATUS_VAR`/`COUNT_GLOBAL_STATUS_VARS` 三个宏圈定。聚合（GLOBAL = 全局基线 + 所有会话求和）就变成一条循环：

```cpp
void add_to_status(System_status_var *to_var, System_status_var *from_var) {
  ulonglong *end = (ulonglong *)((uchar *)to_var + offsetof(System_status_var, LAST_STATUS_VAR) + sizeof(ulonglong));
  ulonglong *to = (ulonglong *)to_var, *from = (ulonglong *)from_var;
  while (to != end) *(to++) += *(from++);
  to_var->com_other += from_var->com_other;
  for (int c = 0; c < SQLCOM_END; c++) to_var->com_stat[c] += from_var->com_stat[c];
}
```

**代价是布局脆弱性**：连续块中间插非 `ulonglong` 字段会静默破坏数组语义。`last_query_cost`（`double`）被刻意放 `LAST_STATUS_VAR` 之后——聚合对它无意义（"上一条查询"不能求和），这既是布局约束也是语义声明。被否决的方案是 `std::map<string, ulonglong>` 或指针数组（hash/间接寻址开销对每条语句都要 ++ 的计数器不可接受）。

#### 5. 状态变量的"观测者效应"控制

`SHOW STATUS` 本身会污染计数器（它也是语句：`Com_show_status` +1、内部临时表让 `Created_tmp_tables` +1）。`Sql_cmd_show_status::execute` 的应对是**语句前快照 + 增量转移**：

```cpp
System_status_var old_status_var = thd->status_var;
thd->initial_status_var = &old_status_var;      // 快照指向语句开始前
... 执行内层 SELECT ...
add_diff_to_status(&global_status_var, &thd->status_var, &old_status_var);  // 增量并入全局，不丢
thd->status_var = old_status_var;               // 会话计数回滚
```

session 值用快照（PFS 的 `set_status_vars()` 优先用 `initial_status_var`），语句自身增量用 `add_diff_to_status` **转移**进全局（不丢）。`Com_show_status++` 发生在 `mysql_execute_command` 更早处、快照之后，所以它本身不回滚——SHOW STATUS 确实计入 `Com_show_status`。

#### 6. `SET PERSIST` 的持久化设计（WL#8688）

四个关键决策：① **为什么是新文件不改 my.cnf**——需要 server 独占、整体重写、可原子替换、承载结构化元数据（谁在何时设的）的机器生成文件；② **JSON 而非 INI**——值可能含特殊字符，JSON 自描述、直接复用现成 DOM；③ **写入原子性 = write-ahead + rename**——永远先写 `.backup`，`fflush`+`my_sync` 后 `my_rename` 覆盖，崩溃窗口内要么旧文件完好要么新文件完整；④ **只读变量怎么生效：追加到命令行**——`PERSIST_ONLY` 存的只读变量进 `mysql_server_static_options` 段，启动时 `append_read_only_variables` 把它们**拼回 argv**（`----persist-args-separator----` 分隔），因为只读变量不能运行时 SET。

#### 7. 变量来源追踪（`variables_info`）

`VARIABLE_SOURCE` 的 10 个取值回答"这个变量是谁设的"：编译默认 / 哪个配置文件（按路径映射）/ 命令行 / 运行时 SET / 持久化文件。实现不是事后反推，而是**写值的同时顺手记录**——`my_option.arg_source` 与 `sys_var::source` 是同一份内存。官方点破的坑：`mysqld_safe` 把 my.cnf 选项当命令行参数传给 mysqld，所以 my.cnf 里设的变量可能显示为 `COMMAND_LINE` 而非 `GLOBAL`。

### 理论溯源

- **值对象与描述对象分离 + offset 定位**：`sys_var`（描述符）+ `offset`（定位器）与 `SHOW_VAR`（状态描述符）+ `System_status_var`（值块）同构——两套变量用同一招，data-oriented design 的经典形态。
- **Copy-on-write**：插件动态变量块的增量拷贝。
- **SoA 化聚合**：`System_status_var` 连续块 + 线性循环求和，缓存友好。
- **观测者效应**：监控系统的 self-instrumentation 经典问题。
- **Write-ahead + atomic rename**：与 binlog/redo 同源——先写完整副本，再一个原子操作切换可见性。

### 他库对比与演进动机

| 维度 | MySQL 8.0 | PostgreSQL | Oracle |
|---|---|---|---|
| 运行时持久化 | `SET PERSIST` → `mysqld-auto.cnf`(JSON) | `ALTER SYSTEM` → `postgresql.auto.conf`(INI) | `ALTER SYSTEM ... SCOPE=SPFILE/MEMORY/BOTH` |
| 持久化优先级 | **最高**（最后读，覆盖 my.cnf 与命令行） | 覆盖 postgresql.conf，但**被命令行 `-c` 覆盖** | 取决于 SCOPE |
| 来源追踪 | `variables_info`（10 种 source + who/when） | `pg_settings.source/sourcefile/sourceline`（无 who/when，非超管隐藏） | 无 |
| 配置预检 | 无 `pg_file_settings` 等价物，parse error 退出 | `pg_file_settings` 视图预检 | 无 |
| 会话级默认值层级 | 仅 GLOBAL→SESSION 两级 | **有** `ALTER DATABASE`/`ALTER ROLE` 级默认值 | 无 |
| 状态变量形态 | 计数器 + PFS 表 | `pg_stat_*` 视图（无"session 值求和"体系） | `v$sysstat/v$sesstat`（三 scope，对应 MySQL 三种） |
| 变更权限粒度 | 按操作拆动态权限 | 按参数 context + 参数级 GRANT | 系统权限 |

演进动机：8.0 前"改全局变量跨重启生效"只能手工改 my.cnf，而 my.cnf 与运行值可能不一致——这是 `SET PERSIST` 的直接动机。`SESSION_VARIABLES_ADMIN`（WL#12217）的动机是**最小化权限 footprint**（之前为改受限会话变量只能给 SUPER，顺带获得改所有全局变量的能力）。8.0 把 SHOW STATUS 投靠 PFS，动机是**一套实现服务两种接口**（SHOW 兼容 + SQL 表查询）并消灭双份实现的分歧。

---

## 核心实现

### A. 系统变量：`sys_var` 类体系与注册

#### 类继承树（`sql/set_var.h` / `sql/sys_vars.h`）

```
sys_var
├── Sys_var_integer<T, ARGT, SHOWT, SIGNED>        ← 整数模板：int32/uint/ulong/ha_rows/ulonglong/long
│     ├── Sys_var_bit                                @@autocommit 这类位域
│     └── Sys_var_session_special                    @@timestamp/@insert_id：纯函数读写，offset 是假的
├── Sys_var_typelib
│     ├── Sys_var_enum / Sys_var_bool / Sys_var_multi_enum
│     ├── Sys_var_set                                @@sql_mode（位图存储、字符串展示）
│     └── Sys_var_flagset                             @@optimizer_switch
├── Sys_var_charptr / Sys_var_version / Sys_var_lexstring / Sys_var_double
├── Sys_var_keycache / Sys_var_plugin / Sys_var_struct / Sys_var_have
├── Sys_var_transaction_isolation / _read_only       三态变量（GLOBAL/SESSION/one-shot）
└── Sys_var_gtid_next / _list / _purged ...          GTID 系列（读写语义归 replication/gtid.md，此处只列描述符类型）
```

flag 位（`sys_var::flag_enum`）：`GLOBAL`、`SESSION`、`ONLY_SESSION`、`READONLY`、`INVISIBLE`、`TRI_LEVEL`、`NOTPERSIST`、`HINT_UPDATEABLE`、`PERSIST_AS_READ_ONLY`、`SENSITIVE`。

**集合型变量的双向转换**（`Sys_var_set`，代表 `sql_mode`）：底层存 `ulonglong` 位图，展示时是逗号分隔字符串，两条转换路径：

```cpp
// 字符串 → 位图（do_check）
var->save_result.ulonglong_value =
    find_set(&typelib, res->ptr(), res->length(), nullptr, &error, &error_len, &not_used);
// 位图 → 字符串（session_value_ptr / global_value_ptr）
return (uchar *)set_to_string(running_thd, nullptr, session_var(target_thd, ulonglong), typelib.type_names);
```

`SET sql_mode='ANSI'` 还要触发**宏展开**——`check_sql_mode` 调 `expand_sql_mode`，把 `MODE_ANSI` 展开成 `REAL_AS_FLOAT|PIPES_AS_CONCAT|ANSI_QUOTES|IGNORE_SPACE|ONLY_FULL_GROUP_BY` 五个子模式（`MODE_TRADITIONAL` 同理），启动时还会对 global 值再展开一次。`on_update` 钩子 `fix_sql_mode` 同步 `thd->server_status` 的 `SERVER_STATUS_NO_BACKSLASH_ESCAPES` 位。`sql_mode_names` 表共 33 项、含 `NOT_USED_*` 占位保持位序稳定（注释警告：加新模式要同步 `mysql.event/routines/triggers` 的列定义）。

定义变量的宏展开成 `(flag, offset, size)` 三元组：

```cpp
#define GLOBAL_VAR(X)   sys_var::GLOBAL, (((const char *)&(X)) - (char *)&global_system_variables), sizeof(X)
#define SESSION_VAR(X)  sys_var::SESSION, offsetof(System_variables, X), sizeof(((System_variables *)0)->X)
#define SESSION_ONLY(X) sys_var::ONLY_SESSION, offsetof(System_variables, X), ...
#define READ_ONLY sys_var::READONLY +      // 带尾随 + 的修饰宏可链式叠加
#define HINT_UPDATEABLE sys_var::HINT_UPDATEABLE +
#define NO_CMD_LINE CMD_LINE(NO_ARG, -1)   // option.id == -1 ⇒ 没有命令行对应项
```

#### 注册表：静态对象挂链 + 双 hash

```cpp
// sql/set_var.cc
static collation_unordered_map<string, sys_var *> *static_system_variable_hash;
static collation_unordered_map<string, sys_var *> *dynamic_system_variable_hash;
ulonglong dynamic_system_variable_hash_version = 0;
```

**双 hash 的理由**：静态变量编译进二进制、指针永久有效，查表**免锁**；插件/组件变量会被 UNINSTALL 摘除，查表持 `LOCK_system_variables_hash` 读锁，每次增删 `++dynamic_system_variable_hash_version` 作版本号。`add_dynamic_system_variable_chain` 显式拒绝与静态变量同名（`ER_DUPLICATE_SYS_VAR`）。

启动时序：

```
mysqld_main
  ├─ sys_var_init()                                    ← 静态对象构造期已挂好 all_sys_vars 链，这里建 hash 塞入
  ├─ load_defaults(MYSQL_CONFIG_NAME, load_default_groups, &argc, &argv)   ← my.cnf 读进 argv
  ├─ persisted_variables_cache.init() + load_persist_file()
  │    └─ append_parse_early_variables()               ← PERSIST 的 PARSE_EARLY 只读变量拼回命令行
  ├─ handle_early_options()                            ← 第一批：PARSE_EARLY
  ├─ ... append_read_only_variables()                  ← PERSIST_ONLY 普通只读变量拼回命令行
  ├─ handle_options()                                  ← 第二批：PARSE_NORMAL
  ├─ init_server_components()                          ← 插件/组件初始化（插件变量此刻注册）
  └─ persisted_variables_cache.set_persisted_options(false)  ← dynamic 持久化变量生效（最晚期）
```

### B. 配置文件与命令行：来源链路

这是回答"配置文件是不是同一部分"的核心——**配置文件的最终落点就是系统变量**，my.cnf 读取机制是变量体系的"来源半链"，全套如下。

#### B.1 搜索列表与顺序：`init_default_directories` + `my_search_option_files`

Unix 下搜索目录顺序（源码注释原样列出）：

```
/etc/my.cnf → /etc/mysql/my.cnf → $SYSCONFDIR/my.cnf（编译期 --sysconfdir）
→ $MYSQL_HOME/my.cnf → --defaults-extra-file=<path> → ~/.my.cnf
```

Windows 是：系统目录 → Windows 目录 → `C:/` → 可执行文件父目录 → 之后的同上。扩展名 Unix 只有 `.cnf`，Windows `.ini`/`.cnf`。`add_directory` 用 `array_append_string_unique` 去重，**重复目录移到列表末尾**（注释：避免重复读且保持优先级）。

`my_search_option_files` 是三种互斥模式：

1. `--defaults-file=...` 给出 → **只读这一个文件**，打不开报 `EE_FAILED_TO_OPEN_DEFAULTS_FILE` 并**退出**；
2. conf_file 自带目录 → 只读它；
3. 否则（且非 `--no-defaults`）→ 依序遍历目录列表；空槽处若给了 `--defaults-extra-file` 则读它（在全局文件之后、`~/.my.cnf` 之前）。

两个坑（源码行为）：`--defaults-file` 与 `--defaults-extra-file` 同时给 → extra 被**静默忽略**；`--no-defaults` 的实现是跳过目录循环分支（不是真的清列表），此时 login 文件（`~/.mylogin.cnf`，仅客户端）**仍会读**，`mysqld-auto.cnf` 也因 `no_defaults` 被禁用。另外 **world-writable 的配置文件会被忽略**并告警（防读入被篡改的文件）。

#### B.2 argv 组装与优先级：last-wins 的实现（重点）

`my_load_defaults` 把配置文件选项**放在命令行参数之前**（头注释明说："put them BEFORE the arguments that are already in argc and argv. This way ... command line options override options in configuration files"）：

```
res[0] = 程序名
res[1..N]   各配置文件的选项（--opt=value 形式，按搜索顺序）
res[N+1]    ----args-separator----        ← 分隔符（mysqld 设置 my_getopt_use_args_separator=true）
res[N+2..M] 原始命令行参数（剔除 --defaults-* 之后）
```

而 `my_handle_options2` 从头到尾**顺序 setval**（每命中一个选项立即写 `optp->value`，即 sys_var 的全局存储）——后解析的覆盖先解析的 ⇒ **last-wins**。argv 里命令行在配置文件之后 ⇒ **命令行 > 配置文件**。

`mysqld-auto.cnf` 的选项**不经过** `my_search_option_files`：`Persisted_variables_cache` 直接把内容拼成 `--loose_<key>=<value>`（按时间戳排序）append 到 argv **尾部**，前插 `----persist-args-separator----`：

```
[0] 程序名
[1..N]   配置文件选项
[N+1]    ----args-separator----
[N+2..M] 命令行选项
[M+1]    ----persist-args-separator----   ← 仅当存在 persisted 变量
[M+2..K] mysqld-auto.cnf 的选项（--loose_x=v）
```

persisted 段在最后 ⇒ **persisted > 命令行 > 配置文件**——这正是 WL#8688 的 R4/R7"持久化值优先"的实现，不是额外判断，是顺序的自然结果。

分隔符存在的动机（源码注释，Bug#25192）：配置文件的选项必须是 `--opt=value` 形式，命令行的值可以是下一个参数——解析器需要知道当前参数来自哪段才能决定取值方式；且 `update_variable_source` 按段预登记来源（配置文件段/命令行段/persisted 段分别用不同 path）。

最终全景（`mysqld_main` 的多轮解析）：

```
mysqld_main
  ├─ load_defaults("my", load_default_groups)         ← my.cnf 读进 argv（配置段+分隔符+命令行段）
  ├─ persisted_variables_cache.init + load_persist_file   ← 读 mysqld-auto.cnf（JSON）
  ├─ append_parse_early_variables()                   ← argv 尾部追加 PARSE_EARLY 持久化变量
  ├─ handle_early_options()                           ← 第 1 轮解析（PARSE_EARLY，skip_unknown=true）
  ├─ append_read_only_variables()                     ← remaining_argv 尾部追加只读持久化变量
  ├─ get_options() → handle_options()                 ← 第 2 轮解析（my_long_options + 全部 sys_var）
  └─ 收尾空选项表检查                                  ← 兜底报 ER_EXCESS_ARGUMENTS
```

#### B.3 分组（groups）匹配

mysqld 读的 group：`load_default_groups = { "mysql_cluster"(仅NDB), "mysqld", "server", "mysqld-8.0", <Windows服务名> }`。**`[mysqld-8.0]` 是字面完整 group 名**（`MYSQL_BASE_VERSION` 宏展开成 `"mysqld-8.0"`），没有任何 `-` 后缀截断逻辑。匹配用 `find_type`（`FIND_TYPE_NO_PREFIX`）：不区分大小写、不做前缀匹配。`--defaults-group-suffix=x`（或环境变量 `MYSQL_GROUP_SUFFIX`）把每个 group 追加一份 `group+x`（如 `[mysqld-x]`）。`[client]` 由客户端声明（`{"工具名", "client"}`），mysqld 不读。

#### B.4 行解析细节（`search_default_file_with_ext`）

- **注释**：行首 `#`、`;` 整行跳过；行中 `#` 由 `remove_end_comment` 截断（**引号内的 `#` 不截断**，引号内 `\` 转义也跟踪）；行中 `;` 不截断。
- **长行**：静态缓冲 4096 字节，超出自动拼接，无传统反斜杠续行。
- **引号**：值两端配对引号（`"`/`'`，长度>1）被剥离；不配对按普通字符保留。
- **转义**：值中支持 `\b \t \n \r \\ \s \" \'`，未知转义保留 `\` 本身。
- **`option=` 空值**：生成 `--option=`，字符串类型合法（重置手段），数值类型报错。
- **大小写**：group 名不区分大小写；选项名区分大小写但 `-`/`_` 等价（`getopt_compare_strings`）。
- **无环境变量展开**：8.0.39 的 my.cnf 解析里 `$` 只出现在注释中——`$MYSQL_HOME` 这类写法**不支持**。
- 解析出的每个选项统一组装成 `--name=value`，经 `handle_default_option` 判 group 后 push 进 `my_args`，同时调 `update_variable_source(option, cnf_file)` 登记来源文件。

#### B.5 `!include` / `!includedir`

在 `search_default_file_with_ext` 里行首 `!` 开头解析：`!include` 递归读指定文件、`!includedir` 遍历目录中扩展名匹配的文件（Unix `.cnf`），**递归上限 10 层**（超限 WARNING 跳过）。被 include 的文件**继承当前 group 状态**，其选项按"include 处的位置"插入 argv（同 group 内仍是 last-wins）。include 的文件路径会登记进 `default_paths`（source 追踪用）。目录打不开是 fatal。

#### B.6 选项前缀（`my_getopt` 的 `special_opt_prefix`）

前缀表只有 5 个：`skip / disable / enable / maximum / loose`（**没有 `--minimum-`**）。语义：

- `--skip-x` / `--disable-x` → 值强制 `"0"`；但 `--skip-x=0` 是**双重否定** → `"1"`。`--enable-x` 对称（`=0` → 假）。
- `--maximum-x=N` → 写 `my_option::u_max_value`（`max_system_variables` 的对应偏移，即 `--maximum-max_connections` 这类软上限）。
- `--loose-x` → 未知选项降级为 WARNING 且不退出（`--loose-` 剥掉后重查表）。**persisted 变量全部以 `--loose_` 前缀生成**——保证 mysqld-auto.cnf 里有过时变量时服务器不至于起不来。
- 前缀可叠加（`--loose-skip-innodb`），剥离循环 `i = -1` 重启。

#### B.7 首参选项：`--no-defaults` / `--defaults-file` / `--print-defaults`

- **必须第一个参数**（头注释："If used, they must be first argument on the command line!"），检查是 `strcmp(argv[1], "--no-defaults")`——不在第一位就**静默不生效**（无专门错误码），后续被当普通未知选项。
- `--print-defaults`：打印合并后的 argv（密码打码 `--password=*****`）后 `exit(0)`，用于排障"最终生效的参数是什么"。
- `my_print_default_files`（`--verbose --help` 时输出）打印搜索文件列表与 groups。

#### B.8 来源追踪的落点

来源的路径→枚举映射（`my_default.cc` 的 `default_paths`）：

```cpp
default_paths["/etc/my.cnf"]        = enum_variable_source::GLOBAL;
default_paths[home + ".my.cnf"]     = enum_variable_source::MYSQL_USER;
default_paths[home + ".mylogin.cnf"]= enum_variable_source::LOGIN;
default_paths[datadir + "mysqld-auto.cnf"] = enum_variable_source::PERSISTED;
default_paths[""]                   = enum_variable_source::COMMAND_LINE;
```

写入时机分三处：配置文件选项在 `handle_default_option` 收集时、命令行段在 `my_handle_options2` 开头扫描分隔符后、persisted 段在扫描 persist 分隔符后——都是 `update_variable_source` 预登记进 `variables_hash`。真正回填在解析每命中一个选项时：`setval_source()` → `set_variable_source()` 查 `variables_hash` 把 `{路径, enum}` 拷进 `my_option.arg_source`——它正是 `sys_var::source` 的地址（构造时 `option.arg_source = &source`），于是 `performance_schema.variables_info` 直接读 `sys_var::source` 即可溯源。

### C. GLOBAL / SESSION 双份存储

```cpp
// sys_var::session_var_ptr / global_var_ptr —— offset 是同一份
uchar *sys_var::session_var_ptr(THD *thd) { return ((uchar *)&(thd->variables)) + offset; }
uchar *sys_var::global_var_ptr()          { return ((uchar *)&global_system_variables) + offset; }
```

写路径分派（`sys_var::update`）：

```cpp
if (type == OPT_GLOBAL || type == OPT_PERSIST || scope() == GLOBAL) {
  AutoWLock lock1(&PLock_global_system_variables);
  AutoWLock lock2(guard);
  return global_update(thd, var) || (on_update && on_update(this, thd, OPT_GLOBAL));
} else {
  mysql_mutex_lock(&thd->LOCK_thd_sysvar);
  bool ret = session_update(thd, var) || (on_update && on_update(this, thd, OPT_SESSION));
  mysql_mutex_unlock(&thd->LOCK_thd_sysvar);
  if ((var->type == OPT_SESSION) || !is_trilevel()) {
    if (!ret && thd->session_tracker.get_tracker(SESSION_SYSVARS_TRACKER)->is_enabled())
      thd->session_tracker.get_tracker(SESSION_SYSVARS_TRACKER)->mark_as_changed(thd, name);
  }
  return ret;
}
```

GLOBAL 拿**两把锁**（`PLock_global_system_variables` + 变量自己的 `guard`），读路径 `value_ptr` 对称。`on_update` 钩子若做重活，放在 `pre_update` 阶段提前到加锁前执行（注释明说避免死锁——钩子里可能去拿别的锁）。

#### 插件 THDVAR：从宏到内存

插件声明 `MYSQL_THDVAR_ULONG(lock_wait_timeout, ...)` 展开为"头部 + 私有区"结构：头部 `MYSQL_PLUGIN_VAR_HEADER`（flags/name/comment/check/update 五字段），私有区首字段 `int offset`（初始 **-1**）。服务器注册时：

1. `register_var()` 在动态块按类型大小**对齐分配 offset**，写入永久保留的 `st_bookmark`（卸载重装也复用）；
2. `construct_options()` 里 `*(int *)(opt + 1) = v->offset;` —— **回写真实 offset**（`opt+1` 跳过头部 5 字段）；
3. 绑定 resolve thunk：`((thdvar_ulong_t *)opt)->resolve = mysql_sys_var_ulong;`（内部调 `intern_sys_var_ptr(thd, offset, ...)`）；
4. 包装成 `sys_var_pluginvar` 注册进 dynamic hash。

插件侧读值一行宏：`THDVAR(thd, name)` 展开为 `*(ulong*)(thd->variables.dynamic_variables_ptr + offset)`（若会话未分配动态块，先增量 COW）。注意 `sys_var_pluginvar` 给基类传的 offset 是 **0**——插件变量完全不走 `sys_var::offset`，走 `plugin_var + 1` 私有区。

#### 存储引擎（InnoDB）实例

`static SYS_VAR *innobase_system_variables[]`（`ha_innodb.cc:23199`）挂在 `mysql_declare_plugin(innobase)` 的 `system_vars` 字段（**不是** `handlerton`——它没这字段）：

```cpp
// 1. 无钩子：直写 srv_flush_log_at_trx_commit
static MYSQL_SYSVAR_ULONG(flush_log_at_trx_commit, srv_flush_log_at_trx_commit,
                          PLUGIN_VAR_OPCMDARG, "...", nullptr, nullptr, 1, 0, 2, 0);
// 2. update 钩子钳制 + 警告
static MYSQL_SYSVAR_ULONG(io_capacity, srv_io_capacity, PLUGIN_VAR_RQCMDARG,
                          "...", nullptr, innodb_io_capacity_update, 200, 100,
                          SRV_MAX_IO_CAPACITY_LIMIT, 0);
// 3. 超大变量 + 异步 resize 状态机
static MYSQL_SYSVAR_LONGLONG(buffer_pool_size, srv_buf_pool_curr_size,
                             PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_PERSIST_AS_READ_ONLY,
                             "...", nullptr, innodb_buffer_pool_size_update,
                             srv_buf_pool_def_size, srv_buf_pool_min_size,
                             srv_buf_pool_max_size, 1024 * 1024L);
// 4. 会话级 THDVAR
static MYSQL_THDVAR_ULONG(lock_wait_timeout, PLUGIN_VAR_RQCMDARG,
                          "...", nullptr, nullptr, 50, 1, 1024 * 1024 * 1024, 0);
```

`innodb_buffer_pool_size_update` 是"update 钩子不能做重活"的反例：它**不直接 resize**，只做状态机检查（`buf_pool_resize_status_code` 必须 COMPLETE/FAILED）→ 校验对齐 → `os_event_set(srv_buf_resize_event)` 唤醒**后台线程** → 写回对齐值。InnoDB 还消费来源追踪：`innodb_dedicated_server` 生效前判断 `innodb_buffer_pool_size` 的 source 是否 `COMPILED`（用户没显式设才允许自动调优）。

### D. SET 语句与 `SET PERSIST`

#### 主链路：`SET GLOBAL max_connections = 1000`

```
parse → PT_set → PT_set_scoped_system_variable("GLOBAL", "max_connections", expr)
  → add_system_variable_assignment(): System_variable_tracker 查 hash 找 sys_var → new set_var(OPT_GLOBAL, ...)
  → mysql_execute_command case SQLCOM_SET_OPTION → sql_set_variables(thd, &lex->var_list, true)
      ├─ [趟1] set_var::resolve()   ← 权限 + 作用域 + fix_fields
      ├─ [趟2] set_var::check()     ← 类型/范围/on_check，值入 save_result union，不落盘
      ├─ [趟3] set_var::update()    ← 拿锁、写内存、on_update、记录来源
      └─ 若 OPT_PERSIST → Persisted_variables_cache::set_variable() + flush_to_file()
```

**三趟扫描**实现多变量伪原子性，核心是 `sql_set_variables`（`sql/set_var.cc`）：

```cpp
int sql_set_variables(THD *thd, List<set_var_base> *var_list, bool opened) {
  List_iterator_fast<set_var_base> it(*var_list);
  ...
  if (!thd->lex->unit->is_prepared()) {
    Prepared_stmt_arena_holder ps_arena_holder(thd);
    while ((var = it++)) {
      if ((error = var->resolve(thd))) goto err;      // 第 1 趟：resolve（全部）
    }
    if ((error = thd->is_error())) goto err;
    thd->lex->unit->set_prepared();
    if (!thd->stmt_arena->is_regular()) thd->lex->save_cmd_properties(thd);
  }
  if (opened && lock_tables(thd, lex->query_tables, lex->table_count, 0)) { error = 1; goto err; }
  thd->lex->set_exec_started();
  it.rewind();
  while ((var = it++)) {
    if ((error = var->check(thd))) goto err;          // 第 2 趟：check（全部）
  }
  if ((error = thd->is_error())) goto err;

  it.rewind();
  while ((var = it++)) {
    if ((error = var->update(thd)))                   // 第 3 趟：update（全部）
      goto err;
  }
  if (!error) {
    /* At this point SET statement is considered a success. */
    Persisted_variables_cache *pv = nullptr;
    it.rewind();
    while ((var = it++)) {
      set_var *setvar = dynamic_cast<set_var *>(var);
      if (setvar && (setvar->type == OPT_PERSIST || setvar->type == OPT_PERSIST_ONLY)) {
        pv = Persisted_variables_cache::get_instance();
        if (pv->set_variable(thd, setvar)) return 1;   // 先改内存缓存
      }
    }
    if (pv && pv->flush_to_file()) {                  // 最后一次性落盘
      my_error(ER_VARIABLE_NOT_PERSISTED, MYF(0));
      return 1;
    }
  }
err:
  for (set_var_base &v : *var_list) v.cleanup();
  free_underlaid_joins(thd->lex->query_block);
  return error;
}
```

逐段解释：

1. **`resolve` 全部 → `check` 全部 → `update` 全部**：任何一趟有变量失败，后面的都不执行——"要么全改、要么全不改"。注释明说这是刻意设计。
2. **`if (!unit->is_prepared())` 包住 resolve**：PS 的 EXECUTE 阶段（`prepared == true`）跳过 resolve，直接 check+update——resolve 的结果（找到的 sys_var 等）在 PREPARE 时已固化。
3. **PERSIST 落地在所有 update 成功后**：先 `set_variable` 改内存容器（`Persisted_variables_cache` 的六容器），最后一次性 `flush_to_file()`。如果 flush 失败，运行值已经改了但文件没写成——所以报 `ER_VARIABLE_NOT_PERSISTED`（"Please retry"）。
4. `check` 与 `update` 之间隔着 `set_var::save_result` union（`{ulonglong_value, double_value, plugin, time_zone, string_value, ptr}`）——update 阶段只搬运不求值。`check` 不拿锁，`update` 才拿锁。

`set_var::resolve` 的权限与作用域检查是"难点"——同一段代码浓缩了只读变量、作用域、双权限的全部规则：

```cpp
int set_var::resolve(THD *thd) {
  auto f = [this, thd](const System_variable_tracker &, sys_var *var) -> int {
    var->do_deprecated_warning(thd);
    if (var->is_readonly()) {
      if (type != OPT_PERSIST_ONLY) {
        my_error(ER_INCORRECT_GLOBAL_LOCAL_VAR, MYF(0), var->name.str, "read only");
        return -1;
      }
      if (type == OPT_PERSIST_ONLY && var->is_non_persistent() &&
          !can_persist_non_persistent_var(thd, var, type)) {
        my_error(ER_INCORRECT_GLOBAL_LOCAL_VAR, MYF(0), var->name.str,
                 "non persistent read only");
        return -1;
      }
    }
    if (!var->check_scope(type)) {
      int err = (is_global_persist()) ? ER_LOCAL_VARIABLE : ER_GLOBAL_VARIABLE;
      my_error(err, MYF(0), var->name.str);
      return -1;
    }
    if (type == OPT_GLOBAL || type == OPT_PERSIST) {
      if (check_priv(thd, false)) return -1;   // SUPER 或 SYSTEM_VARIABLES_ADMIN
    }
    if (type == OPT_PERSIST_ONLY) {
      if (check_priv(thd, true)) return -1;    // 双权限：SYSTEM_VARIABLES_ADMIN + PERSIST_RO_VARIABLES_ADMIN
    }
    if ((type == OPT_PERSIST || type == OPT_PERSIST_ONLY) && var->is_non_persistent() &&
        !can_persist_non_persistent_var(thd, var, type)) {
      my_error(ER_INCORRECT_GLOBAL_LOCAL_VAR, MYF(0), var->name.str, "non persistent");
      return -1;
    }
    /* value is a NULL pointer if we are using SET ... = DEFAULT */
    if (value == nullptr || value->fixed) return 0;
    if (value->fix_fields(thd, &value)) return -1;   // 右值可能引用表列
    ...
    return 0;
  };
  return m_var_tracker.access_system_variable<int>(thd, f, Suppress_not_found_error::NO).value_or(-1);
}
```

要点：① 只读变量**唯一**能碰的语法是 `PERSIST_ONLY`（重启生效）；② `check_scope` 里 `ONLY_SESSION` 变量不能被 `SET GLOBAL` 改（报 `ER_LOCAL_VARIABLE`）；③ `value == nullptr` 是 `SET ... = DEFAULT` 的约定（语法层 `set_expr_or_default: DEFAULT_SYM { $$ = NULL; }` 保证）；④ 所有变量访问都经过 `System_variable_tracker::access_system_variable`——它按变量的 Lifetime（STATIC/KEYCACHE/PLUGIN/COMPONENT）决定要不要持 `LOCK_system_variables_hash`/`LOCK_plugin`，静态变量免锁直取（注释明说这是性能关键路径）。

#### `SET PERSIST` 的存储与启动

`Persisted_variables_cache` 按"static/dynamic × parse_early/sensitive/普通"分六容器（+两个 plugin 容器）。文件格式：

```json
{ "Version": 2,
  "mysql_server": { "max_connections": {
      "Value": "152",
      "Metadata": { "Timestamp": 1519921341372531, "User": "root", "Host": "localhost" } } },
  "mysql_server_static_options": { ... } }   // PERSIST_ONLY 的只读变量
```

`sql_set_variables` 里 PERSIST 落地在**所有 update 成功后**（先改内存缓存、最后一次性 `flush_to_file()`）。`SET PERSIST_ONLY` 的三处特殊待遇：① `resolve` 里只读变量**只有** PERSIST_ONLY 能碰；② `check` 跳过值校验（重启后才生效，当前实例无法验证）；③ `update` 不写运行值也不记来源。

`RESET PERSIST` 只删文件不回滚运行值，且不触发隐式提交（`case SQLCOM_RESET: return lex->option_type != OPT_PERSIST;`）。

#### 权限矩阵

| 操作 | 权限 |
|---|---|
| `SET GLOBAL` / `SET PERSIST` | `SUPER`（废弃）或 `SYSTEM_VARIABLES_ADMIN` |
| `SET PERSIST_ONLY` | `SYSTEM_VARIABLES_ADMIN` **且** `PERSIST_RO_VARIABLES_ADMIN` |
| 受限会话变量（`sql_log_bin` 等） | 8.0.14+：`SESSION_VARIABLES_ADMIN`（SYSTEM_VARIABLES_ADMIN/SUPER 隐含） |
| 查看敏感变量值 | `SENSITIVE_VARIABLES_OBSERVER`（8.0.29+） |

实现：`set_var::resolve` 里 `OPT_GLOBAL/OPT_PERSIST` 走 `check_priv(thd, false)`（SUPER 或 SYSTEM_VARIABLES_ADMIN），`OPT_PERSIST_ONLY` 走 `check_priv(thd, true)`（双权限，报 `ER_PERSIST_ONLY_ACCESS_DENIED_ERROR`）。

### E. 用户变量（@x）

`THD::user_vars` 是 `collation_unordered_map<std::string, unique_ptr_with_deleter<user_var_entry>>`，`LOCK_thd_data` 保护。

```cpp
class user_var_entry {
  char *m_ptr;          // 值区：对象尾部 8 字节内联，或 realloc 外部缓冲
  size_t m_length;
  Item_result m_type;   // INT/REAL/STRING/DECIMAL 四选一
  query_id_t m_used_query_id;  // binlog 去重用
  ...
};
```

链路：`SET @a := expr` 语法层生成 `set_var_user`（包装 `Item_func_set_user_var`）→ `check()` 求值右表达式按类型存入 `save_result` union（`{vint,vreal,vstr,vdec}`）→ `update()` 按 `cached_result_type` 调 `update_hash()` → 持锁 `memcpy` 进 `user_var_entry::store()`。`SELECT @a := expr` 走同一组 `val_int()/val_str()`（**先求值再返回**，保证 `@a:=@a+1` 的求值顺序）。

两个精妙点：

1. **PS 每次执行重绑 entry**：`fix_fields` 末尾 `entry = nullptr`、`cleanup()` 再置空、`update()` 每次 `set_entry(current_thd, true)` 重绑。注释给出原因：trigger 缓存在 TABLE 里可能被**另一 THD** 复用，Item 不能长期持有某 THD 的 entry 指针。
2. **`Item_func_get_user_var::const_item()` 返回 true**（继承 `used_tables()==0`，它无参数）——优化器把它当常量折叠，但值每次执行动态读 entry（类里有 `is_non_const_over_literals()=true` 兜底）。

**用户变量会写 binlog**（常被误认为不写）：`get_var_with_binlog()` 在 `(binlog 打开 && sql_log_bin) && (is_update_query || thd->in_sub_stmt)` 时构造 `Binlog_user_var_event` 进 `thd->user_var_events`，语句写 binlog 前先写 `User_var_log_event`；同语句刚赋值的（`m_used_query_id == thd->query_id`）不重复记录。

### F. 特殊 SET 分支与 `SET_VAR` hint

**`SET NAMES` vs `SET CHARACTER SET`**（都落 `set_var_collation_client`）：

| | character_set_client | character_set_results | collation_connection |
|---|---|---|---|
| `SET NAMES cs` | cs | cs | cs |
| `SET CHARACTER SET cs` | cs | cs | **当前库默认 collation** |

**`SET TRANSACTION`（无前缀）**：语法层 `PT_start_option_value_list_transaction` 把 `lex->option_type` 强置 `OPT_DEFAULT`，转成对 `transaction_isolation`/`transaction_read_only` 的 `set_var(OPT_DEFAULT, ...)`。关键在 `Sys_var_transaction_isolation::session_update`：

```cpp
if (var->type == OPT_DEFAULT || !(thd->in_active_multi_stmt_transaction() || thd->in_sub_stmt)) {
  bool one_shot = (var->type == OPT_DEFAULT);
  return set_tx_isolation(thd, tx_isol, one_shot);   // 只改 thd->tx_isolation，不改 session 变量
}
```

"只影响下一个事务"的本质：OPT_DEFAULT 时**不改 `thd->variables.transaction_isolation`**，只改 `thd->tx_isolation`（InnoDB 每次事务开始经 `thd_get_trx_isolation()` 读它）。

**`SET @@x = DEFAULT`**：语法层 `set_expr_or_default: DEFAULT_SYM { $$ = NULL; }`，右值 Item 为 nullptr；`set_var::check` 对 `value==nullptr` 放行，`update` 改走 `sys_var::set_default()`（把编译默认值填进 save_result 再正常 check/update）。

**`SET_VAR` optimizer hint**：

```cpp
class Sys_var_hint {  // sql/opt_hints.h
  Mem_root_array<Hint_set_var *> var_list;   // { set_var *var; Item *save_value; }
  void update_vars(THD*);   // 执行前生效：resolve+check → copy_value 存旧值 → update
  void restore_vars(THD*);  // 执行后恢复：std::swap(value, save_value) → update → swap 回来
};
```

调用点在 `mysql_execute_command` 执行前（3350 行）与 finish 标签（4881 行）。限制：只有带 `HINT_UPDATEABLE` flag 的**静态**变量可 hint（`is_hint_updateable()` 要求 `m_tag == STATIC`，插件/组件变量不行）；slave 线程跳过；非 hint-updateable 变量报 `ER_NOT_HINT_UPDATABLE_VARIABLE` 警告。

### G. 状态变量：`SHOW STATUS` → PFS

#### 反直觉开场：SHOW STATUS 是 SELECT 的皮

8.0 的 `SHOW GLOBAL STATUS` 在语法分析阶段被重写成：

```sql
SELECT * FROM (SELECT VARIABLE_NAME AS Variable_name, VARIABLE_VALUE AS Value
               FROM performance_schema.global_status) global_status;
```

（`sql/sql_show_status.cc` 的 `build_query()`，末尾手工恢复 `lex->sql_command = SQLCOM_SHOW_STATUS`）。三个连带结论：① `Com_*` 只在 SHOW STATUS 里出现（`filter_by_name` 按命令类型过滤整棵 `Com` 子树）；② `SHOW SESSION STATUS` 显示语句开始前快照；③ `Innodb_*` 前缀是运行时拼的，源码 grep 不到。

#### `SHOW_VAR` 的"三重身份"

```cpp
struct SHOW_VAR { const char *name; char *value; enum enum_mysql_show_type type; enum enum_mysql_show_scope scope; };
```

| type | value 含义 | 例子 |
|---|---|---|
| `SHOW_LONG` 等 | **真实指针** | `{"Aborted_clients", (char*)&aborted_threads, SHOW_LONG, GLOBAL}` |
| `SHOW_LONG_STATUS` | **offsetof 偏移** | `{"Bytes_received", (char*)offsetof(System_status_var, bytes_received), ...}` |
| `SHOW_FUNC` | **函数指针** | `{"Threads_running", (char*)&show_num_thread_running, ...}` |
| `SHOW_ARRAY` | **子数组**（如 `Com_`） | `{"Com", (char*)com_status_vars, SHOW_ARRAY, ALL}` |

偏移量型解引用点：`value = (char*)status_var + (size_t)value;`（加基址）——对比 `SHOW_LONG` 不加基址。`SHOW_LONG` 与 `SHOW_LONG_NOFLUSH` 唯一差别是 `FLUSH STATUS` 是否清零。

#### 聚合：GLOBAL = global_status_var + ΣTHD

`PFS_status_variable_cache::do_materialize_global()` 分两阶段：

```
[阶段1 INITIALIZE] mysql_mutex_lock(&LOCK_status)   ← 防插件装卸
  → init_show_var_array(OPT_GLOBAL, strict=true)
      → filter_show_var(): match_scope / filter_by_name / can_aggregate
      → expand_show_var_array(): 展开 Com / 引擎 SHOW_ARRAY
[聚合] PFS_connection_status_visitor
  → visit_global(): add_to_status(totals, &global_status_var)
  → visit_global(..., with_THDs=true) → do_for_all_thd → 每个 THD: add_to_status(totals, &thd->status_var)
[阶段2 MATERIALIZE] manifest()
  → SHOW_FUNC → 执行回调（InnoDB 的 Innodb 条目在此变身 SHOW_ARRAY）
  → SHOW_ARRAY → 递归 manifest（前缀拼接）
  → Status_variable::init() → get_one_variable() → 数值转字符串
```

公式：`GLOBAL = global_status_var + Σ(所有登记在 Global_THD_manager 的 THD 的 status_var)`——**含后台线程**（replica IO/SQL thread、event scheduler 等），源码留 `// TODO: filter bg threads?`。`calc_sum_of_all_status()`（旧实现）只剩 `COM_STATISTICS`（`mysqladmin status`）一个调用者。

会话生命周期归并：连接结束 `THD::release_resources()` → `add_to_status(&global_status_var, ...)` + PSI `aggregate_thread_status()`（折进 account/user/host）+ `status_var_aggregated = true`；`CHANGE USER`/`RESET CONNECTION` 走 `cleanup_connection()`；顺序协议"先 release 再 remove_thd"保证不重不漏。

#### 平行账本：两套聚合通道必须隔离

`PFS_account::aggregate_status()` 的注释："Never aggregate to global_status_var, because of the parallel THD -> global_status_var flow."——account/user/host 维度与全局是**两条平行通道**，不交叉，否则双倍计数。`PFS_status_stats`（`ulonglong m_stats[COUNT_GLOBAL_STATUS_VARS]`）是断开会话的"遗留账本"：线程断开折进 account，account 清理折进 user/host。

#### `Com_xxx` 计数器

存储 `System_status_var::com_stat[(uint)SQLCOM_END]`，名字表 `com_status_vars[]`（`sql/mysqld.cc:3984`）。主计数点 `mysql_execute_command` 的 `thd->status_var.com_stat[lex->sql_command]++`（readonly 检查后、执行前）——**语法错误不计数、执行失败也计数**（官方：`Com_stmt_*` 对应"request issued"而非"successfully completed"）。协议命令（COM_PING 等）在 `dispatch_command` 手工 ++，复制 applier 也有手工补计数。

#### 存储引擎状态变量：InnoDB 双通道

**通道一 `SHOW ENGINE INNODB STATUS`（长文本）**：走 `handlerton::show_status`（字段名就叫 `show_status`，类型 `show_status_t`，**不存在** `show_status_func`）。`ha_show_status()` 发三列头（Type/Name/Status），引擎经 `stat_print` 回吐——**走 `protocol->store_string()`，不是 error log**。InnoDB 先写临时文件 `srv_monitor_file` 再读回输出，上限 1 MiB 截断（对应 `Innodb_truncated_status_writes`）。权限 `PROCESS_ACL`。

**通道二 `SHOW GLOBAL STATUS LIKE 'Innodb_%'`（结构化计数器）**：

```
all_status_vars 只有一条 {"Innodb", &show_innodb_vars, SHOW_FUNC, GLOBAL}
  → manifest() 执行 show_innodb_vars():
      ├─ innodb_export_status() → srv_export_innodb_status()   ★ 此刻才把 srv 计数器刷进 export_vars（懒刷新）
      └─ var->type = SHOW_ARRAY; var->value = &innodb_status_variables
  → 递归 manifest(子数组, prefix="Innodb") → "buffer_pool_reads" + 前缀 = "Innodb_buffer_pool_reads"
```

`export_var_t export_vars` 是 InnoDB 内部计数器的对外镜像，`innodb_status_variables[]` 逐项 `(char*)&export_vars.xxx`。

#### `FLUSH STATUS` 的精确语义

`refresh_status()`（**不存在** `mysql_reset_status`）干五件事：① 每个活跃 THD 先 `add_to_status(&global_status_var, ...)` 再清零（session 归零、增量保留）；② `reset_pfs_status_stats()`（account 先汇入 user/host 再清零）；③ `reset_status_vars()` 清 `SHOW_LONG`/`SHOW_SIGNED_LONG`（指针型全局变量），**`SHOW_LONG_NOFLUSH`（PFS `*_lost`）故意不清**；④ 清 key cache；⑤ `Max_used_connections` 重置为**当前连接数**（不是 0）。

关键事实：`refresh_status()` **并不 `memset(&global_status_var, 0)`**——全仓库对它的写只有启动初始化与 `add_to_status` 累加。手册只写一句 "Many status variables are reset to 0" 且**无完整清单**，这本身是坑。`TRUNCATE TABLE performance_schema.global_status` 与 `FLUSH STATUS` 等价。权限 `RELOAD` 或 `FLUSH_STATUS`。

### H. 与 PFS / I_S 系统表的关系

变量体系与 Performance Schema 的关系比表面深得多：**PFS 不仅暴露变量，它本身就是变量查询的实现者**——`SHOW VARIABLES` 和 `SHOW STATUS` 一样，在语法分析阶段被重写成 PFS 查询。

#### 1. `SHOW VARIABLES` 也被重写成 SELECT

与 `SHOW STATUS` 同文件同构（`sql/sql_show_status.cc`）：

```cpp
Query_block *build_show_session_variables(const POS &pos, THD *thd, ...) {
  static const LEX_CSTRING table_name = {STRING_WITH_LEN("session_variables")};
  return build_query(pos, thd, SQLCOM_SHOW_VARIABLES, table_name, wild, where_cond);
}
Query_block *build_show_global_variables(...) {
  static const LEX_CSTRING table_name = {STRING_WITH_LEN("global_variables")};
  ...
}
```

`SHOW [GLOBAL|SESSION] VARIABLES` ⇒ `SELECT ... FROM performance_schema.global_variables / session_variables`。**系统变量与状态变量在 8.0 走了同一条"SHOW → PFS 表"的通道**，这就是为什么 PFS 的实现注释说它们"implemented differently in the server, but the steps to process them are essentially the same"。

#### 2. PFS 里的变量表全集（12 张）

| 表 | 类别 | 数据源 | ACL |
|---|---|---|---|
| `global_variables` | 系统变量 | `PFS_system_variable_cache` + `enumerate_sys_vars`（GLOBAL 值） | `pfs_readonly_world_acl`（人人可读） |
| `session_variables` | 系统变量 | 同上（当前会话 SESSION 值） | world |
| `variables_by_thread` | 系统变量 | 同上（任意线程，含后台） | **`pfs_readonly_acl`（需显式 SELECT 授权）** |
| `variables_info` | 元信息 | `sys_var::source`（来源/范围/时间/用户） | world |
| `persisted_variables` | 持久化 | `Persisted_variables_cache` 容器 | world |
| `global_status` | 状态变量 | `PFS_status_variable_cache::do_materialize_global` | world |
| `session_status` | 状态变量 | `do_materialize_all(current_thd)` | world |
| `status_by_thread` | 状态变量 | `do_materialize_session(PFS_thread*)` | `pfs_truncatable_acl`（常规授权） |
| `status_by_user` / `status_by_host` / `status_by_account` | 状态变量 | `do_materialize_client` + `PFS_status_stats` 遗留账本 | truncatable |
| `user_variables_by_thread` | 用户变量 | `THD::user_vars` hash（遍历） | `pfs_readonly_acl` |

两个值得注意的权限差异：`variables_by_thread`/`user_variables_by_thread` 用 `pfs_readonly_acl`（**不是** world）——能看到别的会话/后台线程的变量值属于敏感能力，需要显式授权；而 `global_variables`/`session_variables`/`global_status` 人人可读（world），因为它们是"所有会话共享"或"本会话自己"的。

#### 3. 统一的 PFS 处理管线

PFS 侧两个平行的 cache 类（`storage/perfschema/pfs_variable.h/.cc`）：

```
PFS_system_variable_cache（系统变量）
  ├─ INITIALIZE: System_variable_tracker::enumerate_sys_vars(sort, scope, strict, output)
  │              —— 遍历 static/dynamic 双 hash，把 sys_var 转成 SHOW_VAR 中间格式
  └─ MATERIALIZE: manifest() → get_one_variable() → System_variable 对象

PFS_status_variable_cache（状态变量）
  ├─ INITIALIZE: init_show_var_array + expand_show_var_array（展开 SHOW_ARRAY）
  └─ MATERIALIZE: manifest() → Status_variable 对象
```

`SHOW_VAR` 是二者（再加用户变量）的**共同中间格式**——PFS 先把三类变量都统一成 `{name, value, type, scope}` 描述符，再走同一条求值/转字符串/物化管线。这是"描述符与值分离"设计在监控层的一次复用。

`enumerate_sys_vars`（`sql/set_var.h`）是系统变量的枚举入口：持 `LOCK_system_variables_hash` 读锁遍历两个 hash，按 `strict` 参数过滤 scope（`variables_by_thread` 用 strict 语义，`global_variables` 用宽松语义），输出 `Prealloced_array<System_variable_tracker, 200>`（预分配 200 槽，避免每次物化都分配）。

#### 4. I_S 的变量表已被 PFS 取代（8.0 演进）

`INFORMATION_SCHEMA.SYSTEM_VARIABLES / SESSION_VARIABLES / GLOBAL_VARIABLES` 在 8.0 中**已移除**——grep 8.0.39 的 `sql/` 无这三张表的定义，只有 `sql_show_status.cc` 里对 PFS 同名表的引用。演进脉络：

| 版本 | I_S | PFS | SHOW 实现 |
|---|---|---|---|
| 5.7 | `I_S.SYSTEM_VARIABLES`/`SESSION_VARIABLES`/`GLOBAL_VARIABLES`（基于 `mysql_system_variables` 临时表） | 只有 `session_variables`/`global_variables` | 独立实现 |
| 8.0 | **移除**（被 PFS 取代） | 表族补齐（`variables_by_thread`/`variables_info`/`persisted_variables`/`user_variables_by_thread`） | **重写为 PFS 查询** |

动机与 `SHOW STATUS` 投靠 PFS 相同：一套实现服务 SHOW 语法 + SQL 表查询两种接口，消灭 I_S 临时表、SHOW、PFS 三份实现的分歧。

#### 5. 其他相关系统对象

- **`mysql.component`**（DD 表）：组件注册系统变量时，变量名是 `component_name.var_name` 点分格式——组件被卸载，其变量随 `UNINSTALL COMPONENT` 从 dynamic hash 摘除（`component_sys_variable_unregister`）。
- **`sys` schema**：`sys.metrics` 等视图基于 `performance_schema.global_status`/`status_by_user` 等表构建——变量体系是 sys 视图的底层数据源。
- **`performance_schema.global_variables` vs `variables_info`**：前者给"当前值"，后者给"值从哪来"——排障时两者配合（`variables_info` 无索引、不可 TRUNCATE）。

---

## 变量体系相关的元变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `persisted_globals_load` | ON | 只读 | 是否读 `mysqld-auto.cnf`；自身不可 PERSIST（在 `never_persistable_vars` 黑名单） |
| `persisted_variables_key` | 空 | 只读 | 敏感变量加密 key（8.0.29+） |
| `persist_sensitive_variables_in_plaintext` | OFF | 只读 | 无 keyring 时是否允许明文持久化敏感变量 |
| `persist_only_admin_x509_subject` | 空 | 只读 | persist-restricted 变量要求的证书 Subject |
| `session_track_system_variables` | 见说明 | G/S | 哪些会话变量变更时经 `SESSION_TRACK_SYSTEM_VARIABLES` 通知客户端 |
| `session_track_state_change` | OFF | G/S | 是否跟踪会话状态变化 |

---

## Misc

### 易混淆概念

| 名字 | 是什么 | 不是什么 |
|---|---|---|
| `sys_var` | 系统变量描述符（类） | 不是值容器（值在 `global_system_variables`/`thd->variables`） |
| `my_option` | getopt 命令行选项描述符 | 不是独立配置体系——`sys_var` 内嵌它，共享值地址与来源 |
| `set_var` | 一条 SET 赋值的执行载体 | 不是变量本身 |
| `System_status_var` | 状态计数器块 | 与 `System_variables`（系统变量值）名字像但两回事 |
| `SHOW_VAR` | 状态变量描述符 | 不是值容器 |
| `SET GLOBAL` vs `SET PERSIST` | PERSIST = 改运行值**且**写文件 | PERSIST_ONLY = 只写文件不改运行值（重启生效） |
| `SET SESSION` vs `SET TRANSACTION` | TRANSACTION（无前缀）= one-shot | 不改 session 默认值 |
| `OPT_DEFAULT` | `SET @@x=1` 无前缀的作用域（等价 SESSION） | 与 `SET @@x=DEFAULT` 的 DEFAULT 无关 |
| `Questions` vs `Queries` vs `Com_*` | 客户端语句 / 含内层语句 / 按命令分类 | 三套口径常被混用来算吞吐 |

### 已知坑

- `mysqld_safe` 把 my.cnf 选项转成命令行 ⇒ `variables_info` 里显示 `COMMAND_LINE` 而非 `GLOBAL`。
- `mysqld-auto.cnf` 解析失败 ⇒ server **报错退出**，无自动备份回滚（`.backup` 只用于崩溃恢复）；删除会丢所有持久化设置。
- `mysqld-auto.cnf` 持久化值**优先于** my.cnf 与命令行（设计本意），故"重启后 read_only 与 my.cnf 不一致"是设计行为——常见运维事故源。
- `SHOW STATUS` 每次建内部临时表 ⇒ `Created_tmp_tables` 自增（监控高频采集自我污染）。
- `Created_tmp_disk_tables` 不统计 mmap 溢出的临时表（`temptable_use_mmap` 默认开）。
- `Com_stmt_*`/`Com_restart`/`Com_shutdown` 失败也计数；`Innodb_rows_updated` 官方标注 "not meant to be 100% accurate"。
- 压缩表下 `Innodb_buffer_pool_pages_data` 可能大于 `_total`（Bug #59550）；`Last_query_cost` 8.0.16 前对复杂查询为 0（Bug #92766）。
- `Innodb_row_lock_time_avg` 是启动以来累计平均，被历史长尾拉高后不回落，与 `SHOW ENGINE INNODB STATUS` 的瞬时语义不同。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 7.1.9.1 System Variable Privileges*、*7.1.9.3 Persisted System Variables*、*7.1.10 Server Status Variables*、*29.12.14.2 variables_info Table*、*29.12.15 Status Variable Tables*
- WorkLog: [WL#8688 SET PERSIST](https://dev.mysql.com/worklog/task/?id=8688)（含 variables_info 定义）、[WL#9720](https://dev.mysql.com/worklog/?k=persist)、[WL#9763](https://dev.mysql.com/worklog/?k=persist)、[WL#9787](https://dev.mysql.com/worklog/?k=persist)、[WL#681 SET_VAR hint](https://dev.mysql.com/worklog/task/?id=681)、[WL#8131 动态权限](https://dev.mysql.com/worklog/?k=dynamic+privilege)、[WL#12217 SESSION_VARIABLE_ADMIN](https://dev.mysql.com/worklog/task/?id=12217)

**他库文档**
- *PostgreSQL Documentation → 19.1 Setting Parameters*、*53.25 pg_settings*（`ALTER SYSTEM`/`postgresql.auto.conf`/`pg_file_settings`/context 分类）

**相关文档**
- PFS 的通用表访问机制、PFS 系统变量 cache 的线程间读细节见 [`pfs.md`](pfs.md)
- GTID 系列变量（`gtid_next`/`gtid_purged` 等）的读写语义见 [`../replication/gtid.md`](../replication/gtid.md)
- 引擎接口（`handlerton`/`handler`）见 [`../handler.md`](../handler.md)
- SET 语句解析（`PT_set_*` 语法树族）属于查询主链第 2 步，见 [`../query/README.md`](../query/README.md)
