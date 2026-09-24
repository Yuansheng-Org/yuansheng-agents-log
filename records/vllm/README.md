# vLLM：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`vLLM`；目录标识：`vllm`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 2      | 37     | 2                | 1      | 1             | 原图空白／待补录     |



## 已筛选补丁的产物归档

本分支仅整理 `add-vllm` 已筛选的 1 个函数目录，按 `records/vllm/<测试用例>/<函数>/` 归档。测例和 Blueprint 来自 Agent Debug 的 Trace；原有专家复核补丁保留在 `review/`，Agent Debug 的 AI 审核另存于 `craft/reviews/`。

| 测试用例 | 函数目录 | Blueprint ID |
| --- | --- | --- |
| `vllm-bench-throughput` | [002_cpu_attention_AttentionMainLoop_cpu_attention_AttentionImpl_cpu_](vllm-bench-throughput/002_cpu_attention_AttentionMainLoop_cpu_attention_AttentionImpl_cpu_/) | `bp-vllm-vllm-bench-throughput-002` |

所有 `craft/patch.diff` 均与 Agent Debug 对应的原始 Craft 补丁逐字节一致；历史统计保持原值。AI 审核与专家复核分别保留，不自动推断编译、回归或上游接收状态。
