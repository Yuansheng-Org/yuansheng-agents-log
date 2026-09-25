# `related/` 补充追踪信息 — 记录 011

> **本目录下的内容不是本记录的补丁，不构成 craft 证据。**
> 它只是"方向一致的追踪补充信息"：同一测试用例、同命名空间、同动词族或同代码族的
> 上游产物，用于把一个方向上的上下文补齐。判定与原件的关系等级见下。

**路径约定**

| 前缀 | 含义 |
| --- | --- |
| `craft/related/` | craft 侧补充：`craft/faiss` 的 AI 计划/候选/审核，或快照 `blueprint/` 诊断工件 |
| `trace/related/` | trace 侧补充：Agent Debug 的采集产物（annotate/metadata/perf_stat/diaglog/blueprint），或快照 `evidence/` |
| 本记录的 `craft/` `reviews/` `trace/`（无 `related/` 前缀） | **原件**，未因本次补充而改动 |


## 关系判定


- 补录 trace 用例 `rcq-search/012_faiss_HeapBlockResultHandler_CMax_float_long_false_add_results_c` — T3 同代码族（堆/结果集 `add_results`）；`MinimaxHeap`/`pop_best` 在 106 个 trace 用例中 0 命中。
- 补录 trace 用例 `rcq-search/023_void_faiss_top1_add_results_tpl_CMax_float_long_false_SIMDLevel_` — T3 同代码族（`top1` 堆结果集）。

## 本记录新增文件（10 个）

| 目录 | 文件数 |
| --- | ---: |
| `trace/related/rcq-search/` | 10 |

逐个文件清单见 `E:/2026/yuansheng/outputs/faiss-agents-log-records-20260924/faiss-related-supplement.md`。
