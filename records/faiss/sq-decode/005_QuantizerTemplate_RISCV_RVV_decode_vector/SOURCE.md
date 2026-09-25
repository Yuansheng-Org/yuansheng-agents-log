# `related/` 补充追踪信息 — 记录 005

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

本记录的 `craft/patch.diff` 与快照 `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988` 的 `candidate-d5e4afe64193b60e461707799a1c7d25.patch` **LF 归一同字节**，
且本记录的 `trace/blueprint_*.json` 与该快照用例的 `blueprint/blueprint.json`
来自同一次采集。
其 `source.testcaseIds` / `target.testcase` 一致，因此该快照用例的
`blueprint/`（craft 侧）与 `evidence/`（trace 侧）已整体并入本记录。

| 项 | 值 |
| --- | --- |
| 本记录原件 trace 用例 | `sq-decode/005_QuantizerTemplate_Codec4bit_RISCV_RVV_Scaling_SL_0_decode_vector` |
| 快照并用例 | `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988` |



## 本记录新增文件（11 个）

| 目录 | 文件数 |
| --- | ---: |
| `craft/related/snapshot/` | 7 |
| `trace/related/snapshot/` | 4 |

逐个文件清单见 `E:/2026/yuansheng/outputs/faiss-agents-log-records-20260924/faiss-related-supplement.md`。
