# Faiss：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`Faiss`；目录标识：`faiss`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 6      | 106    | 20               | 11     | 5             | 原图空白／待补录     |

## 已筛选补丁的产物归档

本目录按 `records/faiss/<测试用例>/<函数>/` 组织，与 `records/mxnet` 同构。
测试用例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，
新增的 Trace、Craft 中间产物来自 Agent Debug 与 `yuansheng-patches` 快照。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID | 提交包 | PR 描述 | 文件数 |
| --- | --- | --- | --- | --- | --- | ---: |
| [001_DCTemplate_Quantizer8bitDirectSigned_RISCV_RVV](sq-distance/001_DCTemplate_Quantizer8bitDirectSigned_RISCV_RVV/) | `sq-distance` | `trace/faiss/sq-distance/013_DCTemplate_Quantizer8bitDirectSigned_6_SimilarityIP_6_SL_0_query` | `bp-faiss-sq-distance-013` | `1-faiss-pr5602-e813ff36` | `1-PR-5602.md` | 14 |
| [002_QuantizerFP16_RISCV_RVV_encode_vector](sq-encode/002_QuantizerFP16_RISCV_RVV_encode_vector/) | `sq-encode` | `trace/faiss/sq-encode/007_QuantizerFP16_SL_0_encode_vector` | `bp-faiss-sq-encode-007` | `2-faiss-pr5603-201706e0` | `2-PR-5603.md` | 14 |
| [003_Quantizer8bitDirect_RISCV_RVV_encode_vector](sq-encode/003_Quantizer8bitDirect_RISCV_RVV_encode_vector/) | `sq-encode` | `trace/faiss/sq-encode/010_Quantizer8bitDirect_SL_0_encode_vector` | `bp-faiss-sq-encode-010` | `3-faiss-pr5604-2ae54867` | `3-PR-5604.md` | 14 |
| [004_decode_bf16_simd](sq-decode/004_decode_bf16_simd/) | `sq-decode` | `trace/faiss/sq-decode/012_QuantizerBF16_SL_0_decode_vector` | `bp-faiss-sq-decode-012` | `4-faiss-pr5631-e683890a` | `4-PR-5631.md` | 14 |
| [005_QuantizerTemplate_RISCV_RVV_decode_vector](sq-decode/005_QuantizerTemplate_RISCV_RVV_decode_vector/) | `sq-decode` | `trace/faiss/sq-decode/005_QuantizerTemplate_Codec4bit_RISCV_RVV_Scaling_SL_0_decode_vector` | `bp-faiss-sq-decode-005` | `6-rvv-sq-decode-codec-template` | `6-rvv-sq-decode-codec-template.md` | 14 |
| [006_QuantizerFP16_BF16_8bitDirect_RISCV_RVV_decode_vector](sq-decode/006_QuantizerFP16_BF16_8bitDirect_RISCV_RVV_decode_vector/) | `sq-decode` | `trace/faiss/sq-decode/013_QuantizerFP16_SL_0_decode_vector` | `bp-faiss-sq-decode-013` | `7-rvv-sq-decode-raw-codecs` | `7-rvv-sq-decode-raw-codecs.md` | 14 |
| [007_QuantizerTemplate_RISCV_RVV_encode_vector](sq-encode/007_QuantizerTemplate_RISCV_RVV_encode_vector/) | `sq-encode` | `trace/faiss/sq-encode/013_QuantizerTemplate_Codec8bit_RISCV_RVV_Scaling0_SL_0_encode_vecto` | `bp-faiss-sq-encode-013` | `8-rvv-sq-encode-codec-template` | `8-rvv-sq-encode-codec-template.md` | 14 |
| [008_QuantizerFP16_RISCV_RVV_encode_vector](sq-encode/008_QuantizerFP16_RISCV_RVV_encode_vector/) | `sq-encode` | `trace/faiss/sq-encode/007_QuantizerFP16_SL_0_encode_vector` | `bp-faiss-sq-encode-007` | `9-rvv-sq-encode-raw-codecs` | `9-rvv-sq-encode-raw-codecs.md` | 14 |
| [009_QuantizerLloydMax_RISCV_RVV_encode_vector](sq-encode/009_QuantizerLloydMax_RISCV_RVV_encode_vector/) | `sq-encode` | —（无同符号 trace 用例） | — | `10-rvv-sq-encode-lloydmax` | `10-rvv-sq-encode-lloydmax.md` | 3 |
| [010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment](sq-decode/010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment/) | `sq-decode` | —（无同符号 trace 用例） | — | —（未进入提交包） | —（未提供） | 2 |
| [011_pop_best_rvv_MinimaxHeap](hnsw/011_pop_best_rvv_MinimaxHeap/) | `hnsw` | —（无同符号 trace 用例） | — | `patch.hnsw-popmin-on-main` | `patch.hnsw-popmin-on-main.md` | 3 |

`craft/reviews/` 为 AI 审核；`reviews/` 中 `.patch` 为复核后的补丁、`.md` 为提交用的 PR 描述。
AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

### 与 `records/mxnet` 的形态差异

| 项 | mxnet | faiss（本次整理后） |
| --- | --- | --- |
| 层级 | 3 级 `<category>/<序号>_<符号>/` | 3 级 `<测试用例>/<序号>_<函数>/`（本次由 2 级扁平升级） |
| 每条文件数 | 12–13 | 14（完整源）；3 / 2（无上游来源） |
| craft 无补丁时 | 有 `craft/` 全套 | `craft/NOTE.md`（说明为何无优化前补丁） |
| AI 审核 | `craft/reviews/opencode-patch-review.{json,md}` | 8 条为**派生件** + 2 个上游原件；详见 `PROVENANCE.md` |
| `reviews/*.md` | 复核说明 | 提交用 PR 描述（`pr-des`） |

### 说明

- `README.md` 顶部统计表与 `records/mxnet/README.md` 同口径，数值沿用原截图，未做改动。
- 本目录为**重新整理**：原 2 级扁平结构（每条 2 文件）已升级为 3 级嵌套；
  原有 `craft/patch.diff`、`craft/NOTE.md`、`review/patch-opt.diff` 三个内容全部保留，
  仅 `review/` 改为 mxnet 的复数 `reviews/`，并按 mxnet 约定重命名。
- 8 条记录的 `craft/patch.diff` 与 `yuansheng-patches` 快照的候选补丁**逐字节相同**（LF 归一后）；
  009 / 010 / 011 三条在快照与 `craft/faiss` 中均无对应候选，故只有 `craft/NOTE.md`。
- `trace/` 的 5 件产物为 Agent Debug Trace 用例的**原件拷贝**（LF 归一）；
  009 / 010 / 011 在 106 个 trace 用例中**无同符号命中**，故无 `trace/`。
- 每个 `craft/reviews/opencode-patch-review.*` 是否派生、派生自哪个字段，见 `PROVENANCE.md`。
