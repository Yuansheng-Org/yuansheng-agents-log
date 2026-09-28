# Faiss：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`Faiss`；目录标识：`faiss`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 6      | 106    | 20               | 11     | 5             | 原图空白／待补录     |

> 上表为**原截图历史汇总**，数值保持原值。其中“开源专家复核后补丁数量 11”含本次已移除的 009 / 010 / 011 三条；本目录当前归档 **8** 条（见下节）。

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

`craft/reviews/` 为 AI 审核；`reviews/` 中 `.patch` 为复核后的补丁、`.md` 为提交用的 PR 描述。
AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

上表“提交包 / PR 描述”列记录的是**首次提交**（PR #5602 / #5603 / #5604 / #5631，均已关闭）。这四条随后以补丁逐字节相同的内容**重新提交**为 PR #5638（记录 001）/ #5639（记录 002）/ #5640（记录 003）/ #5641（记录 004）。其中记录 003 的 PR #5640 已由 Meta codesync 导入并合并进 faiss `main`（squash `f323c18d`，`behind_by == 0`）；其余三条仍为 open。当前状态见根 `README.md` 追踪总表。

## `related/`：追踪补充信息（非同补丁）

8 条记录各自新增一个可选成员 `related/`，收纳**方向一致但并非同一条补丁**的上游产物：

| 前缀 | 内容 | 等级 | 覆盖记录 |
| --- | --- | --- | --- |
| `craft/related/snapshot/<case>/blueprint/` | 快照诊断工件（`blueprint.json`/`diagnosis.md`/`claim-to-evidence.json`/校验件） | **原件**（同次采集） | 001–008 |
| `trace/related/snapshot/<case>/evidence/` | 快照采集证据（`annotate.txt`/`metadata.json`/`perf-stat.txt`/`hardware-profile.json`） | **原件**（同次采集） | 001–008 |
| `craft/related/faiss/<dir>/{craft,reviews}/` | `yuansheng-agent-debug` 的 `craft/faiss` 整块 20 目录 | **原件**（C1/C2/C3） | 仅 001 |

**这些不是本记录的 craft 证据。** 关系等级定义：

| 等级 | 判据 |
| --- | --- |
| **C1** | 同一 `blueprintId`（即同一 testcase、同一次分析） |
| **C2** | 同一蓝图命名空间（`bp-faiss-<category>-` 前缀相同） |
| **C3** | 只共享被改文件（如 `sq-rvv.cpp`） |
| **T2** | 同一 trace 命名空间 + 同一目标动词 |
| **T3** | 同一代码族（如堆/结果集），命名空间不同 |

每条记录的关系判定与新增文件清单见其 `SOURCE.md`。

### 为什么 `craft/faiss` 整块挂到 001 而不拆分

`craft/faiss` 的 20 个目录**与 `trace/faiss` 用例目录名 20/20 精确配对**，其中
`013_DCTemplate_Quantizer8bitDirectSigned_6_SimilarityIP_6_SL_0_query` 与记录 001
**共用同一 `blueprintId`（`bp-faiss-sq-distance-013`）和同一 testcase**。
但 17 个 `DCTemplate` 目录的 `patch.diff` LF 归一后是**同一份累积补丁**
（触及 `Clustering.cpp` + `sq-rvv.cpp` + `distances_dispatch.h` + `distances_rvv.cpp`），
与 5 条记录（001 / 002 / 003 / 005 / 006）都只共享 `sq-rvv.cpp`（C3）。若拆分到各条，同一正文会伪装成"各条各自的 craft"，
故以 20 目录为不可分割单位挂在 C1 命中的 001 下。

### 与 `records/mxnet` 的形态差异

| 项 | mxnet | faiss（本次整理后） |
| --- | --- | --- |
| 层级 | 3 级 `<category>/<序号>_<符号>/` | 3 级 `<测试用例>/<序号>_<函数>/`（本次由 2 级扁平升级） |
| 每条文件数 | 12–13 | 14（完整源，不含 `SOURCE.md`）；另各有 `related/` 补充区 |
| craft 无补丁时 | 有 `craft/` 全套 | 现无此类记录（原 009/010/011 曾用 `craft/NOTE.md`，已移除） |
| AI 审核 | `craft/reviews/opencode-patch-review.{json,md}` | 每条 4 件：上游原件 2（`semantic-validation.json` / `machine-validation.json`）+ 派生件 2（`opencode-patch-review.{json,md}`）；详见 `PROVENANCE.md` |
| `reviews/*.md` | 复核说明 | 提交用 PR 描述（`pr-des`） |
| 补充追踪信息 | 无该层 | `related/` 子目录，**非同补丁**，逐条 `SOURCE.md` |

### 说明

- `README.md` 顶部统计表与 `records/mxnet/README.md` 同口径，数值沿用原截图，未做改动。
- 本目录为**重新整理**：原 2 级扁平结构（每条 2 文件）已升级为 3 级嵌套；
  原有 `craft/patch.diff`、`review/patch-opt.diff` 等内容全部保留，
  仅 `review/` 改为 mxnet 的复数 `reviews/`，并按 mxnet 约定重命名。
  （原 `craft/NOTE.md` 随 009/010/011 三条一并移除，见文末。）
- 8 条记录的 `craft/patch.diff` 与 `yuansheng-patches` 快照的候选补丁**逐字节相同**（LF 归一后），均有完整 `craft/` 产物。
- `trace/` 的 5 件产物为 Agent Debug Trace 用例的**原件拷贝**（LF 归一），8 条记录均有对应诊断日志。
- 每个 `craft/reviews/opencode-patch-review.*` 是否派生、派生自哪个字段，见 `PROVENANCE.md`。
- **已移除记录（本目录 11 → 8）**：原 `009_QuantizerLloydMax_RISCV_RVV_encode_vector`（`sq-encode`）、`010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment`（`sq-decode`）、`011_pop_best_rvv_MinimaxHeap`（`hnsw`）三条**无 Agent Debug Trace 诊断日志、亦无优化前 Craft 原始补丁**（仅有复核后的 `reviews/*.patch` 与 `craft/NOTE.md`），已从本目录删除；其复核补丁仍保留在桌面提交包中。连同其 `trace/related/` 同族补录一并撤销。
