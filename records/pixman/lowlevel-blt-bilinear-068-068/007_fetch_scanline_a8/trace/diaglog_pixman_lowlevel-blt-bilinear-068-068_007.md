Functions under analysis: [fetch_scanline_a8]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`fetch_scanline_a8`，libpixman-1.so.0.46.5 @ 0xc3ca，32 行，含完整 hot loop c3ec–c3fc）
- perf stat（bound/context）：已提供（共享 perf stat：duration 96.77s、cpu_cycle 212.68G、instruction 486.52G、IPC 2.287515，status PASSED）
- workload/binary/DSO/source context：已提供（`lowlevel-blt-bench` PIE ELF64 riscv；annotate 无 DWARF 源码行 → 无函数内 file:line 归属，标 `source_context_gap: 无 DWARF 行`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata 冻结 `lowlevel-blt-bench-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`，**不含 `v`**）
- hardware ISA：已提供（SpacemiT X100，`rv64imafdcvh_...`，含 `v`；OoO，metadata 冻结）
- vlenb：已提供（vlen_bits=256，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：部分（event=`cycles:u`；percent type=`local period`；单次采样窗口；该函数对 workload 的贡献占比未知 → `baseline_gap: sampling metadata`）
- Sampling IP precision：缺失（无 `precise_ip`/Exact-IP 信息 → `baseline_gap: sampling IP precision`）
- 样本数：592（cycles:u，函数局部 period）——足够支撑区间级归属，不支持无 skid 前提下的单指令 latency 归因

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v`（RVV 1.0）存在；另有 zba/zbb/zbc/zbs、zvbb/zvbc、zvk* 系列；core 为 OoO（metadata 冻结：out-of-order） |
| Build ISA | `Tag_RISCV_arch = rv64i2p1_..._zcd1p0`，**无 `v`**（也无 zba/zbb）→ **L0 gate：hardware 有 `v` 而 build 无 `v`，mismatch，最高优先级 baseline finding** |
| Vector flavor | annotate 全 scalar（0 条 `v*`、0 条 `th.v*`）；无 th.v* 迁移问题；mismatch 形态是 hw-v / build-no-v |
| VLEN | 256 bits（vlenb=32，hwprobe 快照冻结） |
| Bound type | perf stat IPC=2.2875；hot loop 每像素 1 条 lbu + 1 条 slliw + 1 条 sw + 2 条 addi + 1 条 bne，短 loop-carried 依赖链，68×68 量级工作集，非带宽主导 → latency/ILP 受限的标量逐元素循环；无 cache/memory counter → bound 分类部分成立 |
| Sampling semantics | event=`cycles:u`；`--percent-type` 为 **local period**；同一窗口；函数级 workload 贡献未知 → 百分比只表示函数内局部样本份额，不得推导 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip / Exact-IP 信息）；单行高占比（sw 41.84%、bne 44.49%）只锚定 c3ec–c3fc 循环区间，不做单指令 latency/cost 归因 |

L0 baseline gate 判定：hardware 有 `v`、build 无 `v` → mismatch 作为顶层 baseline finding 置顶；依赖 `v` 的修复 route 需先修正 build ISA（重建），不停止全局 row 扫描。Bound-type gate：IPC 已知、无 cache counter → 非内存带宽主导成立（像素规模小、store:load=4:1 字节），compute/latency-bound 分类可用但带部分 gap。

## Phase 2 — Scope / 分析边界

- 函数清单与承诺声明一致：`fetch_scanline_a8`。
- hot basic block / loop interval：`c3ec–c3fc`（取数→扩位→写回→回边），592 samples 中该区间占 97.47%（1.52+8.94+0.34+0.34+41.84+44.49）。
- 最高行原文（trace anchor）：`44.49 :   c3fc:   bne     a3,a4,c3ec <fetch_scanline_a8+0x22>`；次高 `41.84 :   c3f8:   sw      a5,-4(a4)`。
- Sampling IP precision 未确认 → 归因收敛到区间级机制：该区间是 1 字节 A8 alpha 逐像素扩为 32-bit ARGB 字（`slliw a5,a5,0x18` 把 alpha 置于 bits 24–31）的标量转换循环。
- annotate 覆盖完整（prologue c3ca–c3e8 约 1.0%、epilogue c400–c406 约 1.9%），不走 annotate_incomplete。

## Phase 3 — Pattern scan / 模式扫描：fetch_scanline_a8

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码为 compiler-generated 标量循环（lbu/slliw/sw/addi/bne 形态，无手写 `.S` 的 unroll/register 管理特征，annotate 无 `.S`/DWARF provenance）。
2. `rows-operator-rvv.md` — **include** — compiler-generated 标量循环、算子语义明确（A8→ARGB32 alpha 注入/扩位转换：窄 load/store + shift），为必选主归属 class。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel/compare/checksum；store 值经 `slliw` 依赖 load 值且宽度 1→4 字节变化，是格式转换而非数据搬运。
4. `rows-vectorized-tuning.md` — **exclude** — annotate 内 0 条 `v*`，无已向量化代码可调。
5. `rows-codegen.md` — **exclude** — 无 kernel-selection 错选信号（本函数即 A8 fetch slot 的既定实现）、无 spill/冗余扩展/代码形态异常（`mulw`+`slli` 行指针计算已循环外提；`slliw` 为转换语义必需）；prologue 仅 s0+ra 两寄存器最小化。L0 build-ISA finding 属 baseline 层，不构成 codegen row 命中。
6. `rows-offload.md` — **exclude** — 无矩阵引擎 / packed-SIMD 证据。
7. `rows-crypto.md` — **exclude** — 非密码学原语。
8. `rows-runtime-os.md` — **exclude** — 用户态 benchmark，非 RTOS/kernel/CSR 路径。

`Classes scanned: rows-operator-rvv.md`（含对 rows-string-memory.md / rows-codegen.md 互斥判据行的核对）

### Local performance pattern scan: fetch_scanline_a8

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Layout and Channel Packing Kernels（primary） | 循环区间逐字行：`1.52 :   c3ec: lbu a5,0(a1)`、`0.34 :   c3f4: slliw a5,a5,0x18`、`41.84 :   c3f8: sw a5,-4(a4)`；1B→4B alpha 注入/扩位，全 scalar，0 条 `v*`；硬件有 `v`、build 无 `v`（L0） | High | Medium | `patterns/rvv_layout_and_channel_packing_kernels.md` |

**三件套（primary：RVV Layout and Channel Packing Kernels）**

- **(a) 逐字 evidence 引用**：hot loop interval c3ec–c3fc 内：
  `1.52 :   c3ec:   lbu     a5,0(a1)`（逐字节 A8 载入）
  `0.34 :   c3f4:   slliw   a5,a5,0x18`（alpha 移到 bits 24–31）
  `41.84 :   c3f8:   sw      a5,-4(a4)`（写 32-bit ARGB 字）
  `44.49 :   c3fc:   bne     a3,a4,c3ec`（end-pointer 回边）
  该 6 行区间合计 97.47% 函数内样本；行内语义链为 1-byte load → 左移 24 → 4-byte store 的逐像素通道扩位（alpha 注入），无任何 multiply-accumulate、无颜色矩阵系数。
- **(b) 互斥邻居排除**：
  - color-conversion row：排除——循环内无颜色矩阵/chroma sampling（无 MUL/MAC/系数；唯一算术为 `slliw` 位移），证据见 (a) 三行。
  - strided-layout row：排除——源指针 `addi a1,a1,1`（单位步长）、目标指针 `addi a4,a4,4`（单位步长），循环内无固定 element/byte stride 计算。
  - indexed-gather row：排除——地址无 data-dependent 索引/LUT，源地址为顺序 +1。
  - resampling row：排除——无插值坐标/权重。
  - precision-conversion row：排除——转换是固定位域布局构造（alpha 注入到 bits 24–31、低 24 位清零），无数值转换/rounding 语义；其行内互斥也指明"低比特 layout decomposition 主导且无 accumulator → layout-packing row"。
  - weight-repack / quantized-matmul / matrix-engine rows：排除——非 GEMM 语境、无 zero-point/requantization、无矩阵引擎。
  - register-group/unroll（rows-vectorized-tuning）：排除——0 条 `v*`，无 LMUL/live-set 可调。
  - rows-string-memory copy/fill row：排除——store 值依赖 load 值经 shift 且 1→4B 变宽，非等宽 copy/fill。
- **(c) 双 Confidence 推导式**：`route: compiler-generated 标量循环 + disassembly 直接证明固定 lane/bit-field 映射（lbu→slliw 0x18→sw）+ 互斥邻居逐一排除 → High；impact: 区间样本份额 0.975 + VLEN 256 已知 + bound 非带宽主导，但 sampling metadata 部分缺失（local period、workload 贡献未知）、IP precision 未知、build 无 v 需先重建 → Medium`。

**仲裁小段**：本函数仅一个顶层 primary finding（layout-packing，L1 vectorization/semantic 层）。L0 build-ISA mismatch（hw `v` / build 无 `v`）为 baseline gate finding，置于一切 pattern 之上——它解释"为何二进制内不可能出现 `v*`"，与 L1 语义命中属同一证据链（同一 hot loop），L0 修复（重建）是 L1 修复（RVV 化）的前置，不构成独立顶层 finding。`no-vectorization.md` 的自身 gate 不成立（其行内互斥：更具体 semantic row 认领后本行不认领），仅作 symptom 提及，不计命中数。

## Phase 4 — Root-cause blueprint / 根因蓝图：fetch_scanline_a8

- 对应 Phase 3 通过 gate 的 row：`RVV Layout and Channel Packing Kernels`（rows-operator-rvv.md 行内判据成立）。
- **Root cause**：A8→32-bit ARGB 的逐像素标量扩位循环。机制引用 pattern 独有内容——`patterns/rvv_layout_and_channel_packing_kernels.md` §Why this is slow："纯布局路径的根因是逐通道标量访存、固定 lane permutation 被展开为长标量序列"；此处每像素以 1 条窄 load + 1 条 shift + 1 条窄 store 完成 alpha 注入，循环控制（addi×2+bne）逐像素重复。根因两层：① L0——build `Tag_RISCV_arch` 无 `v`，二进制根本无法包含 RVV 指令（依据 metadata 冻结的 `lowlevel-blt-bench-elf-A` attribute）；② L1——即便具备向量能力，该转换也应以 layout-packing 形态批量完成，当前是逐像素标量。
- **The fix / 修复方式**：先修 L0（build），再按 pattern §5/§8 形态向量化：
  - 修复前（标量形态，即当前 annotate）：每像素 `lbu a5,0(a1); slliw a5,a5,0x18; sw a5,-4(a4)`，加 `addi a1,a1,1; addi a4,a4,4; bne a3,a4`。
  - 修复后（RVV 形态示意，VLEN-agnostic，遵循 `kernel-conventions.md` §3 fixed-VL main loop + runtime-VL tail）：
    ```c
    size_t n;                     /* width */
    int vl = __riscv_vsetvlmax_e8m1();          /* 或按 live budget 选 LMUL */
    vuint8m1_t a = __riscv_vle8_v_u8m1(src, vl);
    vuint32m4_t w = __riscv_vzext_vf4_u32m4(a, vl);  /* e8 → e32 扩位 */
    w = __riscv_vsll_vx_u32m4(w, 24, vl);            /* alpha 置于 bits 24–31 */
    __riscv_vse32_v_u32m4(dst, w, vl);
    src += vl; dst += vl;
    /* runtime-VL tail 处理剩余 0..vl-1 */
    ```
    适用前提：以硬件/工具链共同支持的精确 `-march`（含 `v`，可加 `zvl256b`）重建（依据 `patterns/no-vectorization.md` §The fix step 1 与 L0 gate）。
  - 不可破坏的 correctness contract：输出像素位布局（alpha 在 bits 24–31、低 24 位为零，即 `a<<24`）、行宽/stride 语义、tail 完整性、little-endian 字节序、in-place/overlap 边界（依据 pattern §Shared conventions 与 §Verification）。
  - 限制/风险：若 pixman 官方 policy 走 fetcher 表函数指针（见本批次 `_pixman_bits_image_src_iter_init`/`bits_image_fetch_untransformed_32` 的 jalr 分派），RVV 化实现需注册到同一 fetch slot 才被真实 workload 采用；未确认工具链 tuple/intrinsic 形态时保留 scalar fallback。
  - 修复后预期 Profile signals：c3ec–c3fc 区间标量样本大幅下降，出现 `vle8`/`vzext`/`vsll`/`vse32` 指令，loop-control 份额下降（对应 pattern §Verification 成功判据）。
- **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（含 `v`，OoO）；build ISA=`rv64i2p1_m2p0_..._zcd1p0`（无 `v`）→ `baseline_gap` 已消除为明确 mismatch finding；VLEN=256；bound type=latency/ILP 受限标量循环（IPC 2.29，非带宽主导）。
- **收益上界**：「当前 sampled event（cycles:u）下该函数的局部样本份额」：hot loop 区间合计 **0.975**（592 samples）；因 percent type=local period 且 workload 级贡献未知，**不得**表述为 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`）。
- **三维路由判定**：`current source` = compiler-generated C 代码（disassembly 形态证据，无 `.S`/DWARF，annotate 无源码行 → `source_context_gap: 无 DWARF 行`）；`implementation existence/reachability` = 本 build 无任何 RVV fetch kernel（build 无 `v`），A8 fetch slot 由本标量实现承担，fix 载体为同一 slot 的向量化重写（reachability 证据见同批次 002/003 的 fetcher_info 分派链）；`function-level policy` = pixman fetcher 表函数指针分派（disassembly 可见），无独立 `.S` policy 证据 → **不进入 policy-backed missing `.S` 分支**，`implementation_shape` 不适用。
- **Related PRs 小节**：`Related PRs：20 条 URL`（patterns/rvv_layout_and_channel_packing_kernels.md §Related PRs）

Related PRs URL 清单：
- https://github.com/OpenMathLib/OpenBLAS/commit/c00afc86a6fd318473e14150fec8964104b9cfe6
- https://github.com/OpenMathLib/OpenBLAS/commit/ef7f54b35713a315671f64f03fe0d417f1bb0360
- https://github.com/OpenMathLib/OpenBLAS/commit/07d0e742c2ac18d6342f6a792dc3f8da04499674
- https://github.com/OpenMathLib/OpenBLAS/commit/a8a00bbf4f9169646e1f42ee419a3cc3b4badd82
- https://github.com/opencv/opencv/commit/8a36f119cee5a90c59f03a24f8a6efda3179600a
- https://github.com/opencv/opencv/commit/189f64726437a3756329890ea75c8ca5fde46bcf
- https://github.com/uxlfoundation/oneDNN/commit/ba07c4e658a1ccbb2f6587ab0e5df9831984ee5b
- https://github.com/uxlfoundation/oneDNN/commit/79a1557d2119ceb79ab50897b5c481d66b5381f4
- https://github.com/uxlfoundation/oneDNN/pull/4548
- https://github.com/uxlfoundation/oneDNN/commit/aadd85dd6a2d100fed3a42bc01e669fe4ef0297a
- https://github.com/alibaba/MNN/pull/4079
- https://github.com/alibaba/MNN/commit/73bfaa46fe4f9c0e24e0238073f5ab1a94edca6c
- https://github.com/alibaba/MNN/pull/4021
- https://github.com/alibaba/MNN/commit/0a5ee52e7d209b3b5735419f52dd5dd366657478
- https://github.com/alibaba/MNN/pull/4067
- https://github.com/alibaba/MNN/commit/2c2d7fa68675191bcdad9ebd9727e9c271b0db07
- https://github.com/alibaba/MNN/pull/3813
- https://github.com/alibaba/MNN/commit/6afcf99fedaec90b5b363ce4c78317800f291443
- https://github.com/alibaba/MNN/pull/4426
- https://github.com/vllm-project/vllm/pull/47538

## Phase 5 — Verification forecast / 验证预测：fetch_scanline_a8

- 修复对象：c3ec–c3fc 标量转换循环（fetch_scanline_a8 主体）。
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`c3ec: lbu a5,0(a1)`、`c3f4: slliw a5,a5,0x18`、`c3f8: sw a5,-4(a4)`、`c3fc: bne a3,a4,c3ec` 的样本份额应显著下降；`41.84`/`44.49`/`8.94` 不再主导。
- 应出现侧（锚定 pattern §Verification 性能预测）：同一函数 annotate 出现 `vle8`/`vzext*`/`vsll*`/`vse32`（pattern §Verification："出现预期 segment/stride/reorder 指令"；本函数为 unit-stride 扩位，对应连续 load/store + 批量 shift）；逐通道 scalar load/store 与 loop-control share 下降、bytes/cycle 改善。
- 验证方法（锚定 pattern §Verification + kernel-conventions §Verification checklist）：用目标 `-march`（含 `v`）与相同优化级别重建；与 scalar reference 对比覆盖 n=0、小 n、main-loop 整倍数与全部 0..vl-1 tail 长度；对短中长真实输入 benchmark；检查生成反汇编无 per-iteration `vsetvli`；同一 fetch slot 的真实 workload annotate 应显示新 kernel 指令、原标量份额显著下降。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status / Anchor |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（fetch_scanline_a8） | ✅ — `1/1 组；fetch_scanline_a8` |
| 2 | Phase 1 输出要求（7 行 baseline 表 + 2 L0 gate + bound gate） | ✅ — 7 项结论齐全；`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata` 已点名 |
| 3 | Phase 3 输出要求（8 项 Class selection trace；Classes scanned 声明；1 个顶层 primary；三件套；排除 8 项邻居；仲裁小段） | ✅ — 8 项 trace 齐；`Classes scanned: rows-operator-rvv.md`；顶层 finding=1；evidence 锚点 `c3ec/c3f4/c3f8/c3fc`；supporting=0；排除条数=8；推导式 2 条 |
| 4 | Phase 4 输出要求（pattern 独有内容引用、The fix before/after、correctness、风险、Profile signals、baseline 回填、收益上界、三维路由、Related PRs 20 条 URL） | ✅ — 引用 `rvv_layout_and_channel_packing_kernels.md` §Why this is slow 短引、§5/§8、§Shared conventions、§Verification；`Related PRs：20 条 URL` |
| 5 | 路径合规（trace 可解释扫描集、L0/L1 归属、blueprint leaf 来自通过 gate 的 row、模式 A 动态份额排序） | ✅ — 模式 A（profile_backed）；primary 排序按局部份额 0.975；`th.v*` 不适用 |
| 6 | Phase 5 两侧锚定（消失侧 `c3ec/c3f4/c3f8/c3fc`；出现侧 layout-packing §Verification） | ✅ |
| 7 | 契约边界合规（无实施询问/代码修改/补丁生成；无向用户追问） | ✅ — 交付止于 Profile 证据、根因蓝图、The fix、验证预测 |

修正记录：无