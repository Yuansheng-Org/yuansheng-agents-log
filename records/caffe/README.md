# Caffe：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`caffe`；目录标识：`caffe`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 4      | 16     | 13               | 3      | 原图空白／待补录      | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-caffe` 已筛选的 3 个caffe补丁目录，按 `records/caffe/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [004_im2col_cpu](bvlc_alexnet/004_im2col_cpu/) | `bvlc_alexnet` | `trace/caffe/bvlc_alexnet/004_im2col_cpu` | `bp-caffe-bvlc_alexnet-004` |
| [006_caffe_PoolingLayer_float_Forward_cpu](bvlc_alexnet/006_caffe_PoolingLayer_float_Forward_cpu/) | `bvlc_alexnet` | `trace/caffe/bvlc_alexnet/006_caffe_PoolingLayer_float_Forward_cpu` | `bp-caffe-bvlc_alexnet-006` |
| [011_void_vPowx_float_int_float_const_float_float](bvlc_alexnet/011_void_vPowx_float_int_float_const_float_float/) | `bvlc_alexnet` | `trace/caffe/bvlc_alexnet/011_void_vPowx_float_int_float_const_float_float` | `bp-caffe-bvlc_alexnet-011` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

`011_void_vPowx_float_int_float_const_float_float` 的 Trace Blueprint ID 为 `bp-caffe-bvlc_alexnet-011`，Craft Plan 所引 ID 为 `bp-caffe-bvlc-alexnet-011`。两者拼写不一致；该项通过函数目录和原有 Craft diff 的字节匹配确定来源，原始 JSON 未改写。
