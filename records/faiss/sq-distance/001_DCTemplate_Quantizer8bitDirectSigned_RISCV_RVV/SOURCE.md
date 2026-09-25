# `related/` 补充追踪信息 — 记录 001

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

本记录的 `craft/patch.diff` 与快照 `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea` 的 `candidate-42f3412bbfa105b82a1767fc307da3ad.patch` **LF 归一同字节**，
且本记录的 `trace/blueprint_*.json` 与该快照用例的 `blueprint/blueprint.json`
来自同一次采集。
其 `source.testcaseIds` / `target.testcase` 一致，因此该快照用例的
`blueprint/`（craft 侧）与 `evidence/`（trace 侧）已整体并入本记录。

| 项 | 值 |
| --- | --- |
| 本记录原件 trace 用例 | `sq-distance/013_DCTemplate_Quantizer8bitDirectSigned_6_SimilarityIP_6_SL_0_query` |
| 快照并用例 | `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea` |


- 并入 `craft/faiss` 全部 20 个目录 — 其中 `013_DCTemplate_Quantizer8bitDirectSigned_6_SimilarityIP_6_SL_0_query`
  与本记录**共用同一 `blueprintId`（`bp-faiss-sq-distance-013`）与同一 testcase**（等级 **C1**）。
  其余 17 个 `DCTemplate` 目录的 `patch.diff` 归一到**同一份**累积补丁
  （触及 `Clustering.cpp` / `sq-rvv.cpp` / `distances_dispatch.h` / `distances_rvv.cpp`），
  与本记录只共享 `sq-rvv.cpp`（等级 **C3**）；另有 `pq-dis-tables-dsub2` 1 个、`rcq-search` 2 个（亦 C3/C2）。
  **该整块以 20 目录为不可分割单位保存**——17 份是同一正文，拆分到各条记录会伪装成"各条自己的 craft"。


## 本记录新增文件（111 个）

| 目录 | 文件数 |
| --- | ---: |
| `craft/related/faiss/` | 100 |
| `craft/related/snapshot/` | 7 |
| `trace/related/snapshot/` | 4 |

逐个文件清单见 `E:/2026/yuansheng/outputs/faiss-agents-log-records-20260924/faiss-related-supplement.md`。
