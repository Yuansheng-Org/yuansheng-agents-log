Functions under analysis: [cv::ReduceR_Invoker<float, float, float, cv::OpAddSqr<float, float, float>, cv::OpSqr<float, float, float> >::operator()(cv::Range const&) const]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（rank 027，453 samples，event=cpu-clock，percent type=local period，覆盖 hot loop）
- perf stat（可选 bound/context）：已提供（IPC=0.9176、L1_dcache_load_miss_rate=1.359%、branch_miss_rate=1.673%）
- workload/binary/DSO/source context：已提供（libopencv_core.so.5.1.0，含 DWARF；源码 `modules/core/src/matrix_operations.cpp` ReduceR_Invoker/ReduceR_SIMD 模板）
- readelf -A：已提供（metadata：`rv64i2p1_..._v1p0_...zvl128b`，含 `v`）
- hardware ISA：已提供（SpacemiT X100/K3，`rv64imafdcvh_...`，RVV 1.0）
- `vlenb`：已提供（VLEN=256 bits，vlenb=32；annotate 内 `csrr s2,vlenb`）
- 采样元数据：已提供（cpu-clock；local period；453 samples 单窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（`baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcvh_..._zve64d_zvfh...` — 暴露标准 `v`（RVV 1.0） |
| Build ISA | 已提供：`rv64i2p1_..._v1p0_..._zvl128b1p0` — 含 `v`，min VLEN=128，与硬件 VLEN=256 兼容 |
| Vector flavor | annotate 显示 RVV 1.0 `v*` mnemonic（`vsetvli`/`vle32.v`/`vse32.v`/`vfmacc.vv`），无 `th.v*`，无 mismatch |
| VLEN | 256 bits（vlenb=32）；热循环 `vsetvli zero,a5,e32,m1,ta,ma`（242286）→ 每轮仅 8 元素/32 字节 |
| Bound type | 整体 IPC=0.9176、L1 miss 1.36%、branch miss 1.67%；本函数 76.38% sample 落在单条 `vle32.v v2,(s7)` src 流式 load 上 → 函数级 memory-streaming/latency-bound |
| Sampling semantics | cpu-clock；local period；同一窗口；函数级 workload 贡献未知 → 收益上界仅函数内局部份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（单行 76.38% 只锚定 loop interval，不做单指令 latency 归因） |

L0 baseline gate：hardware 有 `v` 且 build 有 `v`（一致）；无 `th.v*`。Bound-type gate：热循环 76.38% 在流式 load → memory-streaming-bound gate 生效，本地 compute/RVV 调优的 performance-impact confidence 下调；RVV config/LMUL 类 anti-pattern 单独定 confidence。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`cv::ReduceR_Invoker<float,float,float,cv::OpAddSqr<float,float,float>,cv::OpSqr<float,float,float>>::operator()(cv::Range const&) const`。

Hot loop / loop interval：`0x242286 – 0x2422a4`（row 归约内循环：`buf[i] += src[i]*src[i]`，OpAddSqr，外层 `for(; --height;)` 逐行累加）。最高占比行原文：
- `76.38 :   24228e: vle32.v v2,(s7)`（src 流式 load）
- `19.43 :   242292: mv      t6,s6`（循环控制/计数准备，cpu-clock skid 可能聚在瓶颈相邻行）
- `0.22 :   242286: vsetvli zero,a5,e32,m1,ta,ma`（每迭代仅 1 条，AVL 逐轮变化）
- `0.00 :   24228a/24229a/24229e`：`vle32.v v1`（buf load）、`vfmacc.vv v1,v2,v2`（FMA）、`vse32.v v1`（buf store）
- `0.88 :   2422a2: add t5,t5,a0`、`0.44 :   2422a4: bltu a2,t6`

annotate 覆盖完整。Sampling IP precision 不足 → 上述行只锚定 loop interval 与指令组合，不做单指令 latency 归因。

## Phase 3 — Pattern scan / 模式扫描：cv::ReduceR_Invoker<...OpAddSqr/OpSqr...>::operator()(...)

### Class selection trace
1. `rows-asm.md` — exclude：compiler-generated（GCC autovec 向量化 matrix_operations.cpp 模板 scalar 循环），非手写 `.S`；无 missing-`.S` 四证
2. `rows-operator-rvv.md` — include（已向量化语义核对）：elementwise 平方和归约已用 `vfmacc.vv` 直接 lowering，不满足任何要求 scalar hot path 的 operator row；widening-reduction row 的 scalar 语义不成立 → 无命中
3. `rows-string-memory.md` — exclude：非 string/memory
4. `rows-vectorized-tuning.md` — include（必选：有 `v*`，非手写 `.S`）：命中 register-group row（e32,m1 欠利用）；vector-state row 不命中（每迭代仅 1 条 vsetvli，AVL 逐轮变化，无重复兼容状态）
5. `rows-codegen.md` — include（compiler-generated 指令形态）：循环用指针递增（add/sub），无 index scaling 冗余；仅 1 条 vsetvli，无 loop-invariant mul；无 spill → 无命中
6. `rows-offload.md` — exclude：无矩阵引擎
7. `rows-crypto.md` — exclude：非密码学
8. `rows-runtime-os.md` — exclude：非 RTOS/kernel

Classes scanned: `rows-vectorized-tuning.md`、`rows-operator-rvv.md`、`rows-codegen.md`

### Local performance pattern scan: `cv::ReduceR_Invoker<...OpAddSqr/OpSqr...>::operator()(...)`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | hot loop 已向量化但 `vsetvli zero,a5,e32,m1,ta,ma`（242286）固定 m1；VLEN=256 下每轮仅 8 元素；峰值 live vector=2（v1,v2），m1·2=2≪32，m2/m4 合法无 spill；`mv`(19.43%)+loop-control(0.88%+0.44%) 迭代开销偏高 | High | Medium（memory-bound 下调） | `patterns/rvv_register_group_utilization.md` |

### 顶层命中 row 三件套

**Finding 1 — RVV Register-Group Utilization and LMUL Sizing（primary）**
(a) 逐字 evidence 引用：
- `0.22 :   242286: vsetvli zero,a5,e32,m1,ta,ma`（loop interval 0x242286–0x2422a4）
- `76.38 :   24228e: vle32.v v2,(s7)`（src 流式 load，受 LMUL/MLP 影响的访存侧）
- `19.43 :   242292: mv t6,s6`、`0.88 :   2422a2: add t5,t5,a0`、`0.44 :   2422a4: bltu a2,t6`（迭代/控制开销）
- 峰值 live vector=2（v1 buf load、v2 src load；`vfmacc.vv` 就地累加 v1），LMUL·peak_live=m1·2=2≤32，m2·2=4、m4·2=8 均合法无 spill。
(b) 互斥邻居排除：
- 非 `no-vectorization`：main loop 有 `vle32.v`/`vse32.v`/`vfmacc.vv`/`vsetvli`（242286–24229e），不是 scalar/zero-`v*` loop。
- 非 `rvv_vector_state_management`：每迭代仅 1 条 `vsetvli`（242286），AVL(a5=min(a2,s6)) 逐轮变化，无兼容状态重复建立。
- 非 `rvv_register_budgeted_loop_unrolling`：backward branch（2422a4 `bltu`）0.44%，非 loop-control/recurrence 主导；主导样本在 load（memory streaming）与 mv（skid 聚合）→ 由 register-group 认领，unroll 延后。
- 非 cache-aware-blocking：无 A/B/C tile、tile-residency/复用拐点证据，reduce 是逐行流式归约。
(c) 双 Confidence 推导式：
- route: 已向量化 + e32,m1 固定小 LMUL + 峰值 live=2 + VLEN=256 + build 含 `v` → High
- impact: 函数级 memory-streaming-bound（76.38% 在 load）+ 缺函数级 perf stat → Medium

### 多候选仲裁小段
仅一个顶层命中（register-group），无多命中仲裁。supporting 候选（vector-state）自身 row gate 不成立（无重复 vsetvli），不输出。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::ReduceR_Invoker<...OpAddSqr/OpSqr...>::operator()(...)

**命中 row（通过 gate）**：`rows-vectorized-tuning.md` → `rvv_register_group_utilization.md`（primary，route High/impact Medium）。

1. **Root cause**：OpenCV row 归约（REDUCE_SUM/平方和，float32，`buf[i] += src[i]*src[i]`）的编译器自动向量化内循环在 VLEN=256 的 RISC-V 目标上固定使用 `e32,m1`（每轮仅 8 元素/32 字节），而循环体峰值 live vector 仅 2 个（v1/v2），`LMUL·peak_live=2` 远低于 32 架构预算，`m2/m4` 完全合法且无 spill。小 LMUL 使流式 src load 每轮 bytes-in-flight 受限（76.38% sample 在 load），迭代与循环控制开销偏高。依据 `patterns/rvv_register_group_utilization.md` §Why this is slow："Underutilized register-group frontier… 单次迭代的有效元素数偏低，循环、vsetvl 和分支开销按更多迭代重复发生"，以及预算式 `LMUL * peak_live_vectors <= 32`（本循环 m1·2=2、m2·2=4、m4·2=8 均满足）。

2. **The fix / 修复方式**：
   - 修正对象：`modules/core/src/matrix_operations.cpp` 的 `ReduceR_Invoker::operator()`/`ReduceR_SIMD` 模板（当前依赖 GCC autovec，autovec 默认选 m1）。
   - 修复方向（register-group，primary）：在 RVV-capable 目标上为该 elementwise 平方和归约提供显式 universal-intrinsics SIMD 路径（或调整循环结构让 autovec 使用更大 LMUL），先枚举 m1/m2/m4/m8×unroll=1/2/4/8 frontier 实测选定（不预设 m4/m8）。提升 LMUL 直接增加每轮覆盖元素与 bytes-in-flight（缓解 76.38% load 侧的 memory-latency）并减少迭代/控制开销。修复前形态：
     ```
     // 当前（GCC autovec 生成）: e32,m1 → 每轮 8 元素（VLEN=256）
     vsetvli zero,a5,e32,m1,ta,ma; vle32.v v1,(t5); vle32.v v2,(s7);
     vfmacc.vv v1,v2,v2; vse32.v v1,(t5); ...
     ```
     修复后形态（示意，LMUL=2；实际按 frontier 选定）：
     ```
     vsetvli zero,a5,e32,m2,ta,ma; vle32.v v1,(t5); vle32.v v2,(s7);
     vfmacc.vv v1,v2,v2; vse32.v v1,(t5); ...
     ```
   - correctness contract：平方和归约数值输出逐元素一致；FP 归约顺序不因 LMUL 改变单元素语义（本循环是 elementwise accumulate，非跨 lane reduction，无 reorder 问题）；`saturate_cast` 语义（此处 float→float 无饱和）不变；REDUCE_SUM 输出通道语义不变。
   - 限制/风险：memory-streaming-bound（76.38% load）意味着收益受内存带宽/延迟约束，LMUL 放大可能不呈线性；需实测确认无 spill、无 group 对齐破坏；更大 LMUL 不必然更快。
   - 修复后预期 Profile signals：热区间 `vsetvli` 使用候选 LMUL；每轮元素数上升、迭代数下降；`vle32.v` 的 bytes-in-flight 提升；`mv`/loop-control sample share 下降。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（SpacemiT X100/K3，有 `v`）；build ISA=`rv64i2p1_..._v1p0_...zvl128b`（含 `v`）；VLEN=256 bits；bound type=函数级 memory-streaming（整体 IPC=0.9176、L1 miss 1.36%）。

4. **收益上界**：入口条件 A（profile-backed）。primary（register-group）evidence sample share 加总 ≈ 0.2075（`mv` 19.43% + loop-control `add` 0.88% + `bltu` 0.44% + vsetvli 0.22%）；表述为「当前 sampled event（cpu-clock, local period）下函数内局部样本份额」，非 workload 级 Amdahl 上界。76.38% 的 load 侧 MLP/bytes-in-flight 潜在收益不并入样本份额（memory-bound 无法由 sample 直接归账）。

5. **三维路由判定**：
   - current source：compiler-generated（GCC autovec 将 matrix_operations.cpp 模板 scalar 循环向量化；`ReduceR_SIMD` base template 原样返回 start，未提供 RVV universal-intrinsics SIMD specialization）
   - implementation existence/reachability：OpenCV 已有 `intrin_rvv_scalable.hpp` 后端且 build 含 `v`，但当前 ReduceR sum/sqr 路径未显式使用更高 LMUL 的 RVV kernel，依赖 autovec
   - function-level policy：无独立 `.S` 政策证据；本路径属 compiler-generated 调优，不进入 missing `.S` 分支

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**：
   - RVV Register-Group Utilization and LMUL Sizing：`Related PRs：3 条 URL` — https://github.com/opencv/opencv/pull/26318 ；https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5 ；https://github.com/opencv/opencv/pull/25586

## Phase 5 — Verification forecast / 验证预测：cv::ReduceR_Invoker<...OpAddSqr/OpSqr...>::operator()(...)
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：
  - `242286: vsetvli zero,a5,e32,m1,ta,ma` 应改为候选 LMUL（m2/m4）的 vtype。
  - `242292: mv t6,s6`（19.43%）与 `2422a2: add t5,t5,a0`/`2422a4: bltu a2,t6` 的每元素成本应随迭代数减少而下降。
  - `24228e: vle32.v v2,(s7)`（76.38%）的每元素成本应在更大 LMUL 的 MLP/bytes-in-flight 提升下下降。
- 应出现侧（锚定 pattern §Verification）：
  - 按 `rvv_register_group_utilization.md` §Verification：枚举 m1/m2/m4/m8×unroll=1/2/4/8 frontier 并记录合法性、live interval、group 对齐、EMUL、mask 使用与 spill；`objdump`/`perf annotate` 确认主循环 vsetvli 使用候选 LMUL 且无 `vlmul_ext`/`vlmul_trunc`；不同 VLEN 与编译器上无 vector spill 回归；REDUCE_SUM 输出逐元素一致。
- 对真实 workload（OpenCV core perf，REDUCE_SUM/sqr 用例）重跑 benchmark，对比该函数 CPU 时间与函数内 sample share；注意 memory-bound 限制，验证以实测为据，不预设倍数。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 全部出现（载荷：`1/1 组；cv::ReduceR_Invoker<...OpAddSqr/OpSqr...>::operator()`） | ✅ 1/1 组；cv::ReduceR_Invoker<float,float,float,cv::OpAddSqr<float,float,float>,cv::OpSqr<float,float,float>>::operator()(cv::Range const&) const |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 2 个 L0 gate + bound-type gate；含 Sampling IP precision 行） | ✅ 7 行；gap 标签：`baseline_gap: sampling IP precision`；无 th.v*/build mismatch |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；`Classes scanned:` 3 文件；顶层 finding=1；evidence 锚点；exclusion 逐条） | ✅ 8 项 trace；Classes scanned: rows-vectorized-tuning.md, rows-operator-rvv.md, rows-codegen.md；顶层 finding 1 个（register-group） |
| 4 | Phase 4 输出要求满足（已读 pattern 文件 + 命中 row + 引用短语；The fix 含 before/after、correctness、风险、Profile signals；Related PRs：3 条 URL） | ✅ 已读 patterns/rvv_register_group_utilization.md |
| 5 | 路径合规：入口模式 A；primary 归属与 L3 层归属合规；blueprint leaf 均来自通过 gate 的 row；无 th.v* 停扫 | ✅ 模式 A + L3 + class 列表（见 Check 3） |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3 引用行（242286/242292/24228e/2422a2/2422a4），出现侧标注 pattern §Verification | ✅ 锚点：vsetvli e32,m1、mv t6,s6、vle32.v v2、add t5、bltu a2 |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成；交付物止于 Profile 证据、根因蓝图、The fix 与验证预测 | ✅ |

修正记录：无