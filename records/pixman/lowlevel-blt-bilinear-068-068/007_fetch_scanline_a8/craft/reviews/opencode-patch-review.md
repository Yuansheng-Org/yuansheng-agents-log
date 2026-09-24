# AI 补丁审核报告

- 审核对象：PatchCandidate `pc-bp-pixman-lowlevel-blt-bilinear-068-068-007`
- 审核人：yuansheng-craft-reviewer（独立只读审核）
- 审核时间：2026-09-08

## 审核范围

本审核只基于落盘事实来源，不依赖实现会话记忆：

- RootCauseBlueprint：`.yuansheng/trace/pixman/lowlevel-blt-bilinear-068-068/007_fetch_scanline_a8/blueprint_pixman_lowlevel-blt-bilinear-068-068_007.json`
- 同目录 diaglog：`diaglog_pixman_lowlevel-blt-bilinear-068-068_007.md`
- `craft/patch-plan.json`、`craft/patch.diff`、`craft/patch-candidate.json`

变更文件（4 个，全部为项目代码，无 `.yuansheng/` 协议产物）：

1. `pixman/pixman-rvv.c` — 新增 `_pixman_rvv_fetch_scanline_a8` RVV 向量化取数内核（+45 行）
2. `pixman/pixman-access.c` — `setup_accessors` 中为 `PIXMAN_a8` 在 RVV 可用时覆盖 `fetch_scanline_32`
3. `pixman/pixman-riscv.c` — 新增全局标志 `_pixman_have_rvv` 并在 RVV 实现创建时置 TRUE
4. `pixman/pixman-private.h` — 声明 `_pixman_have_rvv` 与 `_pixman_rvv_fetch_scanline_a8`

## RISC-V 架构审核

机器判定 `arch-scan` 结果为 `archSpecific: true`（命中 `rvv-intrinsic` 与 `riscv-macro` 规则），本审核执行 RISC-V 架构专项审核。

1. **指令/特性是否在目标 ISA 内**：通过。内核使用 `vsetvli`、`vle8.v`、`vzext.vf4`、`vsll.vi`、`vse32.v`，均为 RVV 1.0 基础整数向量指令。目标硬件 SpacemiT X100（blueprint `source.targetHardware`）为 RVV 1.0 / VLEN=256；该文件在 `meson.build` 中以 `rvv_flags = ['-march=rv64gcv1p0']` 编译（含 `v`），指令在目标 ISA 内。已用 `riscv64-linux-gnu-gcc -march=rv64gcv1p0` 独立编译并通过。
2. **vsetvl/vtype 与数据流一致**：通过。`__riscv_vsetvl_e8m1` 设定 SEW=8/LMUL=1；`__riscv_vzext_vf4_u32m4` 将 lane 零扩展为 SEW=32/LMUL=4，后续 `vsll`/`vse32` 均为 e32m4，SEW/LMUL 全程一致，无状态错乱（反汇编确认 vzext/vsll/vse32 前编译器正确插入 `vsetvli zero,a5,e32,m4`）。
3. **tail/mask policy 与寄存器组**：通过。使用默认 tail-agnostic/mask-agnostic；循环按 `vl = vsetvl_e8m1(n)` 逐段处理，`n=0` 时不进入，最后一整段即 tail，无掩码尾、无寄存器组重叠。
4. **标量↔向量边界与运行时分发**：通过。标量 `fetch_scanline_a8` 保留为回退；仅当 `_pixman_have_rvv`（在 `_pixman_riscv_get_implementations` 的 hwprobe/getauxval 检测通过后置 TRUE）时才覆盖函数指针，非 RVV 目标走原标量路径，运行时 ISA 分发正确。
5. **未混入 x86/ARM 指令**：通过。diff 中无任何 x86/ARM 指令或 intrinsic。

## 审核结果

- **根因解决**：blueprint 根因为「A8→ARGB32 取数循环标量执行、未向量化」。补丁将该逐像素 `lbu→slliw 0x18→sw` 标量链改写为 `vle8.v→vzext.vf4→vsll.vi 24→vse32.v` 批量向量化，直接命中 blueprint `diagnosis.recommendedFirstAction` 所述 RVV 改写形态。blueprint 同时声称「build 不含 v」的 L0 配置缺口（`rootCauseType: config_mismatch`），但该结论在 blueprint `currentGaps` 中被标注为「构建文件路径未提供」的 baseline_gap（Tag_RISCV_arch 由 bench ELF 推断，非 libpixman）；实际 `meson.build` 已有 `-march=rv64gcv1p0` 且 combine 层 `rvv_*` 热点已运行，说明配置本身正确，实质根因为 L1 缺向量化内核，故补丁只做向量化是正确且最小。
- **约束保持**：通过。输出布局保持 `a<<24`（alpha 位于 24–31、低 24 位为 0，由 vzext.vf4 零扩展 + vsll.vi 24 保证）；fetch slot 分派语义保留（运行时覆盖 + 标量回退）；stride（`image->bits + y*rowstride`）与 tail 完整性（vsetvl 逐段）保持；little-endian 用原生 vle8/vse32；fetch 写入独立 `buffer`，无 in-place/overlap 风险。
- **最小性**：通过。变更聚焦、无无关重构，除被改动行邻接的一处行尾空白清理外无格式噪音。
- **产物一致性**：通过。`patch-candidate.json` 的 `gitDiff` 与 `patch.diff` 一致，`changedFiles` 与 diff 文件头一致。
- **安全**：通过。无硬编码密钥、无危险命令/路径、无越权建议。

## 发现问题

无 critical / major / minor / suggestion 级别问题。

## 幻觉自检

- **技术精度** `[PASS]`：RVV 指令归属与 ISA 判定均以 blueprint `targetHardware` + `meson.build` 的 `rvv_flags` + 独立交叉编译为证据。
- **声明溯源** `[PASS]`：无 critical/major finding，故无需要锚点的声明；本报告所有证据均引用 diff 原文或 blueprint 字段。
- **可解释性** `[PASS]`：无 critical/major finding 需解释。
- **内部一致性** `[PASS]`：`reviewResult: pass` 与空 findings 一致（pass 无需 critical/major）。
- **安全** `[PASS]`：未引入新风险。

## 结论

补丁准确、最小且行为保持地解决了 `fetch_scanline_a8` 未向量化的根因，RISC-V 架构代码位于目标 ISA 内并已验证可编译。**审核通过（pass）**。
