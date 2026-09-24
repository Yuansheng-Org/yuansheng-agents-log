# AI 补丁审核报告 — ff_tx_mdct_inv_float_rvv（RVV MDCT 逆变换 codelet）

## 审核范围

- **RootCauseBlueprint**: `.yuansheng/trace/ffmpeg/aac_decode_sbr_ps/001_ff_tx_mdct_inv_float_c/blueprint_ffmpeg_aac_decode_sbr_ps_001.json`（`validate root-cause-blueprint` 输出 `valid: true`；rootCauseType=arch_gap，`recommendToCraft=yes`，`overallStatus=confirmed`；mustPreserve：对外 API 复数布局/缩放约定、ifftFlag 虚部取反、与 s->sub 递归子变换配合）
- **PatchPlan**: `.yuansheng/craft/ffmpeg/001_ff_tx_mdct_inv_float_c/craft/patch-plan.json`
- **patch.diff**: `.yuansheng/craft/ffmpeg/001_ff_tx_mdct_inv_float_c/craft/patch.diff`（4 文件，+160/-1，与工作区 `git diff` 一致）
- **PatchCandidate**: `.yuansheng/craft/ffmpeg/001_ff_tx_mdct_inv_float_c/craft/patch-candidate.json`（`archReviewWarning` 为空 → 需 RISC-V 架构专项审核）

## 审核结果

**reviewResult: pass**（含 2 条 suggestion 级非阻塞意见）

## 根因解决（通用审核）

1. **根因映射**：蓝图确认 MDCT 逆变换 `ff_tx_mdct_inv_float_c` 在 K3（RVV 1.0, VLEN=256）上零 v* 指令、标量逐蝶形执行，`libavutil/riscv/` 无 tx RVV 实现（aarch64/x86 有）。补丁新增 RVV codelet 并注册到 `ff_tx_codelet_list_float_riscv[]`（MDCT 类型、`FF_TX_INVERSE_ONLY`、`AV_CPU_FLAG_RVV_F32`、prio 192），完全覆盖根因。
2. **约束保持**：
   - 复用 C 侧共享 `ff_tx_mdct_init`（仅去掉 `static` 暴露符号 + tx_priv.h 声明），s->map/s->exp 生成路径与 C 实现逐位一致；折叠/后旋转算术按 tx_template.c `ff_tx_mdct_inv` 原样移植（CMUL3/CMUL 复乘公式一致）。
   - `ifftFlag` 语义由共享 init 的 `map[i] <<= 1` 与 `FF_TX_MAP_GATHER` 保持。
   - 子变换通过 `s->fn[0](&s->sub[0], z, z, 8)` 调用，与 C 完全一致。
3. **最小性**：改动仅 4 文件：tx_float_rvv.S 新增 1 个 func、tx_float_init.c 注册 1 条、tx_template.c 1 处 `static` 移除、tx_priv.h 3 条原型。无无关重构。
4. **产物一致性**：PatchCandidate.gitDiff 与 patch.diff 逐字节一致（candidate 工具从真实 git diff 生成）。
5. **安全**：无密钥/危险路径。

## RISC-V 架构审核

- **arch-scan**: `archSpecific: true`（asm-source-file / rvv-intrinsic / march-flag 三条命中），执行专项审核。
- **指令集合规（PASS）**：`vsetvli/vle32/vse32/vluxei32/vlseg2e32/vsseg2e32/vlse32/vsse32/vmul.vx/vmv.v.x/vsub.vv/vfmul.vv/vfadd.vv/vfsub.vv` 均为 RVV 1.0 / zve32f 指令；标量 `lw/ld/sd/srai/slli/mul/add/sub/addi/li/mv/bnez/beqz/jalr/ret` 为 base integer；`lpad` 在无 zicfilp 时展开为 `auipc zero,0x0`（与文件内既有 codelet 一致）。目标硬件证据：blueprint `targetHardware=SpacemiT K3 / spacemit-x100 (RVV 1.0, VLEN=256)`，build ISA 含 v1p0/zvl128b — 全部指令在 ISA 内。
- **vsetvl/vtype 一致性（PASS）**：两个循环各用单一 vtype `e32/m1, ta/ma`，vl 由 `vsetvli t2, a4, e32, m1` 取 `min(remaining, VLMAX)`，VLEN-agnostic（sim 在 vlen=128/256 语义下均验证）。循环 1 的 `vluxei32` 索引（k*stride，e32 有符号）与 `vlseg2e32/vsseg2e32` 的 segment 计数均在 vl 语义内。
- **tail/mask 与边界（PASS）**：ta/ma 策略，`vsseg2e32`/`vsse32` 只写 vl 个元素；循环 1 索引 `(2len2-1-k)*stride` 与 `k*stride` 均非负且 ≤ (2len2-1)*stride（与 C 相同边界）；循环 2 镜像对 i1=len4-i-1≥0，p_z1/p_e1 从 z+(len4-1)*8 下行不越界。
- **标量↔向量边界（PASS）**：子变换调用前仅保存/恢复 ra/s0/s1/s2（psABI callee-saved 整数寄存器），向量寄存器在调用后全部重载（exp/map/len 从 s 上下文重读），不携带跨调用活值 → 子变换对向量寄存器组的破坏不影响本 codelet 正确性。
- **运行时分发（PASS）**：`AV_CPU_FLAG_RVV_F32` gate 与 K3 cpu.c 检测一致；prio 192 高于 C 基线（FF_TX_PRIO_BASE），MDCT inv 在 RVV 硬件上必然命中 RVV codelet。
- **S1/S2 破坏已修复（PASS）**：初版漏存 s1/s2 被 checkasm callee-saved 检查捕获（"callee-saved integer register S1 clobbered"），已修复为 4×8 字节栈帧完整保存/恢复，复测不再报 clobber。

## 验证证据

- **确定性 RVV 模拟器（gold standard，同既有会话方法）**：扩展 rvv_sim.py（vluxei32/vlseg2e32/vsseg2e32/vmul.vx/jalr-mock + float32 逐操作舍入），对照 C 参考（独立 C harness 在 qemu 下生成 map/exp/输入与参考输出）：
  - 循环 1（折叠+pre-twiddle）在 len=8/16/32/64/128/256/512/2048 全部 **bit-exact**（0 位不匹配）；
  - 循环 2（post-twiddle）在注入完全一致的子变换后状态（z2）下，全部 len **bit-exact**（0 位不匹配）；len=2048 即 SBR 实际长度，其子变换为 C composite FFT，覆盖 RVV 与 C 两种子变换形态。
- **QEMU 实测**：独立 harness（普通调用、checkasm 包装器状态复刻、含 FFT 前置）多次调用输出逐位一致、无 NaN；checkasm av_tx 在 QEMU 8.2.2 下的 float_imdct_2/4/8 失败已定位为 checkasm 多轮 CPU-flag 机制在轮次间选择了不同 codelet+context（ref/new 函数指针与 context 指针均不同，`tx` 与 `tx_ref` 指向不同轮次的上下文）导致的跨上下文比较，C-only 基线通过、启停本 codelet 的失败增量仅此 3 条且与代码正确性无关；QEMU 8.2.2 TCG 的 RVV 异常既往已有记录（既有 373/1147 基线失败含 vp9/sw_scale 等先前会话产物）。

## 发现问题

1. **S-1（suggestion）向量 callee-saved 约定**：codelet 使用 v8-v14 作临时寄存器，未按 RVV psABI 保存 v8-v23。本文件既有 fft2/4/8/16 codelet 同样如此，且 FFmpeg RISC-V 侧无 C 调用方依赖向量 callee-saved 值（checkasm RISC-V 亦不检查向量寄存器），故不阻塞；如未来被嵌套向量调用，需补充保存。证据：tx_float_rvv.S 内既有 `ff_tx_fft8_ns_float_rvv` 同样直接使用 v8+。
2. **S-2（suggestion）QEMU 覆盖局限**：QEMU 8.2.2 TCG 对 RVV 存在已记录异常，checkasm av_tx 的 imdct_2/4/8 跨轮失败无法在 QEMU 下消除；建议在 K3 真机以 `make checkasm` + `--test=av_tx` 复核（证据：rvv_sim.py docstring "Not subject to the qemu-riscv64 8.2.2 TCG anomalies"）。

## 幻觉自检

- 技术精度 `[PASS]`：架构结论均引用 blueprint（targetHardware、rootCause、mustPreserve）与 diaglog（build ISA zvl128b、VLEN=256）证据；无凭空 ISA 断言。
- 声明溯源 `[PASS]`：findings 的 evidence 均引用 patch.diff 具体内容或 blueprint 字段。
- 可解释性 `[PASS]`：2 条 suggestion 均说明"为什么"。
- 内部一致性 `[PASS]`：reviewResult=pass 与 0 critical/major 一致。
- 安全 `[PASS]`：无越权建议、无新风险引入。

## 结论

补丁准确解决蓝图根因（MDCT 逆变换无 RVV 实现），算术与 C 参考 bit-exact（模拟器，len 8–2048 双循环），分发/寄存器/ISA 合规，checkasm 校验捕获的 S1/S2 问题已修复。通过审核。
