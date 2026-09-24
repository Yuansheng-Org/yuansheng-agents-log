# oneDNN：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`oneDNN`；目录标识：`onednn`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 41     | 269    | 116              | 12     | 12            | 7            |

## 已筛选补丁的产物归档

本分支仅整理 `add-onednn` 已筛选的 12 个函数目录，按 `records/onednn/<测试用例>/<函数>/` 归档。来源为 `yuansheng-patches/onednn-d22de940f301e97591e04a2cc6f0010c52109ac7` 的对应批次和函数目录；原有 `craft/patch.diff` 与来源候选补丁逐字节一致。

| 测试用例 | 函数目录 | 来源批次 | 原有专家复核文件 |
| --- | --- | --- | --- |
| `bnorm_f16_upstream` | [002_ncsp_batch_normalization_fwd_t_execute_forward_lambda](bnorm_f16_upstream/002_ncsp_batch_normalization_fwd_t_execute_forward_lambda/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `conv_f16_upstream` | [001_std_Function_handler_void_long_long_long_long_long_long_dnnl_imp](conv_f16_upstream/001_std_Function_handler_void_long_long_long_long_long_long_dnnl_imp/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `conv_f32_upstream` | [jit_uni_reorder_kernel_f32.6](conv_f32_upstream/jit_uni_reorder_kernel_f32.6/) | `a670dab844e24e9b9da4d4670841a0fd` | 有 |
| `eltwise_f16_upstream` | [jit_uni_kernel.0](eltwise_f16_upstream/jit_uni_kernel.0/) | `a670dab844e24e9b9da4d4670841a0fd` | 有 |
| `eltwise_f16_upstream` | [jit_uni_kernel.2](eltwise_f16_upstream/jit_uni_kernel.2/) | `a670dab844e24e9b9da4d4670841a0fd` | 有 |
| `ip_f16_upstream` | [001_001-std_Function_handler_void_long_long_dnnl_impl_cpu_ref_inner_](ip_f16_upstream/001_001-std_Function_handler_void_long_long_dnnl_impl_cpu_ref_inner_/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `lnorm_f16_upstream` | [006_jit_rvv_layernorm_f16_fused_kernel_t.3](lnorm_f16_upstream/006_jit_rvv_layernorm_f16_fused_kernel_t.3/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `pool_f16_upstream` | [001_jit_uni_pool_ncsp_kernel_t.8](pool_f16_upstream/001_jit_uni_pool_ncsp_kernel_t.8/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `rnn_f16_upstream` | [001_ref_matmul_t_execute_ref_lambda_1_operator](rnn_f16_upstream/001_ref_matmul_t_execute_ref_lambda_1_operator/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `softmax_f16_upstream` | [008_jit_rvv_softmax_f16_exp_sub_sum](softmax_f16_upstream/008_jit_rvv_softmax_f16_exp_sub_sum/) | `d9d90bec5aef8f6fe89ddee57a93e5ff` | 有 |
| `softmax_f16_upstream` | [dnnl_impl_cpu_rv64_jit_rvv_softmax_f16_scatter_dnnl_impl_float16_t_const_dnnl_impl_float16_t](softmax_f16_upstream/dnnl_impl_cpu_rv64_jit_rvv_softmax_f16_scatter_dnnl_impl_float16_t_const_dnnl_impl_float16_t/) | `a670dab844e24e9b9da4d4670841a0fd` | 无 |
| `softmax_f32_upstream` | [dnnl_impl_cpu_rv64_anonymous_namespace_compute_softmax_f32_rvv_float_const_float_long_bool_7bb41bca](softmax_f32_upstream/dnnl_impl_cpu_rv64_anonymous_namespace_compute_softmax_f32_rvv_float_const_float_long_bool_7bb41bca/) | `a670dab844e24e9b9da4d4670841a0fd` | 无 |

`trace/` 保留来源 `blueprint/` 的全部内容；`craft/` 保留原有 diff，补入来源 `candidate-*.patch` 以及来源存在的 Plan、Candidate 和 AI 审核文件。`a670dab...` 批次的 5 项只有 Blueprint bundle 和候选补丁，没有对应协议文件，本次未补造。原有 `review/` 保持不变。来源候选仅是候选交付物，不能据此推断已通过构建、回归或上游接收；历史统计保持原值。
