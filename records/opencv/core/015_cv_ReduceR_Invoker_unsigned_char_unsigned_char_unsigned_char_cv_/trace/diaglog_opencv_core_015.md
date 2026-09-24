Functions under analysis: [cv::ReduceR_Invoker<unsigned char, unsigned char, unsigned char, cv::OpMin<unsigned char>, cv::OpNop<unsigned char, unsigned char, unsigned char> >::operator()(cv::Range const&) const]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 015，`ReduceR_Invoker<uchar,...,OpMin<uchar>,OpNop<...>>::operator()`，event=`cpu-clock`，730 samples，percent: local period；覆盖逐行 min 归约主循环 [0x243f90..0x2440a6]、buf 初始化、dst 写出）
- perf stat：已提供（`7-opencv-perf-benchmark-riscv-core.txt`；IPC 0.9176、L1_dcache_load_miss_rate 1.359%、branch_miss_rate 1.673%）
- workload/binary/DSO/source context：已提供（libopencv_core.so.5.1.0，OpenCV 5.x，annotate 含 C++ 源码行号；`cv::reduce` REDUCE_MIN 行向归约的 invoker）
- readelf -A（热点 object 的 Tag_RISCV_arch）：已提供（metadata 快照 `opencv_perf_core-elf-A`：`rv64i2p1_..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0...`）
- hardware ISA：已提供（metadata 快照：SpacemiT X100，`rv64imafdcvh_...`，含 `v`、`zve64d`、`zvbb/zvbc/zvk*` 等）
- vlenb：已提供（metadata vector：VLEN=256 bits，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock；percent type=local period；单次运行同一窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（cpu-clock、precise_ip 未知 → `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`：暴露标准 `v`（RVV 1.0）、`zve64d`、`zvbb/zvbc/zvk*` |
| Build ISA | `Tag_RISCV_arch` 含 `v1p0`、`zve32f/zve64d/zve64f`、`zvl128b`：object 目标架构含 V；无 hardware/build mismatch |
| Vector flavor | 整个函数 annotate 无任何 `v*`（纯 scalar `lbu`/`subw`/`addiw`/`sb`），也无 `th.v*` |
| VLEN | 256 bits（vlenb=32）；`VLMAX(e8m1)=32` |
| Bound type | IPC 0.918、L1_dcache_load_miss_rate 1.359%、branch_miss_rate 1.673% → compute/latency-bound（非 memory-bound） |
| Sampling semantics | event=cpu-clock；percent type=**local period**；同一运行窗口；函数级 workload 贡献未知 → 只能表述局部样本份额，`baseline_gap: sampling metadata` |
| Sampling IP precision | `baseline_gap: sampling IP precision`（cpu-clock、precise_ip 未知）→ 单指令高占比只锚定 loop interval，不做单指令 latency 归因 |

L0 baseline gate：hardware 有 `v` 且 build 有 `v`（RVV 1.0），无 mismatch；annotate 无 `th.v*`；**硬件/build 均支持 RVV，但本函数执行代码 zero `v*`** —— 作为本函数最高优先级 baseline 事实置顶，不停止扫描。
Bound-type gate：compute/latency-bound，本地向量化修复是第一杠杆；无 memory-bound 竞争 bottleneck。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）：`cv::ReduceR_Invoker<unsigned char, unsigned char, unsigned char, cv::OpMin<unsigned char>, cv::OpNop<unsigned char, unsigned char, unsigned char> >::operator()`（rank 015，OpenCV `cv::reduce` REDUCE_MIN 的逐行最小归约：对每一列维护跨行的 running min）。

Hot interval 锚点（cpu-clock，local period）：
- **逐行 min 归约主循环 `[0x243f90..0x2440a6]` 持有函数内 ≈99% 样本**：4 路 unroll 段 `[0x243fe6..0x24406a]` 合计 ≈99.0%，最高行 `244002: addiw a3,a3,256`（13.70%，CV_MIN_8U 差值偏置）、`243ff6: lbu t0,3(a5)`（11.51%，buf load）、`243ffa: subw a3,t2,a3`（8.49%，buf-src 差值）、`244012: lbu a4,0(a4)`（7.40%，min tab 查表）。
- buf 初始化 `[0x243f90..0x243fa8]` 与 dst 写出 `[0x2440a8..0x2440c6]` 均为 0.00–0.14%。

结论：**整个行向 min 归约内核（含全部热循环）都是逐 byte 的 scalar 实现，zero `v*`**；每元素执行 `lbu`（buf+src 双 load）+ `subw`+`addiw +256`+`add`（tab 基址）+ `lbu`（min 查表）+ `subw`+`sb`，即 OpenCV `CV_MIN_8U` 的查表实现。annotate 覆盖完整，入口条件 A（profile_backed）。Sampling IP precision 不足 → 锚定 scalar 主循环 interval 整体。

## Phase 3 — Pattern scan / 模式扫描：cv::ReduceR_Invoker<...OpMin...>

### Class selection trace
1. `rows-operator-rvv.md` — include：行向 min 归约是"只维护 best/min/max value、不返回 index"的 extrema-reduction 语义（`rvv_extrema_reduction_kernels` row）；并核对 no-vectorization 与其互斥。
2. `rows-codegen.md` — include：compiler-generated scalar 循环的查表/差值偏置形态逐 row 核对（预期为 OpMin 的参考实现语义，非可化简 codegen 反模式）。
3. `rows-vectorized-tuning.md` — include 后 exclude：annotate 无 `v*`，修正对象不是已向量化 RVV 的配置（rows-vectorized-tuning 前提"已有 v*"不成立）。
4. `rows-string-memory.md` — include 后 exclude：非 copy/fill/sentinel/compare 语义（min 归约是 reduction，非字符串/内存原语）。
5. `rows-asm.md` — exclude：非手写 `.S`（compiler-generated C++ 模板）；无 missing-`.S` 四证。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 证据。
7. `rows-crypto.md` — exclude：非密码学原语（查表是 CV_MIN_8U 参考实现，非 crypto polynomial LUT）。
8. `rows-runtime-os.md` — exclude：用户态 OpenCV 库函数。

### Classes scanned: rows-operator-rvv.md, rows-codegen.md（rows-vectorized-tuning.md、rows-string-memory.md 前提检查后排除）

### Local performance pattern scan: `cv::ReduceR_Invoker<...OpMin<unsigned char>...>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Extrema Reduction Kernels（primary） | 行向 min 归约（running min per column），scalar CV_MIN_8U 查表循环持有 ≈99% 函数内样本，value-only 无 index | High | Medium | `patterns/rvv_extrema_reduction_kernels.md` |
| No vectorization (autovec gap / RVV kernel not built)（supporting） | 全函数 zero `v*`，硬件 `v`+build `v1p0` | Medium | — | `patterns/no-vectorization.md` |

**RVV Extrema Reduction Kernels（primary）三件套**

(a) 逐字 evidence 引用（逐行 min 归约主循环 `[0x243fe6..0x24406a]`，cpu-clock local period）：
```
13.70 : 244002: addiw   a3,a3,256        <- CV_MIN_8U 差值 +256 偏置
11.51 : 243ff6: lbu     t0,3(a5)         <- buf[i+3] load
 8.49 : 243ffa: subw    a3,t2,a3         <- buf - src 差值
 7.40 : 244012: lbu     a4,0(a4)         <- min tab 查表
 6.85 : 243ff2: lbu     t6,1(a5)         <- buf[i+1] load
 4.66 : 244016: addiw   t4,t4,4          <- i+=4 循环控制
 4.38 : 244034: subw    a3,t0,a3         <- 第二组差值
 2.05 : 24400e: lbu     a3,0(a3)         <- min tab 查表
```
语义：`buf[i] = min(buf[i], src[i])`（OpMin<uchar> = `CV_MIN_8U(a,b)`，源码行 251），跨行 running min 存回 `buf`；value-only，无 index。整个函数无 `v*`。

(b) 互斥邻居排除：
- `RVV Arg-Extrema Selection`（rows-operator-rvv）：排除——该 row 要求"同时维护 best value 与 corresponding index"；本归约只写回 min value（`sb t2,0(a5)`），不返回 index。
- `RVV Widening Additive Reduction`（rows-operator-rvv）：排除——min 是格序归约（idempotent, no widening），非加性累加；row 行内互斥"只返回 Min/Max value → extrema row"。
- `No vectorization`（rows-operator-rvv）：排除为 primary——该 row 行内互斥"operator semantic shape → 各自更具体 row"；本函数热点是 min-reduction 语义，由 `rvv_extrema_reduction_kernels` 认领 primary，no-vectorization 作 supporting。
- codegen rows（register-pressure/induction-var/control-flow/lookup）：排除——`subw`+`addiw +256`+`add`+`lbu` 是 `CV_MIN_8U` 参考实现的查表语义（OpenCV 对 uchar min 的标准实现），非可化简的冗余代码；无 spill/归纳变量问题。
- 排除合计 4 条 + 整组排除 6 个 class。

(c) 双 Confidence 推导式：
- route：min-reduction 语义直接可证（source `CV_MIN_8U(a,b)`）+ value-only 无 index（annotate `sb` 写回无索引维护）+ hardware `v`/build `v1p0` 确认向量化能力就绪 + 无更具体 row 认领 → **High**。
- impact：hot interval sample share（主循环 ≈99% 函数内局部）+ VLEN 已知（256）+ bound type 已知（compute/latency）成立；但 sampling metadata 四条不齐（percent=local period、workload 贡献未知）→ 不可称 workload 级 Amdahl，**Medium**。

**No vectorization（supporting）三件套**

(a) 逐字 evidence 引用：整个函数（730 samples 覆盖的 `[0x243f90..0x2440a6]` 主循环）无任何 `v*` 指令；主循环由 scalar `lbu`/`subw`/`addiw`/`add`/`sb` 组成。
(b) 互斥邻居排除：kernel-selection（rows-codegen）——无证据表明 OpenCV 存在已注册未选中的 RVV reduce-min 实现；missing-`.S`（rows-asm 四证）——不成立，OpenCV 接受 intrinsic；extrema-reduction 已认领 primary（supporting 只解释缺少向量执行，不决定贡献载体）。
(c) 双 Confidence 推导式：route Medium（direct zero-`v*` evidence + 无 backend dump 证明 autovec 失败原因）；impact —（supporting 不单独评级）。

### 多候选仲裁小段
- primary = RVV Extrema Reduction Kernels；supporting = No vectorization。同一 hot loop、同一机制（scalar 逐 byte min 归约）：extrema-reduction 认领语义与修复对象（vredminu），no-vectorization 解释执行载体缺向量化；因果消除——按 primary 向量化后 zero-`v*` signal 自然消失 → no-vectorization 并入 supporting，不单列顶层 finding。
- Evidence-mechanism layer：L1 vectorization/semantic dispatch（`rvv_extrema_reduction_kernels.md`）。收益排序：入口条件 A，primary evidence sample share ≈0.99 函数内局部（保守 bound 0.95）。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::ReduceR_Invoker<...OpMin...>

### 1. Root cause

OpenCV 的 `cv::reduce` REDUCE_MIN 行向归约（`ReduceR_Invoker<uchar, OpMin<uchar>>`）在具备 RVV 1.0 能力（hardware `v`、build `v1p0`、VLEN=256）的目标上，整个逐行 min 归约主循环完全未向量化：每列跨行维护 running min，逐 byte 执行 `lbu`（buf+src 双 load）→ `subw`（buf-src）→ `addiw +256`（CV_MIN_8U 偏置）→ `add`（tab 基址）→ `lbu`（min 查表）→ `subw`（buf-tab）→ `sb`（写回），4 路 unroll 后主循环仍占 ≈99% 函数内样本。依据 `patterns/rvv_extrema_reduction_kernels.md` §Why this is slow："loop-carried scalar best-value accumulator 串行化 compare/update，并让每个输入元素承担分支与循环控制。主要 leverage point 是 per-chunk vector extrema reduction"；以及 `patterns/no-vectorization.md` §Why this is slow："vector unit 没有处理 hot main-loop 的并行元素……zero `v*` 只证明当前 loop 没有 RVV execution"。

### 2. The fix / 修复方式

依据 `patterns/rvv_extrema_reduction_kernels.md` §The fix §2（每块独立 reduction + 当前全局标量作 seed）与 §3（无符号整数归约用 `vredminu`）：

Before（当前 scalar 主循环，对应 annotate 0x243fe6..0x24406a）：
```cpp
for (; i <= range.end - 4; i += 4) {
    buf[i]   = CV_MIN_8U(buf[i],   src[i]);     // subw+addiw+add+lbu+subw+sb
    buf[i+1] = CV_MIN_8U(buf[i+1], src[i+1]);
    buf[i+2] = CV_MIN_8U(buf[i+2], src[i+2]);
    buf[i+3] = CV_MIN_8U(buf[i+3], src[i+3]);
}
```
After（blueprint：RVV e8m1 向量 min 归约，保持 uchar 无符号语义）：
```cpp
// 内层按列向量化：每个 e8m1 向量处理 32 列，跨行更新 running min。
size_t col = range.start;
for (; col + vl <= (size_t)range.end; col += vl) {   // vl = vsetvl_e8m1(...)
    vuint8m1_t s = vle8_v_u8m1(src + col, vl);
    vuint8m1_t b = vle8_v_u8m1(buf + col, vl);
    vuint8m1_t m = vminu_vv_u8m1(b, s, vl);          // uchar min（无符号，vredminu 的逐元素形态）
    vse8_v_u8m1(buf + col, m, vl);
}
for (; col < (size_t)range.end; ++col)               // 标量 tail，同一状态续接
    buf[col] = CV_MIN_8U(buf[col], src[col]);
```
适用前提：uchar 为无符号（`vminu`/`vredminu` 语义 = `CV_MIN_8U` 查表结果，逐元素 bit 一致）；`range.start..range.end` 为列区间；硬件 `v`+build `v1p0` 成立；外循环（行迭代 `for(;--height;)`）与 buf 初始化/dst 写出逻辑不变。

不可破坏的 correctness contract：**min 结果逐元素 bit 一致**（`CV_MIN_8U` 查表 == `vminu`，两者都返回 uchar min，无 NaN/舍入问题）；跨行 running min 的 seed/累积语义不变（每行向量 min 后写回 buf，下一行继续）；tail 列（`col` 之后的 `range.end-col < vl` 列）沿用 scalar `CV_MIN_8U`，状态续接一致；buf 类型为 WT（uchar），dst 写出 `(ST)buf[i]` 不变；range.start 非 0 时只处理指定列区间。

限制/风险：`CV_MIN_8U` 的查表实现与 `vminu` 数值等价（可先做对拍验证）；若 OpenCV 该模板存在 RVV 专属 hal 路径（rvv_hal::reduce 或 v_min 结构），应优先接入既有实现而非重写（`source_context_gap: 未核对该文件是否已有 RVV reduce 实现`）；行方向外循环每行一次向量化，收益随行数/列宽放大；短列宽（range.end-range.start < vl）时向量化收益有限，需 crossover 阈值。

预期 Profile signals：`244002 addiw +256`（13.70%）、`243ff6/243ff2/243fee lbu`（11.51/6.85/3.70%）、`243ffa subw`（8.49%）、`244012 lbu`（7.40%）等 scalar 查表/差值/双 load 指令大幅缩小或消失；主循环出现 `vsetvli`/`vle8.v`/`vminu.vv`/`vse8.v`；每元素指令数与 cycles 显著下降。

### 3. Baseline facts 回填

hardware ISA：`rv64imafdcvh_...`（含 `v`/`zve64d`/`zvbb/zvbc/zvk*`）；build ISA：`Tag_RISCV_arch` 含 `v1p0`、`zve64d`、`zvl128b`（无 mismatch）；VLEN=256 bits（vlenb=32，e8m1 VLMAX=32）；bound type：compute/latency-bound（IPC 0.918、L1 miss 1.36%、branch miss 1.67%）。

### 4. 收益上界

primary 的 evidence sample share：逐行 min 归约主循环 `[0x243f90..0x2440a6]` ≈99% 函数内样本。表述为「当前 sampled event（cpu-clock）下的函数内局部样本份额」；`baseline_gap: sampling metadata`（percent=local period、workload 贡献未知）→ 不得表述为 workload 级 Amdahl 上界。

### 5. 三维路由判定

- current source：compiler-generated C++（OpenCV `ReduceR_Invoker` 模板 + `CV_MIN_8U` 查表宏，annotate 源码行 361-385），非手写 `.S`。
- implementation existence/reachability：无证据表明存在已注册未选中的 RVV reduce-min 实现；`source_context_gap`——未核对 OpenCV 该路径是否已有 `rvv_hal` 或 universal-intrinsic 的 reduce 实现（若存在应走 kernel-selection 而非重写）。
- function-level policy：无"必须独立 `.S`"的 policy 证据（OpenCV 接受 intrinsic/HAL）→ 不走 missing-`.S` 分支，按 intrinsic/通用向量化修复。

### 6. Related PRs

patterns/rvv_extrema_reduction_kernels.md：Related PRs：5 条 URL（https://github.com/uxlfoundation/oneDNN/pull/5361；https://github.com/uxlfoundation/oneDNN/commit/a95f0060cfcb75aeee7b937f18dbb2ff32064f50；https://github.com/alibaba/MNN/pull/4036；https://github.com/alibaba/MNN/commit/09c339c65cb73dda7c055c3e49e9026dfd998e3e；https://github.com/alibaba/MNN/pull/4433）
patterns/no-vectorization.md：Related PRs：19 条 URL（https://github.com/opencv/opencv/pull/22179；https://github.com/opencv/opencv/pull/22520；https://github.com/opencv/opencv/pull/23980；https://github.com/opencv/opencv/pull/24058；https://github.com/opencv/opencv/pull/24132；https://github.com/opencv/opencv/pull/24166；https://github.com/opencv/opencv/pull/24301；https://github.com/opencv/opencv/pull/24325；https://github.com/opencv/opencv/pull/27160；https://github.com/opencv/opencv/pull/27119；https://github.com/opencv/opencv/pull/27097；https://github.com/opencv/opencv/pull/27007；https://github.com/opencv/opencv/pull/26958；https://github.com/opencv/opencv/pull/26865；https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d；https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a；https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706；https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f；https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a）

## Phase 5 — Verification forecast / 验证预测：cv::ReduceR_Invoker<...OpMin...>

primary（RVV Extrema Reduction Kernels）：
- 应消失/缩小（锚定 Phase 3(a) 引用行）：`244002: addiw a3,a3,256`（13.70%）、`243ff6/243ff2/243fee: lbu`（11.51/6.85/3.70%）、`243ffa: subw`（8.49%）、`244012: lbu`（7.40%）等 CV_MIN_8U 查表/差值/双 load 指令样本大幅下降或消失。
- 应出现（锚定 `patterns/rvv_extrema_reduction_kernels.md` §Verification）：主循环出现 `vsetvli`/`vle8.v`/`vminu.vv`/`vse8.v`；对拍 empty/单列/`vl` 整倍数/tail、全相等/极值在首尾、signed/unsigned 边界（uchar 无符号）、跨行多行归约，min 结果与 scalar `CV_MIN_8U` reference 逐元素 bit 一致；覆盖 `range.start!=0` 列区间与行数=1/多行；perf stat 复核 IPC/instructions（确认每元素指令数下降）。
- 验证说明：`vminu` 与 `CV_MIN_8U` 查表数值等价（整数 min 无 FP 语义问题）；无 NaN/±0 风险。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：`1/1 组；cv::ReduceR_Invoker<...OpMin...>`） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 含 Sampling IP precision；gap 标签：`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding 1 + supporting 1；evidence 锚点 `244002 addiw` 13.70%、`243ff6 lbu` 11.51%、`243ffa subw` 8.49%、`244012 lbu` 7.40%、`244016 addiw` 4.66%；排除 4 条 + 6 class 整组；推导式 2 条） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 patterns/rvv_extrema_reduction_kernels.md（命中 RVV Extrema Reduction Kernels row；引用「loop-carried scalar best-value accumulator 串行化 compare/update…per-chunk vector extrema reduction」；The fix §2/§3、before/after、correctness、风险、Profile 信号锚点齐全；Related PRs：5 条 URL）+ patterns/no-vectorization.md（supporting，Related PRs：19 条 URL）） | ✅ |
| 5 | 路径合规（载荷：模式 A profile_backed；include 2 class 全扫；L1 primary + L1 supporting；`th.v*` 无；hardware/build 均有 `v` 但执行代码 zero `v*` 作为 L0 finding 置顶） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `244002 addiw`/`243ff6 lbu`/`243ffa subw`/`244012 lbu`；出现侧 pattern §Verification 的 vsetvli/vminu/vse8/bit-一致验证） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问/无代码修改/无补丁生成；交付止于证据+根因蓝图+The fix+验证预测） | ✅ |

修正记录：无