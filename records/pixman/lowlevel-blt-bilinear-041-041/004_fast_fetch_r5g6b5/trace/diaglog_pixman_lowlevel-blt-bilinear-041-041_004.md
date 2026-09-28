Functions under analysis: [fast_fetch_r5g6b5]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`fast_fetch_r5g6b5`，libpixman-1.so.0.46.5，1260 samples，event=cycles:u，percent: local period，含 hot loop 与全部路径）
- perf stat（可选 bound/context）：已提供（IPC=2.476；仅 cycles/instructions，无 cache/memory/branch counters，详见 Phase 1）
- workload/binary/DSO/source context：已提供（pixman `lowlevel-blt-bilinear-041-041` benchmark 的 r5g6b5 fetch scanline；binary=libpixman-1.so.0.46.5，elf64-littleriscv，含完整符号与反汇编）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：部分提供（benchmark 可执行文件 Tag_RISCV_arch 均为 `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`，无 `v`；热点 object libpixman-1.so.0.46.5 的 readelf -A 未提供，详见 Phase 1）
- hardware ISA（`/proc/cpuinfo` 或 hwprobe）：已提供（SpacemiT X100，`rv64imafdcvh_zicbom_..._zvbb_zvbc_zve64d_..._zvkned_...`，含 `v`（RVV 1.0），out-of-order）
- `vlenb`：已提供（vlenb=32 → VLEN=256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cycles`，percent type=local period，同一运行窗口；函数级 workload 贡献未知，详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcvh_...`（含 `v`，RVV 1.0 全套子集；SpacemiT X100，OoO 乱序执行） |
| Build ISA | `baseline_gap: build ISA (hot object)` —— 承载热点地址的 libpixman-1.so.0.46.5 未提供 readelf -A；benchmark 可执行文件 Tag_RISCV_arch 全部无 `v`；本函数 annotate 全函数 zero `v*` 且 zero `th.v*`（含 2b9c4 等全部行）→ 可确认执行代码不含 RVV。可选命令：`readelf -A /path/to/libpixman-1.so.0.46.5` |
| Vector flavor | annotate 内无 `v*` 也无 `th.v*`（全 scalar）→ 无 flavor mismatch，但存在 L0 判定：hardware 有 `v`、执行代码 zero `v*` |
| VLEN | 已提供：vlenb=32 → VLEN=256 bits |
| Bound type | IPC=2.476（较高，全程序）；无 cache/memory/branch counters → 仅能判定非显著 memory/latency-bound，`baseline_gap: bound type`。可选命令：`perf stat -e cycles,instructions,cache-misses,branch-misses -- <workload>` |
| Sampling semantics | event=`cycles`（可解释为时间）；percent type=local period；同一运行窗口；函数占整个 workload 的贡献未知 → 收益只能表述为「当前 sampled event 下的函数内局部样本份额」，不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（precise_ip/Exact-IP 未知）。可选命令：`perf evlist -v` / `perf report --header-only` |

L0 baseline gate：hardware 暴露 `v`（RVV 1.0），而本函数执行代码 zero `v*`/`th.v*`、benchmark 二进制 Tag_RISCV_arch 无 `v` → **最高优先级 baseline finding：当前 binary 未以 V 构建，scalar 主循环是"未向量化"的直接结果**。此 finding 冻结依赖 vector flavor 的路线判定，但不停止其它 row 扫描；无 `th.v*`。
Bound-type gate：缺 cache/memory counters → 本轮命中 row 的 performance-impact confidence 受此约束封顶。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`[fast_fetch_r5g6b5]`。
hot basic block / loop interval：
- 主循环 interval：`2b8bc–2b930`（每迭代处理 2 个 16-bit r5g6b5 像素：`lw` 一次载入 32-bit word → SWAR shift/mask/or 展开为两个 8-8-8 通道 → `sw` 两次写 32-bit a8r8g8b8）；anchor 行 `9.27 :  2b8c6:  and     a3,a3,a6`（同区间高位行 `6.81 :  2b8ce:  and     a4,a4,t5`、`6.28 :  2b8f8:  and     a4,a4,t1`、`5.80 :  2b8da:  slliw   a5,a5,0x3`、`4.77 :  2b930:  bne     a2,t6,2b8bc`）。
- 次区间：unaligned-entry 路径 `2b9ae–2b9fe`（src 非 4 字节对齐时每调用处理 1 像素，约占函数内样本 ~8%，其中 `5.62 :  2b9c4:  and     a3,a3,a6`）；单像素 tail 路径 `2b964–2b9a2`（~1%）。
- annotate 覆盖完整（prologue `2b858–2b878`、主循环、unaligned 路径、tail、epilogue `2b934–2b9ac` 均覆盖）。
Sampling IP precision 未知 → 单行占比只锚定所属 basic block / loop interval 的 interval-level 机制，不承担 instruction-latency / cycle-cost 单指令归因。

## Phase 3 — Pattern scan / 模式扫描：fast_fetch_r5g6b5

### Class selection trace（8 项）
1. rows-asm.md — exclude：当前代码是 compiler-generated scalar（libpixman C fast-path 编译产物；无 `.S`/DWARF provenance 信号）；无 missing-`.S` 的 policy/existence 四证（无 dispatch slot / scalar fallback / 目标缺失 / policy 要求独立 `.S` 证据）。
2. rows-operator-rvv.md — include：compiler-generated scalar loop，语义为 r5g6b5→a8r8g8b8 位域/channel packing（第一级必选）。
3. rows-string-memory.md — include：存在 src 4-byte 对齐分支（`2b87a: andi a3,a5,3` / `2b87e: bnez a3,2b9ae`）与固定宽度 logical access（每 32-bit word 两个 16-bit 元素）。
4. rows-vectorized-tuning.md — exclude：annotate 全函数 zero `v*`，无已向量化代码可调（该类要求已有 `v*` 且非手写 `.S`）。
5. rows-codegen.md — include：实现存在性/可达性判别（pixman fast-path fetch 经 implementation lookup 派发；需排除 kernel-selection 与代码形态 row）。
6. rows-offload.md — exclude：无矩阵引擎 / packed-SIMD（P）/ 权重重排 / 可移植层信号。
7. rows-crypto.md — exclude：无密码学原语（AES/SHA/CRC/GF 等）证据。
8. rows-runtime-os.md — exclude：用户态 pixman benchmark，无 timer/CSR/ISR/特权路径热点。

Classes scanned: rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md

### Local performance pattern scan: `fast_fetch_r5g6b5`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Layout and Channel Packing Kernels（primary） | main loop 全 scalar shift/mask/or 位域展开（anchor `9.27 : 2b8c6: and a3,a3,a6`），掩码常量 0xf800f8/0xfc00fc/0xff000/0xff000000 = r5g6b5→a8r8g8b8 位域合同；hardware 有 `v`、build 无 `v` | High | Medium | `patterns/rvv_layout_and_channel_packing_kernels.md` |
| No vectorization（supporting） | 主循环全 scalar、zero `v*`/`th.v*`（如 `2b8bc: lw`），hardware isa 含 `v`（Phase 1），build 无 `v` | —（supporting，自身 gate 成立） | — | `patterns/no-vectorization.md` |

**（a）逐字 evidence 引用（primary = RVV Layout and Channel Packing Kernels）：**
- `9.27 :  2b8c6:  and     a3,a3,a6` —— main loop basic block（interval `2b8bc–2b930`）内最高占比行；绿色通道掩码展开（`0xf800f8`）
- `6.81 :  2b8ce:  and     a4,a4,t5` —— 同一 interval；`0xfc00fc` 掩码
- `6.28 :  2b8f8:  and     a4,a4,t1` —— 同一 interval；`0xff000` 掩码
- `5.80 :  2b8da:  slliw   a5,a5,0x3` —— 同一 interval；5→8 位域左移
- `4.77 :  2b930:  bne     a2,t6,2b8bc` —— 同一 interval 的循环控制
- `0.63 :  2b8bc:  lw      a5,0(a1)` / `0.87 :  2b924:  sw      a4,-8(a2)` —— 窄访存形态（16-bit 元素 ×2/word）
- 常量物化在循环外一次完成：`2b898: lui a6,0xf80` / `2b8a4: addi a6,a6,248 # f800f8`、`2b89c: lui t5,0xfc0` / `2b8a8: addi t5,t5,252 # fc00fc`、`2b8a0: lui t4,0x10` / `2b8ac: addi t4,t4,-256 # ff00`、`2b8b8: lui a7,0xff000`（alpha 注入 0xff000000）
- 单像素 tail 同型位域展开：`2b970: slliw a5,a3,0x3` / `2b974: srliw a2,a3,0x2` / `2b97a: andi a1,a3,2016` 等
- 注意：Sampling IP precision 未知 → 上述行只锚定 main loop interval（2b8bc–2b930），不承担单指令 latency 归因。

**（b）互斥邻居排除（primary）：**
- `RVV Color Conversion Kernels`：行内互斥「只有 channel reorder/alpha/bit-field/panel packing → layout-packing row」——本函数机制是 shift/mask/or 位域展开 + alpha 注入（`2b922: or a4,a4,a7`），无颜色矩阵、无 YUV/chroma sampling → 不归 color-conversion。
- `RVV Precision Conversion Kernels`：行内互斥「低比特 nibble/layout decomposition 主导且无 accumulator → layout-packing row」——r5g6b5 是 5/6/5 低比特 packed 分解，非 FP/int rounding/widening 转换语义 → 不归 precision-conversion。
- `RVV Quantized Matmul and Requantization Kernels`：无 widening MAC / zero-point / Int32 accumulator / requantization 合同（全部是 and/or/sll 位操作）→ 不命中。
- `RVV Strided Memory Access for Layout Transforms`：src/dst 均 unit-stride 连续（`2b8be: addi a2,a2,8` / `2b8c0: addi a1,a1,4`），无固定 stride 变换 → 不命中。
- `RVV Indexed Gather for Table Lookup`：无 LUT / data-dependent 索引访问 → 不命中。
- `RVV Resampling Kernels`：行内互斥「像素格式/颜色转换 → layout/color rows」——本函数只做像素格式 fetch（r5g6b5→8888），分数坐标/插值权重/插值 FMA 在 bilinear iterator 其它 kernel，不在此函数 → 不命中。
- `No vectorization`（同组行）：行内互斥「operator semantic shape … → 各自更具体 row」——layout-packing semantic row 更具体认领该 scalar main loop → no-vectorization 降为 supporting（自身 gate：main loop 全 scalar + zero `v*` + hardware 有 `v` 成立）。

**（c）双 Confidence 推导式：**
- route: main-loop scalar + hardware `v`（cpuinfo）+ 位域语义合同（反汇编掩码 0xf800f8/0xfc00fc/0xff000/0xff000000）+ 行内互斥排除（color/precision/quantized/strided/gather/resampling 逐项排除）→ **High**
- impact: VLEN=256 已知 + 函数内局部 sample share 高（main loop interval ≈ 88–90%）+ 但缺 `bound type`（无 cache/mem counters）、采样语义仅 local（函数 workload 贡献未知）、`baseline_gap: build ISA (hot object)` → **Medium**

**Supporting evidence（no-vectorization）：**
- (a) `0.63 :  2b8bc:  lw      a5,0(a1)`（main loop interval 2b8bc–2b930 全行 zero `v*`/`th.v*`）；hardware isa line 含 `v`（Phase 1）→ supporting because: 同一机制（binary 未以 V 构建 / 编译）解释主循环为何全标量执行，不独立决定收益载体。两种 confidence 保持 `—`，不计入顶层命中数。

### 多命中仲裁小段
- 顶层 finding：**1 个**（primary = RVV Layout and Channel Packing Kernels）。
- no-vectorization 为 **supporting**（同一 hot loop、同一机制：scalar 位域展开 loop 未向量化；layout-packing semantic row 更具体认领，只写入 primary 的 Supporting evidence 行，不计入顶层 finding 数、不单独排序）。
- 证据按地址分账：main loop `2b8bc–2b930`（函数内局部样本份额 ≈ 88–90%，加总各行占比）→ primary 归属；unaligned-entry `2b9ae–2b9fe`（≈ 8%）与 tail `2b964–2b9a2`（≈ 1%）是同一函数内次要区间，不构成独立 finding（同型位域展开，同根因）。
- 机制层次：L1 vectorization/semantic dispatch（layout-packing、no-vectorization 均属 L1 层）；无 L0 之外的顶层命中（L0 baseline finding 已在 Phase 1 置顶）。
- 入口模式 A：primary 的 evidence sample share（函数内）≈ 88–90%；函数 workload 级贡献未知 → 收益上界仅限函数内局部份额。

### rows-string-memory.md 逐行排除（scan 集内，无命中）
- Scalar SWAR Checksum：无 checksum 语义 → 排除。
- Scalar Remainder Handling for Wide Kernels：行内互斥「main loop 本身 scalar/zero-`v*` → 对应 semantic/no-vectorization row」；tail `2b964–2b9a2` 仅 ~1% 且不主导 → 排除。
- Overlap-Expanding Back-Reference Copy：无 LZ match-copy → 排除。
- RVV Memory Copy/Fill：非 copy/fill 语义（是格式转换 kernel）→ 排除。
- RVV Sentinel Scan / Compare / Word-Wide First-Difference：无 scan/compare 语义 → 排除。
- Endianness-Specialized Memory Access Helpers：无 endian dispatch / byte-swap wrapper（固定小端 16-bit 载入）→ 排除。
- glibc RVV IFUNC/ABI Integration：非 glibc/libc target → 排除。
- Alignment Priming for Word-Sized Memory Operations：unaligned 分支存在（`2b87a/2b87e`、路径 `2b9ae–2b9fe`）但样本未「主导」在该 fallback（≈ 8% 函数内份额、每调用 1 像素）；根因是主循环本身未向量化，而非 alignment prologue 计算错误或 fault 主导；无 alignment-fault counter、无错误对齐掩码证据 → 排除。
- Wide Scalar Memory-Access Code Generation：主循环已直接 `lw` 32-bit word（`2b8bc`），无逐 byte 拼装 / tiny memcpy / 重复 field load → 排除。
- Adaptive-Width Unrolled Memory Zeroing：非 zeroing → 排除。

### rows-codegen.md 逐行排除（scan 集内，无命中）
- Kernel Selection and Runtime Specialization：无证据表明存在满足 r5g6b5 fetch 合同的专用 RVV kernel 且未被选中（annotate 内无 dispatch/lookup 信号；无仓库源码证据）→ 排除。
- Cache-Aware Blocking / Kernel Operation Fusion：非 tiled kernel；本函数为单 pass fetch，无独立 post-op pass → 排除。
- Compiler Workaround Retirement：无 `-O0/-fno-*` 历史 workaround 证据 → 排除。
- Forced Inlining：hot loop 内无 `call`/`jal` → 排除。
- Control-Flow Layout / Code Layout：`4.77% : 2b930: bne` 是主循环自身循环控制（scalar 每 2 像素迭代的分支），无 branch diamond/indirect call/constant-pool/frontend 主导证据 → 排除。
- Load/Store Addressing-Mode Fusion：`2b8be/2b8c0` 的 `addi` 是指针归纳递增，非只服务单条 memory op 的地址生成序列 → 排除。
- Register Pressure and Save/Restore：prologue 仅保存 s0–s4（`2b85a–2b878`），hot loop 内无 spill/reload → 排除。
- GP-Relative / ALU Constant：常量在循环外一次物化（`2b898–2b8b8`），无重复物化/rodata load → 排除。
- FP Semantic Lowering / Eliminate Precision Conversions：整数路径，无 FP → 不适用。
- Trap-Based Guard：无 bounds/trap 路径 → 排除。
- Resource-Aware Scheduling：IPC=2.476 健康，无 load-use/stall 证据；Sampling IP precision 未知 → 排除。
- Algebraic Simplification：SWAR 序列无冗余 NEG/MUL/SEXT/重复 loop-invariant 运算 → 排除。
- ISA Extension Substitution：hot loop 无 andn/orn/xnor/rotation/min-max 候选序列；且已使用 `zext.b`（2b8e8/2b8fe，Zbb）→ 排除。
- Native-Width / Zero-Based / Induction-Variable：循环已用指针递增 + end-pointer 比较（`2b930: bne a2,t6`），无 index scaling → 排除。
- Atomics / Spin-Wait / Redundant Sync / JIT / Runtime Dispatch / Tail Call：无同步、无 JIT、无 wrapper → 不适用/排除。

## Phase 4 — Root-cause blueprint / 根因蓝图：fast_fetch_r5g6b5

**命中 row：** Phase 3 通过 gate 的 `rows-operator-rvv.md` → `RVV Layout and Channel Packing Kernels`（primary）；supporting：`No vectorization`。

1. **Root cause**：`fast_fetch_r5g6b5` 的主循环（interval `2b8bc–2b930`）以每迭代 2 像素的标量 SWAR 位域展开实现 r5g6b5→a8r8g8b8：一次 `lw` 载入两个 16-bit 像素，随后 ~26 条 `srliw/and/or/slliw/slli/zext.b` 序列（掩码 0xf800f8 / 0xfc00fc / 0xff000 / 0xff000000）逐位展开 5-6-5 通道并注入 alpha，两次 `sw` 写出。依据 `patterns/rvv_layout_and_channel_packing_kernels.md` §Why this is slow：「位域 pack/unpack 每元素重复」（每 2 像素重复整套 shift/mask/OR）与「低比特数据在 shift/mask 完成前过早扩宽而放大 register group」；本函数的 scalar 展开正是这种固定 lane permutation 被展开为长标量序列的形态。叠加 L0 baseline finding：hardware 暴露 `v`（RVV 1.0、VLEN=256、OoO）而执行代码 zero `v*`（binary 未以 V 构建）——依据 `patterns/no-vectorization.md` §Why this is slow「vector unit 没有处理 hot main-loop 的并行元素」，标量主循环是未向量化的直接结果。

2. **The fix / 修复方式**（诊断蓝图，不实施）：
   - **修复对象**：libpixman 的 r5g6b5 fetch scanline 转换循环（本函数 main loop + unaligned-entry + tail）。
   - **Step 1（L0）**：按 no-vectorization §The fix 第 1 步，用硬件与工具链共同支持的精确实测 `-march`（含 `v`，以及已观察到依赖的 Zbb）重建 libpixman，使该 fetch loop 具备 RVV 向量化准入。修复前形态示意：`cc -O3 -march=rv64gc -c fetch.c`（无 RVV）；修复后基线示意：`verified_march=<含 v 的精确实测值>` + 同 O3 重建（示意，部署值须验证）。
   - **Step 2（向量化 fetch loop）**：按 layout-packing §8「先分解规则低比特 packed 数据再扩宽」与 §7 结构：
     - before（scalar, 2px/iter）：`lw a5,0(a1); srliw a3,a5,8; and a3,a3,0xf800f8; srliw a4,a5,3; and a4,a4,0xfc00fc; … or a4,a4,a7; sw ×2`（~26 条/2px）
     - after（RVV 形态示意，e16 主循环 + 输出边界扩宽，VLEN-agnostic）：
       ```
       vsetvli t0, remaining, e16, m1, ta, ma
       vle16.v  v8, (src)                    // 每 lane 一个 16-bit r5g6b5 像素
       vsrl.vx  v9, v8, 11                   // R 5-bit
       vand.vx  v9, v9, 31
       vsrl.vx  v10, v8, 5                   // G 6-bit
       vand.vx  v10, v10, 63
       vand.vx  v11, v8, 31                  // B 5-bit
       // 5→8: (x<<3)|(x>>2)；6→8: (x<<2)|(x>>4) —— 窄 e16 内完成，输出边界再扩宽到 e32
       … vsll/vsrl/vor …
       vse32.v  v12, (dst)                   // 或 vsseg4e8 输出 4 个 e8 通道
       ```
     - 适用前提：RVV intrinsic/codegen 可用；VLEN-agnostic（VLEN=256 时 e16/m1 = 16 lane/iter，m2 可到 32 lane）；fixed-VL main loop + runtime-VL tail 约定（依据 kernel-conventions.md §2 live-vector budget）。
   - **不可破坏的 correctness contract**：r5g6b5 位域合同（R=5bit、G=6bit、B=5bit，5→8 高位复制 `(x<<3)|(x>>2)`、6→8 高位复制 `(x<<2)|(x>>4)`，A=0xff 注入 0xff000000）；小端 16-bit 载入、小端 32-bit 输出；奇数宽度（本 testcase 41 像素）与 tail、src 非 4 字节对齐的 unaligned-entry 处理；producer-consumer 布局一致（fetch 输出被 bilinear iterator 消费）。
   - **限制/风险**：misaligned vector load 在目标核（SpacemiT X100）的代价未知（hwprobe `RISCV_HWPROBE_KEY_MISALIGNED_VECTOR_PERF` 未提供）→ 向量化后需核对 src 对齐与 unaligned 性能；e8 通道拆分的 register 压力需按 live-vector budget 验证；tail 不得硬编码固定 `vl`。
   - **修复后预期 Profile signals**：main loop 的 scalar `srliw/and/or/slliw/zext.b` 与 `bne` 循环控制 share 大幅下降；annotate 出现 `vsetvli`/`vle16`/`vse32`（或 `vsseg*`）指令；bytes/cycle 改善。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（v, RVV 1.0, OoO）；build ISA=bench 二进制无 `v`（hot object `baseline_gap: build ISA (hot object)`）；VLEN=256 bits；bound type=`baseline_gap: bound type`（IPC=2.476）。

4. **收益上界**：当前 sampled event（cycles）下函数内局部样本份额 ≈ 88–90%（main loop interval `2b8bc–2b930` 各行占比加总）；函数 workload 级贡献未知、采样语义仅 local → 不得称 workload 级 Amdahl 上界；不写估算数字之外的比例。

5. **三维路由判定**：
   - current source：compiler-generated scalar（libpixman C fast-path 编译产物；annotate 零 `.S` provenance 信号，全部 scalar 指令）。
   - implementation existence/reachability：无现存 RVV kernel / dispatch 选择证据（kernel-selection row 无证据命中；build 无 `v` 时不可能有 RVV 实现被加载）。
   - function-level policy：pixman 有 SIMD fast-path 传统，但本次无 RISC-V policy/existence 四证（dispatch slot / scalar fallback / 目标缺失 / policy 要求独立 `.S`）→ 不走 policy-backed missing `.S` 分支；fix 载体为「重建含 V + 普通 C/intrinsic 向量化」，不进入 `.S` 载体决策。

6. **Implementation-shape proof**：不适用 —— 非 policy-backed missing `.S` 分支（四证不全）。

7. **Related PRs 小节**：
   - `patterns/rvv_layout_and_channel_packing_kernels.md`：Related PRs：20 条 URL —— OpenBLAS c00afc86a6fd、ef7f54b35713、07d0e742c2ac、a8a00bbf4f91；OpenCV 8a36f119cee5、189f64726437；oneDNN ba07c4e658a1、79a1557d2119、#4548、aadd85dd6a2d；MNN #4079、73bfaa4、#4021、0a5ee52、#4067、2c2d7fa、#3813、6afcf99、#4426；vLLM #47538。
   - `patterns/no-vectorization.md`：Related PRs：19 条 URL —— OpenCV #22179、#22520、#23980、#24058、#24132、#24166、#24301、#24325、#27160、#27119、#27097、#27007、#26958、#26865、b902a8e792e1、2c16f3b7d2b2、e06502a254f7、a2d784b6f53a、83104bed3209。

## Phase 5 — Verification forecast / 验证预测：fast_fetch_r5g6b5

**Primary（RVV Layout and Channel Packing Kernels）：**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`9.27 :  2b8c6:  and     a3,a3,a6`、`6.81 :  2b8ce:  and     a4,a4,t5`、`6.28 :  2b8f8:  and     a4,a4,t1`、`5.80 :  2b8da:  slliw   a5,a5,0x3`、`4.77 :  2b930:  bne     a2,t6,2b8bc`、`0.63 :  2b8bc:  lw      a5,0(a1)` 的 scalar share 显著下降/消失。
- 应出现侧（锚定 `patterns/rvv_layout_and_channel_packing_kernels.md` §Verification：「逐通道 scalar load/store、shift/OR 和 loop-control share 下降，出现预期 segment/stride/reorder 指令，bytes/cycle 改善」）：同一函数重新 annotate 应出现 `vsetvli`/`vle16`/`vse32`（或 `vsseg*`）等 RVV 指令；与 `patterns/no-vectorization.md` §Verification「annotate 现在包含 RVV instructions」一致。
- 覆盖：41 像素（奇数）scanline、src 各对齐偏移（含 4-byte 未对齐 entry）、空/短输入、main-loop 整倍数与全部 tail length；r5g6b5 round-trip 与逐像素 packed 整数值比较（不能只靠显示图像）；真实 workload 代表性短/中/长输入 benchmark。
- supporting（no-vectorization）不单独验证，跟随 primary。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | 1/1 组；[fast_fetch_r5g6b5] |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 两个 L0 gate + bound-type gate，含 Sampling IP precision） | ✅ | 7 行 baseline；gap 标签：`baseline_gap: build ISA (hot object)`、`baseline_gap: bound type`、`baseline_gap: sampling IP precision`；`Sampling IP precision` 行已含 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`（4 include / 4 exclude + 理由）；`Classes scanned: rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md`；顶层 finding=1（layout-packing, primary），supporting=1（no-vectorization）；evidence 锚点=`9.27 : 2b8c6: and a3,a3,a6` 等 6 行；supporting 1 条；排除条数=operator 组 7 + string-memory 组 12 + codegen 组 17；推导式 2 条（route/impact） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`rvv_layout_and_channel_packing_kernels.md`（命中 row：RVV Layout and Channel Packing Kernels；引用「位域 pack/unpack 每元素重复」「先分解规则低比特 packed 数据再扩宽」；The fix 含 before/after、correctness、风险、Profile signals）、`no-vectorization.md`（supporting row；引用 §The fix 第 1 步）；Related PRs：layout-packing 20 条 URL、no-vectorization 19 条 URL |
| 5 | 路径合规：trace 可解释扫描集；零/多命中、关系与 evidence-mechanism layer 合规；每个 blueprint leaf 均来自已通过 gate 的 row；入口模式 A 按动态份额排序 | ✅ | 模式 A（profile-backed）；class 列表 = 上述 3 个 scanned 文件；primary/supporting 关系按 arbitration；L1 层归属；leaf=layout-packing（gate 通过）+ no-vectorization（supporting） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧=`2b8c6: and a3,a3,a6` / `2b8ce: and a4,a4,t5` / `2b8f8: and a4,a4,t1` / `2b8da: slliw a5,a5,0x3` / `2b930: bne a2,t6,2b8bc` / `2b8bc: lw a5,0(a1)`；出现侧=`patterns/rvv_layout_and_channel_packing_kernels.md` §Verification + `patterns/no-vectorization.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成/实施分支；无向用户追问（无 object-clarification 例外触发）；交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 |

修正记录：无