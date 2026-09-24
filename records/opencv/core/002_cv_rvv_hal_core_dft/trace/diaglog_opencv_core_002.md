Functions under analysis: [cv::rvv_hal::core::dft(unsigned char const*, unsigned char*, int, int, int*, double, int*, void*, int, int, bool, bool)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`002-cv：：rvv_hal：：core：：dft(...)-annotate.txt`；libopencv_core.so.5.1.0 反汇编；event=cpu-clock；6802 samples；percent type=local period；4935 行；函数覆盖完整，含全部热点循环）
- perf stat（可选 bound/context）：已提供（`7-opencv-perf-benchmark-riscv-core.txt`）
- workload/binary/DSO/source context：已提供（libopencv_core.so.5.1.0；带 DWARF/source 行——annotate 内联了 `cv::rvv_hal::core::dft<float>/<double>` 与 `cv::rvv_hal::core::rvv<T>::vlseg/vlsseg/vsseg` 模板源码行；OpenCV `modules/core/src/dxt.cpp` RVV 路径）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（opencv binary：`rv64i2p1_..._v1p0_..._zvl128b1p0...`，含 RVV 1.0；本函数所在 DSO 为 libopencv_core.so.5.1.0，同一构建）
- hardware ISA（/proc/cpuinfo 或 hwprobe）：已提供（metadata snapshot：`rv64imafdcvh_...`，含 `v`、`zbb` 等）
- vlenb：已提供（VLEN=256 bits，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：已提供（cpu-clock；local period；单次运行窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（cpu-clock 软件事件，非 Exact-IP；IRQ 边界 skid 大 → 单行只锚定区间）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：SpacemiT X100，`rv64imafdcvh_...`（RVV 1.0 `v`，`zbb/zbc/zbs`，`zvk*`，`zaamo/zalrsc`），OoO 核 |
| Build ISA | 已提供：OpenCV binary `Tag_RISCV_arch` = `rv64i2p1_..._v1p0_..._zvl128b1p0...`（RVV 1.0，无 `th.v*`）；libopencv_core.so 同构建。无 IFUNC/multiversion 证据 |
| Vector flavor | annotate 显示标准 RVV 1.0 `v*` mnemonic（`vsetvli`/`vlsseg2e32`/`vsseg2e64`/`vloxei32` 等），无 `th.v*` → flavor 匹配 |
| VLEN | 已提供：VLEN=256 bits（vlenb=32）→ e32,m1=8 lane/op，e32,mf2=4 lane/op，e64,m1=4 lane/op，e32,m8=64 lane/op |
| Bound type | 全 workload：IPC=0.9176，L1_dcache_load_miss_rate=1.36%，branch_miss_rate=1.67% → compute/latency 型，非 memory-bound；DFT 热点为 FP 向量计算 + 重排数据搬运，本地向量调优是正确杠杆 |
| Sampling semantics | event=cpu-clock；percent type=**local period**；同一运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为函数内局部份额 → `baseline_gap: sampling metadata` |
| Sampling IP precision | `precise_ip`/Exact-IP 未知；cpu-clock 软事件 IRQ 边界 skid 大 → `baseline_gap: sampling IP precision`；单行只锚定 basic block / loop interval，不作单指令 latency 归因 |

L0 baseline gate：
- hardware 有 `v`，build 有 `v`（RVV 1.0）→ 无 mismatch。
- 无 `th.v*`，flavor gate 不触发。
- Bound-type gate：非 memory-bound；本地 RVV 调优方向保留。

## Phase 2 — Scope / 分析边界

- 函数清单与承诺声明一致：[`cv::rvv_hal::core::dft(...)`]（1 个）。该符号是 dispatcher，内含 `dft<float>`/`dft<double>` 实例的 radix-3/5 蝶形内核、itab 重排 gather-scatter 循环与 permute 收尾。
- 热点区间（按地址归属分账）：
  1. f32 itab 重排 gather-scatter 循环 `[0x328238–0x328276]` 与 `[0x32a3ea–0x32a424]`、f64 重排循环 `[0x32a452–0x32a498]`（`vlse32`+`vmul.vx`+`vloxei32`+`vsse32/vsse64`）
  2. f32 radix-5 内核 `[0x3283ce–0x328562]`（`e32,mf2`；含标量 DC 分量 `flw/fadd.s/fsw`）
  3. f64 radix-5 内核 `[0x3287b8–0x328904]`（`e64,m1`；含 `csrr vlenb` 与栈 spill）
  4. f32 radix-2 内核 `[0x3289aa–0x328a06]`（`e32,mf2`）
  5. f32 radix-3 内核 `[0x3296c8–0x3297a0]`（`e32,mf2`）
- 各区间 trace anchor（最高占比行）：
  - 区间 1：`4.25 : 32a414: vadd.vi v0,v0,4`；`2.56 : 32a480: vsetvli zero,zero,e32,m4,ta,ma`
  - 区间 2：`4.63 : 32855e: blt s3,a6,3283ce`；`1.57 : 328562: ld s6,8(sp)`；`0.56 : 3283e6: vsetvli t6,zero,e32,mf2,ta,ma`
  - 区间 3：`2.31 : 328900: vsseg2e64.v v4,(a3)`；`1.65 : 328864: vfmul.vv v13,v5,v22`；`0.94 : 32851c: csrr a4,vlenb`
  - 区间 4：`3.41 : 3289c8: vlseg2e32.v v8,(s6)`
  - 区间 5：`2.06 : 3296e4: add s3,a6,t3`
- annotate 覆盖完整；Sampling IP precision 未确认 → 高占比行锚定 interval 级机制，不作单指令 cycle 归因。入口条件 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：cv::rvv_hal::core::dft

### Class selection trace（8 项）

1. `rows-asm.md` — exclude：当前代码来源为 intrinsic/template 生成代码（annotate 内联 `__riscv_vlseg2e64_v_f64m1x2` 等 intrinsic 源码行），非手写 `.S`。
2. `rows-operator-rvv.md` — include：DFT/FFT 频域变换语义 + itab 重排的 indexed gather/scatter 数据访问信号。
3. `rows-string-memory.md` — exclude：无 string/memory 语义循环。
4. `rows-vectorized-tuning.md` — include（intrinsic 生成 RVV 代码必选）：LMUL/register-group、vector-state、operand-form、inactive-lane、unroll。
5. `rows-codegen.md` — include：观察到的栈 spill（`ld s6,8(sp)` 1.57%、`vs1r.v v10,(a2)` 0.75%）与重复 `addi a2,a2,160; add a2,a2,sp` 地址生成 → 检查 register-pressure 与 addressing 类 row。
6. `rows-offload.md` — exclude：无矩阵引擎 / packed-SIMD / 权重重排信号。
7. `rows-crypto.md` — exclude：非密码学原语。
8. `rows-runtime-os.md` — exclude：用户态库函数，非 kernel/timer/特权边界。

### Classes scanned: `rows-vectorized-tuning.md`, `rows-operator-rvv.md`, `rows-codegen.md`

### 命中总表

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | f32 radix 内核全程 `e32,mf2`（VLEN=256 下每 op 仅 4 lane，天然下限 m1=8 lane）；回边分支/配置开销高（`blt` 4.63%/1.81%/1.48%/1.13%）；f64 m1 内核活寄存器集大导致栈 spill | High | Medium | `patterns/rvv_register_group_utilization.md` |
| RVV Vector-State Management（independent） | e64 gather 循环每轮重复 4 次 vtype 切换（`vsetvli zero,zero,e32,m4` 2.56%）；内层循环重复 `csrr vlenb` 推导栈偏移（0.94%+0.15%+0.10%），vl 全程不变 | Medium | Medium | `patterns/rvv_vector_state_management.md` |

#### Finding 1（primary）：RVV Register-Group Utilization and LMUL Sizing

(a) 逐字 evidence 引用：
- `0.56 : 3283e6: vsetvli t6,zero,e32,mf2,ta,ma`（f32 radix-5 内核建立 mf2 vtype；VLEN=256 下 vl=4 lane/op，而 m1 天然为 8 lane/op）
- `4.63 : 32855e: blt s3,a6,3283ce`（f32 radix-5 mf2 内层循环回边分支）
- `0.54 : 328504: vsetvli zero,a5,e32,mf2,ta,ma`（每轮重新建立同型 mf2 vtype）
- `3.41 : 3289c8: vlseg2e32.v v8,(s6)`（f32 radix-2 mf2 循环数据载入）
- `2.31 : 328900: vsseg2e64.v v4,(a3)` 与 `1.65 : 328864: vfmul.vv v13,v5,v22`（f64 m1 radix-5 内核主体）
- `1.57 : 328562: ld s6,8(sp)`（mf2 循环出口的 GPR 栈重载——活寄存器集压力）
- `2.06 : 3296e4: add s3,a6,t3`（f32 radix-3 mf2 循环地址计算）
- Supporting evidence：`0.75 : 328814: vs1r.v v10,(a2)`（f64 m1 内核把向量 v10 spill 到栈）——supporting because：同为 live-set 超出寄存器预算的同一机制（pattern §The fix item 4 "Control live range to prevent spill"）。
- 所属区间：f32 radix-5 `[0x3283ce–0x328562]`、radix-2 `[0x3289aa–0x328a06]`、radix-3 `[0x3296c8–0x3297a0]`、f64 radix-5 `[0x3287b8–0x328904]`。

(b) 互斥邻居排除：
- 非 no-vectorization：各区间完整 annotate 有大量 `v*`（`vlsseg2e32/vlseg2e32/vsseg2e64/vsetvli`），不是 scalar 主循环。
- 非 FFT-scalar row：蝶形已按级向量化（有 `vlseg2/vsseg2`），row 的 signal 明确要求 hot loop 内 zero `vlseg2e32/vsseg2e32/vsetvli`——与 evidence 相反。
- 非 inactive-lane：所有 vtype 为 `ta,ma`，无 `tu/mu`、无 `vmclr.m`。
- 非 operand-form：`vmv.v.i v8,0` 的 zero broadcast 被多条 store 复用（pattern 行内排除"同一 broadcast 被多条 op 复用时仅低置信"），不主导。
- 非 register-budgeted-unrolling：单条 back-edge 高占比但无区间聚合/unroll 合法性证据，且按行内判据"若缩短 live range 后更大 LMUL 合法、无 spill，则由 register-group row 认领"。

(c) 双 Confidence 推导式：route：intrinsic 生成 RVV + 反汇编直接证据（`e32,mf2` vtype、回边/配置开销、spill）+ VLEN=256 → High；impact：缺 instruction-level live-range 合法性证明（m1/m2 候选需 source 侧证明无 spill，属 `implementation_shape_gap`）、采样语义 local period → Medium。

#### Finding 2（independent）：RVV Vector-State Management

(a) 逐字 evidence 引用：
- `2.56 : 32a480: vsetvli zero,zero,e32,m4,ta,ma`（f64 itab 重排 gather 循环每轮第 3 次 vtype 切换；该循环每轮共 4 次 `vsetvli`：`32a45a e32,m4`→`32a478 e64,m8`→`32a480 e32,m4`→`32a488 e64,m8`，vl 全程不变）
- `0.94 : 32851c: csrr a4,vlenb`（f32 radix-5 mf2 循环内重复读 vlenb 推导 wave 临时缓冲栈偏移）
- `0.15 : 328818: csrr a2,vlenb`、`0.10 : 32883c: csrr a2,vlenb`（f64 radix-5 m1 循环内重复 vlenb 推导；同一 vlenb 值循环不变）
- 所属区间：f64 重排循环 `[0x32a452–0x32a498]`、f32/f64 radix-5 循环内部（`csrr vlenb` 各点）。

(b) 互斥邻居排除：
- 非 register-group：修正对象是循环内不变量的重复状态建立（vlenb 推导、vl 不变的 vtype 往返），不是 LMUL 选型本身；LMUL 修正后这些 bookkeeping 仍存在。
- 非 inactive-lane：无 `tu/mu`、mask liveness 问题。
- 非 operator/data-movement 症状：vtype 切换由 `vloxei32` 的 32-bit index + 64-bit data SEW 不兼容造成，但循环内以相同两态往复；`csrr vlenb` 是纯循环不变量推导，不属于任何数据访问算法本体。

(c) 双 Confidence 推导式：route：反汇编直接证据（重复 `vsetvli`/`csrr vlenb` + 状态保持分析：vl 不变、无 call/inline-asm clobber）+ 具名地址有实际 sample → Medium（`csrr vlenb` 提升为循环不变量的方向明确；vtype 往返消除需算法结构调整，存在选择余量）；impact：single-line 2.56% + csrr 合计约 1.2% 局部份额、local period 采样、IP precision 不足 → Medium。

#### 多命中仲裁

- Finding 1 与 Finding 2 的关系：**independent**。因果消除测试：把 f32 radix 内核 LMUL 从 mf2 提到 m1/m2（缩短 live range 后）不消除 `csrr vlenb` 循环内推导（wave 缓冲栈寻址仍需要），也不消除 gather 循环的 vtype 往返（Sew 不兼容由算法索引形态决定）；反之修 vector-state 也不改变每 op 4 lane 的 mf2 粒度。两者机制、修复对象、验证预测均可分离，且证据行地址不相交（Finding 1 锚定 mf2 vtype/回边/spill 行；Finding 2 锚定 `vsetvli zero,zero`/`csrr vlenb` 行）→ independent，按 arbitration L 层：Finding 1 属 register-group/LMUL 层（L1/L4），Finding 2 属 vector-state/config 层（L3）。
- itab 重排 gather-scatter 数据搬运本身（`vlse32`+`vloxei32`+`vsse32/vsse64`，合计约 22% 局部份额）无 row 匹配：FFT row（要求 scalar 蝶形 + zero v*）与 gather row（要求 scalar 逐 lane 回退）的行内 signal 均不成立——已向量化 gather 的成本主要落在 index 向量算术（`vadd.vi` 4.25%+1.53%）与 scatter store（`vsse32/vsse64` 合计 6.76%）；其中 vtype 切换份额归 Finding 2，其余归该数据搬运层的固有成本（负证据，见下）。
- 动态优先级（入口 A）：按 cited evidence 行加总——Finding 1 局部样本份额 ≈ 17.24%（0.56+4.63+0.54+3.41+2.31+1.65+1.57+2.06+0.51）；Finding 2 ≈ 3.75%（2.56+0.94+0.15+0.10）。不同机制不简单相加；同层仅此两项，按份额排序：Finding 1 第一。

#### 负证据（No local pattern matched 的区间）

- itab 重排 gather-scatter 循环的固有数据搬运无本地 row 命中：FFT row 因"已向量化"排除；Indexed Gather row 因"已向量化（vloxei32）非 scalar 回退"排除；Strided Memory row 针对标量固定 stride。该区间成本记为算法固有重排层（Stockham autosort 可消除位反转级——作为方向性观察，不构成 matched row，因为 row gate 未通过）。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::rvv_hal::core::dft

### Finding 1（primary，rows-vectorized-tuning.md → `patterns/rvv_register_group_utilization.md`）

1. **Root cause**：OpenCV 的 `cv::rvv_hal::core::rvv<T>` 模板把 float32 映射到 `e32,mf2`（`__rvv_float32mf2x2_t`），在 VLEN=256 上每个向量 op 只处理 4 个 f32 lane（天然 m1 下限为 8 lane/op），radix-5/3/2 蝶形与重排循环的每轮固定开销（回边分支、vtype 建立、地址计算）被摊到 4 lane 上，迭代数翻倍。f64 路径 `m1`（=4 lane/op）已是 VLEN=256 的天然下限，但内核同时保持 30+ 活向量（wave 表多组 vlsseg + 数据 vlseg + 中间结果），触发栈 spill（`vs1r.v v10`、`ld s6,8(sp)`）。这正对应 pattern §Why this is slow 的 "Underutilized register-group frontier"（LMUL 小于合法无 spill 候选 → 有效元素数偏低、迭代/`vsetvl`/分支开销偏高）与 §4 "Allocation constraints beyond the register count"（`LMUL * peak_live_vectors <= 32` 只是必要上限，FMA 的 source/accumulator、mask、widening EMUL 都要计入）。
2. **The fix / 修复方式**：按 pattern §The fix 执行——(1) 用 `LMUL * peak_live_vectors <= 32`（§1）逐 instruction 统计 f32 radix-5/3/2 内核的 peak live vectors（当前含 5 组 wave vlsseg pair + 5 组数据 vlseg pair + 中间蝶形结果）；(2) 枚举 `m1/m2/m4 × unroll=1/2/4/8` 候选 frontier（§2），f32 从 `mf2` 至少提至 `m1`；(3) 提高 LMUL 前先按 §4 "Control live range to prevent spill" 缩短 live range——把 wave/twiddle 系数 load 移近 consumer、或把交织 AoS wave 改为 split/SoA 连续载入以削减同时存活的 wave 向量数；(4) 验证 f64 内核提升 `m1→m2` 前先压缩活寄存器集，消除 `vs1r.v v10` 与 `ld s6,8(sp)` 型 spill；(5) 写回阶段保持合法 register group 完整（§8），不为对齐把 m2 accumulator 拆回 m1 再逐组 FMA/store。
   - before 形态（f32 radix-5）：`vsetvli t6,zero,e32,mf2,ta,ma` → 每轮 4 lane，回边 4.63%；
   - after 候选形态：`vsetvli t6,zero,e32,m1/m2,...`（在无 spill 的 live-range 压缩前提下）→ 每轮 8/16 lane，迭代数与控制开销减半/减四分之三。
   - correctness contract：LMUL 改变不改变数值结果（蝶形为逐 lane 独立复乘加，无跨 lane 归约；保持同一 FP 运算顺序）；须保持 `dft` 的 `nf/factors/scale/isInverse/noPermute` 语义与输出布局不变。
   - 限制/风险：更大 LMUL 不必然更快——若 live-range 压缩不充分会引入 spill（pattern §5 "Not always faster"）；必须用生成指令 + 实机 benchmark 验证；SpacemiT X100 为 OoO，寄存器组带宽收益需实测。
   - 预期 Profile signals：mf2 区间 `blt`（4.63%/1.13%）、`vsetvli e32,mf2`（0.56%/0.54%）、`328562: ld s6,8(sp)`（1.57%）与 `vs1r.v v10`（0.75%）占比显著下降；迭代次数下降使回边分支总占比收敛。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（RVV 1.0，VLEN=256）；build ISA=OpenCV `..._v1p0_..._zvl128b1p0...`（RVV 1.0）；VLEN=256（vlenb=32）；bound type=compute/latency（IPC 0.918，L1 miss 1.36%）；`baseline_gap: sampling metadata`（local period）；`baseline_gap: sampling IP precision`。
4. **收益上界**：本函数内局部样本份额 ≈ 17.24%（cited evidence 行加总；`dynamic priority` 有效——入口 A）。`baseline_gap: sampling metadata` 禁止 workload 级 Amdahl 上界；`implementation_shape_gap: f32 radix 内核 live-range/m1/m2 合法性与无 spill 证明`。
5. **三维路由判定**：current source = intrinsic/template 生成代码（`cv::rvv_hal::core::rvv<T>` → `__riscv_vlseg2e64_v_f64m1x2` 等，源码行 inline 可见，非 `.S`）；implementation existence/reachability = 本函数即热点实现（RVV 路径已构建并被 dispatch 命中，annotate 证实）；function-level policy = OpenCV RVV 通用模板的 LMUL 映射策略（float→mf2）是该函数合同的直接载体。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`；记录 `implementation_shape_gap` 于收益上界项）。
7. **Related PRs**（patterns/rvv_register_group_utilization.md §Related PRs，保留原始 URL）：
   - Related PRs：7 条 URL —— https://github.com/opencv/opencv/pull/26318 、https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5 、https://github.com/opencv/opencv/pull/25586 、https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c 、https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e 、https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870 、https://github.com/vllm-project/vllm/pull/47538

### Finding 2（independent，rows-vectorized-tuning.md → `patterns/rvv_vector_state_management.md`）

1. **Root cause**：内层循环对 loop-invariant 的 vector-state 重复建立——`vl` 全程不变，但 f64 重排循环每轮以 `vsetvli zero,zero` 往返切换 `e32,m4 ↔ e64,m8` 4 次（`32a45a→32a478→32a480→32a488`），其中 `32a480` 单行占 2.56%；f32/f64 radix-5 循环内每轮重复 `csrr vlenb` 并做 `slli/addi/add sp` 推导同一组 wave 临时缓冲栈偏移（0.94%+0.15%+0.10%），vlenb 是循环不变量。这正对应 pattern §Why this is slow："重复设置相同状态会增加循环控制指令"；`csrr vlenb` 属于 §The fix item 3 的 "VLMAX/vlenb 派生值应集中到一个语义 helper 中"。
2. **The fix / 修复方式**：
   - (a) 把 vlenb 推导**提升出循环**：循环外读一次 `vlenb`，循环内用 GPR 增量维护 wave 缓冲栈指针（或把 wave 表缓冲地址作为 loop-invariant base + 每轮固定步长），消除 `32851c/328818/32883c` 每轮 `csrr vlenb` + `slli/addi/add sp` 序列（pattern §The fix item 3 "Hoist duplicated vector-state bookkeeping only to safe common paths"）。
   - (b) 减少 gather 循环的 vtype 往返：候选方向包括把 index 更新并入单一 vtype 域（如以 `e64` index + `vloxei64` 免去 e32 往返，或按 pattern §The fix item 2 "仅在 VL 与 VTYPE 同时匹配时复用"重构循环使每个 vtype 连续段只建立一次状态），以及把 `vmul.vx`+`vadd.vi` 的 index 算术段与 gather 数据段分组，避免每轮交替重建。
   - before：每轮 `vsetvli e32,m4; vsetvli e64,m8; vsetvli e32,m4; vsetvli e64,m8`；
   - after：每轮固定 vtype 域内完成连续 op，vtype 重建次数减半或消除。
   - correctness contract：不得改变 `vloxei32` 的 index 位宽语义（index 值 = 字节偏移 ÷ 元素大小）、不得改变 `vl`（每轮覆盖元素数不变）、不得在 call/fault 边界错误复用状态（pattern §2/§3 的 invalidation boundary 保留）。
   - 限制/风险：`vloxei32` 的 32-bit index + 64-bit data 的 SEW 不兼容是架构要求，消除全部切换需要改 index 表示（可能增大 index 寄存器占用）；须实测取舍。
   - 预期 Profile signals：`32a480: vsetvli zero,zero,e32,m4`（2.56%）占比消失/大幅缩小；`csrr vlenb` 各点（0.94%/0.15%/0.10%）退出循环体。
3. **Baseline facts 回填**：同 Finding 1（RVV 1.0，VLEN=256，compute/latency，`baseline_gap: sampling metadata` + `baseline_gap: sampling IP precision`）。
4. **收益上界**：本函数内局部样本份额 ≈ 3.75%（cited evidence 行加总）。`baseline_gap: sampling metadata` 禁止 workload 级上界；`implementation_shape_gap: gather 循环 vtype 消除的 index 表示重构取舍未实测`。
5. **三维路由判定**：current source = intrinsic/template 生成代码；implementation existence/reachability = 已向量化热点实现（同 Finding 1）；function-level policy = compiler/intrinsic 代码生成的 vector-state bookkeeping 形态。
6. **Implementation-shape proof**：不适用（非 missing `.S`）。
7. **Related PRs**（patterns/rvv_vector_state_management.md §Related PRs，保留原始 URL）：
   - Related PRs：9 条 URL —— https://github.com/v8/v8/commit/c81ffb7a356dc408940d479fbbec1d048181be71 、https://github.com/qemu/qemu/commit/d57dfe4b37ae542cec84a0cf751ecef313614cb6 、https://github.com/qemu/qemu/commit/944b6dfd3d67236882f2bc09d1d30ed923268e16 、https://github.com/qemu/qemu/commit/25669d275ce70346b94e3d5e4475d619eb979f5e 、https://github.com/qemu/qemu/commit/bd2c82283d21e3400d7d89676a221935904c2fe6 、https://github.com/qemu/qemu/commit/81b9ef995a3b2fa5b08fab0615a1c9ed7cbe053e 、https://github.com/qemu/qemu/commit/949b6bcb27295eb04350afac32a45b698fc50104 、https://github.com/qemu/qemu/commit/b8e1f32cda7805236c2bd497106a9356431c2d60 、https://github.com/llvm/llvm-project/pull/148246

## Phase 5 — Verification forecast / 验证预测：cv::rvv_hal::core::dft

Finding 1（primary，LMUL sizing）：
- **应消失/缩小**：`4.63 : 32855e: blt s3,a6,3283ce`、`1.13 : 328904: blt a0,a6,3287b8` 回边占比显著下降；`0.56 : 3283e6: vsetvli t6,zero,e32,mf2,ta,ma` 与 `0.54 : 328504: vsetvli zero,a5,e32,mf2,ta,ma` 的 `mf2` 形态消失；`1.57 : 328562: ld s6,8(sp)` 与 `0.75 : 328814: vs1r.v v10,(a2)` spill 消失（或明显减少）。
- **应出现**：f32 radix 区间 `vsetvli` 显示 `m1/m2`（或更高合法值）vtype；每轮覆盖 lane 数翻倍；无新增 vector spill/reload（pattern §Verification "register spill 验证"）；重采 annotate 中 mf2 各循环回边/配置开销收敛。
- 验证动作：以相同 `-march`（含 `v`）重建；对同一函数重跑 `perf annotate --stdio -l -s "cv::rvv_hal::core::dft..."`；覆盖短/中/长 n 与 tail（pattern §Verification "长度与 VLEN 无关性验证"）；逐元素正确性对照（LMUL 改变不改数值）。

Finding 2（independent，vector-state）：
- **应消失/缩小**：`2.56 : 32a480: vsetvli zero,zero,e32,m4,ta,ma` 占比消失/大幅缩小；`0.94 : 32851c: csrr a4,vlenb`、`0.15 : 328818: csrr a2,vlenb`、`0.10 : 32883c: csrr a2,vlenb` 退出循环体。
- **应出现**：gather 循环每轮 vtype 切换次数由 4 降至 ≤2；`csrr vlenb` 只在循环外出现一次；重采 annotate 对应区间指令数下降（pattern §Verification "count vsetvl* in the exact hot interval before and after"）。
- 验证动作：同函数重采；核对状态复用边界无 call/inline-asm clobber、VL/VTYPE 在所有 predecessor 一致（pattern §Verification）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现（载荷：1/1 组；`cv::rvv_hal::core::dft(...)`） | ✅ 1/1 组；`cv::rvv_hal::core::dft(...)` |
| 2 | Phase 1 输出要求满足（载荷：7 项结论；gap 标签：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅ 7 行含 `baseline_gap: sampling IP precision` |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；`Classes scanned: rows-vectorized-tuning.md, rows-operator-rvv.md, rows-codegen.md`；顶层 finding=2（primary 1 + independent 1）；evidence 锚点：`3283e6`/`32855e`/`328504`/`3289c8`/`328900`/`328864`/`328562`/`3296e4`/`32a480`/`32851c`/`328818`/`32883c`/`328814`；supporting=1（`328814`）；排除条数=7+20+27；推导式=2） | ✅ 三件套齐备，2 顶层 finding 各含 (a)(b)(c)，supporting 归 primary |
| 4 | Phase 4 输出要求满足（载荷：pattern 文件 2 个——`rvv_register_group_utilization.md`（命中 rows-vectorized-tuning row5，引用 `Underutilized register-group frontier`/`LMUL * peak_live_vectors <= 32`/`Control live range`/§The fix item 1/2/4/5/8）、`rvv_vector_state_management.md`（命中 row7，引用 `重复设置相同状态`/§The fix item 2/3）；The fix 含 before/after、correctness、风险与 Profile 信号锚点；Related PRs：7+9 条 URL） | ✅ 各字段逐项输出 |
| 5 | 路径合规（载荷：模式 A；class 列表 rows-vectorized-tuning/rows-operator-rvv/rows-codegen；动态份额排序生效；`th.v*` 未停扫） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧对应 Phase 3 引用行 `32855e`/`3283e6`/`328562`/`32a480`/`32851c` 等；出现侧标注 pattern §Verification） | ✅ 双侧锚定一致 |
| 7 | 契约边界合规（载荷：无实施询问、无代码修改、无补丁生成；交付止于诊断蓝图 + The fix + 验证预测） | ✅ 边界内 |

修正记录：无
