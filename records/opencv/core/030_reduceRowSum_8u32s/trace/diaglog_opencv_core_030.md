Functions under analysis: [cv::cpu_baseline::reduceRowSum_8u32s(cv::Mat const&, cv::Mat&)::{lambda(cv::Range const&)#1}::operator()(cv::Range const&) const [clone .isra.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`030-cv：：cpu_baseline：：reduceRowSum_8u32s(...)-annotate.txt`，414 samples，event=cpu-clock，percent type=local period，覆盖主行累加循环 0x2f4aca–0x2f503c）
- perf stat（可选 bound/context）：已提供（`7-opencv-perf-benchmark-riscv-core.txt`，IPC=0.917553，含 L1-dcache/branch counters）
- workload/binary/DSO/source context：已提供（`libopencv_core.so.5.1.0`，`opencv_perf_core`；源码 `modules/core/src/reduce.simd.hpp:797-884` 已核对）
- readelf -A（热点 object 的 Tag_RISCV_arch）：已提供（metadata `binaries.opencv_perf_core-elf-A`，含 v1p0/zve32f/zve64d/zvl128b）
- hardware ISA（/proc/cpuinfo 或 hwprobe）：已提供（metadata `cpuinfo.isa`，SpacemiT X100，含 `v`/RVV 1.0）
- vlenb：已提供（metadata vector.vlen_bits=256 → vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=cpu-clock，percent=local period，单次运行；函数级 workload 贡献未知，详见 Phase 1）
- Sampling IP precision（precise_ip / Exact-IP / skid）：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | rv64imafdcvh_...（`v` RVV 1.0、zve32f、zve64d、zvl128b、zfh/zfa、zbb/zba/zbs 等）；metadata cpuinfo.isa |
| Build ISA | `opencv_perf_core-elf-A` Tag_RISCV_arch = rv64i2p1_..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0... → 含 `v`；无 IFUNC/multiversion 证据 |
| Vector flavor | annotate 全部为 RVV 1.0 `v*` mnemonic（vsetvli/vle8.v/vle16.v/vwcvtu.x.x.v/vsaddu.vv/vse16.v/vse32.v/vzext.vf4/vid.v/vmul.vx），无 `th.v*` |
| VLEN | vlenb=32 → VLEN=256 bits；vlanes8=32（e8,m1 VLMAX）、vlanes16=16（e16,m1 VLMAX）、vlanes32=8（e32,m1 VLMAX） |
| Bound type | IPC=0.917553、L1_dcache_load_miss_rate=1.359%、branch_miss_rate=1.673% → compute/latency-bound（非 memory/branch-bound）；该函数内实际向量数据处理仅约 6% 样本，其余为配置/记账开销 |
| Sampling semantics | event=cpu-clock；percent type=local period；单次运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为当前 event 的函数内局部份额 → `baseline_gap: sampling metadata`（函数级贡献） |
| Sampling IP precision | `precise_ip`/Exact-IP 能力未提供 → `baseline_gap: sampling IP precision`；单指令归因受限，全部结论锚定 basic block / loop interval |

L0 baseline gate：hardware 有 `v` 且 build 有 `v` → 无 hardware/build mismatch；annotate 无 `th.v*` → 无 vector flavor mismatch。Bound-type gate：compute/latency-bound → 本地 RVV 配置/结构修复相关。

## Phase 2 — Scope / 分析边界
函数清单：1 个（reduceRowSum_8u32s lambda operator()，[clone .isra.0]）。该函数做 u8→u16（饱和 vsaddu）行方向累加，每 256 行 flush 到 u32，末行写 dst。
hot loop 区间（单行 u8 累加主循环，执行 len/vlanes8 次/行）：
1. 主累加循环体：0x2f4d14–0x2f4d5e（≈全部样本的 90%+：vsetvli 57.5% + 整型记账 34% + 向量数据处理 ~6%）
2. 冷路径：初始化 0x2f4b7e–0x2f4b98、flush 0x2f4e70–0x2f4f14、dst 拷贝 0x2f4bce–0x2f4be4 / 0x2f4e06–0x2f4e1e（各 ≤0.5%）

trace anchor（最高占比行）：`47.83 : 2f4d20: vsetvli s0,zero,e16,m1,ta,ma`（主循环内 e16,m1 VLMAX 派生）。
次高：`27.05 : 2f4d50: add a4,a4,s1`（循环计数器）、`7.73 : 2f4d28: vsetvli zero,s1,e8,m1,ta,ma`。

Sampling IP precision 未知 → 单指令不承担 latency 根因；以区间聚合 + 指令形态（vsetvli 计数/结构）+ 源码交叉证据为归因基础。annotate 覆盖完整（hot loop body 全覆盖）。

## Phase 3 — Pattern scan / 模式扫描：cv::cpu_baseline::reduceRowSum_8u32s::operator()

Class selection trace（8 项）：
1. rows-asm.md — exclude：当前代码来源是 compiler/intrinsic 生成（OpenCV universal intrinsics v_expand/v_add/v_load/v_store → __riscv_v* 内建），非手写 `.S`，无 missing-.S 四证
2. rows-operator-rvv.md — exclude：主循环已向量化（v*）；widening-reduction 行要求 scalar hot path，且本行不是 conversion/packing 语义主导
3. rows-string-memory.md — exclude：非 string/memory/copy/compare 函数
4. rows-vectorized-tuning.md — include（必选）：annotate 完整且已有 `v*`，来源非 `.S`，修正对象为 RVV 配置/寄存器/状态（LMUL 选型、vsetvli churn）
5. rows-codegen.md — include：循环归纳变量强度削减（地址重建/limit reload）、控制流
6. rows-offload.md — exclude：无矩阵引擎/packed-SIMD 信号
7. rows-crypto.md — exclude：非密码原语
8. rows-runtime-os.md — exclude：用户态 OpenCV benchmark

Classes scanned: rows-vectorized-tuning.md, rows-codegen.md

### Local performance pattern scan: cv::cpu_baseline::reduceRowSum_8u32s::operator()

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | u16 累加域用 e16,m1（16 lane/指令，VLEN=256 一半），v_expand 后 lo/hi 拆分处理 | High | Medium | patterns/rvv_register_group_utilization.md |
| RVV Vector-State Management（independent） | 主循环每迭代 4 次 vsetvli（57.5% 样本），含循环不变 VLMAX 派生逐迭代重建 | Medium | Medium | patterns/rvv_vector_state_management.md |
| Loop Induction Variable Strength Reduction（independent） | buf16-hi 地址每迭代从 Mat 字段重载重建（ld/lw/add/slli/add），索引计数器 add a4,a4,s1=27.05% | High | Low | patterns/loop_induction_variable_strength_reduction.md |

**Finding 1 — RVV Register-Group Utilization and LMUL Sizing（primary, L4）**

(a) 逐字 evidence 引用：
- `47.83 : 2f4d20: vsetvli s0,zero,e16,m1,ta,ma` — u16 域以 e16,m1 工作（VLMAX=16 lane/指令，仅为 VLEN=256 的一半）
- `7.73 : 2f4d28: vsetvli zero,s1,e8,m1,ta,ma`、`0.48 : 2f4d18: vsetvli zero,s1,e8,m1,ta,ma` — e8,m1（32 lane）
- `2.17 : 2f4d4c: vle16.v v1,(a5)`（buf16-hi）、`1.45 : 2f4d36: vsaddu.vv v1,v1,v2`（buf16-lo 饱和加）— lo/hi 两个 u16m1 半区分别 load/add
- 归属：主循环 0x2f4d14–0x2f4d5e；vlanes8=32（e8m1 VLMAX）、vlanes16=16（e16m1 VLMAX）、vlanes32=8；源码 reduce.simd.hpp:848-854 `v_expand(vx_load(src+i), lo, hi)` + 两个 `v_store(buf16+i…)`/`v_store(buf16+i+vlanes16…)`，v_uint16 类型映射为 u16m1，导致 32-lane 数据拆成 lo（16）+hi（16）两个 m1 处理

(b) 互斥邻居排除：
- vs rvv_vector_state_management：该行管"兼容状态重复建立"；此处 e16m1 的根因是 LMUL=1 使 u16 域每指令仅 16 lane、被迫 lo/hi 拆分（v_expand 天然产生 e16m2 组却按两个 m1 消费），vsetvli 密度是 LMUL 结构的症状之一 → 本行认领结构主因
- vs rvv_register_budgeted_loop_unrolling：LMUL 未达合法边界——m2 候选（32-lane u16）合法且无 spill，按 arbitration 由 register-group 认领、unroll 延后
- vs no-vectorization / widening-reduction：主循环已向量化，operator 行要求 scalar hot path → 均不命中

(c) 双 Confidence 推导式：
- route: 反汇编直接显示 e16,m1（16 lane）与 vlanes16=16 的固定-VL 结构；源码 v_uint16=u16m1 强制 lo/hi 拆分；live-set 核算 m2 候选峰值 live≈4 组（load 1 + widened 2 + acc 1）×2 寄存器 = 8 ≤ 32，无 spill → High
- impact: vsetvli 57.5% + 整型记账 34% 的绝大部分为该结构所致（迭代次数、u16 指令数与配置次数均可降半）；VLEN/bound 已知；`baseline_gap: sampling IP precision` 且函数级贡献未知 → Medium

**Finding 2 — RVV Vector-State Management（independent, L3）**

(a) 逐字 evidence 引用：
- `47.83 : 2f4d20: vsetvli s0,zero,e16,m1,ta,ma` — 循环不变 VLMAX 派生（avl=x0）每迭代重建，且与 `1.45 : 2f4d32: vsetvli s0,zero,e16,m1,ta,ma` 为同一 vtype/vl 重复建立
- `7.73 : 2f4d28: vsetvli zero,s1,e8,m1,ta,ma` — e8 状态在 e16 块之间重建（s1=vlanes8 循环不变）
- 主循环每迭代 4 次 vsetvli（2f4d18/20/28/32），仅服务 7 条向量指令（2×vle + vwcvtu + 2×vsaddu + 2×vse16）；循环内 vl 恒定不变
- 归属：主循环 0x2f4d14–0x2f4d5e

(b) 互斥邻居排除：
- vs rvv_register_group_utilization：即使提升 LMUL，循环内 vsetvli 仍逐迭代重建；循环不变 VLMAX 派生与同 SEW 分组可独立消除（本循环 vl/vtype 跨迭代完全不变）→ 独立
- vs rvv_inactive_lane_policy：无 tu/mu tail/mask 活性数据流证据 → 不归

(c) 双 Confidence 推导式：
- route: 直接反汇编显示相同/兼容 vtype（e16,m1,ta,ma VLMAX）与循环不变 vl 在同一 hot interval 重复建立、中间无 call/inline-asm/CSR clobber；但 emitter 失效点未证明（仅反汇编）→ Medium（pattern 要求 "Keep route confidence below High when only disassembly is available"）
- impact: vsetvli 直接局部份额 57.49%（47.83+7.73+1.45+0.48）；存在竞争 bottleneck（F1）→ Medium

**Finding 3 — Loop Induction Variable Strength Reduction（independent, L4）**

(a) 逐字 evidence 引用：
- `0.72 : 2f4d40: ld a5,24(s2)` + `0.48 : 2f4d44: lw a5,0(a5)` + `1.45 : 2f4d46: add a5,a5,a4` + `2f4d48: slli a5,a5,0x1` + `0.97 : 2f4d4a: add a5,a5,a7` — buf16+i+vlanes16 地址每迭代从 Mat 对象字段重载并重建（index scaling + limit reload）
- `27.05 : 2f4d50: add a4,a4,s1` + `0.97 : 2f4d5e: bge s8,t0` — 索引归纳变量 i 的更新与上界比较（循环携带整型链）
- 归属：主循环 0x2f4d14–0x2f4d5e；对应源码 reduce.simd.hpp:853 `v_store(buf16 + i + vlanes16, …)` 每次以表达式重建地址

(b) 互斥邻居排除：
- vs algebraic_simplification：本行是循环级归纳变量变换（每迭代重建 base+i*stride 地址、Mat 字段 limit reload），非 peephole 级恒等式 → 归本行
- vs register_pressure_and_save_restore：无 spill/reload 栈证据 → 不归

(c) 双 Confidence 推导式：
- route: 反汇编直接显示 index scaling（slli+add）与 limit reload（ld/lw Mat 字段）每迭代执行，访问为连续固定 stride → High
- impact: 地址重建段直接份额约 3.6%（2f4d40–2f4d4a），循环控制链缩短受益；竞争 bottleneck（F1/F2 主导）→ Low

**多候选仲裁**：
- 三个 finding 位于同一主循环（0x2f4d14–0x2f4d5e），机制可分账：F1=LMUL 选型导致 u16 域 lo/hi 拆分（结构根因），F2=vsetvli 状态逐迭代重复建立（含循环不变 VLMAX 派生），F3=归纳变量地址重建与 limit reload。地址归属：F1 锚定 vsetvli e16m1 + lo/hi 两个 u16 数据路径（2f4d24/2f4d36/2f4d4c/2f4d56）；F2 锚定 2f4d18/20/28/32 的 vsetvli 簇；F3 锚定 2f4d40–2f4d4a 与 2f4d50。
- 因果消除：提升 LMUL（F1）会显著减少 F2 的 vsetvli 密度与 F3 的 hi 地址计算，但 F2 的循环不变 VLMAX 派生（即使 m2 也逐迭代重建）与 F3 的 src 指针增量/端指针比较仍可独立优化 → F1 为 primary，F2/F3 为 independent。
- 因果层：F1、F3 属 L4（compute/codegen micro-structure），F2 属 L3（vector/runtime configuration）。样本份额排序 F1≈F2>F3；三者份额在时间上重叠，**不得简单相加**。
- 排除候选：unroll 行——LMUL 未达合法边界、register-group 优先认领；inactive-lane/operand-form/operator 行——无相应证据；maximal-LMUL byte 行——非 byte copy 路径。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::cpu_baseline::reduceRowSum_8u32s::operator()

### Finding 1（primary）— rvv_register_group_utilization.md（matched row: rows-vectorized-tuning.md "RVV Register-Group Utilization and LMUL Sizing"）

1. **Root cause**：主循环以固定 vlanes8=32（e8,m1）逐行累加，但 u16 累加域类型 v_uint16 映射为 e16,m1（仅 16 lane/指令，VLEN=256 的一半）。`v_expand` 对 32-lane u8 加宽后天然得到 e16m2 组，代码却按 lo（v2）+hi（v3）两个 m1 半区消费：两次 vle16、两次 vsaddu、两次 vse16，并伴随两套地址计算与更多 vsetvli。依据 patterns/rvv_register_group_utilization.md §Why this is slow："LMUL 决定每条向量指令处理的元素数和占用的 register group 数量。选得过小会浪费寄存器带宽和循环开销"。loop/config overhead 主导：主循环 57.5% 样本在 vsetvli、34% 在整型记账，实际向量数据处理仅 ~6%。
2. **The fix**：
   - 修复对象：reduceRowSum_8u32s 主累加循环的 LMUL 选型（reduce.simd.hpp:848-854）。在 RVV 路径将 u16 累加域提升到 e16,m2（32 lane，单条 vle16/vsaddu/vse16 覆盖 lo+hi），或整行用 v_load_expand + m2 累加链；e8 侧可同步升到 e8,m2（64 元素/迭代）进一步摊薄 per-iteration 开销。
   - before/after 伪代码：
     ```cpp
     // Before（reduce.simd.hpp:848-854）：vlanes8=32（e8m1），vlanes16=16（e16m1）
     for (; i <= len - vlanes8; i += vlanes8) {
         v_uint16 lo, hi;
         v_expand(vx_load(src + i), lo, hi);              // 32 u8 → e16m2 组按 lo/hi 两个 m1 消费
         v_store(buf16 + i, v_add(vx_load(buf16 + i), lo));
         v_store(buf16 + i + vlanes16, v_add(vx_load(buf16 + i + vlanes16), hi));
     }
     // After：u16 域 e16,m2（32 lane 单条处理）；枚举 m1/m2/m4 × unroll=1/2/4 frontier 实测选定
     for (; i <= len - vlanes8; i += vlanes8) {
         // vle8 e8m2(64) → vwcvtu e16m4；或保持 32-lane：vle8 e8m1 → vwcvtu e16m2 → 单条 vle16m2/vsaddu.m2/vse16.m2
     }
     ```
   - 适用前提：LMUL×peak_live_vectors≤32。m2 候选核算：load u16 (m2) + widened u16 (m2) + acc (m2) ≈ 3 组 = 6 寄存器 ≤ 32，无 spill；e8m2→e16m4 变体峰值 live≈(2+16+8)=26 寄存器，仍合法但需核对 group 对齐与 allocator（pattern §The fix "Count peak live vectors and apply the budget"）。
   - correctness contract：不得改变 vsaddu 饱和累加语义（u16 饱和、65280 上限合同）、不得改变 tail（len % vlanes8 余数仍由标量循环处理）、不改变 u32 flush 合同。
   - 限制/风险：v_uint16=u16m1 是 OpenCV universal-intrinsic 类型映射，改 LMUL 需在 RVV 专用路径（如 #if CV_RVV 分支或 hal 层）实现，不能破坏 x86/NEON 通用路径；更大 LMUL 需实测防 frontend/spill 回归（pattern §Why this is slow "未必优于自动向量化"）。
   - 预期 Profile 信号：2f4d20/2f4d32 的 e16,m1 vsetvli 与 2f4d24/2f4d4c/2f4d36/2f4d56 的 lo/hi u16 指令簇份额显著下降；每迭代 vsetvli 计数下降；实际向量数据处理指令占比上升。
3. **Baseline facts 回填**：hardware ISA=rv64...v（RVV 1.0）；build ISA=Tag_RISCV_arch 含 v1p0/zve32f/zve64d/zvl128b；VLEN=256b（vlanes8=32/vlanes16=16/vlanes32=8）；bound type=compute/latency-bound。
4. **收益上界**：当前 event（cpu-clock）函数内局部样本份额：主循环开销区间 ≈ 0.90（vsetvli 57.5% + 整型记账 34%），与 F2/F3 份额重叠、**不可相加**。`baseline_gap: sampling metadata`（函数级 workload 贡献未知）→ 不得称 workload 级 Amdahl 上界。
5. **三维路由判定**：current source = compiler/intrinsic 生成（OpenCV universal intrinsics，非 `.S`）；implementation existence/reachability = 无 dispatch/多版本证据；function-level policy = cpu_baseline SIMD 路径以 universal intrinsic 为准，无独立 `.S` policy 证据。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs**：Related PRs：16 条 URL（OpenCV #26318、OpenCV 5be158a2b6ed、OpenCV #25586、Linux a4348546332c、Linux a894e8ed09c6、Linux c2a658d41924、OpenJDK bdd37b0e5eaa、OpenBLAS cc1b5794a040、OpenBLAS d69be17b6ff7、OpenBLAS 4a12cf53ec11、OpenBLAS 240695862984、V8 384433993606、vLLM #47538）。

### Finding 2（independent）— rvv_vector_state_management.md（matched row: rows-vectorized-tuning.md "RVV Vector-State Management"）

1. **Root cause**：主循环每迭代 4 次 vsetvli（2f4d18/20/28/32），其中 `vsetvli s0,zero,e16,m1,ta,ma` 是 avl=x0 的 VLMAX 派生——VLEN 在循环内恒定，vlanes8/vlanes16 均循环不变，该状态本可一次建立；2f4d20 与 2f4d32 为同一 vtype/vl 重复建立，中间 e8 切换仅为 vwcvtu 服务。依据 patterns/rvv_vector_state_management.md §The fix："仅在 VL 与 VTYPE 同时匹配时复用"；§Why this is slow："重复设置相同状态或在每个局部分支里重复标记会增加循环控制指令"。该 vsetvli 簇独占 57.5% 样本。
2. **The fix**：
   - 修复对象：主循环 vsetvli 发射/调度（reduce.simd.hpp:848-854 的 intrinsic 组合或对应 codegen）。方向：把循环不变 vsetvli 提到循环外（固定-VL 循环 vl/vtype 跨迭代不变），并令同 SEW 指令分组执行——e8 块（vle8+vwcvtu）后一次性进入 e16 块（vle16+vsaddu+vse16），消除 e16→e8→e16 乒乓。
   - before/after 伪代码：
     ```cpp
     // Before（当前每迭代）: vsetvli e8; vle8; vsetvli e16; vle16(lo); vsetvli e8; vwcvtu; vsetvli e16; vsaddu; vse16; ... （4 次 vsetvli/迭代）
     // After: 循环外一次 vsetvli e8（或循环内仅 2 次：e8 块完成后单次切 e16 执行全部 u16 指令）
     ```
   - correctness contract：不得改变 vtype/vl 语义、不得在 call/clobber 边界复用状态；固定-VL 前提（vlanes 循环不变）成立。
   - 限制/风险：属编译器调度/intrinsic 结构问题，OpenCV universal-intrinsic 每操作内嵌 vsetvli；需在 RVV 专用路径手写/重组 intrinsic 或在编译器侧验证 vsetvli 合并。
   - 预期 Profile 信号：主循环 vsetvli 计数从 4/迭代降至 ≤2/迭代（F1 后进一步下降）；2f4d20 的 47.83% 份额消失。
3. **Baseline facts 回填**：同 finding 1（VLEN=256b、compute-bound）。
4. **收益上界**：vsetvli 指令直接局部份额 0.5749（47.83+7.73+1.45+0.48）；与 F1 份额重叠、不可相加。`baseline_gap: sampling metadata`。
5. **三维路由判定**：current source = compiler/intrinsic 生成；存在性/可达性 = N/A；function-level policy = N/A。
6. **Implementation-shape proof**：不适用。
7. **Related PRs**：Related PRs：13 条 URL（V8 c81ffb7a356d；QEMU d57dfe4b37ae、944b6dfd3d67、25669d275ce7、bd2c82283d21、81b9ef995a3b、949b6bcb2729、b8e1f32cda78；LLVM #148246、d0554ae4cf26、#118285、#123878、f59307bfdc01）。

### Finding 3（independent）— loop_induction_variable_strength_reduction.md（matched row: rows-codegen.md "Loop Induction Variable Strength Reduction"）

1. **Root cause**：buf16+i+vlanes16（hi 半区）地址每迭代从 Mat 对象字段重载（`ld a5,24(s2); lw a5,0(a5)`）并执行 add/slli/add 重建，而非指针递增；循环计数器 `add a4,a4,s1`（27.05%）与 `bge s8,t0` 构成循环携带整型链。依据 patterns/loop_induction_variable_strength_reduction.md §Why this is slow："索引归纳变量每轮重建地址"；"循环体很小，循环控制指令在其中占比很高，每轮几条冗余整数指令被迭代次数放大"。访问为连续固定 stride（src+buf16），指针递增 + 预计算 end-pointer 可消除重建。
2. **The fix**：
   - 修复对象：主循环地址生成（reduce.simd.hpp:848-854 的 `buf16 + i + vlanes16` 表达式）。方向：归纳变量从索引 i 改为指针递增（p_src += vlanes8、p_buf16 += vlanes16*2），终止条件用预计算 end-pointer。
   - before/after 伪代码：
     ```cpp
     // Before: for (i; i <= len - vlanes8; i += vlanes8) { use(src + i, buf16 + i, buf16 + i + vlanes16); }
     // After:  const uchar* ps = src; ushort* pb = buf16; const uchar* ps_end = src + (len - vlanes8);
     //         for (; ps <= ps_end; ps += vlanes8, pb += vlanes16) { use(ps, pb, pb + vlanes16); }
     ```
   - correctness contract：end-pointer 不越界（len 有界、vlanes8 整除循环由余数 tail 承接）；n==0/短行必须零次进入主循环；消除 Mat 字段每迭代重载不改变任何别名/内存语义。
   - 限制/风险：编译器本可强度削减，但 OpenCV universal-intrinsic 的地址表达式阻止了它；RVV 专用路径重组或编译器 -fivopts 验证。收益集中于循环控制链（~34% 整型记账的地址重建部分）。
   - 预期 Profile 信号：2f4d40–2f4d4a 的 ld/lw/add/slli/add 地址重建簇消失；`add a4,a4,s1` 被指针 addi 替代；loop backedge `bge s8,t0` 保持或改端指针比较。
3. **Baseline facts 回填**：同 finding 1。
4. **收益上界**：整型记账区间局部份额 ≈ 0.34（含计数器与地址重建），其中地址重建直接段约 0.036；与 F1/F2 份额重叠、不可相加。`baseline_gap: sampling metadata`。
5. **三维路由判定**：current source = compiler/intrinsic 生成；存在性/可达性 = N/A；function-level policy = N/A。
6. **Implementation-shape proof**：不适用。
7. **Related PRs**：Related PRs：3 条 URL（Linux 18be4ca5cb4e、OpenBLAS 477dd40f073c、OpenBLAS d832ee50868a）。

## Phase 5 — Verification forecast / 验证预测：cv::cpu_baseline::reduceRowSum_8u32s::operator()

- **F1（LMUL sizing）**：
  - 应消失/缩小：`47.83 : 2f4d20: vsetvli s0,zero,e16,m1,ta,ma`、`2.17 : 2f4d4c: vle16.v v1,(a5)`（hi 半区）、`1.45 : 2f4d36: vsaddu.vv v1,v1,v2` 的 lo/hi 拆分簇。
  - 应出现：e16,m2 的 vsetvli 与单条 vle16/vsaddu/vse16（对齐 pattern §Verification "objdump 确认主循环 vsetvli 使用了候选 LMUL，且没有因跨宽度不匹配插入多余转换"）。
  - 正确性：枚举 m1/m2/m4×unroll=1/2/4 frontier 记录合法性/spill；相同输入逐元素一致（vsaddu 饱和语义不变）；覆盖 len=0、短行、行宽非 32 倍数 tail。
  - 基准：opencv_perf_core reduce 用例，X100 cycles/op 前后对比，长中短行宽。
- **F2（vector-state）**：
  - 应消失：主循环内 2f4d20/2f4d32 的重复 e16,m1 VLMAX 派生与 e16→e8→e16 乒乓（每迭代 vsetvli 从 4 降至 ≤2）。
  - 应出现：循环外/循环顶单次 vsetvli（pattern §Verification "count vsetvl* in the exact hot interval before and after"）。
  - 正确性：vtype/vl 语义与 tail/mask 行为不变，结果逐位一致。
- **F3（induction strength reduction）**：
  - 应消失：`0.72 : 2f4d40: ld a5,24(s2)`、`0.48 : 2f4d44: lw a5,0(a5)`、`1.45 : 2f4d46: add a5,a5,a4`、`2f4d48: slli`、`0.97 : 2f4d4a: add a5,a5,a7` 地址重建簇。
  - 应出现：循环内指针递增 + 预计算 end-pointer 比较（pattern §Verification "确认循环体中的 slli/add 与 limit reload 减少，替换为指针递增与指针比较"）。
  - 正确性：边界回归（len=0、短行、行宽整倍数/余数）、逐元素等价。
- 组合验证：F1 先行（结构主因），F2/F3 各自独立复核；不要用单次组合修复替代单假设归因（arbitration §5）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅ | 1/1 组；cv::cpu_baseline::reduceRowSum_8u32s::operator() |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表；gap 标签：baseline_gap: sampling metadata、baseline_gap: sampling IP precision；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；Classes scanned: rows-vectorized-tuning.md, rows-codegen.md；顶层 finding 3 个；evidence 锚点 2f4d20/2f4d28/2f4d18/2f4d32/2f4d4c/2f4d36/2f4d40/2f4d44/2f4d46/2f4d4a/2f4d50/2f4d5e；supporting 0；排除条数 5（unroll/inactive-lane/operand-form/operator/maximal-LMUL-byte）；推导式 3 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：rvv_register_group_utilization.md、rvv_vector_state_management.md、loop_induction_variable_strength_reduction.md；对应命中 row 3；各自引用短语首词："LMUL 决定每条向量指令处理的元素数"（F1）、"仅在 VL 与 VTYPE 同时匹配时复用"（F2）、"索引归纳变量每轮重建地址"（F3）；The fix 含 before/after/correctness/风险/预期 Profile 信号；Related PRs：F1=16、F2=13、F3=3 条 URL |
| 5 | 路径合规 | ✅ | 模式 A（profile_backed）；8 项 trace 可解释扫描集；3 findings 按份额排序 F1≈F2>F3（份额重叠、注明不可相加）；L3/L4 归属明确；th.v* 未停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 2f4d20/2f4d4c/2f4d36/2f4d32/2f4d40/2f4d44/2f4d46/2f4d4a/2f4d50；出现侧指向三个 pattern §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无