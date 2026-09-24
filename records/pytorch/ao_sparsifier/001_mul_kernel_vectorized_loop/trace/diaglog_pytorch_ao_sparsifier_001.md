Functions under analysis: [`void at::native::RVV::vectorized_loop<...mul_kernel...>`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`/data/perfdata/.../ao_sparsifier/annotate/001-...-annotate.txt`，`13 samples, percent: local period`；覆盖 RVV main loop 38fbc68–38fbde0、38fc0d8–38fc276 与 scalar tail 38fbe52–38fc048）
- perf stat（可选 bound/context）：缺失（详见 Phase 1）
- workload/binary/DSO/source context：已提供（DSO `libtorch_cpu.so`；source `aten/src/ATen/native/cpu/Loops.h:199` `vectorized_loop`、`aten/src/ATen/cpu/vec/rvv/vec_float.h:18` `Vectorized<float>`）
- `readelf -A`（热点 object 的 `Tag_RISCV_arch`）：缺失（详见 Phase 1）
- hardware ISA（`/proc/cpuinfo`）：已提供（metadata cpuinfo `isa=rv64imafdcv_..._zve32f_zve32x_zve64d_zve64f_zve64x_...`）
- `vlenb`：已提供（metadata `vector.vlenb=16` → VLEN=128）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock`、freq=99、threads=64；annotate header `percent: local period`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 支持 RVV 1.0：`v` + `zve32f/zve64f/zve64x`（另有 `zfa/zfh`）。出处：metadata cpuinfo `isa`。 |
| Build ISA | `baseline_gap: build ISA`；metadata `binaries` 为空，无 `readelf -A`。可选命令：`readelf -A libtorch_cpu.so`。间接证据：annotate 出现 RVV 1.0 `v*`（`vfmul.vv`/`vle32.v`/`vsetivli`），说明该 DSO 按含 `v` 目标编译。 |
| Vector flavor | RVV 1.0 `v*`：`0.00 :   38fbd4e:        vfmul.vv        v2,v2,v4`、`0.00 :   38fbbda:        vsetivli        zero,8,e32,m2,ta,ma`；无 `th.v*`，无 `vector_flavor_mismatch`。 |
| VLEN | 128 bits（`vlenb=16`，metadata）。`e32,m2` + `vsetivli zero,8` 与 `VLEN/32*2=8` 自洽。 |
| Bound type | `baseline_gap: bound type`；无 perf stat。可选命令：`perf stat -e cycles,instructions,cache-misses -- <workload>`。 |
| Sampling semantics | event=`cpu-clock`（可解释为时间），但 `percent: local period` 仅为函数内局部份额；scope/同一窗口未证、函数 workload 贡献未知 → 不得称 workload 级 Amdahl，标 `baseline_gap: sampling metadata`。 |
| Sampling IP precision | `baseline_gap: sampling IP precision`；无 `precise_ip`/Exact-IP 证据 → 单行只锚定 basic block / loop interval，不承担 instruction-latency 归因。 |

L0 baseline gate：hardware 有 `v`，build 亦由 annotate 证明有 `v`，无 hardware/build mismatch；无 `th.v*`，flavor gate 不触发。
Bound-type gate：`baseline_gap: bound type` → 本轮所有命中的 performance-impact confidence 封顶。
入口模式 A（`profile_backed`，有完整 hot loop annotate）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数）。hot interval：RVV main loop 首个实例 38fbc68–38fbde0，第二个实例 38fc0d8–38fc276；区间内最高占比行 `53.85 :   38fc238:        add     a3,a3,a5`（位于 38fc228–38fc23a 的 32-byte `std::memcpy` 降级循环）。Sampling IP precision 不足（Phase 1），单行只锚定区间，不单独归因 instruction latency。

## Phase 3 — Pattern scan / 模式扫描：`void at::native::RVV::vectorized_loop<...mul_kernel...>`
Class selection trace（8 项）：
1. `rows-asm.md` — exclude — provenance 是 compiler/intrinsic 生成的 `libtorch_cpu.so` C++（`Loops.h`/`vec_float.h`），无手写 `.S`，无 dispatch slot + 目标 `.S` 缺失的 policy/existence 证据。
2. `rows-operator-rvv.md` — include — 热点为 elementwise mul 语义循环（已向量化）。逐 row：`rvv_contiguous_elementwise_arithmetic_kernels` 的算术体不是 correction target（`38fbd4e: vfmul.vv` 为 0.00），activation/normalization/reduction/matmul/layout/gather/conv/resample/FFT 由算子语义排除，`no-vectorization` 由存在 `v*` 排除。
3. `rows-string-memory.md` — exclude — 无独立 copy/fill/sentinel/compare/checksum 语义；区间内 `vle8.v/vse8.v` 是 `Vectorized::store` 的 `std::memcpy` 降级，不是 string/memory kernel。
4. `rows-vectorized-tuning.md` — include — 已向量化 RVV，修正对象候选是配置/寄存器/operand/state。逐 row 均不命中：operand-form 的 `vfmv.v.f` 只是 broadcast 子步骤（0.00）；vector-state 的 e8/m1 与 e32/m2 切换由真实 memcpy 需要；register-group 的 LMUL=m2 与类型合同匹配、无 `vlmul_ext/trunc`；unrolling 的 back-edge 无区间聚合证据；inactive-lane 无 `tu/mu`。
5. `rows-codegen.md` — include — compiler-generated 指令形态；命中 `Register Pressure and Save/Restore Optimization`。
6. `rows-offload.md` — exclude — SG2044/C920v2 无专用矩阵引擎，算子为 elementwise。
7. `rows-crypto.md` — exclude — 无密码学原语。
8. `rows-runtime-os.md` — exclude — 非 RTOS/kernel/timer 路径。

`Classes scanned: rows-codegen.md（命中）, rows-vectorized-tuning.md（逐行排除）, rows-operator-rvv.md（逐行排除）`

### Local performance pattern scan: `...vectorized_loop<...mul_kernel...>`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Register Pressure and Save/Restore Optimization（primary） | hot interval 由 32-byte `Vectorized` 临时值的 stack spill/reload 与 e8/m1 字节搬运循环主导：`53.85 :   38fc238:        add     a3,a3,a5`、`7.69 :   38fc20c:        ld      a5,-560(s0)`、`7.69 :   38fc240:        ld      a2,-544(s0)` | Medium | Low | `patterns/register_pressure_and_save_restore.md` |
| RVV Vector-State Management（sibling，not matched） | `vsetivli zero,8,e32,m2` 与 `vsetvli a5,a4,e8,m1` 交替，但 e8 是 memcpy 所需真实 vtype，非可复用相同状态 | — | — | `patterns/rvv_vector_state_management.md` |

三件套（primary）：
- (a) 逐字 evidence：`53.85 :   38fc238:        add     a3,a3,a5`，同属 `out1.store`/`out2.store` 的 32-byte `std::memcpy` 降级循环 38fc228–38fc23a；该循环由 `0.00 :   38fc228:        vsetvli a5,a4,e8,m1,ta,ma` 建立 e8/m1，`vle8.v`/`vse8.v` 每次 16 字节、32 字节需两次迭代。相邻 `7.69 :   38fc20c:        ld      a5,-560(s0)` 与 `7.69 :   38fc240:        ld      a2,-544(s0)` 是从栈槽取回 Vectorized 地址/参数。所属 interval：38fc0d8–38fc276。
- (b) 互斥邻居排除：`rvv_register_group_utilization` — LMUL 已由 `Vectorized<float>` 类型合同固定为 m2（`vsetivli zero,8,e32,m2`，`VFLOAT32_VL=8`），无 `vlmul_ext/vlmul_trunc`；spill 源于 `values` 内存数组驻留而非 LMUL 选型，排除。`rvv_operand_form_selection` — `0.00 :   38fbbe8:        vfmv.v.f        v2,fa5` 仅 opt_scalar broadcast，样本 0.00，不主导，排除。`rvv_vector_state_management` — e8/m1 与 e32/m2 属不兼容 vtype，切换由 memcpy 真实需要，排除。`rvv_inactive_lane_policy` — 无反汇编 `tu/mu` 文本，排除。
- (c) 双 Confidence 推导式：`route: compiler-generated C++ provenance + stack spill/byte-copy 直接命中 + 源文件确认 values 为 float[8] 内存数组 → Medium（无 RA dump/live-interval 直接证据）`；`impact: 缺 bound type、采样语义仅 local、IP precision 三项 → Low`。

## Phase 4 — Root-cause blueprint / 根因蓝图：`void at::native::RVV::vectorized_loop<...mul_kernel...>`
1. **Root cause**：RVV `Vectorized<float>` 把向量数据承载为内存数组 `fixed_vfloat32m2_t values`（`aten/src/ATen/cpu/vec/rvv/vec_float.h:20`，32 字节），所有构造/转换都经 `__riscv_vse32_v_f32m2`/`__riscv_vle32_v_f32m2` 往返内存，`store()` 直接 `std::memcpy`（`vec_float.h:154-156`）。`vectorized_loop`（`aten/src/ATen/native/cpu/Loops.h:212-219`）每轮构造多个这种 memory-resident 临时值并调用 out-of-line `dereference_vec_impl`（`0.00 :   38fc0e8:        jal     3834fc6 <...dereference_vec_impl...>`），编译器只能在 576 字节栈帧内反复 spill/reload（`0.00 :   38fbb7e:        addi    sp,sp,-576`）。依据 `patterns/register_pressure_and_save_restore.md §Why this is slow`：堆栈 spill/reload 与过宽保存恢复直接增加 load/store、cache traffic 与 call latency。该行内唯一机制句（"短引"）：`"Spill/reload 和过宽保存恢复会直接增加 load/store、cache traffic 与 call latency"`。
2. **The fix / 修复方式**：与该 pattern §The fix 一致——让向量值在寄存器中就地存活、缩短 live range、避免 memory-resident 临时值。修复前/后伪代码：
   ```cpp
   // Before (vec_float.h): every Vectorized round-trips through a memory array
   fixed_vfloat32m2_t values;
   Vectorized(float val) { vfloat32m2_t v = __riscv_vfmv_v_f_f32m2(val, VFLOAT32_VL); __riscv_vse32_v_f32m2(values, v, VFLOAT32_VL); }
   operator vfloat32m2_t() const { return __riscv_vle32_v_f32m2(this->values, VFLOAT32_VL); }
   void store(void* ptr, int64_t count = size()) const { std::memcpy(ptr, this->values, count * sizeof(float)); }

   // After: keep the vector value register-resident across the op, and store with the RVV store
   vfloat32m2_t values;                       // register-typed member, no memory array
   Vectorized(float val) : values(__riscv_vfmv_v_f_f32m2(val, VFLOAT32_VL)) {}
   void store(void* ptr, int64_t count = size()) const { __riscv_vse32_v_f32m2(reinterpret_cast<float*>(ptr), values, count); }
   ```
   适用前提：`vfloat32m2_t` 成员/返回值在 ABI 与编译器下可保持寄存器分配；RVV 1.0 的 `vse32.v` 支持 runtime `vl`。correctness contract：必须保持 `mul` 的逐元素数值结果、`Vectorized::size()`=8、广播语义与 TensorIterator 步长/尾块处理不变。限制/风险：改变 `Vectorized` 成员类型影响整个 RVV vec 层 ABI，改动面大；`RVV_SUPPORT_UNALIGN` 分支与 unaligned `loadu` 需保留。修复后预期 Profile signals：栈帧与 `ld/sd` 槽位访问大幅减少，`e8,m1` 字节搬运循环消失，`store` 变为单条 `vse32.v`。
3. **Baseline facts 回填**：Hardware ISA=RVV 1.0（`v`+`zve32f/zve64f`）；Build ISA=`baseline_gap: build ISA`（annotate 间接证明含 `v`）；VLEN=128；bound type=`baseline_gap: bound type`。
4. **收益上界**：入口条件 A，命中 evidence 的 local sample share 加总 = `53.85 + 7.69 + 7.69 + 7.69 + 7.69 = 84.61%`（本函数 13 samples 的局部份额；本函数占总采样事件 1.81%）。因 `baseline_gap: sampling metadata`，只表述为「当前 sampled event 下的局部样本份额」，不称 workload 级 Amdahl 上界。
5. **三维路由判定**：current source = compiler/intrinsic-generated C++（`Loops.h`/`vec_float.h`）；implementation existence/reachability = RVV `Vectorized` 实现存在且被 dispatch 采用（annotate 直接命中）；function-level policy = PyTorch 无要求独立手写 `.S` 的函数级契约。
6. Implementation-shape proof：不适用（非 policy-backed missing `.S`）。
7. **Related PRs**：`patterns/register_pressure_and_save_restore.md §Related PRs`：V8 类 26 条、QEMU 9 条、Zephyr 3 条，共 38 条 URL，例：https://github.com/v8/v8/commit/023bbe3a249fa1890cf3d7dee309322ec92d83b6 、https://github.com/v8/v8/commit/9e974a766900c7bf90625f43daebb78c35c3a82e 、https://github.com/qemu/qemu/commit/caf3bef5f01e85b8bbd24e1851b50616fa03a5bb 、https://github.com/zephyrproject-rtos/zephyr/commit/6cb74ad968478852d5a597e8f0507119850590d0 。

## Phase 5 — Verification forecast / 验证预测：`void at::native::RVV::vectorized_loop<...mul_kernel...>`
- 应消失/缩小：Phase 3(a) 引用的 `53.85 :   38fc238:        add     a3,a3,a5` 与同区间 38fc228 的 `vsetvli a5,a4,e8,m1,ta,ma` 字节搬运循环应消失或显著缩小；栈槽 `ld a5,-560(s0)`/`ld a2,-544(s0)` 类访问减少。
- 应出现：与 `patterns/register_pressure_and_save_restore.md §Verification` 一致——RA dump/disassembly 的 peak live registers、spill/reload 数、`mv` 数与 save/restore bytes 下降；重跑 annotate 后该 stack/move 区间收缩；对比 cycles/instructions/load-store counters。
- 正确性：对短/中/长输入与所有尾块长度比较修复前后输出逐元素一致；覆盖非对齐 `loadu`、`S>0` 标量广播、`n<2*Vec::size()` 的 basic_loop 路径。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现（载荷：`1/1 组；void at::native::RVV::vectorized_loop<...mul_kernel...>`） | ✅ |
| 2 | Phase 1 输出要求（载荷：7 行 baseline + L0 gate + bound gate；gap：`baseline_gap: build ISA`、`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`） | ✅ |
| 3 | Phase 3 输出要求（载荷：8 项 Class selection trace；`Classes scanned: rows-codegen.md, rows-vectorized-tuning.md, rows-operator-rvv.md`；顶层 finding 1；锚点 `38fc238`/`38fc20c`/`38fc240`；supporting 0；排除 4；推导式 2） | ✅ |
| 4 | Phase 4 输出要求（载荷：`register_pressure_and_save_restore.md` + `Register Pressure and Save/Restore Optimization` + 短引 "Spill/reload…"；before/after、correctness、风险、Profile signal；Related PRs 38 条 URL） | ✅ |
| 5 | 路径合规（载荷：模式 A；path 单一 provenance；class 列表如上；`th.v*` 未触发停扫） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `38fc238`/`38fc228`；出现侧 `register_pressure_and_save_restore.md §Verification`） | ✅ |
| 7 | 契约边界合规（无实施询问/代码修改/补丁生成） | ✅ |

修正记录：无