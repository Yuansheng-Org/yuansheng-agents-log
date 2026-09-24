# AI 补丁审核报告

## 审核范围

- **补丁候选**：`pc-bp-openblas-cblas-cgemm-512x512-003`
- **蓝图**：`bp-openblas-cblas_cgemm_512x512-003`（`.yuansheng/trace/openblas/cblas_cgemm_512x512/003_cgemm_oncopy/`）
- **改动文件**：
  - `kernel/riscv64/zgemm_ncopy_8_rvv.c`（新增，RVV 8 列复数打包器）
  - `kernel/riscv64/KERNEL.RISCV64_ZVL256B`（CGEMMONCOPY 绑定行）
- **审核依据**：`patch.diff`、蓝图 `problem/rootCause/constraints.mustPreserve/diagnosis`、仓库真实文件（`kernel/generic/zgemm_ncopy_8.c`、`kernel/riscv64/zgemm_ncopy_4_rvv.c`、`kernel/riscv64/KERNEL.RISCV64_ZVL256B`）。独立只读审核，不修改候选补丁。

## RISC-V 架构审核

- **状态**：passed
- **指令集合规**：使用 `vlseg2e32`（段 2 装载）、`vssseg8e32`（strided 段 8 存储）、`vsseg8e32/vsseg4e32/vsseg2e32`、`vsetvl_e32m1`，均在 RVV 1.0 / zve32f 范围内；`ZVL256B`（VLEN=256, e32m1 VLMAX=8）满足 `vsetvl(i)` 的尾块语义。
- **vtype 一致性**：装载/存储以 `vsetvl_e32m1` 设定 SEW=32、LMUL=m1，`e32,m1` 下 `vlseg2` 的每字段 1 寄存器、`vssseg8` 的 8 字段各 1 寄存器与数据流匹配；编译产物核对无 `e8/mf4` 等不一致残留。
- **布局等价性**：逐元素输出 `[r1,i1,r2,i2,r3,i3,r4,i4]`（组 A）与 `[r5..r8]`（组 B），以 stride=16*sizeof(FLOAT) 的 strided 段 8 存储写出，等价于 generic `zgemm_ncopy_8.c` 的 16 个连续 float；`n&4/n&2/n&1` 尾块用连续 `vsseg8/vsseg4/vsseg2`，与 generic 尾块逐个对应。`vlseg2` 的 field0/field1 与源行 `(real,imag)` 配对一致，`(r,i)` 复数语义不变。
- **编译自检**：`riscv64-linux-gnu-gcc -march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -DNN -fsyntax-only kernel/riscv64/zgemm_ncopy_8_rvv.c` → 通过；`-O2 -S` 产物确认 8×`vlseg2e32`、2×`vssseg8e32`(stride 64B)、尾块 `vsseg8/vsseg4/vsseg2`。
- **分发表**：`grep` 确认 141 行 `CGEMMONCOPY = zgemm_ncopy_8_rvv.c`，`CGEMMONCOPYOBJ = cgemm_oncopy$(TSUFFIX).$(SUFFIX)` 不变，目标对象名与调用合同不受影响。

## 审核结果

**pass**。补丁聚焦蓝图诊断的 `Load/Store Addressing-Mode Fusion`（generic 打包循环逐 store 的 base+offset 地址生成），以段指令消除冗余地址生成，同时保持 `constraints.mustPreserve` 的 packed 布局合同；无 critical/major 问题。

## 发现问题

1. **[minor / performance]** `kernel/riscv64/zgemm_ncopy_8_rvv.c`：`vssseg8e32` 在 X100 上的微架构分解（是否被拆成多条 uop）未知，收益存在不确定性；蓝图本函数仅 5 samples（函数级贡献≈1%），函数内收益上界低。证据：patch.diff `+    VSSSEG8_FLOAT(boffset, pstride, vxx8a, vl);` 与 `+    VSSSEG8_FLOAT(boffset + 8, pstride, vxx8b, vl);`；蓝图 `diagnosis.blockReason`（函数级贡献≈1%）。建议：在 X100 上以 perf annotate 复核打包循环 add/addi 形态是否消失。
2. **[suggestion / scope]** `kernel/riscv64/KERNEL.RISCV64_ZVL256B:141`：未采纳蓝图 `recommendedFirstAction` 的 split-plane 布局改造。蓝图 `diagnosis.allowAutoForwardToCraft=false` 且明确要求确认布局变更作用域（所有消费 kernel 与 M&4/M&2/M&1 尾块）；本补丁选择语义可证明等价的局部寻址改写，避免跨函数布局风险。证据：patch.diff `-CGEMMONCOPY    =  ../generic/zgemm_ncopy_$(CGEMM_UNROLL_N).c` / `+CGEMMONCOPY    =  zgemm_ncopy_8_rvv.c`。建议：split-plane 作为独立后续项评估。
3. **[minor / verification]** 缺少 X100 硬件运行验证：`cblas_cgemm_512x512` 的 GFLOPS（16.57）与 perf stat 未在本环境复测。证据：蓝图 `diagnosis.currentGaps`（无 ELF/build ISA 缺失、无 ARM 对比）。建议：在目标硬件上用 `agent1 patch_regression --case cblas_cgemm_512x512` 与 perf annotate 复核。

## 幻觉自检

- **技术精度**：PASS —— 段指令语义、LMUL/SEW、stride 字节数均与 RVV 规范及编译产物一致。
- **声明溯源**：PASS —— 所有引用均来自 patch.diff 原文或蓝图字段，未凭模型知识补全。
- **可解释性**：PASS —— 改动动机、等价性论证与风险披露完整。
- **内部一致性**：PASS —— PatchPlan 目标、说明与新文件/分发表改动一致。
- **安全**：PASS —— 未引入越界访存；`vl` 由 `vsetvl(i)` 控制，尾块按 `m/n` 余数处理；无硬编码敏感信息。

## 结论

补丁为针对 cgemm_oncopy 打包循环寻址冗余的局部等价改写，保持 packed 布局与消费 kernel 读取顺序，`patch.diff` 仅含项目代码（新增 RVV 打包器 + 分发表绑定行），编译自检通过。审核通过（pass），剩余为 X100 运行时收益的不确定性，作为 minor/suggestion 记录，不阻断交付。
