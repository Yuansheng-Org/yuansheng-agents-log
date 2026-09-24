# riscv-performance-optimization 分析报告 — avg_h264_chroma_mc4_16_c (rank 003, checkasm_h264chroma)

**Functions under analysis: [avg_h264_chroma_mc4_16_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（003-avg_h264_chroma_mc4_16_c-annotate.txt，1 sample，cpu-clock，percent: local period，覆盖完整函数）
- perf stat（bound/context）：已提供（IPC 2.046；speedup_total=0.96×）
- workload/binary/source context：已提供（FFmpeg master @1d7b14f；libavcodec/h264chroma_template.c H264_CHROMA_MC 宏生成，_16 后缀=16-bit 高 bit depth 变体；h264_chroma_init_riscv.c 仅 bit_depth==8 注册 RVV（h264_mc_chroma.S 用 vle8/vse8）；16-bit 无 RVV 覆盖 → C fallback 被 checkasm 采用；对比 aarch64 NEON、x86 有 mmxext/ssse3（含 10-bit））
- readelf -A（build ISA）：已提供（metadata binaries：含 v1p0+zba+zvl128b1p0）
- hardware ISA：已提供（rv64imafdcvh_...，SpacemiT X100，含 v/zve16x）
- vlenb：已提供（32 bytes = VLEN 256 bits）
- 采样元数据：已提供（cpu-clock；percent: local period；单函数；同窗口）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0（v）+ zve64d + zba/zbb；K3/X100，out-of-order |
| Build ISA | 含 v1p0+zba 但按 zvl128b1p0（128-bit）构建 |
| Vector flavor | annotate 内 zero v*（全标量 lhu/mulw/sraiw/sh——C fallback） |
| VLEN | 硬件 256 bits；build 假定 128 bits |
| Bound type | IPC 2.046；非 memory/branch-bound；16-bit chroma MC C fallback |
| Sampling semantics | event=cpu-clock；percent=local period；函数级贡献未知 |
| Sampling IP precision | baseline_gap: sampling IP precision → interval-level 归因 |

L0 baseline gate：hardware 有 v，build 有 v（无 mismatch）；无 th.v*；入口模式 A。Bound-type gate：非 memory-bound。关键：16-bit chroma MC 无 RVV 实现（init 仅 bit_depth==8 gate）。

## Phase 2 — Scope / 分析边界
函数清单与承诺一致：avg_h264_chroma_mc4_16_c（H.264 16-bit chroma MC avg 变体，C fallback）。
- hot interval：b2cc2–b2de0（主循环：4 像素行 × 4-tap 加权，lhu/mulw/addiw/sraiw/sh 标量）+ dx/dy 分支
- 1 sample：b2e26: sh t3,0(a0)（avg 写回）100%
- DSP slot 证据：H264ChromaContext.avg_h264_chroma_pixels_tab（h264chroma.h）；RVV init 仅 8-bit；16-bit 走 C template；aarch64 NEON/x86（含 10-bit mmxext）有架构实现
- annotate 覆盖完整

## Phase 3 — Pattern scan / 模式扫描：avg_h264_chroma_mc4_16_c
**Class selection trace：**
| Class | include/exclude | 触发观察 |
|---|---|---|
| rows-asm.md | include | policy-backed missing .S 候选：DSP slot + 16-bit fallback + RVV 16-bit 缺失 + FFmpeg policy（四证） |
| rows-operator-rvv.md | include（扫描后评估） | elementwise 加权语义作为 supporting |
| rows-string-memory.md | exclude | 非 string/memory 语义 |
| rows-vectorized-tuning.md | exclude | C fallback 非已向量化代码 |
| rows-codegen.md | exclude | 无 codegen 形态主导 |
| rows-offload.md | exclude | 无矩阵引擎 |
| rows-crypto.md | exclude | 非密码学 |
| rows-runtime-os.md | exclude | 用户态 |

Classes scanned: rows-asm.md, rows-operator-rvv.md

### Local performance pattern scan: avg_h264_chroma_mc4_16_c

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，弱命中） | 四证：① avg_h264_chroma_pixels_tab 为 SIMD/DSP dispatch slot；② 16-bit C fallback 被 checkasm 采用（1 sample 落在该实现）；③ RVV 无 16-bit chroma MC 实现（init bit_depth==8 gate）；④ FFmpeg SIMD/DSP policy 适用。准入 gate：K3 支持 v/zve16x 成立 | High | Low | patterns/policy_backed_missing_riscv_assembly_kernel.md |

**(a) 逐字 evidence 引用（primary）：**
- 100.00 : b2e26: sh t3,0(a0)（16-bit avg 写回——C fallback 标量链）
- 所属 interval：b2cc2–b2de0（主循环 4-tap 加权）。全标量

**(b) 互斥邻居排除：**
- kernel_selection_and_runtime_specialization：排除——不存在已适配 16-bit RVV kernel 未采用的问题；init 无 16-bit 赋值
- riscv-assembly-kernel-performance-optimization（现存 .S 内部瓶颈）：排除——当前热点不是已存在的 8-bit .S
- no-vectorization：排除——missing-.S 四证路径优先
- rvv_contiguous_elementwise_arithmetic_kernels（supporting）：排除 primary 归属——16-bit chroma MC 是 DSP slot 合同

**(c) 双 Confidence 推导式：**
- route: 四证 ①②④ 直接成立，③ 源码确认 → High
- impact: 1 sample 过少、16-bit H.264（High 10）场景相对小众；缺 Sampling IP precision、缺 workload 级贡献 → Low

## Phase 4 — Root-cause blueprint / 根因蓝图：avg_h264_chroma_mc4_16_c

1. **Root cause**：16-bit（High bit depth）H.264 chroma MC 的 RISC-V 向量实现不存在：h264_chroma_init_riscv.c 仅在 bit_depth==8 时注册 RVV（h264_mc_chroma.S 全为 8-bit），16-bit 变体长期执行 C template 标量 4-tap 加权链。对比 aarch64 NEON、x86 mmxext/ssse3（含 10-bit）均有实现。依据 patterns/policy_backed_missing_riscv_assembly_kernel.md §Why：函数合同要求的 target implementation 不存在。注意：16-bit 场景仅 H.264 High 10+ profile 触发，样本仅 1 个——需评估优先级。
2. **The fix / 修复方式**（§The fix + Shape-first）：
   - 实现 16-bit chroma MC RVV kernel：h264_mc_chroma.S 增加 16-bit 变体（vle16/vse16 + e16 加权），init 增加 bit_depth>8 分支注册
   - Shape-first：mc4 W=4 像素/行（e16，VLEN=256 下 m1 覆盖 16 元素=4 像素×4 行）；mc8 W=8；mc2 W=2；4-tap 加权（vwmulu/vwmaccu e16→e32 + vnclipu.wi 窄化）
   - Correctness contract：16-bit 像素输出逐位一致（4-tap 加权与 avg 舍入语义）；_16 模板语义不变
   - 限制/风险：High 10 profile 使用率较低，收益需按真实 workload 评估；需 zve16x gate
   - 预期 Profile signals：lhu/mulw/sraiw/sh 消失，出现 vle16/vwmulu/vnclipu/vse16
3. **Baseline facts 回填**：hardware ISA=v/zve64d+zve16x；build ISA=含 v1p0 但 zvl128b1p0；VLEN=256；bound=compute-leaning（IPC 2.046）
4. **收益上界**：局部样本份额——本函数 1 sample（rank 003）；16-bit 场景小众；缺 workload 级贡献 → 不称 Amdahl 上界
5. **三维路由判定**：current source=C compiler-generated；implementation existence=16-bit 缺失（8-bit 已存在证明框架支持）；function-level policy=FFmpeg SIMD/DSP policy 适用
6. **Implementation-shape proof**：mc4/16-bit：W=4 固定 lane 沿行轴向量化，SEW=16；VLEN=256 下 m1 覆盖 16 元素；tail 无（4 整除）；GPR ≤6；gate zve16x
7. **Related PRs**：Related PRs：1 条 URL（FFmpeg PR 23712 synth_filter_float shape-first 实例）

## Phase 5 — Verification forecast / 验证预测：avg_h264_chroma_mc4_16_c
- 应消失/缩小：b2e26: sh 等标量链（lhu/mulw/sraiw/sh）local sample share 应下降；16-bit RVV kernel 出现 vle16/vwmulu/vnclipu/vse16
- 验证方法：checkasm h264chroma（bit_depth 16 用例）bit-exact；K3 上对比 16-bit C vs RVV cycles

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组（载荷：1/1 组；[avg_h264_chroma_mc4_16_c]） | ✅ |
| 2 | Phase 1 输出要求：7 行 + L0 gate + 准入 gate + bound gate + Sampling IP precision（载荷：7 项；baseline_gap: sampling IP precision） | ✅ |
| 3 | Phase 3 输出要求：8 项 trace；Classes scanned: rows-asm.md, rows-operator-rvv.md；顶层 finding 1（primary missing-.S 弱命中）；evidence 锚点 b2e26；排除 4 条；推导式 1 条 | ✅ |
| 4 | Phase 4 输出要求：pattern policy_backed_missing_riscv_assembly_kernel.md 已读；命中 row Policy-Backed Missing RISC-V Assembly Kernel；四证逐项；Implementation-shape proof；The fix 完整；Related PRs：1 条 URL | ✅ |
| 5 | 路径合规：模式 A；8 项 trace 可解释；单命中；L0/L1 归属 | ✅ |
| 6 | Phase 5 两侧锚定：消失侧 b2e26；出现侧 policy_backed_missing_riscv_assembly_kernel.md §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁 | ✅ |

修正记录：无