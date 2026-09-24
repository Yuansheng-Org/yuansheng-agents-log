Functions under analysis: [cv::hal::normL2Sqr_(float const*, float const*, int)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 008，`cv::hal::normL2Sqr_`，event=`cpu-clock`，2629 samples，percent: local period；覆盖 4×vl main loop、single-vl loop、vfredusum 归约与 residual tail loop）
- perf stat：已提供（`7-opencv-perf-benchmark-riscv-core.txt`；IPC 0.9176、L1_dcache_load_miss_rate 1.359%、branch_miss_rate 1.673%）
- workload/binary/DSO/source context：已提供（libopencv_core.so.5.1.0，OpenCV 5.x，annotate 含 C++ 源码行号，符号完整）
- readelf -A（热点 object 的 Tag_RISCV_arch）：已提供（metadata 快照 `opencv_perf_core-elf-A`：`rv64i2p1_..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0...`）
- hardware ISA：已提供（metadata 快照：SpacemiT X100，`rv64imafdcvh_...`，含 `v`、`zve64d`、`zvbb/zvbc/zvk*`、`zicond` 等）
- vlenb：已提供（metadata vector：VLEN=256 bits，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock 可解释为时间；percent type=local period 非 global-period；单次运行同一窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（cpu-clock 事件、precise_ip/Exact-IP 未提供 → `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`：暴露标准 `v`（RVV 1.0）、`zve64d`、`zvbb/zvbc/zvk*`（metadata cpuinfo） |
| Build ISA | `Tag_RISCV_arch` 含 `v1p0`、`zve32f/zve64d/zve64f`、`zvl128b`：object 目标架构含 V；无 hardware/build mismatch |
| Vector flavor | annotate 全为 RVV 1.0 `v*` mnemonic（`vle32.v`/`vfsub.vv`/`vfmacc.vv`/`vfredusum.vs`/`vfmv.s.f`/`vfmv.f.s`），无 `th.v*` |
| VLEN | 256 bits（vlenb=32）；`VTraits<v_float32>::vlanes()`=8（`vsetvli a5,zero,e32,m1` → vl=8），4×vl=32 |
| Bound type | IPC 0.918、L1_dcache_load_miss_rate 1.359%、branch_miss_rate 1.673% → compute/latency-bound（非 memory-bound） |
| Sampling semantics | event=cpu-clock（时间可解释）；percent type=**local period**（非 global-period）；同一运行窗口；函数级 workload 贡献未知 → 只能表述局部样本份额，`baseline_gap: sampling metadata` |
| Sampling IP precision | `baseline_gap: sampling IP precision`（cpu-clock、precise_ip 未知）→ 单指令高占比只锚定 loop interval，不做单指令 latency 归因 |

L0 baseline gate：hardware 有 `v` 且 build 有 `v`（RVV 1.0），无 mismatch；annotate 无 `th.v*`，无 vector flavor mismatch；不冻结任何 flavor-dependent route。
Bound-type gate：compute/latency-bound，本地 RVV compute/代码形态修复是第一杠杆；无 memory-bound 竞争 bottleneck 压制 compute fix。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）：`cv::hal::normL2Sqr_(float const*, float const*, int)`（rank 008，OpenCV core `norm` 的 L2 平方和 hal 入口）。

Hot interval 锚点：
- 4×vl main loop `[0x2a8cac..0x2a8d0c]`：逐指令 sample <1%（最高 `2a8cf4 vfmacc.vv` 0.27%、`2a8cd0 vle32.v` 0.27%）；loop body 合计 ≈ 4–6% —— main loop 基本健康、几乎不热。
- single-vl loop `[0x2a8d36..0x2a8d4c]`：合计 ≈ 1–2%。
- **residual tail interval `[0x2a8d50..0x2a8d9c]`：合计 ≈ 70.6% 函数内样本**；最高行 `2a8d58: bge a4,a2,2a8d9e`（34.42%，tail 入口/存在性检查）与 `2a8d9c: bnez a3,2a8d6a`（29.67%，tail 回边）。tail 内另有 `2a8d6a vsetvli e8,mf4`（0.49%，VLMAX 推导）、`2a8d76 vsetivli zero,1`（0.53%）、`2a8d7e vsetvli zero,a5`（3.20%）、`2a8d90 vfmul.vv`（0.76%）、`2a8d72 vle32.v`（0.27%）、`2a8d98 vfmv.f.s`（0.15%）。
- 一次调用固定 setup（`2a8c64 vsetvli` 2.55%、`2a8c68 vmv.v.i v13,0` 10.00%、`2a8c70 slliw` 4.07%、`2a8c7a/2a8c86 vmv1r.v` 1.22%/1.14%）合计 ≈ 19.4%。

结论：**该 workload 对 normL2Sqr_ 的调用以短向量为主（n<32，甚至 n<vl=8）**，4×vl 主循环几乎不执行；样本集中在一次调用固定成本（setup）与 compiler 自动向量化的 residual tail。annotate 覆盖完整，hot loop body 全在；入口条件 A（profile_backed）。Sampling IP precision 不足 → 不把 `bge`/`bnez` 的 34%/30% 解读为该单指令的 cycle cost，而是锚定 tail interval 整体。

## Phase 3 — Pattern scan / 模式扫描：cv::hal::normL2Sqr_

### Class selection trace
1. `rows-vectorized-tuning.md` — include：annotate 已有 `v*`、代码为 compiler/intrinsic-generated（非手写 `.S`），tail 内重复 `vsetvli`/`vsetivli`（vector-state signal）。
2. `rows-operator-rvv.md` — include：normL2Sqr_ 是平方和加性归约（widening-reduction semantic），需核对 no-vectorization / widening-reduction rows。
3. `rows-string-memory.md` — include：RVV main loop 有效而 compiler-generated residual tail 主导 sample（scalar-remainder signal）。
4. `rows-asm.md` — exclude：无手写 `.S` provenance（源码行指向 C++ intrinsic 实现）；无 missing-`.S` 的 policy/existence 四证。
5. `rows-codegen.md` — include：compiler-generated tail 的 vsetvl 序列、分支/回边、寄存器保存属 codegen 层形态。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 证据，纯归约不涉及 offload。
7. `rows-crypto.md` — exclude：非密码学原语。
8. `rows-runtime-os.md` — exclude：用户态 OpenCV 库函数，非 RTOS/kernel 路径。

### Classes scanned: rows-vectorized-tuning.md, rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md

### Local performance pattern scan: `cv::hal::normL2Sqr_`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Scalar Remainder Handling for Wide Kernels（primary） | RVV main loop 有效而 residual tail 地址段主导（≈70.6%）；tail 对有序 reduction 状态逐块 load/update/pointer advance/回跳；source 可证 remainder bound（<vl=8） | High | Medium | `patterns/scalar_remainder_handling_for_wide_kernels.md` |
| RVV Vector-State Management（supporting） | 同一 tail interval 内每迭代重复 3 次 `vsetvli`/`vsetivli`（e8/mf4 VLMAX 推导 + e32/m1 两次 vl 重建） | Medium | — | `patterns/rvv_vector_state_management.md` |

**Scalar Remainder Handling for Wide Kernels（primary）三件套**

(a) 逐字 evidence 引用（tail interval `[0x2a8d50..0x2a8d9c]`，cpu-clock local period）：
```
34.42 : 2a8d58: bge     a4,a2,2a8d9e        <- tail 入口/存在性检查
 0.49 : 2a8d6a: vsetvli a5,a3,e8,mf4,ta,ma   <- VLMAX 推导 cap vl
 0.53 : 2a8d76: vsetivli zero,1,e32,m1,ta,ma <- vl=1（为 vfmv.s.f）
 3.20 : 2a8d7e: vsetvli zero,a5,e32,m1,ta,ma <- 恢复 vl=a5
 0.76 : 2a8d90: vfmul.vv v1,v1,v1
 0.15 : 2a8d98: vfmv.f.s fa0,v2              <- 串行链抽取
29.67 : 2a8d9c: bnez    a3,2a8d6a            <- tail 回边
```
对照 main loop 健康度：`0.27 : 2a8cf4: vfmacc.vv v1,v2,v2`（main 4×vl loop 内最高行）。Sample 归属：tail 地址段独立分账，不共享 main-loop evidence。

(b) 互斥邻居排除：
- `no-vectorization`（rows-operator-rvv）：排除——main loop 已用 `vle32.v`/`vfsub.vv`/`vfmacc.vv`（zero-`v*` 前提不成立）。
- `RVV Widening Additive Reduction`（rows-operator-rvv）：排除——该 row 行内互斥明确"只在 residual tail 占样 → scalar-remainder row"；本函数 main loop 已向量化（4 个独立 vector accumulator），不是 scalar additive-reduction 主循环。
- `RVV Register-Group/LMUL Sizing` 与 `Register-Budgeted Unrolling`（rows-vectorized-tuning）：排除——无 LMUL/live-set 不匹配、无 spill/reload/`vlmul_ext`；main loop 已 4× unroll，residual tail 是单迭代短尾，无 unroll frontier 证据。
- `RVV Inactive-Lane Policy`（rows-vectorized-tuning）：排除——无 `tu/mu`、mask clear 或 inactive-lane 保留证据（tail 用 `ta`）。
- `RVV Operand-Form Selection`（rows-vectorized-tuning）：排除——`vfmv.s.f`/`vfmv.f.s` 是 vfredosum 归约的 scalar-seed/抽取，不是"scalar 先物化 temporary 再被单条 `vv` op 消费"形态。
- codegen 层 rows（register-pressure/control-flow/induction-var）：排除——无 spill/reload/save-restore 证据；`bge`/`bnez` 是普通 loop control，非 branch-layout 问题。
- 排除合计 6 条 + 整组排除 3 个 class（rows-asm/offload/crypto/runtime-os）。

(c) 双 Confidence 推导式：
- route：RVV main loop 有效（main loop body <6% 且每指令 <1%）+ residual tail 地址段主导（≈70.6%）+ source 证明 remainder bound（`for(; j<n; j++)`，tail 长 < vl=8）→ **High**（全 gate 直接证据）。
- impact：hot interval sample share（70.6% 函数内局部）+ VLEN 已知（256）+ bound type 已知（compute/latency）成立；但 sampling metadata 四条不齐（percent=local period、workload 贡献未知）→ 不可称 workload 级 Amdahl，**Medium**。

**RVV Vector-State Management（supporting）三件套**

(a) 逐字 evidence 引用（同 tail interval 内）：
```
 0.49 : 2a8d6a: vsetvli a5,a3,e8,mf4,ta,ma
 0.53 : 2a8d76: vsetivli zero,1,e32,m1,ta,ma
 3.20 : 2a8d7e: vsetvli zero,a5,e32,m1,ta,ma
```
三次 state setup 之间无 call/inline-asm/CSR clobber；`2a8d76` 与 `2a8d7e` 同为 e32/m1，`2a8d7e` 重建 `2a8d6a` 已求得的 `a5`。

(b) 互斥邻居排除：
- register-group/LMUL row：排除——LMUL=1 固定、无 EMUL/live-set 问题，问题在重复 state 建立而非 LMUL 选型。
- inactive-lane row：排除——`ta` policy、无 mask 数据流。
- scalar-broadcast/operand-form row：排除——`vfmv.s.f` 是归约 seed，非 operand 形态问题。
- hand-written `.S` assembly row：排除——compiler-generated intrinsic 代码。

(c) 双 Confidence 推导式：
- route：重复 `vsetvl*` 的直接反汇编证据充分，但 emitter invalidation point 未由 backend dump 证明（pattern 文件要求 route<High when only disassembly）→ **Medium**。
- impact：supporting 不单独评级（—）。

### 多候选仲裁小段

- primary = Scalar Remainder Handling for Wide Kernels；supporting = RVV Vector-State Management。同一 hot interval（`[0x2a8d50..0x2a8d9c]`）、同一机制（residual tail 的固定成本与串行归约）：因果消除测试——按 primary 修复（tail 改为单次 bounded 向量 pass，消除回边与逐迭代归约抽取）后，vector-state 的重复 vsetvl 信号自然消失 → vector-state 并入 supporting，不单列顶层 finding，不计入命中数，不单独排序。
- Evidence-mechanism layer：primary 归 L4（compute/codegen micro-structure）；supporting 归 L3（vector/runtime configuration），随 primary 归属，不独立认领。
- 收益排序：入口条件 A，primary 的 evidence sample share ≈70.6% 函数内局部份额，作为收益上界；supporting 无独立份额。

## Phase 4 — Root-cause blueprint / 根因蓝图：cv::hal::normL2Sqr_

### 1. Root cause

OpenCV `cv::hal::normL2Sqr_` 的 4×vl RVV main loop 本身健康（每指令 <1% sample，4 个独立 `vfmacc` accumulator v1/v8/v6/v7），但**该 workload 以短向量调用为主（n<32，大量 n<vl=8）**，计算几乎全部落入 compiler 自动向量化的 residual tail loop。该 tail 每处理 ≤8 个元素就要支付：一次回边分支（`2a8d9c bnez` 29.67%）+ 入口存在性分支（`2a8d58 bge` 34.42%）+ 3 次 `vsetvli`/`vsetivli` vtype/vl 重建 + 经 scalar `fa0` 的串行归约链（`vfmv.s.f → vfredosum.vs → vfmv.f.s`），形成 loop-carried 水平归约依赖。依据 `patterns/scalar_remainder_handling_for_wide_kernels.md` §Why this is slow："逐 byte residual loop 每处理一个很小的工作单元就支付一次 branch、pointer update 和地址生成成本。有序 accumulator 还会形成串行依赖。对短输入或经常落在非整倍数长度的 workload，即使 main loop 吞吐良好，tail 固定成本仍可能占据主导。"——本函数正是"tail 固定成本主导"的实例。

### 2. The fix / 修复方式

Before（当前源级结构，对应 annotate 的 tail）：
```cpp
d = v_reduce_sum(v_d0);                    // 2a8d50 vfredusum.vs
for( ; j < n; j++ )                        // 2a8d58..2a8d9c 编译器向量化为串行 tail loop
{
    float t = a[j] - b[j];
    d += t*t;
}
```

After（blueprint：bounded single-pass vector tail，消除回边与逐迭代水平归约；依 `patterns/scalar_remainder_handling_for_wide_kernels.md` §The fix 的"受真实 remainder 上界约束"思想在 RVV 上的具体化）：
```cpp
int r = n - j;                             // remainder: 0 <= r < vl（vl = vlanes() = 8）
if (r > 0) {
    v_float32 t0 = vx_load(a + j, r);      // avl=r 的有界 load（不得用 VLMAX 全宽 load）
    v_float32 t1 = vx_load(b + j, r);
    v_float32 diff = v_sub(t0, t1);
    v_d = v_fma(diff, diff, v_d);          // FMA 并入 vector accumulator，不逐迭代抽取到标量
}
d = v_reduce_sum(v_d);                     // 整个函数仅一次水平归约
```
若 intrinsic 层无 avl-bounded load，退化为 pattern 的 descending scalar 分解（4/2/1，按 float 元素）在残余长度上直落，同样消除回边。

适用前提：remainder 上界 < vl（源码 `for(; j<=n-vl; j+=vl)` 保证 tail 长 < 8）；硬件支持 RVV 1.0 与 VLEN=256；tail 使用 tail-agnostic 策略。

不可破坏的 correctness contract：保持输入顺序与 accumulator 初始/最终状态；**FP 归约顺序改变**——将"串行标量累加"改为"单次 `vfredusum`"会重排浮点加法顺序，必须落在 OpenCV norm 既有容差内（main loop 已用 `vfmacc` 重排，norm 测试本就允许 reorder）；NaN/±Inf/±0 传播语义与 `vfredusum` 语义一致；`r=0` 时跳过 tail，返回值与现状一致；不得 over-read 到 `a[j+vl..]`（页边界安全）。

限制/风险：若真实 length distribution 证明 tail 极少执行（n 全为 vl 整倍数），收益为零——本 profile 显示 tail 区间占 70.6%，分布已偏向短向量；编译器可能重新 autovec 出新形态，需在 intrinsic 层显式表达或有界标量直落；code-size 变化为减少（回边/多余 vsetvl 删除），无 spill 风险。

预期 Profile signals：`2a8d9c bnez` 回边消失；`2a8d58 bge` 保留单次存在性检查但占比大幅下降；`2a8d6a/2a8d76/2a8d7e` 三条 vsetvl 消失；`vfmv.s.f`/`vfmv.f.s` 串行链消失；`vfredusum.vs` 每调用一次；main loop 指令与结果不变。

### 3. Baseline facts 回填

hardware ISA：`rv64imafdcvh_...`（含 `v`/`zve64d`/`zvbb/zvbc/zvk*`）；build ISA：`Tag_RISCV_arch` 含 `v1p0`、`zve64d`、`zvl128b`（无 mismatch）；VLEN=256 bits（vlenb=32）；bound type：compute/latency-bound（IPC 0.918、L1 miss 1.36%、branch miss 1.67%）。

### 4. 收益上界

primary 的 evidence sample share：residual tail + reduce interval `[0x2a8d50..0x2a8d9c]` 函数内局部份额 ≈ **70.6%**（1.07+34.42+0.04+0.49+0.27+0.53+3.20+0.76+0.15+29.67）。表述为「当前 sampled event（cpu-clock）下的函数内局部样本份额」；`baseline_gap: sampling metadata`（percent=local period、workload 贡献未知）→ 不得表述为 workload 级 Amdahl 上界。

### 5. 三维路由判定

- current source：compiler/intrinsic-generated（OpenCV rvv_hal_baseline intrinsic + GCC 对 scalar tail 的 autovec；annotate 源码行确认），非手写 `.S`。
- implementation existence/reachability：RVV 实现已存在且可达（main loop 实际发出 `v*`），无 kernel-selection/dispatch 问题。
- function-level policy：OpenCV hal 层无"必须独立 `.S`"的 policy 证据 → 不走 missing-`.S` 分支。

### 6. Related PRs

patterns/scalar_remainder_handling_for_wide_kernels.md：Related PRs：1 条 URL（https://github.com/zlib-ng/zlib-ng/commit/5fd8b67c1a586302106606b2be3c784fb6dc2124）
patterns/rvv_vector_state_management.md：Related PRs：13 条 URL（https://github.com/v8/v8/commit/c81ffb7a356dc408940d479fbbec1d048181be71；https://github.com/qemu/qemu/commit/d57dfe4b37ae542cec84a0cf751ecef313614cb6；https://github.com/qemu/qemu/commit/944b6dfd3d67236882f2bc09d1d30ed923268e16；https://github.com/qemu/qemu/commit/25669d275ce70346b94e3d5e4475d619eb979f5e；https://github.com/qemu/qemu/commit/bd2c82283d21e3400d7d89676a221935904c2fe6；https://github.com/qemu/qemu/commit/81b9ef995a3b2fa5b08fab0615a1c9ed7cbe053e；https://github.com/qemu/qemu/commit/949b6bcb27295eb04350afac32a45b698fc50104；https://github.com/qemu/qemu/commit/b8e1f32cda7805236c2bd497106a9356431c2d60；https://github.com/llvm/llvm-project/pull/148246；https://github.com/llvm/llvm-project/commit/d0554ae4cf264dd05024a753c66e15e4d16bf6e8；https://github.com/llvm/llvm-project/pull/118285；https://github.com/llvm/llvm-project/pull/123878；https://github.com/llvm/llvm-project/commit/f59307bfdc01c584bfa7cd31a55226831bf5590f）

## Phase 5 — Verification forecast / 验证预测：cv::hal::normL2Sqr_

primary（Scalar Remainder Handling for Wide Kernels）：
- 应消失/缩小（锚定 Phase 3(a) 引用行）：`2a8d9c: bnez a3,2a8d6a`（29.67%）回边消失；`2a8d58: bge a4,a2,2a8d9e`（34.42%）占比大幅下降；`2a8d6a/2a8d76/2a8d7e` 三条 vsetvl 序列消失；`2a8d98: vfmv.f.s` 串行抽取消失。
- 应出现（锚定 `patterns/scalar_remainder_handling_for_wide_kernels.md` §Verification）：tail 以单次 avl-bounded 向量 pass 或 4/2/1 直落处理，无逐块回边；对拍 `n=0`、`1..vl-1`、`vl` 整倍数、`4*vl±1` 及边界前后值，结果在 OpenCV norm 浮点容差内一致（FP reorder 说明：单次 `vfredusum` vs 串行标量累加的舍入差必须在 norm 既有容差内；要求严格 bit 级一致时改用 `vfredosum` ordered 或保留串行顺序）；重新 annotate 确认 residual 地址样本与 branch 明显下降且 main loop 未改变；记录真实 length distribution、instructions、branches、code size、I-cache 与 spill 指标。成功判据：结果容差内一致 + residual sample/branch 显著下降；失败：收益被 code-size/spill 抵消或 short input 变慢。

supporting（RVV Vector-State Management）不单独验证，随 primary 预测：tail 内 `vsetvl*` 计数应降至每 tail 一次或零次。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：`1/1 组；cv::hal::normL2Sqr_(float const*, float const*, int)`） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 含 Sampling IP precision；gap 标签：`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-vectorized-tuning.md, rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md；顶层 finding 1 + supporting 1；evidence 锚点 `2a8d58: bge a4,a2,2a8d9e` 34.42%、`2a8d9c: bnez a3,2a8d6a` 29.67%、`2a8cf4: vfmacc.vv v1,v2,v2` 0.27%；排除 6 条 + 整组 4 class；推导式 2 条） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 patterns/scalar_remainder_handling_for_wide_kernels.md（命中 row；引用「tail 固定成本仍可能占据主导」）+ patterns/rvv_vector_state_management.md（supporting；引用 VLMAX 推导/重复 state 建立）；`The fix` before/after、correctness、风险、Profile 信号锚点齐全；Related PRs：1+13 条 URL） | ✅ |
| 5 | 路径合规（载荷：模式 A profile_backed；include 4 class 全扫；L4 primary + L3 supporting；residual tail 按地址分账；`th.v*` 无） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `2a8d9c: bnez`/`2a8d58: bge`/`2a8d6a/2a8d76/2a8d7e vsetvli`；出现侧 pattern §Verification 对拍清单） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问/无代码修改/无补丁生成；交付止于证据+根因蓝图+The fix+验证预测） | ✅ |

修正记录：无