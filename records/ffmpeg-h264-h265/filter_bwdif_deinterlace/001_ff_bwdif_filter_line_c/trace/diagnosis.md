# RISC-V 性能分析报告 — ff_bwdif_filter_line_c (rank 001, filter_bwdif_deinterlace)

**Functions under analysis: [ff_bwdif_filter_line_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（001-ff_bwdif_filter_line_c-annotate.txt，~176 samples）
- perf stat：已提供（IPC 2.519；branch_miss_rate=2.318%；L1D miss=0.245%；131fps/240 帧）
- source context：libavfilter/vf_bwdif.c ff_bwdif_filter_line_c（bwdif 反交错主滤波行）；**libavfilter/riscv/ 无 bwdif 实现**
- hardware：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0（v）+ zve64d + zba/zbb；K3/X100，out-of-order |
| Build ISA | 含 v1p0+zba 但按 zvl128b1p0（128-bit）构建 |
| Vector flavor | annotate 内 zero v*（全标量 lbu/add/addw——C 滤波） |
| VLEN | 硬件 256 bits；build 假定 128 bits |
| Bound type | IPC 2.519；compute-leaning |
| 关键 | libavfilter/riscv/ 无 bwdif 实现 |

L0 baseline gate：模式 A。关键：bwdif 主滤波行无 RVV 实现。

## Phase 2 — Scope / 分析边界
函数：ff_bwdif_filter_line_c（bwdif 时空插值滤波主循环：上下行加权 + 运动检测）。
- hot interval：3dc9b8–3dc9cc（地址计算 + lbu 加载 + 滤波累加）
- 采样：3dc9cc lbu 6.82%、3dc9c4 add 3.41%、3dc9b8 add 2.27%

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证：①bwdif filter_line DSP slot；②C fallback 被采用（全标量）；③RVV bwdif 缺失（libavfilter/riscv/ 无 bwdif）；④FFmpeg policy。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 6.82 : 3dc9cc lbu t4,0(t1)（像素加载）
- 3.41 : 3dc9c4 add a4,a7,s8（地址计算）

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除（热点非已存在 .S）

**(c) 双 Confidence：** route=High；impact=Medium（bwdif 是通用反交错高频滤镜）

## Phase 4 — Root-cause blueprint
1. **Root cause**：bwdif 反交错主滤波行（filter_line）的 RISC-V 向量实现不存在：libavfilter/riscv/ 无 bwdif 实现，C 标量时空插值链全量执行。
2. **The fix**：与 bwdif 滤波族统一规划（SEW=8 像素加载 + 时空加权插值 + 运动检测向量化）；init 补齐。Correctness contract：像素逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——176 samples（rank 001，全用例最高）
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：逐行像素；SEW=8；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712）

## Phase 5 — Verification forecast
- 应消失：lbu 加载链占比下降；RVV kernel 出现 vle8/vadd/vmacc/vse8
- 验证方法：bwdif 反交错输出一致；K3 对比时间

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无