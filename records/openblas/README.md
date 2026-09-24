# OpenBLAS：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`OpenBLAS`；目录标识：`openblas`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 44     | 61     | 47               | 7      | 7             | 7            |

## 已筛选补丁的产物归档

本分支仅整理 `add-openblas` 已筛选的 9 个openblas补丁目录，按 `records/openblas/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [001_cgemm_kernel_n](cblas_cgemm_512x512/001_cgemm_kernel_n/) | `cblas_cgemm_512x512` | `trace/openblas-2/cblas_cgemm_512x512/001_cgemm_kernel_n` | `bp-openblas-cblas_cgemm_512x512-001` |
| [001_cgemm_kernel_r](cher2k_512x512/001_cgemm_kernel_r/) | `cher2k_512x512` | `trace/openblas-2/cher2k_512x512/001_cgemm_kernel_r` | `bp-openblas-cher2k_512x512-001` |
| [001_sgemm_kernel](cblas_sgemm_512x512/001_sgemm_kernel/) | `cblas_sgemm_512x512` | `trace/openblas-2/cblas_sgemm_512x512/001_sgemm_kernel` | `bp-openblas-cblas_sgemm_512x512-001` |
| [001_zgemm_kernel_n](cblas_zgemm_512x512/001_zgemm_kernel_n/) | `cblas_zgemm_512x512` | `trace/openblas-2/cblas_zgemm_512x512/001_zgemm_kernel_n` | `bp-openblas-cblas_zgemm_512x512-001` |
| [001_zgemm_kernel_r](zher2k_512x512/001_zgemm_kernel_r/) | `zher2k_512x512` | `trace/openblas-2/zher2k_512x512/001_zgemm_kernel_r` | `bp-openblas-zher2k_512x512-001` |
| [002_saxpy_k](sger_2048x2048/002_saxpy_k/) | `sger_2048x2048` | `trace/openblas-level2/sger_2048x2048/002_saxpy_k` | `bp-openblas-sger_2048x2048-002` |
| [002_sgemv_n](sgemv_2048x2048/002_sgemv_n/) | `sgemv_2048x2048` | `trace/openblas-level2/sgemv_2048x2048/002_sgemv_n` | `bp-openblas-sgemv_2048x2048-002` |
| [003_cgemm_oncopy](cblas_cgemm_512x512/003_cgemm_oncopy/) | `cblas_cgemm_512x512` | `trace/openblas-2/cblas_cgemm_512x512/003_cgemm_oncopy` | `bp-openblas-cblas_cgemm_512x512-003` |
| [004_ctrmm_kernel_LN](ctrmm_512x512/004_ctrmm_kernel_LN/) | `ctrmm_512x512` | `trace/openblas-2/ctrmm_512x512/004_ctrmm_kernel_LN` | `bp-openblas-ctrmm_512x512-004` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。
