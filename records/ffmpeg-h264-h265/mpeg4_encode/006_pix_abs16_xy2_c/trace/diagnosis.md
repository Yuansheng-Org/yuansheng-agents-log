# RISC-V 性能分析报告 — pix_abs16_xy2_c (rank 006, mpeg4_encode)

**Functions under analysis: [pix_abs16_xy2_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（006-pix_abs16_xy2_c-annotate.txt）
- perf stat：已提供（IPC 1.474；branch_miss_rate=4.278%；88fps/300 帧）
- source context：libavcodec/mpegvideo_enc.c pix_abs16_xy2（像素绝对差 xy2 插值）；libavcodec/riscv/ mpegvideoencdsp 仅注册 pix_abs16（x2/xy2 变体无 RVV？）——需确认
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
L0 gate：模式 A。关键：pix_abs16_xy2 为 C fallback（采样全标量）。

## Phase 2 — Scope / 分析边界
函数：pix_abs16_xy2_c（像素绝对差 xy2 水平+垂直插值，C fallback）。
- 采样点：80ad1e addw 插值累加 / 80ad70 sraiw

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证齐全（mpegvideoencdsp 仅 pix_abs16 x2 有 RVV？xy2 无；C fallback 被采用）。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 16.67 : 80ad1e addw t0,a6,t0（插值累加）
- 5.56 : 80ad70 sraiw s0,s6,0x1f

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除

**(c) 双 Confidence：** route=High；impact=Medium（运动估计 SAD 每搜索点执行）

## Phase 4 — Root-cause blueprint
1. **Root cause**：pix_abs16_xy2（水平+垂直插值 SAD）为 C 标量实现（riscv 仅覆盖部分 SAD 变体）。
2. **The fix**：与 SAD 族统一规划（插值 + vwsubu 绝对差向量化）；init 补齐 xy2 槽。Correctness contract：编码输出逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——rank 006
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：SEW=8；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712）

## Phase 5 — Verification forecast
- 应消失：标量插值链占比下降；RVV kernel 出现 vle8/vwsubu/vse
- 验证方法：MPEG-4 编码输出一致；K3 对比时间

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无