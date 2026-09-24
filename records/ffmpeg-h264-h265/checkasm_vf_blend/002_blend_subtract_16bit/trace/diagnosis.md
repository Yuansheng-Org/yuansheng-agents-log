# RISC-V 性能分析报告 — blend_subtract_16bit (rank 002, checkasm_vf_blend)

**Functions under analysis: [blend_subtract_16bit]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（002-blend_subtract_16bit-annotate.txt）
- perf stat：已提供（IPC 1.908；branch_miss_rate=1.309%；L1D miss=0.973%；**compared_functions=0——vf_blend 无任何 RVV 对照**）
- source context：libavfilter/vf_blend.c blend_expr_subtract 逐像素表达式宏；libavfilter/riscv/ 无 vf_blend 实现（仅 af_afir/vf_blackdetect）
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0（v）+ zve64d + zba/zbb；K3/X100，out-of-order |
| Build ISA | 含 v1p0+zba 但按 zvl128b1p0（128-bit）构建 |
| Vector flavor | annotate 内 zero v*（全标量——C fallback） |
| VLEN | 硬件 256 bits；build 假定 128 bits |
| Bound type | IPC 1.908；compute-leaning |
| 关键 | compared_functions=0 → vf_blend 全 C |

L0 baseline gate：模式 A。关键：blend_subtract（16-bit）无 RVV 实现。

## Phase 2 — Scope / 分析边界
函数：blend_subtract_16bit（vf_blend subtract 混合算子，C fallback）。
- 采样点：85f56 fcvt.wu.s a6,fa5,rtz（float 转 uint 后减法）
- DSP slot：AVFilterContext blend 表达式求值路径；RVV 无覆盖

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证：①blend 算子 DSP slot；②C fallback 被 checkasm 采用（compared=0，全标量）；③RVV blend 缺失（libavfilter/riscv/ 无 vf_blend）；④FFmpeg policy。gate：RVV_I32 | High | Low |

**(a) 逐字 evidence：**
- 100.00 : 85f56 fcvt.wu.s a6,fa5,rtz（float 转 uint 后减法）

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除（热点非已存在 .S）

**(c) 双 Confidence：** route=High（四证齐全）；impact=Low（16-bit 场景小众）

## Phase 4 — Root-cause blueprint
1. **Root cause**：vf_blend subtract 混合算子的 RISC-V 向量实现不存在：libavfilter/riscv/ 无 vf_blend 实现，blend 表达式逐像素 C 标量求值全量执行。
2. **The fix**：与 blend 算子族统一规划（SEW=16 逐像素表达式向量化 + clip）；init 补齐。Correctness contract：像素逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——本函数 rank 002
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：逐像素独立；SEW=16；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712 shape-first 实例）

## Phase 5 — Verification forecast
- 应消失：标量求值链占比下降；RVV kernel 出现 vle/vadd/vsub/vse
- 验证方法：checkasm vf_blend（subtract）bit-exact；K3 对比 cycles

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无