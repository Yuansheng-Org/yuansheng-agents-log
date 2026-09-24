Functions under analysis: [saxpy_k]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 perf annotate | 已提供（`002-saxpy_k-annotate.txt`，459 行，覆盖 0x3fee–0x4378 全函数含 hot loop body；22 samples，event=cpu-clock，百分比为函数内局部份额） |
| perf stat | 已提供（`14-openblas-benchmark-riscv-sger_2048x2048.txt`：IPC 0.703、cpu_cycle 581,169,346、instruction 408,798,179、L1_dcache_load_miss_rate 0.053%、branch_miss_rate 0.395%、threads 1、gflops 3.19） |
| workload/binary/DSO/source context | 已提供（metadata.json：software=openblas、commit 70c2410f17b60575f34dbb7eac88d16a9f485861、branch develop；DSO=sger.goto；热点实现对应 workspace 源码 `OpenBLAS/kernel/riscv64/axpy_vector.c`，annotate 源码行 FLOAT_V_T/VLEV_FLOAT/VFMACCVF_FLOAT 与该文件逐字一致） |
| readelf -A（build ISA） | 缺失（metadata warnings: "No ELF executable binaries were found for this run"；annotate 已证明 build 含 RVV 1.0 `v*` mnemonic） |
| hardware ISA | 已提供（metadata cpuinfo.isa：rv64imafdcvh_zicbom_..._zve64d_..._zvbb_...，含 `v`/zve32f/zve64d 等） |
| vlenb | 已提供（metadata vector：vlen_bits=256、vlenb=32） |
| 采样元数据 | 部分（event=cpu-clock，可解释为时间；annotate 百分比为函数内局部份额（22 samples 合计 100%）；同一运行窗口；函数全局贡献已知：perf report 显示 saxpy_k=16.54%） |
| Sampling IP precision | 缺失（cpu-clock 采样，无 precise_ip/Exact-IP 记录） |

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcvh_...`（含 `v`、zve32f/zve32x/zve64d/zve64f/zve64x、zvbb/zvbc/zvkg/zvkned/zvknha/zvknhb/zvksed/zvksh/zvkt、zfa/zfh/zfhmin、zba/zbb/zbc/zbs 等；SpacemiT X100，OoO，VLEN=256）→ 硬件暴露 RVV 1.0 |
| Build ISA | `baseline_gap: build ISA`（无 ELF binary 可 readelf -A）；但 annotate 反汇编含 `vle32.v`/`vfmacc.vf`/`vsetvli e32,m2` 等 RVV 1.0 mnemonic → build 已包含 `v`；源码宏 `RISCV64_ZVL256B → LMUL m2` 与反汇编 e32,m2 一致 |
| Vector flavor | RVV 1.0（`v*` mnemonic），无 `th.v*` → 无 flavor mismatch |
| VLEN | 256 bits（vlenb=32）→ e32,m2 时 VLMAX=16 元素/vector op |
| Bound type | compute/latency-bound：L1_dcache_load_miss_rate 0.053%（工作集 x 列 8KB + y 行 8KB，L1 驻留）、branch_miss 0.395%（非 branch-bound）、全局 IPC 0.703；saxpy 主循环非 memory-bound、非 branch-bound |
| Sampling semantics | event=cpu-clock（时间事件）；annotate 为函数内局部份额（22/22=100% 落在主循环区间）；同一窗口；函数级 workload 贡献=16.54%（perf report 全局份额）→ 收益上界按函数全局份额表述 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（cpu-clock、无 precise_ip 记录）→ 单行高占比只锚定 hot loop interval，不做单指令延迟归因 |

L0 baseline gate：hardware 有 `v` ✓、build 有 `v`（annotate 证明）✓ → 无 hardware/build mismatch；无 `th.v*` → 无 flavor gate 冻结。Bound-type gate：compute/latency-bound（非 memory-bound）→ 本地 RVV 重调优不被 memory-bound 降级。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：saxpy_k（1 个）。
hot loop interval：`0x4040–0x406e`（`inc_x==1 && inc_y==1` 快速路径主循环，每轮处理 2*gvl=32 元素，n=2048 时 64 次迭代，无 tail 执行）。
最高占比行（trace anchor，受 IP precision 限制仅锚定区间）：`31.82 :  4066: vfmacc.vf       v2,fa0,v4`；同区间其它行：`27.27 :  405c: add a2,a2,t4`、`18.18 :  4044: vle32.v v2,(a2)`、`18.18 :  4048: add a7,a2,t3`、`4.55 :  4062: vle32.v v2,(a7)`。
区间内样本构成：标量地址算术 45.45%（4048+405c）、vfmacc 31.82%（4066）、y-load 22.73%（4044+4062）。
`baseline_gap: sampling IP precision` → 区间级机制结论，不归因单条指令延迟。

## Phase 3 — Pattern scan / 模式扫描：saxpy_k

### Class selection trace（8 项）
1. `rows-asm.md` — exclude：当前代码来源是 compiler/intrinsic 生成的 RVV（workspace 源码 `kernel/riscv64/axpy_vector.c` 用 `__riscv_v*` intrinsic，annotate 源码行逐字匹配），非手写 `.S`；无 policy-backed missing `.S` 四证（riscv64 端口已有 intrinsic 实现且可达）。
2. `rows-operator-rvv.md` — include：saxpy 属 elementwise 算术（y=da*x+y）；检查已向量化失效形态 → 扫描后排除：vle32/vfmacc.vf/vse32 是正确 direct RVV lowering，无 missing-lowering/mask-select 形态。
3. `rows-string-memory.md` — exclude：非 string/memory 原语（FP 算术流）。
4. `rows-vectorized-tuning.md` — include（必选）：完整 annotate 已有 `v*` 且非 `.S`，修正对象是 RVV 配置/寄存器组/展开/policy → **命中 register-group row**。
5. `rows-codegen.md` — include：检查 kernel-selection（inc_x/inc_y 运行时特化已在 0x4008/0x400a 正确命中快速路径，无 missed kernel）、IV-strength-reduction（循环已用指针递增 `add a2,a2,t4`，无 index scaling）、register-pressure（hot loop 无 spill/reload）→ 均不命中。
6. `rows-offload.md` — exclude：无矩阵引擎；saxpy 非 GEMM/卷积/矩阵卸载。
7. `rows-crypto.md` — exclude：非密码原语。
8. `rows-runtime-os.md` — exclude：非 OS/RTOS/timer 热点。

`Classes scanned:` rows-vectorized-tuning.md、rows-operator-rvv.md、rows-codegen.md

### Local performance pattern scan: `saxpy_k`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | 主循环 `vsetvli zero,a4,e32,m2,ta,ma`（0x403c）；峰值 live vector 仅 v2+v4（2 个）；`LMUL*peak_live=2*2=4 ≤ 32`，m4/m8 均合法无 spill；循环迭代 64 次（n=2048）；标量地址算术占区间样本 45.45%，vfmacc 31.82%，y-load 22.73% | High | Medium | `patterns/rvv_register_group_utilization.md` |

**三件套（primary：RVV Register-Group Utilization and LMUL Sizing）**

**(a) 逐字 evidence 引用**（hot loop interval 0x4040–0x406e，22 samples，函数内局部份额）：
- `27.27 :  405c: add     a2,a2,t4`（y 指针更新，loop-carried，6 samples）
- `18.18 :  4048: add     a7,a2,t3`（第二块 y[j+gvl] 地址重算，4 samples）
- `31.82 :  4066: vfmacc.vf       v2,fa0,v4`（第二块 FMA，7 samples）
- `18.18 :  4044: vle32.v v2,(a2)`（vy0 load，4 samples）
- `4.55 :  4062: vle32.v v2,(a7)`（vy1 load，1 sample）
区间内标量算术行合计 10/22=45.45%；IP precision 不足 → 锚定区间整体，结论为 interval-level mechanism。

**(b) 互斥邻居排除**：
- no-vectorization：主循环含 vle32.v/vfmacc.vf/vse32.v（`v*` 存在）→ 非 scalar main loop。
- maximal-LMUL byte row：e32 FP 流，非 byte/misaligned 路径 → 不适用。
- operand-form selection：vfmacc.vf 直接消费标量 fa0（正确 vf 形态），主循环无 vmv.v.x/vfmv.v.f temporary → 排除。
- vector-state management：vsetvli 在循环外单次建立（0x403c），主循环内无重复 vsetvl* → 排除。
- inactive-lane policy：vtype 为 ta,ma，无 vmclr.m/旧值保留 → 排除。
- register-pressure/spill：hot loop 内无 sd/ld spill/reload（0x407e+ 冷路径保存 s0-s2 不属热区）→ 排除。
- kernel-selection：inc_x==1&&inc_y==1 快速路径被 0x4008/0x400a 正确命中，非 fallback → 排除。
- unroll row（`rvv_register_budgeted_loop_unrolling`）：按 arbitration，「更大 LMUL 合法、无 spill 且直接减少迭代数时由 register-group 认领、unroll 延后」——LMUL=m2 未达合法边界（m4/m8 合法）→ 本 row 不通过 gate，unroll 维度并入 primary 的 frontier 枚举。

**(c) 双 Confidence 推导式**：
- route: 主循环 v* provenance（annotate）+ 源码确认 intrinsic 生成（axpy_vector.c）+ `vsetvli e32,m2` + peak live=2、`LMUL*peak_live=4≤32` + 45.45% 标量算术样本 → **High**（所有 route gate 成立且有直接证据）。
- impact: 函数全局样本份额 16.54%（perf report）+ 区间局部 100% + VLEN=256 已知 + bound type 已知（compute/latency-bound）；但 `baseline_gap: sampling IP precision`（annotate 局部百分比）+ `baseline_gap: build ISA` + 无 A/B benchmark → **Medium**（有竞争 bottleneck 考量：OoO 核可部分隐藏标量开销，m8 未必最优）。

**多命中仲裁小段**：仅 1 个顶层命中（register-group sizing），无 primary/companion 并列。unroll row 未通过 gate（LMUL 未达边界），按 arbitration L3 归属 register-group sizing 认领 root cause，unroll 延后为 frontier 枚举维度；IV-strength-reduction 已满足（指针递增已在用）不命中。evidence mechanism layer = L3（vector/runtime configuration），无 L0/L1/L2 命中。

## Phase 4 — Root-cause blueprint / 根因蓝图：saxpy_k

1. **Root cause**：saxpy_k 主循环（inc_x==1 && inc_y==1）以 `e32,m2`（VLEN=256 → VLMAX=16 元素/op）运行，每轮仅处理 2*gvl=32 元素、n=2048 需 64 次迭代；峰值 live vector 只有 v2（vy）与 v4（vx）两个，`LMUL * peak_live_vectors = 2*2 = 4 ≤ 32`，架构预算远未用满——**寄存器组候选边界欠利用**（依据 `patterns/rvv_register_group_utilization.md` §Why this is slow 第 1 条 "Underutilized register-group frontier / 寄存器组候选边界欠利用"）。后果是循环控制开销按 64 次迭代重复累积：区间样本 45.45% 落在标量地址算术（405c y 指针更新 + 4048 第二块地址重算），加上仅 2 条独立 load→FMA→store 链（vfmacc 31.82% + y-load 22.73%）在 OoO 核上形成 latency/issue 约束；L1 miss 0.053% 证明非 memory-bound，属 compute/latency-bound 欠利用形态。
2. **The fix / 修复方式**：保持 fixed-VL main loop 结构（vsetvli 已在循环外）与 tail 循环不变，按 live-set 预算**提高主循环 LMUL**：峰值 live=2 vector 时 `LMUL * 2 ≤ 32` → m4/m8 均合法且无 spill（m8 用 16 个 vreg，仍余 16）。生产实现先枚举 `m1/m2/m4/m8 × unroll=1/2/4/8` 合法 frontier，A/B 实测选定，**不得预设 m8 最优**（依据 pattern §Presenting "更大 LMUL 并不必然更快…最终结论应依据生成指令、register spill 检查、正确性对照和目标平台 benchmark"）。修复前（当前形态，来自 `axpy_vector.c:80-96`）：
   ```c
   #ifdef RISCV64_ZVL256B
   #       define LMUL m2      /* VLEN=256 时 e32,m2 → VLMAX=16 */
   #else
   #       define LMUL m4
   #endif
   gvl = VSETVL(n);            /* e32,m2 → 16 */
   if (gvl <= n/2) {
       for (i = 0, j = 0; i < n/(2*gvl); i++, j += 2*gvl) {
           vx0 = VLEV_FLOAT(&x[j], gvl);      vy0 = VLEV_FLOAT(&y[j], gvl);
           vy0 = VFMACCVF_FLOAT(vy0, da, vx0, gvl);  VSEV_FLOAT(&y[j], vy0, gvl);
           vx1 = VLEV_FLOAT(&x[j+gvl], gvl);  vy1 = VLEV_FLOAT(&y[j+gvl], gvl);
           vy1 = VFMACCVF_FLOAT(vy1, da, vx1, gvl);  VSEV_FLOAT(&y[j+gvl], vy1, gvl);
       }
   }
   ```
   修复后（代表形态，LMUL 提升使 gvl 自动变大、迭代数同比减少）：
   ```c
   /* 枚举 m1/m2/m4/m8 × unroll=1/2/4/8；peak live=2 vectors，
      m8 时 LMUL*peak_live=16 ≤ 32 合法无 spill；实测选定，不预设 m8。 */
   #ifdef RISCV64_ZVL256B
   #       define LMUL m4      /* 或 m8，由 A/B 决定；main loop 与 tail 结构不变 */
   #else
   #       define LMUL m4
   #endif
   gvl = VSETVL(n);            /* e32,m4 → VLMAX=32（m8 → 64） */
   if (gvl <= n/2) {
       for (i = 0, j = 0; i < n/(2*gvl); i++, j += 2*gvl) {
           /* 同前：vx0/vy0 与 vx1/vy1 两组 fixed-VL block；
              指针步长 t4=2*gvl*4 自动放大，标量地址算术每元素摊薄 2~4 倍 */
       }
   }
   /* tail: for(; j<n;) 保持 runtime-VL 覆盖剩余元素 */
   ```
   适用前提：build 含 RVV 1.0（annotate 已证明）；目标 VLEN=256；e32/SGE 路径；RISCV64_ZVL256B 宏 gate 保持。不可破坏的 correctness contract：① 每元素 `y[i]=da*x[i]+y[i]` 的 FP 语义与 vfmacc.vf 的 FMA/rounding 行为不变（主循环无跨 lane 归约，无 reassociation 问题）；② tail 循环 `for(; j<n;)` 用 runtime-VL 精确覆盖剩余元素；③ `gvl <= n/2` 分支与 `da==0`/`n<=0` 提前返回不变；④ inc_x/inc_y 各 stride 分支（0x407e 起的 4 个特化路径）不动。限制/风险：m8 在 OoO 核可能增加单 vector op 的 latency/LSU 占用，需实测；跨 VLEN（非 256B 构建）已用 m4，不应被本改动破坏；unroll>1 时需同步 step/tile 元数据。预期 Profile signals：主循环 vsetvli 显示 e32,m4/m8；405c/4048 标量算术样本占比从 45.45% 显著下降；函数总指令数与迭代数减半/减四分之一；saxpy_k 全局份额从 16.54% 下降、IPC 上升。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（含 v，RVV 1.0）；build ISA=`baseline_gap: build ISA`（annotate 证明含 v，e32,m2 与源码宏一致）；VLEN=256（vlenb=32）；bound type=compute/latency-bound（L1 miss 0.053%）。
4. **收益上界**：入口模式 A。saxpy_k 全局样本份额 16.54%（perf report 同窗口 cpu-clock）；主循环区间占函数局部 100%（22/22）。表述：**当前 sampled event 下 saxpy_k 的全局份额 16.54%，主循环局部 100%；benefit upper bound ≈ 0.1654**。受 `baseline_gap: sampling IP precision`（annotate 局部百分比）约束，不宣称超出函数级份额的 workload Amdahl 上界。
5. **三维路由判定**：`current source` = compiler/intrinsic 生成的 RVV（`kernel/riscv64/axpy_vector.c`，annotate 源码行逐字匹配）→ 非 `.S`；`implementation existence/reachability` = RVV intrinsic kernel 即当前执行实现，可达（无 missed dispatch）；`function-level policy` = OpenBLAS riscv64 端口 intrinsic-kernel 政策（`RISCV64_ZVL256B` 宏选 LMUL），无 assembly-default 政策 → **不进入 policy-backed missing `.S` 分支**。
6. **Implementation-shape proof**：不适用（非 missing `.S` 分支）。
7. **Related PRs**（来自 `patterns/rvv_register_group_utilization.md` §Related PRs）：
   Related PRs：13 条 URL — https://github.com/opencv/opencv/pull/26318 ；https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5 ；https://github.com/opencv/opencv/pull/25586 ；https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c ；https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0 ；https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05 ；https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e ；https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870 ；https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b ；https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3bca6046cb17 ；https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946932df7 ；https://github.com/v8/v8/commit/3844339936068c529170dcb4f2aa160654d25943 ；https://github.com/vllm-project/vllm/pull/47538

## Phase 5 — Verification forecast / 验证预测：saxpy_k

primary（RVV Register-Group Utilization and LMUL Sizing）：
- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`405c: add a2,a2,t4`（27.27%）与 `4048: add a7,a2,t3`（18.18%）的样本占比显著下降（LMUL 提升后迭代数 64→32/16，标量地址算术执行次数同比减少）；`4066: vfmacc.vf` 与 y-load 行的绝对样本数随循环指令总数下降。
- **应出现侧**（锚定 `patterns/rvv_register_group_utilization.md` §Verification）：主循环 `vsetvli` 显示候选更高 LMUL（e32,m4 或 m8）；`m1/m2/m4/m8 × unroll=1/2/4/8` 候选全部无 vector spill/reload（"候选 frontier 验证"+"register spill 验证"）；同输入输出逐元素一致（"正确性对照"）；覆盖 n=0、小 n、n=2048（2*gvl 整倍数）、非整倍数 tail 与不同 VLEN（"长度与 VLEN 无关性验证"）；真实 workload 重跑 annotate 确认 405c/4048 份额缩小、saxpy_k 全局份额下降（"指令验证"）。
- 验证顺序：先 frontier 枚举与 spill 检查 → A/B benchmark（m2 vs m4 vs m8）→ 正确性对照 → sger_2048x2048 全量重跑（perf stat + annotate）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（载荷） |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 | ✅（1/1 组；saxpy_k） |
| 2 | Phase 1 输出要求：7 行 baseline 表 + Sampling IP precision 行 + 2 个 L0 gate + bound-type gate | ✅（7 行结论；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求：8 项 Class selection trace + Classes scanned + 顶层 finding 数 + evidence 锚点 + 互斥排除 | ✅（8 项 trace；`Classes scanned: rows-vectorized-tuning.md、rows-operator-rvv.md、rows-codegen.md`；顶层 finding=1（register-group）；evidence 锚点 5 行：`405c add a2,a2,t4`、`4048 add a7,a2,t3`、`4066 vfmacc.vf v2,fa0,v4`、`4044 vle32.v v2,(a2)`、`4062 vle32.v v2,(a7)`；supporting=0；排除 8 条（no-vectorization/max-LMUL-byte/operand-form/vector-state/inactive-lane/register-pressure/kernel-selection/unroll）；推导式 2 条（route High / impact Medium）） |
| 4 | Phase 4 输出要求：已读 pattern 文件 + 引用短语 + The fix 四要素 + missing `.S` 分支 + Related PRs | ✅（已读 `patterns/rvv_register_group_utilization.md`、`patterns/rvv_register_budgeted_loop_unrolling.md`；引用短语首词："Underutilized register-group frontier"、"不得默认 m8 或更大 LMUL 最优"；The fix 含 before/after、correctness contract、风险、预期 Profile signals；missing `.S` 不适用；`Related PRs：13 条 URL`） |
| 5 | 路径合规：trace 可解释扫描集 + 单/多命中仲裁 + 动态份额排序 | ✅（模式 A；8 项 trace；1 顶层命中来自通过 gate 的 row；unroll 按 L3 归属延后；按动态份额排序：局部 100%、全局 16.54%） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧锚 Phase 3(a) 的 405c/4048/4066；出现侧锚 `rvv_register_group_utilization.md` §Verification） |
| 7 | 契约边界合规：无实施询问/代码修改/补丁生成；无向用户追问 | ✅ |

修正记录：无