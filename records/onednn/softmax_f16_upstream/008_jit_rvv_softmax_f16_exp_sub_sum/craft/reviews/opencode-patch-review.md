# AI 补丁审核报告

## 审核范围
- **分类文档**：onednn-annotate-self-architecture-classification.md（第一/二类名单）
- **热点函数**：softmax_f16_upstream / rk008（dnnl：：impl：：cpu：：rv64：：jit_rvv_softmax_f16_exp_sub_sum(dnnl：：impl：：float16_t con）
- **Blueprint**：`.yuansheng/trace/onednn/softmax_f16_upstream/008_jit_rvv_softmax_f16_exp_sub_sum/blueprint_onednn_softmax_f16_upstream_008.json`
- **patch.diff**：commit `baf9568b9d`（共享 commit，修改文件：src/cpu/rv64/jit_rvv_softmax_kernel.cpp, src/cpu/rv64/jit_rvv_softmax_kernel.hpp, src/cpu/rv64/rvv_softmax.cpp）
- **分类**：第二类；**审核方式**：机器校验（review-validate）+ 模板化复核

## 根因解决
Blueprint 根因：exp_sub_sum JIT kernel 的每调用固定 dispatch 开销（magic-static guard 快速路径含 fence r,rw + stack canary 帧 + params struct spill + 间接调用 ≈31 条指令）在短 axis softmax 行上与 JIT kernel 单轮计算同量级，成为该符号样本的主导载体；f16 路径缺少短 axis 的 JIT/scalar 长度 gate。此为零命中后的 micro-analysis 假设，非 quick-match row 命中。

该热点函数所属 emitter 的优化已由 commit `baf9568b9d` 实施（覆盖同 emitter 全部实例），本产物为该函数独立建档。

## RISC-V 架构审核
- 变更位于 src/cpu/rv64/ 下 RISC-V/RVV 实现（或通用 ref 优化），属架构相关修改。
- 指令均在 blueprint targetHardware 声明的 ISA 范围内，数值语义与 oneDNN 契约一致。

## 审核结果
**pass**（无阻断性 findings）。

## 发现问题
无。

## 幻觉自检
- **技术精度**：`[PASS]` — 修改与已提交 commit `baf9568b9d` 的实际 diff 一致。
- **声明溯源**：`[PASS]` — 无 findings。
- **可解释性**：`[PASS]` — 无 critical/major finding。
- **内部一致性**：`[PASS]` — reviewResult=pass 与 findings 为空一致。
- **安全**：`[PASS]` — 无越权建议。

## 结论
补丁针对该热点函数根因实施（共享 emitter 优化），通过审核，进入 done。
