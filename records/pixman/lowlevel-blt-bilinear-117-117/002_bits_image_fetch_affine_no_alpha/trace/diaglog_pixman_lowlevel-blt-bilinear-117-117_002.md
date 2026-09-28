Functions under analysis: [_bits_image_fetch_affine_no_alpha]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`__bits_image_fetch_affine_no_alpha`，libpixman-1.so.0.46.5，7934 samples，event=cycles:u，percent=local period；覆盖 1637c–1746c 全函数，入口模式 A profile-backed）
- perf stat：已提供（`lowlevel-blt-bilinear-117-117`：duration_time 262,542,559,285；cpu_cycle 577,081,392,118；instruction 1,453,402,972,228；IPC 2.518541；status PASSED）
- workload/binary/DSO/source context：已提供（pixman master @14735ce；源码树可读：`pixman/pixman/pixman-bits-image.c:475` `__bits_image_fetch_affine_no_alpha`，affine 变换 fetch 迭代器；bilinear 分支调用 `bits_image_fetch_pixel_bilinear_32`（pixman-bits-image.c:433）→ 4× `image->fetch_pixel_32` 间接调用 → `bilinear_interpolation`（pixman-inlines.h:130 SIZEOF_LONG>4 64-bit 变体））
- readelf -A（build ISA）：已提供（Tag_RISCV_arch `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`；**无 `v`**）
- hardware ISA：已提供（SpacemiT X100，RVV 1.0 `v` + `zvbb`/`zvbc`/`zvk*`/`zfa`/`zbb`）
- vlenb：已提供（vlenb=32 → VLEN=256 bits）
- 采样元数据：部分提供（event=cycles；percent type=**local period**；同一窗口；函数 workload 贡献未知）→ Phase 1 gap
- Sampling IP precision：缺失（precise_ip / skid 能力未知）→ Phase 1 gap

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：SpacemiT X100，RVV 1.0 `v` + `zvbb`/`zvbc`/`zvk*`/`zfa`/`zbb`；VLEN=256（vlenb=32） |
| Build ISA | 已提供：libpixman-1.so Tag_RISCV_arch 无 `v`；meson `rvv` 选项未启用，pixman-rvv.c 未编译入 binary（pixman-riscv.c:125-130 运行时检测不可达） |
| Vector flavor | annotate 全 scalar（zero `v*`/`th.v*`）；hardware/build V 能力不一致 |
| VLEN | 已提供：vlenb=32 → VLEN=256 bits |
| Bound type | `baseline_gap: bound type`：perf stat 仅 cycles/instructions（IPC=2.519），无 cache/memory/branch counter |
| Sampling semantics | event=cycles；percent type=**local period**；同一窗口；函数 workload 贡献未知 → `baseline_gap: sampling metadata` |
| Sampling IP precision | `baseline_gap: sampling IP precision`：precise_ip/skid 能力未知 |

L0 baseline gate：**hardware 有 `v`、build 无 `v` → L0 baseline finding 置顶**（同批次共享基线：本 binary 中 RVV 实现整体不可达）；无 `th.v*`。Bound-type gate：`baseline_gap: bound type` → 命中 finding 的 performance-impact confidence 封顶，不得 High。hardware/build mismatch 不停止扫描。

## Phase 2 — Scope / 分析边界
函数清单：[_bits_image_fetch_affine_no_alpha]（与承诺一致）。hot 区间按地址分账（7934 samples local）：
- bilinear 插值计算块 16fba–170ce ≈52.9%（anchor：`5.52 : 170b6: and a6,a4,a6`、`5.00 : 17076: srli a1,a1,0x20`、`4.42 : 170c6: ld a3,-280(s0)`）
- 4× fetch_pixel 间接调用点块 16f56–16fb6 ≈21.2%（anchor：`2.41 : 16f9a: mv a0,s7`、`2.36 : 16f78: mv a2,s8`）
- per-scanline filter 分派 + 循环控制 1648c–164c6 ≈11.6%、1672c–16744 ≈6.2%（anchor：`2.70 : 164ac: lw a4,68(s7)`、`2.07 : 16744: bne a5,s3,1648c`）
- bilinear repeat-normal 入口 1678a–167bc ≈6.9%
Sampling IP precision 未确认 → 单行仅锚定区间，不做单指令 latency 归因。annotate 覆盖完整（模式 A）。

## Phase 3 — Pattern scan / 模式扫描：_bits_image_fetch_affine_no_alpha

### Class selection trace（8 项）
1. `rows-asm.md` — exclude：compiler-generated C（pixman-bits-image.c:475），非手写 `.S`；无独立 `.S` policy
2. `rows-operator-rvv.md` — include：compiler-generated scalar 代码，bilinear 重采样语义（分数坐标 + 4-tap 插值 + 数据相关邻居 load + border mode）
3. `rows-string-memory.md` — exclude：非 string/memory/checksum 语义
4. `rows-vectorized-tuning.md` — exclude：annotate 无任何 `v*`
5. `rows-codegen.md` — include：per-pixel 间接调用形态 + 大栈帧（432B）+ 分派跳表，需评估 call overhead / register-pressure / kernel-selection 等 row
6. `rows-offload.md` — exclude：无矩阵引擎参与像素重采样
7. `rows-crypto.md` — exclude：非密码学原语
8. `rows-runtime-os.md` — exclude：非 RTOS/kernel 热路径

### Classes scanned: rows-operator-rvv.md, rows-codegen.md

### Local performance pattern scan: `_bits_image_fetch_affine_no_alpha`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | bilinear 插值计算块 16fba–170ce ≈52.9% 局部（64-bit 通道并行 `mul`/shift/mask 链，anchor `5.52 : 170b6`、`5.00 : 17076`）+ 4× fetch_pixel 间接调用点 16f56–16fb6 ≈21.2% 局部（数据相关邻居 load）；zero `v*`；hardware V 可用，build 无 v | High | Low | `patterns/rvv_resampling_kernels.md` |

**(a) 逐字 evidence 引用**（均属 bilinear 热区间；采样 IP precision 不足，锚定区间）：
- `5.52 : 170b6: and a6,a4,a6`（插值结果 alpha 通道截取：`mul` 累加后 `f & 0xff0000ff` 语义）
- `5.00 : 17076: srli a1,a1,0x20`（通道重排回 32-bit 的 64-bit shift）
- `4.42 : 170c6: ld a3,-280(s0)`（输出指针 load）
- `2.43 : 17032: and a5,a4,t3`（`0xff00` G 通道 mask）
- `2.07 : 16fca: li a5,256` + `0.00 : 16fce: subw t5,a5,a3`（256-disty 权重计算）
- `1.71 : 17048: mul a5,a5,t0` / `1.71 : 17050: mul a6,a6,t5`（64-bit 通道并行乘加核心，对应 pixman-inlines.h:152 `f = tl64*distixiy + tr64*distxiy + bl64*distixy + br64*distxy`）
- `2.36 : 16f78: mv a2,s8` / `2.41 : 16f9a: mv a0,s7` / `2.19 : 16fa4: ld a1,-288(s0)`（4× fetch 调用点的参数传递与 reload）

**(b) 互斥邻居排除**：
- Spatial Convolution and Pooling row：本路径是**运行期分数坐标 + 权重 + 数据相关邻居选择**（affine 变换后的 x/y 坐标决定 4 邻居），非固定 stencil tap；行内互斥「固定 stencil tap → spatial-convolution」不成立；`bilinear_interpolation` 显式带 distx/disty 权重合同 → 归 resampling
- Indexed Gather row（无插值语义）：4 邻居 load 后紧跟 bilinear 插值 FMA/权重合同，非任意 LUT gather；行内互斥「带 interpolation 坐标/权重合同 → resampling row」→ 排除
- Layout/Color Conversion rows：插值对象是 a8r8g8b8 像素的四个通道，但主导合同是 4-tap 插值而非颜色矩阵/chroma/位域重排 → 排除
- No vectorization row：行内互斥「operator semantic shape → 各自更具体 row」；resampling 是更具体语义 row → 排除
- Kernel Selection row：pixman-rvv.c `rvv_fast_paths[]` 无任何 bilinear/affine fetch 实现，「已有专用实现未选中」不成立 → 排除（对比：sse2/mips 均有 SIMPLE_BILINEAR_FAST_PATH，RISC-V 缺失）
- Hot-Helper Forced Inlining row：fetch 经 `image->fetch_pixel_32` 函数指针间接调用，dispatch 合同禁止内联 → 排除
- Register Pressure and Save/Restore row：prologue 1637c–1638a 占比 ≈0.1%，大栈帧（432B）无热点证据 → 排除
- Loop Induction Variable Strength Reduction row：1672c–16744 已是 `addw` 指针/坐标递增（归纳变量已强度削减），剩余份额属分派与分支控制 → 排除

**(c) 双 Confidence 推导式**：
- route：源码 provenance（pixman-bits-image.c:475 + pixman-inlines.h:130 64-bit bilinear）+ 反汇编语义（distx/disty 权重、`0xff0000ff`/`0xff00` mask、4 个 64-bit mul）+ 坐标/index 合同（affine 变换 pixman_fixed_t 分数坐标 → `sraiw`/`addiw` 邻居坐标生成）+ hardware V → **High**
- impact：局部样本份额可得（计算块 52.9% + fetch 调用点 21.2% ≈74.1% 受向量化 bilinear scanline kernel 影响）但缺 `baseline_gap: sampling metadata`（local period）+ `baseline_gap: bound type` → **Low**（只能讨论方向，不得估算收益幅度）

### 多命中仲裁
仅 1 个顶层 finding（primary：RVV Resampling Kernels）。4× fetch 调用点（≈21.2%）与 per-scanline 分派/循环（≈17.8%）是同一 per-pixel 标量 resampling 架构的症状：专用向量化 bilinear scanline kernel（含 index/weight 预计算与向量插值）会同时消除调用点、分派与插值链；它们无独立 row gate（见互斥排除 5-8），并入 primary evidence。入口模式 A：finding 的 evidence sample share 函数内加总 ≈0.74；workload 级贡献未知。

## Phase 4 — Root-cause blueprint / 根因蓝图：_bits_image_fetch_affine_no_alpha
（依据 `patterns/rvv_resampling_kernels.md`；对应 Phase 3 通过 gate 的 row：RVV Resampling Kernels）

1. **Root cause**：机制核心句短引（§Why this is slow）：「逐输出重复坐标计算、数据相关地址导致 scalar gather、bilinear/cubic 权重与边界处理割裂成多遍」。affine fetch 迭代器对每个输出像素执行：filter 分派 + 4 次 `fetch_pixel_32` 函数指针间接调用（数据相关邻居 load，≈21.2%）+ 64-bit 通道并行标量 bilinear 插值（pixman-inlines.h:130 变体：每个 8-bit 通道移入 64-bit lane，4 个 `mul` + shift/mask 组合，≈52.9%）+ per-scanline 分派/循环（≈17.8%）。整个 hot 路径 zero `v*`；hardware 有 RVV 1.0 + VLEN 256，build 无 v（L0），且 pixman-rvv.c 无任何 bilinear/affine fetch 实现。
2. **The fix / 修复方式**：
   - 前置（L0）：启用 meson `rvv` 选项（meson.options:67）使 USE_RVV 生效、pixman-rvv.c 编入 binary（pixman-riscv.c:125-130）。
   - 新增专用 RVV bilinear scanline fetch kernel（可挂到 rvv_fast_paths 的 SIMPLE_BILINEAR_FAST_PATH 或 affine fetch 迭代器，参考 sse2/mips 既有 `SIMPLE_BILINEAR_FAST_PATH (SRC, x8r8g8b8, ...)` 表项结构，pixman-fast-path.c:2650-2690 的 bilinear 组合函数形态）：
     ```
     // before（每输出像素）：
     //   4× 间接调用 fetch_pixel_32（fetch_pixel_x8b8g8r8 等）
     //   + 64-bit 通道并行标量插值：4× mul + ~30 条 shift/mask/or（16fba–170ce）
     // after（每行，VLEN=256, SEW=16/32, LMUL=8）：
     //   1) 坐标/权重预计算（pattern §6）：per-scanline 生成 x0/x1 邻居坐标与水平权重向量
     //      —— 沿用 affine 步长 vx/vy 的归纳更新，消除逐像素坐标重算
     //   2) 邻居像素行 load：按 repeat 边界 clamp/reflect 后 vle32 加载两行（或 vluxei32 indexed load，pattern §7）
     //   3) 向量插值：SEW=16 lane 展开各通道，vwmaccu 式 8→16-bit 乘加
     //      v = (tl&0xff00ff)*distixiy + (tr&0xff00ff)*distxiy + (bl&0xff00ff)*distixy + (br&0xff00ff)*distxy（hi 半区同理）
     //      或直接 4× vluxei32 + vfmacc（pattern §7 浮点形态，需保持 pixman 定点舍入合同）
     //   4) vsrl/vnclipu 窄化回 8-bit + vor alpha 通道（0xff）→ vse32
     ```
   - 适用前提：build 启用 RVV；坐标/权重合同（half-pixel 对齐 `x - pixman_fixed_1/2`、`pixman_fixed_to_bilinear_weight`、BILINEAR_INTERPOLATION_BITS=8 定点）保持不变。
   - correctness contract：必须保持 pixman-inlines.h:94/130 的**精确定点算术顺序与舍入**（`(lo >> 8) & 0xff00ff | hi & ~0xff00ff`，无附加 rounding bias）；repeat 模式（NONE/PAD/REFLECT/NORMAL）边界映射（pixman-bits-image.c:122-140 与 16c8e–16d42 区间的 remw/not/remw 序列）；odd width tail；alpha 合同（x8 格式 → 0xff）。
   - 限制/风险：RVV 整数 widen-MAC 与 64-bit channel-parallel 定点计算的等价性需要逐位验证（算术顺序合同）；indexed load（vluxei32）在 OoO 核上的吞吐；index/EMUL 与 register budget（pattern §8 风险）；小宽度退化需保留 scalar fallback（pattern §6「短图像或单次调用应比较预计算和即时计算」）。
   - 预期 Profile 信号：16fba–170ce 插值链与 16f56–16fb6 调用点消失；出现预计算 index/weight 向量 load、`vle32`/`vluxei32`、`vwmaccu`/`vfmacc`、`vsrl`/`vnclipu`、`vse32`；per-scanline 分派（1648c–164c6）与 loop bottom（1672c–16744）随专用 kernel 消失。
3. **Baseline facts 回填**：hardware ISA=SpacemiT X100 RVV 1.0；build ISA=rv64i2p1…zcd1p0（无 v）；VLEN=256；bound type=`baseline_gap: bound type`。
4. **收益上界**：当前 sampled event（cycles）下函数内局部份额——插值计算块 ≈52.9% + fetch 调用点 ≈21.2% ≈ **74.1%**（含分派/循环的全函数约 91.9% 受 per-pixel 标量 resampling 架构影响）；因 percent type=local period、函数 workload 贡献未知（`baseline_gap: sampling metadata`），**不得表述为 workload 级 Amdahl 上界**。
5. **三维路由判定**：`current source`=compiler-generated C（pixman-bits-image.c:475 `__bits_image_fetch_affine_no_alpha`）；`implementation existence/reachability`=pixman-rvv.c 无 fetch/bilinear 实现（rvv_fast_paths 3023-3093 仅 composite ops），本 binary build 无 v 使既有 RVV 实现不可达；`function-level policy`=无 assembly-default policy（RVV 载体为 intrinsic），不进入 policy-backed missing `.S` 分支。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs 小节**（来自 `patterns/rvv_resampling_kernels.md` §Related PRs）：
   - MNN：https://github.com/alibaba/MNN/pull/4053 、https://github.com/alibaba/MNN/commit/f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d
   - Related PRs：2 条 URL

## Phase 5 — Verification forecast / 验证预测：_bits_image_fetch_affine_no_alpha
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：插值计算块（`5.52 : 170b6: and a6,a4,a6`、`5.00 : 17076: srli a1,a1,0x20`、`4.42 : 170c6: ld a3,-280(s0)`、`2.43 : 17032: and a5,a4,t3`、`1.71 : 17048: mul a5,a5,t0`、`1.71 : 17050: mul a6,a6,t5`）与 4× fetch 调用点（`2.36 : 16f78: mv a2,s8`、`2.41 : 16f9a: mv a0,s7`、`2.19 : 16fa4: ld a1,-288(s0)`）应消失或显著缩小；分派/循环（`2.70 : 164ac: lw a4,68(s7)`、`2.07 : 16744: bne a5,s3,1648c`）应随专用 kernel 消失。
- 应出现侧（锚定 `patterns/rvv_resampling_kernels.md` §Verification「成功：逐 lane coordinate/index arithmetic 与 scalar loads 下降，出现预期 indexed vector loads，cycles/output 改善」）：预计算 index/weight 向量、`vle32`/`vluxei32`、`vwmaccu`/`vfmacc`、`vsrl`/`vnclipu`、`vse32` 出现在新 kernel annotate；scalar 插值链与间接调用消失。
- 验证步骤：启用 RVV 重建 → 重跑 lowlevel-blt-bilinear-117-117 → 对 `__bits_image_fetch_affine_no_alpha` 重跑 perf annotate；用 checkasm/pixman test suite 验证定点插值逐位一致（arithmetic order 合同）；覆盖 repeat NONE/PAD/REFLECT/NORMAL、odd width、half-pixel 对齐与负坐标（pattern §Verification 正确性合同清单）。短/长宽度均需验证（预计算流量 vs 即时计算拐点）。
- 收益上界仅 1 个 primary，无排序问题。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：1/1 组；[_bits_image_fetch_affine_no_alpha]） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 表 + 2 个 L0 gate + bound-type gate；gap 标签：[baseline_gap: bound type, baseline_gap: sampling metadata, baseline_gap: sampling IP precision]；含 Sampling IP precision 行） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding 1 个 = RVV Resampling Kernels，evidence 锚点 170b6/17076/170c6/17032/17048/17050/16f78/16f9a/16fa4；supporting 0；排除 8 条；推导式 2 条） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern 文件 patterns/rvv_resampling_kernels.md；命中 row=RVV Resampling Kernels；引用短语「逐输出重复坐标计算、数据相关地址导致 scalar gather」；The fix 含 before/after 伪代码、correctness contract（定点算术顺序）、限制/风险与预期 Profile 信号；Related PRs：2 条 URL） | ✅ |
| 5 | 路径合规（载荷：入口模式 A profile_backed；primary 唯一；8 项 trace 可解释；L0 未停扫） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 16fba–170ce/16f56–16fb6/1648c–164c6/1672c–16744；出现侧 rvv_resampling_kernels.md §Verification） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问、无代码修改、无补丁生成；止于证据/蓝图/The fix/验证预测） | ✅ |

修正记录：无