# 14 计划稳定性与优化器能力横向对照

> 本篇不是 MySQL 的实现剖析，而是一篇**横向对照**——讲清 MySQL 优化器在"计划稳定性、GROUP BY、下推"三个维度上**有什么、缺什么**，以及业界（Oracle / HyPer / Presto）怎么做。这三个维度是 MySQL OLAP 能力偏弱的集中体现；理解"缺什么"比理解"有什么"更能看清它的定位。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、计划回归与 SPM：计划稳定性的两种思路](#一计划回归与-spm计划稳定性的两种思路)
- [二、GROUP BY 优化的三条路线](#二group-by-优化的三条路线)
- [三、下推优化的横向对照](#三下推优化的横向对照)
- [参考](#参考)

---

## 设计思想与理论基础

### 一、计划回归与 SPM：计划稳定性的两种思路

#### 1.1 计划回归（plan regression）

CBO 依赖**统计信息**做决策。当统计刷新、数据量变化、版本升级、参数调整时，优化器可能为同一条 SQL **突然换一个计划**——而新计划**不一定更好**，甚至更差（代价模型失真、统计偏差）。线上一条跑得好好的 SQL 某天突然变慢，就是**计划回归**。

#### 1.2 SPM：SQL Plan Management（Oracle）

SPM（Oracle 11g 引入）是一种**预防性机制**，核心是 **SQL plan baseline（计划基线）**：与一条 SQL 关联的一组**可接受计划**。

关键点：**计划不再由优化器自由生成，而是受基线约束**——

- 优化器只从 baseline 里**选择**，不自由探索
- 新计划要先**验证**，验证通过才能进 baseline

工作流程三段：

| 阶段 | 动作 | 关键点 |
|---|---|---|
| **捕获（capture）** | 把 SQL 的计划纳入 baseline | 自动（`optimizer_capture_sql_plan_baselines`）或从 AWR/游标缓存手动加载 |
| **选择（select）** | 优化器从 baseline 里挑代价最小的 | 只挑 `enabled + accepted` 的计划 |
| **进化（evolve）** | 新计划先"试用" | 验证性能更优才 `accepted`，否则丢弃 |

一个计划的三个状态：`ENABLED`（可考虑）/ `ACCEPTED`（验证过，正式成员）/ `FIXED`（锁死，即使有更优也不换）。

**SPM 与 hints 的本质区别**：

> **hint 是"锁死一个计划"，SPM 是"管理一个计划集合并让它受控进化"。**

hint 静态、一次性、DBA 人肉维护；SPM 自动、持续、允许计划随数据缓慢演进——但演进是**受控**的（先验证再接受）。

#### 1.3 为什么 MySQL 和 Presto 都没有

SPM 需要一整套"计划身份（signature）+ 基线存储 + 进化验证"的基础设施。

| 系统 | 计划管理手段 | 本质 |
|---|---|---|
| **MySQL** | `optimizer hints` + `optimizer_switch` | 手动逃生舱 |
| **Presto/Trino** | session 属性（`join_distribution_type`、`join_reordering_strategy`） | 会话级手动控制 |
| **Oracle** | SPM（plan baseline） | 自动、受控进化 |

MySQL 与 Presto 都停在了"手动"这一档。分布式引擎（Presto）环境更多变、计划更难基线化，普遍不做 SPM。MySQL 则是从未投入这个方向——它选择了更轻的 hints + switch 作为手动逃生舱。

**★★★ "MySQL 没有 SPM" 的源码反证**（★ 这不是推断，是 grep 结论）

在 **8.0.39 全仓**（`sql/` + `include/` + `storage/`）下逐个关键词检索：

| 关键词 | 命中数 | 结论 |
|---|---|---|
| `sql plan management` / `SQL Plan Management` | **0** | 不存在 |
| `plan baseline` | **0** | 不存在 |
| `plan_management` / `sql_plan_management` | **0** | 不存在 |
| `plan_history` | **0** | 不存在 |
| `accepted plan` | **0** | 不存在 |
| `evolve`（SPM 语义） | **0** | 不存在 |
| `baseline` | 3 处，**全部无关**（`rpl_binlog_sender.h:488,498` 缓冲区基线；`item_sum.cc:4858` 注释 `// set baseline`） | 无 SPM 语义 |
| `SPM`（单词） | **仅 1 处且无关**：`sql/sql_lex.h:3874` 注释里阿里云文档 URL 的查询参数 `?spm=a2c4g...` | 不存在 |
| `sql_plan` | 全部是文件名 `sql_planner.cc` / 宏 `SQL_PLANNER_INCLUDED` | 不存在 |

> ★★★ 结论可直接引用：**8.0.39 源码中不存在任何 SPM 基础设施**——没有 plan baseline 存储、没有 plan history、没有 accepted/verified 计划状态机、没有 evolve 验证逻辑。所谓 "SQL Plan Management" 只存在于 Oracle / Aurora PG / PolarDB，**社区 MySQL 没有**。
>
> 所以本篇下面要写的不是"MySQL 的 SPM 怎么实现"，而是**"没有 SPM 时，MySQL 用什么替代"**——答案是三样：**Optimizer Hints**（§1.5）、**`optimizer_switch`**（§1.6）、**统计信息持久化**（§1.7）。

**★★ 为什么"给 MySQL 加 SPM"更难**（★ 上面 1.4 第 1 点的源码侧佐证）

SPM 的前提是"**计划有一个可锚定的身份**"。对比两边的计划表示：

| | Oracle / PG | MySQL 8.0.39 |
|---|---|---|
| 计划的"身份" | plan node 树有稳定序列化 / signature | `POSITION` 数组（**注释明说 `has to stay a POD, because it is memcpy'd in many places`**），且随 join 前缀动态变化 |
| 计划的稳定性 | 一个 SQL 对应若干可枚举 baseline | `best_positions` 每次重新搜索，无"计划 ID"概念 |
| 半连接等策略 | 计划的一部分 | **倒序记录在 `POSITION::sj_strategy`**（注释："must be traversed from right to left"），需 `fix_semijoin_strategies()` 翻转后才可读 |

> ★ 尤其第三条：semi-join 策略在搜索期是**倒序**记的（每个 POSITION 记"到我这里为止用什么策略"），只有最后 `fix_semijoin_strategies()` 从右往左扫一遍才翻转成正序。**这样一个"连自身可读性都要靠事后翻转"的结构，很难做稳定的 signature**。

#### 1.4 云厂商的 SPM：谁在 MySQL 生态里补上了

社区 MySQL 没有 SPM，但云厂商在各自产品里做了。调研结论：

| 产品 | 计划管理 | 说明 |
|---|---|---|
| 社区 MySQL | ❌ 只有 hints + `optimizer_switch` | — |
| AliSQL（阿里云 MySQL 内核） | ❌ 未见独立 SPM | 增强在别处（线程池 / 审计等） |
| Aurora **MySQL** 版 | ❌ 只有 optimizer hints | 与社区一致 |
| **Aurora PostgreSQL** 版 | ✅ `apg_plan_mgmt` 扩展 | 真实 SPM：`capture_plan_baselines` / `Evolve_Plan_Baselines` / `Validate_Plans` |
| **PolarDB MySQL 版 / PolarDB-X** | ✅ 「执行计划管理」 | 阿里云自研，`BASELINE` 指令 + baseline |
| Oracle | ✅ 原生 SPM | 源头 |

**两个值得注意的点**：

1. **Aurora 只在 PG 版做了 SPM，MySQL 版没做**。`apg_plan_mgmt` 仅存在于 Aurora PostgreSQL 版，Aurora MySQL 版仍是 hints。侧面说明"给 MySQL 加 SPM"更难——MySQL 优化器没有 PG 那种清晰的 plan node 序列化机制，计划的"身份"不好锚定。

2. **PolarDB-X 的 SPM 直接借鉴 Oracle，但做了云化适配**：

| 维度 | PolarDB-X SPM | Oracle SPM |
|---|---|---|
| 接口 | `BASELINE ADD/FIX/LOAD/PERSIST/...` 命令 | `DBMS_SPM` 包 |
| 计划键 | **参数化 SQL 文本**（`?` 占位符） | SQL signature |
| 状态 | `FIXED` / `ACCEPTED` | `FIXED` / `ACCEPTED` / `ENABLED` |
| 前置 | 先过 **Plan Cache**，复杂查询才进 SPM | 共享池与 baseline 分离 |
| 分布式 | 计划含 `Gather` / `ParallelHashJoin` / `LogicalView` 算子 | 无 |

核心机制同构：**参数化 → baseline → 多计划选代价最小 → `FIXED` 固定 → 自动演化**。PolarDB-X 的独到之处在于把 **Plan Cache**（参数化 SQL 的计划复用）与 **Plan Management**（基线固化）绑在一起——这比 Oracle 更贴合云上"高并发 + 计划稳定"的需求。

---

### ★★ 1.5 MySQL 的替代方案 A：Optimizer Hints（唯一的官方计划稳定性工具）

既然没有 SPM，**Hints 就是 MySQL 唯一能"钉住计划"的官方手段**。它的基础设施在 `sql/opt_hints.{h,cc}`（798 + 966 行）。

#### 级别体系（四类对象，不是"三级"）

`Opt_hints` 基类的注释是官方定义（`opt_hints.h:145`）：

```cpp
/**
  Opt_hints class is used as ancestor for Opt_hints_global,
  Opt_hints_qb, Opt_hints_table, Opt_hints_key classes.

  Opt_hints_global class is hierarchical structure.
  It contains information about global hints and also
  contains array of QUERY BLOCK level objects (Opt_hints_qb class).
  Each QUERY BLOCK level object contains array of TABLE level hints
  (class Opt_hints_table). Each TABLE level hint contains array of
  KEY lelev hints (Opt_hints_key class).
*/
```

> ⚠️ 枚举真名是 **`opt_hints_enum`**（不存在 `Opt_hints_type`）；键级类真名是 **`Opt_hints_key`**（不存在 `Opt_hints_index`）。另有第 4.5 层 `Compound_key_hint`（复合索引 hint 的容器）。

**★ 提示位图：两张 `Bitmap<64>` 分离**（`opt_hints.h:101`，★ 这是"hint 能覆盖 optimizer_switch"的数据结构基础）：

```cpp
class Opt_hints_map {
  Bitmap<64> hints;            // hint state
  Bitmap<64> hints_specified;  // true if hint is specified
 public:
  bool is_specified(opt_hints_enum type_arg) const {
    return hints_specified.is_set(type_arg);
  }
  void set_switch(opt_hints_enum type_arg, bool switch_state_arg) {
    if (switch_state_arg) hints.set_bit(type_arg);
    else hints.clear_bit(type_arg);
    hints_specified.set_bit(type_arg);   /* ★ 无论取值如何，"已指定"都置位 */
  }
  bool switch_on(opt_hints_enum type_arg) const { return hints.is_set(type_arg); }
};
```

> ★★ **"是否指定"与"取值"分成两张位图**——这正是 hint 能**表达"关闭"（NO_BNL 也占一个 specified 位）**的关键。若只有一张位图，`NO_BNL` 与"没写 BNL"就无法区分。
> ★ `Bitmap<64>` 硬编码 64 位 ⇒ **hint 类型数上限 64**，目前 `MAX_HINT_ENUM = 23`。

**★ hint 元信息表**（`opt_hints.cc:54`，★ 三个布尔列决定了级别继承与打印行为）：

```cpp
struct st_opt_hint_info opt_hint_info[] = {
    {"BKA", true, true, false},                  /* check_upper_lvl, switch_hint, irregular */
    {"BNL", true, true, false},
    {"ICP", true, true, false},
    {"MRR", true, true, false},
    {"NO_RANGE_OPTIMIZATION", true, true, false},
    {"MAX_EXECUTION_TIME", false, false, false},
    {"QB_NAME", false, false, false},
    {"SEMIJOIN", false, false, false},
    {"SUBQUERY", false, false, false},
    {"MERGE", true, true, false},
    {"JOIN_PREFIX", false, false, true},         /* ★ irregular_hint：打印格式特殊 */
    {"JOIN_SUFFIX", false, false, true},
    {"JOIN_ORDER", false, false, true},
    {"JOIN_FIXED_ORDER", false, true, false},
    {"INDEX_MERGE", false, false, false},
    {"RESOURCE_GROUP", false, false, false},
    {"SKIP_SCAN", false, false, false},
    {"HASH_JOIN", true, true, false},
    {"INDEX", false, false, false},
    {"JOIN_INDEX", false, false, false},
    {"GROUP_INDEX", false, false, false},
    {"ORDER_INDEX", false, false, false},
    {"DERIVED_CONDITION_PUSHDOWN", true, true, false},
    {nullptr, false, false, false}};
```

> ★ `check_upper_lvl=true` 的 8 个 hint 允许"表级写、QB 级兜底"；`irregular_hint` 只给三个 join order hint（打印格式特殊）。
> ★ **`NO_` 前缀不是独立项**——由 `append_hint_type()` 在打印时按 switch 位动态加。

#### ★★★ "hint 覆盖 optimizer_switch" 的唯一合并点：`get_hint_state()`

```cpp
/* sql/opt_hints.cc:845 */
static bool get_hint_state(Opt_hints *hint, Opt_hints *parent_hint,
                           opt_hints_enum type_arg, bool *ret_val) {
  if (opt_hint_info[type_arg].switch_hint) {
    if (hint && hint->is_specified(type_arg)) {          /* ① 本级指定 */
      *ret_val = hint->get_switch(type_arg);
      return true;
    }
    if (opt_hint_info[type_arg].check_upper_lvl &&
        parent_hint->is_specified(type_arg)) {           /* ② 上级指定（仅当 check_upper_lvl） */
      *ret_val = parent_hint->get_switch(type_arg);
      return true;
    }
  } else {
    if (hint && hint->is_specified(type_arg) &&
        /* This check is necessary because function idx_merge_key_enabled()
           can operate only with resolved hint. */
        hint->is_resolved(type_arg)) {                   /* ★ 复杂 hint 还要求"已解析" */
      *ret_val = hint->get_switch(type_arg);
      return true;
    }
  }
  return false;
}

bool hint_table_state(const THD *thd, const Table_ref *table_list,
                      opt_hints_enum type_arg, uint optimizer_switch) {
  if (table_list->opt_hints_qb) {
    bool ret_val = false;
    if (get_hint_state(table_list->opt_hints_table, table_list->opt_hints_qb,
                       type_arg, &ret_val))
      return ret_val;
  }
  return thd->optimizer_switch_flag(optimizer_switch);   /* ★★★ 唯一回退出口 */
}
```

> ★★★ **优先级链**：键/表级指定 → QB 级指定（仅当 `check_upper_lvl`）→ **`optimizer_switch` 标志**。最后一行 `return thd->optimizer_switch_flag(...)` 就是"hint 未指定时回退到 switch"的唯一出口。
> ⚠️ `hint_table_state` **声明在 `opt_hints.h:749`、定义在 `opt_hints.cc:908`**，不在 `sql_optimizer.cc`。

**★ `hint_table_state` 的全部调用点 = "被 hint 影响的优化器决策清单"**：

| 文件:行 | 用途 |
|---|---|
| `sql_optimizer.cc:3458/3460` | `setup_join_buffering()`：BNL / BKA |
| `sql_planner.cc:1006` | greedy 搜索中禁用 join buffer（BNL） |
| `handler.cc:6975` | `DsMrr_impl::choose_mrr_impl()`：DS-MRR（走键级 `hint_key_state`） |
| `sql_derived.cc:1030` | `DERIVED_CONDITION_PUSHDOWN` |
| `sql_resolver.cc:3501` | `MERGE`（derived merge） |
| `sql_optimizer.cc:8173` + `range_optimizer.cc:690/751/773` | `SKIP_SCAN` |
| `range_optimizer.cc:1057` + `opt_hints.cc:953` | `INDEX_MERGE` |

#### ★★ `SET_VAR` hint：语句级改会话变量

这是**"hint 之外最强的计划干预"**——它能改任何 `HINT_UPDATEABLE` 的会话变量。

```cpp
/* sql/opt_hints.h:667 —— 用 Item* 保存旧值 */
class Hint_set_var {
 public:
  Hint_set_var(set_var *var_arg) : var(var_arg), save_value(nullptr) {}
  set_var *var;      // Pointer to set_var object
  Item *save_value;  // Original variable value
};
```

```cpp
/* sql/opt_hints.cc:781 —— 改 → 还原 → 再改回来（供 PS 重执行） */
void Sys_var_hint::restore_vars(THD *thd) {
  for (uint i = 0; i < var_list.size(); i++) {
    Hint_set_var *hint_var = var_list[i];
    set_var *var = hint_var->var;
    if (hint_var->save_value) {
      /* Restore original value for update */
      std::swap(var->value, hint_var->save_value);
      (void)var->check(thd);
      (void)var->update(thd);
      /* Restore hint value for further executions */
      std::swap(var->value, hint_var->save_value);   /* ★★ 再换回来 */
    }
  }
}
```

> ★★ **两次 `std::swap`** 是刻意的：先换出旧值写回（还原），再换回 hint 值（供 prepared statement 下次执行继续用）。这是"语句级作用域 + PS 可重执行"两个需求的妥协产物。

**★ 三个生效/还原时机**：

```cpp
/* ① 普通语句：sql_parse.cc:3350（ACL 检查之前） */
  if (lex->opt_hints_global && lex->opt_hints_global->sys_var_hint)
    lex->opt_hints_global->sys_var_hint->update_vars(thd);
/* sql_parse.cc:4881 —— finish: 标签处，正常/错误路径都会走到 */
    lex->opt_hints_global->sys_var_hint->restore_vars(thd);
/* ② 预备语句 PREPARE 阶段：sql_prepare.cc:2583 */
    m_lex->opt_hints_global->sys_var_hint->update_vars(thd);
    error = prepare_query(thd);       /* ★ prepare 期的优化器决策也受变量影响 */
    m_lex->opt_hints_global->sys_var_hint->restore_vars(thd);
```

> ⚠️ **`update_vars()` 第一行是 `if (thd->slave_thread) return;`** —— **从库上 `SET_VAR` 直接失效**（这是主从行为不一致的一个隐藏源）。
> ⚠️ 只有注册时带 `HINT_UPDATEABLE` 的变量可用（`is_hint_updateable()` 白名单），否则报 `ER_NOT_HINT_UPDATABLE_VARIABLE`。
> ⚠️ `PT_hint_sys_var` 用 **`MAX_HINT_ENUM`（哨兵值）** 作为 hint_type——它不占正式槽位，不参与位图，走完全独立的路径。

#### ★★★ JOIN_ORDER / JOIN_PREFIX / JOIN_SUFFIX：直接固定 join order

这是**最接近 SPM 效果**的 hint——它不写死计划，而是**注入依赖位图**。

```cpp
/* sql/opt_hints.cc:340 —— 三种语义的数学本质 */
static table_map get_other_dep(opt_hints_enum type, table_map hint_tab_map,
                               table_map table_map) {
  switch (type) {
    case JOIN_PREFIX_HINT_ENUM:
      if (hint_tab_map & table_map) return 0;        /* hint 表：无额外依赖 */
      return hint_tab_map;                           /* ★ 非 hint 表：必须依赖所有 hint 表 */
    case JOIN_SUFFIX_HINT_ENUM:
      if (hint_tab_map & table_map) return ~hint_tab_map;  /* ★ hint 表：必须依赖所有非 hint 表 */
      return 0;
    case JOIN_ORDER_HINT_ENUM:
      return 0;                                      /* ORDER：只给 hint 表之间串链 */
    default: assert(0); break;
  }
  return 0;
}
```

```cpp
/* sql/opt_hints.cc:447 —— 主逻辑 */
static bool set_join_hint_deps(JOIN *join,
                               const Hint_param_table_list *hint_table_list,
                               opt_hints_enum type) {
  Join_order_hint_handler hint_handler(join);   /* ★ RAII：失败时还原 dependent */
  table_map hint_tab_map = 0;
  if (hint_handler.init()) return true;

  for (const Hint_param_table *hint_table = hint_table_list->begin(); ...) {
    for (uint i = 0; i < join->tables; i++) {
      const Table_ref *table = join->join_tab[i].table_ref;
      if (!compare_table_name(hint_table, table)) {
        /* Const tables are excluded ... since they are always first */
        if (join->const_table_map & table->map()) break;

        JOIN_TAB *tab = &join->join_tab[i];
        tab->dependent |= hint_tab_map;              /* ★★ 注入前驱依赖 */
        update_nested_join_deps(join, tab, hint_tab_map);
        hint_tab_map |= tab->table_ref->map();
        break;
      }
    }
  }

  for (uint i = 0; i < join->tables; i++) {          /* 非 hint 表的依赖 */
    JOIN_TAB *tab = &join->join_tab[i];
    table_map dependent_tables =
        get_other_dep(type, hint_tab_map, tab->table_ref->map());
    tab->dependent |= dependent_tables;
  }

  if (join->propagate_dependencies()) return true;   /* ★★ 传递闭包；成环 → 失败 */
  hint_handler.no_restore_deps();                    /* 成功：放弃还原 */
  return false;
}
```

> ★★★ **机制全貌**：join order hint **不是"写死顺序"**，而是往 `JOIN_TAB::dependent` 注入前驱依赖 → `propagate_dependencies()` 做传递闭包 → join order 搜索自然只能产出满足该偏序的计划。
> ★★ **失败是静默的**：闭包成环时 `propagate_dependencies()` 返回 true，RAII 析构函数把 `dependent` 全部还原，**hint 被忽略但不报错**——只置 `join_order_hints_ignored` 位（EXPLAIN 里不再回显该 hint）。排查"hint 为什么没生效"时这是第一嫌疑。
> ★ `const table` 被显式排除（它们天然排最前）。
> ★ 调用点 `sql_optimizer.cc:5373`，且**前置 `!(active_options() & SELECT_STRAIGHT_JOIN)`**——**STRAIGHT_JOIN 优先于一切 join order hint**。

#### ★ hint 的三个生效阶段（时序）

| 阶段 | 位置 | 做什么 |
|---|---|---|
| **① PARSING** | `parse_tree_hints.cc:53` `get_qb_hints()` / `get_table_hints()` | 建 QB/表/键三级对象骨架（**此时表名未与真实 TABLE 绑定**） |
| **② PREPARE（setup_tables）** | `sql_resolver.cc:1195` | `adjust_table_hints()` 绑定真实 TABLE/KEY → `update_index_hint_maps()` 改写索引可用性 → `check_unresolved()` 发警告 |
| **③ OPTIMIZE** | `sql_optimizer.cc:5373` | `apply_join_order_hints()` 注入依赖 |

```cpp
/* sql/sql_resolver.cc:1195 —— ② 的原貌 */
    if (opt_hints_qb && !tr->opt_hints_table) {
      tr->opt_hints_table = opt_hints_qb->adjust_table_hints(tr);
    }
    if (!tr->opt_hints_table ||
        // Ignore old index hint processing if new style hints are specified.
        !tr->opt_hints_table->update_index_hint_maps(thd, tr->table)) {
      if (tr->process_index_hints(thd, table)) return true;
    }
```

**★ 冲突裁决**（`PT_table_level_hint::contextualize`，★ 表级写时第三参 `check_parent=true`）：

```cpp
    if (tab->set_switch(switch_on(), type(), true))     /* ★ true = 先看 QB 级是否写过 */
      print_warn(pc->thd, ER_WARN_CONFLICTING_HINT, ...);
```

> ★ 而 `JOIN_ORDER` 是**例外**——它 `no_warn = true`，允许同时存在多个（`opt_hints.cc:283` 注释：*"Don't print 'conflicting hint' warning since it could be several JOIN_ORDER hints at the same time"*）。
> ★ `JOIN_FIXED_ORDER` 不进 `apply_join_order_hints()`——它在解析期就被翻译成 `SELECT_STRAIGHT_JOIN`（`pc->select->add_base_options(SELECT_STRAIGHT_JOIN)`）。
> ★ `INDEX_MERGE` 只带 1 个索引是**非法** hint（`ER_WARN_INVALID_HINT`）——语义上至少要两个索引。
> ★ **只有索引类 hint 能写进视图**（`PT_hint::supports_view()` 用枚举区间 `INDEX_HINT_ENUM..ORDER_INDEX_HINT_ENUM` 白名单），其余在 `CREATE VIEW` 时被静默丢弃。

### ★ 1.6 替代方案 B：`optimizer_switch` 与 `use_best_so_far`

**★ `use_best_so_far` 不是 MySQL 自己的机制**——它是**二级引擎的回调出参**：

```cpp
/* sql/handler.h:2365 */
typedef bool (*compare_secondary_engine_cost_t)(
    THD *thd, const JOIN &join, double initial_cost, bool *use_best_so_far,
    double *best_cost, double *secondary_engine_cost);
```

> ★ MySQL 自身实现中该标志**恒为 false**；只有 HeatWave 这类二级引擎才会置位说"这个计划够好了，别再搜了"。所以它是**"提前终止搜索"的钩子**，不是计划稳定性机制。

**★ `optimizer_switch` 的 flags 全集**（`sql/sql_const.h`）——⚠️ 真实消费点是 `sys_vars.cc:3447` 的 `optimizer_switch_names[]`（**不存在** `optimizer_switch_typelib`，只在 `sql_const.h:190` 的注释里被提到）。源码自己加了警告：

```cpp
/**
  @note
  @b BEWARE! These must have the same order as the \#defines in sql_const.h!
*/
```

> ★ 名字数组与位定义的**顺序耦合**是长期隐患，源码自己标了 `BEWARE!`。

### ★★ 1.7 替代方案 C：统计信息持久化——"计划不稳定"的物理根源

**★ `ha_statistics::records` 的注释就是答案**（`sql/handler.h:3892`）：

```cpp
  /*
    The number of records in the table.
      0    - means the table has exactly 0 rows
    other  - if (table_flags() & HA_STATS_RECORDS_IS_EXACT)
               the value is the exact number of records in the table
             else
               it is an estimate
  */
  ha_rows records;
```

> ★★★ **InnoDB 不声明 `HA_STATS_RECORDS_IS_EXACT`** ⇒ `stats.records` **永远是采样估计值**。这就是 MySQL 计划漂移的物理根源。

**★ `rec_per_key` 走"双轨"**（`sql/key.h:93`，★ 新旧两套数组并存）：

```cpp
typedef float rec_per_key_t;
#define REC_PER_KEY_UNKNOWN -1.0f      /* 新数组：浮点，-1.0f 表示未知 */
#define IN_MEMORY_ESTIMATE_UNKNOWN -1.0

  /** Array of AVG(number of records with the same field value) ...
      0 means 'not known'. */
  ulong *rec_per_key{nullptr};         /* 老数组：整数，0 表示未知 */
  rec_per_key_t *rec_per_key_float{nullptr};  /* 新数组，优先使用 */

  rec_per_key_t records_per_key(uint key_part_no) const {
    if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)
      return rec_per_key_float[key_part_no];       /* ★ 新数组优先 */
    return (rec_per_key[key_part_no] != 0)
               ? static_cast<rec_per_key_t>(rec_per_key[key_part_no])
               : REC_PER_KEY_UNKNOWN;
  }
```

> ★ 源码 `@todo` 明说："*In the next release the rec_per_key array above should be removed and only this should be used*"——**双轨是迁移期的中间状态**。

**★ `records_in_range` 的兜底实现直接返回常量 10**（`sql/handler.h:5515`）：

```cpp
  virtual ha_rows records_in_range(uint inx, key_range *min_key,
                                   key_range *max_key) {
    return (ha_rows)10;
  }
```

> ★ 真实值由引擎覆盖（InnoDB 走 `btr_estimate_n_rows_in_range` 采样）。**`rec_per_key` 与 `records_in_range` 两个估计共同构成 range 优化器的代价输入——两者都是采样值，都会漂移。**

**★★★ `mysql.innodb_table_stats` / `innodb_index_stats` 不是 SQL 建的表**

这是最容易踩的一处认知错误。⚠️ **`scripts/mysql_system_tables.sql` 里搜不到它们（0 命中）**——它们在 **`ha_innodb.cc:12956` 用 C++ 现场拼装** `dd::Object_table_definition`：

```cpp
  def = innodb_table_stats->target_table_definition();
  def->set_table_name("innodb_table_stats");
  def->add_field(0, "database_name", db_field.c_str());
  def->add_field(1, "table_name", table_field.c_str());
  def->add_field(2, "last_update", "... TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP");
  def->add_field(3, "n_rows", "n_rows BIGINT UNSIGNED NOT NULL");
  def->add_field(4, "clustered_index_size", "clustered_index_size BIGINT UNSIGNED NOT NULL");
  def->add_field(5, "sum_of_other_index_sizes", "sum_of_other_index_sizes BIGINT UNSIGNED NOT NULL");
  def->add_index(0, "index_pk", "PRIMARY KEY (database_name, table_name)");
```

```cpp
  def = innodb_index_stats->target_table_definition();
  def->set_table_name("innodb_index_stats");
  ...
  def->add_field(4, "stat_name", "stat_name VARCHAR(64) NOT NULL");
  def->add_field(5, "stat_value", "stat_value BIGINT UNSIGNED NOT NULL");
  def->add_field(6, "sample_size", "sample_size BIGINT UNSIGNED");   /* ★★ 采样页数就在这里 */
  def->add_field(7, "stat_description", "stat_description VARCHAR(1024) NOT NULL");
```

> ★★ `sample_size` 列的存在本身就是"统计是采样"的证据——**同一个表两次 ANALYZE 可能算出不同的 `n_diff_pfx%`，这就是计划漂移的直接来源**。

**★ 统计什么时候被重算**（`ha_innodb.cc:17231`，★ 三级回退：表标志 → 全局开关）：

```cpp
      if (dict_stats_is_persistent_enabled(ib_table)) {
        if (is_analyze) opt = DICT_STATS_RECALC_PERSISTENT;
        else /* This is e.g. 'SHOW INDEXES', fetch the persistent stats from disk. */
             opt = DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY;
      } else {
        opt = DICT_STATS_RECALC_TRANSIENT;
      }
      ret = dict_stats_update(ib_table, opt);
```

```cpp
/* storage/innobase/include/dict0stats.ic:76 */
static inline bool dict_stats_is_persistent_enabled(const dict_table_t *table) {
  uint32_t stat_persistent = table->stat_persistent;
  if (stat_persistent & DICT_STATS_PERSISTENT_ON)  return true;   /* ① 表级显式 ON */
  else if (stat_persistent & DICT_STATS_PERSISTENT_OFF) return false;  /* ② 表级显式 OFF */
  else return (srv_stats_persistent);                             /* ③ 回退全局开关 */
}
```

> ★ `innodb_stats_persistent` **默认 `true`**（`ha_innodb.cc:22210`）——8.0 起持久化统计默认开启。

**★ `fetch_number_of_rows()` 的一个关键差异**（`sql/table.cc:6546`，★ 同一个表在两个优化器下可能得到完全不同的估计）：

```cpp
  @param fallback_estimate A fallback row estimate to use if the storage engine
  doesn't provide one for us. The old optimizer uses
  PLACEHOLDER_TABLE_ROW_ESTIMATE, which is 2. The hypergraph optimizer uses a
  more pessimistic estimate of 1000 rows.
```

> ★ 兜底值**老优化器 2、hypergraph 1000**——相差 500 倍。同一条 SQL 在两个优化器下的代价估算可能天差地别。

### 二、GROUP BY 优化的三条路线

`JOIN + GROUP BY` 的朴素执行是**先 join 出全部中间结果，再 group by**。当 join 产生海量中间行、而聚合后结果很小（如"每个分组取前几名"）时，中间结果膨胀成瓶颈。三种系统给出了三种解法：

| | GROUP BY 优化思路 | 系统定位 |
|---|---|---|
| **MySQL** | 单库内部：loose index scan + 临时表/filesort | 单机 OLTP |
| **HyPer** | **groupjoin** 融合算子（grouping 融合进 join） | 内存 HTAP |
| **Presto/Trino** | **aggregation pushdown**（下推数据源）+ partial/final 分布式聚合 | 分布式联邦查询 |

**HyPer 的 groupjoin**（论文《Accelerating Queries with Group-By and Join By Groupjoin》）：把 grouping **融合进 join**，在 join 过程中就做聚合，避免物化完整中间结果。前提是分组键与 join 键有特定关系、且 join 不会"复制"分组。本质是 **group-by pushdown** 的激进形式。

**Presto 的 aggregation pushdown**：把整个 `GROUP BY` 下推到数据源执行（EXPLAIN 里不出现 `Aggregate` 算子）；下推失败则退化为两阶段分布式聚合：

```
Aggregate(PARTIAL)   -- 各节点局部聚合
      ↓ shuffle
Aggregate(FINAL)     -- 全局聚合
```

复杂 grouping（`ROLLUP` / `CUBE` / `GROUPING SETS`）不下推，本地执行。

**三种思路没有谁对谁错，而是被系统定位逼出来的**：

- MySQL 是单机，只能在内部优化，所以走索引/临时表
- HyPer 是内存库、追求极致，所以敢做 groupjoin 这种激进融合
- Presto 是联邦引擎，数据在别人那里，所以第一反应是"下推"，下推不了才自己分布式算

**MySQL 的代价**：它的 GROUP BY 一定在 join **之后**做（先 join 再 `AggregateIterator` 聚合），唯一的 GROUP BY 优化是**单表**的 loose index scan（见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)）。所以面对 `JOIN + GROUP BY` 时中间结果膨胀无法避免。

### 三、下推优化的横向对照

#### 3.1 MySQL 的下推清单

MySQL 是单机，它的"下推"不是推到别的数据源，而是**从 SQL 层推到自己的存储引擎层（handler）**。全景如下：

| 下推优化 | 作用 | 系统讲解 |
|---|---|---|
| **ICP**（Index Condition Pushdown） | 把 WHERE 条件下推到引擎，在索引遍历时就过滤（减少回表） | [`10_plan_refinement.md`](10_plan_refinement.md)、[`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) |
| **MRR**（Multi-Range Read） | 批量化随机 IO（rowid 排序后顺序读） | [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)、[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **derived_condition_pushdown** | 把外层条件推进派生表（物化前过滤） | [`logical/04_logical_join.md`](logical/04_logical_join.md)、[`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) |
| **index_merge** | 多索引取交集/并集 | [`physical/07_access_method.md`](physical/07_access_method.md)、[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **skip scan**（8.0 新增） | 联合索引跳首列扫描 | [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| **BKA**（Batched Key Access） | 批量 key 做 MRR 索引查找 | [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md) |
| **push_to_engines** | 把 join/条件整体下推给引擎（如 NDB 集群） | [`10_plan_refinement.md`](10_plan_refinement.md) |

#### 3.2 Presto/Trino 的 pushdown 体系

Presto 的"下推"是**把算子推到别的数据源**（MySQL、PG、Hive 等），方向完全不同：

| pushdown 类型 | 作用 |
|---|---|
| **Predicate** | 下推 WHERE 过滤 |
| **Projection** | 只读需要的列 |
| **Dereference** | 只读 ROW 类型的指定字段 |
| **Aggregation** | 下推 GROUP BY / 聚合 |
| **Join** | 把多表 join 委托给数据源 |
| **Limit / Top-N** | 下推 LIMIT / ORDER BY LIMIT |

成功下推时，EXPLAIN 里对应的算子（`Filter` / `Aggregate` / `Join` / `TopN`）**消失**，只剩 `TableScan`。

#### 3.3 差异的本质

| | MySQL | Presto |
|---|---|---|
| 下推方向 | SQL 层 → **自己的存储引擎层**（handler） | 引擎 → **别的数据源**（connector） |
| 收益 | 减少回表、减少随机 IO | 减少网络传输、借力底层数据库的优化 |
| 粒度 | 条件、key 批处理、join | 条件、投影、聚合、join、limit 全套 |

> MySQL 的"下推"是单机内部的纵深优化；Presto 的"下推"是联邦架构的水平借力。两者名字都叫 pushdown，解决的问题和方向完全不同。

---

## 四、国产内核优化器对照

> 这一节把视线从 MySQL 本体转向**国产数据库内核**。写它的动机：对内核研发这个定位，"社区版 MySQL 缺了什么、国产内核补了什么"是最直接的横向参照——尤其是面试与实际工作中经常被问到。★ 特性细节以各产品官方文档为准，这里只列**可确认的、与社区 MySQL 的差异**，不展开未经证实的细节。

| 内核 | 底层 | 优化器/执行器相对社区 MySQL 的差异 |
|---|---|---|
| **PolarDB MySQL** | MySQL 8.0 | ★ **分布式并行查询（Parallel Query，PQ）**：单条 SQL 的扫描/join 并行化，利用多核提升 CPU/IO 利用率。这是社区 MySQL 没有的——社区版只有 InnoDB 并行读与 DDL 并行，**没有查询级并行** |
| **OceanBase** | 自研 | 优化器 = 查询改写 + **动态规划（DP）+ Skyline 剪枝**；支持**并行执行（DOP）**。对比 MySQL 的 greedy 搜索，OB 用更完整的 DP 框架 + 帕累托剪枝 |
| **TiDB** | 自研（Go） | ★ **Cascades 优化器**（4.0+ 引入）：规则 + 代价搜索的 Cascades 框架。这正是 MySQL 经典优化器**没有**采用、hypergraph 只是部分借鉴的框架（见 [`17_optimizer_decisions.md`](17_optimizer_decisions.md) 理论基础） |
| **openGauss** | PostgreSQL | 基于 PG 内核，优化器在 PG 基础上增强（列存、向量化）。与 MySQL 家族差异最大，属于 PG 血统 |

### 对"内核研发"的三点启示

1. **"并行执行"是国产内核的普遍补强点**：PolarDB、OB 都做了查询级并行，而社区 MySQL 至今没有。这说明"并行查询"是从社区 MySQL 出发做内核研发时，最被反复触碰的空白之一（可对照本目录各篇「社区边界澄清」里反复出现的"社区版没有 X"）。

2. **优化器框架的分野**：MySQL（greedy / hypergraph）vs TiDB（Cascades）vs OB（DP + Skyline）。理解这四种框架，比死记某个内核的 API 更有价值——它们对应的是[`17_optimizer_decisions.md`](17_optimizer_decisions.md) 里"规则 vs 代价"这条主轴的不同取舍。

3. **社区边界澄清的价值**：本目录每一篇末尾的「社区边界澄清」之所以重要，正是因为**国产内核/云厂商的能力常常被误当成社区 MySQL 的能力**（如 HeatWave、并行查询、向量化执行）。读任何一篇前，先确认"这是社区版行为，还是某个内核/云的能力"。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| MySQL 有 SPM / plan baseline | **完全不存在**。9 个关键词全仓 0 命中（`sql plan management`、`plan baseline`、`plan_history`、`accepted plan`、`evolve` 等） |
| hint 级别枚举叫 `Opt_hints_type` | **不存在** → **`enum opt_hints_enum`**（`opt_hints.h:64`，`MAX_HINT_ENUM = 23`） |
| 键级 hint 类叫 `Opt_hints_index` | **不存在** → **`Opt_hints_key`**（`opt_hints.h:641`） |
| 存在 `sql/opt_hints_base.h` | **不存在**（全仓 0 命中）。只有 `opt_hints.h` + `opt_hints.cc` |
| `hint_table_state()` 在 `sql_optimizer.cc` | 声明 `opt_hints.h:749`，**定义 `opt_hints.cc:908`** |
| `SET_VAR` 的 PT 节点叫 `PT_hint_set_var` | **不存在** → **`PT_hint_sys_var`**；运行时容器 **`Sys_var_hint`**，元素 **`Hint_set_var`** |
| `mysql.innodb_table_stats` 建表在 `mysql_system_tables.sql` | **不在**（0 命中）→ **`ha_innodb.cc:12956`** 用 C++ `dd::Object_table_definition` 现场拼装 |
| `optimizer_switch_typelib` | **不存在**（只在 `sql_const.h:190` 注释里被提到）→ 真身 `sys_vars.cc:3447` 的 `optimizer_switch_names[]` |
| `use_best_so_far` 是 MySQL 自己的机制 | 是**二级引擎回调** `compare_secondary_engine_cost_t` 的出参（`handler.h:2365`）；MySQL 自身恒为 false |
| `fetch_number_of_rows()` 在 `TABLE` 上 | 真身在 **`Table_ref`** 上（`sql/table.cc:6546`） |
| JOIN_ORDER hint 在 `sql_planner.cc` 实现 | 实现在 **`opt_hints.cc:459-527`**，调用点 `sql_optimizer.cc:5373` |
| hint 有三级（global/qb/table） | **四类**：`Opt_hints_global` / `Opt_hints_qb` / `Opt_hints_table` / `Opt_hints_key`，另有第 4.5 层 **`Compound_key_hint`** |
| `Opt_hints_map` 一张位图 | **两张** `Bitmap<64>`：`hints`（取值）+ `hints_specified`（是否指定） |
| `JOIN_ORDER` 与其他 join order hint 一样会报冲突 | **例外**：`no_warn = true`，允许同时存在多个 |
| `JOIN_FIXED_ORDER` 在 optimize 期处理 | **不是**——解析期就翻译成 `SELECT_STRAIGHT_JOIN` |
| 所有 hint 都能写进视图 | **只有索引类**（`INDEX_HINT_ENUM..ORDER_INDEX_HINT_ENUM` 白名单），其余静默丢弃 |
| `SET_VAR` 在从库也生效 | **不生效**——`update_vars()` 第一行 `if (thd->slave_thread) return;` |

**★ 四个"结构性事实"**

1. **"MySQL 没有 SPM"是可以被 grep 证明的**。14 个关键词全仓 0 命中，这是本篇最硬的一条结论。因此本篇的立场不是"MySQL 的 SPM 怎么实现"，而是"**没有 SPM 时用什么替代**"——答案是 hints + `optimizer_switch` + 统计持久化三件套。而"给 MySQL 加 SPM 更难"的源码侧佐证是：**`POSITION` 被注释要求 `has to stay a POD, because it is memcpy'd in many places`，且 semi-join 策略在其中是倒序记录的**（需 `fix_semijoin_strategies()` 翻转）——**连自身可读性都要靠事后翻转的结构，很难做稳定的计划 signature**。

2. **hint 覆盖 `optimizer_switch` 的机制，全部浓缩在 `get_hint_state()` 一个函数里**。优先级链是"本级指定 → 上级指定（仅当 `check_upper_lvl`）→ `optimizer_switch` 标志"，而最后一行 `return thd->optimizer_switch_flag(...)` 是唯一的回退出口。数据结构基础是 **`Opt_hints_map` 的两张分离位图**——正因为"是否指定"与"取值"解耦，`NO_BNL` 才能与"没写 BNL"区分开。

3. **join order hint 不是"写死顺序"，而是"注入依赖位图"**。`set_join_hint_deps()` 往 `JOIN_TAB::dependent` 注入前驱依赖，再由 `propagate_dependencies()` 做传递闭包——搜索自然只能产出满足该偏序的计划。而**失败是静默的**：成环时 RAII 还原所有 `dependent`，只置 `join_order_hints_ignored` 位（EXPLAIN 不再回显）**但不报错**。这是"hint 写了却没生效"的第一排查点。

4. **计划不稳定的物理根源是"两个采样估计"**：`stats.records`（InnoDB 不声明 `HA_STATS_RECORDS_IS_EXACT`，永远是采样值）与 `records_in_range`（handler 兜底实现直接返回常量 10，InnoDB 走 `btr_estimate_n_rows_in_range` 采样）。二者共同构成代价输入。而 `mysql.innodb_index_stats.sample_size` 列的存在本身就是证据。**同一个表两次 ANALYZE 可能算出不同的 `n_diff_pfx%`，这就是计划漂移的直接来源**。

> 注：`sql/opt_hints.cc`、`sql/handler.h` 等文件中的部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**理论 / 论文**
- *SQL Plan Management*（Oracle 11g，plan baseline 机制）
- Gubner, Leis, et al. *Accelerating Queries with Group-By and Join By Groupjoin*（HyPer 的 groupjoin 算子）
- *Efficient generation of query plans containing group-by, join...*（grouping 与 join 的代数重排序）

**官方文档**
- Trino 官方 *Pushdown*（predicate / projection / aggregation / join / limit pushdown）

**相关文档**
- MySQL 计划管理替代方案：[`11_optimizer_hints.md`](11_optimizer_hints.md)
- GROUP BY 单表优化（loose index scan）：[`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
- 各下推优化：见「三、下推优化的横向对照」表内链接
