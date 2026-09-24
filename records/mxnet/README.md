# MXNet：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`MXnet`；目录标识：`mxnet`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 23     | 87     | 60               | 8      | 原图空白／待补录      | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-mxnet` 已筛选的 9 个mxnet补丁目录，按 `records/mxnet/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [001_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_fal](category-activation/001_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_fal/) | `category-activation` | `trace/mxnet/category-activation/001_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_fal` | `bp-mxnet-category-activation-001` |
| [002_MapPlan_UpSamplingNearestExp_saveto](category-misc/002_MapPlan_UpSamplingNearestExp_saveto/) | `category-misc` | `trace/mxnet/category-misc/002_MapPlan_UpSamplingNearestExp_saveto` | `bp-mxnet-opperf-category-misc-002` |
| [002_mxnet_warpctc_CpuCTC_float_log_softmax](category-loss/002_mxnet_warpctc_CpuCTC_float_log_softmax/) | `category-loss` | `trace/mxnet/category-loss/002_mxnet_warpctc_CpuCTC_float_log_softmax` | `bp-mxnet-category-loss-002` |
| [002_void_mxnet_op_im2col_float_mshadow_Stream_mshadow_cpu_float_cons](category-convolution/002_void_mxnet_op_im2col_float_mshadow_Stream_mshadow_cpu_float_cons/) | `category-convolution` | `trace/mxnet/category-convolution/002_void_mxnet_op_im2col_float_mshadow_Stream_mshadow_cpu_float_cons` | `bp-mxnet-opperf-category-convolution-002` |
| [002_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_log_softmax_fwd](category-activation/002_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_log_softmax_fwd/) | `category-activation` | `trace/mxnet/category-activation/002_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_log_softmax_fwd` | `bp-mxnet-category-activation-002` |
| [003_ConvolutionOp_cpu_float_BackwardData](category-convolution/003_ConvolutionOp_cpu_float_BackwardData/) | `category-convolution` | `trace/mxnet/category-convolution/003_ConvolutionOp_cpu_float_BackwardData` | `bp-mxnet-opperf-category-convolution-003` |
| [003_mxnet_op_mxnet_op_Kernel_mxnet_op_cumsum_forward_mshadow_cpu_Lau](category-misc/003_mxnet_op_mxnet_op_Kernel_mxnet_op_cumsum_forward_mshadow_cpu_Lau/) | `category-misc` | `trace/mxnet/category-misc/003_mxnet_op_mxnet_op_Kernel_mxnet_op_cumsum_forward_mshadow_cpu_Lau` | `bp-mxnet-opperf-category-misc-003` |
| [003_void_mxnet_op_broadcast_seq_reduce_compute_mshadow_red_minim](category-reduction/003_void_mxnet_op_broadcast_seq_reduce_compute_mshadow_red_minim/) | `category-reduction` | `trace/mxnet/category-reduction/003_003-void_mxnet_op_broadcast_seq_reduce_compute_mshadow_red_minim` | `bp-mxnet-opperf-category-reduction-003` |
| [003_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_tru](category-activation/003_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_tru/) | `category-activation` | `trace/mxnet/category-activation/003_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_tru` | `bp-mxnet-category-activation-003` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。
