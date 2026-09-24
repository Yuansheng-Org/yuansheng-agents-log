# yuansheng-agents-log

源生 Agents 测试与补丁追踪仓库。

面向 19 款软件，记录“测试用例 → 热点函数 → Trace 智能体根因诊断 → Craft智能体补丁生成 → 开源专家复核调整 → 上游社区提交/接收”的证据链。

## 1. 当前资料概述

- 资料登记日期：2026-09-08至2026-09-24。
- 18 款软件的已筛选产物已按“测试用例 → 函数”归档，共 85 个函数目录；Faiss 目前只有软件级 README。下方历史总表尚未逐行关联实际用例、函数和上游链接，具体已归档内容见各软件 README。

19款软件的测试原始记录为：测试用例 **545**、分析热点函数 **2229**、Craft成功生成补丁 **534**。

## 2. 19 款软件追踪总表

本表按“每个开源专家复核有效补丁一行”登记。OpenCV 按已核实的 10 份补丁登记后，现预列 **89 行**，另预留 **30 行**，合计 **119 行**；预留行不计入有效补丁数量。

- 序号 1–89 沿用既有总表的软件分配，并将 OpenCV 调整为已核实的 10 份补丁；已核实的行关联具体补丁文件、用例、函数、Trace 日志和复核意见，其余行待逐项关联。
- 序号 90–119 为待分配预留行，补充真实补丁及所属软件后再纳入统计。
- 软件汇总数量保留在各软件页面；逐补丁的上游提交/接收状态依据实际链接填写。
- OpenCV 的 10 条已提供 PR 与本仓库 10 份 `reviews/*.patch` 逐字节一致，对应下表第 44–53 行。

| 序号  | 被测软件                                                   | 测试用例 | 热点函数 | Trace 日志 | Craft 补丁生成 | 开源专家复核有效补丁  | 上游社区提交   |
| ---:| ------------------------------------------------------ | ---- | ---- | -------- | ---------- | ----------- | -------- |
| 1   | [Redis](records/redis/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 2   | [Redis](records/redis/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 3   | [MariaDB](records/mariadb/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 4   | [MariaDB](records/mariadb/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 5   | [OpenJDK](records/openjdk/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 6   | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 7   | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 8   | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 9   | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 10  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 11  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 12  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 13  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 14  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 15  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 16  | [Faiss](records/faiss/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 17  | [vLLM](records/vllm/README.md)                         | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 18  | [OpenSSL](records/openssl/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 19  | [OpenSSL](records/openssl/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 20  | [OpenSSL](records/openssl/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 21  | [ONNX Runtime](records/onnxruntime/README.md)          | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 22  | [ONNX Runtime](records/onnxruntime/README.md)          | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 23  | [MNN](records/mnn/README.md)                           | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 24  | [PostgreSQL](records/postgresql/README.md)             | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 25  | [PostgreSQL](records/postgresql/README.md)             | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 26  | [Go](records/golang/README.md)                         | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 27  | [Go](records/golang/README.md)                         | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 28  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 29  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 30  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 31  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 32  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 33  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 34  | [OpenBLAS](records/openblas/README.md)                 | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 35  | [glibc](records/glibc/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 36  | [glibc](records/glibc/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 37  | [glibc](records/glibc/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 38  | [glibc](records/glibc/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 39  | [pixman](records/pixman/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 40  | [pixman](records/pixman/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 41  | [pixman](records/pixman/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 42  | [pixman](records/pixman/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 43  | [pixman](records/pixman/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 44 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [002_cv_rvv_hal_core_dft](records/opencv/core/002_cv_rvv_hal_core_dft/) | [诊断日志](records/opencv/core/002_cv_rvv_hal_core_dft/trace/diaglog_opencv_core_002.md) | [原始补丁](records/opencv/core/002_cv_rvv_hal_core_dft/craft/patch.diff) | [复核补丁](records/opencv/core/002_cv_rvv_hal_core_dft/reviews/002_cv_rvv_hal_core_dft.patch)、[复核说明](records/opencv/core/002_cv_rvv_hal_core_dft/reviews/002_cv_rvv_hal_core_dft.md) | [PR #29906](https://github.com/opencv/opencv/pull/29906) |
| 45 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi](records/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/) | [诊断日志](records/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/trace/diaglog_opencv_core_026.md) | [原始补丁](records/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/craft/patch.diff) | [复核补丁](records/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/reviews/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi.patch)、[复核说明](records/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/reviews/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi.md) | [PR #29917](https://github.com/opencv/opencv/pull/29917) |
| 46 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [030_reduceRowSum_8u32s](records/opencv/core/030_reduceRowSum_8u32s/) | [诊断日志](records/opencv/core/030_reduceRowSum_8u32s/trace/diaglog_opencv_core_030.md) | [原始补丁](records/opencv/core/030_reduceRowSum_8u32s/craft/patch.diff) | [复核补丁](records/opencv/core/030_reduceRowSum_8u32s/reviews/030_reduceRowSum_8u32s.patch)、[复核说明](records/opencv/core/030_reduceRowSum_8u32s/reviews/030_reduceRowSum_8u32s.md) | [PR #29923](https://github.com/opencv/opencv/pull/29923) |
| 47 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo](records/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/) | [诊断日志](records/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/trace/diaglog_opencv_core_027.md) | [原始补丁](records/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/craft/patch.diff) | [复核补丁](records/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/reviews/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo.patch)、[复核说明](records/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/reviews/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo.md) | [PR #29967](https://github.com/opencv/opencv/pull/29967) |
| 48 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_](records/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/) | [诊断日志](records/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/trace/diaglog_opencv_core_015.md) | [原始补丁](records/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/craft/patch.diff) | [复核补丁](records/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/reviews/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_.patch)、[复核说明](records/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/reviews/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_.md) | [PR #29930](https://github.com/opencv/opencv/pull/29930) |
| 49 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [008_cv_hal_normL2Sqr_float_const_float_const_int](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/) | [诊断日志](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/trace/diaglog_opencv_core_008.md) | [原始补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/craft/patch.diff) | [复核补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int.patch)、[复核说明](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int.md) | [PR #29944](https://github.com/opencv/opencv/pull/29944) |
| 50 | [OpenCV](records/opencv/README.md) | [video](records/opencv/video/) | [010_cv_detail_LKTrackerInvoker_operator](records/opencv/video/010_cv_detail_LKTrackerInvoker_operator/) | [诊断日志](records/opencv/video/010_cv_detail_LKTrackerInvoker_operator/trace/diaglog_opencv_video_010.md) | [原始补丁](records/opencv/video/010_cv_detail_LKTrackerInvoker_operator/craft/patch.diff) | [复核补丁](records/opencv/video/010_cv_detail_LKTrackerInvoker_operator/reviews/010_cv_detail_LKTrackerInvoker_operator.patch)、[复核说明](records/opencv/video/010_cv_detail_LKTrackerInvoker_operator/reviews/010_cv_detail_LKTrackerInvoker_operator.md) | [PR #29947](https://github.com/opencv/opencv/pull/29947) |
| 51 | [OpenCV](records/opencv/README.md) | [video](records/opencv/video/) | [009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator](records/opencv/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/) | [诊断日志](records/opencv/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/trace/diaglog_opencv_video_009.md) | [原始补丁](records/opencv/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/craft/patch.diff) | [复核补丁](records/opencv/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/reviews/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator.patch)、[复核说明](records/opencv/video/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator/reviews/009_DISOpticalFlowImpl_PatchInverseSearch_ParBody_operator.md) | [PR #29961](https://github.com/opencv/opencv/pull/29961) |
| 52 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [008_cv_hal_normL2Sqr_float_const_float_const_int_2](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/) | [诊断日志](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/trace/diaglog_opencv_core_008.md) | [原始补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/craft/patch.diff) | [复核补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int_2.patch)、[复核说明](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_2/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int_2.md) | [PR #30000](https://github.com/opencv/opencv/pull/30000) |
| 53 | [OpenCV](records/opencv/README.md) | [core](records/opencv/core/) | [008_cv_hal_normL2Sqr_float_const_float_const_int_1](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/) | [诊断日志](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/trace/diaglog_opencv_core_008.md) | [原始补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/craft/patch.diff) | [复核补丁](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int_1.patch)、[复核说明](records/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int_1/reviews/008_cv_hal_normL2Sqr_float_const_float_const_int_1.md) | [PR #30018](https://github.com/opencv/opencv/pull/30018) |
| 54  | [PyTorch](records/pytorch/README.md)                   | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 55  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 56  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 57  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 58  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 59  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 60  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 61  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 62  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 63  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 64  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 65  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 66  | [FFmpeg/H264/H265](records/ffmpeg-h264-h265/README.md) | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 67  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 68  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 69  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 70  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 71  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 72  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 73  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 74  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 75  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 76  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 77  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 78  | [oneDNN](records/onednn/README.md)                     | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 79  | [Caffe](records/caffe/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 80  | [Caffe](records/caffe/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 81  | [Caffe](records/caffe/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 82  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 83  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 84  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 85  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 86  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 87  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 88  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 89  | [MXNet](records/mxnet/README.md)                       | 待关联  | 待关联  | 待归档      | 原始补丁待归档    | 复核后补丁/意见待归档 | 链接/状态待补录 |
| 90  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 91  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 92  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 93  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 94  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 95  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 96  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 97  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 98  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 99  | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 100 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 101 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 102 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 103 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 104 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 105 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 106 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 107 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 108 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 109 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 110 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 111 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 112 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 113 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 114 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 115 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 116 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 117 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 118 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |
| 119 | 待分配（预留）                                                | 待补充  | 待补充  | 待补充      | 待补充        | 预留，不计入有效数   | 待补充      |

## 3. 已筛选产物索引

下表按当前归档目录计数；它与历史总表中的“专家复核有效补丁”统计口径不同。Pixman 和 OpenBLAS 整理分支分别保留 5 项、7 项，均少于对应 `add-*` 分支的 6 项、9 项。

| 软件 | 已归档函数目录 | 软件记录 |
| --- | ---: | --- |
| Caffe | 3 | [记录](records/caffe/README.md) |
| FFmpeg/H264/H265 | 11 | [记录](records/ffmpeg-h264-h265/README.md) |
| Glibc | 4 | [记录](records/glibc/README.md) |
| Go | 3 | [记录](records/golang/README.md) |
| MariaDB | 1 | [记录](records/mariadb/README.md) |
| MNN | 1 | [记录](records/mnn/README.md) |
| MXNet | 15 | [记录](records/mxnet/README.md) |
| oneDNN | 12 | [记录](records/onednn/README.md) |
| ONNX Runtime | 2 | [记录](records/onnxruntime/README.md) |
| OpenBLAS | 7 | [记录](records/openblas/README.md) |
| OpenCV | 10 | [记录](records/opencv/README.md) |
| OpenJDK | 1 | [记录](records/openjdk/README.md) |
| OpenSSL | 3 | [记录](records/openssl/README.md) |
| Pixman | 5 | [记录](records/pixman/README.md) |
| PostgreSQL | 2 | [记录](records/postgresql/README.md) |
| PyTorch | 1 | [记录](records/pytorch/README.md) |
| Redis | 3 | [记录](records/redis/README.md) |
| vLLM | 1 | [记录](records/vllm/README.md) |
| Faiss | 0 | [记录](records/faiss/README.md) |

## 4. 日志目录与记录模板

```text
README.md                              # 本说明及19软件历史汇总索引
records/<software>/README.md           # 软件级索引和统计口径
records/<software>/<testcase>/<function>/
  trace/                               # 原始 Trace 证据或 Blueprint bundle
  craft/                               # 原始 Craft 补丁、协议和 AI review（来源存在时）
  review/ 或 reviews/                  # 已筛选的专家复核结果
  provenance/                          # 补丁版本不同时保留的旧版文件（来源存在时）
```
