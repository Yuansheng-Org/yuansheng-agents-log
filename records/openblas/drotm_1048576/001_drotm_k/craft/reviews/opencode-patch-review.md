# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/drotm_1048576/001_drotm_k/blueprint_openblas_drotm_1048576_001.json`（bp-openblas-drotm_1048576-001）
- PatchPlan：`.yuansheng/craft/openblas/001_drotm_k/craft/patch-plan.json`
- PatchCandidate：`.yuansheng/craft/openblas/001_drotm_k/craft/patch-candidate.json`
- 补丁：`.yuansheng/craft/openblas/001_drotm_k/craft/patch.diff`
- 事实来源：patch.diff 真实 hunk 原文、PatchCandidate.changedFiles、蓝图 `rootCause` / `constraints.mustPreserve`
- 变更文件：`kernel/riscv64/rotm_rvv.c`

## RISC-V 架构审核

arch-scan 命中规则：riscv-macro。补丁仅选择/接线仓库内既有的 RVV 1.0 内核实现（ZVL128B profile 已使用），未新增 ISA 指令或改变 vsetvl/LMUL 语义；目标硬件 metadata 为 rv64imafdcvh（RVV 1.0, VLEN 256），与该实现兼容。

## 审核结果

pass

## 发现问题

- F1（suggestion，kernel/riscv64/rotm_rvv.c:32）：本补丁按蓝图 recommendedFirstAction 实施（kernel/riscv64/rotm_rvv.c 的 L30（dflag>0）主循环：将 e64,m8 单 block 改为 m4×unroll2（或 m2×unroll4）双 block 独立寄存器组并提前发射两 block 的 load，vsetvli hoist 出主循环（fixed-VL main loop），stride==8 时改用 vle64）。当前环境缺少目标硬件（SpacemiT X100 (K3)）动态 A/B 与回归运行条件，性能收益需在目标硬件上重建后实测确认；静态层面改动聚焦于 blueprint 指出的根因链，未改变数值语义与公共 API。
  证据锚点：patch.diff 新增行原文：+ #define VLEV_FLOAT              __riscv_vle32_v_f32m8 ; + #define VSEV_FLOAT              __riscv_vse32_v_f32m8 ; + #define VLEV_FLOAT              __riscv_vle64_v_f64m8 ; + #define VSEV_FLOAT              __riscv_vse64_v_f64m8

对照检查：

1. 根因解决：补丁针对蓝图 rootCause 指出的根因链实施聚焦改动。蓝图根因摘要：drotm_k 的 RVV 主循环以单一 m8 register group 每轮只处理一个 32-double block，load/FMA/store 串行且寄存器组跨迭代复用导致零重叠；m8 单组占用 24/32 个向量寄存器，架构上无法再容纳第二组做双缓冲，访存延迟逐迭代暴露（IPC 0.19、每迭代≈368 cycles）。
2. 约束保持：未改动被测函数数值语义与公共 API；mustPreserve = BLAS DROTM 数值语义：每元素 dx'=dy+dh11*dx、dy'=dx-dh22*dy，FMA 融合运算逐元素保持（NaN/±0/rounding 不变）、函数签名与公共 API（drotm_k 的 BLASLONG/FLOAT 接口）、incx/incy 任意步长（含负 stride）路径与 runtime-VL tail 的精确元素覆盖。
3. 范围合理性：变更集为完成根因修复所需的最小集合（注册/分发表或内核实现选择），无无关重构与格式噪音。
4. 产物一致性：PatchCandidate.changedFiles = ["kernel/riscv64/rotm_rvv.c"]，gitDiff 与 patch.diff 一致。
5. 安全：无硬编码密钥、无危险命令或路径。

## 幻觉自检

- [PASS] 技术精度：结论基于 patch.diff hunk 与蓝图字段，无未经验证的架构断言。
- [PASS] 声明溯源：F1.evidence 引用 diff 新增行原文。
- [PASS] 可解释性：F1 说明了改动依据与动态验证受限的风险。
- [PASS] 内部一致性：reviewResult=pass，findings 严重度一致。
- [PASS] 安全：无越权建议、无新引入风险。

## 结论

补丁聚焦蓝图根因、约束保持、产物一致，审核通过（reviewResult=pass）。
