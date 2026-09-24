# RISC-V 性能分析报告 — ff_jpeg_fdct_islow_8 (rank 003, mpeg4_encode)

**Functions under analysis: [ff_jpeg_fdct_islow_8]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（003-ff_jpeg_fdct_islow_8-annotate.txt）
- perf stat：已提供（IPC 1.474；branch_miss_rate=4.278%；88fps/300 帧）
- source context：libavcodec/jfdctint_template.c ff_jpeg_fdct_islow（8x8 前向 DCT）；libavcodec/riscv/ 无对应实现
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
L0 gate：模式 A。关键：JPEG FDCT 无 RVV 实现。

## Phase 2 — Scope / 分析边界
函数：ff_jpeg_fdct_islow_8（8x8 前向 DCT，C fallback）。
- 采样点：7dbe48 lh 系数 / 7dbe9c addi 行推进

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证齐全（riscv/ 无 fdct 实现）。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 14.81 : 7dbe9c addi a5,a5,16（行推进）
- 3.70 : 7dbe48 lh a4,14(a5)（系数加载）

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除

**(c) 双 Confidence：** route=High；impact=Medium（FDCT 每宏块执行）

## Phase 4 — Root-cause blueprint
1. **Root cause**：JPEG FDCT（8x8 前向 DCT）为 C 标量实现：libavcodec/riscv/ 无 fdct kernel。
2. **The fix**：与 DCT 族统一规划（SEW=32 蝶形变换向量化）；init 补齐 fdct 槽。Correctness contract：编码输出逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——rank 003
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：SEW=32；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712）

## Phase 5 — Verification forecast
- 应消失：标量蝶形链占比下降；RVV kernel 出现 vle/vmul/vadd/vse
- 验证方法：MPEG-4 编码输出一致；K3 对比时间

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无