Functions under analysis: [bits_image_fetch_bilinear_affine_normal_a8r8g8b8]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`bits_image_fetch_bilinear_affine_normal_a8r8g8b8`，703 samples，cycles:u，percent: local period，覆盖 hot loop 31e62–32016 与 cold setup/epilogue）
- perf stat（bound/context）：已提供（IPC 1.406223；16,125,136,481 cycles / 22,675,535,924 instructions；无 cache/memory/branch counter）
- workload/binary/DSO/source context：已提供（libpixman-1.so.0.46.5，pixman，commit 14735ce，testcase affine-bench-bilinear-rotate-30deg；batch rank 001）
- readelf -A（build ISA）：已提供（`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` — 无 `v`）
- hardware ISA：已提供（SpacemiT X100；`rv64imafdcvh_...` 含 `v`、`zvbb`、`zvbc` 等）
- vlenb：已提供（32 bytes → VLEN 256 bits，冻结快照）
- 采样元数据：部分已提供（event=cycles:u，percent type=local period，同一运行窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（无 precise_ip/Exact-IP/skid 能力信息）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`v`（RVV 1.0）、VLEN=256（vlenb=32）、OoO SpacemiT X100；另含 zvbb/zvbc/zfa 等 |
| Build ISA | 已提供：`rv64i2p1_..._zcd1p0` **无 `v`** → L0 mismatch：hardware 有 `v` 而 build 无 `v` |
| Vector flavor | annotate 全 scalar、zero `v*` 与 `th.v*`；无 flavor mismatch，但 build 缺 `v` 是 L0 baseline finding |
| VLEN | 已提供：256 bits（vlenb=32） |
| Bound type | 部分：IPC=1.406（cycles/instructions）指向 compute/latency 主导；无 cache/memory/branch counter → `baseline_gap: bound type` |
| Sampling semantics | event=cycles:u（可解释为时间 ✓）；percent=local period（非 global）；同一运行窗口 ✓；函数级 workload 贡献未知 → 只能表述局部样本份额，禁称 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP 信息）→ 单行占比只锚定 basic block/loop interval |

L0 baseline gate：hardware 有 `v`（RVV 1.0），build 无 `v` → 最高优先级 baseline finding；`th.v*` gate：无 `th.v*` 证据，不适用。Bound-type gate：compute/latency 方向，但缺 cache/memory counter → 命中 findings 的 performance-impact confidence 受封顶。

## Phase 2 — Scope / 分析边界
函数清单：bits_image_fetch_bilinear_affine_normal_a8r8g8b8（1 个）。
hot loop 边界：per-pixel 主循环 31e62–32016（`31e62: beqz s6,31e6e` … `32016: bne s4,t1,31e62`），其中插值计算区间 31ef4–32012（四 tap 加载、通道拆/装、定点插值乘加、重组与 store）。
最高行锚点：`19.08 : 31f98: and t2,a0,t6`（插值计算区间内最高），次高 `16.51 : 31f50: srli s3,s9,0x10`、`8.76 : 31fc0: and s9,a1,t6`、`5.28 : 31fca: mul a7,s3,a7`。
annotate 覆盖完整（含 hot loop body 与 border clamp 内循环 31e94–31ef0）。
Sampling IP precision 未确认 → 单行不做 instruction-latency 归因；根因锚定插值计算区间（31ef4–32012，局部样本份额 ≈92%）。

## Phase 3 — Pattern scan / 模式扫描：bits_image_fetch_bilinear_affine_normal_a8r8g8b8

### Class selection trace（8 项）
1. rows-asm.md — exclude：当前代码来源为 compiler-generated C（pixman 源内 bilinear affine fetcher，函数名与 C 实现一致；无 `.S`/DWARF/object-mapping 证据）；无 policy-backed missing `.S` 四证。
2. rows-operator-rvv.md — include：compiler-generated scalar loop，语义为 bilinear affine resampling（16.16 分数坐标、运行期权重、数据相关四 tap load、插值 mul/add、border clamp）。
3. rows-string-memory.md — exclude：非 copy/fill/sentinel/compare/checksum；热点是插值算子。
4. rows-vectorized-tuning.md — exclude：完整 annotate zero `v*`（该类要求已有 `v*` 且非手写 `.S`）。
5. rows-codegen.md — include：compiler-generated 代码形态；逐行评估 kernel-selection、register-pressure、native-width、extension-substitution、addressing、induction 等行。
6. rows-offload.md — exclude：无矩阵引擎/packed-SIMD/权重重排证据；非 GEMM。
7. rows-crypto.md — exclude：非密码原语。
8. rows-runtime-os.md — exclude：非 RTOS/kernel/timer/CSR 热点。

Classes scanned: rows-operator-rvv.md、rows-codegen.md

### Local performance pattern scan: bits_image_fetch_bilinear_affine_normal_a8r8g8b8

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | 主循环 31e62–32016 全 scalar：分数坐标 `sraiw a1,t5,0x10`、权重提取 `sraiw a2,t5,0x9`+`andi a2,a2,254`、`subw t2,s5,a2`（s5=256 权重互补）、四 tap 数据相关 load、插值 `mul`+`add`、border clamp 循环 | High | Medium | `patterns/rvv_resampling_kernels.md` |
| No vectorization（supporting） | 同一 hot main loop zero `v*`/`th.v*`，全 scalar integer load/compute/store；hardware 有 `v`；build ISA 无 `v` | — | — | `patterns/no-vectorization.md` |

(a) 逐字 evidence 引用（primary，RVV Resampling Kernels）：
- `19.08 : 31f98: and t2,a0,t6`（插值计算区间 31ef4–32012 内最高行——ARGB 通道提取与定点插值的标量实现）
- `16.51 : 31f50: srli s3,s9,0x10`（同区间：通道右移取 G 通道）
- `3.39 : 31f2e: lw a6,0(a4)` / `3.11 : 31f6c: lw a0,0(a0)`（四 tap 数据相关 load）
- `5.28 : 31fca: mul a7,s3,a7`、`0.85 : 31fbe: add a5,a5,a6`（插值乘加）
- 权重/坐标合同：`0.85 : 31f0a: andi a2,a2,254`、`31f14: subw t2,s5,a2`（s5=256，见 `31e5a: li s5,256`）、`31e76: sraiw a1,t5,0x10`、`31e8c: sraiw a2,t5,0x9`
- border mode：`31e94: blt a1,a7,31ea0` 及 31e98–31ef0 的 clamp/reflect 内循环
(b) 互斥邻居排除：
- 无插值语义的 indexed gather（rows-operator-rvv.md row 22）：排除——区间含 256 制权重互补 `subw t2,s5,a2` 与加权插值 `mul a6,a6,s9`/`add a5,a5,a6`，是 bilinear 插值合同而非任意 LUT 查找。
- 固定 stencil tap 的 spatial-convolution（row 28）：排除——四 tap 地址由运行期分数坐标（`sraiw a1,t5,0x10`）推导，非固定偏移 stencil。
- color-conversion（row 24）：排除——无 RGB/BGR/YUV 颜色矩阵与 chroma sampling；`and`/`slli`/`srli`/`or` 序列是 packed ARGB 上插值算术本身，非独立颜色转换 pass。
- layout/channel packing（row 23）：排除——无 segment/stride/narrow load-store 主导；loads 为 4 个普通 `lw`，位运算服务于插值权重算术。
(c) 双 Confidence 推导式：
- route: 主循环 scalar + 语义合同（bilinear affine 16.16，权重 256 制）+ 四 tap 数据相关 load + 插值 mul/add + border clamp → High
- impact: 局部样本份额 ≈92%（插值区间）/≈100%（整个函数）成立、VLEN 256 已知、build baseline 已知；但采样语义非 global（函数级贡献未知）+ 缺 cache/memory counter（bound type 细分）+ IP precision 未知 → Medium（仅方向性收益表述，禁 Amdahl）

多命中仲裁小段（arbitration）：
- L0 baseline finding（Step 0）：hardware 有 `v` 而 build 无 `v` —— binary 不可能包含 RVV 指令，是一切下层信号的前置。
- primary：RVV Resampling Kernels（L1 semantic，机制 = 逐输出重复标量坐标/权重计算 + scalar gather + 标量通道插值）；supporting：No vectorization（L1，同一地址区间同一机制，只解释"无向量执行"这一事实，不决定贡献载体）→ 顶层 finding 数 = 1。
- evidence mechanism layer：resampling 语义为 L1；channel unpack/repack 的标量位运算属该插值算术在 L4 的实现细节，被 L1 修复（向量化插值）自然消除 → 不单列 codegen finding。
- 入口模式 A：顶层 finding 的 evidence sample share 加总 ≈92%（插值计算区间 31ef4–32012 / 703 局部样本）。

rows-codegen.md 逐行排除（节选判别性行）：
- kernel-selection（row 13）：排除——本 annotate 内无 dispatch/lookup/fallback 证据（分派发生在 pixman_image_composite32，独立函数，本批 rank 005）；无证据表明更快的已适配 kernel 未选中。
- register-pressure（row 21）：排除——hot loop 31e62–32016 内无 stack spill/reload（`sd`/`ld` 至栈帧仅在 cold setup 31e54/31e5e 出现）。
- native-width state（row 32）/ redundant extension（row 33）：排除——`slli 0x20;srli 0x20`、`andi`、`slliw;andi` 是 packed 32-bit ARGB 通道提取/8.8 权重算术的必要实现（源码语义 = 8-bit 通道），非冗余 truncate/extension；range proof 会破坏 packed layout 合同。
- ISA-substitution（row 31）：排除——hot 位运算/乘加均为运行期操作数，无 Zbb 多指令合成序列（`not`+`and` 等）证据。
- induction-variable（row 39）：排除——hot loop 已用指针递增（`31f1c: addi t1,t1,4`）与预计算 end pointer（`31e42: add s4,s4,s2`、`32016: bne s4,t1`）比较，已做强度削减。
- addressing-mode fusion（row 20）：排除——`31f2a: add a5,a5,a1` 等是二维多基址像素地址算术（4 个 tap 基址不同），非单 AGU offset 折叠候选。
- compiler-workaround（row 16）、hot-helper-inlining（row 17）、control-flow（row 18）、code-layout（row 19）、ALU-constant（row 27）、gp-relative（row 22）、FP-lowering（row 23）、trap（row 24）、resource-aware-scheduling（row 25）、algebraic（row 26）、atomic（row 29）、spin（row 30）、zero-based（row 36）、tail-call（row 35）、redundant-sync（row 37）、JIT（row 38）、fusion（row 15）、cache-blocking（row 14）、runtime-ISA-dispatch（row 34）：排除——对应 signal 缺失（hot loop 无 helper call、无 branch-diamond 主导、无 constant-pool 主导、无 gp 访问、无 fence/atomic、无 spin、无 `li reg,0`+branch 形态、无 workaround 证据；scheduling 行要求 target-core PMU/dependency 证据，IP precision 未知且无 stall 证据 → 不命中）。

## Phase 4 — Root-cause blueprint / 根因蓝图：bits_image_fetch_bilinear_affine_normal_a8r8g8b8

（对应 Phase 3 命中 row：RVV Resampling Kernels（primary）；supporting：No vectorization）

1. Root cause：
- 机制（依据 `patterns/rvv_resampling_kernels.md` §Why this is slow）："重采样的主要根因是逐输出重复坐标计算、数据相关地址导致 scalar gather、bilinear/cubic 权重与边界处理割裂成多遍，以及坐标/权重精度阻止安全向量化"——本函数主循环 31e62–32016 每输出像素重复执行 16.16 分数坐标→索引/权重推导（`sraiw`×2、`slliw;andi 254`、`subw 256-权重`）、四 tap 数据相关标量 load、逐通道 8.8 定点插值（`mul`+`add`）与 packed ARGB 通道拆/装（`and`/`slli`/`srli`/`or`），以及 border clamp 内循环；插值计算区间 31ef4–32012 承载 ≈92% 局部样本，其中通道拆装位运算与标量定点乘加行（`19.08 : 31f98: and t2,a0,t6`、`16.51 : 31f50: srli s3,s9,0x10`、`8.76 : 31fc0: and s9,a1,t6`、`5.28 : 31fca: mul a7,s3,a7`）为样本主导——即"每个输出 lane 有运行期 source index/weight"（§When to apply 正向 signal）的 scalar gather + 标量插值形态。
- L0 前置：hardware 有 `v`（RVV 1.0, VLEN=256）而 build ISA 无 `v` → 当前 binary 不可能含 RVV 指令，即使存在向量化实现也不会被编入。

2. The fix / 修复方式：
- 纠正对象：bilinear affine fetcher 的整条 scalar 主循环（坐标/权重预计算 + 四 tap gather + 通道插值），纠正方式为按 §7 "Vectorize bilinear resampling with indexed loads" 与 §6 "Precompute resampling indices and weights" 组织向量化；同时先按 no-vectorization §The fix 第 1 步用硬件/工具链共同支持的精确 `-march`（含 `v`，如 `rv64gcv` 变体，X100 另支持 zvbb 等）重建 build。
- Before（现状形态，来自 annotate）：
```
# 每像素：分数坐标 sraiw 0x10/0x9 → 权重 andi 254 / subw 256-权重 → 4× lw 四 tap
# → 逐通道 slli/srli/andi 拆包 + mul/add 8.8 定点插值 → or 重组 → sw 输出
31f50: srli s3,s9,0x10        # 通道拆
31f98: and  t2,a0,t6          # 通道拆
31fca: mul  a7,s3,a7          # 定点插值
31fbe: add  a5,a5,a6          # 插值加
```
- After（预期形态示意，非固定实现配方；依据 pattern §7 结构）：
```
// 每输出行：预计算 x0/x1 byte offsets + 水平权重数组（§6）
vsetvli  vl, remaining, e32, m1, ta, ma
vle32.v  x0Off, x0ByteOffsets+i, vl
vluxei32.v topL, topRow, x0Off, vl        // 四 tap indexed load
vluxei32.v topR, topRow, x1Off, vl
vluxei32.v botL, botRow, x0Off, vl
vluxei32.v botR, botRow, x1Off, vl
// 通道拆解后 16-bit 定点：vwmulu/vwmacc 保持 8.8 权重(0..256) 合同
// 垂直插值按 weightY，水平插值按 weightX，arith order 与 reference 一致
vse32.v  out, packedResult, vl
```
- 适用前提：像素 4 通道共享同一坐标/权重合同（affine 变换每行可增量推进，权重/索引可预计算复用）；VLEN 256 ≥ 多 lane 有效收益；需证明 indexed load 在该核上优于 scalar gather（§Verification 失败判据之一）。
- correctness contract（不可破坏）：16.16 定点坐标与 floor/rounding 规则（half-pixel/align-corners 合同）、border clamp/reflect 语义（31e94–31ef0 内循环语义）、8-bit 通道 packed ARGB 布局、8.8 权重归一化（两权重之和 = 256，`subw t2,s5,a2` 合同）、算术顺序（reference 定义的通道插值顺序，FMA/定点 widening 不得改变 reference 舍入）、index 为 byte offset 且宽度不截断大图。
- 限制/风险：indexed load（vluxei32）在目标核的吞吐未知；4 tap × index/weight 向量的 live set 可能引发 LMUL 压力/spill；预计算流量对小尺寸退化；短 scanline 应比较预计算 vs 即时计算（§6 说明）；多通道 interleaved 图不能直接套单平面示例。
- 修复后预期 Profile signals：插值计算区间 31ef4–32012 的标量 `and`/`srli`/`slli`/`mul`/`add` 行与四 tap `lw` 行大幅缩小，出现 `vsetvli`/`vle32`/`vluxei32`/`vwmacc`（或 FP 变体 `vfmacc`）指令行；cycles/output 改善（局部）。

3. Baseline facts 回填：hardware ISA = RVV 1.0（X100，含 `v`/`zvbb`）；build ISA = 无 `v`（`rv64i2p1_m2p0_..._zcd1p0`）；VLEN = 256 bits（vlenb=32）；bound type = compute/latency 方向（IPC 1.406）+ `baseline_gap: bound type`（缺 cache/memory counter）。

4. 收益上界：入口模式 A；该 finding 的 evidence sample share 加总 ≈ 92%（插值计算区间 31ef4–32012，703 局部样本中的份额；整个重采样循环 ≈100%）。表述为「当前 sampled event 下的局部样本份额」；因 percent=local period 且函数级 workload 贡献未知 → 不得称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`）。

5. 三维路由判定：
- current source：compiler-generated C（pixman `bits_image_fetch_bilinear_affine_normal_a8r8g8b8`，libpixman-1.so.0.46.5；无 `.S`/DWARF/object-mapping 反证）→ 普通 compiler-generated scalar loop。
- implementation existence/reachability：本 DSO 中该符号即此实现；pixman 存在 fast-path 分派体系（`pixman_image_composite32` 为独立热点，rank 005）但本函数 annotate 内无 dispatch 证据 → kernel-selection 不命中；build ISA 无 `v` 意味着任何 RVV 实现当前不可达。
- function-level policy：pixman fetcher 层以 C + 可选 SIMD fast path 组织；缺 policy/existence 四证（dispatch slot/scalar fallback/缺失目标 `.S`/policy 要求独立 `.S`）→ 不进入 missing `.S` 分支；fix 形态 = 精确 `-march`（含 `v`）重建 + 普通代码/intrinsic 向量化 kernel。

6. Implementation-shape proof：不适用（非 policy-backed missing `.S` 分支）。

7. Related PRs：
- `patterns/rvv_resampling_kernels.md`：Related PRs：2 条 URL（https://github.com/alibaba/MNN/pull/4053、https://github.com/alibaba/MNN/commit/f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d）
- `patterns/no-vectorization.md`：Related PRs：14 条 URL（https://github.com/opencv/opencv/pull/22179、https://github.com/opencv/opencv/pull/22520、https://github.com/opencv/opencv/pull/23980、https://github.com/opencv/opencv/pull/24058、https://github.com/opencv/opencv/pull/24132、https://github.com/opencv/opencv/pull/24166、https://github.com/opencv/opencv/pull/24301、https://github.com/opencv/opencv/pull/24325、https://github.com/opencv/opencv/pull/27160、https://github.com/opencv/opencv/pull/27119、https://github.com/opencv/opencv/pull/27097、https://github.com/opencv/opencv/pull/27007、https://github.com/opencv/opencv/pull/26958、https://github.com/opencv/opencv/pull/26865、https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d、https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a、https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706、https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f、https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a）

## Phase 5 — Verification forecast / 验证预测：bits_image_fetch_bilinear_affine_normal_a8r8g8b8

primary（RVV Resampling Kernels）：
- 应消失/缩小：`19.08 : 31f98: and t2,a0,t6`、`16.51 : 31f50: srli s3,s9,0x10`、`8.76 : 31fc0: and s9,a1,t6`、`5.28 : 31fca: mul a7,s3,a7`、`3.39 : 31f2e: lw a6,0(a4)`、`3.11 : 31f6c: lw a0,0(a0)`、`2.40 : 32012: sw a5,-4(t1)` 的局部样本份额显著下降（scalar gather 与逐通道标量插值消失或缩小）。
- 应出现（依据 `patterns/rvv_resampling_kernels.md` §Verification）：逐 lane coordinate/index arithmetic 与 scalar loads 下降、出现预期 indexed vector loads（`vluxei32`）与 vector 插值指令（`vwmacc`/`vfmacc`），cycles/output 改善；失败判据：坐标/border 偏一、indexed load 在目标核更慢、index vector/EMUL 导致 spill、预计算流量超过节省、小尺寸退化。
- supporting（No vectorization）跟随 primary 验证：同一 annotate 中 hot loop 出现 `vsetvli`/`vle*`/`vse*` 等 RVV 指令（`patterns/no-vectorization.md` §Verification），scalar 指令不再主导 hot loop；与 scalar reference 对比覆盖空/小/整倍数/tail 长度，报告 cycles per pixel。
- 附：重建后先以 `readelf -A` 复核 libpixman DSO `Tag_RISCV_arch` 含 `v`，并确认真实 workload 的 dispatch 采用路径。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status + Anchor |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组；`bits_image_fetch_bilinear_affine_normal_a8r8g8b8` |
| 2 | Phase 1 输出要求 | ✅ 7 行 baseline 表；gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ 8 项 Class selection trace（asm exclude / operator-rvv include / string-memory exclude / vectorized-tuning exclude / codegen include / offload exclude / crypto exclude / runtime-os exclude）；`Classes scanned:` rows-operator-rvv.md、rows-codegen.md；顶层 finding 数 = 1（primary：RVV Resampling Kernels；supporting：No vectorization）；evidence 锚点：`19.08 : 31f98: and t2,a0,t6`、`16.51 : 31f50: srli s3,s9,0x10`、`3.39 : 31f2e: lw a6,0(a4)`、`0.85 : 31f0a: andi a2,a2,254`、`31f14: subw t2,s5,a2`；supporting 数 = 1；排除条数 = 5（gather/spatial-conv/color/layout）+ rows-codegen 全行排除；推导式：route High / impact Medium |
| 4 | Phase 4 输出要求 | ✅ 已读 pattern：`patterns/rvv_resampling_kernels.md`（命中 row：RVV Resampling Kernels；引用短语："重采样的主要根因是逐输出重复坐标计算"、"Vectorize bilinear resampling with indexed loads"）、`patterns/no-vectorization.md`（命中 row：No vectorization；引用短语："先核对 hardware ISA、实际 hot object 的 build ISA、vector flavor 和真实 dispatch path"）；The fix 含 before/after、correctness、风险、Profile signals 锚点；Related PRs：resampling 2 条 URL、no-vectorization 14 条 URL |
| 5 | 路径合规 | ✅ 模式 A（profile_backed）；8 项 class 扫描集 trace 齐全；顶层 finding 1 个来自通过 gate 的 row；`th.v*` 无证据未停扫；L0 hardware-v/build-no-v 作为 baseline finding 置顶 |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧：`31f98: and t2,a0,t6`、`31f50: srli s3,s9,0x10`、`31fca: mul a7,s3,a7`、`31f2e: lw a6,0(a4)`、`32012: sw a5,-4(t1)`；出现侧：`patterns/rvv_resampling_kernels.md` §Verification + `patterns/no-vectorization.md` §Verification |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁生成；无向用户追问（无 object-clarification 场景）；交付止于 Profile 证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无