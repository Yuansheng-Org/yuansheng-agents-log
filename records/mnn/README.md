# MNN：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`MNN`；目录标识：`mnn`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 4      | 48     | 18               | 1      | 1             | 1            |

## 已筛选补丁的产物归档

本分支仅整理 `add-mnn` 已筛选的 1 个mnn补丁目录，按 `records/mnn/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [004_CPUConvolutionDepthwise_BasicFloatExecution_onResize_lambda](benchmark_MobileNetV2_224.mnn_1/004_CPUConvolutionDepthwise_BasicFloatExecution_onResize_lambda/) | `benchmark_MobileNetV2_224.mnn_1` | `trace/mnn/benchmark_MobileNetV2_224.mnn_1/004_CPUConvolutionDepthwise_BasicFloatExecution_onResize_lambda` | `bp-mnn-benchmark_MobileNetV2_224.mnn_1-004` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。
