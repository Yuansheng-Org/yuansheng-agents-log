# OpenBLAS：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`OpenBLAS`；目录标识：`openblas`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 44     | 61     | 47               | 7      | 7             | 7            |

## 已筛选补丁的产物归档

本分支仅整理 `add-openblas` 已筛选的 8 个openblas补丁目录，按 `records/openblas/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。其中 `001_drotm_k` 为本次新增，其原有 `craft/patch.diff`、`review/patch.diff` 和 `review/performance-comparison.md` 一并迁入；该项现已提交上游 [PR #6070](https://github.com/OpenMathLib/OpenBLAS/pull/6070)（open）。线上提交版本给 `dflag>0`、`dflag==0`、`dflag<0` 三个分支均加了单位步长快路径，是本目录复核补丁（仅覆盖 `dflag>0`）的超集；本目录按现状保留复核补丁原件，未随线上版本更新。PR 尚未合入，上方截图历史汇总保持原值。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [001_cgemm_kernel_n](cblas_cgemm_512x512/001_cgemm_kernel_n/) | `cblas_cgemm_512x512` | `trace/openblas-2/cblas_cgemm_512x512/001_cgemm_kernel_n` | `bp-openblas-cblas_cgemm_512x512-001` |
| [001_sgemm_kernel](cblas_sgemm_512x512/001_sgemm_kernel/) | `cblas_sgemm_512x512` | `trace/openblas-2/cblas_sgemm_512x512/001_sgemm_kernel` | `bp-openblas-cblas_sgemm_512x512-001` |
| [001_zgemm_kernel_n](cblas_zgemm_512x512/001_zgemm_kernel_n/) | `cblas_zgemm_512x512` | `trace/openblas-2/cblas_zgemm_512x512/001_zgemm_kernel_n` | `bp-openblas-cblas_zgemm_512x512-001` |
| [002_saxpy_k](sger_2048x2048/002_saxpy_k/) | `sger_2048x2048` | `trace/openblas-level2/sger_2048x2048/002_saxpy_k` | `bp-openblas-sger_2048x2048-002` |
| [002_sgemv_n](sgemv_2048x2048/002_sgemv_n/) | `sgemv_2048x2048` | `trace/openblas-level2/sgemv_2048x2048/002_sgemv_n` | `bp-openblas-sgemv_2048x2048-002` |
| [003_cgemm_oncopy](cblas_cgemm_512x512/003_cgemm_oncopy/) | `cblas_cgemm_512x512` | `trace/openblas-2/cblas_cgemm_512x512/003_cgemm_oncopy` | `bp-openblas-cblas_cgemm_512x512-003` |
| [004_ctrmm_kernel_LN](ctrmm_512x512/004_ctrmm_kernel_LN/) | `ctrmm_512x512` | `trace/openblas-2/ctrmm_512x512/004_ctrmm_kernel_LN` | `bp-openblas-ctrmm_512x512-004` |
| [001_drotm_k](drotm_1048576/001_drotm_k/) | `drotm_1048576` | `trace/openblas-level1/drotm_1048576/001_drotm_k` | `bp-openblas-drotm_1048576-001` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。
