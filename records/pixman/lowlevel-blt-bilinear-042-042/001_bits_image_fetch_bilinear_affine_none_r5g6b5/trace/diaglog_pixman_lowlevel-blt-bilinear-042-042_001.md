Functions under analysis: [bits_image_fetch_bilinear_affine_none_r5g6b5]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-bits_image_fetch_bilinear_affine_none_r5g6b5-annotate.txt`，`cycles:u`，11595 samples，`percent: local period`；含 hot loop body 与 epilogue，覆盖完整）
- perf stat（可选 bound/context）：已提供（`21-pixman-benchmark-riscv-lowlevel-blt-bilinear-042-042.txt`：duration_time 132,836,960,760；cpu_cycle 291,970,434,246；instruction 829,960,673,029；IPC 2.842619；status PASSED）
- workload/binary/DSO/source context：已提供（annotate header 指明 DSO `libpixman-1.so.0.46.5`（`/workspace/build/pixman/`）；metadata 冻结 pixman@14735ced17e0053abbb925f9cf18c05ed9f52378、testcase `lowlevel-blt-bilinear-042-042`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：部分（冻结 metadata 提供同树 bench ELF 的 Tag_RISCV_arch = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`，无 `v`；承载热点的 DSO 未单独 readelf，详见 Phase 1）
- hardware ISA（`/proc/cpuinfo` / `riscv_hwprobe`）：已提供（冻结 hardware profile：SpacemiT X100（K3），OoO，`rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`）
- `vlenb`：已提供（vlenb=32 → VLEN=256 bits，RVV 1.0）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cycles:u`；percent type=local period；单窗口；该函数 workload 级贡献未知）
- Sampling IP precision：缺失（`precise_ip` 未知，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`v`（RVV 1.0）、`zba/zbb/zbc/zbs`、`zvbb/zvbc/zvk*`、`zvfh/zvfhmin`、`zfa/zfh` 等齐全（冻结 cpuinfo snapshot） |
| Build ISA | 部分：冻结 metadata 中同树 bench ELF `Tag_RISCV_arch` = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` — **无 `v`，亦无 `zba/zbb/zbc/zbs`**；热点 DSO `libpixman-1.so.0.46.5` 未单独 `readelf -A`（同 build tree 产物，标 `baseline_gap: build ISA` 的受限项） |
| Vector flavor | annotate 内无 `v*` 也无 `th.v*`（全 scalar），无 flavor mismatch 可判；硬件为 RVV 1.0 |
| VLEN | 已提供：vlenb=32 → VLEN=256 bits |
| Bound type | 仅 cycles/instructions counter：IPC=2.842619（829.96B instr / 291.97B cycles）→ compute/throughput-bound 倾向，非 memory-bound；无 cache/branch counter，精确 bound 分类受限（记录） |
| Sampling semantics | event=`cycles:u`（可解释为时间）；percent type=local period（**local**，非 global）；同一运行窗口；函数 workload 级贡献未知 → 百分比只代表函数内局部样本份额，不得外推 workload 级 Amdahl 上界 |
| Sampling IP precision | `precise_ip`/Exact-IP 未知 → `baseline_gap: sampling IP precision`；单条指令 sample 只锚定 basic block / loop interval，不承担单指令 latency/cost 根因 |

L0 baseline gate 1（hardware `v` vs build 无 `v`）：**成立** — hardware 暴露完整 RVV 1.0（VLEN=256），而冻结 build ISA（同树 bench ELF 属性，与热点 DSO 同一 build tree）不含 `v`。作为最高优先级 baseline finding 置顶报告，同时继续常规 row 扫描。L0 baseline gate 2（`th.v*` flavor）：不触发（无 `th.v*`，无 `v*`）。Bound-type gate：compute-bound（IPC 2.84）→ 本地 compute-vectorization fix 的 impact 不受 memory-bound 降级。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）。hot loop 边界与锚点：

- 像素主循环：`0x30098`–`0x300ec`（mask 检查、越界检查、增量仿射坐标步进、回边 `1.66 : 300ec: bne a5,t3,30098`），每像素路径落到 bilinear 体；
- bilinear 插值体（最高占比区间）：`0x30126`–`0x303be`，最高行原文 ` 9.99 :  30274: slli t1,t1,0x20`，次高 ` 5.97 :  303be: sw a2,0(a5)`、` 4.53 :  3039a: add a2,a2,t1`；该区间另含数据相关 tap load（` 1.52 :  3018e: lhu s2,0(s6)`、` 1.44 :  3029e: lhu a7,2(s6)`）与权重预计算（` 1.35 :  30184: andi t6,t6,254`）等；
- 该体（0x30126–0x303be）合计约 91.4% 函数局部样本；含循环控制/边界检查的完整像素循环约 99%（11595 samples，local period）；
- 冷启动（0x2ffbc–0x30097）约 0.6%，epilogue 与 stack_chk 路径 <0.1%。

annotate 覆盖完整（含 hot loop body、epilogue 与失败路径）→ 入口模式 A（profile_backed）。Sampling IP precision 未知 → 锚点统一为 interval 级，不做单指令 latency 归因。

## Phase 3 — Pattern scan / 模式扫描：bits_image_fetch_bilinear_affine_none_r5g6b5

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源为 compiler-generated scalar（annotate 无 `.S`/DWARF provenance，指令全为编译器风格 RV64IMC 标量码）；policy-backed missing `.S` 的 policy/existence 四证不齐（无 dispatch-slot、无独立 `.S` policy 证据）
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar 热点；语义为 bilinear affine fetch（分数坐标/权重/数据相关 tap load/插值乘法）→ resampling row；零 `v*` + hardware V + build 无 v → no-vectorization row
3. `rows-string-memory.md` — **exclude**：非 copy/fill/sentinel/compare/checksum 语义
4. `rows-vectorized-tuning.md` — **exclude**：annotate 零 `v*`（无可调的已向量化 RVV loop）
5. `rows-codegen.md` — **include**：compiler-generated；检查 register-pressure/save-restore（热循环内 stack reload）、constant materialization、IV strength reduction、kernel-selection 等
6. `rows-offload.md` — **exclude**：无矩阵引擎/权重重排/packed-SIMD/多线程分块信号
7. `rows-crypto.md` — **exclude**：非密码原语
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel/timer/CSR 热点

### Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | bilinear affine fetch：权重计算 ` 0.01 : 30180: subw t0,a2,t4` / ` 0.04 : 3025c: subw s7,a6,t6`（256−frac）；数据相关 tap gather ` 1.52 : 3018e: lhu s2,0(s6)`、` 1.44 : 3029e: lhu a7,2(s6)`（s6/s5 为按运行期 (x,y) 计算的源行指针）；插值乘法 30260–30296 与 30388–3039c 共 10 次 mul/mulw；边界检查 300a0–300da；区间 0x30126–0x303be ≈91.4% 函数局部样本 | High | Medium（局部份额已知、VLEN/bound 已知；采样为 local period → 无 workload 级上界） | `patterns/rvv_resampling_kernels.md` |
| No vectorization（supporting） | 同一像素循环全 scalar、全 annotate 零 `v*`/`th.v*`；` 1.66 : 300ec: bne a5,t3,30098`（scalar 循环控制）、` 5.97 : 303be: sw a2,0(a5)`（scalar store）；hardware 有 `v`、build 无 `v` | — | — | `patterns/no-vectorization.md` |

**（a）逐字 evidence 引用（resampling primary）**：

- ` 9.99 :  30274: slli t1,t1,0x20`（bilinear 体 0x30126–0x303be 内 32-bit 通道合并/权重乘法区间）
- ` 5.97 :  303be: sw a2,0(a5)`（32-bit 双像素 store，即 2×16-bit R5G6B5 输出）
- ` 1.52 :  3018e: lhu s2,0(s6)` 与 ` 1.44 :  3029e: lhu a7,2(s6)`（按运行期 (x,y) 计算地址的 16-bit tap 标量 gather；s6/s5 在 30136–3016e 由 `mulw a2,a2,t1/t0` + `slli a2,a2,0x2` + 行基址相加得到）
- ` 1.35 :  30184: andi t6,t6,254` 与 ` 0.04 : 3025c: subw s7,a6,t6`（定点权重 0–256 预计算区间 0x30170–0x30184 / 0x30258–0x3025c）
- ` 1.66 :  300ec: bne a5,t3,30098`（scalar 循环回边）
- ` 0.06 : 30260: mulw s11,t0,s7`、` 1.52 : 30288: mul a2,a2,s7`、` 1.18 : 30390: mul a7,a7,t4`（插值乘法族 30260–30296 + 30388–3039c，共 10 次 mul/mulw）

**（b）互斥邻居排除**：

- vs 布局/通道 packing row（R5G6B5 位域展开/回装确实存在，3018e–30214 与 3029e–30342 的 `slliw/srliw/andi/or` 序列）：该循环同时携带插值合同（权重 `subw t0,a2,t4`、10 次插值乘法、按坐标选 tap），符合 gather row 行内互斥 "带 interpolation 坐标/权重合同 → resampling row" 与 color row 互斥 "resize/interpolation → resampling row"；纯 layout packing 无此算术。→ 排除 layout/color rows
- vs spatial-convolution row：tap 地址来自运行期仿射坐标（`lw t5,164(s1)`/`mulw`+行基址），非固定 stencil 偏移。→ 排除
- vs indexed-gather row：有插值权重/乘法合同，非"无插值语义的任意 LUT gather"。→ 排除
- vs elementwise row：非 unit-stride lane-independent，tap 数据相关。→ 排除
- vs no-vectorization：本行（resampling）为更具体语义 row，按 no-vectorization row 行内互斥 "operator semantic shape → 各自更具体 row" 认领语义贡献；no-vectorization 降为 supporting（仅解释零 `v*` 执行载体，不解释贡献机制）。→ supporting

**（c）双 Confidence 推导式**：

- `route: compiler-generated scalar provenance（直接）+ bilinear/affine 坐标-权重-indexed-load-插值合同（逐字反汇编直接证明）+ 行内互斥判别成立（插值合同 vs layout/gather/conv）→ High`
- `impact: 函数局部 sample share ≈91.4%（0x30126–0x303be）/≈99%（完整像素循环）已提供；VLEN=256 已提供；bound type=compute-bound（IPC 2.842619）已提供；但采样 percent type=local period + 函数 workload 贡献未知 → 收益仅限局部份额、无 Amdahl 上界；且 build 无 `v`（L0）要求先精确重建 → Medium`

**Supporting evidence（no-vectorization）**：`supporting because: 同一 main loop 全 scalar、zero `v*`；只解释缺少向量执行载体（build 无 `v` + 无 RVV kernel），不决定贡献载体（语义贡献由 resampling 认领）。`

**rows-codegen.md 逐 row 排除表**（有信号者给判别性观察，其余整组一句排除）：

- Kernel selection/runtime specialization：exclude — 无 dispatch/fallback/kernel-lookup 信号
- Cache-aware blocking：exclude — 非 tiled kernel，无 tile-residency/拐点证据
- Kernel operation fusion：exclude — 本函数已是 fetch+expand+interp+pack 单遍融合，无独立第二遍可融合
- Compiler workaround retirement：exclude — 无 `-O0/-fno-*` workaround 证据
- Forced inlining：exclude — 热循环内无调用（`pixman_transform_point_3d@plt` 在 3001a 每 scanline 一次，非热区）
- Control-flow layout：exclude — 热区为直线代码，无 branch diamond/indirect-call 主导
- Code layout/constant-pool：exclude — 常量经 lui/栈重载，无 constant-pool/trampoline/i-cache 证据
- Load/store addressing-mode fusion：exclude — lhu/lw 均为 base+imm 已折叠形态，地址生成源于 gather 语义
- **Register pressure/save-restore：exclude** — 热循环内确有 stack reload（` 0.29 : 30214: ld a2,-152(s0)`、` 1.62 : 302fe: ld s9,-192(s0)`、` 1.40 : 3034e: ld t5,-152(s0)`、` 1.40 : 30352: and s2,s2,s7` 上游 `ld s11,-184(s0)`（30356 0.02）、` 1.48 : 303a0: ld a0,-160(s0)`、` 1.61 : 303ac: ld a0,-168(s0)`，合计 ≈7.8%），但 row gate 要求 spill/reload **主导**，此处不成立（top-10 指令位元操作/乘法合计 >40%）；记录为 scalar codegen 症状，向量化后自然消失，不单独命中
- GP-relative / FP-semantic / trap-guard / resource-scheduling / algebraic / float-conv / atomic / spin-wait / native-width / ext-elim / runtime-dispatch / tail-call / zero-based / redundant-sync / JIT：exclude — 无对应信号
- ISA substitution（Zbb/Zba）：exclude — 当前 build ISA 不含 zba/zbb/zbc/zbs（route gate "build profile 应具备" 不成立），仅硬件具备；且非主导机制
- IV strength reduction：exclude — 已应用（300e2/300e6 增量仿射 `addw a3,a3,s4`、指针递增 `addi a5,a5,4`、end-pointer 比较 `bne a5,t3`）

**多命中仲裁小段**：resampling（L1，primary）与 no-vectorization（L1，supporting）解释同一热循环、同一机制族（标量流水未向量化）；no-vectorization 只认领"零 `v*`"载体事实，按 rows-operator-rvv.md row 30 行内互斥降为 supporting，不计顶层命中数。L0 build-mismatch 为 baseline finding（先于一切 row 报告）。顶层 finding：1（resampling，primary）。未列名机制兜底：无。入口条件 A → 该 finding 的 evidence sample share：0x30126–0x303be ≈91.4% 函数局部；含循环控制的完整像素循环 ≈99%。

## Phase 4 — Root-cause blueprint / 根因蓝图：bits_image_fetch_bilinear_affine_none_r5g6b5

**L0 baseline finding（置顶）**：hardware 暴露完整 RVV 1.0（`v`、zvbb/zvbc/zvk*、VLEN=256），而冻结 build ISA（同树 bench ELF `Tag_RISCV_arch` 无 `v`；热点 DSO 同 tree）按无 V 标量配置构建 → 当前 binary 不可能发出任何 `v*` 指令。修复对象：以硬件+工具链共同支持的**精确** `-march`（含 `v` 及所需 subset）clean rebuild（依据 `patterns/no-vectorization.md` §The fix 步骤 1：`verified_march=rv64gcv; cc -O3 -march="$verified_march"` 示意）。

**1. Root cause**（依据 `patterns/rvv_resampling_kernels.md` §Why this is slow 与 §The fix 7）：像素主循环（0x30126–0x303be）对每个输出像素完整执行标量 bilinear fetch 流水：仿射分数坐标增量步进（300e2/300e6）、定点权重 256−frac 预计算（30170–30184、30258–3025c）、按运行期坐标标量 gather 四个 16-bit tap（`lhu s2,0(s6)` / `lhu a6,0(s5)` / `lhu a7,2(s6)` / `lhu t5,2(s5)`）、两路 R5G6B5→RGB888 位域展开（3018e–30214、3029e–302fa）、10 次 32-bit 插值乘法（30260–30296、30388–3039c）、RGB888→R5G6B5 位域回装（30218–30270、30302–303be）、32-bit 双像素 store（`sw a2,0(a5)`）。每 2 个 16-bit 输出像素约 60+ 条标量指令。pattern 关键句（§Why this is slow）："重采样的主要根因是逐输出重复坐标计算、数据相关地址导致 scalar gather、bilinear/cubic 权重与边界处理割裂成多遍，以及坐标/权重精度阻止安全向量化"。在 VLEN=256、OoO、IPC≈2.84 的 X100 上，这是纯 issue/arithmetic 吞吐负担，非访存带宽主导。

**2. The fix / 修复方式**（与 `patterns/rvv_resampling_kernels.md` §The fix 7 "Vectorize bilinear resampling with indexed loads" 一致，具体化到本函数）：

- 前置（L0）：用精确 `-march`（含 `v`；建议核对 `zvbb` 等 optional extension 后按需启用）clean rebuild pixman；
- 主修复：将每像素标量流水改写为 lane-wise RVV bilinear fetch —— 预计算 tap byte-offset 向量与权重向量（对应 pattern §6 的 x0ByteOffsets/x1ByteOffsets/xWeights 预计算数组），`vluxei*` 批量 gather 4 taps（pattern §7 形态：`__riscv_vluxei32_v_f32m1(sourceTopRow, x0Offsets, vl)` 的整数/16-bit 变体），展开为 8-bit 通道后以 256 定点权重做 widening 插值（`vwmacc` 风格），再 narrowing + pack 回 16-bit，最后合并为 32-bit 双像素 store 或 `vsseg` store；对 16-bit R5G6B5 源，可用 `vle16` 整行连续段 + 每像素 offset 的 indexed 变体，注意 interleaved 双像素输出不能直接套单平面示例（pattern §7 明确："多通道 interleaved 图像不能直接套用单平面示例"）；
- 修复前/后伪代码：

```text
// Before（每像素标量，当前 annotate 0x30126–0x303be 形态）
for (i in 0..w) { x=...; y=...;                    // affine 增量
  s6 = row_top + y*stride + x*2; s5 = row_bot + ...;
  t0 = 256 - xfrac; s7 = 256 - yfrac;
  s2 = lhu(s6); a6 = lhu(s5); a7 = lhu(s6+2); t5 = lhu(s5+2);
  t1 = expand565(s2); a2_ = expand565(a6); a0_ = expand565(a7); t6_ = expand565(t5);
  // 10× mul/mulw + or/add 位域回装
  sw(pack16(interp), dst+i*4);

// After（RVV lane-wise 形态；示意，非固定 recipe）
vl = vsetvl(e16, ...);                              // lane = 输出像素
x0 = vle16(x0ByteOffsets+i); x1 = vle16(x1ByteOffsets+i); wx = vle16(xWeights+i);
topL = vluxei16(srcTopRow, x0, vl); topR = vluxei16(srcTopRow, x1, vl);
botL = vluxei16(srcBotRow, x0, vl); botR = vluxei16(srcBotRow, x1, vl);
top  = vwmacc(topL,  topR - topL,  wx);             // 8-bit 展开 + widening
bot  = vwmacc(botL,  botR - botL,  wx);
out  = vwmacc(top,   bot - top,    wy);             // narrow + pack565
vsseg/32-bit-pack-store(out, dst + 2*i);
```

- 适用前提：pixman 以支持 RVV 的 toolchain 重建、bilinear fetch 有稳定 lane 语义；坐标/权重合同（`pixman_fixed` 16.16 定点、0–256 权重）保持与 reference 一致；
- correctness contract（不可破坏）：border 语义（0x300a0–0x300da 的越界检查 + zero 像素 store）、mask 语义（0x30098–0x3009e）、affine 每 scanline 步进与 clamp；index element width/EMUL 不截断大图 byte offset（pattern §7："超过 32-bit byte offset 范围时应选择适合的 index width"）；定点插值顺序与 reference 位精确（pattern §7 末：FMA 改变舍入，生产实现必须保持 reference 定义的 arithmetic-order 与 rounding 合同）；tail 与奇数宽、最后一列 tap；
- 限制/风险：indexed load 在目标核的实际吞吐需 benchmark（pattern §Verification："indexed load 在目标核更慢" 是失败判据之一）；短 scanline 可能退化（§6："短图像或单次调用应比较预计算和即时计算"）；index/EMUL 或 live-set 过大可能 spill（kernel-conventions §2：`LMUL * peak_live_vectors <= 32`，widening 时按 destination LMUL 预算）；
- 修复后预期 Profile signals：0x30126–0x303be 区间标量 `slli/srliw/andi/or/mul` 序列显著缩小或消失，出现 `vsetvli`/`vluxei*`/`vwmacc*` 等；`sw a2,0(a5)` 双像素标量 store 被向量 store 取代。

**3. Baseline facts 回填**：hardware ISA = `rv64imafdcvh_...zvbb_zvbc...`（RVV 1.0）；build ISA = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`（无 `v`，受限项见 Phase 1）；VLEN = 256（vlenb=32）；bound type = compute/throughput（IPC 2.842619，无 cache counter）。

**4. 收益上界**：当前 sampled event（`cycles:u`，local period）下的**局部样本份额** —— 直接修复对象 bilinear 体 0x30126–0x303be ≈91.4%（含循环控制/边界检查的完整像素循环 ≈99%，11595 samples）。采样语义四条不全（percent type=local + workload 贡献未知）→ 不称 workload 级 Amdahl 上界。

**5. 三维路由判定**：

- `current source`：compiler-generated scalar C（annotate 全标量 RV64IMC 码，无 `.S` provenance）→ resampling semantic row 适用；
- `implementation existence/reachability`：pixman 该函数仅 generic C fetch 路径可达；无 RISC-V vector kernel 存在/可达证据（build 无 `v` 即排除任何 `v*` 实现），亦无 kernel-selection/dispatch 信号 → 不命中 kernel-selection row；
- `function-level policy`：pixman 无 RISC-V assembly-default 独立 `.S` policy 证据；policy/existence 四证不齐 → **不进入 missing `.S` 分支**，修复载体由普通代码/intrinsic 承担。

**6. Implementation-shape proof**：不适用 —— 非 policy-backed missing `.S` 分支（四证不齐：无 dispatch slot、无目标 `.S` 缺失证明、无 assembly-default policy、hardware 向量 ISA gate 虽成立但不改变载体结论）。

**7. Related PRs 小节**：

- `patterns/rvv_resampling_kernels.md`：Related PRs：2 条 URL（MNN #4053 https://github.com/alibaba/MNN/pull/4053、MNN f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d https://github.com/alibaba/MNN/commit/f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d）
- `patterns/no-vectorization.md`：Related PRs：20 条 URL（OpenCV #22179 https://github.com/opencv/opencv/pull/22179、#22520 https://github.com/opencv/opencv/pull/22520、#23980 https://github.com/opencv/opencv/pull/23980、#24058 https://github.com/opencv/opencv/pull/24058、#24132 https://github.com/opencv/opencv/pull/24132、#24166 https://github.com/opencv/opencv/pull/24166、#24301 https://github.com/opencv/opencv/pull/24301、#24325 https://github.com/opencv/opencv/pull/24325、#27160 https://github.com/opencv/opencv/pull/27160、#27119 https://github.com/opencv/opencv/pull/27119、#27097 https://github.com/opencv/opencv/pull/27097、#27007 https://github.com/opencv/opencv/pull/27007、#26958 https://github.com/opencv/opencv/pull/26958、#26865 https://github.com/opencv/opencv/pull/26865、commit b902a8e792e1702b40f19dbd48dff0bfdca8b36d https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d、2c16f3b7d2b28f6cac444046b8f95b40d9266a6a https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a、e06502a254f79f9d3184de2803d087c6914b7706 https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706、a2d784b6f53aa1fdfde21ab8e3787a93b59af24f https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f、83104bed32093ff0c5c935e8920c72b6b74ae07a https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a）—— 未联网补齐，仅按 pattern 本地表）

## Phase 5 — Verification forecast / 验证预测：bits_image_fetch_bilinear_affine_none_r5g6b5

- 消失/缩小侧（锚定 Phase 3(a) 引用行）：重建并重采 annotate 后，`9.99 : 30274: slli t1,t1,0x20`、`5.97 : 303be: sw a2,0(a5)`、`1.52 : 3018e: lhu s2,0(s6)`、`1.44 : 3029e: lhu a7,2(s6)`、`1.66 : 300ec: bne a5,t3,30098`、`1.62 : 302fe: ld s9,-192(s0)`（栈重载）应显著缩小或消失（scalar 展开/回装、逐像素 gather、标量循环控制与 spill reload 不再主导）；
- 出现侧（`patterns/rvv_resampling_kernels.md` §Verification）：出现预期 indexed vector loads（`vluxei*`），"逐 lane coordinate/index arithmetic 与 scalar loads 下降，cycles/output 改善"；`patterns/no-vectorization.md` §Verification：annotate 出现 `vsetvli`/`vle*`/`vse*`/`vwmacc*` 等 RVV 指令，scalar 指令不再主导 hot loop；
- 正确性验证：覆盖 border clamp（0x300a0–0x300da 路径）、最后一行/列、奇数宽度与 tail、mask 非零路径；定点插值结果与 reference 逐像素位精确比对；`vluxei` index 为 byte offset 且不截断大图；
- benchmark：同 benchmark 输入重跑 `lowlevel-blt-bilinear-042-042`，对比 cycles/输出像素；短/中/长 scanline 均测（短行防退化）；
- 升级 profile-backed 数据的最低采集项：`perf record` 使用 `--percent-type=global-period`（或同窗口全局统计）以获得该函数 workload 级占比，并以 `perf evlist -v` 核对 `precise_ip` 消除 `baseline_gap: sampling IP precision`。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现（载荷：`1/1 组；bits_image_fetch_bilinear_affine_none_r5g6b5`） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 齐；gap 标签：`baseline_gap: build ISA`（DSO 未单独 readelf）、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（resampling primary）+ supporting 1（no-vectorization）；evidence 锚点 `9.99 : 30274: slli t1,t1,0x20`/`5.97 : 303be: sw a2,0(a5)`/`1.52 : 3018e: lhu s2,0(s6)`/`1.44 : 3029e: lhu a7,2(s6)`/`1.35 : 30184: andi t6,t6,254`/`1.66 : 300ec: bne a5,t3,30098`/`0.06 : 30260: mulw s11,t0,s7`/`1.52 : 30288: mul a2,a2,s7`；supporting 1（no-vectorization）；排除条数 26（rows-operator-rvv 19 行 + rows-codegen 26 行均给出理由，其中 register-pressure/ISA-substitution/IV-strength 为逐条判别）；推导式 2 条（route/impact 各 1）） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern 文件 `patterns/rvv_resampling_kernels.md`、`patterns/no-vectorization.md`；命中 row：resampling（`### RVV Resampling Kernels`）、no-vectorization（supporting，`### No vectorization...`）；引用短语首词 "重采样的主要根因"/"Vectorize bilinear resampling with indexed loads"/"多通道 interleaved 图像不能直接套用单平面示例"/"超过 32-bit byte offset 范围时应选择适合的 index width"/"verified_march=rv64gcv"；`The fix` 含 before/after、correctness（border/mask/定点舍入）、风险（indexed-load 慢/短行退化/spill）与 Profile 信号锚点（`9.99 : 30274`/`5.97 : 303be` 消失、`vsetvli`/`vluxei*` 出现）；非 missing `.S` 分支 → 无 shape 字段要求；Related PRs：rvv_resampling_kernels.md 2 条 URL、no-vectorization.md 20 条 URL） | ✅ |
| 5 | 路径合规（载荷：模式 A + 路径=compiler-generated scalar→resampling primary + no-vectorization supporting；class 列表 `rows-operator-rvv.md, rows-codegen.md`；按动态份额排序仅用于入口模式 A） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `9.99 : 30274: slli t1,t1,0x20`、`5.97 : 303be: sw a2,0(a5)`、`1.52 : 3018e: lhu s2,0(s6)`、`1.44 : 3029e: lhu a7,2(s6)`、`1.66 : 300ec: bne a5,t3,30098`、`1.62 : 302fe: ld s9,-192(s0)`；出现侧 `patterns/rvv_resampling_kernels.md` §Verification + `patterns/no-vectorization.md` §Verification） | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 | ✅ |

修正记录：无