# OpenCV：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`OpenCV`；目录标识：`opencv`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 12     | 30     | 21               | 11     | 11            | 7            |

## 已筛选补丁的产物归档

本分支整理 `add-opencv` 已筛选的 10 个目录，按 `records/opencv/<测试用例>/<函数目录>/` 归档。测例和 Blueprint 取自 Agent Debug；原有专家复核补丁保留在 `reviews/`，Agent Debug 的 AI 审核另存于 `craft/reviews/`。

| 测试用例 | 函数目录 | Agent Debug 来源 | Blueprint ID |
| --- | --- | --- | --- |
| `core` | [002_cv_rvv_hal_core_dft](core/002_cv_rvv_hal_core_dft/) | `trace/opencv/core/002_cv_rvv_hal_core_dft` | `bp-opencv-core-002` |
| `core` | [008_cv_hal_normL2Sqr_float_const_float_const_int](core/008_cv_hal_normL2Sqr_float_const_float_const_int/) | `trace/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int` | `bp-opencv-core-008` |
| `core` | [008_cv_hal_normL2Sqr_float_const_float_const_int_1](core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/) | `trace/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int` | `bp-opencv-core-008` |
| `core` | [008_cv_hal_normL2Sqr_float_const_float_const_int_2](core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/) | `trace/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int` | `bp-opencv-core-008` |
| `video` | [009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator](video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/) | `trace/opencv-2/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator` | `bp-opencv-video-009` |
| `video` | [010_cv_detail_LKTrackerInvoker_operator](video/010_cv_detail_LKTrackerInvoker_operator/) | `trace/opencv-2/video/010_cv_detail_LKTrackerInvoker_operator` | `bp-opencv-video-010` |
| `core` | [015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_](core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/) | `trace/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_` | `bp-opencv-core-015` |
| `core` | [026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi](core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/) | `trace/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi` | `bp-opencv-core-026` |
| `core` | [027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo](core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/) | `trace/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo` | `bp-opencv-core-027` |
| `core` | [030_reduceRowSum_8u32s](core/030_reduceRowSum_8u32s/) | `trace/opencv/core/030_reduceRowSum_8u32s` | `bp-opencv-core-030` |

`008_cv_hal_normL2Sqr_float_const_float_const_int_1` 和 `_2` 是同一原始 Craft 补丁的两份独立专家复核结果。它们与不带后缀的目录共用 `opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int` 的 Trace、Plan、Candidate 和 AI 审核；这些源文件分别复制进三个目录，原有各自的 `reviews/` 文件保持独立。

`009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator` 和 `010_cv_detail_LKTrackerInvoker_operator` 的原始产物位于 Agent Debug 的 `opencv-2/video`，其余记录来自 `opencv/core`。10 份 `craft/patch.diff` 均与对应源目录逐字节一致；历史统计保持原值，AI 审核不自动证明编译、回归或上游接收。
