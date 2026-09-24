# AI 补丁审核报告

## 审核范围
- 软件 / 测试用例：openblas / sgemv_2048x2048
- 热点函数：`sgemv_n`（rank 002）
- 根因蓝图：`.yuansheng/trace/openblas/sgemv_2048x2048/002_sgemv_n/blueprint_openblas_sgemv_2048x2048_002.json`
- 补丁计划：`.yuansheng/craft/openblas/002_sgemv_n/craft/patch-plan.json`
- 补丁差异：`.yuansheng/craft/openblas/002_sgemv_n/craft/patch.diff`
- 候选补丁：`.yuansheng/craft/openblas/002_sgemv_n/craft/patch-candidate.json`（`pc-bp-openblas-sgemv-2048x2048-002`）
- 变更文件：`kernel/riscv64/gemv_n_vector.c`（`SGEMVNKERNEL`；同一源亦服务 `dgemv_n`）
- 审核类型：独立只读，基于落盘事实来源；未修改源码

## RISC-V 架构审核
机器判定 `arch-scan`：`archSpecific: true`（命中 `rvv-intrinsic`），执行架构专项审核。

1. **指令/特性在目标 ISA 内**：`vsetvlmax_e32/64m8`、`vle_v_f*m8`、`vfmacc_vf_f*m8`、`vse_v_f*m8` 均为 RVV 1.0 基座指令；目标 SpacemiT X100 RV64GCV VLEN=256。**通过**。
2. **LMUL/EMUL 一致**：LMUL 保持 m8；每次 `VLEV_FLOAT` 直接作为 FMA 第三操作数，峰值 live ≤ 2×m8 = 16 reg ≤ 32，objdump 未见向量 spill。**通过**。
3. **vsetvl/policy 与数据流一致**：主循环固定 `epr=VSETVLMAX`（`vsetvli` 移出热循环），尾部 `VSETVL(m-i)` runtime-VL 处理余块。**通过**。
4. **运行时分发**：`SGEMVNKERNEL = gemv_n_vector.c` 静态选入。**通过**。
5. **无 x86/ARM 指令混入**。**通过**。

## 审核结果
**pass**（通用 + 架构专项均通过；0 critical / 0 major）

## 发现问题
- `F1`（suggestion，性能/验证）：`kernel/riscv64/gemv_n_vector.c` — 4 列 register blocking + 固定 VL 降低 y round-trip 与地址链开销；X100 实际收益须 A/B 实测。证据：diff `+#define GEMV_N_BLOCK            4`。
- `F2`（suggestion，范围）：仅优化 `inc_y==1` 热路径，`inc_y!=1` 冷路径未改。证据：diff 中 `else` 分支未出现变更。
- `F3`（suggestion，验证）：功能 bit-exact 已由子代理以 qemu 对 15126 组 m/n/lda/inc 用例对拍 FAILS=0；建议补充极端尺寸性能表征。

**根因解决核对**：diff 将 `sgemv_n`（单精度 `gemv_n_vector.c` 实例化）内层 i-loop 改为固定 VL 主循环 + 4 列 register blocking，消除单累加器串行双 FMA、逐列 y 往返与逐迭代地址链；与 blueprint `recommendedFirstAction`（"fixed-VL main loop 重构 + 独立 accumulator + j-loop 多列扩展"）一致。**已解决**。

**约束保持核对**：
- 数值输出：列按升序消费，每 `y[i]` 的 FMA 序列与原创逐列循环相同 → bit-exact（qemu 对拍 FAILS=0，含负/非单位 inc、lda>m、m 非 VLMAX 整数倍、奇偶 n）。
- 结构保持：`n>>1`/`n&1` 余列结构、标量 prologue、`n<0` 早退、`inc_y!=1` 路径保留。
- 编译：`TARGET=RISCV64_ZVL256B` 完整构建成功，无新增 error/warning。

**产物一致性核对**：`changedFiles` 与 diff 一致；`gitDiff` 与 `patch.diff` 字节一致。

**安全核对**：无硬编码密钥、无危险命令/路径。

## 幻觉自检
- 技术精度：`[PASS]` — 指令/寄存器预算结论有 diff 与 blueprint 证据；bit-exact 由对拍支撑。
- 声明溯源：`[PASS]` — findings 的 evidence 引用 diff 原文与 blueprint 字段。
- 可解释性：`[PASS]` — 无 critical/major；suggestion 说明性质。
- 内部一致性：`[PASS]` — `reviewResult: pass` 与 findings 严重度一致。
- 安全：`[PASS]` — 无越权建议。

## 结论
补丁针对 `sgemv_n` 的固定 VL + 4 列 register blocking 重构解决蓝图根因，保持列升序数值序列与边界语义，编译通过并有大规模 bit-exact 对拍证据。**审核通过（pass）**。遗留项：X100 实机收益待 A/B 实测。
