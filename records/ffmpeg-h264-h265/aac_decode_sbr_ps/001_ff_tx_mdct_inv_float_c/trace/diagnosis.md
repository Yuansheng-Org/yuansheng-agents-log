# riscv-performance-optimization 分析报告 — ff_tx_mdct_inv_float_c (rank 001)

**Functions under analysis: [ff_tx_mdct_inv_float_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（001-ff_tx_mdct_inv_float_c-annotate.txt，83 samples，cpu-clock，percent: local period，覆盖两个 hot loop）
- perf stat（bound/context）：已提供（IPC 1.631，branch miss 2.158%，L1D miss 1.193%）
- workload/binary/source context：已提供（FFmpeg master @1d7b14f，aac_decode_sbr_ps；libavutil/tx_template.c:1312，C 实现，_c 后缀=scalar C fallback；ff_tx_codelet_list_float_c[] 注册表，libavutil/riscv/ 无 tx RVV 实现，aarch64/x86 有 NEON/asm 实现）
- readelf -A（build ISA）：已提供（metadata binaries.ffmpeg-elf-A：Tag_RISCV_arch: rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_..._zvl128b1p0_zvl32b1p0_zvl64b1p0）
- hardware ISA：已提供（metadata cpuinfo：rv64imafdcvh_..._zvbb_zvbc_..._zvkb_zvkg_zvkned_zvknha_zvksh...，SpacemiT X100）
- vlenb：已提供（32 bytes = VLEN 256 bits，metadata vector）
- 采样元数据（event / percent type / scope / 窗口）：已提供（cpu-clock；percent: local period；单函数 annotate；同运行窗口）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 支持 RVV 1.0（v）+ zve64d + zvbb/zvbc/zvkb/zvkg/zvkned/zvknha/zvksh 等；K3/X100，out-of-order |
| Build ISA | 含 v1p0（RVV 1.0 已启用），但 zvl128b1p0（按 VLEN=128 构建），未启用 zvk/zvbb 系列 |
| Vector flavor | annotate 内 zero v*/th.v*（全 scalar flw/fmul.s/fsw） |
| VLEN | 硬件 256 bits（vlenb=32）；build 假定 128 bits（zvl128b）→ build 低估 VLEN，存在 2x 向量宽度未利用空间 |
| Bound type | IPC 1.631 接近 compute-bound；L1D miss 仅 1.19%、branch miss 2.16% → 非 memory/branch-bound |
| Sampling semantics | event=cpu-clock；percent=local period；单函数窗口；函数级 workload 贡献未知 → 收益上界仅限局部样本份额 |
| Sampling IP precision | baseline_gap: sampling IP precision（无 precise_ip 信息）→ 单行占比只锚定 loop interval |

L0 baseline gate：hardware 有 v，build 也有 v（无 mismatch，但 VLEN 128→256 未利用）；无 th.v*；annotate 已提供（入口模式 A）。Bound-type gate：非 memory-bound，compute 向量化收益成立。

## Phase 2 — Scope / 分析边界
函数清单与承诺一致：ff_tx_mdct_inv_float_c（AV_TX MDCT 逆变换 C 实现）。
- hot loop 1（CMUL3 pre-twiddle 复乘蝶形）：f5cb10–f5cb64，最高行 f5cb50: fsw fa4,-8(a7) 18.07%、f5cb38: fmul.s fa3,fa5,fa2 12.05%、f5cb2a: mul a4,a4,a3 8.43%
- hot loop 2（CMUL post-twiddle）：f5cb8c–f5cbfc，最高行 f5cbb8: addi a4,a4,8 10.84%、f5cbf8: fsw fa5,4(a5) 8.43%、f5cbcc: fsw fa4,-4(a4) 6.02%
- 两 loop 均无 v*；annotate 覆盖完整。Sampling IP precision 不足 → 只做 interval-level 归因。

## Phase 3 — Pattern scan / 模式扫描：ff_tx_mdct_inv_float_c
**Class selection trace：**
| Class | include/exclude | 触发观察 |
|---|---|---|
| rows-asm.md | exclude | 当前代码来源是 C（_c 后缀 + tx_template.c），非手写 .S |
| rows-operator-rvv.md | include | 标量逐蝶形 + 交织复数 AoS + twiddle + zero v*，符合 FFT/DCT row signal |
| rows-string-memory.md | exclude | 非 copy/fill/sentinel/compare/checksum 语义 |
| rows-vectorized-tuning.md | exclude | zero v*，无已向量化代码可调 |
| rows-codegen.md | exclude | 热点主导是变换语义与访存形态，非 compiler codegen 形态 |
| rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 参与 |
| rows-crypto.md | exclude | 非密码学原语 |
| rows-runtime-os.md | exclude | 用户态 FFmpeg 负载 |

Classes scanned: rows-operator-rvv.md

### Local performance pattern scan: ff_tx_mdct_inv_float_c

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV FFT/DCT/MFCC Transform Kernels（primary） | scalar butterfly 复乘链 flw/fmul.s/fsub.s/fsw，交织复数 AoS 取用，zero vlseg2e32/vsetvli；hot loop 1 f5cb38 fmul.s (12.05%) + f5cb50 fsw (18.07%)，hot loop 2 f5cbb8 addi (10.84%) + f5cbf8 fsw (8.43%)；CPU 有 v、build 有 v（VLEN 128 假定） | High | Medium | patterns/rvv_fft_transform_kernels.md |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero v*，只解释缺少向量执行，不决定贡献载体 | — | — | patterns/no-vectorization.md |

**(a) 逐字 evidence 引用（primary）：**
- 12.05 : f5cb38: fmul.s fa3,fa5,fa2（hot loop 1 = MDCT pre-twiddle CMUL3 复乘）
- 18.07 : f5cb50: fsw fa4,-8(a7)（hot loop 1 写回 z[i] 交织复数）
- 10.84 : f5cbb8: addi a4,a4,8（hot loop 2 = CMUL post-twiddle 指针推进）
- 8.43 : f5cb2a: mul a4,a4,a3（hot loop 1 stride×index 整数乘，交织存储地址计算成本）
- 所属 interval：f5cb10–f5cb64（loop1）、f5cb8c–f5cbfc（loop2）。均无 v*，复数按 [re,im] AoS 交织存取。

**(b) 互斥邻居排除：**
- rvv_complex_arithmetic_kernels（逐元素复乘）：排除——本函数是蝶形网络 + twiddle + MDCT 前后重排（src 反向索引 in2[-k*stride]、s->fn[0] 递归子变换），非 lane-independent 逐元素复乘
- rvv_fixed_size_separable_transform_kernels：排除——MDCT 有 twiddle/stage 网络与子 FFT 组合
- rvv_floating_point_matmul_and_gemv_kernels：排除——无 M/N/K 嵌套结构
- no-vectorization：不构成互斥——FFT row 是其变换专属具体化，降为 supporting
- rvv_strided_layout_transform_kernels：排除——k*stride 是蝶形数据访问伴生索引计算，主机制是复乘与变换结构

**(c) 双 Confidence 推导式：**
- route: 主循环全 scalar + 硬件 v + 蝶形/twiddle 变换结构（provenance 来自 _c 后缀与 tx.c 注册表）+ 互斥排除成立 → High
- impact: 有函数内 sample share 与采样语义，但缺 Sampling IP precision、VLEN 利用率差（build 128 vs hw 256）、workload 贡献未知 → Medium

**多命中仲裁：** primary = rvv_fft_transform_kernels（L1 vectorization/semantic layer）；supporting = no-vectorization。Evidence-mechanism layer：L1。

## Phase 4 — Root-cause blueprint / 根因蓝图：ff_tx_mdct_inv_float_c

1. **Root cause**：MDCT 逆变换在 RVV 1.0 核（VLEN=256）上仍走标量逐蝶形实现：CMUL3/CMUL 复乘每蝶形 4 次标量 fmul.s+fsub.s/fadd.s 串行发射，交织 AoS 复数 [re,im] 用 flw/fsw 逐点取用，twiddle 交织读取。依据 patterns/rvv_fft_transform_kernels.md §Why this is slow：同一级内 N/2 蝶形完全同构无依赖（"one butterfly per scalar op wastes vector width"），沿蝶形轴向量化后一条 vfmul/vfadd 处理 vl 个蝶形；交织复数阻塞向量化，vlseg2e32 一条指令即可完成 re/im 拆包。FFmpeg 侧 libavutil/riscv/ 无 tx RVV 实现（对比 x86 tx_float.asm、aarch64 tx_float_neon.S），ff_tx_codelet_list_float_c[] 中 C 实现被实际采用。

2. **The fix / 修复方式**：
   - before（当前）：标量逐蝶形——每蝶形 4 次 fmul.s + fadd/fsub.s + 交织 fsw，twiddle 逐点读
   - after：沿蝶形轴向量化一整级——vlseg2e32 一次拆 re/im 两个向量寄存器；蝶形 vfadd/vfsub/vfmul（或 vfmacc）并行；twiddle 改 split(SoA) re[]/im[] 连续 vle32（_init 预生成）；vsseg2e32 交织写回；VLEN-agnostic（vsetvli 自适应 tail）
   - 前置：硬件 v/zve32f（K3 满足，VLEN=256 更佳）；复数 AoS 存储保持
   - Correctness contract：对外 API 输入/输出复数布局与缩放约定不变（SBR 上层 ff_aac_sbr_apply 依赖）；ifftFlag 虚部取反
   - 限制/风险：twiddle split 表需在 _init 一次性生成；与 s->sub 递归子变换（ff_tx_fft*_ns_float_c）配合需一致化
   - 预期 Profile signals：hot loop 内 flw/fsw/fmul.s 占比显著下降，出现 vlseg2e32/vsseg2e32/vsetvli/vfmul

3. **Baseline facts 回填**：hardware ISA=v(RVV 1.0)+zvbb/zvk*；build ISA=含 v1p0 但 zvl128b1p0；VLEN=256（build 128，2x 未利用）；bound type=compute-leaning（IPC 1.63）
4. **收益上界**：当前 sampled event 下的局部样本份额——本函数 83 samples 为用例内最大热区（rank 001），修复对象是两 hot loop 主导的复乘链；缺 workload 级贡献 → 不称 Amdahl 上界
5. **三维路由判定**：
   - current source：C compiler-generated（tx_priv.h:30 TX_NAME→_float_c；tx_template.c:1312）
   - implementation existence/reachability：AV_TX 有 codelet 注册机制；riscv 无向量 codelet，C 实现被采用；aarch64/x86 有 NEON/asm 证明框架支持架构专有实现
   - function-level policy：FFmpeg SIMD/DSP 政策默认性能关键路径手写汇编 + runtime dispatch；但 AV_TX C 实现是合法注册实现，无强制 .S 四证 → 按 vectorized C/intrinsic 方向即可
6. Implementation-shape proof：不适用（非 policy-backed missing .S 分支）
7. **Related PRs**：Related PRs：4 条 URL（NMSIS 952264aaa7 / dea09437a6 / 6752d263ba / b14e3dafb0）

## Phase 5 — Verification forecast / 验证预测：ff_tx_mdct_inv_float_c
- 应消失/缩小：hot loop 1 f5cb38: fmul.s fa3,fa5,fa2、f5cb50: fsw fa4,-8(a7)、f5cb2a: mul a4,a4,a3 与 hot loop 2 f5cbb8: addi a4,a4,8、f5cbf8: fsw fa5,4(a5) 的 local sample share 应显著下降；flw/fsw 成对取用交织复数的模式消失
- 应出现：rvv_fft_transform_kernels.md §Verification 预期信号——vlseg2e32/vsseg2e32/vsetvli/vfmul 出现在蝶形 hot interval；objdump -d 核对非成对 flw/fsw + 标量复乘链
- 验证方法：同一 workload 重跑 annotate；checkasm SBR/AV_TX 正确性对照（re/im 逐 bin 比对、往返测试）；K3 上检查 VLEN=256 下 zvl256b 或 vsetvli 自适应效果

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：1/1 组；[ff_tx_mdct_inv_float_c]） | ✅ |
| 2 | Phase 1 输出要求：7 行 baseline + 2 个 L0 gate + bound gate + Sampling IP precision（载荷：7 项；gap 标签 baseline_gap: sampling IP precision） | ✅ |
| 3 | Phase 3 输出要求：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md；顶层 finding 1（primary FFT）+ supporting 1；evidence 锚点 f5cb38/f5cb50/f5cb2a/f5cbb8/f5cbf8；supporting 1；排除 7 条；推导式 2 条 | ✅ |
| 4 | Phase 4 输出要求：pattern rvv_fft_transform_kernels.md 已读；命中 row RVV FFT/DCT/MFCC Transform Kernels；引用短语「one butterfly per scalar op wastes vector width」；The fix before/after 完整；Related PRs：4 条 URL | ✅ |
| 5 | 路径合规：模式 A；class 扫描集 8 项可解释；单命中 + supporting 合规；L1 归属 | ✅ |
| 6 | Phase 5 两侧锚定：消失侧 f5cb38/f5cb50/f5cb2a/f5cbb8/f5cbf8；出现侧 rvv_fft_transform_kernels.md §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；无追问 | ✅ |

修正记录：无