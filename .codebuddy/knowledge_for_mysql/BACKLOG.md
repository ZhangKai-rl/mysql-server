# 知识库深度补全 Backlog

> 扫描时间：**2026-10-14（本次全量审计重排）** · 审计范围：`.codebuddy/knowledge_for_mysql/` 全库 · 达标线：**代码占比 ≥ 45%**
> 「代码占比」= 围栏内代码行数 / 总行数，是本库既定的深度硬指标（见 `feedback_depth_metric.md`）。
> ⛔ **排除**：`cloud/` 与 `papers/` 不纳入（2026-10-08 用户裁定，见文末）。

## 一、总览

| 指标 | 值 |
|---|---|
| 文档总数 | **193**（已剔除 cloud/papers 4 篇） |
| 达标（≥45%） | **134** |
| 不达标 | **59**（30.6%） |
| 全库总代码行 | **183525 / 343036（53.5%）** |

> ⚠️ **2026-10-14 全量审计（本次）**：上一版为「197 篇 / 129 达标 / 68 不达标 / 178494 / 338916 / 52.7%」。
> **篇数口径有 bug**：197 是把 `cloud/`+`papers/` 这 4 篇**又算了回来**——排除判断写的是
> `any(x in r for x in ('cloud/','papers/'))`，而 `os.walk` 给出的根是 `./cloud`（**无尾斜杠**），匹配失败。
> 本次按同一口径重扫：真实 **193 篇 / 129 达标 / 64 不达标 / 178004 / 335773 / 53.0%**（64 = 68 − 那 4 篇）。**以本表为准。**
>
> 历史漂移记录（已被本次覆盖，仅留档）：10-05「65/127」→ 10-08「192 篇」→ 10-13「197 篇 / 123 达标 /
> 74 不达标」→ 同日六次递增至「129 达标 / 68 不达标」。
>
> 📌 **2026-10-14 行格式改版（用户要求瘦身）**：
> ① 删掉每行末尾的「备注 / 【补源码 X%→Y%：…】」长说明——最宽的一行曾达 **19 KB**，
> 这些内容与各篇正文的 `★★★ 真名勘误` / `★ 机制纠错` 章节**完全重复**，不在此处二次维护；
> ② 每行的**行数 / 代码 / 占比 / 优先级全部刷成 2026-10-14 实测值**（此前有 17 行存在 1~23 行漂移）；
> ③ 行宽上限 ~180 字符，BACKLOG 从 671 KB 降到 38 KB。
> 后续补完一篇，只改这一行：`| 新档位 | \`路径\` | 行 | 码 | 旧% → **新%** ✅ |`。

| 优先级 | 篇数 | 含义（2026-10-14 重排口径） |
|---|---|---|
| **P0** | **2** | ① `server/query` 主线未达标者（链路完整性优先，**不看占比**；当前主线已**全数达标**）；② 占比 **<25%**（近乎无代码） |
| **P1** | **23** | 25% ≤ 占比 < 32% |
| **P2** | **22** | 32% ≤ 占比 < 40% |
| **P3** | **10** | 40% ≤ 占比 < 45%（缺口最小，+8~+147 行即可封口） |
| ⛔ 豁免 | **2** | 综述 / 编年，无一一对应的源码（见文末） |
| 合计 | **59** | 与「不达标 59」一致 |

> 旧档位「P0 9 / P1 21 / P2 50 / P3 34 = 114」是 2026-10-05 口径，且**从未与不达标数对齐**。
> 本次重排后合计 59（10-14 当天又补完 `index/transaction.md`（P1−1）、`06_join_order.md`（P0−1）、
> `infra/threading/threading_architecture.md`（P1−1）、`server/replication/binlog.md`（P2−1）、
> `infra/observability/pfs/pfs.md`（P2−1））；逐篇档位见文末快照表的「新档位」列。

## 二、补全顺序建议（2026-10-14 重排后）

```
本轮（10-05 ~ 10-14）已补 10 篇：statistics / system_views / sdi / lob / 06_rollup / dd / 07_temptable /
gipk / transaction / undo_log / 06_join_order
→ server/query 主线 46 篇**全部达标**（剩下 2 篇不达标是豁免的综述/编年），主线链路已闭环。

P0（2）先做
  ├─ server/infra/reading_guide.md                       18.9%  +223  ← 全库「近乎无代码」第一名
  └─ feat/ddl/online_ddl_tools.md                        24.3%  +120

P1（23）先啃「体量大 × 占比低」的 4 篇（补 1 篇 ≈ 补 4~5 篇小篇）
  ├─ infra/threading/thread_pool_mariadb_percona.md      35.7%  +360  ← 当前缺口最大
  ├─ infra/observability/show_commands.md                28.3%  +258
  └─ infra/threading/stack_depth.md                      29.5%  +227
  （其余 19 篇按占比升序逐个推进）

P2（23）→ P3（10）→ 收尾
  └─ P3 里 general_log +24 / compression +8 / innodb_monitor +41 / thr_lock +52 / sys_schema +55
     共 5 篇只需几十行即可封口，可批量收尾
```

> ✅ = 10-05~10-14 本轮已补（04_parser / 03_grammar_notation / 15_groupby / 14_sql_digest / 02_lexer /
> statistics / system_views / sdi / lob / 06_rollup / dd / 07_temptable / gipk）。
> 豁免从上轮的 3 篇**减为 2 篇**：`07_optimize/14_plan_stability`（36.6%，综述）与 `22_optimizer_worklog_timeline`（17.4%，WL 编年）仍豁免；
> `07_optimize/17_optimizer_decisions` **已补到 46.0% 达标，不再豁免**（见文末）。

**执行套路**（沿用 field 篇与 PT 篇的成功做法，**不生硬搬移**）：
1. 审计缺口（本篇特有内容 vs 已有内容，避免收敛时丢失）
2. 派 code-explorer agent 深挖真实源码（贴完整原代码 + 就近注释）
3. 写入/加深目标篇（代码占比冲 45~55%）
4. 校验：围栏 / 锚点 / 跨篇断链 / 代码占比

---


## server/query（主线：编译→优化→执行）（46 篇，不达标 **2**，**全是豁免** → 实缺 **0**，主线已闭环）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ⛔豁免 | `server/query/07_optimize/14_plan_stability.md` | 691 | 253 | 1.5% → **36.6%** |
| ⛔豁免 | `server/query/07_optimize/22_optimizer_worklog_timeline.md` | 626 | 109 | 4.0% → **17.4%** |
| ✅ 已达标 | `server/query/07_optimize/17_optimizer_decisions.md` | 1236 | 568 | 5.3% → **46.0%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/00_overview.md` | 1193 | 683 | 13.3% → **57.3%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/23_optimizer_trace_internals.md` | 1145 | 688 | 13.8% → **60.1%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/10_plan_refinement.md` | 1253 | 797 | 15.1% → **63.6%** ✅ |
| ✅ 已达标 | `server/query/runtime/02_subquery_runtime.md` | 1224 | 760 | 17.0% → **62.1%** ✅ |
| ✅ 已达标 | `server/query/08_access_path.md` | 2245 | 1044 | 18.0% → **46.5%** ✅ |
| ✅ 已达标 | `server/query/03_grammar_notation.md` | 697 | 372 | 18.2% → **53.4%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/logical/03_semijoin.md` | 2638 | 1850 | 18.7% → **70.1%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/01_cost_model.md` | 3963 | 2818 | 18.9% → **71.1%** ✅ |
| ✅ 已达标 | `server/query/runtime/04_window_function.md` | 2522 | 1951 | 19.3% → **77.4%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/16_join_object_model.md` | 1959 | 1284 | 19.5% → **65.5%** ✅ |
| ✅ 已达标 | `server/query/runtime/05_join_buffer.md` | 2072 | 1596 | 20.1% → **77.0%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/21_collation_index_usability.md` | 1485 | 879 | 21.9% → **59.2%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/physical/18_hypergraph_advanced.md` | 1409 | 733 | 24.6% → **52.0%** ✅ |
| ✅ 已达标 | `server/query/runtime/03_cte.md` | 2269 | 1302 | 24.8% → **57.4%** ✅ |
| ✅ 已达标 | `server/query/04_parser.md` | 2164 | 1418 | 25.5% → **65.5%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/physical/09_hypergraph.md` | 1525 | 892 | 25.9% → **58.5%** ✅ |
| ✅ 已达标 | `server/query/runtime/01_filesort.md` | 1921 | 1061 | 27.7% → **55.2%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/12_partition_pruning.md` | 1235 | 705 | 28.1% → **57.1%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/20_view_resolution.md` | 1558 | 827 | 28.2% → **53.1%** ✅ |
| ✅ 已达标 | `server/query/15_like.md` | 1986 | 1061 | 28.5% → **53.4%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/15_groupby_distinct_order.md` | 3958 | 2870 | 29.1% → **72.5%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/physical/08_range_optimizer.md` | 2236 | 1464 | 29.1% → **65.5%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/19_set_operation.md` | 1616 | 859 | 29.7% → **53.2%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/logical/04_logical_join.md` | 1549 | 800 | 29.8% → **51.6%** ✅ |
| ✅ 已达标 | `server/query/14_sql_digest.md` | 2136 | 1348 | 29.8% → **63.1%** ✅ |
| ✅ 已达标 | `server/query/02_lexer.md` | 2119 | 1616 | 31.2% → **76.3%** ✅ |
| ✅ 已达标 | `server/query/09_executor_iterator.md` | 2868 | 1639 | 33.0% → **57.1%** ✅ |
| ✅ 已达标 | `server/query/01_protocol_to_dispatch.md` | 3185 | 1905 | 34.7% → **59.8%** ✅ |
| ✅ 已达标 | `server/query/05_contextualize.md` | 3164 | 1837 | 41.5% → **58.1%** ✅ |
| ✅ 已达标 | `server/query/12_prepared_statement.md` | 1917 | 1024 | 32.4% → **53.4%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/13_functional_mv_index.md` | 1827 | 1072 | 32.4% → **58.7%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/11_optimizer_hints.md` | 1613 | 889 | 32.8% → **55.1%** ✅ |
| ✅ 已达标 | `server/query/10_dml.md` | 1631 | 952 | 34.1% → **58.4%** ✅ |
| ✅ 已达标 | `server/query/runtime/06_rollup.md` | 1231 | 655 | 35.2% → **53.2%** ✅ |
| ✅ 已达标 | `server/query/runtime/07_temptable.md` | 2148 | 1299 | 35.3% → **60.5%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/physical/07_access_method.md` | 1981 | 895 | 36.4% → **45.2%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/logical/02_subquery.md` | 1908 | 1108 | 39.2% → **58.1%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/logical/05_logical_predicate.md` | 2217 | 1247 | 39.7% → **56.2%** ✅ |
| ✅ 已达标 | `server/query/07_optimize/physical/06_join_order.md` | 1999 | 1045 | 39.9% → **52.3%** ✅ |

<details><summary>✅ 达标 4 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `server/query/11_parse_tree.md` | 3940 | 2478 | 62.9% ✅ |
| `server/query/13_item_expression.md` | 4583 | 2310 | 50.4% ✅ |
| `server/query/06_resolver_prepare.md` | 1728 | 837 | 48.4% ✅ |
| `server/query/runtime/08_materialization.md` | 1324 | 604 | 45.6% ✅ |

</details>


## server（其它：handler/field/table/auth/repl/lifecycle）（53 篇，不达标 **21**）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ✅ 已达标 | `server/view.md` | 1066 | 568 | 0.0% → **53.3%** ✅ |
| ✅ 已达标 | `server/infra/dbug.md` | 1632 | 1059 | 3.2% → **64.9%** ✅ |
| ✅ 已达标 | `server/datatype/gis.md` | 2211 | 1084 | 35.2% → **49.0%** ✅ |
| ✅ 已达标 | `server/dd/stored_program_dd.md` | 1908 | 1128 | 18.6% → **59.1%** ✅ |
| ✅ 已达标 | `server/replication/replication.md` | 2404 | 1132 | 18.7% → **47.1%** ✅ |
| ✅ 已达标 | `server/infra/mtr.md` | 2592 | 1626 | 18.8% → **62.7%** ✅ |
| **P0** | `server/infra/reading_guide.md` | 470 | 89 | 18.9% |
| ✅ 已达标 | `server/plugin/service.md` | 1263 | 651 | 20.1% → **51.5%** ✅ |
| ✅ 已达标 | `server/plugin/abi.md` | 1394 | 657 | 20.3% → **47.1%** ✅ |
| ✅ 已达标 | `server/datatype/json.md` | 4333 | 2052 | 21.7% → **47.4%** ✅ |
| ✅ 已达标 | `server/plugin/architecture.md` | 747 | 385 | 22.5% → **51.5%** ✅ |
| **P1** | `server/plugin/keyring.md` | 308 | 77 | 25.0% |
| **P1** | `server/replication/binlog_encryption.md` | 357 | 91 | 25.5% |
| **P1** | `server/plugin/validate_password.md` | 455 | 117 | 25.7% |
| **P1** | `server/replication/binlog_sbr.md` | 449 | 121 | 26.9% |
| **P1** | `server/plugin/clone.md` | 512 | 138 | 27.0% |
| **P1** | `server/replication/mysqlbinlog.md` | 506 | 137 | 27.1% |
| **P1** | `server/infra/encoding.md` | 507 | 139 | 27.4% |
| ✅ 已达标 | `server/plugin/component.md` | 1345 | 623 | 28.0% → **46.3%** ✅ |
| **P1** | `server/plugin/mysqlx.md` | 362 | 115 | 31.8% |
| **P2** | `server/plugin/registry.md` | 310 | 100 | 32.3% |
| **P2** | `server/replication/semisync.md` | 520 | 168 | 32.3% |
| ✅ 已达标 | `server/infra/vio.md` | 1515 | 1136 | 32.4% → **75.0%** ✅ |
| ✅ 已达标 | `server/infra/charset.md` | 4800 | 2205 | 32.7% → **45.9%** ✅ |
| **P2** | `server/auth/mfa.md` | 719 | 240 | 33.4% |
| **P2** | `server/auth/account_management.md` | 1212 | 414 | 34.2% |
| ✅ 已达标 | `server/infra/build.md` | 1744 | 964 | 34.7% → **55.3%** ✅ |
| ✅ 已达标 | `server/dd/statistics.md` | 1309 | 624 | 35.7% → **47.7%** ✅ |
| ✅ 已达标 | `server/dd/innodb_dict.md` | 1963 | 1107 | 35.9% → **56.4%** ✅ |
| **P2** | `server/auth/definer.md` | 531 | 191 | 36.0% |
| ✅ 已达标 | `server/dd/system_views.md` | 1670 | 841 | 36.7% → **50.4%** ✅ |
| **P2** | `server/plugin/plugin.md` | 784 | 288 | 36.7% |
| **P2** | `server/stored_program.md` | 1018 | 378 | 37.1% |
| **P2** | `server/infra/diagnostics.md` | 778 | 289 | 37.1% |
| **P2** | `server/xa.md` | 603 | 225 | 37.3% |
| ✅ 已达标 | `server/replication/binlog.md` | 3688 | 1931 | 38.8% → **52.4%** ✅ |
| ✅ 已达标 | `server/lifecycle/01_initialize.md` | 2383 | 1417 | 38.9% → **59.5%** ✅ |
| **P2** | `server/replication/preserve_commit_order.md` | 666 | 260 | 39.0% |
| ✅ 已达标 | `server/dd/dd.md` | 4303 | 1948 | 41.1% → **45.3%** ✅ |
| ✅ 已达标 | `server/replication/binlog_event.md` | 3234 | 1784 | 42.5% → **55.2%** ✅ |
| ✅ 已达标 | `server/infra/variables.md` | 2687 | 1470 | 42.8% → **54.7%** ✅ |
| **P3** | `server/auth/security_context.md` | 716 | 313 | 43.7% |
| ✅ 已达标 | `server/infra/io_cache.md` | 2944 | 1530 | 44.2% → **52.0%** ✅ |

<details><summary>✅ 达标 10 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `server/lifecycle/03_shutdown.md` | 2274 | 1306 | 57.4% ✅ |
| `server/field.md` | 5213 | 2978 | 57.1% ✅ |
| `server/auth/authorization.md` | 2776 | 1439 | 51.8% ✅ |
| `server/lifecycle/02_startup.md` | 2304 | 1178 | 51.1% ✅ |
| `server/replication/gtid.md` | 3508 | 2070 | 49.6% → **59.0%** ✅ |
| `server/handler.md` | 3269 | 1890 | 49.3% → **57.8%** ✅ |
| `server/table.md` | 8571 | 6482 | 47.4% → **75.6%** ✅ |
| `server/auth/authentication.md` | 4099 | 1884 | 46.4% → **46.0%** ✅ |
| `server/replication/prpl.md` | 3599 | 1539 | 46.3% → **42.8%** |
| `server/replication/replica.md` | 1161 | 524 | 45.1% ✅ |

</details>


## innodb（存储引擎）（17 篇，不达标 **1**）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ✅ 已达标 | `innodb/row_ops.md` | 1177 | 685 | 19.2% → **58.2%** ✅ |
| ✅ 已达标 | `innodb/undo_log.md` | 4824 | 2978 | 24.6% → **61.7%** ✅ |
| **P1** | `innodb/fil.md` | 602 | 152 | 25.2% |
| ✅ 已达标 | `innodb/parallel_scan.md` | 1591 | 910 | 26.1% → **57.2%** ✅ |
| ✅ 已达标 | `innodb/io.md` | 3026 | 1379 | 27.4% → **45.6%** ✅ |
| ✅ 已达标 | `innodb/redo_log.md` | 4027 | 2101 | 28.4% → **52.2%** ✅ |
| ✅ 已达标 | `innodb/physical/lob.md` | 1163 | 570 | 36.3% → **49.0%** ✅ |
| ✅ 已达标 | `innodb/buffer_pool.md` | 4568 | 2438 | 36.8% → **53.4%** ✅ |

<details><summary>✅ 达标 9 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `innodb/row_search.md` | 3420 | 2065 | 59.8% → **60.4%** ✅ |
| `innodb/query_graph.md` | 2336 | 1411 | 51.9% → **60.4%** ✅ |
| `innodb/physical/tablespace.md` | 2638 | 1482 | 51.8% → **56.2%** ✅ |
| `innodb/trx.md` | 3772 | 2096 | 49.8% → **55.6%** ✅ |
| `innodb/physical/record.md` | 1884 | 1124 | 49.3% → **59.7%** ✅ |
| `innodb/physical/page_structure.md` | 913 | 442 | 48.4% ✅ |
| `innodb/recovery.md` | 1005 | 474 | 47.2% ✅ |
| `innodb/dblwr.md` | 1133 | 521 | 46.0% ✅ |
| `innodb/mvcc.md` | 2215 | 1254 | 45.7% → **56.6%** ✅ |

</details>


## index（索引）（18 篇，不达标 **8**）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ✅ 已达标 | `index/metadata.md` | 1828 | 1118 | 15.8% → **61.2%** ✅ |
| ✅ 已达标 | `index/record_format.md` | 1113 | 565 | 16.4% → **50.8%** ✅ |
| ✅ 已达标 | `index/construction.md` | 1570 | 894 | 17.0% → **56.9%** ✅ |
| **P3** | `index/operations.md` | 1208 | 506 | 18.7% → **41.9%** |
| ✅ 已达标 | `index/physical_storage.md` | 2149 | 1129 | 39.1% → **52.5%** ✅ |
| **P2** | `index/rtree.md` | 749 | 250 | 20.7% → **33.4%** |
| **P1** | `index/stats.md` | 527 | 132 | 25.0% |
| ✅ 已达标 | `index/secondary.md` | 1569 | 962 | 25.7% → **61.3%** ✅ |
| ✅ 已达标 | `index/transaction.md` | 2012 | 1260 | 26.5% → **62.6%** ✅ |
| **P1** | `index/types.md` | 678 | 188 | 27.7% |
| **P1** | `index/access.md` | 642 | 188 | 29.3% |
| ✅ 已达标 | `index/ibuf.md` | 2799 | 1558 | 32.8% → **55.7%** ✅ |
| **P2** | `index/inverted.md` | 385 | 128 | 33.2% |
| **P2** | `index/constraint.md` | 666 | 238 | 35.7% |
| **P2** | `index/partitioning.md` | 615 | 228 | 37.1% |
| ✅ 已达标 | `index/ahi.md` | 1706 | 866 | 40.8% → **50.8%** ✅ |

<details><summary>✅ 达标 1 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `index/btr.md` | 3525 | 2252 | 63.9% ✅ |

</details>


## infra（基础设施：内存/锁/线程）（43 篇，不达标 **24**）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ✅ 已达标 | `infra/memory/memory_architecture.md` | 1217 | 563 | 17.1% → **46.3%** ✅ |
| ✅ 已达标 | `infra/observability/explain_trace.md` | 1025 | 480 | 19.4% → **46.8%** ✅ |
| ✅ 已达标 | `index/gipk.md` | 1048 | 564 | 53.8% ✅ |
| ✅ 已达标 | `infra/structure/counter.md` | 873 | 393 | 21.4% → **45.0%** ✅ |
| **P3** | `infra/observability/innodb_monitor.md` | 1145 | 493 | 22.5% → **43.1%** |
| **P1** | `infra/observability/logging/slow_log.md` | 429 | 108 | 25.2% |
| ✅ 已达标 | `infra/observability/pfs/thread_pfs.md` | 1400 | 671 | 25.3% → **47.9%** ✅ |
| **P1** | `infra/threading/thread_wait.md` | 599 | 154 | 25.7% |
| **P1** | `infra/observability/system_tools.md` | 549 | 145 | 26.4% |
| **P1** | `infra/memory/jemalloc.md` | 511 | 135 | 26.4% |
| **P1** | `infra/observability/show_commands.md` | 848 | 240 | 28.3% |
| **P1** | `infra/structure/link_buf.md` | 391 | 112 | 28.6% |
| **P1** | `infra/structure/queue.md` | 544 | 156 | 28.7% |
| ✅ 已达标 | `infra/lock/primitives/rcu.md` | 653 | 342 | 29.2% → **52.4%** ✅ |
| **P1** | `infra/threading/stack_depth.md` | 806 | 238 | 29.5% |
| **P1** | `infra/threading/internal_thd.md` | 649 | 193 | 29.7% |
| ✅ 已达标 | `infra/threading/threading_architecture.md` | 2870 | 1486 | 31.5% → **51.8%** ✅ |
| **P1** | `infra/lock/transactional/global_lock.md` | 584 | 184 | 31.5% |
| **P1** | `infra/observability/logging/error_log.md` | 539 | 171 | 31.7% |
| ✅ 已达标 | `infra/lock/primitives/mysys_primitives.md` | 1149 | 629 | 32.9% → **54.7%** ✅ |
| **P2** | `infra/structure/list.md` | 753 | 249 | 33.1% |
| **P2** | `infra/observability/status_variables.md` | 690 | 236 | 34.2% |
| ✅ 已达标 | `infra/structure/hash.md` | 1898 | 938 | 34.8% → **49.4%** ✅ |
| **P2** | `infra/lock/lock_order.md` | 257 | 91 | 35.4% |
| **P2** | `infra/observability/processlist.md` | 938 | 335 | 35.7% |
| **P2** | `infra/threading/thread_pool_mariadb_percona.md` | 2128 | 760 | 35.7% |
| ✅ 已达标 | `infra/observability/pfs/pfs.md` | 2216 | 1166 | 37.5% → **52.6%** ✅ |
| **P2** | `infra/observability/pfs/pfs_statement.md` | 1078 | 408 | 37.8% |
| **P2** | `infra/observability/i_s.md` | 1040 | 403 | 38.8% |
| **P2** | `infra/observability/pfs/pfs_events.md` | 1478 | 581 | 39.3% |
| ✅ 已达标 | `infra/lock/transactional/innodb_trx_lock.md` | 3887 | 2151 | 39.9% → **55.3%** ✅ |
| ✅ 已达标 | `infra/lock/primitives/seq_lock.md` | 362 | 200 | 40.6% → **55.2%** ✅ |
| **P3** | `infra/observability/logging/general_log.md` | 306 | 125 | 40.8% |
| **P3** | `infra/memory/allocator.md` | 1015 | 420 | 41.4% |
| **P3** | `infra/lock/transactional/thr_lock.md` | 869 | 363 | 41.8% |
| ✅ 已达标 | `infra/threading/thread_pool.md` | 3763 | 2077 | 41.9% → **55.2%** ✅ |
| **P3** | `infra/observability/sys_schema.md` | 1800 | 780 | 43.3% |
| ✅ 已达标 | `infra/threading/thd.md` | 2491 | 1596 | 44.3% → **64.1%** ✅ |
| ✅ 已达标 | `infra/lock/transactional/mdl.md` | 2380 | 1211 | 44.5% → **50.9%** ✅ |

<details><summary>✅ 达标 5 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `infra/threading/connection_threading.md` | 1070 | 569 | 53.2% ✅ |
| `infra/structure/data_repr.md` | 3415 | 1778 | 50.5% → **52.1%** ✅ |
| `infra/lock/primitives/innodb_sync.md` | 2407 | 1384 | 47.7% → **57.5%** ✅ |
| `infra/threading/thread_lifecycle.md` | 945 | 444 | 47.0% ✅ |
| `infra/observability/telemetry.md` | 1047 | 475 | 45.4% ✅ |

</details>


## feat（特性：DDL等）（16 篇，不达标 **3**）

| 优先级 | 文件 | 行数 | 代码 | 占比 |
|---|---|---:|---:|---:|
| ✅ 已达标 | `feat/ddl/ddl.md` | 2430 | 1102 | 22.1% → **45.3%** ✅ |
| ✅ 已达标 | `feat/fts.md` | 3449 | 1566 | 24.1% → **45.4%** ✅ |
| **P0** | `feat/ddl/online_ddl_tools.md` | 317 | 77 | 24.3% |
| ✅ 已达标 | `feat/partitioning.md` | 2979 | 1405 | 28.4% → **47.2%** ✅ |
| ✅ 已达标 | `feat/sdi.md` | 2564 | 1179 | 30.9% → **46.0%** ✅ |
| ✅ 已达标 | `feat/ddl/instant_ddl.md` | 2513 | 1513 | 58.4% → **60.2%** ✅ |
| ✅ 已达标 | `feat/auto_increment.md` | 876 | 509 | 38.9% → **58.1%** ✅ |
| ✅ 已达标 | `feat/ddl/copy_ddl.md` | 1509 | 818 | 41.5% → **54.2%** ✅ |
| **P3** | `feat/ddl/ddl_engine.md` | 1707 | 710 | 41.6% |
| ✅ 已达标 | `feat/ddl/atomic_ddl.md` | 1420 | 711 | 42.1% → **50.1%** ✅ |
| **P3** | `feat/compression.md` | 478 | 211 | 44.1% |

<details><summary>✅ 达标 5 篇（展开）</summary>

| 文件 | 行数 | 代码 | 占比 |
|---|---:|---:|---:|
| `feat/check_table.md` | 1126 | 727 | 64.6% ✅ |
| `feat/generated_columns.md` | 2428 | 1522 | 61.1% → **62.7%** ✅ |
| `feat/ddl/create_table.md` | 2142 | 1252 | 57.8% → **58.5%** ✅ |
| `feat/ddl/inplace_ddl.md` | 2417 | 1349 | 51.3% → **55.8%** ✅ |
| `feat/ddl/alter_table.md` | 992 | 504 | 50.8% ✅ |

</details>


## ⛔ 不纳入范围：cloud/ 与 papers/

> **2026-10-08 用户裁定**：`cloud/*.md`（云数据库产品向）与 `papers/*.md`（论文笔记）**不参与深度补全**，已从统计中剔除。
>
> 原因：这两类不是 MySQL 内核源码剖析——`cloud` 是产品/架构介绍，`papers` 是文献笔记，本就没有"该贴什么源码"的对应物，用代码占比去卡它们没有意义。
>
> **后续再扫描全库时，请跳过这两个目录**，不要因为它们占比低而重新入账。

原列于此处的 4 篇（均为 cloud/papers）：`cloud/cloud_db.md`（348 行 / 8.0%）、`papers/btrlog.md`（904 行 / 9.4%）、`cloud/cloud_storage.md`（679 行 / 16.8%）、`cloud/cloud_networking.md`（1212 行 / 21.7%）。

> 注：本节标题里的 `dd` / `datatype` 其实**不在**本节——`server/dd/*` 与 `server/datatype/*` 已归入上面的「server（其它）」章节，仍在补全范围内。

## ⛔ 不纳入范围：综述 / 决策记录（2026-10-14 复核：3 篇 → 2 篇）

> **2026-10-08 用户裁定**：以下篇目占比低但**不用补**。判据：是否"存在与正文一一对应的 MySQL 源码"——
> 存在 → 按 45% 卡；不存在（综述 / 编年 / 跨库对比）→ 豁免，不看占比。
>
> ⚠️ **2026-10-14 复核**：`17_optimizer_decisions` 已补到 **46.0% 达标**，从豁免清单移出（共 3 篇 → 2 篇）。

| 文件 | 行数 | 代码 | 占比 | 状态 / 豁免理由 |
|---|---:|---:|---:|---|
| `server/query/07_optimize/14_plan_stability.md` | 691 | 253 | 36.6% | ⛔ 仍豁免：SPM / plan baseline 综述，MySQL 8.0 **无此功能**，主体是 Oracle / PolarDB 跨库对比，没有对应源码可贴 |
| `server/query/07_optimize/22_optimizer_worklog_timeline.md` | 626 | 109 | 17.4% | ⛔ 仍豁免：WL 编年时间线，纯史料 |
| `server/query/07_optimize/17_optimizer_decisions.md` | 1236 | 568 | 46.0% | ✅ **已达标，移出豁免清单**（10-05 记 487 行 / 5.3%，期间补写了决策依据的源码） |

## 📊 全库审计快照（2026-10-14 自动扫描，非估算）

- 规模：**193 篇 / 343036 行 / 183525 码 / 53.5%**
- 达标 **134** / 不达标 **59**（30.6%）
- 本篇快照为 10-14 补完 `server/replication/binlog.md`（2276→3688 行，38.8%→52.4%，+1048 行代码）与
  `infra/observability/pfs/pfs.md`（1178→2216 行，37.5%→52.6%，+724 行代码）后的实测值
- 本篇快照为 10-14 补完 `index/transaction.md`（438→2012 行，26.5%→62.6%）、
  `innodb/undo_log.md`（3719→4824 行，54.8%→61.7%，+939 行代码）与
  `server/query/07_optimize/physical/06_join_order.md`（1261→1999 行，41.4%→52.3%，+523 行代码）后的实测值
- 至此 `server/query` 主线 46 篇全部达标（余下 2 篇为豁免的综述/编年）
- 分布：<25% 3 篇、25–32% 25 篇、32–40% 25 篇、40–45% 11 篇、45–53% 49 篇、53–65% 69 篇、≥65% 11 篇
- 口径：剔除 `cloud/` + `papers/`（4 篇，用户 10-08 裁定）与 `BACKLOG.md` / `README.md` / `_template.md`；
  代码行 = 围栏块内的行（不含围栏本身）
- 硬问题：fence 平衡 ✅（193 篇全部偶数围栏）/ 断链 **6 处已修**（3 锚点 + 3 目录链接，见下）
- 「到 45% 需补」= `ceil((0.45×总行 − 代码行) / 0.55)`：**补进去的行同时进分子也进分母**，所以用 0.55 而不是 0.45

### 本次审计修掉的 6 处硬断链

| 位置 | 原链接 | 修为 | 原因 |
|---|---|---|---|
| `server/dd/statistics.md:142` | `dd.md#22--options-vs-se_private_data` | `#23--options-vs-se_private_data` | `options vs se_private_data` 是 **2.3** 节，不是 2.2（2.2 是 `dd::Properties`） |
| `server/plugin/keyring.md:5 / :105` | `component.md#启动引导三条链` | `#启动引导四条链不是三条` | 目标标题已改为「启动引导：**四**条链（不是三条）」，正文里的"三条链"也一并改 |
| `server/query/04_parser.md:2016` | `01_protocol_to_dispatch.md#七mem_root…` | `#八mem_root为什么-parse-tree-从不-delete` | mem_root 是**第八**节，「七」是 LEX 生命周期 |
| `infra/observability/sys_schema.md:1799` | `../logging/` | `logging/` | 日志目录在 `infra/observability/logging/`，不在 `infra/logging/` |
| `infra/threading/threading_architecture.md:7` | `lock/` | `../lock/` | 锁目录在 `infra/lock/`，与 `infra/threading/` 平级 |
| `infra/threading/threading_architecture.md:1476` | `lock/primitives/` | `../lock/primitives/` | 同上 |

### 低于基线的篇目（按目录，含新档位）

**server/query（主线）** — 共 46 篇，达标 43，不达标 3（91156 行 / 53881 码 / 59.1%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| ⛔豁免 | 17.4% | 626 | 109 | `server/query/07_optimize/22_optimizer_worklog_timeline.md` | — |
| ⛔豁免 | 36.6% | 691 | 253 | `server/query/07_optimize/14_plan_stability.md` | — |
| ✅ 已达标 | 52.3% | 1999 | 1045 | `server/query/07_optimize/physical/06_join_order.md` | 已达标（补于 2026-10-14） |

**server（其它）** — 共 53 篇，达标 31，不达标 22（100280 行 / 52054 码 / 51.9%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| P0 | 18.9% | 470 | 89 | `server/infra/reading_guide.md` | +223 |
| P1 | 25.0% | 308 | 77 | `server/plugin/keyring.md` | +112 |
| P1 | 25.5% | 357 | 91 | `server/replication/binlog_encryption.md` | +127 |
| P1 | 25.7% | 455 | 117 | `server/plugin/validate_password.md` | +160 |
| P1 | 26.9% | 449 | 121 | `server/replication/binlog_sbr.md` | +148 |
| P1 | 27.0% | 512 | 138 | `server/plugin/clone.md` | +168 |
| P1 | 27.1% | 506 | 137 | `server/replication/mysqlbinlog.md` | +165 |
| P1 | 27.4% | 507 | 139 | `server/infra/encoding.md` | +163 |
| P1 | 31.8% | 362 | 115 | `server/plugin/mysqlx.md` | +88 |
| P2 | 32.3% | 310 | 100 | `server/plugin/registry.md` | +72 |
| P2 | 32.3% | 520 | 168 | `server/replication/semisync.md` | +120 |
| P2 | 33.4% | 719 | 240 | `server/auth/mfa.md` | +152 |
| P2 | 34.2% | 1212 | 414 | `server/auth/account_management.md` | +239 |
| P2 | 36.0% | 531 | 191 | `server/auth/definer.md` | +88 |
| P2 | 36.7% | 784 | 288 | `server/plugin/plugin.md` | +118 |
| P2 | 37.1% | 1018 | 378 | `server/stored_program.md` | +146 |
| P2 | 37.1% | 778 | 289 | `server/infra/diagnostics.md` | +112 |
| P2 | 37.3% | 603 | 225 | `server/xa.md` | +85 |
| ✅ 已达标 | 52.4% | 3688 | 1931 | `server/replication/binlog.md` | 已达标（补于 2026-10-14） |
| P2 | 39.0% | 666 | 260 | `server/replication/preserve_commit_order.md` | +73 |
| P3 | 42.8% | 3599 | 1539 | `server/replication/prpl.md` | +147 |
| P3 | 43.7% | 716 | 313 | `server/auth/security_context.md` | +17 |

**innodb（存储引擎）** — 共 17 篇，达标 16，不达标 1（39189 行 / 21143 码 / 54.0%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| P1 | 25.2% | 602 | 152 | `innodb/fil.md` | +217 |

**index（索引）** — 共 18 篇，达标 10，不达标 8（24916 行 / 13118 码 / 52.6%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| P1 | 25.0% | 527 | 132 | `index/stats.md` | +192 |
| ✅ 已达标 | 61.1% | 2062 | 1260 | `index/transaction.md` | 已达标（补于 2026-10-14） |
| P1 | 27.7% | 678 | 188 | `index/types.md` | +213 |
| P1 | 29.3% | 642 | 188 | `index/access.md` | +184 |
| P2 | 33.2% | 385 | 128 | `index/inverted.md` | +83 |
| P2 | 33.4% | 749 | 250 | `index/rtree.md` | +159 |
| P2 | 35.7% | 666 | 238 | `index/constraint.md` | +113 |
| P2 | 37.1% | 615 | 228 | `index/partitioning.md` | +89 |
| P3 | 41.9% | 1208 | 506 | `index/operations.md` | +69 |

**infra（基础设施）** — 共 43 篇，达标 17，不达标 26（52586 行 / 23889 码 / 45.4%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| P1 | 25.2% | 429 | 108 | `infra/observability/logging/slow_log.md` | +155 |
| P1 | 25.7% | 599 | 154 | `infra/threading/thread_wait.md` | +211 |
| P1 | 26.4% | 549 | 145 | `infra/observability/system_tools.md` | +186 |
| P1 | 26.4% | 511 | 135 | `infra/memory/jemalloc.md` | +173 |
| P1 | 28.3% | 848 | 240 | `infra/observability/show_commands.md` | +258 |
| P1 | 28.6% | 391 | 112 | `infra/structure/link_buf.md` | +117 |
| P1 | 28.7% | 544 | 156 | `infra/structure/queue.md` | +162 |
| P1 | 29.5% | 806 | 238 | `infra/threading/stack_depth.md` | +227 |
| P1 | 29.7% | 649 | 193 | `infra/threading/internal_thd.md` | +181 |
| ✅ 已达标 | 51.8% | 2870 | 1486 | `infra/threading/threading_architecture.md` | 已达标（补于 2026-10-14） |
| P1 | 31.5% | 584 | 184 | `infra/lock/transactional/global_lock.md` | +144 |
| P1 | 31.7% | 539 | 171 | `infra/observability/logging/error_log.md` | +131 |
| P2 | 33.1% | 753 | 249 | `infra/structure/list.md` | +164 |
| P2 | 34.2% | 690 | 236 | `infra/observability/status_variables.md` | +136 |
| P2 | 35.4% | 257 | 91 | `infra/lock/lock_order.md` | +45 |
| P2 | 35.7% | 938 | 335 | `infra/observability/processlist.md` | +159 |
| P2 | 35.7% | 2128 | 760 | `infra/threading/thread_pool_mariadb_percona.md` | +360 |
| ✅ 已达标 | 52.6% | 2216 | 1166 | `infra/observability/pfs/pfs.md` | 已达标（补于 2026-10-14） |
| P2 | 37.8% | 1078 | 408 | `infra/observability/pfs/pfs_statement.md` | +141 |
| P2 | 38.8% | 1040 | 403 | `infra/observability/i_s.md` | +119 |
| P2 | 39.3% | 1478 | 581 | `infra/observability/pfs/pfs_events.md` | +153 |
| P3 | 40.8% | 306 | 125 | `infra/observability/logging/general_log.md` | +24 |
| P3 | 41.4% | 1015 | 420 | `infra/memory/allocator.md` | +67 |
| P3 | 41.8% | 869 | 363 | `infra/lock/transactional/thr_lock.md` | +52 |
| P3 | 43.1% | 1145 | 493 | `infra/observability/innodb_monitor.md` | +41 |
| P3 | 43.3% | 1800 | 780 | `infra/observability/sys_schema.md` | +55 |

**feat（特性）** — 共 16 篇，达标 13，不达标 3（29347 行 / 15155 码 / 51.6%）

| 新档位 | 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---:|---:|---:|---|---:|
| P0 | 24.3% | 317 | 77 | `feat/ddl/online_ddl_tools.md` | +120 |
| P3 | 41.6% | 1707 | 710 | `feat/ddl/ddl_engine.md` | +106 |
| P3 | 44.1% | 478 | 211 | `feat/compression.md` | +8 |

---

#### ⬇️ 以下为 2026-10-05 旧快照（留档，勿据此行动）

**server/** — 41 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 17.4% | 626 | 109 | `server/query/07_optimize/22_optimizer_worklog_timeline.md` | +314 |
| 18.9% | 470 | 89 | `server/infra/reading_guide.md` | +223 |
| 57.4% ✅ | 2269 | 1302 | `server/query/runtime/03_cte.md` | 已达标 |
| 25.0% | 308 | 77 | `server/plugin/keyring.md` | +112 |
| 25.5% | 357 | 91 | `server/replication/binlog_encryption.md` | +127 |
| 25.7% | 455 | 117 | `server/plugin/validate_password.md` | +160 |
| 26.9% | 449 | 121 | `server/replication/binlog_sbr.md` | +148 |
| 27.0% | 512 | 138 | `server/plugin/clone.md` | +169 |
| 27.1% | 506 | 137 | `server/replication/mysqlbinlog.md` | +165 |
| 27.4% | 507 | 139 | `server/infra/encoding.md` | +163 |
| 55.2% ✅ | 1921 | 1061 | `server/query/runtime/01_filesort.md` | 已达标 |
| 53.1% ✅ | 1558 | 827 | `server/query/07_optimize/20_view_resolution.md` | 已达标 |
| 53.4% ✅ | 1986 | 1060 | `server/query/15_like.md` | 已达标 |
| 31.8% | 362 | 115 | `server/plugin/mysqlx.md` | +88 |
| 32.3% | 310 | 100 | `server/plugin/registry.md` | +72 |
| 32.3% | 520 | 168 | `server/replication/semisync.md` | +120 |
| 53.4% ✅ | 1917 | 1024 | `server/query/12_prepared_statement.md` | 已达标 |
| 59.0% ✅ | 1827 | 1053 | `server/query/07_optimize/13_functional_mv_index.md` | 已达标 |
| 33.4% | 719 | 240 | `server/auth/mfa.md` | +152 |
| 34.2% | 1212 | 414 | `server/auth/account_management.md` | +239 |
| 59.8% ✅ | 3185 | 1905 | `server/query/01_protocol_to_dispatch.md` | 已达标 |
| 53.2% ✅ | 1231 | 655 | `server/query/runtime/06_rollup.md` | 已达标 |
| 49.7% ✅ | 2211 | 1084 | `server/datatype/gis.md` | 已达标 |
| 60.5% ✅ | 2148 | 1299 | `server/query/runtime/07_temptable.md` | 已达标 |
| 47.7% ✅ | 1309 | 624 | `server/dd/statistics.md` | 已达标 |
| 36.0% | 531 | 191 | `server/auth/definer.md` | +88 |
| 36.6% | 691 | 253 | `server/query/07_optimize/14_plan_stability.md` | +106 |
| 50.4% ✅ | 1670 | 841 | `server/dd/system_views.md` | 已达标 |
| 36.7% | 784 | 288 | `server/plugin/plugin.md` | +118 |
| 37.1% | 1018 | 378 | `server/stored_program.md` | +146 |
| 37.1% | 778 | 289 | `server/infra/diagnostics.md` | +112 |
| 37.3% | 603 | 225 | `server/xa.md` | +85 |
| 38.8% | 2276 | 883 | `server/replication/binlog.md` | +257 |
| 39.0% | 666 | 260 | `server/replication/preserve_commit_order.md` | +73 |
| 56.2% ✅ | 2217 | 1247 | `server/query/07_optimize/logical/05_logical_predicate.md` | 已达标 |
| 45.3% ✅ | 4303 | 1948 | `server/dd/dd.md` | 已达标 |
| 41.4% | 1261 | 522 | `server/query/07_optimize/physical/06_join_order.md` | +83 |
| 55.2% ✅ | 3234 | 1784 | `server/replication/binlog_event.md` | 已达标 |
| 42.8% | 3599 | 1539 | `server/replication/prpl.md` | +147 |
| 54.7% ✅ | 2687 | 1470 | `server/infra/variables.md` | 已达标 |
| 43.7% | 716 | 313 | `server/auth/security_context.md` | +17 |

**infra/** — 26 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 25.2% | 429 | 108 | `infra/observability/logging/slow_log.md` | +155 |
| 25.7% | 599 | 154 | `infra/threading/thread_wait.md` | +211 |
| 26.4% | 549 | 145 | `infra/observability/system_tools.md` | +186 |
| 26.4% | 511 | 135 | `infra/memory/jemalloc.md` | +173 |
| 28.3% | 848 | 240 | `infra/observability/show_commands.md` | +258 |
| 28.6% | 391 | 112 | `infra/structure/link_buf.md` | +117 |
| 28.7% | 544 | 156 | `infra/structure/queue.md` | +162 |
| 29.5% | 806 | 238 | `infra/threading/stack_depth.md` | +227 |
| 29.7% | 649 | 193 | `infra/threading/internal_thd.md` | +181 |
| 31.5% | 1480 | 466 | `infra/threading/threading_architecture.md` | +364 |
| 31.5% | 584 | 184 | `infra/lock/transactional/global_lock.md` | +144 |
| 31.7% | 539 | 171 | `infra/observability/logging/error_log.md` | +131 |
| 33.1% | 753 | 249 | `infra/structure/list.md` | +164 |
| 34.2% | 690 | 236 | `infra/observability/status_variables.md` | +136 |
| 35.4% | 257 | 91 | `infra/lock/lock_order.md` | +45 |
| 35.7% | 938 | 335 | `infra/observability/processlist.md` | +159 |
| 35.7% | 2128 | 760 | `infra/threading/thread_pool_mariadb_percona.md` | +360 |
| 37.5% | 1178 | 442 | `infra/observability/pfs/pfs.md` | +161 |
| 37.8% | 1078 | 408 | `infra/observability/pfs/pfs_statement.md` | +141 |
| 38.8% | 1040 | 403 | `infra/observability/i_s.md` | +119 |
| 39.3% | 1478 | 581 | `infra/observability/pfs/pfs_events.md` | +153 |
| 40.8% | 306 | 125 | `infra/observability/logging/general_log.md` | +24 |
| 41.4% | 1015 | 420 | `infra/memory/allocator.md` | +67 |
| 41.8% | 869 | 363 | `infra/lock/transactional/thr_lock.md` | +52 |
| 43.1% | 1145 | 493 | `infra/observability/innodb_monitor.md` | +41 |
| 43.3% | 1800 | 780 | `infra/observability/sys_schema.md` | +55 |

**index/** — 10 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 25.0% | 527 | 132 | `index/stats.md` | +192 |
| 26.5% | 438 | 116 | `index/transaction.md` | +148 |
| 27.7% | 678 | 188 | `index/types.md` | +213 |
| 29.3% | 642 | 188 | `index/access.md` | +184 |
| 33.2% | 385 | 128 | `index/inverted.md` | +83 |
| 33.4% | 749 | 250 | `index/rtree.md` | +159 |
| 35.7% | 666 | 238 | `index/constraint.md` | +113 |
| 37.1% | 615 | 228 | `index/partitioning.md` | +89 |
| 52.5% ✅ | 2149 | 1129 | `index/physical_storage.md` | 已达标 |
| 42.0% | 1206 | 506 | `index/operations.md` | +67 |

**feat/** — 6 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 45.4% ✅ | 3449 | 1566 | `feat/fts.md` | 已达标 |
| 24.3% | 317 | 77 | `feat/ddl/online_ddl_tools.md` | +120 |
| 46.0% ✅ | 2564 | 1179 | `feat/sdi.md` | 已达标 |
| 54.2% ✅ | 1509 | 818 | `feat/ddl/copy_ddl.md` | 已达标 |
| 41.6% | 1707 | 710 | `feat/ddl/ddl_engine.md` | +106 |
| 44.1% | 478 | 211 | `feat/compression.md` | +8 |

**cloud/** — 3 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 8.0% | 348 | 28 | `cloud/cloud_db.md` | +234 |
| 16.8% | 679 | 114 | `cloud/cloud_storage.md` | +349 |
| 21.7% | 1212 | 263 | `cloud/cloud_networking.md` | +514 |

**innodb/** — 2 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 25.2% | 602 | 152 | `innodb/fil.md` | +217 |
| 49.0% ✅ | 1163 | 570 | `innodb/physical/lob.md` | 已达标 |

**_template.md/** — 1 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 5.7% | 228 | 13 | `_template.md` | +163 |

**papers/** — 1 篇

| 占比 | 行 | 码 | 文件 | 到45%需补 |
|---|---|---|---|---|
| 9.4% | 904 | 85 | `papers/btrlog.md` | +586 |
