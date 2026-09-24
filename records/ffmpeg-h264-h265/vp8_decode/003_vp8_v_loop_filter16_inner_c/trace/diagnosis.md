# RISC-V 性能分析报告 — vp8_v_loop_filter16_inner_c (rank 003, vp8_decode)

**Functions under analysis: [vp8_v_loop_filter16_inner_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（003-vp8_v_loop_filter16_inner_c-annotate.txt）
- perf stat：已提供（IPC 1.803；branch_miss_rate=5.903%；137fps/1500 帧）
- source context：libavcodec/vp8dsp.c vp8_v_loop_filter16_inner（垂直 loop filter）；**libavcodec/riscv/vp8dsp_init.c 无 loop_filter 注册（仅 idct/bilin/epel/pixels 有 RVV）**
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
L0 gate：模式 A。关键：VP8 loop filter 无 RVV 实现（区别于 H.264 loop filter 有 RVV）。

## Phase 2 — Scope / 分析边界
函数：vp8_v_loop_filter16_inner_c（VP8 垂直 loop filter 16 inner，C fallback）。
- 采样点：acda12 lbu 像素 / acda16 lbu

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证齐全（vp8dsp_init 无 loop_filter 注册）。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 3.45 : acda12 lbu t6,0(t1)（像素加载）
- 0.86 : acda0e lbu t4,0(a6)

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除

**(c) 双 Confidence：** route=High；impact=Medium（loop filter 每宏块执行）

## Phase 4 — Root-cause blueprint
1. **Root cause**：VP8 loop filter（v/h 16/8uv inner/non-inner）为 C 标量实现：vp8dsp_init.c 无 loop_filter 注册。
2. **The fix**：与 VP8 loop filter 族统一规划（掩码滤波向量化，H.264 loop filter RVV 可作先例：vmslt/vmerge + vnclip）；init 补齐。Correctness contract：像素逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——rank 003
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：SEW=8；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712）

## Phase 5 — Verification forecast
- 应消失：lbu 像素链占比下降；RVV kernel 出现 vle8/vmslt/vmerge/vse8
- 验证方法：VP8 解码输出一致；K3 对比时间

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无