**Functions under analysis: [jit_rvv_layernorm_f16_fused_kernel_t.3]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`006-jit_rvv_layernorm_f16_fused_kernel_t.3-annotate.txt`，jitted DSO `jitted-12961-3.so`，含全部三个循环体与标量段，覆盖完整）
  - 该 annotate 仅含 **3 samples**（`cpu-clock:u`，percent: local period），三行各 33.33%：`0xc0`（sum 循环回边 `j`）、`0xd4`（sum 归约提取 `vfmv.f.s`）、`0x1b0`（out 循环指针推进 `slli`）。**样本量极低，构成 `evidence_gap: sample count`**，进入全部 confidence 推导。
- 同运行同 event 同窗口的**同码不同 JIT 实例**（`.9` rank 1、12 samples）作为补充证据：`006` 与 `001` 的反汇编逐字节一致（同一 generate() 发射的相同指令序列，仅 symbol 后缀不同），`.9` 提供更密集的采样分布（sum 循环 6/12=50%，out 循环 3/12=25%，var 循环 1/12，标量/归约段 2/12）。标注为同 kernel 家族、同一采样窗口下的辅助证据，不单独构成跨函数比较。
- perf stat：已提供（`8-onednn-benchdnn-benchmark-riscv-lnorm_f16_upstream.txt`；IPC 0.6758，L1_dcache miss 0.234%，LLC miss 15.9%，branch miss 0.219% → compute/latency-bound 特征）。
- workload/binary/DSO/source context：已提供（`benchdnn` lnorm f16；JIT kernel 源码 `src/cpu/rv64/jit_rvv_layernorm_kernel.cpp` 中 `jit_rvv_layernorm_f16_fused_kernel_t::generate()`（L575-743）与调用方 `src/cpu/rv64/rvv_layer_normalization.cpp`（L82-93，逐行 `parallel_nd` 调用）在工作区可直接核对；`annotate_summary.json` 确认三实例 `.9/.6/.3` 为 rank 1/4/6）。
- readelf -A（Tag_RISCV_arch）：缺失（metadata 警告 `No ELF executable binaries were found for this run`；JIT 运行期生成，无 ELF attribute）→ `baseline_gap: build ISA`（详见 Phase 1）。
- hardware ISA（cpuinfo/hwprobe）：已提供（rv64imafdcv…`v` 存在，RVV 1.0，含 `zvfh`/`zfh`/`zfa` 等；T-Head C920v2）。
- `vlenb`：已提供（VLEN=128 bit，vlenb=16）。
- 采样元数据（event/percent/scope/window）：部分提供。event=`cpu-clock:u`，percent=local period，同窗口；**函数级 workload 贡献未知** → `baseline_gap: sampling metadata` 中"函数贡献"一项（不可作 Amdahl 上界）。
- Sampling IP precision：缺失（无 `precise_ip`/Exact-IP 信息）→ `baseline_gap: sampling IP precision`；单指令占比只锚定 basic block / loop interval。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_zicbom_…_zvfh_zvfhmin…`：标准 RVV 1.0 `v` 存在，C920v2 OoO，VLEN=128 |
| Build ISA | `baseline_gap: build ISA`（JIT 运行期发射，无 ELF `Tag_RISCV_arch`）；由已执行指令（`vle16.v/vfwcvt.f.f.v/vfncvt.f.f.w/vse16.v` 等 RVV 1.0 + `zvfh`）实证 build=RVV1.0 指令集有效 |
| Vector flavor | annotate 全为标准 RVV 1.0 `v*` mnemonic（解码后），无 `th.v*` → 无 flavor mismatch |
| VLEN | vlenb=16 → VLEN=128 bit；VLMAX(e16,m4)=32 元素，VLMAX(e32,m8)=32 元素 |
| Bound type | compute/latency-bound：IPC 0.676、L1 miss 0.234%、branch miss 0.219%；`perf stat` counters 有效区分 compute-bound，不归 memory-bound |
| Sampling semantics | event=`cpu-clock:u`（时间可解释）；percent type=local period；同窗口；**函数级 workload 贡献未知** → `baseline_gap: sampling metadata`（贡献项） |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP 信息；单指令归属不可靠，仅区间级结论） |

L0 baseline gate：hardware 有 `v` 且执行代码即 RVV 1.0（JIT 发射），无 hardware/build mismatch；无 `th.v*`；annotate 完整 → 入口条件 A（profile_backed）。
Bound-type gate：compute-bound（IPC 0.676、cache miss 极低）→ 本地向量 kernel 优化是第一杠杆；因 `baseline_gap: sampling metadata`（函数贡献）与 `evidence_gap: sample count`，本轮所有命中的 performance-impact confidence 封顶为 Low，不做 Amdahl 上界。

## Phase 2 — Scope / 分析边界

- 函数清单：`jit_rvv_layernorm_f16_fused_kernel_t.3`（1 个），与承诺声明一致。
- 该函数是 oneDNN RVV LayerNorm（f16）JIT kernel（`with_scale=false, with_shift=false` 实例），逐行（row）执行"统计量计算 + 归一化输出"完整流程，每行 3 个完整 pass 扫描同一 `src`。
- 完整解码后的 hot loop / loop interval 边界（原始 `.insn` 经 Xbyak_riscv 编码表核对）：
  - **sum 循环** 0x98–0xc0（10 条指令/32 元素 chunk）：`vsetvli t0,a6,e16,m4`(0x9c) → `vle16.v v24,(a1)`(0xa4) → `vfwcvt.f.f.v v16,v24`(0xb0) → `vsetvli t1,t0,e32,m8,tu`(0xb4) → `vfadd.vv v0,v0,v16`(0xb8) → `vmv.v.x v16,x0`(0xbc) → 回边 `j`(0xc0)。最高行（本函数）`0xc0` 33.33%。
  - **sum 归约/mean 标量段** 0xc4–0xec：`vfredosum.vs`+`vfmv.f.s ft1,v16`(0xd4，33.33%)+`fdiv.s` 等。
  - **var 循环** 0x104–0x130（12 条/chunk）：0x110 `vle16.v`、0x11c `vfwcvt`、0x120 `vsetvli e32/m8,tu`、0x124 `vfsub.vf v16,v16,ft1`、0x128 `vfmacc.vv v8,v16,v16`。本区间 0 samples。
  - **var 归约/inv 标量段** 0x134–0x174：`vfredosum.vs`、`fdiv.s`(0x144)、f64 链 `fcvt.d.s→fadd.d→fsqrt.d→fdiv.d→fcvt.s.d`(0x158–0x174)。
  - **out 循环** 0x184–0x1bc（16 条/chunk）：0x190 `vle16.v`、0x194 `vfwcvt`、0x198 `vsetvli e32/m8,ta`、0x19c `vfsub.vf`、0x1a0 `vfmul.vf v16,v16,ft4`、0x1a4 `vsetvli e16/m4`、0x1a8 `vfncvt.f.f.w v24,v16`、0x1ac `vse16.v v24,(a2)`。最高行（本函数）`0x1b0 slli` 33.33%。
- Sampling IP precision 未知 → 只锚定 loop interval，不做单指令 latency 归因。
- 补充区间证据（同码实例 .9，12 samples）：sum 循环 6/12（0xb4 vsetvli 33.33%×4、0xbc vmv 8.33%、0xc0 回边 8.33%），out 循环 3/12（0x178 ld 8.33%、0x1ac vse16 16.67%），var 循环 1/12（0x12c vmv），标量/归约段 2/12（0x140、0x148）。
- annotate 覆盖完整（3 个循环体 + 全部标量段都在）；无 `annotate_gap`。

## Phase 3 — Pattern scan / 模式扫描：jit_rvv_layernorm_f16_fused_kernel_t.3

### Class selection trace

1. `rows-asm.md` — **exclude** — 当前代码来源是 JIT-generated（Xbyak_riscv emitter 输出，`jitted-12961-3.so`），无手写 `.S`/DWARF provenance，也无 missing-`.S` 的 policy/existence 证据。
2. `rows-operator-rvv.md` — **include** — 热点是 LayerNorm（cross-lane normalization：mean/var statistics + 逐元素 transform）；kernel 名与 `jit_rvv_layernorm_kernel.cpp` 源码直接证明语义合同；annotate 显示 3 个独立完整 pass 反复扫描同一 `src`（0x98/0x104/0x184），命中 normalization row 的"多遍扫描"失效形态。
3. `rows-string-memory.md` — **exclude** — 无 copy/fill/sentinel/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — **include** — annotate 已有完整 `v*` 且非手写 `.S`（JIT-generated RVV）；修正对象可为 loop-control/vsetvli/LMUL/展开微结构（register-budgeted unrolling row）。
5. `rows-codegen.md` — **include** — 代码来源是 JIT/runtime-generated；需评估跨 pass 流量（kernel-operation-fusion）、f64 精度转换（eliminate-precision-conversions）、emitter 质量（jit-code-quality）等形态 row。
6. `rows-offload.md` — **exclude** — C920v2 无矩阵引擎/packed-SIMD/weight-repack 证据（ISA 字符串无矩阵扩展）。
7. `rows-crypto.md` — **exclude** — 无密码学原语。
8. `rows-runtime-os.md` — **exclude** — 用户态 JIT kernel，无 timer/ISR/CSR/权限域热点。

**Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md**

### Local performance pattern scan: `jit_rvv_layernorm_f16_fused_kernel_t.3`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Normalization Kernels（primary，L1） | LayerNorm 语义合同 + 3 个完整 pass 扫描同一 src（0x98/0x104/0x184）+ sum 循环区间在 .9 占 6/12、.3 占 1/3 | High | Low | `patterns/rvv_normalization_kernels.md` |
| Register-Budgeted RVV Loop Unrolling（independent，L4） | 每 32 元素 chunk 2×`vsetvli`+`vmv.v.x`+4 条标量控制指令；sum 循环区间 .9 6/12，且样本集中在控制/配置指令（0xb4 vsetvli 4/12）；LMUL 已达 m8 上界；同文件 f32 kernel 用 fixed-VL+4-way block（循环内 0 vsetvli） | Medium | Low | `patterns/rvv_register_budgeted_loop_unrolling.md` |

#### Finding 1 — RVV Normalization Kernels（primary）

- **(a) 逐字 evidence 引用**（所属 interval 点名）：
  - sum 循环（0x98–0xc0，本函数最高行区间）：
    ```
    33.33 :   c0:     j       98 <jit_rvv_layernorm_f16_fused_kernel_t.3+0x18>
    ```
  - 三 pass 同源指针的直接证据（同一 `a1` 三次重新加载后各自建循环）：
    ```
     0.00 :   f0:     ld      a1,0(a0)          ; pass2 重新加载 src
     0.00 :  104:     beqz    a6,134            ; var 循环入口
     0.00 :  178:     ld      a1,0(a0)          ; pass3 重新加载 src
     0.00 :  184:     beqz    a6,1c0            ; out 循环入口
    ```
  - 补充（同码实例 .9，同窗口）：sum 循环区间 6/12=50%（`33.33 : b4: vsetvli`、`8.33 : bc`、`8.33 : c0`），out 循环 3/12=25%（`16.67 : 1ac: vse16`），var 循环仅 1/12。
- **(b) 互斥邻居排除**：
  - elementwise-activation row：循环无 ReLU/Exp/SiLU/GELU 等 lane-independent activation，主体是跨 lane 的 mean/var 统计归约 → 排除（判据"Softmax/LayerNorm…→ normalization row"）。
  - widening-additive-reduction row：三个循环不是 standalone additive reduction，而是完整 LayerNorm 算子的 statistics+transform → 排除（判据"LayerNorm 完整算子 → normalization row"）。
  - precision-conversion row：`vfwcvt/vfncvt` 是 LayerNorm 数据路径的 f16↔f32 必要子步骤，算术/归约主体主导，conversion 只是输入/输出子步骤 → 排除（判据"转换只是输入/epilogue 子步骤 → 对应 operator row"）。
  - kernel-operation-fusion（rows-codegen）：三 pass 的中间值（Σx、Σ(x−mean)²）始终留在向量/标量寄存器中，**无中间缓冲区 store→load 往返**；且它们是被测算子的统计 pass 而非主计算后的独立 post-op pass → 该 row 的"中间结果写回再读入"门不成立，不命中。
- **(c) 双 Confidence 推导式**：
  - `route: JIT-generated 源码+名称证明 LayerNorm 语义 + annotate 直接显示 3 个完整 pass 扫描同一 src + 行内互斥全部排除 → High`；
  - `impact: 缺函数级 sample share（baseline_gap: sampling metadata）、样本量仅 3（.3）/12（.9）、row 长度/形状未知、E[x²]−mean² 数值合同未验证 → Low`。

#### Finding 2 — Register-Budgeted RVV Loop Unrolling（independent）

- **(a) 逐字 evidence 引用**：
  - sum 循环每 chunk 的配置/控制序列（0x98–0xc0，本函数 33.33% 落在回边）：
    ```
     0.00 :  9c:     .insn   4, 0x0ca872d7      ; vsetvli t0,a6,e16,m4,ta,ma
     0.00 :  b4:     .insn   4, 0x0932f357      ; vsetvli t1,t0,e32,m8,tu,ma
     0.00 :  bc:     .insn   4, 0x5e004857      ; vmv.v.x v16,x0（tail 清零）
    33.33 :  c0:     j       98
    ```
  - 补充（同码 .9）：`33.33 : b4: .insn 4, 0x0932f357`（4/12 落在 e32/m8 vsetvli）、`8.33 : bc: .insn 4, 0x5e004857`、`16.67 : 1ac: .insn 4, 0x02065c27`（out 循环 vse16）。
  - 结构对照（同一文件 f32 kernel `jit_rvv_layernorm_data_kernel_t::generate()`，L416-520）：主循环固定 `vl=VLMAX(e32,m1)`，`reg_block=4*VLMAX`，循环体内 **0 条 vsetvli**，tail 由独立 runtime-vl 循环承担——f16 kernel 未沿用该结构。
- **(b) 互斥邻居排除**：
  - vector-state-management row：循环内 `vsetvli` 是 e16/m4 ↔ e32/m8 的**真实 SEW/LMUL 切换**（load 需要 e16、widen 后算术需要 e32），并非"兼容且可复用的 vl/vtype 被冗余重建"；改写数据访问算法并不能消除该切换 → 排除。
  - register-group-utilization row：LMUL 已达 e32/m8 合法上界（VLMAX=32），peak live set（acc m8 + work m8 + load m4）无 spill，非"LMUL 小于合法边界" → 排除（其行内判据"LMUL 已达合法边界…转 unroll row"）。
  - jit-generated-code-quality row：无 emitter 表示/relocation/重复序列 bug 证据；loop 结构是算法/微结构设计决策，非 emitter 合同未传播 → 排除。
- **(c) 双 Confidence 推导式**：
  - `route: 区间聚合证据（.9 sum 循环 6/12）+ 同文件 f32 kernel 结构对照 + LMUL 已达 m8 上界（进入本 row 的先决）→ Medium（受 baseline_gap: sampling IP precision 限制，仅区间级）`；
  - `impact: 样本量低、函数贡献未知、row 形状未知、vsetvli 单指令份额可能含 vfwcvt skid → Low`。

#### 多命中仲裁小段

两个 finding 机制可分账：Finding 1 修正对象是**算法 pass 结构**（pass 数 3→2，改变整段地址流），Finding 2 修正对象是**每个 pass 内的 loop 微结构**（固定 VL 主循环+展开、vtype 切换摊薄、独立 tail），修复对象与验证方法均可分离 → `independent`，同层不排序（函数级贡献未知 → 依 `baseline_gap: sampling metadata`，收益上界只写局部份额，同层 independent 保持 unranked）。无 companion 白名单命中。两个 finding 均落在 L1/L4 命名层。

**Omitted candidates（route gate 不成立或证据不足，不列为 matched finding）**：
- kernel-operation-fusion：无中间结果内存往返，pass 间依赖关系由寄存器保持 → 门不成立。
- vector-state-management：vsetvli 为必要 SEW/LMUL 切换 → 门不成立。
- eliminate-unnecessary-precision-conversions（f64 inv 段 0x158–0x174）：f32↔f64 往返确实存在（`fcvt.d.s×2/fadd.d/fsqrt.d/fdiv.d/fcvt.s.d`），但 (1) 该段在 .9/.3 均 **0 直接样本**（附近样本 0xd4/0x140/0x148 归属 vfredosum 尾段，IP precision 未知）；(2) 每 row 仅执行一次，不在向量循环内；(3) f32-only 语义合同未获 reference 容差证明（作者显式选择 f64）→ 省略为 matched finding，仅记录观察。
- jit-generated-code-quality：无 emitter 合同证据。

## Phase 4 — Root-cause blueprint / 根因蓝图：jit_rvv_layernorm_f16_fused_kernel_t.3

### Finding 1（primary，归一化多遍扫描）— 依据 `patterns/rvv_normalization_kernels.md`

1. **Root cause**：`jit_rvv_layernorm_f16_fused_kernel_t` 每行（row）对输入执行 **3 个完整向量 pass**：sum 循环（0x98，Σx）→ mean 标量归约（0xc4）→ var 循环（0x104，Σ(x−mean)²，含 `vfwcvt` 宽化）→ var/inv 标量段（0x134–0x174）→ out 循环（0x184，x−mean 再 `*inv` 并窄化写回）。输入 f16 数据被读取并宽化为 f32 共 **3 次**，而 LayerNorm 只需 2 次遍历即可完成（一次统计 pass 同时累加 Σx 与 Σx²，一次归一化 pass）。依据 pattern 独有内容：`rvv_normalization_kernels.md` §Why this is slow "多阶段完整扫描造成额外流量"及 §The fix §5 "使用安全的分块归约计算统计量（计算 sum 和 squareSum）"（`variance = squareSum / N - mean * mean` 必须明确消减误差风险）。对 compute-bound（IPC 0.676）的 kernel，第三个 pass 是纯开销。
2. **The fix / 修复方式**（诊断蓝图，不直接实施）：
   - 修复前（当前结构，3 pass，伪代码）：
     ```
     pass1: for chunk: vle16 → vfwcvt → vfadd(sum)        // Σx
     mean = sum/N;  var  = (pass2) Σ(x−mean)²/N           // Σx² pass
     inv = 1/sqrt(var+eps)
     pass3: for chunk: vle16 → vfwcvt → (x−mean)*inv → vfncvt → vse16
     ```
   - 修复后（2 pass，pattern §5+§6 形态）：
     ```
     pass1: for chunk: vle16 → vfwcvt → vfadd(sum) → vfmacc(sqsum)   // Σx 与 Σx² 同一 pass
     mean = sum/N;  var = sqsum/N − mean*mean;  inv = 1/sqrt(var+eps)
     pass2: for chunk: vle16 → vfwcvt → (x−mean)*inv → vfncvt → vse16
     ```
   - 适用前提：归约顺序允许（当前 kernel 已用 `vfredosum.vs` 有序归约；合并后每 lane 独立累加 Σx/Σx²，最终归约树不变）；`variance = squareSum/N − mean*mean` 的消减误差在 benchdnn 参考容差内可接受——**必须先做数值验证**（pattern §5 红线：reference 若用 Welford/Kahan 不得改简单 sum/squareSum）。
   - 不可破坏的 correctness contract：eps 语义（当前 `var+eps` 用 f64 计算，合并后须保持同一参考合同）；`vfredosum` 有序语义；`tu` tail 政策下 accumulator 跨 chunk 的 lane 对齐；mean/var 写回（`fsw`）与 `save_stats` 时点不变。
   - 限制/风险：大 mean/小 variance 时 `E[x²]−mean²` 消减误差（f16 输入 11-bit mantissa，f32 累加一般可承受，但需对全等值/极端动态范围输入验证）；小 C 时 pass 合并收益有限。
   - 修复后预期 Profile signals：annotate 中 **0x104–0x130 var 循环整段消失**，sum 循环区间内新增一条 `vfmacc.vv`；每元素 vle16/vfwcvt/vsetvli 次数减少约 1/3；`cycles/element` 下降。
3. **Baseline facts 回填**：hardware ISA=rv64…`v`+`zvfh`（C920v2, RVV 1.0）；build ISA=`baseline_gap: build ISA`（JIT 发射，实证 RVV 1.0）；VLEN=128（VLMAX(e32,m8)=32）；bound type=compute-bound（IPC 0.676）。
4. **收益上界**：`当前 sampled event 下的局部样本份额`。`.3` 中 var 循环区间 0/3；同码 `.9` 中 var 循环 1/12、sum 循环 6/12。合并消除第 2 个 pass（约 1/3 向量遍历工作量）；函数级 workload 贡献未知 → 不作 Amdahl 上界（`baseline_gap: sampling metadata`）。
5. **三维路由判定**：
   - current source：JIT-generated（`jit_rvv_layernorm_f16_fused_kernel_t::generate()`，Xbyak_riscv emitter，工作区源码可直接核对）。
   - implementation existence/reachability：kernel 由 `rvv_layer_normalization.cpp` L82-93 每行 `parallel_nd` 调用，f16 路径唯一实现，可达且被采用。
   - function-level policy：benchdnn lnorm f16 primitive 的 rv64 后端；无独立 `.S` 载体政策；不属于 missing-`.S` 分支。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支；本 finding 的 shape 证据见 Finding 2 的寄存器预算核算）。
7. **Related PRs**：
   - `Related PRs：18 条 URL`（rvv_normalization_kernels.md §Related PRs：oneDNN 36df719b3bc9 / 6dc3e2d84eec / 3b90f9d0650f / #4809 / #4622 / #4453 / #4480 / 0d8f4a9e702b / 7ad03324c2d1 / 3c8c37bab64f / c07b7f4e7776 / #4734 / #4491 / 25cd4a75095a / b8360ec0a1d5；MNN #4508 / #4044 / b7268aa）。

### Finding 2（independent，固定-VL 主循环 + 展开摊薄 vsetvli）— 依据 `patterns/rvv_register_budgeted_loop_unrolling.md`

1. **Root cause**：f16 kernel 每个 pass 每 32 元素 chunk 执行 2 条 `vsetvli`（0x9c e16/m4 → 0xb4 e32/m8，out 循环另有 0x198/0x1a4）+ 1 条 `vmv.v.x` tail 清零 + `slli/add/sub/j` 4 条标量控制指令；sum 循环区间在 .9 占 50% 且样本集中在这批配置/控制指令上（0xb4 vsetvli 4/12、0xbc 1/12、0xc0 1/12）。依据 pattern 独有内容：§Why this is slow "过细的 main-loop iteration 会重复 branch、counter、pointer 和 address update…按 vector block 数量累积"及 §The fix 的 fixed-VL main loop + `UNROLL_FACTOR` frontier（`step = vl * UNROLL_FACTOR`、独立 accumulator、runtime-VL tail 精确覆盖剩余元素）。当前 LMUL 已达 m8 上界（§When to apply："只有当前 LMUL 已达到合法或目标硬件实测最优边界…才由本 pattern 认领"）；同文件 f32 kernel（L416-520）已用固定 VL + 4×VLMAX block（循环内 0 vsetvli + 独立 tail 循环），f16 kernel 未沿用。
2. **The fix / 修复方式**（诊断蓝图）：
   - 修复前（每 chunk 运行时 vsetvli，伪代码）：
     ```
     while (len) { vl = vsetvli(len, e16/m4); vle16; vfwcvt;
                   vsetvli(vl, e32/m8); op; vmv.v.x zero; len -= vl; }
     ```
   - 修复后（固定 VL 主循环 + 多 chunk 一轮 + 独立 runtime-vl tail）：
     ```
     vsetvli(epr, e16/m4);          // VLMAX=32，循环外一次
     for (; len >= k*32; ) {
        vle16  ×k（k 个 m4 load 组）;
        vfwcvt ×k（k 个 m8 work 组）;
        vsetvli(e32/m8);            // 每轮仅一次切换
        op     ×k（k 个独立 accumulator 或同一 per-lane acc）;
        vsetvli(e16/m4);            // 下一轮 load 前一次切换
        ptr += k*64B; len -= k*32;
     }
     tail: while (len) { vl = vsetvli(len, ...); ... }   // runtime-vl
     ```
   - 寄存器预算（VLEN=128，e32/m8）：`LMUL*peak_live ≤ 32`。3-way 展开：acc m8（8 组）+ work m8 ×3（24 组）= 32 组正好占满（load 组复用 work 低半组或 0xfc 预清空寄存器）→ k=3 为架构上限；k=2 留余量更稳（acc m8 + work m8×2 = 24 组）。每 unrolled lane 若拆独立 accumulator 则 k 需相应下调。
   - 正确性 contract：主循环只处理完整 VLMAX chunk（不 over-read），tail 精确覆盖 `len mod (k*32)`；`step/tail` 与 dispatch 同步；由于最终 `vfredosum` 对 32 条 lane 的归约顺序不随 chunk 迭代次序改变，**浮点归约顺序不变**（不引入 reassociation 变化）；主循环可去掉逐 chunk `vmv.v.x` tail 清零（tail 循环仍需）。
   - 限制/风险：k=3 时寄存器组全占、无 spill 余量（live interval 须逐指令核对）；I-cache/代码体积；短 row（C≤32）时收益趋零。
   - 修复后预期 Profile signals：0xb4/0x120/0x1a4 的 vsetvli 与 0xbc/0x12c 的 vmv 在主循环区间样本大幅下降；回边/控制指令每元素占比下降；`vsetvli` 计数从 2/chunk 降到 ~1/(k·chunk) 量级。
3. **Baseline facts 回填**：同 Finding 1；另记录寄存器预算核算（LMUL=8 上界、k≤3）。
4. **收益上界**：`当前 sampled event 下的局部样本份额`（.3：0xc0 回边 1/3 + 0x1b0 控制 1/3；.9：sum 循环区间 6/12 中配置/控制指令占 5/12）；函数级贡献未知 → 不作 Amdahl 上界。
5. **三维路由判定**：current source=JIT-generated；implementation existence=该文件 f32 kernel 已存在同构固定-VL+block 结构（可链接候选在 emitter 内合法）；function-level policy=无独立 `.S` 政策。
6. **Implementation-shape proof**：不适用（非 missing `.S` 分支）；寄存器预算以 §The fix 的 `k≤3` 核算为准，`implementation_shape_gap: 逐指令 live-interval 反汇编核对（需补采 disassembly dump）`。
7. **Related PRs**：
   - `Related PRs：2 条 URL`（rvv_register_budgeted_loop_unrolling.md §Related PRs：XNNPACK 0c7b565c2fc1 / #10403）。

## Phase 5 — Verification forecast / 验证预测：jit_rvv_layernorm_f16_fused_kernel_t.3

- **Finding 1（primary，3→2 pass 合并）**：
  - 应消失/缩小：var 循环整段（锚定 `.3` annotate 0x104–0x130：`0x110: vle16.v v24,(a1)`、`0x124: vfsub.vf v16,v16,ft1`、`0x128: vfmacc.vv v8,v16,v16`、`0x130: j 104`）；重建后 annotate 应只剩 2 个主循环（sum+out 区间），var 区间地址流不再出现。
  - 应出现：依据 `rvv_normalization_kernels.md` §Verification——"多余完整 pass share 下降，出现 vector reduction + transform，cycles/row 或 throughput 改善"；sum 循环区间新增 `vfmacc.vv` 且该区间仍健康；`fdiv.s`/inv 段保留；benchdnn 正确性容差内（epsilon、zero variance、全等值输入、极端 mean/σ 动态范围）。
  - 数值专项：对 `variance = squareSum/N − mean*mean` 与当前 `Σ(x−mean)²/N` 在 f32 下比对参考容差；不满足则回退当前稳定形式（该合同是本 finding 成立的前提）。
- **Finding 2（independent，固定-VL+展开）**：
  - 应消失/缩小：主循环区间内 `0x9c`/`0xb4`（e16/e32 vsetvli）、`0xbc`（vmv）等逐 chunk 配置指令样本（锚定 `.3` 0x98–0xc0 区间与 `.9` 0xb4 4/12、0xbc 1/12、0xc0 1/12）；回边 0xc0 每元素占比下降。
  - 应出现：依据 `rvv_register_budgeted_loop_unrolling.md` §Verification——"每个输出对应的 backedge/control/address-update 减少…没有新 spill/非法内存访问"；tail 循环正确覆盖 `len mod k*32`；k=2/3 反汇编无 vector spill。
  - 补采升级项：**要把结论升级到 profile-backed，最少需采集**：同一 window 下对单一实例（.9 或 .3）用 `precise_ip` 重采 `perf record`（记录 `--header-only` 的 event attr），获取函数级 sample share 与单指令归属；并记录 benchdnn lnorm 的 (N, C) 形状配置。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：N=1 组 Phase 3–5 全部出现 | ✅（`1/1 组；jit_rvv_layernorm_f16_fused_kernel_t.3`） |
| 2 | Phase 1 输出要求：7 行 baseline + Sampling IP precision 行 | ✅（结论项 7 + gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`、`evidence_gap: sample count`） |
| 3 | Phase 3 输出要求：8 项 Class selection trace + Classes scanned + 三件套 | ✅（`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md`；顶层 finding 2（1 primary + 1 independent）；evidence 锚点 `0xc0: j 98` / `0xb4: vsetvli` / `0x104-0x130 var 循环` / `0x1ac: vse16`；supporting 0；排除条数 4（elementwise-activation / widening-reduction / precision-conversion / kernel-operation-fusion）；omitted 4（fusion / vector-state / f64-precision / jit-code-quality）；推导式 2 条） |
| 4 | Phase 4 输出要求：pattern 已读 + 命中 row + The fix 七要素 | ✅（`rvv_normalization_kernels.md` §5/§6 + §Why；`rvv_register_budgeted_loop_unrolling.md` §The fix + §Why；The fix 均含 before/after、correctness contract、风险、Profile signals；`Related PRs：18 条 URL` / `Related PRs：2 条 URL`） |
| 5 | 路径合规：8 项 trace + 零/多命中仲裁 + L1/L4 层归属 + 动态份额排序约束 | ✅（primary=L1 normalization、independent=L4 unrolling；`baseline_gap: sampling metadata` → 只写局部份额、同层 unranked；无 `th.v*` 停扫） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧 `0x104–0x130`/`0x98–0xc0`/`0xb4` 锚点；出现侧 `rvv_normalization_kernels.md` §Verification、`rvv_register_budgeted_loop_unrolling.md` §Verification） |
| 7 | 契约边界合规：无实施询问/代码修改/补丁生成/追问 | ✅（交付物止于证据、根因蓝图、The fix、验证预测；无 object-clarification 以外提问） |

修正记录：无