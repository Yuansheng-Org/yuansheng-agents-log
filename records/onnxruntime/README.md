# ONNX Runtime：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`ONNX Runtime`；目录标识：`onnxruntime`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 9      | 48     | 7                | 2      | 2             | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-onnxruntime` 已筛选的 2 个onnxruntime补丁目录，按 `records/onnxruntime/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_](onnxruntime_perf_test_conv_relu_t1/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_/) | `onnxruntime_perf_test_conv_relu_t1` | `trace/onnxruntime/onnxruntime_perf_test_conv_relu_t1/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_` | `bp-onnxruntime-onnxruntime_perf_test_conv_relu_t1-002` |
| [002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt](onnxruntime_bert_perf_test_bert_like_seq128_t4/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt/) | `onnxruntime_bert_perf_test_bert_like_seq128_t4` | `trace/onnxruntime/onnxruntime_bert_perf_test_bert_like_seq128_t4/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt` | `bp-onnxruntime-onnxruntime_bert_perf_test_bert_like_seq128_t4-002` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

`002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_` 的 Candidate ID `pc-bp-onnxruntime-onnxruntime-perf-test-conv-relu-t1-002` 与 AI Review 引用 `pc-bp-onnxruntime-onnxruntime_perf_test_conv_relu_t1-002` 不一致；原始文件已保留。

`002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt` 的 Candidate ID `pc-bp-onnxruntime-onnxruntime-bert-perf-test-bert-like-seq128-t4-002` 与 AI Review 引用 `pc-bp-onnxruntime-onnxruntime_bert_perf_test_bert_like_seq128_t4-002` 不一致；原始文件已保留。
