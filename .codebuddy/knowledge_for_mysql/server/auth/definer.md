# definer / INVOKER 与 SQL SECURITY 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 definer 概念、`SQL SECURITY DEFINER/INVOKER` 两种权限语义——**两种都剖析到代码分支级**（存储例程 / 视图 / 触发器 / 事件四类对象各自的判定点）、默认 definer 的生成、`binlog_invoker`/`Q_INVOKER` 的主从一致性传递。
>
> **边界**：本篇讲 definer 机制本身（定义者 + 权限委托 + 复制传递）；definer 默认值用到的 `priv_user`/`priv_host` 见 [`security_context.md`](security_context.md)；多因素认证见 [`mfa.md`](mfa.md)；委托过去的权限如何被判定（分层判定链、ACL 缓存、role 展开）见 [`authorization.md`](authorization.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

definer（定义者）是 MySQL 四类"可被执行的持久化对象"的元数据属性，记录"这个对象是谁创建/拥有的"，格式为 `'user'@'host'`：

| 对象类型 | 例子 |
|---|---|
| 存储过程/函数（Stored Procedure/Function） | `CREATE PROCEDURE ...` |
| 触发器（Trigger） | `CREATE TRIGGER ...` |
| 视图（View） | `CREATE VIEW ...` |
| 事件（Event） | `CREATE EVENT ...` |

配合 `SQL SECURITY` 子句，definer 决定"执行这个对象时按谁的权限做检查"。

### 用途

解决"低权限用户安全访问高权限数据"的权限委托问题：一个没有 `SELECT sensitive_data` 权限的用户，可以通过一个 `SQL SECURITY DEFINER` 的存储过程（definer 是高权限用户）安全地读取聚合结果，而不需要直接获得敏感表的权限。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.0 | 引入存储例程 + definer；`SQL SECURITY DEFINER/INVOKER` |
| 5.0.13 | 视图/触发器/事件也支持 DEFINER 子句 |
| 8.0 | 引入 `SET_USER_ID` 动态权限，替代 `SUPER` 作为"可指定任意 definer"的权限 |

---

## 理论基础

### 设计思想与权衡

**核心是"权限受控委托"**。definer 让对象的创建者可以把"自己的一部分访问能力"封装进对象，供调用者按需使用，而调用者本身不必拥有那些权限。

**`SQL SECURITY` 两种模式的取舍**（这是本机制最核心的权衡）：

- `SQL SECURITY DEFINER`（默认）：执行时**切换 security context 到 definer**，按 definer 的权限做检查。代价：definer 权限泄漏风险——任何被授权 `EXECUTE` 的人都能借用 definer 的权限。
- `SQL SECURITY INVOKER`：执行时**用调用者（invoker）**的权限。代价：每个调用者都必须自己拥有对象内访问的所有表的权限，失去了"封装"的价值。

**默认值是 DEFINER**（`SP_DEFAULT_SUID_MAPPING = SP_IS_SUID`，`sp.h`），因为 DEFINER 才是"委托"的语义基础；INVOKER 是给"想让调用者自负其责"的场景。

**★ INVOKER 的安全模型**（源码注释原文划得很清楚）：INVOKER 意味着"**调用者必须先读过对象 body 才能安全调用**"——既然权限用的是你自己，那你本来就有责任审查它要访问什么。所以 MySQL 对 INVOKER 对象**不做额外防护**（如 `I_S.OPTIMIZER_TRACE` 的隐藏逻辑明确写了"Objects which are SQL SECURITY INVOKER are not considered here"）。反过来 DEFINER 才是需要额外约束的那个。

**★ 性能代价的真相（与直觉相反）**：DEFINER 每次 CALL 多付一次 `acl_getroot()` + 一次 `check_routine_access(EXECUTE_ACL)`；INVOKER 省掉这两步，但换来的是"**N 个调用者 × M 次调用**"的表级 ACL 计算。高并发下 DEFINER 反而**更省**（ACL cache 只有一个 key）。`sp_head.h` 里还挂着一条 TODO 承认 DEFINER 的二次检查**没做缓存**。

**★ 四类对象并非都支持 INVOKER**——这是最容易想当然的地方：

| 对象 | 能 INVOKER 吗 | 语法层 | 运行时判定点 |
|---|---|---|---|
| 存储过程 / 函数 | ✅ 能 | `sp_suid` 产生式 | `sp_head::set_security_ctx()` 的 `m_chistics->suid != SP_IS_NOT_SUID` |
| 视图 | ✅ 能 | `view_suid` 产生式 | `Table_ref::view_suid` 真假（`parse_view_definition()`） |
| **触发器** | ❌ **不能** | `CREATE TRIGGER` 走独立 `trigger_tail`，**根本不引用 `sp_suid`** | `assert(m_chistics->suid != SP_IS_NOT_SUID)`，且**无条件**调 `change_security_context()` |
| **事件** | ❌ **不能** | 语法解析结束时硬写 `lex->sp_chistics.suid = SP_IS_SUID; //always the definer!` | `Event_job_data::execute()` 无条件切 definer |

**为什么触发器不能有 INVOKER**：它由 DML **隐式**触发，"调用者"这个概念在语义上不存在——触发一条 `INSERT` 的用户并不知道会连带跑一段代码。所以语法层直接不给 `SQL SECURITY` 子句，`sql/trigger.cc` 里断言 `lex->definer != nullptr` 兜底。事件同理：它由调度器线程自主触发，连"谁调用"都没有。

**definer 必须主从一致**：因为 definer 影响权限语义，主从复制时 `CREATE/ALTER PROCEDURE/TRIGGER/VIEW/EVENT` 的 definer 必须随 binlog 传到从库，从库才能用同一个 definer 重建对象（否则权限语义不一致）。这正是 `m_binlog_invoker` 机制存在的意义。

### 理论溯源

definer 是"授权委托（delegation）"的落地——对象的定义者把自己持有的访问权委托给对象，调用者通过调用对象间接获得该访问权。这与 proxy user（见 [`security_context.md`](security_context.md)）是同一思想的两条路径：proxy 委托的是"账户身份"，definer 委托的是"对象执行时的权限"。

### 算法与数据结构

`SQL SECURITY` 的枚举，存储例程侧（`sql_lex.h:199`）：

```199:203:sql/sql_lex.h
enum enum_sp_suid_behaviour {
  SP_IS_DEFAULT_SUID = 0,
  SP_IS_NOT_SUID,   // SQL SECURITY INVOKER
  SP_IS_SUID        // SQL SECURITY DEFINER
};
```

视图侧（`table.h:2485`）：

```2485:2487:sql/table.h
#define VIEW_SUID_INVOKER 0
#define VIEW_SUID_DEFINER 1
#define VIEW_SUID_DEFAULT 2
```

两个枚举名不同但语义一致：`SP_IS_SUID == VIEW_SUID_DEFINER`，`SP_IS_NOT_SUID == VIEW_SUID_INVOKER`。它们最终都映射到 DD 的 `View::ST_DEFINER/ST_INVOKER`（`dd_routine.cc:352-357`）。

---

## 核心实现

### 默认 definer 的生成

建对象时若未显式写 `DEFINER = ...`，用当前用户。`get_default_definer`（`sql_parse.cc:6754`）：

```6754:6761:sql/sql_parse.cc
void get_default_definer(THD *thd, LEX_USER *definer) {
  const Security_context *sctx = thd->security_context();

  definer->user.str = sctx->priv_user().str;
  definer->user.length = strlen(definer->user.str);

  definer->host.str = sctx->priv_host().str;
  definer->host.length = strlen(definer->host.str);
```

关键：definer 取的是 **`priv_user`/`priv_host`（权限身份）**，不是 `user`（登录身份）——这与 [`security_context.md`](security_context.md) 里讨论的区分一致。

### definer 权限校验

指定了与当前用户不同的 definer 时，需要额外权限（`sql_view.cc:537`）：

```537:545:sql/sql_view.cc
  if (lex->definer &&
      (strcmp(lex->definer->user.str,
              thd->security_context()->priv_user().str) != 0 ||
       my_strcasecmp(system_charset_info, lex->definer->host.str,
                     thd->security_context()->priv_host().str) != 0)) {
    Security_context *sctx = thd->security_context();
    if (!(sctx->check_access(SUPER_ACL) ||
          sctx->has_global_grant(STRING_WITH_LEN("SET_USER_ID")).first)) {
      my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "SUPER or SET_USER_ID");
```

即：想"替别人"定义对象，需要 `SUPER` 或 `SET_USER_ID` 动态权限（8.0 里 `SET_USER_ID` 取代了 `SUPER` 的这个用途）。

### 执行时的权限切换：真正的开关在 `set_security_ctx()`

★ **更正一个常见误指**：SUID 的判断**不在** `sp_head::execute()` 里——`sp_head.cc` 全文只有两处 `suid`（一处是 `execute_trigger` 的断言，一处就是下面这个函数）。真正的开关是 **`sp_head::set_security_ctx()`**，由 `execute_procedure()` / `execute_function()` 在进入 `execute()` **之前**调用：

```cpp
// sql/sp_head.cc
bool sp_head::set_security_ctx(THD *thd, Security_context **save_ctx) {
  *save_ctx = nullptr;
  LEX_CSTRING definer_user = {m_definer_user.str, m_definer_user.length};
  LEX_CSTRING definer_host = {m_definer_host.str, m_definer_host.length};

  if (m_chistics->suid != SP_IS_NOT_SUID &&
      m_security_ctx.change_security_context(thd, definer_user, definer_host,
                                             m_db.str, save_ctx)) {
    return true;
  }

  /*
    If we changed context to run as another user, we need to check the
    access right for the new context again as someone may have revoked
    the right to use the procedure from this user.
  */
  if (*save_ctx &&
      check_routine_access(thd, EXECUTE_ACL, m_db.str, m_name.str,
                           m_type == enum_sp_type::PROCEDURE, false)) {
    m_security_ctx.restore_security_context(thd, *save_ctx);
    *save_ctx = nullptr;
    return true;
  }
  return false;
}
```

**这一个 `if` 就是 ROUTINE 侧 DEFINER / INVOKER 的全部差异**：

- **INVOKER**（`suid == SP_IS_NOT_SUID`）：条件为假 → 整段跳过 → `*save_ctx` 保持 `nullptr` → **连"切换后再查一次 `EXECUTE` 权限"这步都不发生**（因为上下文没变，调用者在 `CALL` 前已被 `SQLCOM_CALL` 分支查过了）。`thd->security_context()` 原封不动，就是连接用户。
- **DEFINER**：切到 definer 后**再查一次** `EXECUTE_ACL`（注释解释了原因：可能有人刚刚撤了 definer 的这个权限）。

### `change_security_context()`：只换指针，不拷贝字段

```cpp
// sql/auth/sql_security_ctx.cc
bool Security_context::change_security_context(
    THD *thd, const LEX_CSTRING &definer_user, const LEX_CSTRING &definer_host,
    const char *db, Security_context **backup, bool force = false) {
  bool needs_change;
  assert(definer_user.str && definer_host.str);

  *backup = nullptr;
  needs_change =
      (strcmp(definer_user.str, thd->security_context()->priv_user().str) ||
       my_strcasecmp(system_charset_info, definer_host.str,
                     thd->security_context()->priv_host().str));
  if (needs_change || force) {
    if (acl_getroot(thd, this, definer_user.str, definer_host.str,
                    definer_host.str, db)) {
      my_error(ER_NO_SUCH_USER, MYF(0), definer_user.str, definer_host.str);
      return true;
    }
    *backup = thd->security_context();
    thd->set_security_context(this);
  }
  return false;
}

void Security_context::restore_security_context(THD *thd,
                                                Security_context *backup) {
  if (backup) thd->set_security_context(backup);
}
```

★ **`backup` 存的不是字段拷贝，只是旧上下文的指针**；恢复就是一次 `thd->set_security_context(backup)` 指针写回。所以新上下文的内容由 `acl_getroot()` 直接写进调用方**自己的** `Security_context` 实例（`sp_head::m_security_ctx`、`Table_ref::view_sctx`、EVENT 里的栈上 `event_sctx`）——源码注释因此强调"临时上下文的内存管理由创建它的模块负责"。

**`backup` 同时兼任两个角色**：

1. **LIFO 恢复栈**：SP 调 SP、视图套视图，`thd->m_security_ctx` 是一条栈，不按 LIFO 恢复就会污染外层；
2. **"是否发生了切换"的标志**：`save_ctx != nullptr` 被上游用来决定要不要做额外动作（如上面那个二次 `EXECUTE` 检查，以及 `Event_job_data::execute()` 里 `if (save_sctx) { set_system_user_flag(thd); set_connection_admin_flag(thd); }`）。

★ **一条零成本路径**：若 definer 恰好就是当前用户（`needs_change == false` 且 `force == false`），**完全不切换、不调 `acl_getroot`**，`backup` 保持 `nullptr`。即"DEFINER 但 definer==invoker"时开销与 INVOKER 相同。

**definer 不存在（被删了）怎么办**：返回 `true` + `ER_NO_SUCH_USER`，且 `*backup` 仍是 `nullptr`（所以 `restore_security_context(thd, nullptr)` 安全）。**注意不会降级成 INVOKER 静默执行**——存储过程直接 CALL 失败；视图则分情况（`SHOW CREATE VIEW` 只给 warning，普通查询报 `ER_ACCESS_DENIED_ERROR` 而非 `ER_NO_SUCH_USER`，**故意不泄露 definer 是否存在**）。

### 视图的 INVOKER：不是"无条件用调用者"，而是向上回溯

视图的 SUID 分叉在 `parse_view_definition()`（`sql/sql_view.cc`）尾部：

```cpp
  Security_context *security_ctx;

  if (view_ref->view_suid) {
    /* For suid views prepare a security context for checking underlying
       objects of the view. */
    view_ref->view_sctx = new (thd->stmt_arena->mem_root) Security_context();
    security_ctx = view_ref->view_sctx;
    thd->m_view_ctx_list.push_back(view_ref->view_sctx);
  } else {
    /* For non-suid views inherit security context from view's table list.
       This allows properly handle situation when non-suid view is used
       from within suid view. */
    security_ctx = view_ref->security_ctx;
  }

  // Assign the context to the tables referenced in the view
  for (Table_ref *tbl = view_tables; tbl != view_tables_tail->next_global;
       tbl = tbl->next_global)
    tbl->security_ctx = security_ctx;
  ...
  result = view_ref->prepare_security(thd);
```

INVOKER 视图不创建 `view_sctx`、不调 `acl_getroot`，底层表的 `Table_ref::security_ctx` 被赋成"继承来的上下文"，最终由 `find_view_security_context()` 决定到底是谁：

```cpp
// sql/table.cc
Security_context *Table_ref::find_view_security_context(THD *thd) {
  Security_context *sctx;
  Table_ref *upper_view = this;
  while (upper_view && !upper_view->view_suid) {
    assert(!upper_view->prelocking_placeholder);
    upper_view = upper_view->referencing_view;   // 沿外层视图向上找
  }
  if (upper_view) {
    sctx = upper_view->view_sctx;   // 用最近的 SUID 视图的 definer
  } else {
    sctx = thd->security_context(); // 全链都不是 SUID → 调用者
  }
  return sctx;
}
```

★ **所以 INVOKER 视图的准确语义是**：沿 `referencing_view` 向上回溯，**用最近的那个 SUID 视图的 definer**；只有整条链都不是 SUID，才回落到调用者。**"INVOKER 视图套在 DEFINER 视图里"时，它用的是外层 DEFINER 的权限，不是调用者的权限。**

然后 `prepare_security()` 临时 `set_security_context(那个 sctx)`，对**每个底层表**调 `fill_effective_table_privileges()` 填 `tbl->grant` 再还原。真正的放行/拒绝仍由后续 `check_table_access()` 完成——它那句 `if (table_ref->security_ctx) sctx = table_ref->security_ctx;` 正是消费这里填好的值。

#### ★ 一个隐式坑：`VIEW_SUID_DEFAULT` 实际等价于 DEFINER

```cpp
#define VIEW_SUID_INVOKER 0
#define VIEW_SUID_DEFINER 1
#define VIEW_SUID_DEFAULT 2     // ← 注意是 2，不是 0
```

而 `view_suid` 的声明是**整数**（`ulonglong view_suid{0};`），所有判断都写成 `if (view_suid)` / `!view_suid`——**`VIEW_SUID_DEFAULT == 2` 非零即真，所以"不写 SQL SECURITY"落在 `if (view_suid)` 的真分支，等价于 DEFINER**。

`I_S.VIEWS` 的展示口径直接印证了这一点：

```cpp
"IF (vw.view_security_type='DEFAULT', 'DEFINER', vw.view_security_type)"
```

`ALTER VIEW` 时这个 bool 语义才被显式化（`fill_defined_view_parts()`）：

```cpp
  if (lex->create_view_suid == VIEW_SUID_DEFAULT)
    lex->create_view_suid = decoy.view_suid ? VIEW_SUID_DEFINER : VIEW_SUID_INVOKER;
```

旧格式视图还有一条兜底：`open_and_read_view()` 里 `view_ref->view_suid = true;`（默认 DEFINER）。

### SQL SECURITY 的持久化：语法 → LEX → DD → 内存 → 展示

`SQL SECURITY` 不是运行时状态，而是**写进数据字典的持久化属性**。完整链路如下（每段都可在源码中定位）。

**① 语法 → LEX**

```cpp
// sql/sql_yacc.yy
sp_suid:
          SQL_SYM SECURITY_SYM DEFINER_SYM { Lex->sp_chistics.suid = SP_IS_SUID; }
        | SQL_SYM SECURITY_SYM INVOKER_SYM { Lex->sp_chistics.suid = SP_IS_NOT_SUID; }
        ;
```

```cpp
// sql/sql_lex.h
enum enum_sp_suid_behaviour {
  SP_IS_DEFAULT_SUID = 0,   // 没写 SQL SECURITY
  SP_IS_NOT_SUID,           // SQL SECURITY INVOKER
  SP_IS_SUID                // SQL SECURITY DEFINER
};
```

★ 注意三态：**"没写"不等于"写了 DEFINER"**，它是独立的 `SP_IS_DEFAULT_SUID`，在写 DD 前才被折叠掉：

```cpp
// sql/sp.h
#define SP_DEFAULT_SUID_MAPPING SP_IS_SUID
```

**② LEX → DD**

```cpp
// sql/dd/dd_routine.cc  fill_dd_routine_info()
  View::enum_security_type sec_type;
  enum_sp_suid_behaviour sp_suid = (sp->m_chistics->suid == SP_IS_DEFAULT_SUID)
                                       ? SP_DEFAULT_SUID_MAPPING      // ← 折叠点
                                       : sp->m_chistics->suid;
  switch (sp_suid) {
    case SP_IS_SUID:     sec_type = View::ST_DEFINER; break;
    case SP_IS_NOT_SUID: sec_type = View::ST_INVOKER; break;
    default: assert(false); return true;
  }
  routine->set_security_type(sec_type);
  routine->set_definer(definer->user.str, definer->host.str);
```

★ **ROUTINE 复用了视图的枚举**（`dd::View::enum_security_type`），但因为折叠发生在写入前，**DD 里永远不会出现 `ST_DEFAULT`**（ROUTINE 侧）。视图侧则相反——`ST_DEFAULT` 是真实存在的值。

```cpp
// sql/dd/types/view.h
enum enum_security_type { ST_DEFAULT = 1, ST_INVOKER, ST_DEFINER };
```

DD 列定义（`sql/dd/impl/tables/tables.cc`）：`view_security_type ENUM('DEFAULT','INVOKER','DEFINER')`；ROUTINE 侧列名 `mysql.routines.security_type`。

> 两个 `*_impl` 的**构造函数默认值都是 `ST_INVOKER`**（`View_impl`、`Routine_impl`），但这只是 C++ 对象初值，不代表 SQL 层默认——SQL 层的默认由 `SP_DEFAULT_SUID_MAPPING` / `CREATE_VIEW_SUID` 决定为 DEFINER。**别把 C++ 初值当成语义默认。**

**③ DD → 内存**

```cpp
// sql/dd_sp.cc
  sp_chistics->suid = (routine->security_type() == dd::View::ST_INVOKER)
                          ? SP_IS_NOT_SUID
                          : SP_IS_SUID;
```

即读回时把三态**重新折叠成两态**：只有 `ST_INVOKER` 是 INVOKER，`ST_DEFAULT` 与 `ST_DEFINER` 都归为 DEFINER。视图侧同理经过 `dd_get_old_view_security_type()` 转回 `VIEW_SUID_*`。

**④ ALTER**

`alter_routine()` 只在 `suid != SP_IS_DEFAULT_SUID` 时才改（`SP_IS_DEFAULT_SUID` 的语义是"这次 ALTER 不动它"）。且**没有**运行时开关——既没有 `@@sql_security` 变量，也无法"临时把 DEFINER 当 INVOKER 跑"。

**⑤ 展示**

```cpp
// sql/sp.cc（binlog / SHOW CREATE 用的重建语句）
  if (chistics->suid == SP_IS_NOT_SUID)
    buf->append(STRING_WITH_LEN("    SQL SECURITY INVOKER\n"));
```

★ 注意这里**只打印 INVOKER**，DEFINER 是默认值不打印——与"默认即 DEFINER"一致。`sql_show.cc` 的 `SHOW CREATE PROCEDURE/FUNCTION` 则两种都打印。

### 可观测性：怎么查一个对象到底是 DEFINER 还是 INVOKER

| 想看什么 | 怎么看 | 注意 |
|---|---|---|
| 存储例程 | `SHOW CREATE PROCEDURE p` / `information_schema.routines.SECURITY_TYPE` | `SHOW CREATE` **只打印 INVOKER**（DEFINER 是默认、不打印）；I_S 列则是显式值 |
| 视图 | `information_schema.views.SECURITY_TYPE` | ★ 源码口径是 `IF(view_security_type='DEFAULT','DEFINER',…)` ⇒ **你永远看不到 `DEFAULT`，只会看到 `DEFINER`**（解析规则见上文「视图的 INVOKER」一节） |
| 触发器 / 事件 | 无 `SECURITY_TYPE` 列 | 语法层就不允许 INVOKER，故无需暴露 |
| definer 是谁 | `information_schema.{routines,views,triggers,events}.DEFINER` | 与 `SECURITY_TYPE` 是**两个独立属性**：DEFINER 记录"谁创建"，SQL SECURITY 决定"按谁判权" |

★ 排查口诀：**`SECURITY_TYPE` 说"按谁判权"，`DEFINER` 说"那个谁是谁"**。看到 `DEFINER` 但查不到该用户时，ROUTINE 是直接 CALL 失败；视图则分情况（`SHOW CREATE` 只 warning，普通查询报 `ER_ACCESS_DENIED_ERROR` 而非 `ER_NO_SUCH_USER`，故意不泄露账号是否存在）。

### 触发器与事件：语法层就锁死为 DEFINER

**触发器**——`sp_head::execute_trigger()` 里**无条件**切换（不判断 suid），并用断言保证：

```cpp
  /*
    While parsing CREATE TRIGGER statement or loading trigger metadata from
    the Data Dictionary we guarantee that definer hasn't empty value.
    It means, that trigger can't never be NOT-SUID.
  */
  assert(m_chistics->suid != SP_IS_NOT_SUID);
  if (m_security_ctx.change_security_context(thd, definer_user, definer_host,
                                             m_db.str, &save_ctx))
    return true;
  /* Check that the definer has TRIGGER privilege on the subject table. */
  if (!(grant_info->privilege & TRIGGER_ACL)) { ... ER_TABLEACCESS_DENIED_ERROR ... }
```

注意它额外查的是 **definer 对 subject table 的 `TRIGGER_ACL`**，不是查调用者。

**事件**——语法解析结束时直接硬写（`sql/sql_yacc.yy`，注释原文）：

```
  lex->sp_chistics.suid = SP_IS_SUID;  //always the definer!
```

执行期 `Event_job_data::execute()` 无条件切到 definer，并在 `save_sctx` 非空时补设 `SYSTEM_USER` / `CONNECTION_ADMIN` 标志位，最后再算回来。

★ **一个漂亮的性能优化**：EVENT 把 body 包装成一个临时 PROCEDURE 执行，而这个包壳 SP **显式写成 INVOKER**（`construct_sp_sql()`）：

```cpp
  /*
    The default SQL security of a stored procedure is DEFINER. We
    have already activated the security context of the event, so
    let's execute the procedure with the invoker rights to save on
    resets of security contexts.
  */
  sp_sql->append(STRING_WITH_LEN("() SQL SECURITY INVOKER "));
```

即 EVENT 已经在 definer 上下文里了，包壳用 INVOKER 是为了**少一次 `change_security_context`**，语义上仍等价于 definer。同样的手法也出现在 `sql/dd/impl/upgrade/server.cc`。

### binlog_invoker：definer 的主从传递（主链路）

```
主库执行 CREATE/ALTER PROCEDURE/TRIGGER/VIEW/EVENT（或 GRANT/REVOKE）
  → binlog_invoker() 置 m_binlog_invoker = true          （标记"要记录执行者"）
  → Query_log_event::write() 检测 need_binlog_invoker()   （log_event.cc:3489）
    → 编码 Q_INVOKER status 字段（user + host）
从库 SQL 线程读 Query_log_event
  → 解析 Q_INVOKER → set_invoker() 存 m_invoker_user/host
  → get_definer() 用 m_invoker_user/host 作为 definer/grantor
```

### 设置标志与编码 invoker

`binlog_invoker()` 是 THD 的 setter（`sql_class.h:4320`），在 `GRANT`/`REVOKE`（`sql_parse.cc:4018`、`4076`）、`CREATE/ALTER PROCEDURE`（`sql_parse.cc:4381`）等处调用。

`Query_log_event::write()` 消费标志，编码 invoker（`log_event.cc:3489-3510`）：

```3489:3510:sql/log_event.cc
  if (thd && thd->need_binlog_invoker()) {
    ...
    if (thd->slave_thread && thd->has_invoker()) {
      invoker_user = thd->get_invoker_user();
      invoker_host = thd->get_invoker_host();
    } else {
      Security_context *ctx = thd->security_context();
      LEX_CSTRING priv_user = ctx->priv_user();
      LEX_CSTRING priv_host = ctx->priv_host();
      ...
```

主库写 invoker 用 `priv_user`/`priv_host`；从库再转发（级联复制）时用 `m_invoker_user`/`m_invoker_host`（从 master 传来的值）。编码成 `Q_INVOKER`（code=11）status 字段：1 字节 user 长度 + user + 1 字节 host 长度 + host。

★ **澄清一个容易想当然的结论**：`Q_INVOKER` **与 `SQL SECURITY` 是完全独立的两条通道**，不要因为都带 "invoker" 就认为它们相关：

- `m_binlog_invoker` 的置位是**无条件**的，**完全不看 `suid`**——`CREATE PROCEDURE ... SQL SECURITY INVOKER` 一样会 `thd->binlog_invoker()`；
- 反过来看 `definer` 也不对：`THD::get_definer()` 是在"definer **缺失**、需要生成默认 definer"时才被打上标记，显式写了 `DEFINER=` 的语句反而不走它；
- **CALL / `SELECT func()` 从不设置**：`sp_head.cc` 全文 grep `binlog_invoker` **0 命中**。所以"调用一个 DEFINER 过程"的 binlog 里**不会**带 `Q_INVOKER`。

`Q_INVOKER` 解决的真正问题是：**从库上这条 DDL 该以谁的身份执行/授权**（grantor + 默认 definer + `sp_automatic_privileges` 的自动授权），属于**复制**范畴；而运行期的 `SQL SECURITY INVOKER` 是**权限判定**范畴。

### Q_INVOKER 的解码与使用

从库 SQL 线程解析 `Q_INVOKER`（`libbinlogevents/src/statement_events.cpp:239`）读出 user/host，存进 `m_invoker_user`/`m_invoker_host`（`set_invoker`，`sql_class.h:4323`）。后续 `get_definer`（`sql_class.cc:2528`）检查 `slave_thread && has_invoker()`，用 master 传来的值作为 definer/grantor。

---

## 相关的系统变量/状态变量

definer 本身无专属系统变量。相关的权限控制是动态权限 `SET_USER_ID`（允许指定任意 definer，见 [`mfa.md`](mfa.md) 的相邻权限主题），以及 `check_proxy_users` 等 proxy 相关变量（见 [`security_context.md`](security_context.md)）。

---

## Misc

### 易混淆概念

- **definer vs invoker**：definer 是对象的**定义者**（创建者，元数据一部分）；invoker 是对象的**调用者**（执行时是谁发起的）。`SQL SECURITY DEFINER` 按 definer 权限执行，`SQL SECURITY INVOKER` 按 invoker 权限执行。
- **definer 的 `DEFINER=` 子句 vs `Q_INVOKER`**：前者是 SQL 文本的一部分（`CREATE DEFINER=... PROCEDURE`），mysqlbinlog 直接可见；后者是 `Query_log_event` 的 status 字段（`Q_INVOKER`，code=11），存的是账户管理语句的 grantor，mysqlbinlog 默认不展示，只给从库 SQL 线程消费。
- **`m_binlog_invoker` 的注释名 `current_user_used` 是历史遗留**：字段注释（`sql_class.h:4356`）写的是 `current_user_used`，但实际字段名是 `m_binlog_invoker`——注释没跟着改名更新。
- **`INVOKER` 视图 ≠ 调用者权限**：它继承**最近的 SUID 外层视图**的 definer，只有整条链都不是 SUID 才回落到调用者（见「核心实现 → 视图的 INVOKER」一节）。

### INVOKER / DEFINER 相关的已知坑

源码注释原文承认的限制，逐条列出：

1. **★ 唯一一处服务端静默把对象改成 INVOKER 的地方，且官方承认会造成主从不一致**（`sql_parse.cc` 的 `sp_process_definer()`）：
   > *"Actually, this leads to the inconsistent state of master and slave (different definers, different SUID behaviour), but it seems, this is the best we can do."*

   触发条件是"从库收到一条不带 definer 的 `CREATE ROUTINE`"——此时 `lex->sphead->m_chistics->suid = SP_IS_NOT_SUID`。触发器走的是另一条路：直接报 `ER_TRG_NO_DEFINER` 失败（此处**注释与实现已不同步**，注释说"mark as non-SUID"，代码是 `my_error`）。

2. **建视图时不下钻存储过程 body**（`Item_func_sp::fix_fields()` 注释）：
   > *"during view creation we do not infer into stored routine bodies and do not check privileges of its statements, which would probably be a good idea especially if the view has SQL SECURITY DEFINER and the used stored procedure has SQL SECURITY DEFINER."*

   ⇒ **DEFINER 视图 + DEFINER 存储过程**这个组合在 `CREATE VIEW` 期是**权限盲区**，只在运行期才暴露。

3. **EXPLAIN 借不到 DEFINER 的权限**：即使视图是 DEFINER，`EXPLAIN` / `SHOW CREATE VIEW` 也强制用**活跃用户**的 sctx 检查 `SHOW_VIEW_ACL` + `SELECT_ACL`（构造一个 `security_ctx == nullptr` 的辅助 `Table_ref` 来强制这一点），避免你借 definer 权限看到无权看的结构。

4. **EVENT 调度线程无法做安全检查**：它的 `thd->m_main_security_ctx.priv_user == NULL`（空用户），所以 `I_S.OPTIMIZER_TRACE` 对它"do no security checks, so cannot safely enable tracing"。相关地，`change_security_context()` 的注释里挂着一句技术债：
   > *"The Security_context_factory should be used as a replacement to this function at every opportunity."*

   而这个 `Security_context_factory` 在 8.0.39 里**并未找到**——属于未完成的重构。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Stored Programs and Views → Access Control for Stored Programs and Views
- MySQL 8.0 Reference Manual → `CREATE PROCEDURE` / `CREATE VIEW` 的 `DEFINER` 与 `SQL SECURITY`

**相关文档**
- definer 默认值用到的 `priv_user`/`priv_host` 见 [`security_context.md`](security_context.md)
- 多因素认证见 [`mfa.md`](mfa.md)
