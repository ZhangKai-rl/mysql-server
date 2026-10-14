# validate_password：插件/组件双轨并存的完整样本

> 基于 MySQL 8.0.39 源码。本篇剖析**唯一一个"同一功能、两套完整实现、运行时择优分派"**的样本：`validate_password` 既有插件版（`plugin/password_validation/`）又有组件版（`components/validate_password/`），且两者都还活着。
>
> **边界**：本篇讲这一个功能的双轨实现与分派。插件框架本身见 [`plugin.md`](plugin.md)，组件框架见 [`component.md`](component.md)，服务契约见 [`service.md`](service.md)，服务注册表见 [`registry.md`](registry.md)，跨边界 ABI 见 [`abi.md`](abi.md)；另一条双轨（keyring）见 [`keyring.md`](keyring.md)；架构层面的迁移全景见 [`architecture.md`](architecture.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`validate_password` 是 MySQL 的**口令强度校验器**：`CREATE USER` / `ALTER USER ... IDENTIFIED BY` / `SET PASSWORD` 时判断新口令是否满足策略，并提供 `VALIDATE_PASSWORD_STRENGTH()` 函数给出 0–100 的强度分。

8.0.39 里它以**两种形态同时存在**：

| 形态 | 源码 | 构建产物 | 契约 |
|---|---|---|---|
| 插件 | `plugin/password_validation/validate_password.cc` | `validate_password.so` | `st_mysql_validate_password`（类型 `MYSQL_VALIDATE_PASSWORD_PLUGIN` = 8） |
| 组件 | `components/validate_password/validate_password_imp.cc` | `component_validate_password.so` | 服务 `validate_password` + `validate_password_changed_characters` |

两者**都是 `MODULE_ONLY`**（`plugin/password_validation/CMakeLists.txt:24`、`components/validate_password/CMakeLists.txt:24`）——都编译成独立 `.so`，都不进 `mysql_mandatory_plugins[]`。这与 keyring 不同：keyring 侧有一个 builtin 的 `daemon_keyring_proxy`，而 validate_password 侧**没有任何 builtin 成分**。

```cmake
# plugin/password_validation/CMakeLists.txt
MYSQL_ADD_PLUGIN(validate_password validate_password.cc MODULE_ONLY)

# components/validate_password/CMakeLists.txt
MYSQL_ADD_COMPONENT(validate_password validate_password_imp.cc MODULE_ONLY
  LINK_LIBRARIES library_mysys)
```

### 用途

- **强口令强制**：长度、数字数、大小写混合数、特殊字符数四档硬指标 + 三档策略（LOW/MEDIUM/STRONG）
- **字典检查**：口令不得包含字典词（用于 STRONG 档判定）
- **用户名关联检查**：口令不得等于/包含用户名或其反转（`check_user_name`）
- **变更比例检查**：新口令相对旧口令必须改动至少 N% 的字符（**组件版独有**）
- **强度评分**：`VALIDATE_PASSWORD_STRENGTH()` 给 UI 用的 0–100 分

### 版本演进（源码证据）

| 时期 | 事件 | 证据 |
|---|---|---|
| 5.6 / 5.7 | 插件版诞生 | `plugin/password_validation/CMakeLists.txt` 版权年 **2012** |
| 8.0.x | 组件版加入（`mysql:validate_password`） | `components/validate_password/CMakeLists.txt` 版权年 **2017**；`DECLARE_COMPONENT(validate_password, "mysql:validate_password")` |
| 8.0.34+ | 组件版新增 `changed_characters_percentage` | `register_variable("validate_password", "changed_characters_percentage", ...)`（`validate_password_imp.cc:854`）；插件版**没有**这个变量 |
| 8.0.39 | 插件版仍可用但**已标记弃用** | `validate_password_init()` 首行 `push_deprecated_warn(thd, "validate password plugin", "validate_password component")` |

> 组件版首次随哪个 8.0 小版本发布**未核实**（WorkLog 未检索），只能给出源码中的版权年证据。

---

## 理论基础

### 设计思想与权衡

**① 为什么同一个功能要维护两套完整实现？**

因为迁移的约束是"**调用点不能动**"。`validate_password` 的调用点散布在认证插件的哈希生成函数里（`sql_authentication.cc:4306/4368`、`sha2_password.cc:1142`），这些地方属于**认证插件的内部逻辑**——如果要求它们改调组件服务，就得改认证插件；而认证插件是 mandatory builtin、是登录路径。于是 Oracle 选择了一个折中：

> **保留插件版作为 fallback，新增组件版作为首选，在"服务器 ↔ 校验器"之间插一层运行时分派器**（`sql/auth/password_policy_service.cc`），调用点**一行都不用改**。

这就是所谓"**就地替换**"策略：迁移的单位不是"一个调用点"，而是"一层间接"。代价是**两套策略代码长期并存**（两版的 `validate_password_policy_strength()` 逻辑几乎重复），收益是迁移对存量代码零侵入。

**② 运行时择优 vs 构建期二选一——为什么不是编译开关？**

分派逻辑的核心两行（`password_policy_service.cc:85`）：

```c
if (!srv_registry->acquire("validate_password", &h_pv_svc)) {
    /* 组件存在 → 用服务 */
} else {
    /* acquire 失败 → 回落插件 */
    plugin = my_plugin_lock_by_name(nullptr, validate_password_plugin,
                                    MYSQL_VALIDATE_PASSWORD_PLUGIN);
}
```

**用 `acquire` 的返回值当"装没装组件"的探测器**。这意味着：

- 决定走哪条路的是**运行状态**（组件有没有被 `INSTALL COMPONENT`），不是编译期常量；
- 装了组件 → 走组件；没装但装了插件 → 走插件；都没装 → `acquire` 失败、`my_plugin_lock_by_name` 返回 `nullptr`，**校验直接跳过**（`res` 保持 0，放行）；
- 这是一个"**能力探测式**"分派，而不是"版本协商式"——没有版本号参与，只有"有没有"这个二值。

如果做成编译开关，那么发行版必须在打包时二选一，且用户装了 `.so` 也用不上。运行时探测把选择权交给了 DBA。

★ 这与 keyring 的路线**完全不同**：keyring 是"插件 → `daemon_keyring_proxy`（builtin 代理）→ 组件服务"的**桥接**（插件和组件在服务器内部被强行打通，见 [`keyring.md`](keyring.md)），而 validate_password 是"二选一"的**分派**（两者从不同时工作）。两条不同的迁移手法，源于 keyring 需要"已有加密数据必须用同一把钥匙解"，而 validate_password 无状态、无存量数据。

**③ 组件版为什么"更强"：服务契约里多一个 `void *thd`。**

对照两个契约的签名：

```c
/* 插件 vtable（include/mysql/plugin_validate_password.h） */
int  (*validate_password)(mysql_string_handle password);
int  (*get_password_strength)(mysql_string_handle password);

/* 组件服务（include/mysql/components/services/validate_password.h） */
DECLARE_BOOL_METHOD(validate, (void *thd, my_h_string password));
DECLARE_BOOL_METHOD(get_strength, (void *thd, my_h_string password,
                                   unsigned int *strength));
```

**组件版的每个方法都显式传入 THD**。这不是风格差异，而是能力差异的直接来源：

- `check_user_name`（口令不能等于用户名）需要**是谁在改密码** → 组件版用 `is_valid_password_by_user_name(thd, password)` 从 THD 取用户；
- `changed_characters_percentage`（新旧口令差异比例）需要**旧口令** → 只有拿到 THD 才能追溯当前用户/当前凭据上下文，插件版根本无法实现，所以**插件版没有这个变量**；
- 组件版还能用 `mysql_thd_security_context` 服务做更细的权限判断（在 `requires` 列表里）。

**结论：插件类型 vtable 是"无状态函数表"，服务是"带会话上下文的函数表"**——这是"为什么很多功能最终要迁到组件"的一条具体理由，而不只是架构洁癖。

**④ 变量命名为什么一个用点、一个用下划线？**

| 版本 | 变量示例 |
|---|---|
| 插件版 | `validate_password_length`、`validate_password_policy`、`validate_password_dictionary_file` |
| 组件版 | `validate_password.length`、`validate_password.policy`、`validate_password.dictionary_file` |

技术根因：插件变量是 `MYSQL_SYSVAR_*` 宏**编译期**声明的，名字带插件名前缀、由插件框架统一注册；组件变量是 `component_sys_variable_register->register_variable(...)` **运行期**一个个注册进**全局系统变量命名空间**的：

```c
mysql_service_component_sys_variable_register->register_variable(
    "validate_password", "length", PLUGIN_VAR_INT | PLUGIN_VAR_RQCMDARG,
    "Password validate length to check for minimum password_length",
    nullptr, length_update, (void *)&length, (void *)&validate_password_length);
```

注册接口是**两段式**（前缀 `"validate_password"` + 名 `"length"`），框架用点号拼成 `validate_password.length`。**必须加点号**，否则 `validate_password_length` 会与插件版撞名——因为两套实现可以同时安装，撞名会让 `SET` 语句指向错误的那个。点号在这里是**命名空间隔离的强制手段**，不是美化。

（状态变量同理：插件版 `validate_password_dictionary_file_last_parsed`，组件版 `validate_password.dictionary_file_last_parsed`。）

**⑤ 弃用路径：不删除、不禁用，只打警告。**

```c
static int validate_password_init(MYSQL_PLUGIN plugin_info) {
  push_deprecated_warn(thd_get_current_thd(), "validate password plugin",
                       "validate_password component");
  dictionary_words = new set_type();
  if (init_logging_service_for_plugin(&reg_srv, &log_bi, &log_bs)) return (1);
  ...
}
```

与 keyring 插件完全同款的三段式（打警告 → 照常工作 → 指向替代品）。注意 `init_logging_service_for_plugin()`——**插件版要写日志必须经过老版 plugin service 桥接**去 `acquire` 组件的 `log_builtins` 服务（见 [`service.md`](service.md) 的"老版 plugin service"章）。一个"插件侧的弃用实现"自己就在使用"组件时代的设施"，这在 8.0 里是常态。

### 算法与数据结构

**三档策略判定**（`validate_password_policy_strength()`，两版逻辑同构，以组件版为准）：

```
policy == LOW    → 只检查长度                          → 满足即通过
policy == MEDIUM → 长度 + 数字数 + 大小写混合数 + 特殊字符数
policy == STRONG → MEDIUM 全部 + 字典检查（不含字典子串）
```

**强度分**（`get_strength()`，`validate_password_imp.cc:531-576`），关键常量 `PASSWORD_SCORE = 25`（`:44`）：

| 情况 | 分值 |
|---|---|
| 字符数 < `MIN_DICTIONARY_WORD_LENGTH`（4） | 直接失败（`return true`） |
| 长度 < `validate_password.length` | **25** |
| 满足 LOW | **50** = `(0+1)*25+25` |
| 满足 MEDIUM | **75** |
| 满足 STRONG（含字典检查通过） | **100** |

即 `*strength = (policy + 1) * PASSWORD_SCORE + PASSWORD_SCORE`。注意这是**离散的 5 档**，不是连续评分——`VALIDATE_PASSWORD_STRENGTH()` 的返回值只有 0/25/50/75/100 五种可能（外加失败时函数返回 0）。

**字典检查**（`validate_dictionary_check()`）：

- 容器：`std::set<std::string>`（`set_type`），二分查找；
- 加载：`read_dictionary_file()`，文件大小硬上限 `MAX_DICTIONARY_FILE_LENGTH = 1MB`，超限直接放弃加载；
- 匹配：对口令取**所有长度 ≥ 4 的子串**逐一在 set 里 `find()`（`:271`），命中即判"含字典词"；
- 切换：新字典读进临时 set，`dictionary_activate()` 里 `std::swap(*dictionary_words, *dict_words)` **整体替换**（`:138`），配合 `mysql_rwlock_t` 读写锁——**保证改 `dictionary_file` 变量时不会读到半新半旧的字典**；
- `dictionary_update()` 是变量的 update 回调（`:437`）：改变量即触发重新加载。

**变更比例检查**（组件独有，`validate_password_changed_characters_imp::validate()`，`:614`）：

1. `validate_password_changed_characters_percentage == 0` → 直接放行（相当于关闭）；
2. 新旧口令都转小写（`mysql_string_case->tolower`），用 `create_scope_guard` 保证句柄释放；
3. 需要变更的字符数 = `max(validate_password.length, 旧口令长度) × 百分比 / 100`（`:659-662`）；
4. 实际变更数与阈值比较，不满足则报错并回填 `minimum_required` / `changed` 供错误消息使用。

### 他库对比

| 数据库 | 口令策略机制 | 差异 |
|---|---|---|
| **PostgreSQL** | `passwordcheck`（contrib 模块，`check_password_hook`） | C 扩展、编译期 `shared_preload_libraries` 加载；无三档策略、无字典、无强度分；只能用 CrackLib 变体做字典 |
| **MySQL 8.0 插件版** | `validate_password.so`，运行时 `INSTALL PLUGIN` | 三档 + 字典 + 强度分，但**无 THD**，做不到"与旧口令比对" |
| **MySQL 8.0 组件版** | `component_validate_password.so`，`INSTALL COMPONENT` | 上述全部 + THD 相关能力 + 变量用点号命名空间 |
| **SQL Server** | Windows 口令策略（NetValidatePasswordPolicy） | 委托操作系统；不能自定义字典 |
| **Oracle** | `PROFILE` + `verify_function` 存储过程 | 策略是**数据库里的 PL/SQL**，可任意扩展；MySQL 的策略是 C/C++ 代码 |

★ MySQL 的独特之处：**校验器是可插拔的二进制模块，而不是数据库内的脚本**——换来更强的表达能力（字典、字符集处理），代价是必须编译、部署、且要处理 ABI 兼容（见 [`abi.md`](abi.md)）。

---

## 核心实现

### 主链路

```
CREATE USER / ALTER USER ... IDENTIFIED BY / SET PASSWORD
  → mysql_create_user() / mysql_alter_user()            (sql/auth/sql_user.cc)
  → 插件的 generate_authentication_string()             ← 认证插件的成哈希函数
      · sql_authentication.cc:4306   (mysql_native_password)
      · sql_authentication.cc:4368   (sha256_password)
      · sha2_password.cc:1142        (caching_sha2_password)
  → my_validate_password_policy(inbuf, inbuflen)        (sql/auth/password_policy_service.cc:73)
      ├─ srv_registry->acquire("validate_password")     成功 → 组件服务 validate()
      └─ 失败 → my_plugin_lock_by_name(..., MYSQL_VALIDATE_PASSWORD_PLUGIN) → 插件 validate_password()
  → 不通过 → my_error(ER_NOT_VALID_PASSWORD)

VALIDATE_PASSWORD_STRENGTH('pw')                        (sql/item_func.cc:4210)
  → my_calculate_password_strength()                    (password_policy_service.cc:131)
      └─ 同样的双轨分派
```

### 双轨分派器：`password_policy_service.cc` 逐行

```c
int my_validate_password_policy(const char *password, unsigned int password_len) {
  plugin_ref plugin;
  String password_str;
  my_h_service h_pv_svc = nullptr;
  SERVICE_TYPE(validate_password) * ret;
  int res = 0;

  if (password) {
    String tmp_str(password, password_len, &my_charset_utf8mb3_bin);
    password_str = tmp_str;
  }
  if (!srv_registry->acquire("validate_password", &h_pv_svc)) {     /* ① 先问注册表 */
    ret = reinterpret_cast<SERVICE_TYPE(validate_password) *>(h_pv_svc);
    if (ret->validate((void *)current_thd, (my_h_string)&password_str)) {   /* ② 带 THD */
      my_error(ER_NOT_VALID_PASSWORD, MYF(0));
      res = 1;
    }
    srv_registry->release(h_pv_svc);                                 /* ③ 立即释放 */
  } else {                                                           /* ④ 回落插件 */
    plugin = my_plugin_lock_by_name(nullptr, validate_password_plugin,
                                    MYSQL_VALIDATE_PASSWORD_PLUGIN);
    if (plugin) {
      st_mysql_validate_password *password_validate =
          (st_mysql_validate_password *)plugin_decl(plugin)->info;
      if (!password_validate->validate_password(&password_str)) {    /* ⑤ 无 THD */
        my_error(ER_NOT_VALID_PASSWORD, MYF(0));
        res = 1;
      }
      plugin_unlock(nullptr, plugin);
    }
  }
  return res;
}
```

五个要点：

1. **`acquire` 即探测**：`acquire` 成功 = 组件已安装。它同时**加锁了服务的引用计数**，所以组件在校验期间不会被卸载（组件卸载时若服务被引用会直接拒绝，见 [`component.md`](component.md)）。
2. **THD 是能力分水岭**：组件路径传 `(void *)current_thd`，插件路径不传。
3. **句柄立即 `release`**：服务句柄只在这一次校验期间持有，不缓存——校验是低频操作，不值得为此长期持有引用（对照 [`abi.md`](abi.md) 的"调用形态三分层"）。
4. **插件路径走 `plugin_lock_by_name` + `plugin_unlock`**：这正是插件引用计数与延迟收割的用法（见 [`plugin.md`](plugin.md)），**每次校验都锁一次、解锁一次**。
5. **`String` 被当成 `my_h_string` 强转**：`(my_h_string)&password_str`——`my_h_string` 是不透明句柄，这里服务器把内部 `String*` 直接塞进去，因为**组件与服务器同进程、同 ABI**（信任模型，见 [`abi.md`](abi.md) 设计思想 ②）。

`my_calculate_password_strength()`（`:131`）结构完全相同，只是调用 `get_strength()` / `get_password_strength()` 并把结果作为函数返回值。

### 组件版骨架

```c
/* validate_password_imp.cc:1127 */
DECLARE_COMPONENT(validate_password, "mysql:validate_password")
validate_password_init, validate_password_deinit
END_DECLARE_COMPONENT();

BEGIN_COMPONENT_REQUIRES(validate_password)
    REQUIRES_SERVICE(component_sys_variable_register),
    REQUIRES_SERVICE(component_sys_variable_unregister),
    REQUIRES_SERVICE(status_variable_registration),
    REQUIRES_SERVICE(log_builtins), REQUIRES_SERVICE(log_builtins_string),
    REQUIRES_SERVICE(mysql_string_factory), ... /* mysql_string_* 共 7 个 */
    REQUIRES_SERVICE(mysql_thd_security_context),
    REQUIRES_SERVICE(validate_password),        /* 自己实现的服务也列在 requires 里 */
END_COMPONENT_REQUIRES();
```

- 提供**两个服务**：`validate_password`（`validate` / `get_strength`）、`validate_password_changed_characters`（`validate`）。
- 注册 **8 个系统变量**：`length`、`number_count`、`mixed_case_count`、`special_char_count`、`policy`、`dictionary_file`、`check_user_name`、`changed_characters_percentage`。
- 失败回滚是经典的 **goto 链**（`:873-891`）：任何一个 `register_variable()` 失败，就按注册的反序逐个 `unregister_variable()`，每个标签对应一个变量——**没有 RAII 兜底**，因为这是 C 风格的服务接口。
- `is_initialized` 是个 `std::atomic`，`get_strength()` 开头检查它（`:540`），未初始化时打 warning 并失败——防止"组件已注册服务但 init 未完成"的窗口期。

### 插件版骨架

```c
/* validate_password.cc:463 */
static struct st_mysql_validate_password validate_password_descriptor = {
    MYSQL_VALIDATE_PASSWORD_INTERFACE_VERSION,
    validate_password,          /* validate function          */
    get_password_strength       /* validate strength function */
};

mysql_declare_plugin(validate_password){
    MYSQL_VALIDATE_PASSWORD_PLUGIN, &validate_password_descriptor,
    "validate_password", PLUGIN_AUTHOR_ORACLE, "check password strength",
    PLUGIN_LICENSE_GPL, validate_password_init, nullptr,
    validate_password_deinit, 0x0101,
    validate_password_status_variables, validate_password_system_variables,
    nullptr, 0,
} mysql_declare_plugin_end;
```

**vtable 只有 2 个函数指针 + interface version**——是全部插件类型里最薄的契约之一。变量 7 个（`MYSQL_SYSVAR_INT` ×4 / `ENUM` / `STR` / `BOOL`），比组件版少 `changed_characters_percentage`。

### 契约对照表

| 维度 | 插件版 | 组件版 |
|---|---|---|
| 契约形式 | `st_mysql_validate_password`（类型 vtable，2 函数） | 2 个服务（`validate_password`、`validate_password_changed_characters`） |
| 会话上下文 | **无 THD** | **每个方法带 `void *thd`** |
| 变量注册 | `MYSQL_SYSVAR_*` 编译期声明 | `component_sys_variable_register` 运行期注册（两段式） |
| 变量命名 | `validate_password_length`（下划线） | `validate_password.length`（点号） |
| 变量数量 | 7 | 8（多 `changed_characters_percentage`） |
| 状态变量 | `validate_password_dictionary_file_last_parsed` / `..._words_count` | `validate_password.dictionary_file_last_parsed` / `..._words_count` |
| 安装 | `INSTALL PLUGIN validate_password SONAME 'validate_password.so'` | `INSTALL COMPONENT 'file://component_validate_password'` |
| 卸载 | `UNINSTALL PLUGIN`（走延迟收割） | `UNINSTALL COMPONENT`（服务被引用则拒绝） |
| 日志 | `init_logging_service_for_plugin()` 桥接组件日志服务 | 直接 `REQUIRES_SERVICE(log_builtins)` |
| 弃用状态 | **已弃用**（init 打 warning） | 现行 |
| 二进制 | `validate_password.so` | `component_validate_password.so`（`MYSQL_ADD_COMPONENT` 命名规则） |

### 上游导出：它自己也是个"老版 plugin service"

`sql/sql_plugin_services.h:271`：

```c
static struct mysql_password_policy_service_st mysql_password_policy_handler = {
    my_validate_password_policy, my_calculate_password_strength};
```

即 `my_validate_password_policy` / `my_calculate_password_strength` 这两个**双轨分派函数**本身被注册成一条**老版 plugin service**，供其它插件（如 X Plugin）调用。于是调用链可以多一层：

```
X Plugin（改口令）
  → mysql_password_policy_service（老版 plugin service，dlsym 注入的指针）
  → my_validate_password_policy()（双轨分派）
  → 组件服务 或 插件 vtable
```

**一条链上同时出现了三代机制**：老版 plugin service（桥接层）→ 运行时双轨分派 → 组件服务。这是 8.0 "三代同堂"最具体的一例。

---

## 可观测性

### 系统变量（两套命名，注意别混用）

| 组件版（点号） | 插件版（下划线） | 默认 | 说明 |
|---|---|---|---|
| `validate_password.length` | `validate_password_length` | 8 | 最小长度 |
| `validate_password.number_count` | `validate_password_number_count` | 1 | 最少数字个数 |
| `validate_password.mixed_case_count` | `validate_password_mixed_case_count` | 1 | 最少大小写混合数 |
| `validate_password.special_char_count` | `validate_password_special_char_count` | 1 | 最少特殊字符数 |
| `validate_password.policy` | `validate_password_policy` | `MEDIUM` | LOW/MEDIUM/STRONG 枚举 |
| `validate_password.dictionary_file` | `validate_password_dictionary_file` | 空 | 字典文件路径（**有 update 回调触发重载**） |
| `validate_password.check_user_name` | `validate_password_check_user_name` | ON | 禁止口令等于/包含用户名 |
| `validate_password.changed_characters_percentage` | —（**无**） | 0 | 新旧口令最少变更百分比 |

### 状态变量

| 组件版 | 插件版 | 说明 |
|---|---|---|
| `validate_password.dictionary_file_last_parsed` | `validate_password_dictionary_file_last_parsed` | 字典最后解析时间 |
| `validate_password.dictionary_file_words_count` | `validate_password_dictionary_file_words_count` | 载入的词数 |

### 错误码

| 错误码 | 触发 |
|---|---|
| `ER_NOT_VALID_PASSWORD` | 校验不通过（两个路径共用一个错误码，DBA 分不出是哪条路判的） |
| `ER_VALIDATE_PWD_STRING_HANDLER_MEM_ALLOCATION_FAILED` | 组件版 `mysql_string_factory->create()` 失败 |
| `ER_VALIDATE_PWD_STRING_CONV_TO_LOWERCASE_FAILED` | 字典检查/变更比例检查的小写转换失败 |
| `ER_VALIDATE_PWD_COULD_BE_NULL` | `get_strength()` 里字符串迭代器创建失败 |

### 诊断要点

- **判断当前生效的是哪条路**：`SELECT * FROM information_schema.plugins WHERE plugin_name='validate_password'` 看插件在不在；组件侧查 `mysql.component` 表。**组件优先**，两者都在时组件生效，插件的变量仍然可见但**不生效**。
- **策略看起来没变**：改了 `validate_password.policy`（组件）却仍在改 `validate_password_policy`（插件）——两者是不同变量，都 `SET` 成功，但只有组件那个被读取。

---

## Misc

### 易混淆点

1. **`validate_password` 这个名字出现 4 次**：插件名、组件名、服务名、变量名前缀。看代码时务必分清是哪个。
2. **组件版的 `REQUIRES_SERVICE(validate_password)` 是自己**：组件既提供又声明依赖同一个服务（供内部调用），不是笔误。
3. **`srv_registry->acquire()` 失败不等于错误**：它是分派器的正常分支（"没装组件"）。
4. **`VALIDATE_PASSWORD_STRENGTH()` 只有 5 种返回值**（0/25/50/75/100），不是连续分数。

### 坑

- **两套同时安装**：完全允许，但只有组件在工作；插件版还会照常打出弃用警告，容易误导排查。
- **变量双份**：两套变量并存于 `SHOW VARIABLES`，`SET` 都不会报错，但只有当前生效那一套起作用。这是"运行时注册撞名"用点号规避的**直接后果**——代价是 DBA 要记住哪个是活的。
- **字典文件超限静默失效**：文件 > 1MB 时 `read_dictionary_file()` 直接返回，**不会报错**，表现为"STRONG 策略下字典检查形同虚设"。
- **字典重载的窗口**：靠 `std::swap` + rwlock 保证原子切换，但**加载期间**整表重读，大字典会有可感知的停顿。
- **组件未初始化窗口**：`is_initialized` 原子标志存在，说明确实有"服务已注册但 init 未完成"的窗口期，此时 `get_strength()` 返回失败。

### 选型建议

新部署一律用**组件版**（`INSTALL COMPONENT 'file://component_validate_password'`）：功能更全、可变更多、是官方现行路线；插件版仅用于"老版本客户端/运维脚本依赖 `validate_password_length` 这类下划线变量名"的过渡场景。

### 与 keyring 双轨的对照（迁移手法分类）

| | validate_password | keyring |
|---|---|---|
| 手法 | **分派**（二选一，运行时探测） | **桥接**（`daemon_keyring_proxy` 把插件包装成组件服务） |
| 原因 | 校验器无状态、无存量数据，可以二选一 | 已加密的数据必须用同一把钥匙解，不能二选一 |
| 并存方式 | 两套代码各活各的 | 插件作为组件的后备实现被调用 |
| 详见 | 本篇 | [`keyring.md`](keyring.md) |

---

## 参考

**源码**

- `plugin/password_validation/validate_password.cc`（插件版：vtable、7 变量、弃用警告）
- `components/validate_password/validate_password_imp.cc`（组件版：2 服务、8 变量、goto 回滚链）
- `sql/auth/password_policy_service.cc`（★ 双轨分派器）
- `include/mysql/plugin_validate_password.h`、`include/mysql/components/services/validate_password.h`（两份契约）
- `sql/sql_plugin_services.h:271`（导出为老版 plugin service）
- `sql/item_func.cc:4210`（`VALIDATE_PASSWORD_STRENGTH()`）

**官方文档**

- *MySQL 8.0 Reference Manual → 6.4.3 [The Password Validation Component](https://dev.mysql.com/doc/refman/8.0/en/validate-password.html)*
- *MySQL 8.0 Reference Manual → [Writing Password-Validation Plugins](https://dev.mysql.com/doc/refman/8.0/en/writing-password-validation-plugins.html)*

**相关文档**

- 插件框架：[`plugin.md`](plugin.md)
- 组件框架与加载链：[`component.md`](component.md)
- 服务契约（含 THD 传递与老版 plugin service 桥接）：[`service.md`](service.md)
- 服务注册表：[`registry.md`](registry.md)
- 另一条双轨（keyring 桥接）：[`keyring.md`](keyring.md)
- 迁移全景与判据：[`architecture.md`](architecture.md)
- ABI 设计（信任模型、调用形态分层）：[`abi.md`](abi.md)
