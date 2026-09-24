**Functions under analysis: [`dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}::operator()(int,int) const`]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`002-...ncsp_batch_normalization_fwd_t...-annotate.txt`，1147 行，含完整 hot loops，10 samples，event=`cpu-clock:u`，percent type=local period）
- perf stat（可选 bound/context）：已提供（`8-onednn-benchdnn-benchmark-riscv-bnorm_f16_upstream.txt`）
- workload/binary/DSO/source context：已提供（oneDNN commit `d22de940f301e97591e04a2cc6f0010c52109ac7`；`src/cpu/ncsp_batch_normalization.cpp:39-291`；`src/cpu/rv64/jit_uni_batch_normalization.hpp`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：缺失（详见 Phase 1）
- hardware ISA：已提供（metadata：`rv64imafdcv_..._zfh_..._zvfh_...`，C920v2）
- `vlenb`：已提供（16 → VLEN=128 bits）
- 采样元数据：已提供（`cpu-clock:u`，local period，单窗口，workload 贡献未知）
- Sampling IP precision：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zfh_zfhmin_..._zve32f_..._zve64d_..._zvfh_...` → 硬件具备 `v`（RVV 1.0）与 `zvfh` |
| Build ISA | `baseline_gap: build ISA`（无 ELF binary，`readelf -A <libdnnl.so.3.14>` 可补）。**强间接证据**：`src/cpu/rv64/jit_uni_batch_normalization.hpp:49-50` 注释明确"Vector kernels are JIT-emitted; the rv64gc baseline build means a non-V CPU must defer"——rv64 libdnnl 按 `rv64gc` baseline 构建，RVV 全部由 JIT（xbyak_riscv）在运行时发出；本函数（含 `PRAGMA_OMP_SIMD(reduction)` 循环）无 JIT 内核覆盖，故编译为标量 |
| Vector flavor | annotate 本函数内 zero `v*`、zero `th.v*`（全 scalar）；硬件为 RVV 1.0 |
| VLEN | 128 bits（`vlenb: 16`） |
| Bound type | IPC=0.669968；L1_dcache_load_miss_rate=0.264%、branch_miss_rate=0.173% → compute/instruction-bound（cache 计数异常不可靠，同 rank 001） |
| Sampling semantics | event=`cpu-clock:u`（可解释为时间）；percent type=`local period`（非 global）；同一窗口；函数 workload 贡献未知 → 不可称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`；单行占比只锚定 loop interval |

L0 baseline gate：**hardware 有 `v`/`zvfh` 而 binary 按 rv64gc 构建（source 注释直接证据）**——构成最高优先级 baseline finding：本函数所有 OMP SIMD 循环因 build 无 `v` 无法由编译器发 RVV 指令；项目策略是以 JIT 内核补向量化，但本 generic 路径无 JIT 覆盖。`th.v*` gate：不适用。Bound-type gate：compute-bound，本地 RVV 向量化非 memory-bound 排除对象。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）。

hot loop 锚点（10 samples 全部落在以下标量 f32 区间）：
- **statistics sum reduction**：`500002`–`50000c`（`sum += scr_fp32[sp]`）
- **statistics sum-of-squares reduction**：`500190`–`50019e`（`sum += m*m`，m=_src-mean）
- **mean_blk ws_reduce 累加**：`5000a0`–`5000b0`
- **normalize 循环**：`4ffecc`–`4ffed8`（`bn_res = sm*(_src[sp]-mean[off])+sv`）

最高占比行：
- `30.00 :  50000c:  bne  a4,a5,500002`（sum reduction 循环）
- `30.00 :  50019e:  bne  a4,a5,500190`（sum-of-squares 循环）
- `20.00 :  4ffecc:  ld   a4,40(s10)`（normalize 循环）
- `10.00 :  4ffed8:  add  a4,a4,s9`（normalize 循环）
- `10.00 :  5000b0:  bne  a4,s5,5000a0`（mean 累加）

统计阶段（sum+sqsum+mean 累加）占 70% samples，normalize 阶段占 30%。Sampling IP precision 未确认 → 结论收敛到 interval-level mechanism。annotate 覆盖完整，无 coverage gap。

## Phase 3 — Pattern scan / 模式扫描：dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源是 compiler-generated C++（`ncsp_batch_normalization.cpp` 模板实现，反汇编为标量 f32 循环），无 `.S` provenance；oneDNN rv64 实现载体是 JIT generator，无要求独立 `.S` 的 policy。
2. `rows-operator-rvv.md` — **include** — compiler-generated scalar loops，语义为跨 lane 归一化统计 + transform（`RVV Normalization Kernels` 候选）；`no-vectorization` 仅当更具体 row 不命中时认领。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/compare/checksum 循环。
4. `rows-vectorized-tuning.md` — **exclude** — 本函数 zero `v*`，修正对象是向量化缺失，不是 RVV 配置/LMUL/unroll。
5. `rows-codegen.md` — **include** — compiler-generated 指令形态；评估 kernel-selection、kernel-operation-fusion、eliminate-unnecessary-conversions。
6. `rows-offload.md` — **exclude** — 无矩阵引擎/packed-SIMD/权重重排/GEMM 分块。
7. `rows-crypto.md` — **exclude** — 无密码学原语。
8. `rows-runtime-os.md` — **exclude** — 用户态应用库热点。

**Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`**

### Local performance pattern scan: `dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Normalization Kernels（**primary**） | BN 统计 sum/sum-of-squares reduction 与 normalize 循环全标量 f32，zero `v*`；硬件 v/zvfh | High | Medium | `patterns/rvv_normalization_kernels.md` |

#### 三件套 — 顶层 primary：RVV Normalization Kernels

**(a) 逐字 evidence 引用**：
```
30.00 :  50000c:  bne     a4,a5,500002     ← sum += scr_fp32[sp] 归约循环（ncsp_batch_normalization.cpp:167-170）
30.00 :  50019e:  bne     a4,a5,500190     ← sum += (m*m) 平方和归约循环（:208-212）
10.00 :  5000b0:  bne     a4,s5,5000a0     ← mean_blk[c] += ws_reduce[...]（:180-183）
20.00 :  4ffecc:  ld      a4,40(s10)       ← normalize 循环加载 mean 指针（:265-279）
10.00 :  4ffed8:  add     a4,a4,s9         ← normalize 循环 mean 偏移
```
循环体全部为标量：`500002-50000c` = `flw`+`addi`+`fadd.s`+`bne`；`500190-50019e` = `flw`+`addi`+`fsub.s`+`fmadd.s`+`bne`；normalize 区间 = `flw`(src)+`fsub.s`+`fmadd.s`+`fsw`+fuse_relu 分支。源码语义：`PRAGMA_OMP_SIMD(reduction(+:sum))` 循环（:167-170、:208-212）与 `PRAGMA_OMP_SIMD()` normalize 循环（:263-279），编译结果为零 `v*` 标量链——与 build rv64gc baseline（jit_uni_batch_normalization.hpp:49-50 注释）一致。

**(b) 互斥邻居排除**：
- `rvv_widening_additive_reduction`：排除——行内互斥"Softmax/LayerNorm/RMSNorm 完整算子 → normalization row"；sum/sum-of-squares 是 BN 完整统计流程（mean/variance + normalize pass）的组成部分，且 normalize 区间也承载 30% samples，非 standalone reduction。
- `no-vectorization`：排除——行内互斥"本行只认领未被更具体 semantic/codegen/assembly row 解释的 generic scalar main loop"；normalization row 认领本循环。zero-`v*` 的成因（rv64gc build）作为 L0 baseline finding 单独报告，不构成顶层命中。
- `kernel_selection`（codegen）：排除——行内互斥"没有可用专用实现且 loop 为 scalar → 对应 operator/no-vectorization row"；rv64 `jit_uni_batch_normalization_fwd_t` 存在但 `VDISPATCH_BNORM(use_global_stats(), ...)`（jit_uni_batch_normalization.hpp:71-72）在 training/非 global-stats 配置下明确 decline，本 benchmark 的 stats 计算路径无可用专用实现（非"存在未选中"）。
- `kernel_operation_fusion`（codegen）：排除——tmp_src f32 缓冲往返（`src_cvt_wsp`，key_bnorm_cvt）确有地址/源码证据，但本函数 sample 未落在转换 pass（转换成本归属 rank 001 的 `cvt_float16_to_float`）；本函数 sample 主导在统计/归一化标量循环，向量化是首要修正对象，fusion 属次要。
- `eliminate_unnecessary_precision_conversions`：不适用——f16→f32 是必要的 dtype 边界转换（已分析于 rank 001）。
- `rvv_elementwise_activation`：排除——normalize 循环的 ReLU 是 fuse 后处理（`fuse_norm_relu`），非独立 lane-independent activation；完整 BN 统计+transform 由 normalization row 认领。

**(c) 双 Confidence 推导式**：
- `route: 当前代码来源=compiler-generated scalar loops（ncsp_batch_normalization.cpp 源码 + 标量反汇编）+ 硬件 v/zvfh（metadata）+ 语义合同=跨 lane BN 统计（sum/sqsum）+逐元素 transform（normalize）+ 行内互斥均以具体指令/源行排除 → High`
- `impact: hot interval sample share=10/10 + VLEN=128 已知 + bound≈compute；但 sampling metadata 仅 local-period、workload 贡献未知、build ISA 为 gap（source 注释间接证实 rv64gc）→ Medium`

#### 仲裁小段

- 单 primary：`RVV Normalization Kernels`（L1 vectorization/semantic dispatch 层），无 supporting/companion/independent。收益排序（入口条件 A）：sample share = 10/10（100%），其中统计阶段 70%、normalize 阶段 30%——同机制（标量执行 + 无向量化）内不作拆分排序；函数为批次 rank 002。

## Phase 4 — Root-cause blueprint / 根因蓝图：dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}

（纳入蓝图 pattern：`patterns/rvv_normalization_kernels.md` 命中 row `RVV Normalization Kernels`）

1. **Root cause**：generic ncsp batch-normalization f16 forward（training 模式，`calculate_stats=!pd()->stats_is_src()`）在 RISC-V 上全部以标量 f32 循环执行：mean 的 sum reduction（ncsp_batch_normalization.cpp:167-170）、variance 的 sum-of-squares reduction（:208-212）与逐元素 normalize transform（:265-279）。依据 `patterns/rvv_normalization_kernels.md` §Why this is slow：归一化根因是 *"跨 lane reduction 造成依赖链，多阶段完整扫描造成额外流量"*，且 *"reciprocal/rsqrt/vector-math 不可达而退回 scalar helper"*。两个原因在此同时成立：① build 为 `rv64gc` baseline（jit_uni_batch_normalization.hpp:49-50），`PRAGMA_OMP_SIMD(reduction)` 循环无法由 GCC 发出 RVV；② rv64 JIT bnorm kernel 仅在 `use_global_stats()` 时可用（:71-72），本 training 统计路径无专用向量实现，退回 generic scalar 路径。

2. **The fix / 修复方式**（与该文件 §The fix §5/§6 一致；诊断蓝图，非实施）：

**修复对象**：`src/cpu/ncsp_batch_normalization.cpp` 的 statistics reduction 与 normalize 循环（或 rv64 侧等价实现），增加运行时 gated 的 RVV intrinsic 快速路径；保留现有标量循环为 rv64gc/non-V fallback。

```cpp
// Before（ncsp_batch_normalization.cpp:167-170 / 208-212，编译为标量 fadd/fmadd 链）：
PRAGMA_OMP_SIMD(reduction(+ : sum))
for (dim_t sp = S_s; sp < S_e; ++sp) { sum += scr_fp32[sp]; }
// ... sqsum: sum += (m*m)

// After（示意；per-chunk vfredusum，seed=标量累计值；vl 动态取 min(remaining, VLMAX)）：
float sum = 0.f;
for (dim_t sp = S_s; sp < S_e; ) {
    size_t vl = vsetvl_e32m1(S_e - sp);
    vfloat32m1_t x = vle32_v_f32m1(scr_fp32 + sp, vl);
    vfloat32m1_t seed = vfmv_v_f_f32m1(sum, 1);
    sum = vfmv_f_s_f32m1_f32(vfredusum_vs_f32m1_f32m1(x, seed, vl));
    sp += vl;
}
// variance 循环同理：x' = vfsub_vf(x, mean)；vfmul_vv(x',x')；vfredusum 累积
// normalize 循环：vle32 → vfsub_vf(mean) → vfmacc_vf(sm) → vfadd_vf(sv) → vse32（fuse_relu 分支按现有语义保留）
```

**适用前提**：运行时 `mayiuse(v)`/`mayiuse(zvfh)` gate（cpu_isa_traits.hpp 已有基建）；仅向量化 f32 中间缓冲（`src_cvt_wsp`）上的统计与 transform，转换 pass 本身由 rank 001 的 fix 覆盖。

**Correctness contract（不可破坏）**：① mean/variance 的 FP32 累加语义在 oneDNN BN 容差合同内（pattern §5：`vfredusum` 改变 FP 加法结合顺序，须按项目精度契约决定 unordered/ordered；若 reference 用 Welford/Kahan 不得改简单 sum/sqsum）；② epsilon、zero variance、`variance=sum²/N-mean²` 消减误差风险（§5 注）须与测试容差核对；③ training 模式 `fuse_norm_relu` 与 workspace(`ws`) 写入语义（:268-275）不得改变；④ 动态 `vl` 的 tail 由 `vsetvl` 原生覆盖，不得假设固定 VLEN=128。

**限制/风险**：① `vfredusum` 的 unordered 归约在极少数输入下与标量参考有尾数级差异，需过 `bnorm_f16_upstream` 正确性断言（pattern §Verification：*"最后 reduction 的 `vl`、tail、ordered/unordered FP 语义 ... 明确"*）；② 若同时融合 `cvt_to_float` 进入 reduction（去掉 tmp f32 往返），须证明中间值无外部观察者且 alias 合同允许（§Verification：*"若融合 passes，证明中间值无外部观察者"*）——`src_cvt_wsp` 被多 pass 复用，融合需谨慎；③ build ISA 未启用 `v` 时 intrinsic 路径不编译/不可达，标量 fallback 保持。

**修复后预期 Profile signals**：`50000c`/`50019e`/`5000b0` 三条标量循环分支与 `4ffecc`/`4ffed8` normalize 加载行的 sample 消失或显著缩小；出现 `vfredusum.vs`/`vle32.v`/`vfsub`/`vfmacc` 等向量指令；函数 instructions/element 显著下降。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_..._zvfh_...`（C920v2）；build ISA = `baseline_gap: build ISA`（source 注释直接证据指向 `rv64gc` baseline，`readelf -A` 确认）；VLEN = 128 bits；bound = compute/instruction-bound（IPC 0.67）。

4. **收益上界**：当前 sampled event 下该函数局部样本份额 = 10/10（100%）落在标量统计/归一化循环；函数为批次 rank 002。sampling 语义四条不满足 → 不声称 workload 级 Amdahl 上界。

5. **三维路由判定**：
   - current source：compiler-generated C++ scalar loops（`ncsp_batch_normalization.cpp` 模板实现；反汇编标量 f32）。
   - implementation existence/reachability：rv64 JIT bnorm（`jit_uni_batch_normalization_fwd_t`）存在但仅 `use_global_stats()`（inference）可达（jit_uni_batch_normalization.hpp:71-72）；本 training 统计路径无专用实现。
   - function-level policy：oneDNN rv64 以 JIT generator 承载向量化，无独立 `.S` policy；generic 路径的向量化应经 runtime-gated intrinsics 或扩展 JIT kernel 达成。

6. **Implementation-shape proof**：N/A（非 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**：`patterns/rvv_normalization_kernels.md` 关联提交：Related PRs：17 条 URL
- https://github.com/uxlfoundation/oneDNN/commit/36df719b3bc9001d994455c2899e42533b2c29ac
- https://github.com/uxlfoundation/oneDNN/commit/6dc3e2d84eec45eff0834e8985cc663db6fe07ee
- https://github.com/uxlfoundation/oneDNN/commit/3b90f9d0650f4087d4a6fbcb3406dc0862f2ae25
- https://github.com/uxlfoundation/oneDNN/pull/4809
- https://github.com/uxlfoundation/oneDNN/pull/4622
- https://github.com/uxlfoundation/oneDNN/pull/4453
- https://github.com/uxlfoundation/oneDNN/pull/4480
- https://github.com/uxlfoundation/oneDNN/commit/0d8f4a9e702b70f6188f58a20247ae06dbd41ce5
- https://github.com/uxlfoundation/oneDNN/commit/7ad03324c2d17480e40cc20114576584888373ce
- https://github.com/uxlfoundation/oneDNN/commit/3c8c37bab64f67bbdebcdb76330db0dcb25b31af
- https://github.com/uxlfoundation/oneDNN/commit/c07b7f4e777636cf527f49093c03f8cacf58b3de
- https://github.com/uxlfoundation/oneDNN/pull/4734
- https://github.com/uxlfoundation/oneDNN/pull/4491
- https://github.com/uxlfoundation/oneDNN/commit/25cd4a75095a2aebeb2f0140abadd466cd0b4e90
- https://github.com/uxlfoundation/oneDNN/commit/b8360ec0a1d566f039d4fe286349c7988a762c98
- https://github.com/alibaba/MNN/pull/4508
- https://github.com/alibaba/MNN/pull/4044

## Phase 5 — Verification forecast / 验证预测：dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}

primary（RVV Normalization Kernels）——收益上界局部份额 10/10：

- **应消失/缩小**（锚定 Phase 3(a) 引用行）：`50000c: bne a4,a5,500002`、`50019e: bne a4,a5,500190`、`5000b0: bne a4,s5,5000a0` 三条标量归约分支，及 `4ffecc: ld a4,40(s10)`、`4ffed8: add a4,a4,s9` normalize 加载行的 sample 消失或显著缩小。
- **应出现**（锚定 `patterns/rvv_normalization_kernels.md` §Verification）：重建后 annotate 出现 vector reduction（`vfredusum.vs`）与 vector transform（`vle32`+`vfsub`+`vfmacc`）；scalar reduction/helper 与多余完整 pass share 下降；cycles/row 或吞吐改善。
- **正确性**：BN 统计在 oneDNN 容差内（epsilon、zero variance、reduction order——`vfredusum` unordered 语义、mixed precision f16→f32→f16、fuse_relu 与 ws 写入、短 SP 与 tail）（§Verification 正确性合同）。
- **回退路径**：non-V/rv64gc 构建保持现有标量路径，行为一致。
- **升级到 profile-backed 定量结论所需补采数据**：重采 `perf record --percent-type=global-period` 获取该函数全局样本份额；`readelf -A <libdnnl.so.3.14>` 确认 build ISA 是否确为 rv64gc。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅ | `1/1 组；dnnl::impl::cpu::ncsp_batch_normalization_fwd_t<(dnnl_data_type_t)1>::execute_forward(...)::{lambda(int,int)#1}` |
| 2 | Phase 1 输出要求满足（7 行 baseline + L0 gate + bound gate + Sampling IP precision） | ✅ | 结论 7 行；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`；L0 finding（hardware v 而 build rv64gc）置顶 |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace、Classes scanned、顶层 finding、evidence 锚点、排除、推导式） | ✅ | 8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md、rows-codegen.md`；顶层 finding=1；evidence 锚点=`50000c bne`/`50019e bne`/`5000b0 bne`/`4ffecc ld a4,40(s10)`/`4ffed8 add a4,a4,s9`；supporting=0；排除 7 条；推导式 2 条（route High / impact Medium） |
| 4 | Phase 4 输出要求满足（已读 pattern 文件、命中 row、引用短语、The fix 的 before/after/correctness/risk/profile signals、Related PRs） | ✅ | 已读 `patterns/rvv_normalization_kernels.md`（命中 row `RVV Normalization Kernels`；引用短语"跨 lane reduction 造成依赖链"、"reciprocal/rsqrt/vector-math 不可达"；fix 含 §5 vfredusum per-chunk 结构与 §6 vector transform）；Related PRs：17 条 URL |
| 5 | 路径合规：trace 扫描集、仲裁、blueprint leaf 来自通过 gate 的 row、动态份额排序、`th.v*` 未全局停扫 | ✅ | 模式 A；路径：operator-rvv（primary L1）；class 列表=2 include + 6 exclude；primary 动态份额 10/10 排序；无 `th.v*` |
| 6 | Phase 5 两侧锚定：消失侧对 Phase 3 引用行、出现侧标注 pattern §Verification | ✅ | 消失侧=`50000c bne a4,a5,500002`/`50019e bne a4,a5,500190`/`5000b0 bne a4,s5,5000a0`/`4ffecc ld a4,40(s10)`/`4ffed8 add a4,a4,s9`；出现侧=`patterns/rvv_normalization_kernels.md §Verification` |
| 7 | 契约边界合规：无实施询问/代码修改/补丁生成/契约外分支；无追问 | ✅ | 全文仅输出证据、根因蓝图、The fix（示意结构）与验证预测；无实施、无提问 |

修正记录：无