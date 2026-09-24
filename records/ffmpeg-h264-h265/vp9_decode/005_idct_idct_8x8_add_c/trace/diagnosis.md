# RISC-V 性能分析报告 — idct_idct_8x8_add_c (rank 005, vp9_decode)

**Functions under analysis: [idct_idct_8x8_add_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（005-idct_idct_8x8_add_c-annotate.txt）
- perf stat：已提供（IPC 2.228；branch_miss_rate=4.806%；32fps/1200 帧）
- source context：libavcodec/vp9dsp.c idct_idct_8x8（VP9 idct 8x8）；libavcodec/riscv/vp9dsp_init.c 无 idct 注册（仅 dc 有 RVV）
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
L0 gate：模式 A。关键：VP9 idct 8x8 无 RVV 实现（仅 ff_dc_8x8_rvv 有）。

## Phase 2 — Scope / 分析边界
函数：idct_idct_8x8_add_c（VP9 idct 8x8 反变换+叠加，C fallback）。
- 采样点：b20aea slli / b20b24 sd

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证齐全（vp9dsp_init 仅 dc 有 RVV，idct 8x8 无）。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 0.82 : b20b24 sd s8,224(sp)（栈保存）
- 0.41 : b20aea slli s4,a1,0x1（蝶形准备）

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除

**(c) 双 Confidence：** route=High；impact=Medium（idct 每块执行）

## Phase 4 — Root-cause blueprint
1. **Root cause**：VP9 idct 8x8 反变换为 C 标量蝶形实现：vp9dsp_init.c 仅 ff_dc_8x8_rvv 有 RVV，idct 8x8 无。
2. **The fix**：与 VP9 itxfm 族统一规划（SEW=32 蝶形变换，hevc/h264 idct RVV 可作先例）；init 补齐。Correctness contract：像素逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——rank 005
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：SEW=32；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712）

## Phase 5 — Verification forecast
- 应消失：标量蝶形链占比下降；RVV kernel 出现 vle/vmul/vadd/vse
- 验证方法：VP9 解码输出一致；K3 对比时间

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无