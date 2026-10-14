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

**★ 三类的"载体"完全不同**（★ 这是理解全篇的骨架）：

| 类别 | 描述符 / 载体 | 真名与位置 |
|---|---|---|
| 系统变量 | C++ 类 `sys_var`（有继承树、双 hash 索引） | `sql/set_var.h:107` |
| 状态变量 | **C 结构体 `SHOW_VAR`**（4 字段，无 hash，就是数组） | ⚠️ 不在 `sql/set_var.h`，在 **`include/mysql/status_var.h:80`** |
| 用户变量 | `user_var_entry` + **没有 `sys_var` 对象** | `sql/item_func.h:2953` |

```cpp
/* include/mysql/status_var.h:80 —— 状态变量没有"注册表类"，就是个 4 字段 C 结构 */
/**
  SHOW STATUS Server status variable
*/
struct SHOW_VAR {
  const char *name;
  char *value;          /* ★ 语义随 type 变化：绝对指针 / offsetof 偏移 / 函数指针 / 子数组指针 */
  enum enum_mysql_show_type type;
  enum enum_mysql_show_scope scope;
};

#define SHOW_VAR_MAX_NAME_LEN 64
#define SHOW_VAR_FUNC_BUFF_SIZE 1024
typedef int (*mysql_show_var_func)(MYSQL_THD, SHOW_VAR *, char *);
```
```cpp
/* sql/set_var.h:1025 —— 用户变量没有 sys_var 对象 */
/* User variables like @my_own_variable */
class set_var_user : public set_var_base {
  Item_func_set_user_var *user_var_item;
 public:
  set_var_user(Item_func_set_user_var *item) : user_var_item(item) {}
  int resolve(THD *thd) override;
  int check(THD *thd) override;
  int update(THD *thd) override;
  ...
};
```
> ★ 用户变量**唯一不走 `system_variables` hash** 的一类——`set_var_user` 只是把它塞进同一条 SET 执行管线（resolve/check/update）而已。

**★ enum_var_type：8.0 新增的两个作用域**（`sql/set_var.h:88`）：

```cpp
enum enum_var_type {
  OPT_DEFAULT,      /* [GLOBAL|LOCAL|SESSION].name */
  OPT_SESSION,      /* [SESSION|LOCAL].name */
  OPT_GLOBAL,       /* GLOBAL.name */
  OPT_PERSIST,      /* PERSIST.name */
  OPT_PERSIST_ONLY  /* PERSIST_ONLY.name */
};
```
> ★ `OPT_PERSIST` / `OPT_PERSIST_ONLY` 是 8.0 新增（5.7 只有前三个），与 GLOBAL/SESSION **平级**——这是变量体系在 8.0 最大的结构性扩展。

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

**写路径**（谁在改变量值）：

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
```

**读路径**（谁在读变量值）——8.0 的"汇聚点"是 PFS 表：

```
读变量值
├─ SHOW [GLOBAL|SESSION] VARIABLES ──(语法重写)──► pfs.global_variables / session_variables
├─ SHOW [GLOBAL|SESSION] STATUS   ──(语法重写)──► pfs.global_status / session_status
├─ SELECT FROM pfs.variables_by_thread / status_by_thread / status_by_user/host/account
├─ SELECT FROM pfs.variables_info / persisted_variables / user_variables_by_thread
└─ SHOW ENGINE INNODB STATUS/MUTEX ──(唯一不经过 PFS 的例外)──► handlerton::show_status
                                                                 └─► protocol 直出长文本（1 MiB 截断）
```

**元数据卫星**（变量相关的周边对象）：

```
mysql.component      ← 组件注册的系统变量（卸载即摘除）
mysql.gtid_executed  ← GTID 变量的持久化落点（见 replication/gtid.md）
sys.metrics          ← 基于 pfs.global_status 等表的视图
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

**★ 官方立场原文**（`storage/perfschema/pfs.cc:298`，★ "变量/状态归 PFS、元数据归 I_S"的分工声明）：

```cpp
  @section PFS_MAIN_INTRO Introduction
  The PERFORMANCE_SCHEMA is a way to introspect the internal execution of
  the server at runtime.
  The performance schema focuses primarily on performance data,
  as opposed to the INFORMATION_SCHEMA whose purpose is to inspect metadata.
```

**★ 8.0 新增的三个 flag**（`sql/set_var.h:128`，★ 都是为 `SET PERSIST` 加的）：

```cpp
  enum flag_enum {
    GLOBAL = 0x0001,
    SESSION = 0x0002,
    ONLY_SESSION = 0x0004,
    SCOPE_MASK = 0x03FF,  // 1023
    READONLY = 0x0400,    // 1024
    ALLOCATED = 0x0800,   // 2048
    INVISIBLE = 0x1000,   // 4096
    TRI_LEVEL = 0x2000,   // 8192 - default is neither GLOBAL nor SESSION
    NOTPERSIST = 0x4000,
    HINT_UPDATEABLE = 0x8000,  // Variable is updateable using SET_VAR hint
    PERSIST_AS_READ_ONLY = 0x10000,
    /**
      Sensitive variable. If keyring is available, the variable will be
      persisted in mysqld-auto.cnf in encrypted format
    */
    SENSITIVE = 0x20000
  };
```
```cpp
/* sql/set_var.h:280 —— "动态 vs 只读"就是一行位测试 */
  bool is_readonly() const { return flags & READONLY; }
  bool not_visible() const { return flags & INVISIBLE; }
  bool is_trilevel() const { return flags & TRI_LEVEL; }
  bool is_persist_readonly() const { return flags & PERSIST_AS_READ_ONLY; }
  bool is_sensitive() const { return flags & SENSITIVE; }
```
> ★ 两个易混概念：`READONLY` = 运行期不可改；`PERSIST_AS_READ_ONLY` = **本身可动态改，但不允许被 PERSIST 持久化**。

**★ `DEPRECATED_VAR` 是个恒等宏**（`sql/sys_vars.h:155`，★ 常见误解）：

```cpp
#define DEPRECATED_VAR(X) X
```
> ★ 它**不做任何运行时行为**，只是在构造参数列表里占一个位置，把"替代名"字符串传给 `sys_var` 的 `deprecation_substitute` 成员。8.0.39 中 `sql/sys_vars.cc` 共出现 **35 处**。
```cpp
/* sql/set_var.cc:479 —— 告警触发点（★ 解析期，不是执行期） */
void sys_var::do_deprecated_warning(THD *thd) {
  if (deprecation_substitute != nullptr) {
    char buf1[NAME_CHAR_LEN + 3];
    strxnmov(buf1, sizeof(buf1) - 1, "@@", name.str, 0);
    /*
       if deprecation_substitute is an empty string,
       there is no replacement for the syntax
    */
    uint errmsg = deprecation_substitute[0] == '\0'
                      ? ER_DEPRECATE_MSG_NO_REPLACEMENT
                      : ER_DEPRECATE_MSG_WITH_REPLACEMENT;
    if (thd)
      push_warning_printf(thd, Sql_condition::SL_WARNING, ER_WARN_DEPRECATED_SYNTAX, ...);
    else
      LogErr(WARNING_LEVEL, errmsg, buf1, deprecation_substitute);   /* 启动期写 error log */
  }
}
```
> ★ 参数甚至可以是"转到某张表"这种非变量名说明（`sys_vars.cc:6042`：`DEPRECATED_VAR("performance_schema.tls_channel_status table")`）。

**⚠️ WL#8688 澄清**：`sql/persisted_variable.cc` 中 grep `WL#` / `8688` → **0 命中**。SET PERSIST 的 WL 编号**不在源码注释里**。持久化缓存类的注释在 `sql/persisted_variable.h:110`（中文注释）。

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

#### `sys_var` 基类解剖：成员与虚函数契约

基类成员分四组（`sql/set_var.h`，构造时初始化）：

| 组 | 成员 | 说明 |
|---|---|---|
| 身份 | `name`（`LEX_CSTRING`）、`next`（挂链指针） | 名字 + `all_sys_vars` 链 |
| 元数据 | `flags`、`m_parse_flag`、`show_val_type`、`deprecation_substitute`、`binlog_status` | 作用域/只读等 flag、PARSE_EARLY/NORMAL、SHOW 类型、废弃替代名 |
| 定位与锁 | `offset`、`guard`（`PolyLock*`） | **offset 是灵魂**——指向值存储的偏移；guard 是第二把锁 |
| 钩子与来源 | `on_check`/`pre_update`/`on_update`、`source`/`user`/`host`/`timestamp` | 校验/更新回调、VARIABLE_SOURCE 追踪 |
| 桥接 | 内嵌 `my_option option`（值语义） | 与 getopt 共享值地址与来源（`option.value = global_var_ptr()`、`option.arg_source = &source`） |

**虚函数契约**——`sys_var` 定义了"SET 语句如何读写一个变量"的完整协议：

```cpp
private:                                          // 私有纯虚：每个具体变量类必须实现
  virtual bool do_check(THD *thd, set_var *var) = 0;              // 值合法性（范围/格式）
  virtual void session_save_default(THD *thd, set_var *var) = 0;  // SET ... = DEFAULT 的会话默认
  virtual void global_save_default(THD *thd, set_var *var) = 0;
  virtual bool session_update(THD *thd, set_var *var) = 0;        // 写会话值
  virtual bool global_update(THD *thd, set_var *var) = 0;         // 写全局值

protected:
  virtual const uchar *session_value_ptr(THD *running_thd, THD *target_thd, ...);  // 读会话值
  virtual const uchar *global_value_ptr(THD *thd, ...);                             // 读全局值
  uchar *session_var_ptr(THD *thd);   // &thd->variables + offset
  uchar *global_var_ptr();            // &global_system_variables + offset
```

模板化的公共骨架（非虚，定义整体流程，内部调上面的虚函数）：

```cpp
bool sys_var::check(THD *thd, set_var *var) {
  if ((var->value && do_check(thd, var)) || (on_check && on_check(this, thd, var))) {
    ... my_error(ER_WRONG_VALUE_FOR_VAR, name.str, ...); return true;
  }
  return false;
}

bool sys_var::update(THD *thd, set_var *var) {
  if (pre_update && pre_update(this, thd, var)) return true;    // 加锁前
  if (type == OPT_GLOBAL || type == OPT_PERSIST || scope() == GLOBAL) {
    AutoWLock lock1(&PLock_global_system_variables);
    AutoWLock lock2(guard);
    return global_update(thd, var) || (on_update && on_update(this, thd, OPT_GLOBAL));
  } else {
    mysql_mutex_lock(&thd->LOCK_thd_sysvar);
    bool ret = session_update(thd, var) || (on_update && on_update(this, thd, OPT_SESSION));
    mysql_mutex_unlock(&thd->LOCK_thd_sysvar);
    ...  // session tracker 通知
    return ret;
  }
}
```

这就是模板方法模式的落地：**基类定义"check = 类型钩子 + 业务钩子"、"update = 拿锁 + 写值 + 通知"的骨架，子类只填充 `do_check`/`*_update` 这些原语**。绝大多数子类（如 `Sys_var_integer`）的 `global_update` 只有一行 `global_var(T) = save_result.ulonglong_value`。

辅助方法：`check_scope(type)`（作用域判定，`ONLY_SESSION` 拒绝 SET GLOBAL）、`is_readonly()`/`is_trilevel()`/`is_hint_updateable()`/`is_persist_readonly()`/`is_sensitive()`/`is_non_persistent()`（flag 位查询）、`show_type()`/`charset()`（供 SHOW 格式化）、`value_ptr(running, target, type)`（读值统一入口，按 type 分派 global/session_value_ptr）、`set_default()`（`SET ... = DEFAULT` 路径）。

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

#### hash 表本身的实现

两个 hash 都是 `collation_unordered_map<std::string, sys_var *>`——`std::unordered_map` 的定制三件套（`include/map_helpers.h`）：

```cpp
class collation_unordered_map
    : public std::unordered_map<Key, Value, Collation_hasher,
                                Collation_key_equal,
                                Malloc_allocator<std::pair<const Key, Value>>> {
 public:
  collation_unordered_map(const CHARSET_INFO *cs, PSI_memory_key psi_key)
      : std::unordered_map<...>(/*bucket_count=*/10, Collation_hasher(cs),
                                Collation_key_equal(cs),
                                Malloc_allocator<>(psi_key)) {}
};

class Collation_hasher {
 public:
  size_t operator()(const std::string &s) const {
    uint64 nr1 = 1, nr2 = 4;
    hash_sort(cs, pointer_cast<const uchar *>(s.data()), s.size(), &nr1, &nr2);
    return nr1;                          // 用 collation 的 hash_sort 哈希
  }
};
```

三个定制点的含义：

1. **`Collation_hasher` + `Collation_key_equal`**：哈希与相等比较都走 MySQL collation（构造传 `system_charset_info`，`sys_var_init()` 里 `new collation_unordered_map(system_charset_info, PSI_INSTRUMENT_ME)`）。`*_ci` collation 大小写不敏感——这就是 `SET @@max_connections` / `@@MAX_CONNECTIONS` / `@@Max_Connections` 都命中同一个变量的原因。
2. **`Malloc_allocator` + PSI key**：hash 的节点内存走 `my_malloc` 且可被 `performance_schema.memory/SQL` 追踪（`PSI_INSTRUMENT_ME`），bucket 初始 10。
3. **key 是 `std::string`**（变量名小写归一后），value 是 `sys_var*` 描述符指针——hash 里**不存值**，值永远在 `global_system_variables`/`THD::variables`（offset 定位），hash 只是"名字 → 描述符"的字典。

#### hash 的消费者全景（谁在什么场景用它）

| 场景 | 代码路径 | 方式 |
|---|---|---|
| `SET xxx=...` 解析 | `intern_find_sys_var()`（`sql/set_var.cc`） | 先查 static hash（**免锁**），未命中再持 `LOCK_system_variables_hash(rd)` 查 dynamic——按名字 O(1) 定位 `sys_var` |
| PFS 物化 | `enumerate_sys_vars` / `init_show_var_array`（`pfs_variable.cc`） | **遍历**两个 hash 构建 tracker 数组（快照 + 版本号 `m_version`） |
| `get_row_count` 数行 | `get_system_variable_count()`（`set_var.cc`，= 两个 hash 的 `size()` 之和） | 表打开时 `ha_perfschema::info(HA_STATUS_VARIABLE)` 调用，持锁读 size——高频 `SHOW VARIABLES` 场景下这里是锁竞争点 |
| SET 的 O(1) 定位 vs PFS 的 O(N) 遍历 | — | 同一个 hash，两种用法：按名点查 vs 全量枚举 |

> 流传的旧材料（含部分网络文章）把 `SHOW VARIABLES` 的消费者写成 `mysqld_show_variables`（`sql_show.cc`）——**该函数在 8.0.39 中 0 匹配**，是 5.7 的历史实现。8.0 的 SHOW VARIABLES 走 PFS 表（本篇 H 节），hash 的消费入口是 PFS 的 `enumerate_sys_vars`。

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

**★ 源码实证**（`mysys/my_default.cc:926`）：

```cpp
  const int max_recursion_level = 10;   /* ★ !include 递归深度硬上限 10 层 */
  ...
  while (true) {
    auto fileline = mysql_file_getline(buff, sizeof(buff), fp, is_login_file);
    char *linebuff = fileline.get();
    if (linebuff == nullptr) break;     /* ★ 一次只读一行 → 没有"行尾 \ 续行" */

    line++;
    /* Ignore comment and empty lines */
    for (ptr = linebuff; my_isspace(&my_charset_latin1, *ptr); ptr++) {}
    if (*ptr == '#' || *ptr == ';' || !*ptr) continue;   /* ★ 行首 # 和 ; 都整行跳过 */
```

**① 组名与"选项必须在组之后"**：

```cpp
    if (*ptr == '[') /* Group name */
    {
      found_group = true;
      if (!(end = strchr(++ptr, ']'))) {
        my_message_local(ERROR_LEVEL,
                         EE_INCORRECT_GRP_DEFINITION_IN_CONFIG_FILE, name, line);
        goto err;
      }
      /* Remove end space */
      for (; my_isspace(&my_charset_latin1, end[-1]); end--) {}
      end[0] = 0;
      strmake(curr_gr, ptr, std::min<size_t>((end - ptr) + 1, sizeof(curr_gr) - 1));
      opt_handler(handler_ctx, curr_gr, nullptr, nullptr);   /* signal new group */
      continue;
    }
    if (!found_group) {                    /* ★ 选项出现在 [group] 之前 = 致命错误 */
      my_message_local(ERROR_LEVEL, EE_OPTION_WITHOUT_GRP_IN_CONFIG_FILE, name, line);
      goto err;
    }
```

**② 行尾注释只认 `#`，且必须在引号外**（`:1154`）：

```cpp
static char *remove_end_comment(char *ptr) {
  char quote = 0;  /* we are inside quote marks */
  char escape = 0; /* symbol is protected by escape chagacter */

  for (; *ptr; ptr++) {
    if ((*ptr == '\'' || *ptr == '\"') && !escape) {
      if (!quote) quote = *ptr;
      else if (quote == *ptr) quote = 0;
    }
    /* We are not inside a string */
    if (!quote && *ptr == '#') { *ptr = 0; return ptr; }
    escape = (quote && *ptr == '\\' && !escape);
  }
  return ptr;
}
```
> ★ 注意调用条件：`if (!is_login_file) end = remove_end_comment(ptr);` —— **login 文件（`mylogin.cnf`）不支持注释**。

**③ 等号两侧空格 + 成对引号剥离**（`:1078`）：

```cpp
    if ((value = strchr(ptr, '='))) end = value; /* Option without argument */
    for (; my_isspace(&my_charset_latin1, end[-1]); end--) {}
    /* ★ end 回退吃掉 '=' 左侧尾部空白 → "key  = v" 与 "key= v" 等价 */

    if (!value) {
      /* 无 '=' 的写法：key value 或裸 key —— 统一加 "--" 前缀 */
      strmake(my_stpcpy(option, "--"), ptr, (size_t)(end - ptr));
      if (opt_handler(handler_ctx, curr_gr, option, name)) goto err;
    } else {
      for (value++; my_isspace(&my_charset_latin1, *value); value++) {}  /* 吃掉右侧前导空白 */
      value_end = strend(value);
      for (; my_isspace(&my_charset_latin1, value_end[-1]); value_end--) {} /* 吃掉值尾部空白 */

      /* remove quotes around argument */
      if ((*value == '\"' || *value == '\'') && (value + 1 < value_end) &&
          *value == value_end[-1]) {      /* ★ 只剥"首尾成对"且长度 > 1 的引号 */
        value++; value_end--;
      }

      ptr = my_stpnmov(my_stpcpy(option, "--"), ptr, (size_t)(end - ptr));
      *ptr++ = '=';                       /* ★ 最终拼成 "--key=value" */
```

**④ 转义只在值内部生效**（`:1116`）：

```cpp
      for (; value != value_end; value++) {
        if (*value == '\\' && value != value_end - 1) {
          switch (*++value) {
            case 'n': *ptr++ = '\n'; break;
            case 't': *ptr++ = '\t'; break;
            case 'r': *ptr++ = '\r'; break;
            case 'b': *ptr++ = '\b'; break;
            case 's': *ptr++ = ' '; break;  /* space */
            case '\"': *ptr++ = '\"'; break;
            case '\'': *ptr++ = '\''; break;
            case '\\': *ptr++ = '\\'; break;
            default: /* Unknown; Keep '\' */
              *ptr++ = '\\'; *ptr++ = *value; break;
          }
        } else
          *ptr++ = *value;
      }
```
> ★ 未知转义**原样保留反斜杠**——所以 `\d` 会变成字面的 `\d`。

#### B.5 `!include` / `!includedir`

在 `search_default_file_with_ext` 里行首 `!` 开头解析：`!include` 递归读指定文件、`!includedir` 遍历目录中扩展名匹配的文件（Unix `.cnf`），**递归上限 10 层**（超限 WARNING 跳过）。被 include 的文件**继承当前 group 状态**，其选项按"include 处的位置"插入 argv（同 group 内仍是 last-wins）。include 的文件路径会登记进 `default_paths`（source 追踪用）。目录打不开是 fatal。

#### B.6 选项前缀（`my_getopt` 的 `special_opt_prefix`）

前缀表只有 5 个：`skip / disable / enable / maximum / loose`（**没有 `--minimum-`**）。语义：

- `--skip-x` / `--disable-x` → 值强制 `"0"`；但 `--skip-x=0` 是**双重否定** → `"1"`。`--enable-x` 对称（`=0` → 假）。
- `--maximum-x=N` → 写 `my_option::u_max_value`（`max_system_variables` 的对应偏移，即 `--maximum-max_connections` 这类软上限）。
- `--loose-x` → 未知选项降级为 WARNING 且不退出（`--loose-` 剥掉后重查表）。**persisted 变量全部以 `--loose_` 前缀生成**——保证 mysqld-auto.cnf 里有过时变量时服务器不至于起不来。
- 前缀可叠加（`--loose-skip-innodb`），剥离循环 `i = -1` 重启。

**★ 前缀表定义**（`mysys/my_getopt.cc:76`，★ 三个数组/枚举必须同序）：

```cpp
/*
  The following three variables belong to same group and the number and
  order of their arguments must correspond to each other.
*/
static const char *special_opt_prefix[] = {"skip",    "disable", "enable",
                                           "maximum", "loose",   nullptr};
static const uint special_opt_prefix_lengths[] = {4, 7, 6, 7, 5, 0};
enum enum_special_opt { OPT_SKIP, OPT_DISABLE, OPT_ENABLE, OPT_MAXIMUM, OPT_LOOSE };

char *disabled_my_option = const_cast<char *>("0");
static char enabled_my_option[] = "1";
```
> ★ 分隔符接受 `-` 或 `_`，所以 `--skip_xxx` 与 `--skip-xxx` 都行。

**★ 语义处理**（`mysys/my_getopt.cc:397`）：

```cpp
            for (int i = 0; special_opt_prefix[i]; i++) {
              if (!getopt_compare_strings(special_opt_prefix[i], opt_str,
                                          special_opt_prefix_lengths[i]) &&
                  (opt_str[special_opt_prefix_lengths[i]] == '-' ||
                   opt_str[special_opt_prefix_lengths[i]] == '_')) {
                opt_str += special_opt_prefix_lengths[i] + 1;
                length -= special_opt_prefix_lengths[i] + 1;
                if (i == OPT_LOOSE) option_is_loose = true;   /* ★ 只置标志，宽容在下面生效 */

                if ((opt_found = findopt(opt_str, length, &optp))) {
                  switch (i) {
                    case OPT_SKIP:
                    case OPT_DISABLE:
                      /*
                        double negation is actually enable again,
                        for example: --skip-option=0 -> option = true
                      */
                      optend = (optend && *optend == '0' && !(*(optend + 1)))
                                   ? enabled_my_option : disabled_my_option;
                      break;
                    case OPT_ENABLE:
                      optend = (optend && *optend == '0' && !(*(optend + 1)))
                                   ? disabled_my_option : enabled_my_option;
                      break;
                    case OPT_MAXIMUM:
                      set_maximum_value = true;
                      must_be_var = true;
                      break;
                  }
                  break;
                }
                i = -1;   /* ★ 一个前缀匹配不到就重启扫描 → 允许 --skip-loose-xxx 叠加 */
              }
            }
```

**★ `--loose-` 的精确语义**（`mysys/my_getopt.cc:457`）：

```cpp
            if (must_be_var) {
              if (my_getopt_print_errors)
                my_getopt_error_reporter(
                    option_is_loose ? WARNING_LEVEL : ERROR_LEVEL,
                    EE_UNKNOWN_VARIABLE, cur_arg);
              if (!option_is_loose) return EXIT_UNKNOWN_VARIABLE;   /* ★ 降级为 WARNING 且不退出 */
            } else {
              ...
              if (!(option_is_loose || ignore_unknown_option))
                return EXIT_UNKNOWN_OPTION;
            }
            if (option_is_loose || ignore_unknown_option) {
              (*argc)--;    /* ★ 把该参数从 argv 里剔除 */
              continue;
            }
```

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

#### B.9 `setval`：字符串如何变成值写进内存

命令行/配置文件解析的**终点**是 `setval()`（`mysys/my_getopt.cc`）——它把字符串 `argument` 按 `var_type` 解析成类型化值，**直接写进 `value` 指针指向的内存**（`value = opts->value`，正是 `sys_var` 的 `global_var_ptr()` = `&global_system_variables + offset`）：

```cpp
static int setval(const struct my_option *opts, void *value,
                  const char *argument, bool set_maximum_value, ...) {
  ulong var_type = opts->var_type & GET_TYPE_MASK;

  if (!argument) argument = enabled_my_option;      // 无值布尔默认 "1"

  // 数值类型拒绝空值（--xxx= 是错误），字符串空值是"重置默认"的合法手段
  if (!*argument && (var_type == GET_INT || ... GET_ENUM)) {
    my_getopt_error_reporter(ERROR_LEVEL, EE_OPTION_WITH_EMPTY_VALUE, ...);
    return EXIT_ARGUMENT_REQUIRED;
  }

  if (value) {
    if (set_maximum_value && !(value = opts->u_max_value)) return ...;  // --maximum- 写 u_max_value
    switch (var_type) {
      case GET_BOOL:
        *((bool *)value) = boolean_as_int ? get_bool_int_argument(argument, &error)
                                          : get_bool_argument(argument, &error);
        break;
      case GET_INT:
        *((int *)value) = (int)getopt_ll(argument, set_maximum_value, opts, &err);
        break;
      case GET_ULONG:
        *((long *)value) = (long)getopt_ull(argument, set_maximum_value, opts, &err);
        break;
      case GET_STR:
        *static_cast<const char **>(value) = argument;   // 字符串直接存指针
        break;
      case GET_ENUM: {
        int type = find_type(argument, opts->typelib, FIND_TYPE_BASIC);  // "ON"/"OFF" 找枚举
        if (type == 0) { /* 接受整数表示 */ *(ulong *)value = strtoul(argument, ...); }
        else *(ulong *)value = type - 1;
      } break;
      case GET_SET:
        *(static_cast<ulonglong *>(value)) = find_typeset(argument, opts->typelib, &err);
        break;
      ...
    }
  }
  ...
}
```

逐段解释——这就是"命令行/配置文件选项 = 系统变量"的最后一步：

1. **`value` 就是变量值的内存地址**：`my_handle_options2` 里 `value = optp->value`，而 `sys_var` 构造时 `option.value = global_var_ptr()`（= `&global_system_variables + offset`）。所以 `setval` 的 `*((int*)value) = ...` 就是**直接改写全局变量内存**，与 `SET GLOBAL` 的 `global_update` 写同一地址。
2. **数值走 `getopt_ll`/`getopt_ull`**：内部做范围校验（`opts->min_value/max_value/block_size`），超出报 `EE_OPTION_OUT_OF_RANGE` 或按 block_size 对齐——这是命令行阶段就完成的"取值合法性"检查，与 `SET` 语句的 `do_check` 对应。
3. **枚举/集合走 `find_type`/`find_typeset`**：把 `"ON"`、`"OFF,STRICT_TRANS_TABLES"` 这类文本用 typelib 查表转成整数/位图（`sql_mode` 这类 `GET_SET` 在这里完成字符串→位图的转换）。
4. **`--maximum-x=N` 写 `u_max_value`**（`set_maximum_value` 分支）：改写 `max_system_variables` 的对应偏移，而不是 `global_system_variables`——这是"软上限"的存储位置。
5. **布尔**：`get_bool_argument` 接受 `1/0/ON/OFF/TRUE/FALSE/Y/N` 等，非法值告警。

对比 `SET` 语句路径：命令行走 `setval`（getopt 的通用解析），`SET` 走 `sys_var::do_check` + `session/global_update`（sys_var 的虚函数链）——**两条路径最终写同一个地址**，这是"命令行选项就是系统变量"的代码级证据。

### C. GLOBAL / SESSION 双份存储

#### 三层存储模型（hash 与 global_system_variables 的关系）

这是整个变量体系最核心的认知点，回答"到底存了几份"：

```
【第 0 层 描述符】编译期静态对象，几百个 sys_var，躺在 .data 段
    每个含：名字/类型/范围/offset/锁/回调 + 内嵌 my_option

【第 1 层 索引】堆上两个 collation_unordered_map（A 节剖析过）
    static_system_variable_hash   内置变量名 → sys_var*
    dynamic_system_variable_hash  插件/组件变量名 → sys_var*
    —— 这是"目录/字典"，存的是【指针】，不是值

【第 2 层 值】内存里真正的"数据"
    global_system_variables      1 份，全局值（+ dynamic_variables_ptr 插件动态块）
    max_system_variables         1 份，--maximum- 软上限（与 global 同布局）
    THD::variables               每连接 1 份，会话值
```

**不是 2×2=4 份值存储。** hash 是"索引"，值存储是"数据"，二者通过 `sys_var->offset` 这座桥连接：

```
查 @@max_connections 的值（三步走）：
  ① hash["max_connections"]         → sys_var*          （名字 → 描述符）
  ② sys_var->offset                  → 偏移量            （描述符 → 位置）
  ③ &global_system_variables + offset → 值               （位置 → 值）
```

**hash 里没有值，值存储里没有名字**——名字只在描述符（`sys_var->name`）里，值只在值存储里，offset 把二者串起来。这就是"描述符与值分离"的落地形态。

为什么拆成三层（重点）：

- **值拆 global/session**：`SET GLOBAL` 只写 1 份 `global_system_variables`，N 个已存在连接各有自己的 `THD::variables` 副本，不受影响；新连接建立时一次性拷贝。
- **索引拆 static/dynamic**：static 编译期固定 → 查表**免锁**；dynamic 可增删（插件装卸）→ 持 `LOCK_system_variables_hash`。
- **描述符（offset）作桥**：同一份描述符代码 + 同一个 offset，`value_ptr(OPT_GLOBAL)` 走 `global_value_ptr`（读全局实例）、`value_ptr(OPT_SESSION)` 走 `session_value_ptr`（读某 THD）——一份描述符服务两份值。

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

**★ 宏本体：`-1` 是占位符**（`include/mysql/plugin.h:463`，⚠️ **不在 `sql/set_var.h`**，那里 grep `THDVAR` 0 命中）：

```cpp
#define MYSQL_THDVAR_BOOL(name, opt, comment, check, update, def)      \
  DECLARE_MYSQL_THDVAR_BASIC(name, bool) = {                           \
      PLUGIN_VAR_BOOL | PLUGIN_VAR_THDLOCAL | ((opt)&PLUGIN_VAR_MASK), \
      #name, comment, check, update,                                   \
      -1,          /* ★ offset 占位符：运行时回填 */                      \
      def,                                                             \
      NULL};       /* ★ resolve 占位符：运行时赋值 */

#define MYSQL_THDVAR_INT(name, opt, comment, check, update, def, min, max, blk) \
  DECLARE_MYSQL_THDVAR_SIMPLE(name, int) = {                                    \
      PLUGIN_VAR_INT | PLUGIN_VAR_THDLOCAL | ((opt)&PLUGIN_VAR_MASK),           \
      #name, comment, check, update, -1, def, min, max, blk, NULL};
```
```cpp
/* 对照 GLOBAL 版：第 6 项是 &varname（直接地址），不是 -1 */
#define MYSQL_SYSVAR_BOOL(name, varname, opt, comment, check, update, def) \
  DECLARE_MYSQL_SYSVAR_BASIC(name, bool) = {                                \
      PLUGIN_VAR_BOOL | ((opt)&PLUGIN_VAR_MASK),                            \
      #name, comment, check, update, &varname, def}
```
```cpp
/* :238 —— ★ 根源差异：GLOBAL 存"指针"，THD 存"offset" */
/*
  for global variables, the value pointer is the first
  element after the header, the default value is the second.
  for thread variables, the value offset is the first
  element after the header, the default value is the second.
*/
#define DECLARE_MYSQL_THDVAR_BASIC(name, type) \
  struct {                                     \
    MYSQL_PLUGIN_VAR_HEADER;                   \
    int offset;                                \
    const type def_val;                        \
    DECLARE_THDVAR_FUNC(type);                 \
  } MYSQL_SYSVAR_NAME(name)

#define DECLARE_THDVAR_FUNC(type) type *(*resolve)(MYSQL_THD thd, int offset)
```

**★ 访问器宏**（`include/mysql/plugin.h:617`，⚠️ 不是"取偏移算地址"）：

```cpp
#define SYSVAR(name) (*(MYSQL_SYSVAR_NAME(name).value))

/* when thd == null, result points to global value */
#define THDVAR(thd, name) \
  (*(MYSQL_SYSVAR_NAME(name).resolve(thd, MYSQL_SYSVAR_NAME(name).offset)))

#define THDVAR_SET(thd, name, value)                 \
  plugin_thdvar_safe_update(thd, MYSQL_SYSVAR(name), \
                            (char **)&THDVAR(thd, name), (const char *)value);
```
> ★ `THDVAR` 是**调用运行时装好的 resolve 函数指针** + 运行时回填的 offset。字符串型必须走 `THDVAR_SET`（处理 `MEMALLOC` 的内存释放）。

**★ 运行时回填**（`sql/sql_plugin.cc:3276`）：

```cpp
  for (plugin_option = tmp->plugin->system_vars;
       plugin_option && *plugin_option; plugin_option++, index++) {
    opt = *plugin_option;
    if (!(opt->flags & PLUGIN_VAR_THDLOCAL)) continue;
    if (!(register_var(plugin_name_ptr, opt->name, opt->flags))) continue;
    /* ★ register_var 在这里把 offset 从 -1 改成真实偏移 */
    switch (opt->flags & PLUGIN_VAR_TYPEMASK) {
      case PLUGIN_VAR_BOOL:   ((thdvar_bool_t *)opt)->resolve = mysql_sys_var_bool; break;
      case PLUGIN_VAR_LONG:   ((thdvar_ulong_t *)opt)->resolve = mysql_sys_var_ulong; break;
      case PLUGIN_VAR_STR:    ((thdvar_str_t *)opt)->resolve = mysql_sys_var_str; break;
      case PLUGIN_VAR_DOUBLE: ((thdvar_double_t *)opt)->resolve = mysql_sys_var_double; break;
      default:
        LogErr(ERROR_LEVEL, ER_PLUGIN_UNKNOWN_VARIABLE_TYPE, opt->flags, plugin_name);
        return -1;
    };
  }
```
```cpp
/* sql/sql_plugin.cc:2986 —— 每种标量一个 resolve 回调 */
static unsigned long *mysql_sys_var_ulong(THD *thd, int offset) {
  return (unsigned long *)intern_sys_var_ptr(thd, offset, true);
}
```

**★★ 惰性 COW 的三步**（`sql/sql_plugin.cc:2907`）：

```cpp
  thd->variables.dynamic_variables_ptr = (char *)my_realloc(
      key_memory_THD_variables, thd->variables.dynamic_variables_ptr,
      global_variables_dynamic_size, MYF(MY_WME | MY_FAE | MY_ALLOW_ZERO_PTR));
  /* ① 扩到与全局同样大（首次调用时 ptr 为 NULL） */

  memcpy(thd->variables.dynamic_variables_ptr +
             thd->variables.dynamic_variables_size,
         global_system_variables.dynamic_variables_ptr +
             thd->variables.dynamic_variables_size,
         global_system_variables.dynamic_variables_size -
             thd->variables.dynamic_variables_size);
  /* ② ★ 只 memcpy "新增的那一段"，老段保持本线程已改过的值 —— 增量 COW */

  for (const auto &key_and_value : *malloced_string_type_sysvars_bookmark_hash) {
    ...
    varoff = *(int *)(pi->plugin_var + 1);   /* ★ plugin_var + 1 正是 int offset */
    thdvar = (char **)(thd->variables.dynamic_variables_ptr + varoff);
    sysvar = (char **)(global_system_variables.dynamic_variables_ptr + varoff);
    *thdvar = nullptr;
    plugin_var_memalloc_session_update(thd, nullptr, thdvar, *sysvar);
    /* ③ ★ MEMALLOC 的字符串型必须逐条 strdup，否则共享指针会 double free */
  }
```
```cpp
/* sql/sql_plugin_var.cc:184 —— 惰性入口 */
uchar *intern_sys_var_ptr(THD *thd, int offset, bool global_lock) {
  if (!thd)
    return (uchar *)global_system_variables.dynamic_variables_ptr + offset;
  if (!thd->variables.dynamic_variables_ptr ||
      (uint)offset > thd->variables.dynamic_variables_head) {
    /* Current THD only. Don't trigger resync on remote THD. */
    if (current_thd == thd)
      alloc_and_copy_thd_dynamic_variables(thd, global_lock);
    else
      return (uchar *)global_system_variables.dynamic_variables_ptr + offset;
  }
  return (uchar *)thd->variables.dynamic_variables_ptr + offset;
}
```
> ★★ "惰性"就在最后一处：**第一次访问某个插件 THDVAR 时才 realloc**；查**别的 THD** 的会话值时不触发扩容（直接退化为读全局值）。

**★ `PLUGIN_VAR_*` 标志**（⚠️ 真值在 `include/mysql/components/services/bits/system_variables_bits.h:63`，`plugin.h` 只是引用）：

```cpp
#define PLUGIN_VAR_UNSIGNED 0x0080  /**< The variable is unsigned */
#define PLUGIN_VAR_THDLOCAL 0x0100  /**< Variable is per-connection */
#define PLUGIN_VAR_READONLY 0x0200  /**< Server variable is read only */
#define PLUGIN_VAR_NOSYSVAR 0x0400  /**< Not a server variable */
#define PLUGIN_VAR_NOCMDOPT 0x0800  /**< Not a command line option */
#define PLUGIN_VAR_NOCMDARG 0x1000  /**< No argument for cmd line */
#define PLUGIN_VAR_RQCMDARG 0x0000  /**< Argument required for cmd line */
#define PLUGIN_VAR_OPCMDARG 0x2000  /**< Argument optional for cmd line */
#define PLUGIN_VAR_NODEFAULT 0x4000 /**< SET DEFAULT is prohibited */
#define PLUGIN_VAR_MEMALLOC 0x8000  /**< String needs memory allocated */
#define PLUGIN_VAR_NOPERSIST 0x10000
#define PLUGIN_VAR_PERSIST_AS_READ_ONLY 0x20000
#define PLUGIN_VAR_INVISIBLE 0x40000
#define PLUGIN_VAR_SENSITIVE 0x80000
```
> ★ `RQCMDARG = 0x0000` 是"默认"——不加 `OPCMDARG` 就是必带参数。

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

#### 锁全景与锁序（防死锁契约）

变量体系横跨"插件管理、会话、全局、状态"四类数据结构，锁有六把以上。防死锁的办法不是逐对分析，而是**一条全局锁序链**——所有路径按同一顺序拿锁。PFS 源码里留了正式契约（`pfs_variable.h` 的 LOCK PRIORITIES 注释）：

```
System Variables:
  LOCK_plugin_delete              (block plugin delete)
   LOCK_system_variables_hash
   LOCK_thd_data                  (block THD delete)
   LOCK_thd_sysvar                (block system variable updates,
                                   alloc_and_copy_thd_dynamic_variables)
     LOCK_global_system_variables (very briefly held)

Status Variables:
  LOCK_status
    LOCK_thd_data                 (block THD delete)
```

补上插件侧的两个既有顺序（`sql_plugin.cc` 多处一致：装载/卸载三件套 `LOCK_plugin_delete → LOCK_system_variables_hash(wr) → LOCK_plugin`；新连接 `plugin_thdvar_init` 里 `LOCK_global_system_variables → LOCK_plugin`），可以串成一条**总序**：

```
LOCK_plugin_delete → LOCK_system_variables_hash → LOCK_thd_data
  → LOCK_thd_sysvar → LOCK_global_system_variables → LOCK_plugin
```

（`LOCK_status → LOCK_thd_data` 是状态变量侧的另一条短链；`LOCK_persist_file` 独立；gtid_mode 的 `Gtid_mode::lock → channel_map → log_lock → global_sid_lock` 是业务侧独立链，见 [`../replication/gtid.md`](../replication/gtid.md)。）

各路径按序核对：

| 路径 | 拿锁顺序 | 依据 |
|---|---|---|
| `SET GLOBAL`（`sys_var::update` 全局分支） | `PLock_global_system_variables` → 变量 `guard` | set_var.cc：两把顺序拿，读路径 `value_ptr` 对称 |
| `SET SESSION` | `thd->LOCK_thd_sysvar` | 保护 `thd->variables` 与跨线程读 |
| PFS 物化会话变量 | `LOCK_plugin_delete` → `LOCK_system_variables_hash(rd)` → `get_THD`（`LOCK_thd_data`）→ `LOCK_thd_sysvar` → `LOCK_global_system_variables` | pfs_variable.h 注释 + do_materialize_* |
| 插件装载/卸载 | `LOCK_plugin_delete` → `LOCK_system_variables_hash(wr)` → `LOCK_plugin` | sql_plugin.cc 1197/1475/2049/2144/2325 |
| 新连接拷贝变量 | `LOCK_global_system_variables` → `LOCK_plugin` | `plugin_thdvar_init` |
| 插件变量增量 COW | `LOCK_system_variables_hash(rd)` → `LOCK_global_system_variables` | `alloc_and_copy_thd_dynamic_variables` |
| 状态变量聚合 | `LOCK_status` → `LOCK_thd_data` | pfs_variable.h + `do_materialize_global` |
| 用户变量读写 | `LOCK_thd_data` | `THD::user_vars` |
| PERSIST 落盘 | cache 内部 mutex → `m_LOCK_persist_file` | `flush_to_file`（独立于上链，文件锁） |

四个防死锁设计点：

1. **`pre_update` 提前到加锁前**：`sys_var::update` 开头注释明说——"Invoke preparatory step ... before we have acquired any locks allows to invoke code which acquires other locks without introducing deadlocks"。`on_update` 钩子（可能拿别的业务锁）被延迟到锁内执行是危险的，所以拆出一个加锁前的阶段。
2. **GLOBAL 写必须拿两把锁**：`PLock_global_system_variables` + `guard`。代码注释给出理由——"If we'll take only 'guard' here, then value_ptr() for strings won't be safe in SHOW VARIABLES anymore"（字符串变量的值指针在多线程下需要双锁保护，否则 SHOW 可能读到半截）。
3. **`LOCK_status` 非递归 + 手工引用计数**：`SHOW STATUS` 内部可能再次触发 PFS 物化（嵌套查询），`THD::fill_status_recursion_level` 手工计数"第一次才拿锁"（字段名是历史遗留，注释里提到的 `fill_status()` 函数早已不存在）。
4. **版本号作乐观快照**：`dynamic_system_variable_hash_version` / `status_var_array_version` 不是锁——物化前记录版本、期间检测变化，避免长事务持锁（读写锁只保护"枚举 hash"那一小段）。

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

**★ `check_priv()` 原文**（`sql/set_var.cc:195`）：

```cpp
bool check_priv(THD *thd, bool static_variable) {
  Security_context *sctx = thd->security_context();
  /* for dynamic variables user needs SUPER_ACL or SYSTEM_VARIABLES_ADMIN */
  if (!static_variable) {
    if (!sctx->check_access(SUPER_ACL) &&
        !(sctx->has_global_grant(STRING_WITH_LEN("SYSTEM_VARIABLES_ADMIN")).first)) {
      my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0),
               "SUPER or SYSTEM_VARIABLES_ADMIN");
      return true;
    }
  } else {
    /*
     for static variables user needs both SYSTEM_VARIABLES_ADMIN and
     PERSIST_RO_VARIABLES_ADMIN
    */
    if (!(sctx->has_global_grant(STRING_WITH_LEN("SYSTEM_VARIABLES_ADMIN")).first &&
          sctx->has_global_grant(STRING_WITH_LEN("PERSIST_RO_VARIABLES_ADMIN")).first)) {
      my_error(ER_PERSIST_ONLY_ACCESS_DENIED_ERROR, MYF(0),
               "SYSTEM_VARIABLES_ADMIN and PERSIST_RO_VARIABLES_ADMIN");
      return true;
    }
  }
  return false;
}
```
> ★★ 注意：**`PERSIST_ONLY` 只认动态权限，SUPER 不够**（这是 8.0 权限细化的关键点）。

**★★ "SET SESSION 通常不需要权限"的源码证据**（`sql/set_var.cc:1580`）：

```cpp
int set_var::resolve(THD *thd) {
  ...
    var->do_deprecated_warning(thd);
    if (var->is_readonly()) {
      if (type != OPT_PERSIST_ONLY) {
        my_error(ER_INCORRECT_GLOBAL_LOCAL_VAR, MYF(0), var->name.str, "read only");
        return -1;
      }
      /* ★ 只读变量唯一可写的口子就是 SET PERSIST_ONLY */
    }
    if (!var->check_scope(type)) { ... }
    if (type == OPT_GLOBAL || type == OPT_PERSIST) {
      if (check_priv(thd, false)) return -1;
    }
    if (type == OPT_PERSIST_ONLY) {
      if (check_priv(thd, true)) return -1;
    }
    /* ★★ OPT_SESSION / OPT_DEFAULT 分支根本不调 check_priv */
```
> ★ 例外：挂了 `ON_CHECK(check_session_admin*)` 的**敏感会话变量**仍需权限。

**★ 敏感会话变量的三档检查**（`sql/sys_vars.cc:300`）：

```cpp
static bool check_session_admin_or_replication_applier(sys_var *, THD *thd, set_var *setv) {
  assert(self->scope() != sys_var::GLOBAL);
  Security_context *sctx = thd->security_context();
  if ((setv->type == OPT_SESSION || setv->type == OPT_DEFAULT) &&
      !sctx->has_global_grant(STRING_WITH_LEN("REPLICATION_APPLIER")).first &&
      !sctx->has_global_grant(STRING_WITH_LEN("SESSION_VARIABLES_ADMIN")).first &&
      !sctx->has_global_grant(STRING_WITH_LEN("SYSTEM_VARIABLES_ADMIN")).first &&
      !sctx->check_access(SUPER_ACL)) {
    my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0),
             "SUPER, SYSTEM_VARIABLES_ADMIN, SESSION_VARIABLES_ADMIN or "
             "REPLICATION_APPLIER");
    return true;
  }
  return false;
}
```
> ⚠️ **真名核查**：`SESSION_VARIABLES_ADMIN` 确实存在，但它**不是 C enum**，而是**字符串型动态权限**（`has_global_grant(STRING_WITH_LEN(...))`），注册点 `sql/auth/dynamic_privileges_impl.cc:223`。
> ★ 三档：`check_session_admin_or_replication_applier`（最宽松，含 SUPER）、`check_session_admin_privileges_only`、`check_session_admin_no_super`（最严，只认两个 ADMIN）。

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

**★ `user_var_entry` 完整定义**（`sql/item_func.h:2953`，⚠️ **不在 `sql_class.h`**，那里只是前向声明）：

```cpp
class user_var_entry {
  ...
  /**
    Position inside a user_var_entry where small values are stored:
    double values, longlong values and string values with length
    up to extra_size (should be 8 bytes on all platforms).
    String values with length longer than 8 are stored in a separate
    memory buffer, which is allocated when needed using the method realloc().
  */
  char *internal_buffer_ptr() {
    return pointer_cast<char *>(this) + ALIGN_SIZE(sizeof(user_var_entry));
  }
  char *name_ptr() { return internal_buffer_ptr() + extra_size; }

  static const size_t extra_size = sizeof(double);   /* ★ 8 字节内联阈值 */
  char *m_ptr;         ///< Value
  size_t m_length;     ///< Value length
  Item_result m_type;  ///< Value type
  THD *m_owner;
  /**
    Set to the id of the most recent query that has used the variable.
    Used in binlogging: When set, there is no need to add a reference to this
    variable to the binlog. Imagine it is this:

        INSERT INTO t SELECT @a:=10, @a:=@a+1.

    Then we have a Item_func_get_user_var (because of the `@a+1`) so we
    think we have to write the value of `@a` to the binlog. But before that,
    we have a Item_func_set_user_var to create `@a` (`@a:=10`), in this we mark
    the variable as "already logged" so that it won't be logged
    by Item_func_get_user_var (because that's not necessary).
  */
  query_id_t m_used_query_id;

 public:
  Simple_cstring entry_name;  // Variable name
  DTCollation collation;
  bool unsigned_flag;

  void set_used_query_id(query_id_t query_id) { m_used_query_id = query_id; }
  query_id_t used_query_id() const { return m_used_query_id; }

  void destroy() {
    assert_locked();
    free_value();   // Free the external value buffer
    my_free(this);  // Free the instance itself
  }
  void lock();
  void unlock();
};
```
> ⚠️ **真名核查**：只有 **`m_used_query_id`** 一个字段 + 两个 accessor，**`update_query_id` 在 8.0.39 不存在**。
> ★★ 一个重要提醒：该字段**只记录"最近一次用过它的 query_id"，服务于 binlog 去重**，**不是**求值顺序标记——所以不能说"它保证了 `@a:=@a+1` 的求值顺序"（顺序由 Item 树求值保证）。

**★ 小值内联优化**（`sql/item_func.cc:6006`）：

```cpp
bool user_var_entry::mem_realloc(size_t length) {
  if (length <= extra_size) {
    /* Enough space to store value in value struct */
    free_value();
    m_ptr = internal_buffer_ptr();      /* ★ ≤8 字节存对象尾部，零额外分配 */
  } else {
    /* Allocate an external buffer */
    if (m_length != length) {
      if (m_ptr == internal_buffer_ptr()) m_ptr = nullptr;
      if (!(m_ptr = (char *)my_realloc(key_memory_user_var_entry_value, m_ptr,
                                       length, MYF(MY_ALLOW_ZERO_PTR | MY_WME |
                                                   ME_FATALERROR))))
        return true;
    }
  }
  return false;
}
```

**★ 查找真名是 `get_variable()` 不是 `get_var()`**（`sql/item_func.cc:5869`）：

```cpp
static user_var_entry *get_variable(THD *thd, const Name_string &name,
                                    const CHARSET_INFO *cs) {
  const std::string key(name.ptr(), name.length());
  mysql_mutex_assert_owner(&thd->LOCK_thd_data);
  user_var_entry *entry = find_or_nullptr(thd->user_vars, key);
  if (entry == nullptr && cs != nullptr) {     /* ★ cs==nullptr = 只读查找，不创建 */
    entry = user_var_entry::create(thd, name, cs);
    if (entry == nullptr) return nullptr;
    thd->user_vars.emplace(key, unique_ptr_with_deleter<user_var_entry>(
                                    entry, &free_user_var));
  }
  return entry;
}
```
> ★ 用 `collation_unordered_map` 而非 `HASH`，因为**用户变量名大小写不敏感**（`system_charset_info` 的 collation hasher）。
> ⚠️ **8.0 已废弃 `HASH hash_user_variables`**。

**★ 为什么用户变量要自带锁**（`sql/item_func.cc:6062`）：

```cpp
void user_var_entry::assert_locked() const {
  mysql_mutex_assert_owner(&m_owner->LOCK_thd_data);
}
```
> ★ 用户变量是会话私有的，但 PFS 的 `user_variables_by_thread` 会**从别的线程窥探**——所以每次读写都要 `LOCK_thd_data`。

**★ `@` 与 `@@` 在 token 层就分开了**（`sql/sql_yacc.yy:11554`，★ 不存在同名歧义）：

```cpp
          '@' ident_or_text
          { $$ = NEW_PTN PTI_user_variable(@$, $2); }
        | '@' '@' opt_rvalue_system_variable_type rvalue_system_variable
          { $$ = NEW_PTN PTI_get_system_variable(@$, $3, @4, $4.prefix, $4.name); }
```
> ★ 所以 `@max_connections` 永远是用户变量，`@@max_connections` 永远是系统变量。但**保留字不能做用户变量名**（`sql/item_func.cc:157` `check_reserved_words()`）。

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

> **先厘清：状态变量与前面 A~F 讲的系统变量是两套独立的体系**。它们唯一的共同点是——`SHOW STATUS` 与 `SHOW VARIABLES` **共用** `build_query` 这个重写函数，且状态变量表的列名恰好也叫 `VARIABLE_NAME`/`VARIABLE_VALUE`（所以列投影常量能复用）。除此之外，存储、索引、描述符、聚合语义全都不同。

| 维度 | 状态变量（status） | 系统变量（variables） |
|---|---|---|
| 能否改 | **只读**（无 SET 路径） | 可 `SET GLOBAL/SESSION/PERSIST` |
| 表 | `global_status`/`session_status`/`status_by_*` | `global_variables`/`session_variables`/`variables_by_thread` |
| 值存储 | `System_status_var`（**连续 ulonglong 块**，per-THD + 全局 1 份） | `global_system_variables` + `THD::variables`（`System_variables` 结构体） |
| 索引/目录 | **无 hash**——是 `all_status_vars`（`SHOW_VAR[]` 数组） | **两个 hash**（static + dynamic，名字→`sys_var*`） |
| 描述符 | `SHOW_VAR`（C 结构：name/value/type/scope），`value` 有三重身份（指针/偏移/函数/子数组） | `sys_var`（C++ 类树），靠 `offset` 定位值 |
| 类型系统 | `enum_mysql_show_type`（SHOW_LONG/LONGLONG/CHAR/FUNC/ARRAY） | `Sys_var_*` 类树（integer/enum/set/charptr/gtid_*） |
| GLOBAL 语义 | **聚合**：全局基线 + **Σ 所有 THD**（跨线程求和） | **选读**：直接读 `global_system_variables`（不求和） |
| 物化 cache | `PFS_status_variable_cache` | `PFS_system_variable_cache` |
| 求值链 | `get_one_variable`→`get_one_variable_ext`（`SHOW_*_STATUS` 型要**加 status_var 基址**解引用） | `get_one_variable_ext` 的 **SHOW_SYS 特判** → `sys_var::value_ptr` |
| 观测者效应 | **有**：`Sql_cmd_show_status::execute` 做"语句前快照 + 增量转移"（SHOW 本身会污染计数器） | **无**：读变量不污染任何东西，`Sql_cmd_show_variables` 是空子类 |
| 会话结束 | `add_to_status` 折进 `global_status_var`（有归并） | 会话值随 THD 销毁（无归并） |
| 名字生成 | 部分前缀**运行时拼接**（`Innodb_*` 由 `show_innodb_vars` 的 SHOW_FUNC→SHOW_ARRAY 拼出，源码 grep 不到） | 名字即注册名（插件变量会带 plugin 前缀） |
| 特殊过滤 | `filter_by_name`（`Com_*` **只在** SHOW STATUS 里出现） | 无此过滤 |
| 版本演进 | 8.0 重写为 PFS 查询 | 8.0 重写为 PFS 查询（同一批改造） |

**记住一句话**：系统变量是"**可写的配置项**"（有 hash 索引、offset 定位、双份存储），状态变量是"**只读的计数器**"（无 hash、连续内存块、跨线程求和）。它们只是恰好都在 PFS 里以 `VARIABLE_NAME/VARIABLE_VALUE` 两列的形式暴露出来。

#### 反直觉开场：SHOW STATUS 是 SELECT 的皮

8.0 的 `SHOW GLOBAL STATUS` 在语法分析阶段被重写成（`sql_show_status.cc` 文件头注释原文，表名是 `global_status`）：

```sql
SELECT * FROM
         (SELECT VARIABLE_NAME as Variable_name, VARIABLE_VALUE as Value
          FROM performance_schema.global_status) global_status
```

即：内层做列投影与别名（`VARIABLE_NAME as Variable_name`、`VARIABLE_VALUE as Value`），外层套一个派生表——**派生表别名就是表名本身**（`PT_derived_table(false, sub_query, table_name, ...)` 的第三个参数就是 `table_alias`）。`LIKE 'x%'` 被翻译成外层的 `WHERE Variable_name LIKE 'x%'`（`Item_func_like`），`WHERE <cond>` 则直接挂外层。末尾手工恢复 `lex->sql_command = SQLCOM_SHOW_STATUS`。

> **出处澄清**：这段注释是 `build_query` 的**函数头注释，以 SHOW STATUS 为例**写的。`build_query` 是通用函数，SHOW VARIABLES 走同一个函数、仅表名不同（`build_show_global_variables` 传 `global_variables`、`build_show_session_variables` 传 `session_variables`），代码里的常量（`VARIABLE_NAME`/`Variable_name`/`VARIABLE_VALUE`/`Value`/`performance_schema`）是共用的。所以 VARIABLES 版本的等价形式成立——**但注释原文里没有直接写出 VARIABLES 版本**，这是我按同一个函数的代码推出来的（常量 + 表名参数）。

三个连带结论：① `Com_*` 只在 SHOW STATUS 里出现（`filter_by_name` 按命令类型过滤整棵 `Com` 子树）；② `SHOW SESSION STATUS` 显示语句开始前快照；③ `Innodb_*` 前缀是运行时拼的，源码 grep 不到。

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

**★ 完整类型枚举**（`include/mysql/status_var.h:30`，★ 共 **23** 个取值）：

```cpp
enum enum_mysql_show_type {
  SHOW_UNDEF, SHOW_BOOL, SHOW_INT, SHOW_LONG, SHOW_LONGLONG,
  SHOW_CHAR, SHOW_CHAR_PTR, SHOW_ARRAY, SHOW_FUNC, SHOW_DOUBLE,
  /* ↓ 以下 server-only，插件不可用 */
  SHOW_KEY_CACHE_LONG, SHOW_KEY_CACHE_LONGLONG, SHOW_LONG_STATUS,
  SHOW_DOUBLE_STATUS, SHOW_HAVE, SHOW_MY_BOOL, SHOW_HA_ROWS, SHOW_SYS,
  SHOW_LONG_NOFLUSH, SHOW_LONGLONG_STATUS, SHOW_LEX_STRING,
  /* ↓ 有符号版，为插件扩展 */
  SHOW_SIGNED_INT, SHOW_SIGNED_LONG, SHOW_SIGNED_LONGLONG
};

/**
  Status variable scope.
  Only GLOBAL status variable scope is available in plugins.
*/
enum enum_mysql_show_scope {
  SHOW_SCOPE_UNDEF, SHOW_SCOPE_GLOBAL,
  /* Server-only values. Not supported in plugins. */
  SHOW_SCOPE_SESSION, SHOW_SCOPE_ALL
};
```

> ⚠️ **两点纠错**：① `SHOW_VAR` **不在 `sql/set_var.h`**（那里只有 `typedef enum enum_mysql_show_type SHOW_TYPE;`），真身在 **`include/mysql/status_var.h:80`**；② `SHOW_SCOPE_*` 是**独立的第二个枚举**，作为 `SHOW_VAR` 的第 4 个**独立字段**——**不存在任何"位或"宏**（常见说法 `SHOW_LONG|SHOW_SCOPE_GLOBAL` 是错的）。

**★ 三重身份的三处实据**：

**① 状态变量导出表**（`sql/mysqld.cc:9690`，★ 同一张表里四种 `value` 语义并存）：

```cpp
SHOW_VAR status_vars[] = {
    {"Aborted_clients", (char *)&aborted_threads, SHOW_LONG, SHOW_SCOPE_GLOBAL},
    {"Bytes_received", (char *)offsetof(System_status_var, bytes_received),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Com", (char *)com_status_vars, SHOW_ARRAY, SHOW_SCOPE_ALL},
    {"Connections", (char *)&show_thread_id_count, SHOW_FUNC,
     SHOW_SCOPE_GLOBAL},
    {"Flush_commands", (char *)&refresh_version, SHOW_LONG_NOFLUSH,
     SHOW_SCOPE_GLOBAL},
    {"Compression", (char *)&show_net_compression, SHOW_FUNC,
     SHOW_SCOPE_SESSION},
};
```

**② 系统变量导出**（`storage/perfschema/pfs_variable.cc:176`，★ 临时包装）：

```cpp
        const SHOW_VAR show_var{sysvar->name.str, pointer_cast<char *>(sysvar),
                                SHOW_SYS, SHOW_SCOPE_UNDEF};
```
> ★ 系统变量**不走** `status_vars[]`，而是在 PFS 侧把 `sys_var*` 塞进 `SHOW_VAR.value`，`type = SHOW_SYS`；scope 字段对系统变量无意义（`SHOW_SCOPE_UNDEF`）。

**③ PFS 表的行定义源**（`storage/perfschema/pfs_variable.cc:1042`）：

```cpp
  for (auto show_var : all_status_vars) {
    if (filter_show_var(&show_var, strict)) continue;
    if (show_var.type == SHOW_ARRAY) {
      /* Expand nested sub array. The name is used as a prefix. */
      expand_show_var_array((SHOW_VAR *)show_var.value, show_var.name, strict);
    } else {
      show_var.name = make_show_var_name(nullptr, show_var.name);
      m_show_var_array.push_back(show_var);
    }
  }
  m_show_var_array.push_back(SHOW_VAR());   /* Last element is NULL. */
```

**★ `SHOW_LONG_STATUS` 为什么必须存在**（`storage/perfschema/pfs_variable.cc:954`，★ PFS 靠类型判断是否可聚合）：

```cpp
bool PFS_status_variable_cache::can_aggregate(
    enum_mysql_show_type variable_type) {
  switch (variable_type) {
    /*
      All server status counters that are totaled across threads are defined in
      system_status_var as either SHOW_LONGLONG_STATUS or SHOW_LONG_STATUS.
      These data types are not available to plugins.
    */
    case SHOW_LONGLONG_STATUS:
    case SHOW_LONG_STATUS:
      return true;
    ...
    default:
      return false;
```
> ★★ 所以 `Com_xxx` 用 `SHOW_LONG_STATUS` 不是风格问题——**PFS 靠这个类型判断"能否跨线程求和"**，插件不可用这些类型，所以插件状态变量永远不参与聚合。

**★ `SHOW_LONG` vs `SHOW_LONG_STATUS` 的源码级差别**（`sql/sql_show.cc:3593`）：

```cpp
    case SHOW_LONG_STATUS:
      value = (char *)status_var + reinterpret_cast<size_t>(value);
      end = longlong10_to_str(*pointer_cast<const ulong *>(value), buff, 10);
      break;
      /* ★ 先把 value（其实是 offsetof 出来的整数）加上 status_var 基址
         → 同一条目在 SESSION 与 GLOBAL 下读的是不同内存 */

    case SHOW_LONG:
      /* the difference lies in refresh_status() */
    case SHOW_LONG_NOFLUSH:
      end = longlong10_to_str(*pointer_cast<const ulong *>(value), buff, 10);
      break;
      /* ★ 源码注释原话：差别"在于 refresh_status()"
         → SHOW_LONG 是绝对指针（全局裸变量），无 SESSION 语义，且 FLUSH STATUS 会清零 */
```

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

> **澄清：这不是"聚合到 hash"，状态变量根本没有 hash。** 状态变量与系统变量是两套存储：系统变量有 `static/dynamic_system_variable_hash`（名字→描述符的字典）；状态变量没有 hash，它的"目录"是 `all_status_vars`（一个 `SHOW_VAR[]` 描述符数组），"值"是 `System_status_var`（连续内存块）。公式里的 `Σ` 是**每次查询现场计算**——`do_materialize_global` 里 `visit_global(..., with_THDs=true)` 持锁遍历所有 THD，用 `add_to_status` 把每个 `thd->status_var` 累加进局部 `status_totals`，不是预先聚合好存在某个地方。

会话生命周期归并：连接结束 `THD::release_resources()` → `add_to_status(&global_status_var, ...)` + PSI `aggregate_thread_status()`（折进 account/user/host）+ `status_var_aggregated = true`；`CHANGE USER`/`RESET CONNECTION` 走 `cleanup_connection()`；顺序协议"先 release 再 remove_thd"保证不重不漏。

#### 平行账本：两套聚合通道必须隔离

`PFS_account::aggregate_status()` 的注释："Never aggregate to global_status_var, because of the parallel THD -> global_status_var flow."——account/user/host 维度与全局是**两条平行通道**，不交叉，否则双倍计数。`PFS_status_stats`（`ulonglong m_stats[COUNT_GLOBAL_STATUS_VARS]`）是断开会话的"遗留账本"：线程断开折进 account，account 清理折进 user/host。

**★★ 隔离的运行时守卫：`status_var_aggregated`**（`sql/sql_class.cc:1399`）：

```cpp
  mysql_mutex_lock(&LOCK_status);
  /* Add thread status to the global totals. */
  add_to_status(&global_status_var, &status_var);
#ifdef HAVE_PSI_THREAD_INTERFACE
  /* Aggregate thread status into the Performance Schema. */
  if (m_psi != nullptr) {
    PSI_THREAD_CALL(aggregate_thread_status)(m_psi);
  }
#endif
  /* Ensure that the thread status is not re-aggregated to the global totals. */
  status_var_aggregated = true;
  mysql_mutex_unlock(&LOCK_status);
```
> ★ 同一个 `status_var` 被"双花"到两个账本，靠 `status_var_aggregated` 保证只花一次。

```cpp
/* sql/handler.cc:1937 —— 一旦 aggregated 就不许再写，否则增量永久丢失 */
      assert(!thd->status_var_aggregated);
      thd->status_var.ha_commit_count++;
```
```cpp
/* sql/sql_class.cc:2379 —— 每个计数器都是"server + PFS"双写 */
void THD::inc_status_created_tmp_disk_tables() {
  assert(!status_var_aggregated);
  status_var.created_tmp_disk_tables++;
#ifdef HAVE_PSI_STATEMENT_INTERFACE
  PSI_STATEMENT_CALL(inc_statement_created_tmp_disk_tables)(m_statement_psi, 1);
#endif
}
```

**★★ "观测者效应"的经典解法**（`sql/sql_show.cc:754`，★ SHOW STATUS 不能污染自己的统计）：

```cpp
bool Sql_cmd_show_status::execute(THD *thd) {
  System_status_var old_status_var = thd->status_var;
  thd->initial_status_var = &old_status_var;

  bool status = Sql_cmd_show::execute(thd);

  // Don't log SHOW STATUS commands to slow query log
  thd->server_status &=
      ~(SERVER_QUERY_NO_INDEX_USED | SERVER_QUERY_NO_GOOD_INDEX_USED);
  // Restore status variables, as we don't want 'show status' to cause changes
  mysql_mutex_lock(&LOCK_status);
  add_diff_to_status(&global_status_var, &thd->status_var, &old_status_var);
  thd->status_var = old_status_var;
  thd->initial_status_var = nullptr;
  mysql_mutex_unlock(&LOCK_status);

  return status;
}
```
> ★★ 自指问题：`SHOW STATUS` 本身会消耗 `Bytes_sent` / `Com_show_status` / `Questions`。解法是**先快照、后只补差值**（`add_diff_to_status`）、最后还原。所以 `SHOW STATUS` 不会让 `Questions` 自己 +1。

**★ `add_to_status()`：把结构体当数组用**（`sql/system_variables.cc:28`）：

```cpp
void add_to_status(System_status_var *to_var, System_status_var *from_var) {
  int c;
  ulonglong *end = (ulonglong *)((uchar *)to_var +
                                 offsetof(System_status_var, LAST_STATUS_VAR) +
                                 sizeof(ulonglong));
  ulonglong *to = (ulonglong *)to_var, *from = (ulonglong *)from_var;

  while (to != end) *(to++) += *(from++);   /* ★ 连续块指针遍历求和 */

  to_var->com_other += from_var->com_other;
  for (c = 0; c < SQLCOM_END; c++)          /* ★ com_stat 是 ulong 数组，不在 ulonglong 块里 */
    to_var->com_stat[(uint)c] += from_var->com_stat[(uint)c];
}
```
```cpp
/* sql/system_variables.h:590 —— 连续块的首尾哨兵 */
#define LAST_STATUS_VAR secondary_engine_execution_count
#define FIRST_STATUS_VAR created_tmp_disk_tables
const int COUNT_GLOBAL_STATUS_VARS =
    ((offsetof(System_status_var, LAST_STATUS_VAR) -
      offsetof(System_status_var, FIRST_STATUS_VAR)) / sizeof(ulonglong)) + 1;
```
> ★★ 这就是为什么 `System_status_var` 开头必须是一段**连续同类型的 ulonglong**——注释明说 *"Must be long/ulong up to last_system_status_var so that add_to_status/add_diff_to_status can work"*。任何在中间插入非 ulonglong 成员的改动都会静默破坏聚合。

**★ 老实现的调用者只剩一个**（`sql/sql_show.cc:3736`）：

```cpp
class Add_status : public Do_THD_Impl {
 public:
  Add_status(System_status_var *value) : m_stat_var(value) {}
  void operator()(THD *thd) override {
    if (!thd->status_var_aggregated)
      add_to_status(m_stat_var, &thd->status_var);
  }
 private:
  System_status_var *m_stat_var;
};

void calc_sum_of_all_status(System_status_var *to) {
  mysql_mutex_assert_owner(&LOCK_status);
  *to = global_status_var;                    /* ★ 全局值作基线 */
  Add_status add_status(to);
  Global_THD_manager::get_instance()->do_for_all_thd_copy(&add_status);
}
```

#### `Com_xxx` 计数器

存储 `System_status_var::com_stat[(uint)SQLCOM_END]`，名字表 `com_status_vars[]`（`sql/mysqld.cc:3984`）。主计数点 `mysql_execute_command` 的 `thd->status_var.com_stat[lex->sql_command]++`（readonly 检查后、执行前）——**语法错误不计数、执行失败也计数**（官方：`Com_stmt_*` 对应"request issued"而非"successfully completed"）。协议命令（COM_PING 等）在 `dispatch_command` 手工 ++，复制 applier 也有手工补计数。

**★ 注册表：名字 ↔ 枚举靠 offsetof 一一对应**（`sql/mysqld.cc:3984`）：

```cpp
SHOW_VAR com_status_vars[] = {
    {"admin_commands", (char *)offsetof(System_status_var, com_other),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"assign_to_keycache",
     (char *)offsetof(System_status_var,
                      com_stat[(uint)SQLCOM_ASSIGN_TO_KEYCACHE]),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"alter_db",
     (char *)offsetof(System_status_var, com_stat[(uint)SQLCOM_ALTER_DB]),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"commit",
     (char *)offsetof(System_status_var, com_stat[(uint)SQLCOM_COMMIT]),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    ...
};
```
```cpp
/* sql/sql_parse.cc:3205 —— 统一主写入点 */
  thd->status_var.com_stat[lex->sql_command]++;
```
> ★ 外层 `status_vars[]` 用 `{"Com", (char*)com_status_vars, SHOW_ARRAY, SHOW_SCOPE_ALL}` 挂进来，所以显示名是 `"Com" + "_" + "alter_db"` = `Com_alter_db`——**子数组里的名字必须小写**。

**★ 7 个"走不到主入口"的特例写入点**（★ 在到达 3205 前就 return / 走快捷分支）：

```cpp
      thd->status_var.com_stat[SQLCOM_CHANGE_DB]++;        // sql_parse.cc:1818
      thd->status_var.com_stat[SQLCOM_SHOW_FIELDS]++;      // :2176
      thd->status_var.com_stat[SQLCOM_FLUSH]++;            // :2295
      thd->status_var.com_stat[SQLCOM_SHOW_STATUS]++;      // :2340
      thd->status_var.com_stat[SQLCOM_SHOW_PROCESSLIST]++; // :2378
      thd->status_var.com_stat[SQLCOM_KILL]++;             // :2401
      thd->status_var.com_stat[SQLCOM_SET_OPTION]++;       // :2408
```
> ★ 复制线程侧的 REPLICA 命令走 `com_other` 而非 `com_stat`（`sql/rpl_source.cc:942` / `:1004`：`thd->status_var.com_other++;`）——所以**从库上 `Com_slave_start` 永远是 0**。

**★★ 一个很多人踩过的坑：直查 PFS 表看不到任何 `Com_xxx`**（`storage/perfschema/pfs_variable.cc:939`）：

```cpp
bool PFS_status_variable_cache::filter_by_name(const SHOW_VAR *show_var) {
  if (show_var->type == SHOW_ARRAY) {
    /* The SHOW_ARRAY name is the prefix for the variables in the sub array. */
    const char *prefix = show_var->name;
    /* Exclude COM counters if not a SHOW STATUS command. */
    if (!my_strcasecmp(system_charset_info, prefix, "Com") && !m_show_command) {
      return true;
    }
  }
  return false;
}
```
> ★★ `SELECT * FROM performance_schema.global_status` **看不到 Com_xxx 行**——只有真的发了 `SHOW STATUS` 命令（`m_show_command=true`）才展开 `Com` 子数组。这是纯兼容性 hack。

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

**★ 通道二的数组与"根"**（`ha_innodb.cc:1126` / `:21956`）：

```cpp
static SHOW_VAR innodb_status_variables[] = {
    {"buffer_pool_dump_status",
     (char *)&export_vars.innodb_buffer_pool_dump_status, SHOW_CHAR,
     SHOW_SCOPE_GLOBAL},
    {"buffer_pool_pages_data",
     (char *)&export_vars.innodb_buffer_pool_pages_data, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    {"buffer_pool_read_requests",
     (char *)&export_vars.innodb_buffer_pool_read_requests, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    ...
};
```
```cpp
/* :21956 —— ★ 对外的"根"只有一条 */
static SHOW_VAR innodb_status_variables_export[] = {
    {"Innodb", (char *)&show_innodb_vars, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    {NullS, NullS, SHOW_LONG, SHOW_SCOPE_GLOBAL}};   /* {NullS,...} 是哨兵 */
```
```cpp
/* :21526 —— ★ 为什么要包一层 SHOW_FUNC */
static int show_innodb_vars(THD *, SHOW_VAR *var, char *) {
  innodb_export_status();                    /* ★ 读之前先刷新计数器快照 */
  var->type = SHOW_ARRAY;
  var->value = (char *)&innodb_status_variables;
  var->scope = SHOW_SCOPE_GLOBAL;
  return (0);
}
```
> ★★ `SHOW_FUNC` 在这里的用途是**延迟计算 + 副作用刷新**：不包这一层，`SHOW STATUS` 读到的永远是 `export_vars` 里的旧值。

**★ 通道一的挂载点**（`ha_innodb.cc:5168`）：

```cpp
  innobase_hton->show_status = innobase_show_status;
```
```cpp
/* sql/handler.h:932 */
enum ha_stat_type { HA_ENGINE_STATUS, HA_ENGINE_LOGS, HA_ENGINE_MUTEX };
```
```cpp
/* sql/handler.cc:7677 —— ★ 走 (Type, Name, Status) 三列自由文本协议 */
bool ha_show_status(THD *thd, handlerton *db_type, enum ha_stat_type stat) {
  mem_root_deque<Item *> field_list(thd->mem_root);
  field_list.push_back(new Item_empty_string("Type", 10));
  field_list.push_back(new Item_empty_string("Name", FN_REFLEN));
  field_list.push_back(new Item_empty_string("Status", 10));
```
> ★★ **这就是"双通道"**：通道一 = 非结构化大文本（`SHOW ENGINE ... STATUS`，完全不经过 `SHOW_VAR`/`all_status_vars`）；通道二 = 结构化数值（`SHOW STATUS` / PFS 表）。**两者互不可见**——所以 `SHOW STATUS` 里永远看不到 `SHOW ENGINE INNODB STATUS` 的事务列表。
> ⚠️ 真名核查：`show_innodb_vars` 是**回调函数**不是数组；数组真名 `innodb_status_variables[]`（`:1126`）与 `innodb_status_variables_export[]`（`:21956`）。

#### `FLUSH STATUS` 的精确语义

`refresh_status()`（**不存在** `mysql_reset_status`）干五件事：① 每个活跃 THD 先 `add_to_status(&global_status_var, ...)` 再清零（session 归零、增量保留）；② `reset_pfs_status_stats()`（account 先汇入 user/host 再清零）；③ `reset_status_vars()` 清 `SHOW_LONG`/`SHOW_SIGNED_LONG`（指针型全局变量），**`SHOW_LONG_NOFLUSH`（PFS `*_lost`）故意不清**；④ 清 key cache；⑤ `Max_used_connections` 重置为**当前连接数**（不是 0）。

关键事实：`refresh_status()` **并不 `memset(&global_status_var, 0)`**——全仓库对它的写只有启动初始化与 `add_to_status` 累加。手册只写一句 "Many status variables are reset to 0" 且**无完整清单**，这本身是坑。`TRUNCATE TABLE performance_schema.global_status` 与 `FLUSH STATUS` 等价。权限 `RELOAD` 或 `FLUSH_STATUS`。

**★ 完整实现**（`sql/mysqld.cc:11753`，⚠️ 真名不是 `reset_status_var()`）：

```cpp
/**
  Reset status for all threads.
*/
class Reset_thd_status : public Do_THD_Impl {
 public:
  void operator()(THD *thd) override {
    /* Update the global status if not done so already. */
    if (!thd->status_var_aggregated) {
      add_to_status(&global_status_var, &thd->status_var);
    }
    reset_system_status_vars(&thd->status_var);
  }
};

void refresh_status() {
  mysql_mutex_lock(&LOCK_status);
  /* For all threads, add status to global status and then reset. */
  Reset_thd_status reset_thd_status;
  Global_THD_manager::get_instance()->do_for_all_thd_copy(&reset_thd_status);
#ifdef WITH_PERFSCHEMA_STORAGE_ENGINE
  /* Reset aggregated status counters. */
  reset_pfs_status_stats();          /* ★★ 平行账本必须单独清，否则 PFS 表不归零 */
#endif
  /* Reset some global variables. */
  reset_status_vars();
  /* Reset the counters of all key caches (default and named). */
  process_key_caches(reset_key_cache_counters);
  flush_status_time = time((time_t *)nullptr);
  mysql_mutex_unlock(&LOCK_status);

  /*
    Set max_used_connections to the number of currently open
    connections.  Do this out of LOCK_status to avoid deadlocks.
    Status reset becomes not atomic, but status data is not exact anyway.
  */
  Connection_handler_manager::reset_max_used_connections();
}
```
> ★★ **FLUSH STATUS 是"全局 + 所有会话"一起重置，不只是 SESSION**。顺序：① 每个活跃 THD 先并入 global 防丢增量 → ② 清零自己的 `status_var` → ③ 清 PFS 聚合账本 → ④ 清全局裸标量 → ⑤ 清 key cache。

```cpp
/* sql/sql_show.cc:3328 —— 只清全局裸标量 */
void reset_status_vars() {
  for (Status_var_array::iterator ptr = all_status_vars.begin();
       ptr < all_status_vars.end(); ptr++) {
    /* Note that SHOW_LONG_NOFLUSH variables are not reset */
    if (ptr->type == SHOW_LONG || ptr->type == SHOW_SIGNED_LONG)
      *(ulong *)ptr->value = 0;
  }
}
```
> ★ `SHOW_LONG_STATUS` / `SHOW_LONGLONG_STATUS` 是 offsetof，**无法在这里清零**——它们靠"每个 THD 清零 + 退出时并入"达到归零效果。

```cpp
/* sql/sql_reload.cc:93 —— 权限 */
  if ((op_type & REFRESH_STATUS) &&
      !sctx->has_global_grant(STRING_WITH_LEN("FLUSH_STATUS")).first) {
    my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "RELOAD or FLUSH_STATUS");
    return true;
  }
```
```cpp
/* sql/sql_reload.cc:343 —— ★ thd 非空才做：FLUSH STATUS 总是会话上下文内 */
  if (thd && (options & REFRESH_STATUS)) refresh_status();
```

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

#### 端到端主链路：`SHOW VARIABLES` 的完整旅程

把上面的重写 + pfs.md 1.9 的 cache 实现串起来，一次 `SHOW VARIABLES` 从敲命令到出结果的完整调用栈是（**本篇唯一完整版**，前端按 yacc/make_cmd/build_query 三段源码展开）：

```
客户端发 "SHOW VARIABLES"

【1】语法解析（sql_yacc.yy: show_variables_stmt）
    SHOW opt_var_type VARIABLES opt_wild_or_where
      → NEW_PTN PT_show_variables(pos, var_type, wild, where)
    opt_var_type: 空 → OPT_SESSION            ★ 裸 SHOW 默认是 SESSION 视图！
                  GLOBAL → OPT_GLOBAL
                  SESSION/LOCAL → OPT_SESSION

【2】make_cmd（parse_tree_nodes.cc）—— 分派器，只决定"用哪张表"
    lex->sql_command = SQLCOM_SHOW_VARIABLES           ① 先标命令类型
    if (m_wild.str) lex->set_wild(m_wild)              ② LIKE 'x%' 暂存进 lex->wild
    if (m_var_type == OPT_SESSION)
        build_show_session_variables(pos, thd, lex->wild, m_where)   → 表名 "session_variables"
    else if (m_var_type == OPT_GLOBAL)
        build_show_global_variables(pos, thd, lex->wild, m_where)    → 表名 "global_variables"
    return &m_sql_cmd                                  ③ Sql_cmd_show_variables

【3】build_query（sql_show_status.cc）—— 手搓器，造出"查 PFS 表"的查询树
    常量：col_name="VARIABLE_NAME"  as_name="Variable_name"
          col_value="VARIABLE_VALUE" as_value="Value"
          pfs="performance_schema"
    内层：PTI_expr_with_alias(ident(VARIABLE_NAME), Variable_name)
        + PTI_expr_with_alias(ident(VARIABLE_VALUE), Value)
        + PT_table_factor_table_ident(performance_schema, <table_name>)
        → PT_query_specification(内层, where=nullptr)
    外层：PT_subquery → PT_derived_table(lateral=false, sub_query,
              table_alias = <table_name>)            ★ 派生表别名就是表名
        + Item_asterisk（SELECT *）
        + where_clause = wild ? PTI_where(Item_func_like(Variable_name, wild_string))
                             : where_cond            ★ LIKE 在这里变成 WHERE
    关键三步：
        lex->sql_command = SQLCOM_SELECT              ← 伪装成 SELECT
        query_expression2->contextualize(&pc)         ← PT 树 → Query_block/Item
        pc.finalize_query_expression()
        lex->sql_command = command                    ← 恢复 SQLCOM_SHOW_VARIABLES

【4】执行
  → mysql_execute_command case SQLCOM_SHOW_VARIABLES
      → Sql_cmd_show_variables::execute()（空子类，无快照回滚——那是 STATUS 特有）
          → Sql_cmd_show::check_privileges()  ← check_table_access(SELECT_ACL, pfs 表)
          → Sql_cmd_select::execute()                ★ 它真的是条 SELECT
              → 优化器/执行器 → ha_perfschema::rnd_init
                  → table_global_variables::rnd_init()            storage/perfschema/table_global_variables.cc
                      → m_sysvar_cache.materialize_global()
                          → PFS_system_variable_cache::do_materialize_global()
                              ├─ mysql_mutex_lock(&LOCK_plugin_delete)
                              ├─ init_show_var_array(OPT_GLOBAL, /*strict=*/true)
                              │    └─ System_variable_tracker::enumerate_sys_vars(sort=true, ...)
                              │        ├─ 遍历 static_system_variable_hash
                              │        ├─ 遍历 dynamic_system_variable_hash   （strict 过滤 + 敏感变量过滤）
                              │        └─ std::sort 按名字排序
                              └─ 对每个 System_variable_tracker:
                                  access_system_variable() 找到 sys_var
                                  → 构造 System_variable(target_thd, &show_var, OPT_GLOBAL)
                                      → System_variable::init()
                                          ├─ [跨线程才拿] target_thd->LOCK_thd_sysvar
                                          ├─ mysql_mutex_lock(&LOCK_global_system_variables)
                                          └─ get_one_variable_ext(current, target, show_var,
                                                                  OPT_GLOBAL, SHOW_SYS, ...)
                                              ├─ ★ SHOW_SYS 特判：
                                              │    show_type = sys_var->show_type()   // 换成真实类型
                                              │    value = sys_var->value_ptr(running, target, OPT_GLOBAL)
                                              │        └─ global_value_ptr() 读 global_system_variables
                                              └─ switch(真实 show_type)：数值/字符串 → 文本
              → table_global_variables::rnd_next()
                  → make_row() → read_row_values()   → 逐行发给客户端
```

**最容易误解的一处：`get_one_variable_ext` 里的 `case SHOW_SYS: /* Cannot happen */`**。这个 case 确实是 `assert(0)` 死分支——但**不是**因为 SHOW_SYS 不被处理，而是因为函数**开头就把 SHOW_SYS 换掉了**：

```cpp
const char *get_one_variable_ext(THD *running_thd, THD *target_thd, ...) {
  if (show_type == SHOW_SYS) {                     // 开头特判
    sys_var *var = ((sys_var *)variable->value);   // show_var->value 就是 sys_var 指针
    show_type = var->show_type();                  // SHOW_SYS → 实际类型（SHOW_LONG/CHAR...）
    value = pointer_cast<const char *>(
        var->value_ptr(running_thd, target_thd, value_type, {}));  // ★ 真正求值点
    value_charset = var->charset(target_thd);
  } else { ... }
  switch (show_type) { ... case SHOW_SYS: /* Cannot happen */ ... }  // 兜底，永远进不来
}
```

所以系统变量的求值**唯一入口是 `sys_var::value_ptr(running_thd, target_thd, value_type)`**——它按 `value_type`（OPT_GLOBAL/OPT_SESSION）决定走 `global_value_ptr()`（读全局实例）还是 `session_value_ptr()`（读 target THD），拿到**原始值指针**后，再由 `switch(show_type)` 按类型格式化（`SHOW_LONG` 用 `longlong10_to_str`、`SHOW_CHAR` 直接取串、`SHOW_BOOL` 转 `ON/OFF`…）。这就是"描述符与值分离"的最后一步：`sys_var` 提供值指针 + 类型，格式化器统一转文本。

**`SELECT * FROM performance_schema.global_variables`（直接查表）的差异**：它**不经过** `build_query` 重写（那是 SHOW 语法专用），作为普通 SELECT 进优化器 → `ha_perfschema` → 同一个 `rnd_init → materialize_global`。`rnd_init` 之后两条路径完全一致。唯一实质差异在**头部**：

- SHOW 语法：`build_query` 包了一层子查询 + 列别名 `Variable_name/Variable_value` + 末尾恢复 `SQLCOM_SHOW_VARIABLES`
- 直接 SELECT：无重写、无列别名（列名就是 `VARIABLE_NAME/VARIABLE_VALUE`）、`sql_command` 保持 `SQLCOM_SELECT`

另外两个值得注意的旁路：`table_global_variables::get_row_count()`——经 handler 标准接口 `ha_perfschema::info(HA_STATUS_VARIABLE)` 的 `stats.records = m_table_share->get_row_count()` 被 SQL 层用作该表的行数估计（EXPLAIN/join 优化用），它同样持 `LOCK_plugin_delete + LOCK_system_variables_hash(rd)` 数 static+dynamic 双 hash 的变量总数（`get_system_variable_count`）；`make_row()` 里 `mysql_audit_notify(MYSQL_AUDIT_GLOBAL_VARIABLE_GET, ...)` —— **读全局变量会触发审计事件**（审计插件能感知"谁读了哪个全局变量"）。

#### SHOW VARIABLES 与直查 PFS 表：是不是一个东西？

**精确回答：数据源上是同一个东西，语法/命令上不是。** 差异全部集中在"进 `ha_perfschema::rnd_init` 之前"的头部；`rnd_init` 之后（物化管线、求值、行输出）逐字节一致。

**差异一：`build_query` 是"手搓语法树"，不是文本拼接**。`PT_show_variables::make_cmd` 用一堆 `PTI_*` 节点直接构造查询树（MAINTAINER 注释明说按 `turn_parser_debug_on()` 反推文法动作）：

```cpp
// 内层：SELECT VARIABLE_NAME as Variable_name, VARIABLE_VALUE as Value
//       FROM performance_schema.<table>
PTI_expr_with_alias *expr_name = new ... PTI_expr_with_alias(pos, ident_name, pos.cpp, as_name);
// 外层：SELECT * FROM (内层) <table_name> [WHERE ...]
PT_derived_table *derived_table = new ... PT_derived_table(false, sub_query, table_name, ...);
// LIKE 'x%' 被翻译成 Item_func_like 挂在外层 WHERE 上
Item_func_like *func_like = new ... Item_func_like(pos, ident_name_where, wild_string);
...
// 关键：contextualize 期间命令类型是 SELECT，结束后恢复
lex->sql_command = SQLCOM_SELECT;
if (query_expression2->contextualize(&pc)) return nullptr;
if (pc.finalize_query_expression()) return nullptr;
lex->sql_command = command;      // 恢复成 SQLCOM_SHOW_VARIABLES
```

**差异二：`Sql_cmd_show_variables` 没有重写 `execute`**——它是 `Sql_cmd_show` 的**空子类**（构造传 `SQLCOM_SHOW_VARIABLES` 而已）。对比 `Sql_cmd_show_status::execute` 那套"语句前快照 + 增量转移回滚"是 STATUS 特有的观测者效应控制，**VARIABLES 侧没有**（读变量不会污染计数器，无需快照）。

**差异三：默认作用域是 SESSION**——最容易踩的坑。yacc 的 `opt_var_type: %empty { $$=OPT_SESSION; }`，所以**裸 `SHOW VARIABLES` 等价于 `SHOW SESSION VARIABLES`**（查 `session_variables` 表），不是 GLOBAL！要全局值必须显式 `SHOW GLOBAL VARIABLES`。

**全维度对比表**：

| 维度 | `SHOW [GLOBAL\|SESSION] VARIABLES` | `SELECT * FROM pfs.global_variables / session_variables` |
|---|---|---|
| 语法树来源 | `PT_show_variables` 手搓的查询树（`build_query`） | 用户 SQL 正常解析 |
| 默认作用域 | **SESSION**（裸 SHOW = session_variables） | 显式指定哪张表 |
| 结果列名 | `Variable_name` / `Value`（内层投影的别名） | `VARIABLE_NAME` / `VARIABLE_VALUE` |
| `sql_command` | `SQLCOM_SHOW_VARIABLES`（contextualize 后恢复） | `SQLCOM_SELECT` |
| `Com_*` 计数 | `Com_show_variables` +1 | `Com_select` +1 |
| LIKE 语法糖 | `LIKE 'x%'` → 外层 `WHERE Variable_name LIKE 'x%'` | 无，自己写 WHERE |
| 命令执行类 | `Sql_cmd_show_variables`（空子类，无快照回滚） | `Sql_cmd_select` |
| 权限 | `check_table_access(SELECT_ACL)`（Sql_cmd_show 的 check_privileges） | 普通 SELECT 授权 |
| `rnd_init` 之后 | **完全一致**（`materialize_global()`/`materialize_all(current_thd)`） | **完全一致** |
| 读全局变量的审计事件 | 有（make_row 里 `MYSQL_AUDIT_GLOBAL_VARIABLE_GET`） | 有（同一条 make_row） |

**驱动入口也对应成对**：`table_global_variables::rnd_init` → `m_sysvar_cache.materialize_global()`；`table_session_variables::rnd_init` → `m_sysvar_cache.materialize_all(current_thd)`（本会话专用入口，无跨线程防护需求）。

#### 流程对比图：语法分叉 → 物化/输出一致（含锁标注）

下面这张图完全按源码调用链绘制（不标文件行号），分四段：语法分叉 → 行数估计 → 物化（两层子阶段）→ 迭代销毁。

**【A 段】语法与命令：两者分叉（还没打开 PFS 表）**

```
                    SHOW VARIABLES              │  SELECT * FROM pfs.global_variables
  ──────────────────────────────────────────────┼──────────────────────────────────────
  ① 语法解析                                     │  ① 语法解析（普通 SELECT）
     yyparse → PT_show_variables                 │     → Query_block（用户的查询树）
     （裸 SHOW 默认 OPT_SESSION！）                │
                                                 │
  ② make_cmd 阶段（直查没有这步）                  │  ② ——（无此阶段）
     PT_show_variables::make_cmd                  │
     ├─ OPT_SESSION → build_show_session_         │
     │                 variables("session_        │
     │                 variables")                │
     ├─ OPT_GLOBAL  → build_show_global_          │
     │                 variables("global_         │
     │                 variables")                │
     └─ build_query() 手搓查询树（PTI_* 节点）：    │
         内层: SELECT VARIABLE_NAME               │
                      AS Variable_name,           │
                      VARIABLE_VALUE AS Value     │
               FROM pfs.<table>                   │
         外层: SELECT * FROM (内层)                │
               AS <table_name>                    │
               ★ 别名=表名本身（PT_derived_table    │
                 第 3 参就是 table_alias），        │
                 不是 "t"                         │
               [LIKE 'x%' → 外层 WHERE 的          │
                Item_func_like]                   │
     contextualize 期间伪装成 SELECT，              │
     结束后恢复 SQLCOM_SHOW_VARIABLES             │  sql_command = SQLCOM_SELECT
                                                 │
  ③ 命令对象：execute() 入口                       │  ③ 命令对象：execute() 入口
     Sql_cmd_show_variables（空子类，              │     Sql_cmd_select
     无快照回滚——那是 STATUS 特有）                │
     → Sql_cmd_show::execute()                    │     → execute()
       → Sql_cmd_select::execute()                │
     此时 lex 里已是②替换好的 SELECT 查询树        │
     ★ 下面的④优化器、⑤执行器都发生在这个          │
       execute() 内部（不是 execute 之后的步骤）    │
                                                 │
  ④ 优化器（SHOW 唯一的额外层）                    │  ④ 优化器
     外层 SELECT * FROM (派生表)                   │     TABLE_SCAN（无谓词）
       → 物化到内部临时表                          │     [点查: REF / INDEX_RANGE_SCAN]
         （Created_tmp_tables +1 的根源）          │
     ★ LIKE 不下推：SHOW ... LIKE 的过滤在外层派生表 │
       上，内层 PFS 表仍是全表扫描（无谓词）        │
       而直查的 WHERE 在内层，可能走索引            │
     内层: TABLE_SCAN                             │
                                                 │
  ★ 两条路径接下来都会打开同一个 PFS 表，
    打开时的行数估计见【B 段】（含 LOCK_plugin_delete 卡点）
  ──────────────────────────────────────────────┴──────────────────────────────────────
```

> **"公有"确认**：`info(HA_STATUS_VARIABLE) → get_row_count()` 这条**两条路径都有**——因为 SHOW 被重写后打开的也是同一张 PFS 表，打开表时 SQL 层都要统计行数。它是"打开表时"的动作，不是分叉点，所以只在【B 段】详写一次（避免与 A 段重复）。

**【B 段】打开 PFS 表：行数估计（两条路径共有）**

```
  handler::info(HA_STATUS_VARIABLE)              ← SQL 层要表统计信息时调用
     → stats.records = m_table_share->get_row_count()
         └─ [锁] LOCK_plugin_delete + LOCK_system_variables_hash(rd)
            数值 = static hash size + dynamic hash size（get_system_variable_count）
            → 供优化器估算 rows（EXPLAIN 的 rows 列来源）
```

**【C 段】物化：rnd_init（或 index_init）里发生，两条路径一致**

```
  ⑤ TableScanIterator::Init → handler::ha_rnd_init(true)
     [点查: RefIterator::Init → ha_index_init → ha_perfschema::index_init]

  ⑥ ha_perfschema::rnd_init / index_init
     → m_open_table()  new 表游标对象（含全新空 cache）
     → materialize_global()   ★ 点查也全量物化，不省

        ├─ [锁] LOCK_plugin_delete ── 物化【全程】持有
        │        保护：防插件装卸把 dynamic hash 改掉
        │
        ├─ C1 清单层（INITIALIZE）：init_show_var_array
        │    [锁] LOCK_system_variables_hash(rd) ── 仅枚举期间，枚举完即放
        │    enumerate_sys_vars 遍历 static + dynamic 双 hash：
        │       · 敏感变量过滤（无 SENSITIVE_VARIABLES_OBSERVER 则跳过）
        │       · scope 过滤（global 表 strict=true；session 表 strict=false）
        │       · std::sort（my_strcasecmp，大小写不敏感）
        │    → m_sys_var_tracker_array（清单层，m_initialized = true）
        │
        └─ C2 值层（MATERIALIZE）：对清单里每个 tracker 求值（循环 N 次）
             ├─ access_system_variable() 找到 sys_var（插件变量按 Lifetime 加相应锁）
             └─ System_variable::init()
                  ├─ [锁] target->LOCK_thd_sysvar（跨线程读会话变量时才拿，先）
                  ├─ [锁] LOCK_global_system_variables（后；求值完即放，非全程）
                  ├─ get_one_variable_ext(SHOW_SYS 特判)：
                  │    show_var.value 就是 sys_var* →
                  │    sys_var::value_ptr(running, target, OPT_GLOBAL)
                  │      → global_value_ptr() = &global_system_variables + offset
                  └─ switch(真实 show_type) 格式化 → 写进 m_value_str
             → push_back 进 m_cache（值层，m_materialized = true）

        ↑ 物化结束：所有锁释放。m_cache = 锁内定格的【文本快照】
```

**【D 段】迭代与销毁：两条路径一致**

```
  ⑦ 逐行 Read → ha_rnd_next → m_table->rnd_next ──【无锁】
       [点查: index_next 线性扫 m_cache + m_index->match(name) 过滤]
     → m_cache[i] 取行
     → make_row()：填 row 结构
           + mysql_audit_notify(MYSQL_AUDIT_GLOBAL_VARIABLE_GET)（审计事件）
           + 若 is_null()（未初始化）→ HA_ERR_RECORD_DELETED，跳过该行
     → read_row_values()：把 row 翻译成 Field 值 → 发给客户端

  ⑧ 结束 → ha_rnd_end（或 index_end）──【无锁】
     → delete m_table   ★ cache 随游标销毁（游标级生命周期）
```

**读图要点**：

- **差异只在 A 段**（语法树来源、make_cmd 手搓、命令类/sql_command、SHOW 多一层派生表物化）。从 B 段开始两条路径**合并为同一条**——因为 SHOW 在语法阶段就被重写成了对 PFS 表的 SELECT，B/C/D 段（打开表 → 物化 → 迭代输出）**逐字节一致**。严谨地说：SHOW 相比直查的**全部额外开销**就是 A 段④那层"外层派生表物化到临时表"（`Created_tmp_tables` +1 的根源）；PFS 表本身的访问完全等价。
- **所有锁都集中在 C 段物化阶段**，且各自只持有最短时间（`LOCK_plugin_delete` 全程、`LOCK_system_variables_hash` 只在枚举期间、`LOCK_global_system_variables` 只在单个变量求值期间）；**D 段迭代与销毁全程无锁**——这就是"快照 + 免锁迭代"的代码形态：迭代期间别的线程 `SET` 变量也影响不到本次扫描。
- **点查不省**：`index_init` 与 `rnd_init` 做同样全量物化，点查只在 `index_next` 里线性 `match` 过滤（PFS 的"索引"没有 hash 结构）。
- 裸 `SHOW VARIABLES` 走 `session_variables` 表（`build_show_session_variables` + `materialize_all(current_thd)`），其余同理。

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

**★ 精确构成：5 变量 + 6 状态 + 1 用户变量 = 12 张**（`storage/perfschema/` 全量 `table_*.cc` 实测）：

| 类别 | 源文件 | PFS 表名 |
|---|---|---|
| 系统变量（5） | `table_global_variables.cc` / `table_session_variables.cc` / `table_variables_by_thread.cc` / `table_variables_info.cc` / `table_persisted_variables.cc` | `global_variables` / `session_variables` / `variables_by_thread` / `variables_info` / `persisted_variables` |
| 状态变量（6） | `table_global_status.cc` / `table_session_status.cc` / `table_status_by_thread.cc` / `table_status_by_user.cc` / `table_status_by_host.cc` / `table_status_by_account.cc` | `global_status` / `session_status` / `status_by_thread` / `status_by_user` / `status_by_host` / `status_by_account` |
| 用户变量（1） | `table_user_variables_by_thread.cc` | `user_variables_by_thread` |

> ⚠️ **一个常见误数**：如果只 grep `variables`/`status` 两类会得出 **11 张**，漏掉 `user_variables_by_thread`——后者是唯一一张"用户变量"表，文件前缀是 `table_user_variables` 而非 `table_variables`。
> ★ 另一个不对称：**变量表只有 `by_thread` 一个维度，状态表才有 user/host/account 三维**——因为"别人的会话变量值"属于敏感信息，而状态计数器可以安全聚合。

**★ `variables_info` 的 9 种来源**（`storage/perfschema/table_variables_info.cc:49`，★ 这就是"变量来源优先级"的完整清单）：

```cpp
Plugin_table table_variables_info::m_table_def(
    /* Schema name */ "performance_schema",
    /* Name */        "variables_info",
    /* Definition */
    "  VARIABLE_NAME varchar(64) not null,\n"
    "  VARIABLE_SOURCE ENUM('COMPILED','GLOBAL','SERVER','EXPLICIT','EXTRA',\n"
    "                       'USER','LOGIN','COMMAND_LINE','PERSISTED',\n"
    "                       'DYNAMIC') DEFAULT 'COMPILED',\n"
    "  VARIABLE_PATH varchar(1024),\n"
    "  MIN_VALUE varchar(64),\n"
    "  MAX_VALUE varchar(64),\n"
    "  SET_TIME TIMESTAMP(6) default null,\n"
    "  SET_USER CHAR(32) collate utf8mb4_bin default null,\n"
    "  SET_HOST CHAR(255) CHARACTER SET ASCII default null\n",
    /* Options */ " ENGINE=PERFORMANCE_SCHEMA", nullptr);
```
> ★ `PERSISTED` 对应 `mysqld-auto.cnf`，`DYNAMIC` 对应运行期 `SET GLOBAL`。

**★ `persisted_variables` 的行数来自内存单例**（`table_persisted_variables.cc`）：

```cpp
PFS_engine_table_share table_persisted_variables::m_share = {
    &pfs_readonly_world_acl,   /* ★ 谁都能读，没人能写 */
    ...
    nullptr, /* write_row */
    nullptr, /* delete_all_rows */
    table_persisted_variables::get_row_count, ...
};

ha_rows table_persisted_variables::get_row_count() {
  Persisted_variables_cache *pv = Persisted_variables_cache::get_instance();
  if (pv) return pv->get_persisted_dynamic_variables()->size();
  return 0;
}
```
> ★ 不落表不读盘，直接取 `Persisted_variables_cache` 单例的容器大小。

#### 3. 统一的 PFS 处理管线

PFS 侧两个平行的 cache 类把三类变量统一成 `SHOW_VAR` 中间格式，走"INITIALIZE（构建排序清单）→ MATERIALIZE（求值转文本）→ OUTPUT"同一条管线——系统变量从双 hash 枚举（`enumerate_sys_vars`），状态变量遍历 `all_status_vars` 展开 `SHOW_ARRAY`。**完整实现（`do_materialize_*` 家族、线程间读三重防护、mem_root 管理）见 [`pfs.md` 的 1.9 节](../../infra/observability/pfs/pfs.md#19-pfs-的变量与状态表实现show-variablesstatus-的幕后)**。

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

**★ grep 实证**（★ 不是"视图转发到 PFS"，而是 **I_S 侧彻底删除 + SHOW 在解析期改写**）：

```
# ① sql/dd/impl/system_views/ 下 grep "variables" → 0 命中
#    （8.0 的 I_S 表全是 DD 系统视图，目录里已无任何变量/状态表）

# ② 全库 grep 那四个名字，只出现在 PFS 表定义与 SHOW 重写目标里：
sql/sql_show_status.cc:274:  table_name = "session_status"
sql/sql_show_status.cc:282:  table_name = "global_status"
sql/sql_show_status.cc:291:  table_name = "session_variables"
sql/sql_show_status.cc:299:  table_name = "global_variables"
storage/perfschema/table_global_status.cc:63:     "global_status"
storage/perfschema/table_session_status.cc:63:    "session_status"
storage/perfschema/table_global_variables.cc:64:  "global_variables"
storage/perfschema/table_session_variables.cc:63: "session_variables"
```
> ★★ **没有任何 I_S 视图转发实现** ⇒ 结论是"删除 + 改写"，不是"转发"。

**★ 四个入口：表名写死**（`sql/sql_show_status.cc:272`）：

```cpp
Query_block *build_show_session_status(const POS &pos, THD *thd,
                                       const String *wild, Item *where_cond) {
  static const LEX_CSTRING table_name = {STRING_WITH_LEN("session_status")};
  return build_query(pos, thd, SQLCOM_SHOW_STATUS, table_name, wild, where_cond);
}
Query_block *build_show_global_status(...)   { ... "global_status" ... }
Query_block *build_show_session_variables(...) { ... "session_variables" ... }
Query_block *build_show_global_variables(...)  { ... "global_variables" ... }
```

**★★ "SHOW STATUS 是 SELECT 的皮"的确凿证据**（`sql/sql_show_status.cc:99`）：

```cpp
  static const LEX_CSTRING col_name = {STRING_WITH_LEN("VARIABLE_NAME")};
  static const LEX_CSTRING as_name = {STRING_WITH_LEN("Variable_name")};
  static const LEX_CSTRING col_value = {STRING_WITH_LEN("VARIABLE_VALUE")};
  static const LEX_CSTRING as_value = {STRING_WITH_LEN("Value")};
  static const LEX_CSTRING pfs = {STRING_WITH_LEN("performance_schema")};
  ...
  lex->sql_command = SQLCOM_SELECT;          /* ★ 先伪装成 SELECT */
  if (query_expression2->contextualize(&pc)) return nullptr;
  if (pc.finalize_query_expression()) return nullptr;
  /* contextualize sets to COM_SELECT */
  lex->sql_command = command;                /* ★ 再把命令字改回 SHOW_STATUS */
```
> ★★ 执行计划**完全是 SELECT**，只是命令字留了个壳——所以 `Com_show_status` 计数器照常 +1，权限检查也走 SHOW 路径。

**★ `SHOW STATUS` 默认作用域是 SESSION**（`sql/sql_yacc.yy:15934`，⚠️ 常被说成 GLOBAL）：

```cpp
opt_var_type:
          %empty      { $$=OPT_SESSION; }    /* ★★ 裸写 SHOW STATUS → SESSION */
        | GLOBAL_SYM  { $$=OPT_GLOBAL; }
        | LOCAL_SYM   { $$=OPT_SESSION; }
        | SESSION_SYM { $$=OPT_SESSION; }
        ;
```
```cpp
/* sql/parse_tree_nodes.cc:2813 —— GLOBAL 与 SESSION 的实现差异只有一处：换 PFS 表名 */
  if (m_var_type == OPT_SESSION) {
    if (build_show_session_status(m_pos, thd, lex->wild, m_where) == nullptr)
      return nullptr;
  } else if (m_var_type == OPT_GLOBAL) {
    if (build_show_global_status(m_pos, thd, lex->wild, m_where) == nullptr)
      return nullptr;
  }
```

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

**★ 另一组"元变量"：`version` / `hostname` / `port` 等**（`sql/sys_vars.cc`，★ 逐个核实）：

| 变量 | 行号 | 类 | 只读 | 命令行 |
|---|---|---|---|---|
| **`version`** | 5307 | **`Sys_var_version`**（专属类） | READ_ONLY | NO_CMD_LINE |
| `version_comment` | 5313 | `Sys_var_charptr` | READ_ONLY | NO_CMD_LINE |
| `version_compile_machine` | 5319 | `Sys_var_charptr` | READ_ONLY | NO_CMD_LINE |
| `version_compile_os` | 5325 | `Sys_var_charptr` | READ_ONLY | NO_CMD_LINE |
| `version_compile_zlib` | 5331 | `Sys_var_charptr` | READ_ONLY | NO_CMD_LINE |
| `hostname` | 5813 | `Sys_var_charptr`（`IN_FS_CHARSET`） | READ_ONLY | NO_CMD_LINE |
| `port` | 3605 | `Sys_var_uint` | READ_ONLY | `CMD_LINE(REQUIRED_ARG,'P')` |
| `socket` | 3995 | `Sys_var_charptr`（`IN_FS_CHARSET`） | READ_ONLY | `CMD_LINE(REQUIRED_ARG)` |
| `character_set_system` | 1889 | `Sys_var_struct<CHARSET_INFO,Get_csname>` | READ_ONLY | NO_CMD_LINE |
| `system_time_zone` | 5078 | `Sys_var_system_time_zone`（**computed**） | — | — |
| `time_zone` | 6529 | `Sys_var_tz` | SESSION | NO_CMD_LINE |

```cpp
/* sql/sys_vars.cc:5306 —— ⚠️ @@version 是专属类，不是 Sys_var_charptr */
static Sys_var_version Sys_version(
    "version", "Server version",
    READ_ONLY NON_PERSIST GLOBAL_VAR(server_version_ptr), NO_CMD_LINE,
    IN_SYSTEM_CHARSET, DEFAULT(server_version));
```
```cpp
/* sql/sys_vars.cc:3605 —— port 是 READ_ONLY，但它是命令行选项且 NON_PERSIST */
static Sys_var_uint Sys_port(
    "port",
    "Port number to use for connection or 0 to default to, my.cnf, "
    "$MYSQL_TCP_PORT, built-in default ...",
    READ_ONLY NON_PERSIST GLOBAL_VAR(mysqld_port), CMD_LINE(REQUIRED_ARG, 'P'),
    VALID_RANGE(0, 65535), DEFAULT(0), BLOCK_SIZE(1));
```
```cpp
/* sql/sys_vars.cc:1887 —— character_set_system 固定 utf8mb3 */
static CHARSET_INFO *charset_system_default = &my_charset_utf8mb3_general_ci;
static Sys_var_struct<CHARSET_INFO, Get_csname> Sys_character_set_system(
    "character_set_system",
    "The character set used by the server for storing identifiers",
    READ_ONLY NON_PERSIST GLOBAL_VAR(system_charset_info), NO_CMD_LINE,
    DEFAULT(&charset_system_default));
```
```cpp
/* sql/sys_vars.cc:5078 —— 纯计算型，没有 GLOBAL_VAR/SESSION_VAR 存储槽 */
static Sys_var_system_time_zone Sys_system_time_zone(
    "system_time_zone", "The server system time zone");
```
> ★ 一个细微差异：`time_zone` 是 `IN_BINLOG`（会写进 binlog 保证复制一致），而 `lc_messages` 是 `NOT_IN_BINLOG`——差别来自"是否影响复制语义"。
> ⚠️ `port` / `socket` 虽是命令行选项，但标了 `NON_PERSIST`，所以 `SET PERSIST port=3307` 会被拒。

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
| `SET GLOBAL x` vs `SET @@global.x` | **语义等价**（都进 `add_system_variable_assignment()`） | 但存在三处不对称，见下 |

**★ `SET GLOBAL x` 与 `SET @@global.x` 的三处不对称**（★ 等价但不完全一样）：

```cpp
/* sql/parse_tree_nodes.cc:3778 —— `SET GLOBAL x = 1` */
  return add_system_variable_assignment(pc->thd, m_opt_prefix, m_name,
                                        pc->thd->lex->option_type, m_opt_expr);
  /* ★ var_type 取自 lex->option_type（LEX 全局状态） */
```
```cpp
/* sql/parse_tree_nodes.cc:3813 —— `SET @@global.x = 1` */
  return add_system_variable_assignment(thd, m_opt_prefix, m_name,
                                        m_scope, m_opt_expr);
  /* ★ var_type 取自语法里的 opt_set_var_ident_type */
```
```cpp
/* sql_yacc.yy —— ★ 关键不对称：option_type 没有 %empty 分支 */
opt_set_var_ident_type:
          %empty          { $$=OPT_DEFAULT; }      /* ★ @@x 可省略前缀 */
        | PERSIST_SYM '.' { $$=OPT_PERSIST; }
        | GLOBAL_SYM '.'  { $$=OPT_GLOBAL; }
        | SESSION_SYM '.' { $$=OPT_SESSION; }
        ;
option_type:
          GLOBAL_SYM      { $$=OPT_GLOBAL; }       /* ⚠️ 没有 %empty → 必须显式写 */
        | PERSIST_SYM     { $$=OPT_PERSIST; }
        | LOCAL_SYM       { $$=OPT_SESSION; }
        | SESSION_SYM     { $$=OPT_SESSION; }
        ;
```
> ★★ 三处差异：① `SET GLOBAL` 前缀形式会先做 `find_sp_variable()` 检查（`:3773`），`@@` 形式不做；② `option_type` 是 **LEX 全局状态**（`PT_option_value_type::contextualize` 里 `lex->option_type = type`），**批量 SET 时会被后续子句共享**——这是历史包袱；③ `SET @@x = 1`（无前缀、有 `@@`）合法且走 `OPT_DEFAULT`（由 `sys_var` 自己的默认作用域决定），而 `SET x = 1`（无前缀无 `@@`）是**另一条规则** `option_value_no_option_type`，语义完全不同。

**★ `SHOW_ARRAY` 的名字是前缀，不是真名**（`sql/sql_show.cc:3410`）：

```cpp
    } else if (list->type == SHOW_ARRAY && list->name &&
               !strncmp(list->name, name, strlen(list->name))) {
      // the variable name matches an array prefix: dive into it and skip the
      // prefix and the underscore
      if (get_recursive_status_var_inner(thd, (SHOW_VAR *)list->value, true,
                                         name + strlen(list->name) + 1, value,
                                         var_type, length, charset))
        return true;
    }
```
> ★ 查 `Com_select` 时先匹配 `{"Com", com_status_vars, SHOW_ARRAY}`，然后 `name + strlen("Com") + 1` **跳过 `"Com_"`**，在子数组里找 `"select"`——所以子数组里的名字必须小写。

### 已知坑

- `mysqld_safe` 把 my.cnf 选项转成命令行 ⇒ `variables_info` 里显示 `COMMAND_LINE` 而非 `GLOBAL`。
- `mysqld-auto.cnf` 解析失败 ⇒ server **报错退出**，无自动备份回滚（`.backup` 只用于崩溃恢复）；删除会丢所有持久化设置。
- `mysqld-auto.cnf` 持久化值**优先于** my.cnf 与命令行（设计本意），故"重启后 read_only 与 my.cnf 不一致"是设计行为——常见运维事故源。
- `SHOW STATUS` 每次建内部临时表 ⇒ `Created_tmp_tables` 自增（监控高频采集自我污染）。
- `Created_tmp_disk_tables` 不统计 mmap 溢出的临时表（`temptable_use_mmap` 默认开）。
- `Com_stmt_*`/`Com_restart`/`Com_shutdown` 失败也计数；`Innodb_rows_updated` 官方标注 "not meant to be 100% accurate"。
- 压缩表下 `Innodb_buffer_pool_pages_data` 可能大于 `_total`（Bug #59550）；`Last_query_cost` 8.0.16 前对复杂查询为 0（Bug #92766）。
- `Innodb_row_lock_time_avg` 是启动以来累计平均，被历史长尾拉高后不回落，与 `SHOW ENGINE INNODB STATUS` 的瞬时语义不同。
- **★ 直查 PFS 表看不到任何 `Com_xxx`**（`pfs_variable.cc:939` `filter_by_name`）：`SELECT * FROM performance_schema.global_status` **不含 `Com_*` 行**——只有真的发 `SHOW STATUS` 命令时才展开 `Com` 子数组。想采集 `Com_select` 必须走 `SHOW STATUS`，或改用 `events_statements_summary_global_by_event_name`。
- **★ `FLUSH STATUS` 是全局 + 所有会话一起清**，不只是本会话（`mysqld.cc:11753`）——本以为"只清 session"是常见误解。
- **★ `SET PERSIST_ONLY` 只认动态权限，`SUPER` 不够**（`set_var.cc:204`）——必须同时有 `SYSTEM_VARIABLES_ADMIN` 和 `PERSIST_RO_VARIABLES_ADMIN`。
- **★ my.cnf 不支持行尾 `\` 续行，也不支持 `$VAR` 环境变量展开**（`my_default.cc:926` 一次 `mysql_file_getline` 只读一行）。
- **★ 行尾注释只认 `#`，且必须不在引号内**；行尾 `;` **不截断**（只有行首 `;` 才整行跳过）。
- **★ 未知转义保留反斜杠**：`\d` 会变成字面的 `\d`（`my_default.cc:1116` `default: /* Unknown; Keep '\' */`）。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `SHOW_VAR` 定义在 `sql/set_var.h` | **不在** → 真身 `include/mysql/status_var.h:80`（plugin 可见的公开头） |
| `SHOW_SCOPE_*` 靠位或拼进 `SHOW_*` | **错** → 是独立枚举 `enum_mysql_show_scope`，作为第 4 个**独立字段**，无任何位或宏 |
| `System_status_var` 在 `sql/mysqld.h` | **不在**（`mysqld.h:377` 只有 `extern`）→ 真身 `sql/system_variables.h:513` |
| `SHOW_VAR status_vars[]` 在 `sql_show.cc` | 在 **`sql/mysqld.cc:9690`** |
| `calc_sum_of_all_status()` 在 `mysqld.cc` | 在 **`sql/sql_show.cc:3736`** |
| `add_to_status` 是宏 | 是**普通函数**，`sql/system_variables.cc:28` |
| `FLUSH STATUS` 调 `reset_status_var()` | **不存在** → 真名 **`refresh_status()`**（`mysqld.cc:11753`） |
| 8.0 仍用 `HASH hash_user_variables` | **已废弃** → `collation_unordered_map<std::string, unique_ptr_with_deleter<user_var_entry>>` |
| `user_var_entry` 在 `sql_class.h` | 在 **`sql/item_func.h:2953`** |
| `user_var_entry` 有 `update_query_id` | **不存在** → 只有 `m_used_query_id`（服务 binlog 去重，**不是**求值顺序保证） |
| 查用户变量用 `get_var()` / 写用 `set_var()` | **都不存在** → `get_variable()`（`item_func.cc:5869`）/ `Item_func_set_user_var::update_hash()`（`:6088`） |
| `show_innodb_vars` 是数组 | 是 **SHOW_FUNC 回调**（`ha_innodb.cc:21526`）；数组真名 `innodb_status_variables[]`（`:1126`）/ `..._export[]`（`:21956`） |
| `SHOW STATUS` 默认 GLOBAL | 默认 **SESSION**（`sql_yacc.yy:15935` `%empty { $$=OPT_SESSION; }`） |
| `MYSQL_THDVAR_*` 宏在 `sql/set_var.h` | **不在**（grep `THDVAR` 0 命中）→ 在 `include/mysql/plugin.h:463` |
| THDVAR 的 offset 是编译期常量 | **错** → 宏里硬写 `-1`，由 `sql_plugin.cc:3281` 在**插件装载时运行时回填**，resolve 也同批赋值 |
| `PLUGIN_VAR_*` 定义在 `plugin.h` | 只是引用 → 真值在 `include/mysql/components/services/bits/system_variables_bits.h:63` |
| 改全局变量只需 `SUPER` | 动态变量 `SUPER` **或** `SYSTEM_VARIABLES_ADMIN`；`PERSIST_ONLY` **只认动态权限**，SUPER 不够 |
| `SESSION_VARIABLES_ADMIN` 不存在 | **存在**，但**不是 C enum**，是字符串型动态权限 |
| PFS 变量/状态表 11 张 | **12 张** = 5 变量 + 6 状态 + **`user_variables_by_thread`**（文件前缀是 `table_user_variables`，按 `variables` grep 会漏） |
| I_S 表是"视图转发到 PFS" | **不是** → I_S 侧**彻底删除**（`sql/dd/impl/system_views/` grep `variables` 0 命中），SHOW 在**解析期**改写为对 PFS 的 SELECT |
| `@@version` 是 `Sys_var_charptr` | **错** → 专属类 **`Sys_var_version`**（`sys_vars.cc:5307`） |
| 配置文件支持 `\` 续行 / `$VAR` 展开 / 行尾 `;` 注释 | **都不支持** |
| `SET PERSIST` 源码里有 `WL#8688` | **0 命中**——该 WL 编号不在源码注释里 |

**★ `SHOW STATUS` vs `SHOW GLOBAL STATUS` 的实现差异只有一处**：换 PFS 表名（`parse_tree_nodes.cc:2813`），两者都走同一个 `build_query()`，生成同一形状的 SELECT。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 7.1.9.1 System Variable Privileges*、*7.1.9.3 Persisted System Variables*、*7.1.10 Server Status Variables*、*29.12.14.2 variables_info Table*、*29.12.15 Status Variable Tables*
- WorkLog: [WL#8688 SET PERSIST](https://dev.mysql.com/worklog/task/?id=8688)（含 variables_info 定义）、[WL#9720](https://dev.mysql.com/worklog/?k=persist)、[WL#9763](https://dev.mysql.com/worklog/?k=persist)、[WL#9787](https://dev.mysql.com/worklog/?k=persist)、[WL#681 SET_VAR hint](https://dev.mysql.com/worklog/task/?id=681)、[WL#8131 动态权限](https://dev.mysql.com/worklog/?k=dynamic+privilege)、[WL#12217 SESSION_VARIABLE_ADMIN](https://dev.mysql.com/worklog/task/?id=12217)

**他库文档**
- *PostgreSQL Documentation → 19.1 Setting Parameters*、*53.25 pg_settings*（`ALTER SYSTEM`/`postgresql.auto.conf`/`pg_file_settings`/context 分类）

**相关文档**
- PFS 的通用表访问机制、PFS 系统变量 cache 的线程间读细节见 [`pfs.md`](../../infra/observability/pfs/pfs.md)
- GTID 系列变量（`gtid_next`/`gtid_purged` 等）的读写语义见 [`../replication/gtid.md`](../replication/gtid.md)
- 引擎接口（`handlerton`/`handler`）见 [`../handler.md`](../handler.md)
- SET 语句解析（`PT_set_*` 语法树族）属于查询主链第 2 步，见 [`../query/README.md`](../query/README.md)
