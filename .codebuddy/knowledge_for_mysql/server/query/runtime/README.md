# MySQL 执行期专题（Executor）

> 本目录是 [`query/09_executor_iterator.md`](../09_executor_iterator.md)（RowIterator 火山模型）的**横向展开**——那些迭代器要处理的特殊场景，各自独立成篇。

## 目录

- [这个目录讲什么](#这个目录讲什么)
- [文件索引](#文件索引)
- [与相邻文档的关系](#与相邻文档的关系)

---

## 这个目录讲什么

`query/09` 讲**迭代器框架**（火山契约、状态机、显式栈翻译）。本目录讲**具体执行能力的物理实现**：

```
query/09  RowIterator 框架
              │
              ├─ 排序/物化      → 01_filesort.md
              ├─ 内部临时表      → 07_temptable.md
              ├─ 子查询运行期    → 02_subquery_runtime.md
              ├─ CTE           → 03_cte.md
              ├─ 窗口函数       → 04_window_function.md
              ├─ join buffer   → 05_join_buffer.md
              └─ ROLLUP        → 06_rollup.md
```

这些主题的共同特点：**跨优化器与执行器边界**——优化器决定"用哪种策略"，执行器负责"怎么把它跑出来"。

| 主题 | 优化器侧（决定策略） | 执行器侧（本目录，实现机制） |
|---|---|---|
| 排序 | `optimizer/10` 的 Ordering index 选择 | filesort 算法、sort buffer、多路归并 |
| 子查询 | `optimizer/logical/03` 的 semi-join 五策略选择 | 物化引擎、IN2EXISTS 运行 |
| CTE | `optimizer/logical/04` 的 merge 判定 | 递归收敛循环、FollowTailIterator |
| 窗口 | `optimizer/10` 的临时表与排序 | partition/peer/frame 判定 |
| join buffer | `optimizer/10` 的 `setup_join_buffering` 决策 | BNL/BKA/HashJoin 的缓冲机制 |

---

## 文件索引

| 文件 | 核心内容 |
|------|----------|
| [01_filesort.md](01_filesort.md) | filesort：边装边溢、addon vs rowid、LIMIT 优化、多路归并 |
| [02_subquery_runtime.md](02_subquery_runtime.md) | 子查询运行期：物化、IN2EXISTS、semi-join 五策略运行期 |
| [03_cte.md](03_cte.md) | CTE（含递归）：merge/materialize 决策、递归收敛循环 |
| [04_window_function.md](04_window_function.md) | 窗口函数执行内部：流式 vs 缓冲、frame 语义 |
| [05_join_buffer.md](05_join_buffer.md) | join buffer：BNL 改 hash join、BKA、build/probe/spill |
| [07_temptable.md](07_temptable.md) | 内部临时表：TempTable 的 Row/Cell 与标准容器索引、RAM→mmap→InnoDB 四级降级链、`create_tmp_table` 的 hash_field 与四类键、共享物化与降级时换 handler |
| [08_materialization.md](08_materialization.md) | 物化：MaterializeIterator、LIMIT 内建、invalidators |
| [06_rollup.md](06_rollup.md) | ROLLUP：多层聚合与分组列 NULL 化 |

---

## 与相邻文档的关系

```
../README.md          主链：文本 → RowIterator 树
        │
        ├── ../09_executor_iterator.md   RowIterator 框架（本目录的上游）
        │
        └── 【本目录】        执行期专题（框架的横向展开）
                 │
                 ▼
        ../../handler.md      handler 接口（position/ref/rnd_pos）
        ../../../innodb/row_search.md  InnoDB 取行
        ../../../innodb/           InnoDB 存储引擎
```

- **与 `query/09` 的分工**：09 讲"迭代器怎么组织"，本目录讲"某个具体能力怎么实现"。例如 hash join 的**迭代器状态机**在 09，hash join 的 **build/probe/spill 缓冲细节**在 05。
- **与 `optimizer/` 的分工**：优化器决定策略（如"用 Materialize 而非 FirstMatch"），本目录讲该策略在执行期怎么跑。
- **与 handler / InnoDB 的分工**：本目录到"迭代器产出一行"为止；handler 接口语义见 [`../../handler.md`](../../handler.md)，InnoDB 取行见 [`../../../innodb/row_search.md`](../../../innodb/row_search.md)。

---

## 参考

本目录相关资料：

| 资料 | 对应 |
|---|---|
| **Graefe《Volcano》(1990)** | 迭代器框架（`query/09` 第 1 节） |
| **Graefe《Query Evaluation Techniques for Large Databases》(1993)** | sort-merge / hash join（`05_join_buffer.md`、`01_filesort.md`） |
| 官方文档 *Internal Temporary Table Use in MySQL* | `07_temptable.md` |
| 官方文档 *Window Functions* | `04_window_function.md` |
| 官方文档 *WITH (Common Table Expressions)* | `03_cte.md` |
| 官方文档 *Optimizing IN/=ANY Subqueries* | `02_subquery_runtime.md` 的优化器侧背景 |
| 内核月报 2021/06《Semi-join 优化与执行逻辑》 | `02_subquery_runtime.md` |
