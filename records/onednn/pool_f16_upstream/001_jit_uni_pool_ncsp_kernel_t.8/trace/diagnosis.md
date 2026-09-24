Functions under analysis: [jit_uni_pool_ncsp_kernel_t.8]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`jitted-14255-8.so`，67 samples，event=cpu-clock:u，percent type=local period；覆盖完整 0x80–0x24c，含 hot window-sweep loop 0x150–0x17c 与 per-channel epilogue 0x1a8–0x1d0）
- perf stat（bound/context）：已提供（`8-onednn-benchdnn-benchmark-riscv-pool_f16_upstream.txt`，workload 级聚合）
- workload/binary/DSO/source context：已提供（oneDNN main @ d22de940f301e97591e04a2cc6f0010c52109ac7；`src/cpu/rv64/jit_uni_pool_kernel.cpp` `generate_f16()`；JIT DSO 内 perf 无法解码 RVV 指令，逐条显示为 `.insn 4, 0x...`，已用 xbyak_riscv 编码（`third_party/xbyak_riscv/xbyak_riscv/xbyak_riscv_v.hpp`）逐条解码确认）
- readelf -A（承载热点地址的 object 的 Tag_RISCV_arch）：缺失（JIT 运行期生成的 DSO，无静态 ELF；`metadata.binaries = {}`；详见 Phase 1）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata cpuinfo.isa：`rv64imafdcv_..._zvfh_zvfhmin_...` → RVV 1.0 + Zvfh/Zvfhmin）
- `vlenb`：已提供（vlenb=16 → VLEN=128 bits）
- 采样元数据（event / percent type / scope / 窗口）：部分（event=cpu-clock:u 可解释为时间；percent type=local period（非 global）；单运行窗口；函数占整个 workload 的贡献未知）
- Sampling IP precision：缺失（precise_ip/Exact-IP 能力未知 → `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt` → 标准 `v`（RVV 1.0）+ `zvfh`/`zvfhmin`（f16 向量 FP） |
| Build ISA | `baseline_gap: build ISA`（JIT DSO 无静态 ELF，readelf 不适用）。但生成代码本身即 evidence：hot loop 全部为 RVV 1.0 OP-V/vector-load 编码（opcode 0x57 / 0x07），含 `vle16.v`/`vlse16.v`/`vmflt.vv`/`vmerge.vvm`/`vmerge.vxm`，需要 `v`+`zvfh`；硬件两者都具备 → 无 hardware/build flavor mismatch |
| Vector flavor | RVV 1.0 `v*` mnemonic（perf 对 JIT DSO 不解码，显示为 `.insn 4, 0x...`；解码后全部为 RVV 1.0 编码，无 `th.v*`）→ 无 `vector_flavor_mismatch` |
| VLEN | vlenb=16 → 128 bits → e16/m1 = 8 f16 lanes/vector op；e8/mf2 = 8 u8 lanes |
| Bound type | IPC=0.845；L1_dcache_load_miss_rate=0.698%；LLC_load_miss_rate=13.44%；branch_miss_rate=0.177% → **compute/latency-bound**（非 memory/branch-bound）。注：derived `cache_miss_rate 100.015%`（cache_misses≈cache_references=5.5M）为 counter 口径异常，不作结论依据 |
| Sampling semantics | event=cpu-clock（时间可解释）；percent type=**local period**（非 global）；同一窗口；函数级 workload 贡献未知 → 收益上界只能表述为函数内局部样本份额，**不得称 workload 级 Amdahl 上界** |
| Sampling IP precision | `baseline_gap: sampling IP precision`（precise_ip 未知）→ 单指令高占比只能锚定所属 basic block / loop interval，不承担单指令 latency 归因 |

L0 baseline gate：hardware 有 `v`，JIT 生成代码也用 `v`/`zvfh` → 无 mismatch；无 `th.v*`；annotate 完整，入口模式 A（profile-backed）。
Bound-type gate：compute/latency-bound 成立（IPC<1 + 低 miss + 低 branch miss）→ 本地 RVV 结构优化是合法第一杠杆，performance-impact confidence 不受 memory-bound 压制。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`jit_uni_pool_ncsp_kernel_t.8`（oneDNN rv64 保留 native ncsp 池化 kernel，f16、max、forward-training，JIT 生成）。

两个 hot interval（按 cpu-clock 采样，67 samples 函数内局部份额）：
1. **Window sweep（iw 内层循环）0x150–0x17c**，聚合 ≈61.2%（41/67）：锚点 `34.33 :   164:    .insn   4, 0x6e441057`（解码 = `vmflt.vv v0, v4, v8`）。
2. **Per-channel epilogue 0x1a8–0x1d0**，聚合 ≈31.3%（21/67）：锚点 `22.39 :   1b8:    ld      t2,200(a0)`（= `ws_vec_byte_stride` 加载，`jit_primitive_conf.hpp` `jit_uni_pool_ncsp_args_t` offset 200）与 `8.96 :   1d0:    j       1d8`。

其余：prologue 0x84/0x98 ≈4.5%、channel advance 0x210 ≈1.5%、ih-loop back 0x190 ≈1.5%（冷路径，不作根因依据）。

Sampling IP precision 未确认 → 两条 interval 均以 basic-block/loop-interval 粒度归因；`0x164` 34.33% 锚定 compare→select 串行链区间，不断言 vmflt 单指令 latency。annotate 覆盖完整，无 coverage gap。

## Phase 3 — Pattern scan / 模式扫描：jit_uni_pool_ncsp_kernel_t.8

### Class selection trace（8 项）

| # | Class 文件 | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | rows-asm.md | exclude | 代码来源是 JIT 模板生成（xbyak_riscv），非手写 `.S`；无 missing-`.S` policy 主张（kernel 已存在且被选中执行） |
| 2 | rows-operator-rvv.md | include | JIT 生成代码、热点由算子语义（spatial max pooling + argmax）决定；逐 row 评估 spatial-pooling 与 arg-extrema |
| 3 | rows-string-memory.md | exclude | 无 copy/fill/sentinel/compare/checksum 语义 |
| 4 | rows-vectorized-tuning.md | include | 完整 annotate 有 `v*`（`.insn` hex 解码确认），非手写 `.S`；修正对象为 RVV 配置/寄存器组/展开/state |
| 5 | rows-codegen.md | include | JIT 生成代码；检查 JIT-codegen-quality row、control-flow row |
| 6 | rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 证据；pooling 非 GEMM |
| 7 | rows-crypto.md | exclude | 无密码学原语 |
| 8 | rows-runtime-os.md | exclude | 用户态 benchdnn workload，非 RTOS/kernel |

### Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md

### 指令解码（证据地基）

| annotate 行 | 解码（xbyak_riscv 编码核对） | 语义 |
|---|---|---|
| `0x158: .insn 0x0208d407` | `vle16.v v8, (a7)`（opVectorLoad 0x5007 + vm + a7<<15 + v8<<7） | 单位 stride f16 窗口加载（src_vec_byte_stride==2 分支） |
| `0x160: .insn 0x0b88d407` | `vlse16.v v8, (a7), s8`（0x8005007 + s8<<20…） | strided f16 窗口加载（ncsp：spatial_stride×2） |
| `0x164: .insn 0x6e441057` | `vmflt.vv v0, v4, v8`（opFVV 0x6c001057 + vm + v4<<20 + v8<<15 + v0<<7 = 0x6e441057） | mask = (acc < tmp)，**strict <（first-wins tie）** |
| `0x168: .insn 0x5c440257` | `vmerge.vvm v4, v4, v8`（0x5c000057 + … = 0x5c440257） | acc = mask ? tmp : acc（max 值选择） |
| `0x16c: .insn 0x5cae4557` | `vmerge.vxm v10, v10, t3`（0x5c004057 + v10<<20 + t3<<15 + v10<<7 = 0x5cae4557） | ind = mask ? cur_idx : ind（argmax 索引选择） |
| `0x1ac/0x1b4` | `vse16.v v4, (s1)` / `vsse16.v v4, (s1), s9` | dst 单位/strided f16 存储 |
| `0x1c0/0x1c4/0x1cc/0x1d4/0x1d8` | `vsetvli t3,s2,e8,mf2` / `vnsrl.wi v8,v10,0` / `vsse8.v v8,(s10),t2` / `vse8.v v8,(s10)` / `vsetvli t0,s2,e16,m1` | argmax u8 窄化+存储 + vtype 恢复 |

对应源码（`jit_uni_pool_kernel.cpp` `generate_f16()`）：iw_loop 行 1597–1624（vle16_v/vlse16_v 1604–1607；vmflt_vv 1611；vmerge_vvm 1612；vmerge_vxm 1614；addi(t3,t3,1) 1615）；epilogue 行 1674–1713（dst store、argmax narrow/store、vtype 恢复）。args 偏移（`jit_primitive_conf.hpp` 140–185 行）与反汇编逐一吻合：src=0…indices=168、pos_base=176、pos_ih_step=184、pos_id_step=192、ws_vec_byte_stride=200。

### Local performance pattern scan: jit_uni_pool_ncsp_kernel_t.8

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| **RVV Register-Budgeted Loop Unrolling（primary）** | window sweep 单 accumulator 串行 max+argmax recurrence：vmflt.vv@0x164 34.33% + vmerge.vvm@0x168 4.48% + vmerge.vxm@0x16c 8.96%；source 1610–1615 确认 v_acc/v_ind 各仅一个、严格 `<`、t3 单计数；可拆 2/4 路独立 (value,index) accumulator 且无 spill | High | Medium | `patterns/rvv_register_budgeted_loop_unrolling.md` |
| **RVV Register-Group Utilization and LMUL Sizing（companion）** | channel 向量固定 m1（VLEN=128→8 f16 lanes/chunk），per-chunk epilogue 含 2 次 vsetvli + vnsrl + strided ws/dst store，0x1a8–0x1d0 ≈31.3%（0x1b8 22.39%、0x1d0 8.96%）；m2/m4 合法候选（live set 小、无 spill） | High | Medium | `patterns/rvv_register_group_utilization.md` |
| RVV Spatial Convolution and Pooling Kernels（排除） | 已向量化且用直接 `vlse16`/`vsse16` strided 访问；行内判据 ①scalar hot path / ②vcompress-mask-deinterleave-roundtrip 均不成立 → row gate 不命中 | — | — | `patterns/rvv_spatial_convolution_and_pooling_kernels.md` |
| RVV Arg-Extrema Selection Kernels（排除，语义背景） | 语义上确实是 value+index 同步维护（每 lane 视角 = argmax）；但 row 面向 scalar→RVV 转换（"scalar loop"），kernel 已向量化，行内未明确接纳 already-vectorized 失效形态 → 不作 blueprint 来源 | — | — | `patterns/rvv_arg_extrema_selection_kernels.md` |
| No vectorization（排除） | 要求 hot main loop zero `v*`；本 kernel 全为 RVV | — | — | `patterns/no-vectorization.md` |
| RVV Vector-State Management（排除） | sweep 内全程单一 vtype（源码注释 1434–1437 明确为设计）；epilogue 的 e8/mf2 切换是 e16→e8 窄化的 **必需** SEW 改变，非兼容状态冗余重建 | — | — | `patterns/rvv_vector_state_management.md` |
| RVV Inactive-Lane Policy（排除） | vsetvli 用 VTA::ta/VMA::ma；masked vmerge 是语义必需（compare mask 驱动 select），无冗余 clear / tail 保留读取 | — | — | `patterns/rvv_inactive_lane_policy.md` |
| JIT-Generated Code Quality Optimization（排除） | emitter 生成的 loop body 已精简（无冗余 zext/mv/helper call/过宽 flush）；根因是算法/recurrence 结构而非 emit 冗余 | — | — | `patterns/jit_generated_code_quality_optimization.md` |
| RVV Operand-Form Selection（排除） | hot loop 内无 `vmv.v.x`/`vfmv.v.f` 临时广播再被单条 vv 消费；`vmerge.vxm` 已直接用 scalar 源 | — | — | `patterns/rvv_operand_form_selection.md` |

#### 顶层 finding 1：RVV Register-Budgeted Loop Unrolling（primary）

**(a) 逐字 evidence 引用**（所属 interval：window sweep 0x150–0x17c，聚合 61.2%）：
```
34.33 :   164:    .insn   4, 0x6e441057     → vmflt.vv v0, v4, v8    (acc < tmp)
 4.48 :   168:    .insn   4, 0x5c440257     → vmerge.vvm v4, v4, v8  (acc = max)
 8.96 :   16c:    .insn   4, 0x5cae4557     → vmerge.vxm v10, v10, t3 (ind = argmax)
 8.96 :   170:    addi    t3,t3,1
 0.00 :   150:    bge     a5,a2,180          (iw_loop back-edge，0 samples)
```
源码锚点（`jit_uni_pool_kernel.cpp` 1610–1615）：`vmflt_vv(v_mask, v_acc, v_tmp); vmerge_vvm(v_acc, v_acc, v_tmp); vmerge_vxm(v_ind, v_ind, t3); addi(t3, t3, 1);`

**(b) 互斥邻居排除**：
- **vs register-group row**：提升 LMUL 只加宽每向量 lane 数、减少 chunk 数，不改变固定 LMUL 下每个窗口元素的 compare→select **串行 recurrence 深度**（acc 跨迭代依赖）；本行证据正是指向该串行 recurrence（vmflt 34.33% 为链头、vmerge.vvm 结果喂给下一轮 vmflt），满足该 row "证据明确指向固定 LMUL 下的串行 recurrence/ILP 时有直接证据"的让位条款。
- **vs vector-state row**：sweep interval 内 0 条 `vsetvl*`（单一 e16/m1 vtype 贯穿窗口扫描，源码 1434–1437 注释为刻意设计），无状态重建问题。
- **vs inactive-lane row**：vta/ma + 语义 mask select，非 tail/clear 浪费。
- **vs no-vectorization**：主循环已 RVV（vle16/vlse16/vmflt/vmerge），非 scalar main loop。
- **vs operator semantic rows**：spatial-pooling row 判据 ①/② 不成立（见 scan 表）；arg-extrema row 面向 scalar→RVV 转换，已向量化形态不命中。

**(c) 双 Confidence 推导式**：
- `route: 已向量化 v* + JIT 模板 provenance（源码逐行对应）+ 反汇编解码确认单 (value,index) accumulator 串行链 + 合法 2/4 路候选（sweep 期间 live=v0,v4,v8,v10 共 4 组，2 路链 +2 组、4 路链 +6 组均 ≤ 32 且无 spill，post-op v24/v28 仅 sweep 后存活）→ High`
- `impact: 有 hot interval 局部份额（61.2%）+ VLEN=128 + bound type=compute/latency，但缺 Sampling IP precision（baseline_gap）与函数级 workload 份额（percent type=local）→ Medium`

#### 顶层 finding 2：RVV Register-Group Utilization and LMUL Sizing（companion）

**(a) 逐字 evidence 引用**（所属 interval：per-channel epilogue 0x1a8–0x1d0，聚合 31.3%）：
```
22.39 :   1b8:    ld      t2,200(a0)     → ws_vec_byte_stride 加载（argmax u8 存储路径，jit_primitive_conf.hpp offset 200）
 8.96 :   1d0:    j       1d8            → ws strided-store 分支后的跳转（vsse8@0x1cc 之后，skid 聚集点）
```
相邻：0x1c0 `vsetvli t3,s2,e8,mf2`、0x1c4 `vnsrl.wi v8,v10,0`（argmax e16→e8 窄化）、0x1cc/0x1d4 `vsse8.v/vse8.v`、0x1d8 `vsetvli t0,s2,e16,m1`（vtype 恢复）。源码锚点 1688–1713 行。epilogue 每 channel-chunk 执行一次，m1→每 chunk 8 f16 channel，C=128 时 16 chunk/输出像素。

**(b) 互斥邻居排除**：
- **vs unrolling row**：LMUL 提升减少 chunk 数与 epilogue/vsetvli 频率（迭代/控制开销），但不断言串行链缩短——链缩短由 finding 1 认领；两机制证据落在不同 interval（epilogue vs sweep），可独立分账。
- **vs vector-state row**：e8/mf2 两次 vsetvli 是 e16→e8 窄化与回存的必需 SEW 切换（窄化目的 SEW=e8），不是兼容状态冗余重建；消除它们的唯一途径是提升 LMUL 后减少 chunk 频率（fractional LMUL 规则仍要求切换）。
- **vs operand-form**：epilogue 无 vmv/广播临时体；chunk 初始化一次 `vmv_v_x`（每 chunk 一次）非 hot-loop 内形态问题。

**(c) 双 Confidence 推导式**：
- `route: annotate 已向量化 + vsetvli 显示固定 m1 + 每 chunk epilogue 开销可观测（31.3%）+ m2/m4 候选合法（sweep 峰值 live 4 组→m2=7 regs/m4=13 regs ≤ 32；窄化路径 e16m2→e8m1 / e16m4→e8m2 的 EMUL 倍率合法，对应 pattern §3 fractional-LMUL 规则）→ High`
- `impact: epilogue 局部份额 31.3% + 提高 LMUL 还减半/四分之一 sweep 向量指令总数（chunk 数 C/vl 下降），但 Sampling IP precision 未知、函数级份额未知、且 strided store 加宽后每 op 成本可能上升 → Medium`

#### 多候选仲裁小段

- 两 finding 证据落在**不相交地址区间**（sweep 0x150–0x17c vs epilogue 0x1a8–0x1d0），机制（串行 recurrence vs register-group/chunk 配置）、修复对象（accumulator 链结构 vs LMUL 与 epilogue 代码生成）、验证方法均可分离 → **primary + companion**（不同时合并为一个修复）。
- L0–L3 检查：无 hardware/build ISA mismatch（L0 不命中）；kernel 已注册且被 driver 调用执行（`jit_uni_pooling.cpp` 979 行 `pooling_train_max_ncsp<isa,d_type>(...)` 调用，无 dispatch/selection 问题，kernel-selection row 不命中）；数据访问为布局固有 strided（ncsp），非 vcompress/buffer 往返（L2 spatial row 不命中）。两 finding 均为 **L4**（compute/codegen 微结构 + RVV 配置）。
- 入口模式 A：按 evidence sample share 排序——finding 1 覆盖 sweep 61.2%（函数内局部），finding 2 覆盖 epilogue 31.3%；同一调用链不简单相加（修复相互独立但作用同一 kernel）。`dynamic priority: finding 1 > finding 2`（基于函数内局部样本份额）。

## Phase 4 — Root-cause blueprint / 根因蓝图：jit_uni_pool_ncsp_kernel_t.8

### Finding 1（primary）— Register-Budgeted RVV Loop Unrolling

1. **Root cause**：window sweep 是**单 accumulator 的 loop-carried 串行 max+argmax recurrence**。每窗口元素关键路径 = `vmflt.vv`（比较产生 mask）→ `vmerge.vvm`（选择出 v_acc）→ 下一轮 `vmflt.vv` 依赖 v_acc，链长 = K×(L_comp+L_sel)；argmax 链（`vmerge.vxm` v10）复用同一 compare mask，与值链并行但同样等待 compare。依据 `patterns/rvv_register_budgeted_loop_unrolling.md` §Why this is slow："归约类循环…单一 accumulator 让下一次 vector FMA/add 必须等待上一次结果…使用多个独立 accumulators 才能暴露可并行工作"。当前 emitter 把窗口固定为单链（t3 单计数、v_acc/v_ind 各一），是固定 LMUL（m1）下串行 recurrence/ILP 的直接证据。
2. **The fix**：保持 SEW=e16/LMUL=m1 与算法语义，把窗口扫描拆为 **2 路（或 4 路）独立 (value, index) accumulator 链**，每链各带独立运行位置计数，窗口结束后按 first-wins 合并。修复前/后形态：
   ```cpp
   // Before（jit_uni_pool_kernel.cpp generate_f16，1610-1615；单链）
   v_tmp = (s8==2) ? vle16.v(a7) : vlse16.v(a7, s8);
   v0   = vmflt.vv(v_acc, v_tmp);        // acc < tmp
   v_acc= vmerge.vvm(v_acc, v_tmp);      // max 值选择（serial！）
   v_ind= vmerge.vxm(v_ind, t3);         // argmax 索引选择
   t3  += 1;
   ```
   ```cpp
   // After（2 路链；伪代码——多独立 accumulator + 结尾合并）
   // 链 A 覆盖窗口位置 0,2,4,…（pos 低）；链 B 覆盖 1,3,5,…（pos 高）
   // 每链独立 (v_accX, v_indX, t3X)，vsetvli 一次、同一 vtype
   v_tmpA = load(a7);            v_tmpB = load(a7 + s3);
   v0A = vmflt.vv(v_accA, v_tmpA);  v0B = vmflt.vv(v_accB, v_tmpB);
   v_accA = vmerge.vvm(v_accA, v_tmpA);  v_accB = vmerge.vvm(v_accB, v_tmpB);
   v_indA = vmerge.vxm(v_indA, t3A);     v_indB = vmerge.vxm(v_indB, t3B);
   t3A += 2;  t3B += 2;
   // 窗口结束合并：strict < 保持 first-wins（链 A 位置更早，相等时保留 A）
   v0   = vmflt.vv(v_accA, v_accB);
   v_acc= vmerge.vvm(v_accA, v_accB);
   v_ind= vmerge.vvm(v_indA, v_indB);    // 向量-向量选择（emitter 已有 vmerge.vvm）
   ```
   - 适用前提：窗口位置可枚举（动态边界 iw_start/iw_end 由 driver 传入，循环内按固定步长 2 取对即可）；K≥2（K=1 时单链即可）。
   - Correctness contract（不可破坏）：(i) **first-wins tie rule**——现实现用 `vmflt`（strict `<`），相等时保留低索引；合并处同样用 strict `<` 且在链 A（更低位置）保留相等；(ii) 全核相对窗口索引 pos_base + 运行偏移 + pos_ih_step/pos_id_step 跳过的 clamped 位置必须对两链分别正确推进（每链独立计数）；(iii) argmax 索引域 e16、u8 ws 窄化路径不变；(iv) NaN 语义不变（`vmflt` 对 NaN 为 false，NaN 永不成为 max）。
   - 限制/风险：峰值 live 从 4 组升到 2 路 8 组 / 4 路 16 组（≤32，无 spill）；代码尺寸增大、frontend/I-cache 压力；奇窗口元素需单链收尾；合并代价每 chunk 固定 2–3 条向量指令（相对 K 次窗口迭代可摊薄）。
   - 预期 Profile signals：`0x164` vmflt 局部占比显著下降；sweep interval 内指令间重叠增加；cycles/element 下降；两链指令各自分到 sample。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv+zvfh（RVV 1.0）；build ISA = `baseline_gap: build ISA`（JIT DSO，生成代码已用 v/zvfh）；VLEN = 128；bound type = compute/latency-bound（IPC 0.845）。
4. **收益上界**：当前 sampled event（cpu-clock）下函数内**局部样本份额** = window sweep interval 61.2%（41/67）。`baseline_gap: sampling metadata`（percent type=local period）→ 不得称 workload 级 Amdahl 上界。
5. **三维路由判定**：`current source` = JIT 模板生成（xbyak_riscv，`src/cpu/rv64/jit_uni_pool_kernel.cpp` `generate_f16`）；`implementation existence/reachability` = kernel 已生成并被 `pooling_train_max_ncsp`（`jit_uni_pooling.cpp:979`）选中执行，profile 直接落在其生成代码上；`function-level policy` = 无独立 `.S` 载体要求，native RVV JIT 是 retained 路径（`jit_uni_pool_kernel.hpp` 175–181 行注释），修复对象是 emitter 模板而非新 kernel 载体。
6. **Related PRs**（`patterns/rvv_register_budgeted_loop_unrolling.md`）：`Related PRs：2 条 URL` — https://github.com/google/XNNPACK/commit/0c7b565c2fc1debcb12a4dc8941a56c9f0ced6fe ；https://github.com/google/XNNPACK/pull/10403

### Finding 2（companion）— RVV Register-Group Utilization and LMUL Sizing

1. **Root cause**：channel 向量固定 `e16/m1`（VLEN=128 → 8 lanes/chunk），而 sweep 期间峰值 live 仅 4 组（v0,v4,v8,v10），m2/m4 合法。m1 使每输出像素的 chunk 数 = C/8，per-chunk epilogue（2×vsetvli + vnsrl + strided ws/dst store + 指针推进）与 ch_loop 迭代开销按 chunk 数重复。依据 `patterns/rvv_register_group_utilization.md` §Why this is slow 第 1/2 条："当前 LMUL 小于合法且无 spill 的候选边界时，单次迭代的有效元素数偏低，循环、`vsetvl` 和分支开销按更多迭代重复发生"。
2. **The fix**：将 channel 向量 LMUL 提升到 **m2（候选首选）或 m4**，寄存器组相应重映射（v_acc=v4(+1/+3)、v_tmp=v8(+1/+3)、v_ind=v10(+1/+3)），vsetvli 与窄化存储路径按 pattern §3 的 **fractional-LMUL 规则**调整：
   ```cpp
   // Before：e16m1 → 8 f16 lanes/chunk
   vsetvli(t0, s2, e16, m1);                 // sweep
   ...epilogue: vsetvli(t3,s2,e8,mf2); vnsrl.wi(v_tmp,v_ind,0); vsse8/vse8; vsetvli(t0,s2,e16,m1);
   // After：e16m2（或 m4）→ 16（32）lanes/chunk
   vsetvli(t0, s2, e16, m2);                 // sweep 加宽
   ...epilogue: vsetvli(t3,s2,e8,m1);  vnsrl.wi(v_tmp,v_ind,0);  // e16m2→e8m1 窄化，EMUL 2:1 合法
                vsse8.v(v_tmp, s10, t2);  vsetvli(t0,s2,e16,m2); // 恢复
   ```
   - 适用前提：VLEN=128、硬件支持 `v`+`zvfh`（已满足）；LMUL×peak_live ≤32（m2=7 组、m4=13 组 ≤32）；group 对齐与 EMUL 倍率合法（narrow：dest m1/m2 = src m2/m4 之半）；post-op 缓冲 v24/v28 仅在 sweep 后存活，m4 下与 v_ind 组需在 post-op 阶段避免冲突或重映射。
   - Correctness contract：vsetvli 动态 AVL（s2 剩余 channels）保持 tail 正确；strided vlse16/vsse16/vsse8 语义与地址不变（每 lane 独立 stride）；argmax u8 窄化逐元素位型不变。
   - 限制/风险：更宽 strided store 的每 op 成本可能上升（vsse16/vsse8 覆盖更多分散地址）；代码尺寸；必须以实机 benchmark 验证 m2/m4 不劣于 m1（pattern 明确"更大 LMUL 不必然更快"）；m4 时 post-op 寄存器冲突需处理。
   - 预期 Profile signals：epilogue interval（0x1b8/0x1d0 聚集）份额下降；ch_loop 迭代数与 `vsetvli` 出现次数按 C/vl 比例减少；sweep 向量指令总数减半（m2）。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv+zvfh；build ISA = `baseline_gap: build ISA`（JIT）；VLEN = 128；bound type = compute/latency。
4. **收益上界**：当前 sampled event 下函数内**局部样本份额** = epilogue interval 31.3%（21/67）；另 m2 使 sweep 向量指令总数减半（chunk 数 C/16）。`baseline_gap: sampling metadata` → 不得称 workload 级 Amdahl 上界。
5. **三维路由判定**：`current source` = JIT 模板生成（register 常量 v_acc/v_tmp/v_ind 与 vsetvli 调用点在 `generate_f16`）；`implementation existence/reachability` = 同一生成 kernel，仅改 emitter 配置；`function-level policy` = 无独立载体要求。
6. **Related PRs**（`patterns/rvv_register_group_utilization.md`）：`Related PRs：13 条 URL` — https://github.com/opencv/opencv/pull/26318 ；https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5 ；https://github.com/opencv/opencv/pull/25586 ；https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c ；https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0 ；https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05 ；https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e ；https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870 ；https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b ；https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3bca6046cb17 ；https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946932df7 ；https://github.com/v8/v8/commit/3844339936068c529170dcb4f2aa160654d25943 ；https://github.com/vllm-project/vllm/pull/47538

## Phase 5 — Verification forecast / 验证预测：jit_uni_pool_ncsp_kernel_t.8

- **Finding 1 消失/缩小侧**：`0x164: .insn 0x6e441057`（vmflt.vv v0,v4,v8）局部占比应显著下降，sweep interval（0x150–0x17c）内串行链指令（vmflt/vmerge 序列）不再主导该区间。**应出现侧**（依据 `patterns/rvv_register_budgeted_loop_unrolling.md` §Verification）：反汇编显示 2/4 个独立 accumulator 链（v_accA/v_accB、v_indA/v_indB）与结尾合并（vmflt.vv + 两次 vmerge.vvm）；backedge/control 指令数与 per-element 串行链深度下降；无新 vector spill/reload；tie test 全部一致。正确性覆盖：n=0、K=1、K 奇偶、所有 tail 长度、不同 VLEN（128 实测）。
- **Finding 2 消失/缩小侧**：epilogue interval（`0x1b8: ld t2,200(a0)` 与 `0x1d0: j 1d8` 聚集）份额下降；vsetvli 出现次数按 C/vl 减少。**应出现侧**（依据 `patterns/rvv_register_group_utilization.md` §Verification）：主循环 `vsetvli` 使用 m2/m4；窄化路径 vnsrl 的 dest/src EMUL 倍率合法（e8m1/e16m2 或 e8m2/e16m4）；无跨宽度插入的 `vlmul_ext`/`vlmul_trunc`；逐元素输出与 m1 baseline 一致；短/中/长 C 与全部 tail 覆盖；实机 A/B m1/m2/m4。
- **升级 evidence 所需最小补采**：`perf report --header-only` / `perf evlist -v` 确认 `precise_ip` 与 event attr（解除 `baseline_gap: sampling IP precision`）；同 event、`--percent-type=global-period` 重采以获得函数级样本份额与可比较占比。
- 两 finding 各自独立修复对象与验证；不批量应用后单次测量替代。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status / Anchor |
|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 | ✅（1/1 组；`jit_uni_pool_ncsp_kernel_t.8`） |
| 2 | Phase 1 输出要求 | ✅（7 行 baseline；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`；含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求 | ✅（8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md`；顶层 finding 2（primary+companion）；evidence 锚点：`34.33 : 164: .insn 4, 0x6e441057`、`22.39 : 1b8: ld t2,200(a0)`；排除条数 7；无 supporting；推导式 2 条） |
| 4 | Phase 4 输出要求 | ✅（已读 pattern：`patterns/rvv_register_budgeted_loop_unrolling.md`（命中 row；引用短语：`单一 accumulator`、`多个独立 accumulators`）+ `patterns/rvv_register_group_utilization.md`（命中 row；引用短语：`vsetvl 和分支开销按更多迭代重复发生`、`Match fractional LMUL`）；The fix 含 before/after、correctness、风险、Profile signals 锚点；Related PRs：2+13 条 URL） |
| 5 | 路径合规 | ✅（入口模式 A；8 项 trace；L4 归属 2 项；每 blueprint leaf 来自通过 gate 的 row；`dynamic priority: finding 1 > finding 2` 基于函数内局部份额；无 `th.v*` 停扫） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧：`0x164: .insn 4, 0x6e441057`、`0x1b8: ld t2,200(a0)`、`0x1d0: j 1d8`；出现侧：两个 pattern 文件 §Verification） |
| 7 | 契约边界合规 | ✅（无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix 与验证预测） |

修正记录：无