# RISC-V 性能分析报告 — put_hevc_qpel_bi_w_hv_8 (rank 001, checkasm_hevc_pel)

**Functions under analysis: [put_hevc_qpel_bi_w_hv_8]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（001-put_hevc_qpel_bi_w_hv_8-annotate.txt，~138 samples，覆盖完整函数）
- perf stat：已提供（IPC 2.597；speedup_total=10.92、geomean=6.107——但仅 5 个 pel_pixels 有 RVV）
- checkasm_bench.csv：已提供（1576 行；rvv 仅 5 行 put_hevc_pel_pixels{4,8,16,32,64}_8）
- source context：libavcodec/hevc/hevcpred_template.c HEVC_PEL 宏；hevcdsp_init.c 仅注册 pel_pixels（qpel/epel [0][0] 槽）→ 本函数（8-bit 双方向加权插值 bi_w_hv）无 RVV 覆盖
- hardware ISA：SpacemiT X100，含 v；vlenb=32（VLEN 256）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0（v）+ zve64d + zba/zbb；K3/X100，out-of-order |
| Build ISA | 含 v1p0+zba 但按 zvl128b1p0（128-bit）构建 |
| Vector flavor | annotate 内 zero v*（全标量 mulw/lbu/lh/sh/bltu——C fallback） |
| VLEN | 硬件 256 bits；build 假定 128 bits |
| Bound type | IPC 2.597；compute-leaning |
| Sampling semantics | event=cpu-clock；percent=local period |

L0 baseline gate：模式 A。关键：8-bit qpel 加权插值无 RVV 实现（仅 pel_pixels 拷贝有 RVV）。

## Phase 2 — Scope / 分析边界
函数：put_hevc_qpel_bi_w_hv_8（H.265/HEVC 8-bit qpel 双方向加权插值 bi_w_hv，C fallback）。
- hot interval：17107a–1711fa（8-tap 双 pass：mulw 系数乘加 + sraw 移位 + bltu clip + sh/sb 输出）
- 样本分散：1710c8 sh 12.32%、1710ac/1710c4 addw 各 3.6%、1711e2 bltu 9.42%、171080 lbu 2.9%
- DSP slot：HEVCDSPContext put_hevc_qpel 成员；RVV 仅注册 [0][0]（pel_pixels）

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证：①put_hevc_qpel DSP slot；②8-bit C fallback 被 checkasm 采用（全标量采样）；③RVV 加权插值缺失（hevcdsp_init.c 仅 [0][0] pel_pixels，qpel/epel h/v/hv 全缺）；④FFmpeg policy。准入 gate：RVV_I32/zve64d 成立 | High | Medium |

**(a) 逐字 evidence：**
- 12.32 : 1710c8: sh a5,-2(a3)（输出写回）
- 9.42 : 1711e2: bltu s1,a5,171138（clip 分支）
- 171080 lbu t1,0(a6) 2.9%（源加载）
- 全函数 zero v* —— 纯 C 标量 8-tap 加权插值

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除（热点非已存在 .S）；vectorized-tuning：排除（C fallback）

**(c) 双 Confidence：** route=High（四证齐全）；impact=Medium（8-bit qpel bi_w_hv 是 HEVC 解码高频路径，样本 138 个为全用例之冠）

## Phase 4 — Root-cause blueprint
1. **Root cause**：HEVC 8-bit qpel 双方向加权插值（bi_w_hv）的 RISC-V 向量实现不存在：hevcdsp_init_riscv.c 仅对 qpel/epel 注册 [0][0]=pel_pixels（纯拷贝），所有 8-tap 加权插值变体（h/v/hv、uni/bi、8/9/10/12-bit）执行 C template 标量乘加链。本函数是全用例采样量最高的 C fallback。
2. **The fix**：在 hevcdsp 宏族增加 8-bit qpel 加权插值 RVV 变体（SEW=8 加载 + e16 8-tap 乘加 + 加权/移位 + clip 输出）；init 补齐 put_hevc_qpel 各槽（bi_w_hv 优先，采样最高）。Correctness contract：8-bit 加权插值像素输出逐位一致。
3. **Baseline facts 回填**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256；bound=compute-leaning
4. **收益上界**：局部样本份额——~138 samples（rank 001，全用例最高）；缺 workload 级贡献 → 不称 Amdahl 上界
5. **三维路由判定**：current source=C compiler-generated；implementation existence=加权插值缺失；policy=FFmpeg SIMD/DSP
6. **Implementation-shape proof**：8-tap 固定；SEW=8→e16 乘加；tail 无；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712 shape-first 实例）

## Phase 5 — Verification forecast
- 应消失：1710c8 sh 等标量链占比下降；8-bit qpel bi_w_hv RVV kernel 出现 vle8/vwmaccu/vnclip/vse8
- 验证方法：checkasm hevc_pel（qpel bi_w_hv 8-bit）bit-exact；K3 对比 C vs RVV cycles

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无