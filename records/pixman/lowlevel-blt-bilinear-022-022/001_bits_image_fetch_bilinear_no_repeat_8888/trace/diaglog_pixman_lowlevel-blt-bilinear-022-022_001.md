Functions under analysis: [bits_image_fetch_bilinear_no_repeat_8888]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`libpixman-1.so.0.46.5`，`bits_image_fetch_bilinear_no_repeat_8888`，2136 samples，`cycles:u`，`percent: local period`，含 hot loop 完整主体）
- perf stat（可选 bound/context）：已提供（`21-pixman-benchmark-riscv-lowlevel-blt-bilinear-022-022.txt`：duration 71.2s，cpu_cycle 156.5G，instruction 353.4G，IPC 2.257969；无 cache/memory/branch counter）
- workload/binary/DSO/source context：已提供（pixman 0.46.5 libpixman-1.so.0.46.5；metadata commit 14735ced17e0053abbb925f9cf18c05ed9f52378；源码/符号映射不完整，详见 Phase 1）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries.lowlevel-blt-bench-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`，**无 `v`**）
- hardware ISA（/proc/cpuinfo 或 riscv_hwprobe）：已提供（SpacemiT X100，`rv64imafdcvh_...`，含 `v` 与 `zvbb/zvbc/zvkg/zvkned/zvknha/zvkb/zvfh` 等）
- `vlenb`：已提供（32 bytes → VLEN 256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cycles:u`；percent type=`local period`；同一运行窗口；该函数占整个 workload 的贡献未知 → 只能表述函数内局部份额）
- Sampling IP precision：缺失（precise_ip / Exact-IP / skid 能力未提供，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：SpacemiT X100，`v`（RVV 1.0）暴露，另有 `zvbb/zvbc/zvkg/zvkned/zvknha/zvkb/zvfh/zvfhmin` 等可选扩展（hw snapshot `cpuinfo.isa`） |
| Build ISA | 已提供：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` —— **不含 `v`**；无 IFUNC/multiversion 证据 |
| Vector flavor | annotate 全 scalar（`lw/sw/mul/mulw/addw/sraiw/slli/and/or`），区间内 zero `v*` 与 `th.v*` → 无 flavor mismatch；根因是 build 缺 `v` |
| VLEN | 已提供：vlenb=32 → 256 bits |
| Bound type | 部分：全 workload IPC=2.257969（高 IPC，compute-ish）；PMU 无 cache/memory/branch counter → `baseline_gap: bound type (cache/memory/branch counters)`；可选命令 `perf stat -e cycles,instructions,cache-misses,branches,branch-misses -- <workload>` |
| Sampling semantics | event=`cycles:u`（可解释为时间）；percent type=`local period`（非 global-period）；同一窗口；函数级 workload 贡献未知 → 收益上界只能表述为「当前 sampled event 下的局部样本份额」 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（precise_ip/skid 能力未知）→ 单行占比只锚定 basic block / loop interval，不做 instruction-latency 归因 |

**L0 baseline gate（置顶 finding）**：hardware 有 `v`，而 build `Tag_RISCV_arch` 无 `v` → 该 binary 不可能包含 RVV 指令，编译器无法为该函数做向量化。此 mismatch 是最优先 baseline finding；不停止扫描，继续评估可见 row。`th.v*` gate：无 `th.v*` 证据，不适用。Bound-type gate：缺 cache/memory counter，本轮命中 performance-impact confidence 相应封顶（不得为 High 依据）。

## Phase 2 — Scope / 分析边界

函数清单：`bits_image_fetch_bilinear_no_repeat_8888`（rank 001/17，2136 samples）。

Hot loop interval：**0x39f66–0x3a0ac**（fast-path 逐像素主循环，back-edge `3a0ac: bnez t5,39f66`）。该区间承载本函数几乎全部 samples（local period 加总 ≈ 100%，含舍入）。次要区间 0x39e2c 循环与 0x3a10e 循环 samples 均为 0.00，非活动热点。

最高占比行（trace anchors，均为 loop interval 内）：
- `6.32 : 3a070: ld a0,-216(s0)`（stack reload）
- `6.04 : 3a07c: mul t6,s7,t6`（插值乘法）
- `5.38 : 3a00c: mul t0,t0,s7`（插值乘法）
- `3.75 : 3a08e: sw a2,0(s1)`（输出 store）

Sampling IP precision 未确认 → 以上各行只锚定 loop interval 0x39f66–0x3a0ac，不单独承担 instruction-latency 根因。annotate 覆盖完整（hot loop body 在覆盖内）。

## Phase 3 — Pattern scan / 模式扫描：bits_image_fetch_bilinear_no_repeat_8888

**Class selection trace（8 项）**：
1. `rows-asm.md` — **exclude**：当前代码来源无 `.S` provenance（compiler 风格 prologue 含 `__stack_chk_guard`，无手写汇编特征；无 policy/existence 四证）
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar 循环，算子语义为 bilinear resampling
3. `rows-string-memory.md` — **exclude**：非 copy/fill/scan/compare/checksum/back-ref；是像素重采样算子
4. `rows-vectorized-tuning.md` — **exclude**：annotate 零 `v*`，无既有 RVV 循环可调优
5. `rows-codegen.md` — **include**：compiler-generated code；hot loop 内观察 stack spill/reload（register-pressure 信号）与逐像素地址重算
6. `rows-offload.md` — **exclude**：hw profile 无矩阵引擎/packed-SIMD 证据；workload 非 GEMM
7. `rows-crypto.md` — **exclude**：无密码学原语
8. `rows-runtime-os.md` — **exclude**：用户态 pixman 库，非 RTOS/kernel timer/ISR/CSR 路径

`Classes scanned: rows-operator-rvv.md, rows-codegen.md`

### Local performance pattern scan: `bits_image_fetch_bilinear_no_repeat_8888`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | hot loop 逐像素做分数坐标变换、4 tap scalar gather、固定点权重插值乘加与 border 检查（见三件套 a） | High | Medium | `patterns/rvv_resampling_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`；hardware 有 `v`；build 无 `v` | — | — | `patterns/no-vectorization.md` |
| Register Pressure and Save/Restore（supporting） | hot loop 内成组 stack `ld/sd`（`3a070`、`3a04e`、`39f8e`、`39ff4` 等） | — | — | `patterns/register_pressure_and_save_restore.md` |

**Primary 三件套**：

(a) 逐字 evidence 引用（hot interval 0x39f66–0x3a0ac）：
- `2.06 : 39f72: sraiw a2,a3,0x10`（逐像素提取 16.16 定点 x 分数坐标）
- `1.22 : 39f9a: lw a7,0(a2)` 与 `3.56 : 39fe0: lw a2,4(a2)`（同一源行相邻 texel 的 scalar gather，数据相关地址）
- `2.20 : 39fae: lw a0,4(a1)`（另一行的 texel load）
- `5.38 : 3a00c: mul t0,t0,s7` 与 `6.04 : 3a07c: mul t6,s7,t6`（固定点权重插值乘法）
- `3.75 : 3a08e: sw a2,0(s1)`（输出 pixel store）

(b) 互斥邻居排除：
- spatial-convolution row：地址来自运行期坐标 `(x>>16)` 与权重 `256-(fx>>8<<1)` 计算，非固定 stencil tap → 不命中
- gather（indexed lookup）row：有明确插值语义（权重乘加合成），非无插值语义的任意 LUT gather → 不命中
- layout/color-conversion rows：无颜色矩阵/chroma sampling；8888 通道 mask 拆解（`0xff0000ff`/`0x00ff00ff` 类常量 `t2/s3/s6/s5`）是插值算术的一部分而非主合同 → 不命中
- strided-layout row：无固定 stride 搬运 → 不命中

(c) 双 Confidence 推导式：
- `route: compiler-generated scalar provenance + bilinear 语义四要素（坐标提取/4 tap 数据相关 load/权重乘加/border 检查）逐字证据 + 互斥排除完成 → High`
- `impact: hot interval 占函数局部 samples ≈ 100%（2136 samples, local period）+ hardware V 与 VLEN=256 已知，但缺 cache/memory counter（bound type）、percent=local period、sampling IP precision 未知 → Medium`

**Supporting evidence**：
- no-vectorization：一行 (a) `main loop 0x39f66–0x3a0ac 全 scalar（lw/mulw/mul/and/or/sw），区间内 zero v*；hardware 有 v、build Tag_RISCV_arch 无 v`。supporting because：同一 scalar main loop 缺向量执行载体，build 缺 `v` 是直接原因；按 rows-operator-rvv 的 no-vectorization 行，本行只作更具体 semantic row（resampling）的 supporting，不另列顶层 finding。
- register pressure：一行 (a) `6.32 : 3a070: ld a0,-216(s0)`、`3.84 : 3a04e: ld t5,-192(s0)`、`3.18 : 39f8e: ld a0,-192(s0)`、`1.92 : 39ff4: sd a1,-200(s0)`（hot loop 内成组 stack spill/reload）。supporting because：同一标量逐像素 bilinear 计算的峰值 live set 过大迫使 RA 溢出；上层向量化消除该计算形态后 spill/reload 信号自然消失（因果消除测试），故不独立成顶层 finding。

**多候选仲裁小段**：顶层 finding 仅 1 个 —— primary = **RVV Resampling Kernels**（证据机制层 L1 vectorization/semantic dispatch；运行期坐标/index/weight 驱动 gather 与插值乘加全部由该 row 认领）。supporting = no-vectorization（L1 同一 loop 的零向量执行方面）、register-pressure（L4 codegen micro-structure，上层修复可消除其 signal，并入 primary）。无 companion、无 independent。收益上界按入口条件 A 排序：仅一个顶层 finding，排序无歧义；supporting 不单独排序。

## Phase 4 — Root-cause blueprint / 根因蓝图：bits_image_fetch_bilinear_no_repeat_8888

对应 Phase 3 通过 gate 的 row：`rows-operator-rvv.md` RVV Resampling Kernels 行（primary）；supporting：no-vectorization 行、rows-codegen.md register-pressure 行。

1. **Root cause**：该 fetch_scanline 为 compiler-generated 标量逐像素 bilinear resampling。依据 `patterns/rvv_resampling_kernels.md` §Why this is slow 的关键机制句（短引）："逐输出重复坐标计算、数据相关地址导致 scalar gather、bilinear/cubic 权重与边界处理割裂成多遍"。annotate 逐字证实：每像素重做 `sraiw a2,a3,0x10`（坐标→index）→ `slli`+`add` 地址生成 → 4 次数据相关 scalar `lw`（0/4 offset 两行两列）→ 通道 mask 拆解（`s3/s6/s5/t2` 常量）→ 8+ 次 `mul/mulw` 固定点权重合成 → `sw` 输出；权重每像素用 `subw`/`slliw` 重算。叠加 **L0 baseline finding**：build `Tag_RISCV_arch` 无 `v`（hardware 有 RVV 1.0 / VLEN 256），编译器不可能向量化，vector unit 完全未参与。

2. **The fix / 修复方式**（纠正对象与操作方式）：
   - **第 1 步（L0 build）**：用硬件与工具链共同支持的**精确** `-march`（含 `v`，如 X100 实测验证的 `rv64gcv` 等价集）clean rebuild libpixman，使 autovec / intrinsic RVV 成为可能（依据 `patterns/no-vectorization.md` §The fix 第 1 条）。
   - **第 2 步（RVV 向量化 fetch）**：按 `patterns/rvv_resampling_kernels.md` §6「Precompute resampling indices and weights」与 §7「Vectorize bilinear resampling with indexed loads」：
     - 行内预计算 x0/x1 source **byte offset** 数组与水平权重数组（§6 列表：左右 tap index、byte offset、水平权重、垂直 row/权重、border 映射后合法 index），成本由整行输出像素摊薄；
     - 用 indexed vector loads（`vluxei32`，index 为 byte offset 不是元素 index）一次取 4 tap；
     - 对 8888 打包格式：SEW=32 载入 4 tap 后做通道拆解（mask/shift，或按需 `vwmulu`/`vwmaccu` widening 定点插值），保持现有 8.8 定点权重（256−fx）与 16.16 坐标合同；垂直方向用行权重在 tap 间合成；`no_repeat` border 经 mask/饱和保持；
     - VLEN-agnostic：fixed-VL main loop + runtime-VL tail（依据 kernel-conventions 的 VLEN-agnostic 约定），LMUL 按 live-vector budget 选择（4 tap + weight + 中间值 ≈ 需评估 register budget）。
   - **修复前/后伪代码**：
     ```c
     // Before（当前 scalar per-pixel，hot loop 0x39f66–0x3a0ac）:
     for (x = ...) {
         fx = xacc >> 16; w = 256 - ((xacc >> 8) & 254);
         top0 = row0[fx]; top1 = row0[fx+1]; bot0 = row1[fx]; bot1 = row1[fx+1];
         // per-channel mask/split + mul/mulw interpolation + recombine + sw
     }
     // After（RVV 向量形态，index 为 byte offset；示意，非最终实现）:
     for (outX = 0; outX < W; outX += vl) {
         vl = vsetvli(...);
         x0 = vle32(x0ByteOffsets + outX); x1 = vle32(x1ByteOffsets + outX);
         hw = vle32(hWeights + outX);
         tl = vluxei32(row0, x0, vl); tr = vluxei32(row0, x1, vl);
         bl = vluxei32(row1, x0, vl); br = vluxei32(row1, x1, vl);
         // 8888: channel-split -> widen-mul w -> 垂直合成 -> pack -> vse32
     }
     ```
   - **适用前提**：build 恢复含 `v` 的精确 `-march`；行长度 ≥ 若干 lane 时才有收益（短行/单次调用需比较预计算流量，§6 明确"预计算只有在其成本能够被多个通道、行或调用复用时才有收益"）。
   - **不可破坏的 correctness contract**：`no_repeat` border 语义（坐标越界路径 `3a22a/3a258` 分支与 memset/zero-fill 行为）；16.16 定点坐标与 8.8 权重（256−fx）的 arithmetic order 与 reference 一致；8888 通道 mask（0xff0000ff / 0x00ff00ff / 0x0000ffff 形态）拆解-合成顺序；`vluxei32` index 为 byte offset、大图需防 32-bit offset 截断（§7 注意事项）。
   - **限制/风险**：indexed load 在目标核（SpacemiT X100，OoO）的吞吐未实测；index vector/EMUL 可能引起 spill；预计算流量在短 row 上可能超过节省（§Verification 失败判据）；算术顺序改变需对照 reference。
   - **修复后预期 Profile signals**：hot loop 的 `sraiw/slli/add` 坐标序列、4 次 scalar `lw`、`mul/mulw` 群与 stack `ld/sd`（`3a070/3a04e/39f8e/39ff4`）占比消失或显著下降；出现 `vsetvli`、`vluxei*`、vector widen-mul、`vse*`。

3. **Baseline facts 回填**：hardware ISA = RVV 1.0（`v`）+ zvk 系列（X100）；build ISA = rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_…（无 `v`）；VLEN = 256（vlenb=32）；bound type = `baseline_gap: cache/memory/branch counters`（全 workload IPC 2.258）；sampling = cycles:u / local period / IP precision 未知。

4. **收益上界**：hot interval 0x39f66–0x3a0ac 占本函数 local samples ≈ 100%（2136 samples，percent=local period）。因 percent type 为 local period、函数级 workload 贡献未知，表述为「当前 sampled event（cycles:u）下的局部样本份额」，不称 workload 级 Amdahl 上界。

5. **三维路由判定**：
   - current source：compiler-generated C（libpixman-1.so.0.46.5；无 `.S`/DWARF provenance）→ 主归属 rows-operator-rvv；
   - implementation existence/reachability：本 build 无 `v`，binary 内不可能有 RVV kernel；pixman fast-path 机制存在（annotate 可见 `fast_path_cache` 符号），但无源码/对象映射证明存在 rvv bilinear kernel 或 dispatch 接线 → `source_context_gap`；
   - function-level policy：pixman 存在 per-arch SIMD fast-path policy，但 policy/existence 四证（dispatch slot / scalar fallback / 目标 `.S` 缺失 / policy 要求独立 `.S`）不齐 → **不进入 policy-backed missing `.S` 分支**，fix 停留在「重建 + 向量化（intrinsic/C 层）」。
6. **Implementation-shape proof**：N/A（非 policy-backed missing `.S` 分支）。
7. **Related PRs 小节**：
   - `patterns/rvv_resampling_kernels.md` → Related PRs：2 条 URL（MNN #4053；MNN f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d）
   - `patterns/no-vectorization.md` → Related PRs：19 条 URL（OpenCV #22179、#22520、#23980、#24058、#24132、#24166、#24301、#24325、#27160、#27119、#27097、#27007、#26958、#26865；b902a8e792e1、2c16f3b7d2b2、e06502a254f7、a2d784b6f53a、83104bed3209）
   - `patterns/register_pressure_and_save_restore.md` → Related PRs：37 条 URL（V8 25 条、QEMU 8 条、Zephyr 4 条，见 pattern 表）

## Phase 5 — Verification forecast / 验证预测：bits_image_fetch_bilinear_no_repeat_8888

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`39f72: sraiw a2,a3,0x10`（逐像素坐标提取）、`39f9a: lw a7,0(a2)`、`39fe0: lw a2,4(a2)`、`39fae: lw a0,4(a1)`（scalar gather）、`3a00c: mul t0,t0,s7`、`3a07c: mul t6,s7,t6`（scalar 插值乘）、`3a070: ld a0,-216(s0)`、`3a04e: ld t5,-192(s0)`（spill reload）的 local sample 占比应消失或显著缩小。
- **应出现侧**（锚定 `patterns/rvv_resampling_kernels.md` §Verification）：fetch loop 出现 indexed vector loads（`vluxei*`）、`vsetvli`、vector widening 定点乘加与 `vse*`；逐 lane coordinate/index arithmetic 与 scalar loads 下降；cycles/output 改善。§Verification 失败判据同时列出：坐标/border 偏一、indexed load 在目标核更慢、index vector/EMUL 导致 spill、预计算流量超过节省、小尺寸退化。
- **验证方法**：目标精确 `-march`（含 `v`）+ 相同优化级别重建；同一函数重跑 annotate；短/中/长 row 与不同 no_repeat 边界样本 benchmark；correctness 覆盖 half-pixel/边界/最后像素/奇偶尺寸；整数定点 arithmetic order 与 reference 逐位一致（无 FP rounding 问题，但通道拆解合成顺序必须保持）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组；`bits_image_fetch_bilinear_no_repeat_8888` | ✅（载荷：1/1 组；`bits_image_fetch_bilinear_no_repeat_8888`） |
| 2 | Phase 1 输出要求满足：7 行 baseline（结论数 7，含 `Sampling IP precision` 行）；gap 标签：`baseline_gap: bound type (cache/memory/branch counters)`、`baseline_gap: sampling IP precision`；L0 gate 判定 2 项 + bound-type gate | ✅ |
| 3 | Phase 3 输出要求满足：8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（RVV Resampling Kernels），evidence 锚点 `39f72/39f9a/39fe0/39fae/3a00c/3a07c/3a08e`；supporting 2（no-vectorization、register-pressure）；排除 5 条（spatial-conv/gather/layout-color/strided/其余 class）；推导式 2（route/impact） | ✅ |
| 4 | Phase 4 输出要求满足：已读 pattern 3 个；命中 row 3 行（resampling 行、no-vectorization 行、register-pressure 行）；引用短语首词：`重采样的主要根因`、`Precompute resampling indices and weights`、`Vectorize bilinear resampling with indexed loads`；`The fix` 含 before/after 伪代码、correctness contract（no_repeat/16.16 定点/8888 mask/byte-offset index）、风险（indexed-load 吞吐、EMUL spill、短行退化）与 Profile signals；Related PRs：resampling 2、no-vectorization 19、register-pressure 37 | ✅ |
| 5 | 路径合规：8 项 trace 可解释扫描集；零/多命中仲裁合规（primary=resampling，supporting 2，无 companion/independent）；每个 blueprint leaf 均来自通过 gate 的 row；入口模式 A 按动态份额排序（函数内局部份额 ≈100%）；`th.v*` 未全局停扫（L0 build-缺-v finding 置顶后继续扫描） | ✅ |
| 6 | Phase 5 两侧锚定：消失侧对 Phase 3 引用行（`39f72/39f9a/39fe0/3a00c/3a07c/3a070/3a04e`）；出现侧标注 `patterns/rvv_resampling_kernels.md` §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测；无向用户追问 | ✅ |

修正记录：无