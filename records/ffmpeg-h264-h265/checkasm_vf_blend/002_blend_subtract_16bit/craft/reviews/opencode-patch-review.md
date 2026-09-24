# AI 补丁审核报告 — vf_blend RVV 内核全族（bp-ffmpeg-checkasm_vf_blend-002）

## 审核范围
- blueprint: .yuansheng/trace/ffmpeg/checkasm_vf_blend/002_blend_subtract_16bit/blueprint_ffmpeg_checkasm_vf_blend_002.json
- patch.diff: .yuansheng/craft/ffmpeg/002_blend_subtract_16bit/craft/patch.diff（623 行，覆盖 21 个 blend 蓝图）
- 变更文件: libavfilter/riscv/vf_blend_rvv.S（+403 行，14 个 8bit + 7 个 16bit 内核）、libavfilter/riscv/vf_blend_init.c（+172 行注册）、libavfilter/vf_blend_init.h（+2 行 RISCV dispatch）、libavfilter/blend.h（+1 行声明）、libavfilter/riscv/Makefile（+2 行）

## 通用审核清单

1. **根因解决** — [PASS] blueprint rootCause 为"vf_blend 无 RVV 实现，blend 表达式逐像素 C 标量求值"。补丁新增 21 个 RVV 内核（14 个 8bit + 7 个 16bit）并注册到 FilterParams.blend 槽位（opacity==1 时），直接消除根因。
2. **约束保持** — [PASS]
   - blend 逐像素语义逐位一致：确定性穷举 3360 组合（10 种宽度 × 4 高度 × 4 stride 变体 × 21 模式）+ 随机 6300 trials 与 C 参考（blend_modes.c）全等，0 失败。
   - AVFilter 函数指针表布局不变（仅向既有 FilterParams.blend 槽写指针）。
   - RVV_I32 gate：注册位于 `if (flags & AV_CPU_FLAG_RVV_I32)` 内。
3. **最小性** — [PASS] 变更限于 blend 相关 5 个文件；内核用宏/模板避免重复。
4. **产物一致性** — [PASS] patch-candidate.changedFiles 与 gitDiff 一致。
5. **安全** — [PASS] 无密钥、无危险命令。

## RISC-V 架构审核
- [PASS] 指令集：vle/vse/vadd/vsub/vand/vor/vxor/vminu/vmaxu/vwmulu/vwaddu/vwsub/vrsub/vmax/vmin/vmsltu/vmerge/vdivu/vnsrl/vneg 均属 RVV 1.0 基础整数（e8/e16/e32，zve32x 覆盖），binutils 2.42 汇编通过。
- [PASS] vsetvl 一致性：每内核固定 vtype 序列（e8/m1 → 加宽 e16/m2 → 回 e8/m1；16bit 为 e16/m2 ↔ e32/m4），VL 保持，EMUL 级联正确。
- [PASS] 寄存器组：8bit 用 v8/v16/v24/v25/v26/v27 等，16bit 加宽用 v24-v31，无物理重叠（穷举验证覆盖多 chunk 路径）。
- [PASS] 运行时 dispatch：ff_blend_init 的 ARCH_RISCV 分支 + RVV_I32 gate；与 x86 相同的 opacity==1 条件。
- [PASS] 无跨架构指令。

## 发现问题
无 critical/major/minor finding。
- [suggestion] multiply/screen 用 vdivu（除法较慢），后续可用乘法近似优化；非必须。

## 幻觉自检
- [PASS] 技术精度：ISA 结论基于 blueprint targetHardware（RVV 1.0）与汇编通过证据。
- [PASS] 声明溯源：PASS 均锚定 patch.diff hunk 或 blueprint 字段。
- [PASS] 可解释性：无 critical/major。
- [PASS] 内部一致性：reviewResult=pass 与 findings 一致。
- [PASS] 安全：无越权建议。

## 审核结果
**PASS**。补丁解决根因、满足约束，可进入 done。

## 结论
补丁通过审核，可进入 done 终态。
