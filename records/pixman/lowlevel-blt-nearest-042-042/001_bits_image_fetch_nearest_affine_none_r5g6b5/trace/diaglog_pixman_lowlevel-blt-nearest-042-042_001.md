Functions under analysis: [bits_image_fetch_nearest_affine_none_r5g6b5]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-bits_image_fetch_nearest_affine_none_r5g6b5-annotate.txt`，libpixman-1.so.0.46.5，2854 samples，`cycles:u`，`percent: local period`，含 hot loop body）
- perf stat（可选 bound/context）：已提供（`21-pixman-benchmark-riscv-lowlevel-blt-nearest-042-042.txt`：duration 44.27s、cycles 97,315,570,061、instructions 201,097,812,207、IPC 2.066、status PASSED）
- workload/binary/DSO/source context：已提供（libpixman-1.so.0.46.5，pixman commit 14735ced；源码树位于 `/root/pixman-workspace/pixman`，命中函数由 `pixman-fast-path.c` 的 `MAKE_NEAREST_FETCHER` 宏生成）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata 快照中 benchmark ELF 的 attribute：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`——**无 `v`**；libpixman-1.so 自身 attribute 未在快照中）
- hardware ISA（`/proc/cpuinfo` / hwprobe 快照）：已提供（SpacemiT X100，`rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_...`——含 `v`（RVV 1.0）、`zicond`、`zba/zbb/zbc/zbs`、`zvbb/zvbc` 等）
- `vlenb`：已提供（hardware_profile_snapshot：vlenb=32 → VLEN=256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cycles:u`、local period、perf_freq=99、perf_cpu=2、单次运行窗口；函数级 workload 贡献未知）
- Sampling IP precision（precise_ip / Exact-IP）：缺失（raw metadata 未提供 precise_ip / Exact-IP 标记，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 暴露 `v`（RVV 1.0，`/proc/cpuinfo` isa 快照含 `v` 与 `zve*` 子集）、`zicond`、`zba/zbb/zbc/zbs`、`zvbb/zvbc`；来源 hardware_profile_snapshot + cpuinfo 快照 |
| Build ISA | 快照中 4 个 benchmark ELF 的 `Tag_RISCV_arch` 均无 `v`（`rv64i2p1_m2p0_..._zcd1p0`）；libpixman-1.so 自身 attribute 未快照 → `baseline_gap: build ISA (热点 DSO)`；可选命令 `readelf -A <libpixman-1.so 路径>` |
| Vector flavor | annotate 内 zero `v*`、zero `th.v*`（全 scalar）；硬件为 RVV 1.0 → 无 `th.v*` mismatch 分支，但 build 未暴露 `v` |
| VLEN | vlenb=32 → VLEN=256 bits（hardware_profile_snapshot） |
| Bound type | IPC=2.066（201.1G instr / 97.3G cycles）→ compute/issue 主导迹象；每像素数据量仅 2B 进 4B 出、042-042 为小图（推断 cache-resident）；**cache/memory/branch counter 缺失** → bound type 部分判定 |
| Sampling semantics | event=`cycles:u`；percent type=local period；同一运行窗口；该函数占 workload 总贡献未知 → `baseline_gap: sampling metadata`，禁止称 workload 级 cycle/耗时份额 |
| Sampling IP precision | precise_ip / Exact-IP / PMU skid 未知 → `baseline_gap: sampling IP precision`；单行占比只锚定 basic block / loop interval，不做 instruction-latency 归因 |

L0 baseline gate（hardware 有 `v` 而 build 无 `v`）：**成立**——快照 ELF attribute 无 `v`，且 `meson.build`（`pixman/meson.build:60`）只对 `pixman-rvv.c` 单独应用 `rvv_flags`（`-march=rv64gcv1p0`），`pixman-fast-path.c`（承载热点函数）在任何配置下都不带 `v` 编译。作为最高优先级 baseline finding 置顶，继续扫描可见 row。
L0 baseline gate（`th.v*` flavor）：annotate 无 `th.v*` → 不触发 flavor mismatch 分支。
Bound-type gate：IPC 2.066 支持 compute/issue 主导，但 cache counter 缺失 → 本轮命中行的 performance-impact confidence 封顶（不高于 Medium）。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`bits_image_fetch_nearest_affine_none_r5g6b5`（1 个）。
hot loop 边界：主循环 interval `32d50–32dde`，另有 mask-skip 变体 `32e0e–32e1e` 与越界写零路径 `32e08–32e0c`（二者 sample≈0）。
最高行 trace anchor：`20.67 :  32da2:  and     a1,a1,t4`（主循环 interval 内，local period）。因 `baseline_gap: sampling IP precision`，该行只锚定 interval 级机制，不承担单指令 latency/cost 归因。
C 源码对应：`pixman-fast-path.c:3004` `bits_image_fetch_nearest_affine()`（每像素 mask 检查 → repeat=NONE 四边界检查 → `convert_pixel(row,x0)|mask`）+ `pixman-fast-path.c:3093` `convert_r5g6b5` → `pixman-private.h:985` `convert_0565_to_0888`（565→8888 位域展开）。

## Phase 3 — Pattern scan / 模式扫描：bits_image_fetch_nearest_affine_none_r5g6b5

### Class selection trace

1. `rows-asm.md` — **exclude**：当前代码来源是 compiler-generated C（annotate 显示编译器约定：stack protector、RVC、lui/addi 常数物化，与源码宏生成一致）；无手写 `.S` 证据；policy-backed missing `.S` 四证不齐（无 pixman 要求汇编实现的 policy 证据，in-tree RVV 先例是 C intrinsic 而非 `.S`）。
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar 循环，热点是 r5g6b5→a8r8g8b8 位域展开（packed decomposition + alpha 注入），属算子语义热点。
3. `rows-string-memory.md` — **exclude**：语义是像素格式转换 + 仿射 fetch，非 copy/fill/sentinel-scan/compare/checksum。
4. `rows-vectorized-tuning.md` — **exclude**：annotate 零 `v*`，无 RVV 配置/寄存器/展开可调。
5. `rows-codegen.md` — **include**：每像素 4 次 bounds compare+branch 与 mask 分支簇（`32d50–32d7a` ≈ 18%），需评估 control-flow / zero-based-compare / induction-variable / kernel-selection 等 row。
6. `rows-offload.md` — **exclude**：无矩阵引擎 / packed-SIMD 信号；K3 X100 该场景非 GEMM/矩阵引擎负载。
7. `rows-crypto.md` — **exclude**：无密码学原语（无 AES/SHA/SM/GHASH/CRC/GF(2^k) 语义）。
8. `rows-runtime-os.md` — **exclude**：用户态 pixman benchmark，无 timer/CSR/特权路径。

Classes scanned: `rows-operator-rvv.md`（全 20 行）、`rows-codegen.md`（全 27 行）。

### Local performance pattern scan: `bits_image_fetch_nearest_affine_none_r5g6b5`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Layout and Channel Packing Kernels（primary） | 转换序列 `32d96–32dcc` 合计 ≈ 51.96%（其中 `32da2 and a1,a1,t4` 20.67%、`32db6 srliw` 12.42%、`32dc6 and` 5.82%、`32dc2 or` 3.68%、`32da6 or` 3.54%、`32db2 or` 2.73%、`32d96 slliw` 2.52%）；窄访存 `lhu` 4.73% + `sw` 8.48% ≈ 13.21%；全 shift/mask/or、无颜色矩阵/chroma；alpha 注入 `or a1,a1,t5`（t5=0xff000→0xff000000） | High | Medium | `patterns/rvv_layout_and_channel_packing_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`、zero `th.v*`；hardware 有 `v`（RVV 1.0, VLEN=256）；build ELF 无 `v` | — | — | `patterns/no-vectorization.md` |

**(a) 逐字 evidence 引用（primary，行属主循环 interval `32d50–32dde`）：**

```
20.67 :  32da2:  and     a1,a1,t4
12.42 :  32db6:  srliw   t0,a0,0x1
 8.48 :  32dd0:  sw      a1,0(a5)
 5.82 :  32dc6:  and     a0,a0,t6
 4.73 :  32d92:  lhu     a0,0(a1)
```

区间内该展开序列（`32d96`–`32dcc` 共 15 条 shift/mask/or）+ 窄 load/store 的局部样本份额合计 ≈ 65.17%（51.96% + 13.21%）。

**(b) 互斥邻居排除：**
- **vs RVV Color Conversion Kernels（`rvv_color_conversion_kernels.md`）**：`32da2/32db6/32dc6/32dcc` 序列只用 `and/srliw/slliw/or` 与位掩码 `t4=0x700ff`、`t3=0xfc00`、`t6=0xf80`、`t5=0xff000`，无颜色矩阵系数、无 chroma sampling、无矩阵 MAC；颜色转换 row 行内互斥明确"只有 channel reorder/alpha/bit-field/panel packing → layout-packing row"。
- **vs RVV Indexed Gather（`rvv_gather_indexed_memory_access.md`）**：源地址为仿射定点走步（`32dd4 addw a2,t1,a2`、`32dd8 addw a4,a7,a4`，x+=ux、y+=uy），逐输出近似连续、非逐 lane 不连续 LUT；地址乘法 `32d88 mulw` 样本 0.00%，不为采样主导。
- **vs RVV Contiguous Elementwise Arithmetic**：非同宽元素算术；是窄 load（`lhu`）→ widening 位域展开 → 宽 store（`sw`）的布局变换，无 add/sub/mul 元素算术主导。
- **vs RVV Strided Layout Transform**：无固定 element/byte stride 访问形态；源寻址经 `rowstride*y0 + x0`（`32d88`–`32d90`）逐像素计算，非 strided load/store 合同。
- **vs RVV Resampling Kernels**：nearest 滤波无 interpolation 权重/分数坐标 FMA；resampling row 行内互斥"像素格式/颜色转换 → layout/color rows"。
- **vs No vectorization（supporting）**：同一 hot interval 零 `v*`，但 layout-packing 是更具体的语义 row；按 `rows-operator-rvv.md` 首段规则，最不具体的 `no-vectorization.md` 只在其余语义 row 均不命中时认领 → 降为 supporting。

**(c) 双 Confidence 推导式：**
- route: main-loop 全 scalar + 源码证实 compiler-generated（`pixman-fast-path.c` 宏展开）+ 位域展开语义直接可见 + hardware `v`（cpuinfo 快照）+ in-tree RVV 先例 `rvv_convert_0565_to_0888_m4`（`pixman-rvv.c:1087`）→ **High**
- impact: 局部 sample share ≈ 65.17%（转换 + 窄访存）→ 收益方向明确；VLEN=256 已知；但 percent-type=local、函数 workload 贡献未知（`baseline_gap: sampling metadata`）、cache/branch counter 缺失、热点 DSO build ISA 未直接快照 → **Medium**

**Supporting evidence 行（挂在 primary 下）：**
(a) `4.17 :  32d50:  beqz    s3,32d58` + hot interval 内 zero `v*`/`th.v*` → supporting because: 同一 main loop 全标量、无任何向量执行路径；只解释缺少向量执行的原因（build 无 `v` 且该 fetch 路径无 RVV kernel），不决定贡献载体；该 row 自身 gate（main loop scalar + hardware 暴露 `v` + zero `v*`）成立。

**rows-codegen 逐 row 排除表（仅列需逐条判别者）：**
- `kernel_selection_and_runtime_specialization`：该 fetcher 无现存 RVV kernel（`pixman-rvv.c` 只注册 combine_32/combine_32_ca/combine_float/fill/blt，未注册任何 iter/fetch），非"已有专用实现未选中"。
- `control_flow_layout_and_transfer_selection`：`32d50–32d7a` 是每像素数据相关 bounds branch，非 branch diamond / 额外无条件跳 / 远跳 / jump-table / RAS 污染；行内互斥"普通 branch prediction → control-flow row"不认领。
- `zero_based_comparison_for_register_pressure_reduction`：`32d64 srliw+32d68 bnez`、`32d76 srliw+32d7a bnez` 是 sign-bit 测试形态，非 `li reg,0`+branch / `SLT/SLTU` 常量形式 / `SNEZ/SEQZ` 物化；且 build ISA 无 `zicond`。
- `loop_induction_variable_strength_reduction`：地址走步 data-dependent（定点进位 `addw a2,t1,a2`），非 `base+i*stride` 固定 stride；`mulw`（`32d88`）样本 0.00% 不主导。
- 其余 22 行无匹配 signal（无 JIT emitter、无 wrapper tail call、无 atomic/同步、无 spin-wait、无 FCSR/FP semantic、无 constant-pool frontend 证据、无 `li reg,0` 物化、无 misaligned 字宽访问等）。

**多命中仲裁小段：**
- 顶层 finding 1 个：RVV Layout and Channel Packing Kernels（primary，L1 vectorization / semantic dispatch）。
- Supporting 1 个：No vectorization（同一 hot interval、同一机制域，证据不可与 primary 分账）。
- L0 baseline finding（hardware `v` vs build 无 `v`，含 fetch 路径无 RVV kernel 的事实）置顶单列，不参与顶层计数。
- 边界检查簇（`32d50–32d7a` ≈ 18.0%）经**因果消除测试**并入 primary：RVV fetcher 改写以向量 compare/merge 吸收该簇（越界 lane 合并为 0），故不单列 independent finding。
- 入口条件 A：按 evidence sample share 排序——唯一顶层 primary，局部份额 ≈ 65.17%，作为收益上界（局部口径）；supporting 不单独排序。

## Phase 4 — Root-cause blueprint / 根因蓝图：bits_image_fetch_nearest_affine_none_r5g6b5

（依据 Phase 3 通过 gate 的 row：`rows-operator-rvv.md`「RVV Layout and Channel Packing Kernels」primary +「No vectorization」supporting）

1. **Root cause**：固定 bit-field packed decomposition（r5g6b5→a8r8g8b8）在暴露 `v` 的核上以约 14 条标量 shift/mask/or 指令**每像素逐元素重复**执行，并在窄整数位域处理完成后未于消费边界才 widening，叠加每像素 4 次 repeat=NONE 边界分支（`32d58–32d7a`）。机制引用（`patterns/rvv_layout_and_channel_packing_kernels.md`）：§Why this is slow「位域 pack/unpack 每元素重复」+「低比特数据在 shift/mask 完成前过早扩宽而放大 register group」；§The fix 第 8 步「先分解规则低比特 packed 数据再扩宽……只在消费边界 widening」。贡献载体：build 未带 `v` 且该 fetch 路径无 RVV kernel（supporting 解释：`no-vectorization.md` §Why this is slow「vector unit 没有处理 hot main-loop 的并行元素」）。
2. **The fix / 修复方式**（纠正对象：每像素标量 565→8888 展开 + 每像素边界分支；操作方式：新增 RVV nearest-affine fetcher）：
   - **Before（scalar 每像素，`32d92`–`32dd0`）**：`lhu` → 15 条 `and/srliw/slliw/or`（掩码 0x700ff/0xfc00/0xf80/0xff000）→ `sw`（每像素重复）。
   - **After（RVV，VLEN=256 → u16m2 = 16 lane）**：`vle16` 一次载入 16 个 565 像素 → 窄向量内位域分解（`vand` + `vsrl/vsll` + `vor`）→ 消费边界 widening 到 u32m4 → `vor` 注入 0xff000000 → `vse32`。该形态与 in-tree 先例 `rvv_convert_0565_to_0888_m4`（`pixman-rvv.c:1087–1109`，`vand`+`vwmulu/vwmaccu`+`vnsrl/vor`）同构，可直接复用。
   - 边界/掩码合同：每 lane 由定点 x,y 计算 x0,y0，向量 compare（越界）→ `vmerge` 为 0x00000000（repeat=PIXMAN_REPEAT_NONE 合同）；mask 非 NULL 时 `vle32` mask 比较合并，NULL 时单路径跳过。
   - 前提：以 `USE_RVV` 构建（in-tree `-march=rv64gcv1p0`，`meson.build:370`），并在 RVV implementation 链注册 fetcher（`_pixman_riscv_get_implementations` → `_pixman_implementation_create_rvv`，`pixman-riscv.c:122`；当前 `pixman-rvv.c` 未注册任何 iter/fetch）。
   - correctness contract（不可破坏）：越界像素输出 0（透明黑，非 clamp）；alpha=0xff 注入来自 `PIXMAN_FORMAT_A(format)?0:0xff000000`（`pixman-fast-path.c:3055`）；每个输出像素与标量逐位一致（packed 整数值直接比较，pattern §Verification）；rowstride 寻址（`bits + rowstride*y0 + x0`）合同。
   - 限制/风险：变换下源地址可能出界 → 必须先用向量 compare 判定再合并像素值，不得对越界地址实际发起 load；X100 上 misaligned vector access 的代价未知（hwprobe `MISALIGNED_VECTOR_PERF` key 未采集）；odd width / tail 用 `RVV_FOREACH_*` 的 runtime-VL 尾处理；不同 LMUL（m1/m2/m4）需按 live-vector budget 实测。
   - 预期 Profile signals：`32d96–32dcc` 转换簇与 `lhu/sw` 的局部份额显著下降；出现 `vsetvli`/`vle16`/`vse32`/`vand`/`vor`/`vw*`；`32d50–32d7a` 分支簇份额下降。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（含 `v`、`zicond`、`zba/zbb/zbc/zbs`、`zvbb/zvbc`）；build ISA=快照 ELF 无 `v`（热点 DSO attribute 未快照 → `baseline_gap: build ISA (热点 DSO)`）；VLEN=256；bound type=compute/issue 主导（IPC 2.066），cache counter 缺失。
4. **收益上界**：当前 sampled event（`cycles:u`）下该函数的**局部样本份额 ≈ 65.17%**（转换 51.96% + 窄访存 13.21%）；`baseline_gap: sampling metadata`（local period + workload 贡献未知）→ 不得表述为 workload 级 Amdahl 上界。dynamic priority：入口 A，唯一顶层 primary。
5. **三维路由判定**：
   - current source：compiler-generated C（`pixman-fast-path.c:3004` + `:3093` + `pixman-private.h:985`，annotate 与源码吻合）。
   - implementation existence/reachability：in-tree RVV 实现存在（`pixman-rvv.c:1087` 的 565→8888 helper；`pixman-riscv.c:122` dispatch，`USE_RVV` gate），但该 nearest-affine fetcher 无 RVV 实现、未注册 iter/fetch → 需**新增** RVV fetcher（C intrinsic 先例，非 `.S`）。
   - function-level policy：pixman 以 `fast_iters[]` / fast-path 表 dispatch（`pixman-fast-path.c:3171`）；无"assembly-default"政策证据 → 不进 policy-backed missing `.S` 分支。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs 小节**：
   - 按 `patterns/rvv_layout_and_channel_packing_kernels.md` §Related PRs：**Related PRs：20 条 URL** — OpenBLAS c00afc86a6fd、ef7f54b35713、07d0e742c2ac、a8a00bbf4f91；OpenCV 8a36f119cee5、189f64726437；oneDNN ba07c4e658a1、79a1557d2119、#4548、aadd85dd6a2d；MNN #4079、73bfaa46、#4021、0a5ee52、#4067、2c2d7fa、#3813、6afcf99、#4426；vLLM #47538。
   - 按 `patterns/no-vectorization.md` §Related PRs（supporting）：**Related PRs：19 条 URL** — OpenCV #22179、#22520、#23980、#24058、#24132、#24166、#24301、#24325、#27160、#27119、#27097、#27007、#26958、#26865、b902a8e792e1、2c16f3b7d2b2、e06502a254f7、a2d784b6f53a、83104bed3209。

## Phase 5 — Verification forecast / 验证预测：bits_image_fetch_nearest_affine_none_r5g6b5

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`32da2: and a1,a1,t4`（20.67%）、`32db6: srliw t0,a0,0x1`（12.42%）、`32dc6: and a0,a0,t6`（5.82%）、`32dd0: sw a1,0(a5)`（8.48%）、`32d92: lhu a0,0(a1)`（4.73%）的样本份额应显著下降；`32d50–32d7a` 每像素边界分支簇（≈18%）应缩小。
- **应出现侧**（锚定 pattern 文件 §Verification）：`patterns/rvv_layout_and_channel_packing_kernels.md` §Verification——「出现预期 segment/stride/reorder 指令，bytes/cycle 改善」：annotate 应出现 `vsetvli`/`vle16`/`vse32`/`vand`/`vor`/`vw*`；「逐通道 scalar load/store、shift/OR 和 loop-control share 下降」。`patterns/no-vectorization.md` §Verification——「同一函数的 annotate 现在包含 RVV instructions，例如 `vsetvli`、`vle*`、`vse*`…；scalar instructions 不再主导 hot loop」。
- **验证动作**：以 `USE_RVV`（`-march` 含 `v`，in-tree 用 `rv64gcv1p0`）重建并在 RVV implementation 注册 fetcher；对同一函数重跑 `perf annotate`；对 `lowlevel-blt-nearest-042-042` 及同系列不同尺寸输入 benchmark（cycles/instructions/IPC 对比）；正确性以 packed 整数值逐位比较（非图像显示）。缺 cache/memory/branch counters 时补采以确认 bound type 与收益幅度。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | `1/1 组；bits_image_fetch_nearest_affine_none_r5g6b5` |
| 2 | Phase 1 输出要求满足 | ✅ | `7 行 baseline 表 + 2 个 L0 gate + bound-type gate；gap: baseline_gap: sampling metadata / baseline_gap: sampling IP precision / baseline_gap: build ISA (热点 DSO) / 缺 cache counter` |
| 3 | Phase 3 输出要求满足 | ✅ | `8 项 Class selection trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding 1（primary layout-packing）；evidence 锚点 20.67 : 32da2: and a1,a1,t4 等；supporting 1；rows-codegen 逐 row 排除 4 条 + 整组排除；推导式 2 条（route High / impact Medium）` |
| 4 | Phase 4 输出要求满足 | ✅ | `已读 pattern: rvv_layout_and_channel_packing_kernels.md（hit row: RVV Layout and Channel Packing Kernels；引用短语: 位域 pack/unpack 每元素重复 / 先分解规则低比特 packed 数据再扩宽）、no-vectorization.md（supporting；The fix 步骤 1: 精确 -march 重建）；before/after、correctness、风险、Profile 信号锚点齐全；Related PRs：20 条 URL + 19 条 URL` |
| 5 | 路径合规 | ✅ | `入口模式 A + primary/supporting 仲裁（L1）+ L0 baseline 置顶 + th.v* 未全局停扫（无 th.v*）+ 唯一顶层 finding 按局部份额 65.17% 排序` |
| 6 | Phase 5 两侧锚定 | ✅ | `消失侧: 32da2 / 32db6 / 32dc6 / 32dd0 / 32d92；出现侧: rvv_layout_and_channel_packing_kernels.md §Verification + no-vectorization.md §Verification` |
| 7 | 契约边界合规 | ✅ | `无实施询问、无代码修改、无补丁生成；交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测` |

修正记录：无