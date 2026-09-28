Functions under analysis: [bits_image_fetch_nearest_affine_none_a8r8g8b8]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 perf annotate：已提供（`libpixman-1.so.0.46.5`，`cycles:u`，2048 samples，`percent: local period`，含 hot loop 0x33c7e–0x33cd0，共 111 行）
- perf stat（可选 bound/context）：已提供（`lowlevel-blt-nearest-022-022`：duration 40.0577s，cpu_cycle 88.055G，instruction 140.117G，IPC 1.591242，status PASSED）
- workload/binary/DSO/source context：已提供（pixman commit `14735ced17e0053abbb925f9cf18c05ed9f52378`，benchmark `21-pixman-benchmark-riscv-lowlevel-blt-nearest-022-022`，热点位于 `libpixman-1.so` 内）
- readelf -A（承载热点地址的 object 的 `Tag_RISCV_arch`）：已提供（metadata binaries 的 Attribute Section：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata hardware snapshot：SpacemiT X100，`rv64imafdcvh_...`，含 `v`、zba/zbb/zbc/zbs、zicond、zfa、zfh、zvbb/zvbc/zvk* 等；指令调度 OoO）
- `vlenb`：已提供（hardware snapshot：vlenb=32 bytes → VLEN=256 bits，RVV 1.0）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=`cycles`，`percent: local period`，2048 samples 单次运行窗口；函数级 workload 贡献未知，详见 Phase 1）
- Sampling IP precision（precise_ip / Exact-IP / PMU skid）：缺失（annotate header 未含 `precise_ip` 信息，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh` + Zb*（zba/zbb/zbc/zbs）+ zicond + zfa + zfh + zvbb/zvbc/zvkg/zvkned/zvknha/zvksed/zvksh + RVV 1.0；硬件暴露标准 `v` |
| Build ISA | `Tag_RISCV_arch: rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` —— **无 `v`，也无 zba/zbb/zbc/zbs、zicond、zfa、zfh** |
| Vector flavor | annotate 全 scalar，hot loop 内零 `v*` 零 `th.v*` → 无 flavor mismatch（`th.v*` gate 不适用） |
| VLEN | vlenb=32 bytes → **256 bits**（RVV 1.0） |
| Bound type | IPC=1.591242（全程）；无 cache/memory/branch counter → `baseline_gap: bound type`（cache/memory counter 缺失；可选命令 `perf stat -e cycles,instructions,cache-misses,branches -- ./lowlevel-blt-bench ...`）；IPC 指向混合 compute/latency，非极端 memory-bound |
| Sampling semantics | event=`cycles`（时间可解释）；percent type=`local period`（非 global-period）；同一运行窗口；函数 workload 贡献未知 → 只能表述「当前 sampled event 下的函数内局部样本份额」，**不得称 workload 级 Amdahl 上界** |
| Sampling IP precision | `precise_ip` / Exact-IP 未知 → `baseline_gap: sampling IP precision`；最高行只锚定 basic block / loop interval，不承担单指令 latency 归因 |

**L0 baseline gate 判定：**
- hardware 有 `v` 而 build 无 `v` → **最高优先级 baseline finding**：libpixman-1.so 按不含 V（也不含 Zb*）的 ISA 构建，当前对象内任何 RVV 指令都不可能存在；该 finding 置顶但不停止全局扫描。
- `th.v*` 不存在 → flavor gate 不适用。
- Bound-type gate：`baseline_gap: bound type`（cache/memory counter 缺失）→ 本轮所有命中的 performance-impact confidence 因此封顶（不得为 High）。

## Phase 2 — Scope / 分析边界

- 函数清单（与承诺声明一致）：`bits_image_fetch_nearest_affine_none_a8r8g8b8`（pixman fast-path fetch 函数，affine 变换 + nearest 采样，a8r8g8b8 32-bit 像素直拷）。
- Hot loop interval：主循环 `0x33c7e – 0x33cd0`（含 bounds-check 分支、gather load、store、坐标增量与循环控制；`0x33d00` 分支为 mask 路径 cold 分支，占比 0.00%）。
- Trace anchor（最高行原文）：`46.02 :  33cc2:  sw      a1,0(a5)`（unit-stride store 到目标缓冲，主循环基本块末尾）。
- 其它主要行（同区间）：`9.14 :  33ca6:  srliw   a6,a7,0x1f`、`8.56 :  33cb2:  ld      t1,168(s1)`、`8.45 :  33cc0:  lw      a1,0(a1)`、`7.77 :  33c7e:  beqz    s3,33c86`、`7.62 :  33c8e:  bge     a1,a6,33cfa`、`6.84 :  33c9a:  lw      a6,160(s1)`。
- Sampling IP precision 未确认 → `46.02%` 单行仅锚定该 basic block / loop interval（interval 级机制），不作单指令 latency/cost 归因。
- annotate 覆盖完整（prologue 0x33bf4–0x33c54 冷区间、主循环、epilogue 0x33cd4–0x33d12 均在场）→ 入口条件 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：bits_image_fetch_nearest_affine_none_a8r8g8b8

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源是 compiler-generated（annotate 为标准 RISC-V 标量指令序列、无 `.S` source/DWARF/object-mapping 证据、libpixman-1.so 普通编译产物）；亦无 policy-backed missing `.S` 四证（无源码仓库访问权，`source_context_gap`）。
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar loop，语义为 affine nearest 采样 = 数据相关索引 gather + unit-stride store。
3. `rows-string-memory.md` — **exclude**：非 copy/fill/sentinel/compare/checksum/back-reference 语义（函数为逐像素采样 fetch，非块复制）。
4. `rows-vectorized-tuning.md` — **exclude**：annotate 全 scalar，hot loop 零 `v*`，无 RVV 配置/寄存器/LMUL 修正对象。
5. `rows-codegen.md` — **include**（保守补选，逐行评估后全排除）：compiler-generated 代码形态检查（寻址、寄存器压力、强度削减、ISA substitution）。
6. `rows-offload.md` — **exclude**：无矩阵引擎/packed-SIMD/权重重排/多线程分块信号。
7. `rows-crypto.md` — **exclude**：非密码学原语（AES/SHA/SM/GHASH/CRC/GF）。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel 侧 timer/ISR/CSR/权限域热点。

**Classes scanned: `rows-operator-rvv.md`（全文）、`rows-codegen.md`（全文，逐行评估）**

### Local performance pattern scan: `bits_image_fetch_nearest_affine_none_a8r8g8b8`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Indexed Gather for Table Lookup and Data-Dependent Access（primary） | affine 变换定点坐标 → 逐 lane 数据相关标量 load（`33cc0: lw a1,0(a1)` 8.45%）+ 索引地址算术（`33cb6: mulw` / `33cba: add` / `33cbc: slli`）+ unit-stride store（`33cc2: sw` 46.02%）；迭代间无跨元素依赖 | High | Medium | `patterns/rvv_gather_indexed_memory_access.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`、hardware 有 `v`（RVV 1.0）、build 无 `v`；只解释缺少向量执行载体，不决定贡献载体 | — | — | `patterns/no-vectorization.md` |

### 三件套 — primary：RVV Indexed Gather

**(a) 逐字 evidence 引用**（主循环 basic block `0x33c7e–0x33cd0`）：
```
8.45 :   33cc0:  lw      a1,0(a1)      ← 数据相关索引 load（gather 源）
0.15 :   33cb6:  mulw    a1,a6,a1      ← y × stride 索引算术
0.00 :   33cba:  add     a1,a1,a7      ← + x
0.19 :   33cbc:  slli    a1,a1,0x2     ← × sizeof(pixel)=4 → byte offset
8.56 :   33cb2:  ld      t1,168(s1)    ← 源图像 base 指针加载
46.02 :  33cc2:  sw      a1,0(a5)      ← unit-stride store 到目标
```
形态即 `dst[i] = src[affine(i)]`：源地址由运行期 affine 变换计算得出、逐像素不连续，退回逐 lane 标量加载（与 pattern §Profile signals「逐 lane 的标量 load、索引地址计算（mul/add 生成 element 地址）」一致）。

**(b) 互斥邻居排除**：
- **resampling row**（`rvv_resampling_kernels.md`）：nearest 采样无 interpolation 坐标/权重合同、无插值 FMA（`33cc0` 是直接 load 而非加权合成）→ 排除。
- **strided-layout row**（`rvv_strided_layout_transform_kernels.md`）：`33cb6 mulw/33cba add` 生成的地址随 affine 矩阵逐像素变化，非固定 element/byte stride → 排除。
- **layout-packing / color-conversion rows**（`rvv_layout_and_channel_packing_kernels.md` / `rvv_color_conversion_kernels.md`）：`33cc2 sw` 直接复制 32-bit a8r8g8b8 像素，无 interleave、shift/mask/OR、颜色矩阵或 chroma 采样 → 排除。
- **elementwise row**（`rvv_contiguous_elementwise_arithmetic_kernels.md`）：`33cc0` 的 load 地址逐 lane 数据相关、非 unit-stride → 排除（该行行内互斥明确写「内存 indexed gather → layout/gather rows」）。

**(c) 双 Confidence 推导式**：
- `route: compiler-generated provenance（annotate 标准标量指令）+ gather 语义合同（数据相关索引 + 迭代间无跨元素依赖，符合 dst[i]=table[idx[i]] 形态）+ 四项互斥排除（resampling/strided/layout/color/elementwise）→ High`；
- `impact: 热区间局部样本份额高（主循环区间约 97.3% 局部；gather 相关行约 63.4% 局部）+ VLEN=256 已知，但缺 bound type（cache/memory counter）、采样语义为 local period 且函数级 workload 贡献未知、build 无 v 需先重建 → Medium`。

### 三件套 — supporting：No vectorization

**(a) 逐字 evidence 引用**：`46.02 :  33cc2:  sw      a1,0(a5)`（主循环全区间零 `v*`；`ld t1,168(s1)` / `lw a1,0(a1)` / `sw a1,0(a5)` 全为标量访存）。
**(b) 互斥排除**：`th.v*` 不存在（flavor gate 不适用）；gather semantic row 已认领具体语义 → 本行只作 supporting，不另立顶层 finding。
**(c) 双 Confidence**：`route: 不单独计（supporting）`；`impact: 不单独计（supporting）`。

### 多命中仲裁小段

- **primary = RVV Indexed Gather**（L1 vectorization/semantic dispatch 层）：唯一因果链顶层 root-cause row —— 数据相关索引使循环无法 unit-stride、逐 lane 标量 gather 主导样本。
- **supporting = No vectorization**（同一机制「main loop 无向量执行」）：自身 row gate 成立（全 scalar + hardware 有 v + zero v*），但行内互斥要求更具体 semantic row 认领；故仅写入 primary 的 Supporting evidence 行，不计入顶层 finding 数，不单独排序。
- **rows-codegen.md 逐行排除结论**（无独立 codegen 形态信号）：Kernel Selection（无 dispatch 证据，`source_context_gap`）；Cache-Aware Blocking（无 tile/residency 证据）；Kernel Operation Fusion（单 pass，无中间结果往返）；Compiler Workaround（无 build log 证据）；Forced Inlining（`33c54 jal pixman_transform_point_3d@plt` 在主循环外冷调用，非热 helper）；Control-Flow Layout（`33c8e/33ca2 bge`、`33c96/33caa bnez` 为语义 bounds check，非布局缺陷）；Code Layout/Constant-Pool（无 frontend/i-cache 证据）；Load/Store Addressing-Mode Fusion（`33cb6/33cba/33cbc` 地址序列为 gather 索引语义必需，不可折叠）；Register Pressure（热循环内零栈访问/spill）；GP-Relative / ALU-Constant / FP-Lowering / Trap-Guard（无对应观察）；ISA Substitution（build 无 Zb*/Zicond/Zfa，route gate 不成立）；Native-Width / Redundant-Extension（`33c8a sraiw`、`33c92 srliw` 为定点坐标语义必要）；Runtime-ISA-Dispatch（build 无 v，无 dispatch 对象）；Loop Induction Variable Strength Reduction（主循环已用增量形式 `33cc6: addw a2,t4,a2` + `33cc4: addi a5,a5,4`，无削减缺口）；其余 rows（atomic/spin-wait/tail-call/zero-based/JIT/sync）无对应信号。
- 入口条件 A 排序：仅一个顶层 finding（+supporting），无同层 independent 竞争；收益上界以**局部样本份额**表述（非 Amdahl）。

## Phase 4 — Root-cause blueprint / 根因蓝图：bits_image_fetch_nearest_affine_none_a8r8g8b8

**对应 Phase 3 通过 gate 的 row**：primary = `RVV Indexed Gather for Table Lookup and Data-Dependent Access`（rows-operator-rvv.md 行内判据）；supporting = `No vectorization`。

1. **Root cause**：affine 变换的 nearest 采样把每个输出像素的源地址变成运行期计算的、逐像素不连续的索引，编译器只能退回逐 lane 标量 gather：每像素执行索引算术（`mulw`+`add`+`slli`）、标量 `lw`、标量 `sw` 与循环控制。机制要点引用（`patterns/rvv_gather_indexed_memory_access.md`）：「Indirect access 的地址逐 lane 不同，标量路径为每个元素重复执行索引算术、地址生成、加载和循环控制」（§Why this is slow 第 1 条）；「gather latency depends on locality and microarchitecture……即使指令数下降，gather 仍可能受 memory-level parallelism 和 cache miss 限制」（§Why this is slow 第 3 条）；「vluxei* 为 unordered、vloxei* 为 ordered，副作用/异常敏感场景使用 ordered」（§The fix 第 2 条）。叠加 L0 baseline finding：libpixman-1.so 按无 `v`（也无 Zb*）的 ISA 构建，任何 RVV 指令在该对象内均不可能存在，向量化不仅缺 kernel，连 build 能力都未启用。

2. **The fix / 修复方式**（与 pattern §The fix 一致；`m1` 仅表达结构，非固定最优）：
   - 先修 build：以硬件与工具链共同支持的精确 `-march`（含 `v`，如 `rv64gcv` 仅示意，须用验证值）clean rebuild libpixman-1.so —— 否则任何 RVV 代码无法进入对象（`patterns/no-vectorization.md` §The fix 第 1 条）。
   - 主循环向量化形态（修复前/后伪代码，表达结构非补丁）：
     ```c
     // Before（当前 annotate 形态）：逐像素标量 gather
     for (x = 0; x < width; x++) {
         src_x = (fx >> 16); src_y = (fy >> 16);   // 定点坐标
         if (bounds_ok(src_x, src_y))
             dst[x] = src[src_y * stride + src_x];  // 33cb6/33cba/33cbc/33cc0
         else
             dst[x] = 0;                            // 33cfa: sw zero
         fx += ux; fy += uy;                        // 33cc6/33cca: addw
     }
     ```
     ```c
     // After（RVV 形态）：向量化 index 构造 + vluxei gather + vse store
     for (; remaining > 0; ) {
         size_t vl = __riscv_vsetvl_e32m1(remaining);
         // 定点坐标向量：每 lane 并行推进 fx += i*ux；索引 = (fy>>16)*stride + (fx>>16)
         // byte offset = idx << 2（sizeof(uint32_t)）
         vuint32m1_t byte_offset = ...;                     // 向量构造，无临时数组往返
         vuint32m1_t pixel = __riscv_vluxei32_v_u32m1(base, byte_offset, vl);
         __riscv_vse32_v_u32m1(dst, pixel, vl);             // 对应 33cc2 unit-stride store
         dst += vl; remaining -= vl;
     }
     // bounds-check 语义保留：越界 lane 在标量 guard 路径或 mask 中置 0（对应 33cfa: sw zero）
     ```
   - 适用前提：每 lane 索引可在寄存器内向量化构造（定点坐标增量 `ux/uy` 是循环不变量，可 `vid`+`vmul`+`vadd` 构造）；索引范围由上游 bounds-check 语义保证或需 clamp/mask；index EEW（32-bit）与 data SEW（32-bit）的 EMUL 组合合法（index LMUL = data LMUL）；迭代间无跨元素依赖（本函数满足 —— 每像素独立，无递推/写后读）。
   - 不可破坏的 correctness contract：affine 定点坐标的 16.16 定点语义（`33c14: slli a5,a5,0x10` + `lui 0x8` = +0.5 四舍五入）、nearest 取整（`sraiw` 算术移位）、越界像素置零（`33cfa: sw zero`）、mask 缓冲非空路径（`33c82/33d00` 分支）、循环边界 `bne a5,a0`。
   - 限制/风险：gather 吞吐依赖源像素 cache locality —— 若 affine 变换下源访问随机导致 memory-bound，RVV gather 收益受带宽限制（pattern §Why this is slow 第 3 条），需 bound-type 实测确认；`vluxei` 需保证 index 合法（越界 clamp 或 mask）；build 无 `v` 时必须先重建并以 runtime dispatch 进入 RVV 路径，不能指望旧 binary。
   - 修复后预期 Profile signals：annotate 中应出现 `vsetvli`、index 构造指令（`vid`/`vmul`/`vadd`）与 `vluxei32.v`（或 64-bit 索引形式）、`vse32.v`；`33cc0 lw`、`33cb6 mulw`、`33cc2 sw` 的标量份额显著下降或消失（见 Phase 5）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcvh`+Zb*+RVV 1.0（VLEN=256，vlenb=32）；build ISA = 无 `v`/无 Zb*（`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_...`）；VLEN = 256 bits；bound type = `baseline_gap: bound type`（IPC 1.591 方向性，缺 cache/memory counter）。

4. **收益上界**：入口条件 A，按 evidence sample share 加总 —— gather 相关行（`33cb2+33cb6+33cba+33cbc+33cc0+33cc2`）合计约 **63.4% 局部样本份额**；主循环区间合计约 **97.3% 局部**。表述为「当前 sampled event（cycles:u, local period）下的函数内局部样本份额」；因 percent type=local period 且函数级 workload 贡献未知，**不得**称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata` 相关项）。

5. **三维路由判定**：
   - `current source`：compiler-generated scalar 代码（annotate 证据：标准 RISC-V 标量指令、无 `.S` provenance）。
   - `implementation existence/reachability`：pixman 存在 fast-path 函数指针分派架构，但本环境无源码访问权（`source_context_gap`），无法证明仓库中存在/缺失 riscv64 RVV fetch kernel，也无法证明 dispatch 未选中某实现 → 不进入 kernel-selection row 命中，也不进入 policy-backed missing `.S` 分支（policy/existence 四证不全）。
   - `function-level policy`：pixman fast-path 架构本身允许按格式/变换组合注册专用 fetch 函数（函数名 `bits_image_fetch_nearest_affine_none_a8r8g8b8` 即专用路径命名），但该推断不足以满足 missing-assembly 的 policy 四证 → 修复载体建议为 intrinsic/普通向量化代码 + runtime dispatch gate，不强制 `.S`。

6. **Implementation-shape proof**：不适用（未进入 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**（按 pattern 分组，来自各 pattern 本地 §Related PRs 表）：
   - `patterns/rvv_gather_indexed_memory_access.md`：Related PRs：6 条 URL — https://github.com/opencv/opencv/commit/ce0516282a53947933387bb8e4262a0594ce0913 ；https://github.com/openjdk/jdk/commit/d13e53346f3cd50bf7a4241ba86d2e21d9081bbe ；https://github.com/openjdk/jdk/commit/88801caef6ccdc5ba9ade2af830f3b3cd96e1467 （Has perf data）；https://github.com/openjdk/jdk/commit/44d3a68d8a73c119b64772687d74e5ce25926f4f ；https://github.com/openjdk/jdk/commit/c37e8638c98cb4516569304e9a0ab477affb0641 ；https://github.com/v8/v8/commit/de361fd2544a89c0c94fd820f987378edbb98fce
   - `patterns/no-vectorization.md`（supporting）：Related PRs：15 条 URL — https://github.com/opencv/opencv/pull/22179 ；https://github.com/opencv/opencv/pull/22520 ；https://github.com/opencv/opencv/pull/23980 ；https://github.com/opencv/opencv/pull/24058 ；https://github.com/opencv/opencv/pull/24132 ；https://github.com/opencv/opencv/pull/24166 ；https://github.com/opencv/opencv/pull/24301 ；https://github.com/opencv/opencv/pull/24325 ；https://github.com/opencv/opencv/pull/27160 （Has perf section）；https://github.com/opencv/opencv/pull/27119 （Has perf section）；https://github.com/opencv/opencv/pull/27097 （Has perf section）；https://github.com/opencv/opencv/pull/27007 （Has perf section）；https://github.com/opencv/opencv/pull/26958 （Has perf section）；https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d ；https://github.com/opencv/opencv/commit/2c16f3b7d2b2 （原文为 8f95b40d9266a6a 截断体，按本地表逐字输出）→ 按本地表逐字 15 条。

## Phase 5 — Verification forecast / 验证预测：bits_image_fetch_nearest_affine_none_a8r8g8b8

**primary finding（RVV Indexed Gather）验证预测**：
- **应消失/缩小**（锚定 Phase 3(a) 引用行）：`8.45 :  33cc0:  lw      a1,0(a1)` 标量 gather load 应消失或显著缩小；索引算术行 `0.15 :  33cb6:  mulw a1,a6,a1`、`0.19 :  33cbc:  slli a1,a1,0x2` 与 `8.56 :  33cb2:  ld t1,168(s1)` 对应标量序列应缩小；`46.02 :  33cc2:  sw a1,0(a5)` 应转为向量 store 形态、单行份额大幅下降。
- **应出现**（锚定 `patterns/rvv_gather_indexed_memory_access.md` §Verification「指令验证」）：annotate 出现 `vsetvli`、index 构造指令（`vid`/`vmul`/`vadd`/`vwaddu`）及 `vluxei32.v`/`vluxei64.v`（unordered；副作用/异常敏感时 `vloxei*`）与 `vse32.v`；同一函数重跑 `perf annotate` 确认标量指令不再主导 hot loop（对应 no-vectorization §Verification）。
- **正确性对照**：对相同输入比较 RVV gather 与标量 reference 逐元素一致；覆盖最小/最大合法索引、越界保护、index×sizeof 溢出边界、零/单元素/VLMAX 整倍数/tail 长度（VLEN-agnostic，不硬编码 lane 数）。
- **bound-type 验证**（pattern §Verification「locality 与 bound-type 验证」）：用随机/局部索引测量 cache 行为，确认不是带宽受限后再设定 gather 收益目标。
- **可达性验证**：通过符号/日志/采样确认真实 workload 经 pixman fast-path dispatch 进入 RVV 路径；build 以精确 `-march`（含 `v`）重建后用 `readelf -A` 复核 `Tag_RISCV_arch` 含 `v`；不支持 RVV 或 index/EMUL 不合法时确认进入 scalar fallback（不执行非法指令）。
- 升级结论所需补采数据（无）：本函数为入口条件 A（profile-backed），不需静态升级；如需称 workload 级收益，需补函数级 sample share（global-period annotate 或全程序热点排序）。

**supporting finding（No vectorization）**：不单独验证（supporting 跟随 primary 的验证预测；其 §Verification 的「annotate 现在包含 RVV instructions」已并入上方「应出现」侧锚点）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status / Anchor 载荷 |
|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ `1/1 组；bits_image_fetch_nearest_affine_none_a8r8g8b8` |
| 2 | Phase 1 输出要求满足 | ✅ 7 行 baseline + 2 个 L0 gate + bound-type gate；gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`（函数级贡献）；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足 | ✅ 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md、rows-codegen.md`；顶层 finding 1（+supporting 1）；evidence 锚点 `8.45 :  33cc0:  lw a1,0(a1)`、`46.02 :  33cc2:  sw a1,0(a5)`、`0.15 :  33cb6:  mulw a1,a6,a1`；supporting 1；排除条数（含 rows-codegen 逐行排除与 4 项行内互斥邻居）；推导式 2 组 |
| 4 | Phase 4 输出要求满足 | ✅ 已读 pattern：`patterns/rvv_gather_indexed_memory_access.md`（§Why this is slow 第 1/3 条、§The fix 第 2 条）、`patterns/no-vectorization.md`（§The fix 第 1 条）；命中 row：`RVV Indexed Gather for Table Lookup and Data-Dependent Access`（primary）、`No vectorization`（supporting）；`The fix` 含 before/after 伪代码、correctness（16.16 定点/越界置零/mask 分支）、风险（locality/bound-type）与 Profile 信号锚点（`vsetvli`/`vluxei32`/`vse32`）；`Related PRs：6 条 URL`（gather）、`Related PRs：15 条 URL`（no-vectorization） |
| 5 | 路径合规 | ✅ 模式 A（profile_backed）+ 路径 gather semantic primary + no-vectorization supporting；8 项 class 扫描集可解释；每个 blueprint leaf 来自通过 gate 的 row；按动态局部份额排序（单一顶层 finding） |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧：`33cc0: lw a1,0(a1)` / `33cc2: sw a1,0(a5)` / `33cb6: mulw a1,a6,a1`；出现侧：`patterns/rvv_gather_indexed_memory_access.md` §Verification 指令验证（`vsetvli`/`vluxei*`/`vse32`） |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁生成、无契约外实施分支；无向用户追问；交付止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 |

修正记录：无