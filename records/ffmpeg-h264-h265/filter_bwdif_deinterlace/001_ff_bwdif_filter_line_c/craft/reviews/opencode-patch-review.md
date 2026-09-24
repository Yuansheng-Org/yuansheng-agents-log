# AI 补丁审核报告：bwdif filter_line + filter_edge RVV

- **reviewId**: rv-ffmpeg-filter-bwdif-001-006
- **patchCandidateId**: pc-bp-ffmpeg-filter-bwdif-deinterlace-001
- **reviewer**: opencode-ai-reviewer
- **reviewResult**: **pass**
- **reviewedAt**: 2026-09-14

## 审核范围

补丁为 `filter_bwdif_deinterlace` testcase 族（queue 001/006）添加 RISC-V RVV 实现：

- `libavfilter/riscv/vf_bwdif_rvv.S`（新文件）：`ff_bwdif_filter_line_rvv` + `ff_bwdif_filter_edge_rvv`
- `libavfilter/riscv/vf_bwdif_init.c`（新文件）：`ff_bwdif_init_riscv` 注册 filter_line/filter_edge（bit_depth==8 且 RVV_I32 且 vlen>=128）
- `libavfilter/bwdifdsp.c/.h`、`libavfilter/riscv/Makefile`：ARCH_RISCV 钩子与 RVV 目标注册

## 算法与 C 参考逐项核对（bwdifdsp.c）

| 阶段 | C 参考 | RVV 实现 | 一致 |
|------|--------|----------|------|
| FILTER1 | d=(prev2+next2)>>1; td0=abs(prev2-next2); td1=(abs(prev[mrefs]-c)+abs(prev[prefs]-e))>>1; td2 同理; diff=max3(td0>>1,td1,td2) | e16 向量：v20=d, v18=td0, v19/v21=td1/td2, v22=diff | ✓ |
| nz | !diff -> dst=d | vmsne.vx v25 (pre-SPAT) | ✓ |
| SPAT_CHECK | b,f,dc,de; max=MAX3(de,dc,min(b,f)); min=MIN3(de,dc,max(b,f)); diff=max3(diff,min,-max) | 完全按序向量化（含 -max 用 vneg） | ✓ |
| FILTER_LINE | HF: ((5570*(p2+n2)-3801*m2+1016*m4)>>2 + 4309*(c+e)-213*cm)>>13; SP: (5077*(c+e)-981*cm)>>13; |c-e|>td0 ? HF : SP | e32 域 vwcvt 扩宽后 vmul/vsub/vadd/vsra，vmerge.vvm 选路（v0 mask=td0<ce） | ✓ |
| FILTER_EDGE | spat ? SPAT_CHECK : (无); interpol=(c+e)>>1 | beqz t3 跳过 SPAT；interpol=(c+e)>>1 | ✓ |
| FILTER2 | interpol>d+diff -> d+diff; <d-diff -> d-diff; clip [0,clip_max] | vsub/vadd 边界 + vmax/vmin | ✓ |
| 输出 | dst = nz ? interpol : d | vxor/vand/vxor 掩码选路 | ✓ |

## RISC-V 架构审核

- `func ... zve32x` + RVV 1.0 指令集；arch-scan 0 findings。
- parity 语义与 C 一致：`prev2 = parity ? prev : cur`、`next2 = parity ? cur : next`（先前版本反了，已修正）。
- HF/SP 选路在 e32 域完成，mask（tdiff0<ce）在 e16 域先生成，随后 vsetvli e32 保持相同 vl（s2），vmerge.vvm 掩码按 lane 对齐。
- vmerge.vvm 约定与已交付 vf_blend hardmix 一致：`vd = mask ? vs1 : vs2`（v0 为掩码寄存器）。
- 栈上参数偏移：prologue 减 128 后按 sp+128+off 读取（mrefs2/prefs3/mrefs3/prefs4/mrefs4/parity/clip_max/spat 全部核对）。
- 存储路径：`vsetvli t1, s2, e16, m2` / `e8, m1` 显式用 s2 保持 VL（不用 zero，避免 VLMAX 截断）。

## 幻觉自检

- **blueprint-anchor PASS**：修改文件与 blueprint rootCause.candidateFiles 一致。
- **root-cause-boundary PASS**：只做性能修复（新增 RVV 内核 + 注册），无无关重构。
- **verification-evidence PASS**：模拟器 30 用例 × 2 函数（parity 0/1 × spat 0/1 全覆盖），filter_line 0/1920、filter_edge 0/1920 逐字节一致。
- **diff-anchor PASS**：review-validate 机器复核 0 findings。

## 审核结果

审核结论：**pass**（补丁实现与 C 参考逐字节一致，模拟器验证 3840/3840）。

## 发现问题

无（review-validate 机器复核 findings=0）。

## 结论

补丁实现与 C 参考逐字节一致（模拟器验证 3840/3840），通过通用质量、RISC-V 架构专项与防幻觉锚点校验。
