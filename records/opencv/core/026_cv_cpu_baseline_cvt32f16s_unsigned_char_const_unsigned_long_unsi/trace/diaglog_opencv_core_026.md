Functions under analysis: [cv::cpu_baseline::cvt32f16s(unsigned char const*, unsigned long, unsigned char const*, unsigned long, unsigned char*, unsigned long, cv::Size_<int>, void*)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（rank 026，498 samples，event=cpu-clock，percent type=local period，覆盖 hot loop）
- perf stat（可选 bound/context）：已提供（IPC=0.9176、L1_dcache_load_miss_rate=1.359%、branch_miss_rate=1.673%）
- workload/binary/DSO/source context：已提供（libopencv_core.so.5.1.0，含 DWARF；源码 `modules/core/src/convert.simd.hpp`、`convert.dispatch.cpp`）
- readelf -A：已提供（metadata：`rv64i2p1_..._v1p0_...zvl128b`，含 `v`）
- hardware ISA：已提供（SpacemiT X100/K3，`rv64imafdcvh_...`，RVV 1.0，VLEN=256）
- `vlenb`：已提供（VLEN=256 bits，vlenb=32）
- 采样元数据：已提供（cpu-clock；local period；498 samples 单窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（`baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcvh_..._zve64d_zvfh...` — 暴露标准 `v`（RVV 1.0），含 Zfa/Zfh/Zvfh 等浮点扩展 |
| Build ISA | 已提供：`rv64i2p1_..._v1p0_..._zvl128b1p0` — 含 `v`（RVV 1.0），min VLEN=128，与硬件 VLEN=256 兼容 |
| Vector flavor | annotate hot loop **zero `v*`、zero `th.v*`**（全 scalar：`flw`/`lrintf@plt`/`saturate_cast` 序列/`sh`）→ 与硬件/build 的 `v` 能力形成反差的未向量化路径 |
| VLEN | 256 bits（vlenb=32）；本函数未用向量寄存器 |
| Bound type | 整体 IPC=0.9176、L1 miss 1.36%、branch miss 1.67%；本函数 ~97% sample 在逐元素标量转换循环（`lrintf@plt` PLT 调用 + `saturate_cast` 分支 + `sh` 存储）→ 函数级 scalar-conversion/helper-call 主导 |
| Sampling semantics | cpu-clock；local period；同一窗口；函数级 workload 贡献未知 → 收益上界仅函数内局部份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（cpu-clock 非精确；单行占比只锚定 loop interval） |

L0 baseline gate：hardware 有 `v` 且 build 有 `v`（一致，无 build/hardware mismatch）→ 不构成 L0 mismatch。**但源码级编译配置 gate 命中**：`convert.simd.hpp:156-157` 的 `#if (CV_SIMD || (CV_SIMD_SCALABLE && !(defined(__GNUC__) && !defined(__clang__))))` 显式将 GCC 排除在 CV_SIMD_SCALABLE 之外（注释 `opencv/issues/26936`），导致该函数 SIMD 路径在 GCC+RVV 构建下被整体编译掉。这是本函数的决定性 finding（L0-adjacent config/code-path 层）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`cv::cpu_baseline::cvt32f16s(...)`。

Hot loop / loop interval：`0x177d0a – 0x177d5c`（float32→int16 转换标量循环，源码 `convert.simd.hpp:172-173` `for( ; j < size.width; j++ ) dst[j] = saturate_cast<_Td>(src[j]);`，外层 `for i < size.height`）。最高占比行原文：
- `29.52 :   177d4e: sext.w  a5,a0`（cvRound 结果的符号扩展，saturate_cast 输入准备）
- `24.70 :   177d40: addi    s10,s10,2`（dst 指针递增）
- `20.28 :   177d56: sh      a0,0(s10)`（int16 存储）
- `7.63 :   177d1e: sext.w  a5,a0`
- `5.62 :   177d0a: mv      s11,s4` / `4.82 :   177d10: mv s10,s5`
- `0.00 :   177d16/177d46: jalr lrintf@plt`（**每元素调用 libm lrintf**）

annotate 覆盖完整（hot loop body 完整）。Sampling IP precision 不足 → 单行占比只锚定 loop interval 与区间内指令组合。

## Phase 3 — Pattern scan / 模式扫描：cv::cpu_baseline::cvt32f16s(...)

### Class selection trace
1. `rows-asm.md` — exclude：compiler-generated C++（convert.simd.hpp DEF_CVT_FUNC 模板），非手写 `.S`；无 missing-`.S` 四证（函数走 universal-intrinsics 模板而非独立 `.S` 契约）
2. `rows-operator-rvv.md` — include（转换语义核对）：命中 RVV Precision Conversion Kernels row（float→int16 rounding/conversion，profile 主导在 scalar helper `lrintf@plt` 与逐 lane 饱和处理）；no-vectorization 作为 supporting（main loop 全 scalar、zero `v*`、硬件有 `v`，但被更具体的 conversion semantic row 认领）
3. `rows-string-memory.md` — exclude：非 string/memory
4. `rows-vectorized-tuning.md` — exclude：annotate **无 `v*`**（非已向量化），不满足该 class 入口
5. `rows-codegen.md` — include（compiler 配置形态）：检查 kernel-selection/dispatch —— 本函数命名空间为 `cv::cpu_baseline::`，SIMD 路径被编译期 `#if` 排除而非运行期 dispatch 未选中 → kernel-selection row 不命中（非 dispatch 接线问题，是源码编译配置问题，由 conversion 语义 row 的 root cause 承载）；cache-blocking/IV 等无证据 → 无独立命中
6. `rows-offload.md` — exclude：无矩阵引擎
7. `rows-crypto.md` — exclude：非密码学
8. `rows-runtime-os.md` — exclude：非 RTOS/kernel

Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`

### Local performance pattern scan: `cv::cpu_baseline::cvt32f16s(...)`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Precision Conversion Kernels（primary） | hot loop 全 scalar，每元素 `flw` + `lrintf@plt` + `saturate_cast`（sext.w/bltu/sgtz/negw/xor）+ `sh`；源码 `convert.simd.hpp:156-157` `#if` 将 GCC 排除在 CV_SIMD_SCALABLE 外（issues/26936）→ 转换 SIMD 路径编译期失效 | High | Medium（缺函数级 perf stat） | `patterns/rvv_precision_conversion_kernels.md` |
| No vectorization（supporting） | main loop 全 scalar（flw/lrintf/sh），hardware 与 build 均含 `v`，zero `v*`/`th.v*`；被更具体的 conversion semantic row 认领 | — | — | `patterns/no-vectorization.md` |

### 顶层命中 row 三件套

**Finding 1 — RVV Precision Conversion Kernels（primary）**
(a) 逐字 evidence 引用：
- `0.00 :   177d16/177d46: jalr lrintf@plt`（每元素 libm lrintf PLT 调用，loop interval 0x177d0a–0x177d5c）
- `29.52 :   177d4e: sext.w  a5,a0`、`24.70 :   177d40: addi s10,s10,2`、`20.28 :   177d56: sh a0,0(s10)`、`0.00 :   177d22/177d52: bltu/bgeu` 饱和分支
- 源码证据：`convert.simd.hpp:156-157` `#if (CV_SIMD || (CV_SIMD_SCALABLE && !(defined(__GNUC__) && !defined(__clang__))))` 注释 `// Excluding GNU in CV_SIMD_SCALABLE because of "opencv/issues/26936"` → GCC 构建下转换 SIMD 路径整体编译掉
(b) 互斥邻居排除：
- 非 `eliminate_unnecessary_precision_conversions`：float32→int16 是 API 必要 dtype 转换（cvRound+saturate_cast 语义），不是可删除的精度往返。
- 非 `rvv_register_group_utilization` / `rvv_vector_state_management`：hot loop 无 `v*`、无 vsetvli，不存在 LMUL/vtype 配置问题。
- 非 `kernel_selection_and_runtime_specialization`：问题不是运行期 dispatch 选错 kernel，而是编译期 `#if` 使 SIMD 路径不存在；本函数命名 `cv::cpu_baseline::` 印证执行的是 baseline 标量实现。
(c) 双 Confidence 推导式：
- route: scalar 转换循环 + 源码编译配置 gate（GCC 排除）+ hardware/build 均支持 RVV 转换 → High
- impact: 缺函数级 perf stat、cpu-clock skid → Medium

**Finding 2 — No vectorization（supporting）**
(a) 逐字 evidence 引用：hot loop 0x177d0a–0x177d5c 全 scalar（`flw`/`lrintf@plt`/`sh`），zero `v*`；hardware `rv64imafdcvh...` 有 `v`。
(b) 互斥邻居排除：该 row 是 operator class 最不具体成员，仅在其他具体语义 row 均不命中时认领；本函数已被 RVV Precision Conversion Kernels 认领（conversion 语义主导）→ no-vectorization 只作 supporting，不另立顶层。
(c) 双 Confidence：—（supporting 不计）。

### 多候选仲裁小段
- primary = RVV Precision Conversion Kernels（conversion 语义主导 + 源码编译配置根因），supporting = No vectorization（同一 hot loop 的 scalar 形态，被 conversion row 认领）。单一热区、单一机制，无独立/companion 拆分。
- 入口条件 A：primary evidence sample share 加总 = 标量转换循环内可归因指令 ≈ 0.9738（29.52+24.70+20.28+7.63+5.62+4.82+1.61+1.00+0.60+0.40+0.20）；表述为「当前 sampled event 下函数内局部样本份额」。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::cpu_baseline::cvt32f16s(...)

**命中 row（通过 gate）**：`rows-operator-rvv.md` → `rvv_precision_conversion_kernels.md`（primary，route High/impact Medium）；supporting：`no-vectorization.md`。

1. **Root cause**：`cvt32f16s`（float32→int16）的 SIMD 转换路径在 GCC+RVV 构建下被编译期排除。`convert.simd.hpp:156-157` 用 `#if (CV_SIMD || (CV_SIMD_SCALABLE && !(defined(__GNUC__) && !defined(__clang__))))` 显式把 GCC 踢出 CV_SIMD_SCALABLE（注释指 `opencv/issues/26936`），因此即使在含 `v` 的 RVV 构建中，该转换函数也只生成标量循环：每元素 `flw` → `lrintf@plt`（libm 函数调用）→ `saturate_cast<short>`（sext.w/bltu/sgtz/negw/xor 饱和序列）→ `sh`。依据 `patterns/rvv_precision_conversion_kernels.md` §Why this is slow："逐元素 helper 放大固定开销… 对长数组，转换本身会成为主热点"，且 §The fix 明确指出浮点到整数 rounding 可用 `vfcvt.*` 向量路径 + FRM/rounding mode 与 mask 修正建立 vector fast path。

2. **The fix / 修复方式**：
   - 修正对象：`modules/core/src/convert.simd.hpp:156-157` 的 `#if` 条件（`CV_SIMD_SCALABLE` 分支对 GCC 的排除），以及对应的 `DEF_CVT_FUNC`/`cvt_` 模板转换路径。
   - 修复方向（primary）：在 RVV-capable 构建中恢复 `cvt32f16s` 的向量 fast path —— 解除/收窄 GCC 排除（修复或规避 issues/26936 的根因），或在 GCC 下提供等价 RVV 转换路径。向量形态示意（依据 pattern §The fix）：
     ```
     // Before（当前）：每元素标量
     for (j=0; j<width; j++) dst[j] = saturate_cast<short>(src[j]);  // flw + lrintf@plt + 饱和分支 + sh

     // After：向量转换 fast path + 特殊值窄修复（示意）
     vfloat32m2_t f = vle32_v_f32m2(src+j, vl);
     vint32m2_t  y = vfcvt_x_f_v_i32m2(f, vl);        // 需匹配 cvRound 的 rounding 语义
     vint16m1_t  d = vnclip_wx_i16m1(y, 0, vl);       // 窄化 + 饱和（saturate_cast）
     vse16_v_i16m1(dst+j, d, vl);
     ```
   - correctness contract：cvRound/lrintf 的 rounding（当前 FRM 语义）与 `saturate_cast<short>` 的饱和边界（SHRT_MIN/SHRT_MAX、NaN→0 等）必须逐元素一致；±Inf、NaN、±0、denormal、overflow/underflow 与 tie 语义保持 scalar reference 一致；必要时用 mask+slow path 修正特殊 lane。
   - 限制/风险：issues/26936 若涉及 GCC 代码生成 bug，解除排除前需复现验证；缺函数级 perf stat；需实测无 vsetvli 反复切换与 spill。
   - 修复后预期 Profile signals：主转换 loop 出现 `vfcvt.*`/`vncvt.*`/`vnclip` 而非逐元素 `lrintf@plt` + 饱和分支；`lrintf@plt`、`sext.w`、`sh` 的每元素成本消失/大幅下降；该函数 CPU 时间下降。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（有 `v`，含 Zfa/Zfh/Zvfh）；build ISA=`rv64i2p1_..._v1p0_...zvl128b`（含 `v`）；VLEN=256 bits；bound type=函数级 scalar-conversion/helper-call 主导（整体 IPC=0.9176、L1 miss 1.36%）。

4. **收益上界**：入口条件 A（profile-backed）。primary evidence sample share 加总 ≈ 0.9738（标量转换循环内可归因指令）；表述为「当前 sampled event（cpu-clock, local period）下函数内局部样本份额」，非 workload 级 Amdahl 上界。

5. **三维路由判定**：
   - current source：compiler-generated（DEF_CVT_FUNC 模板 cvt_ 的标量 fallback 被实际执行）
   - implementation existence/reachability：向量转换路径在源码中存在（`v_float32` universal-intrinsics 分支）但被 `#if` 编译期排除（GCC），运行期不可达
   - function-level policy：无独立 `.S` 政策证据；本路径属 compiler/config 调优，不进入 missing `.S` 分支

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**：
   - RVV Precision Conversion Kernels：`Related PRs：10 条 URL` — https://github.com/openjdk/jdk/commit/d7273ac8b1ad8bc5d0a17fff5dc941c735fdae24 ；https://github.com/openjdk/jdk/pull/17698 ；https://github.com/openjdk/jdk/commit/bcaad515fdedd0c41a719d2a88b2da3036c766a3 ；https://github.com/openjdk/jdk/commit/b3634722655901b8d3e43dd1f8aa2b4487509a34 ；https://github.com/openjdk/jdk/commit/a33b1f7f640e0a9e76d2a686734e472a87d809bf ；https://github.com/openjdk/jdk/commit/5eb8774909bd250c7ff8cfc56506a949b547bda2 ；https://github.com/openjdk/jdk/commit/92fd44992b9326fa10ec8303394dac17bb81b168 ；https://github.com/openjdk/jdk/commit/bacd046062bffb4c95ec7a508a1080ad651a94a4 ；https://github.com/openjdk/jdk/pull/16382 ；https://github.com/v8/v8/commit/f67745295ddf52fe22793c63dbe1482e880aec11

## Phase 5 — Verification forecast / 验证预测：cv::cpu_baseline::cvt32f16s(...)
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：
  - `177d16/177d46: jalr lrintf@plt`（每元素 libm 调用）应消失。
  - `177d4e/177d1e: sext.w a5,a0`（29.52%+7.63%）、`177d56: sh a0,0(s10)`（20.28%）、`177d40: addi s10,s10,2`（24.70%）应大幅缩小（向量化后每元素成本摊薄）。
- 应出现侧（锚定 pattern §Verification）：
  - 按 `rvv_precision_conversion_kernels.md` §Verification：disassembly/perf annotate 确认主转换 loop 出现 `vfcvt.*`/`vfncvt.*`/`vnclip` 而非逐元素 helper/bit-twiddle；scalar reference 逐 lane 对比覆盖 +0/-0/NaN payload/±Inf/denormal/overflow/underflow/rounding mode/tie 与短数组；hot loop 内 vsetvli 不反复切换、无 spill。
  - 对真实 workload（OpenCV core perf，CV_32F→CV_16S 转换用例）重跑 benchmark，对比该函数 CPU 时间与函数内 sample share；确认 scalar helper 与饱和分支占比下降。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 全部出现（载荷：`1/1 组；cv::cpu_baseline::cvt32f16s(...)`） | ✅ 1/1 组；cv::cpu_baseline::cvt32f16s(unsigned char const*, unsigned long, unsigned char const*, unsigned long, unsigned char*, unsigned long, cv::Size_<int>, void*) |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 2 个 L0 gate + 源码级编译配置 gate + bound-type gate；含 Sampling IP precision 行） | ✅ 7 行；gap 标签：`baseline_gap: sampling IP precision`；无 th.v*/build mismatch |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；`Classes scanned:` 2 文件；顶层 finding=1（precision conversion）+supporting=1（no-vectorization）；evidence 锚点；exclusion 逐条） | ✅ 8 项 trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md |
| 4 | Phase 4 输出要求满足（已读 pattern 文件 + 命中 row + 引用短语；The fix 含 before/after、correctness、风险、Profile signals；Related PRs：10 条 URL） | ✅ 已读 patterns/rvv_precision_conversion_kernels.md |
| 5 | 路径合规：入口模式 A；primary/supporting 与 L1 层归属合规；blueprint leaf 均来自通过 gate 的 row；无 th.v* 停扫 | ✅ 模式 A + L1(vectorization/semantic dispatch) + class 列表 |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3 引用行（177d16/177d46/177d4e/177d56/177d40），出现侧标注 pattern §Verification | ✅ 锚点：jalr lrintf@plt、sext.w、sh、addi s10 |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成；交付物止于 Profile 证据、根因蓝图、The fix 与验证预测 | ✅ |

修正记录：无