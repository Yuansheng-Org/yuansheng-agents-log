Functions under analysis: [ossl_x448_int]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`ossl_x448_int`，libcrypto.so.4，cpu-clock，71 samples，percent type: **local period**；覆盖整条 X448 ladder 主循环，coverage 完整）
- perf stat（可选 bound/context）：已提供（`1-openssl-benchmark-riscv-x448.txt`；IPC 2.338、branch_miss 0.474%、L1_dcache_load_miss 0.212%；cache_references/cache_misses/LLC 未提供）
- workload/binary/DSO/source context：已提供（OpenSSL master @ 2924476b5591e691e904c4baf57894c526c4b8de，riscv64 SpacemiT X100；annotate 源码行映射到 `crypto/ec/curve448/curve448.c` / `field.h`，本函数为 compiler-generated C，非 `.S`；`ossl_gf_mul`/`ossl_gf_sqr` 为跨 TU 实调用，见 `jal 15b9a4 <ossl_gf_mul>`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：部分缺失——冻结快照只对 `openssl_bench` 可执行文件运行了 readelf（`rv64i2p1_..._zcd1p0`，无 `v`）；承载热点的 `libcrypto.so.4` 的 attribute 未提供。但 annotate 中该 DSO 明确执行 `vsetvli`/`vle64.v`/`vrgather.vv` 等 RVV 1.0 指令，证明 libcrypto.so.4 实际以含 `v` 的 ISA 构建（详见 Phase 1 `baseline_gap: build ISA`）
- hardware ISA：已提供（metadata cpuinfo：`rv64imafdcvh_..._zvbb_zvbc_zve64d..._zvk*`，含 `v`、Zvbb、Zvbc、Zvkg、Zvkned、Zvknha、Zvknhb、Zvksed、Zvksh、Zvkt、Zbc、Zbb、Zba）
- `vlenb`：已提供（VLEN=256 bits，vlenb=32，e64/m1 下每向量 4 lane）
- 采样元数据（event / percent type / scope / 窗口）：部分已提供（event=cpu-clock，percent type=local period，单次运行窗口；函数级 workload 贡献未知——本函数是该 benchmark 内 4 个热点函数之一，rank 003；详见 Phase 1 `baseline_gap: sampling metadata`）
- Sampling IP precision：缺失（无 `precise_ip`/Exact-IP 信息；详见 Phase 1 `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_..._zvbb_zvbc_zve64d_zve32x...`：暴露标准 RVV 1.0（`v`）+ Zvbb/Zvbc/Zvkg/Zvkned/Zvknha/Zvknhb/Zvksed/Zvksh/Zvkt/Zbc/Zbb/Zba（metadata cpuinfo） |
| Build ISA | `baseline_gap: build ISA`（精度：冻结快照的 readelf 对象是 `openssl_bench` 可执行文件而非承载热点的 `libcrypto.so.4`）。热点 DSO 实际含 `v`：annotate 显示 `vsetivli/vle64.v/vrgather.vv/vse64.v/vid.v/vrsub.vi`（RVV 1.0）。故 hardware-vs-build **无 L0 mismatch**（对热点对象而言），但精确 `Tag_RISCV_arch` 缺该 object 的直接证据；可选命令：`readelf -A <libcrypto.so.4>` |
| Vector flavor | 标准 RVV 1.0 `v*` mnemonic（`vsetvli`/`vle64.v`/`vrgather.vv`/`vsrl.vx`/`vand.vv`/`vadd.vv`/`vse64.v`/`vid.v`/`vrsub.vi`/`vmv.v.x`/`vle8.v`）；无 `th.v*` → 无 flavor mismatch，RVV 依赖 route 冻结生效 |
| VLEN | 256 bits（vlenb=32）；e64/m1 满宽应为 4 lane，但 GCC 实际用 `vsetivli zero,2,e64,m1`（vl=2，仅用 16/32 字节） |
| Bound type | compute/latency-bound：IPC 2.338（高）、L1_dcache_load_miss_rate 0.212%、branch_miss_rate 0.474%、无 LLC counter；非 memory-bound、非 branch-bound → 向量级修正有收益空间，不被 cache/branch 证据反驳 |
| Sampling semantics | `baseline_gap: sampling metadata`：event=cpu-clock（时间可解释 ✓），percent type=**local period**（✗ 非 global-period），同一运行窗口（✓），函数 workload 贡献未知（✗）→ 收益只能表述为「当前 sampled event 下本函数局部样本份额」，不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`：无 precise_ip/Exact-IP 信息 → 单行占比只能锚定所属 basic block / loop interval，不承担单指令 latency 归因 |

L0 baseline gate：hardware 有 `v`、热点对象实际执行 `v*`，无「hardware 有 v 而 build 无 v」mismatch（`openssl_bench` 可执行文件本身无 `v` 仅影响 bench 自身代码，不影响 libcrypto.so.4 热点）；无 `th.v*`。Bound-type gate：compute-bound 成立，不降级向量级 finding 的 route；impact 仍受采样语义约束（→ Low）。
Vector flavor 结论：RVV 1.0（VLEN=256），flavor-dependent route 正常生效。

## Phase 2 — Scope / 分析边界

- 函数清单：`ossl_x448_int`（1 个），与承诺声明一致。
- hot interval 锚点（local period 最高占比行）：ladder 每 bit 的 `gf_weak_reduce` 向量区间——`8.45 : 15ddc2: vle64.v v5,(s2)`、`8.45 : 15de88: vle64.v v5,(a5)`；同区间第二负载族 `5.63 : 15df82: vle64.v v5,(a5)`、`5.63 : 15e048: vle64.v v5,(a7)`、`4.23 : 15e146: vle64.v v5,(t3)`、`4.23 : 15e1f8: vle64.v v5,(s7)`。六处 `vle64.v v5` 局部占比合计 42.3%。
- annotate 覆盖完整：主循环（15dbb2–15e270）与 finish 段（15e29a–15e3e8）均在。
- Sampling IP precision 不足 → 所有占比仅锚定 interval（weak_reduce 向量循环 / 其标量 epilogue / gf_sub_RAW 标量循环 / gf_cond_swap 循环），不做单指令 cycle 归因。
- 归属按地址分账：weak_reduce 全机制（向量循环 + 标量 epilogue + tmp 装载，地址段 15dc1c–15dc8c、15dce4–15dd4a、15dda6–15de1e、15de6e–15dee6、15df68–15dfdc、15e032–15e09a、15e130–15e19c、15e1da–15e24a）≈ 70% 局部样本（其中纯向量循环 vle64.v v5 六处 42.3%、vand.vv 1.4%、标量 epilogue/tmp 段 ≈ 26.8%）；`gf_sub_RAW` 标量 8-limb 循环（15dc9e–15dce0、15dd50–15dda2、15df1a–15df64、15e0cc–15e12c）≈ 11.3%；`gf_cond_swap` 标量循环（15dbb2–15dbf2）≈ 7.0%；`gf_add_RAW` 向量循环（15dc02–15dc18、15de54–15de6a、15e018–15e02e）≈ 2.8%；跨 call spill/reload（15de46、15dfd0、15e00c）≈ 4.2%。

## Phase 3 — Pattern scan / 模式扫描：ossl_x448_int

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源为 compiler-generated C（annotate 内联 DWARF 源码行映射 field.h/curve448.c；`jal 15b9a4 <ossl_gf_mul>`/`jal 15bd66 <ossl_gf_sqr>`/`jal 15bc96 <ossl_gf_mulw_unsigned>` 为跨 TU 函数），无手写 `.S` provenance；missing-`.S` 四证不齐：crypto/ec/asm/ 下无任何 curve448/x448 asm（仅 `ecp_sm2p256-riscv64.pl`、`x25519-x86_64.pl` 等），field ops 无 dispatch slot（直接调用），无要求 x448 独立 `.S` 的 function-level policy。
2. `rows-operator-rvv.md` — **include** — compiler-generated 代码，热点含未向量化 scalar 8-limb 循环（sub/cond_swap）与已向量化区间（add/weak_reduce）。
3. `rows-string-memory.md` — **exclude** — 无 copy/fill/sentinel/compare 语义热点；`gf_copy` 的 `vle8.v/vse8.v` 仅函数入口 cold 路径（15dab4–15db30，0.00%）。
4. `rows-vectorized-tuning.md` — **include（必选）** — annotate 含大量 `v*`，代码来源为 compiler/intrinsic 生成（非手写 `.S`），修正对象为 RVV 配置/公式化。
5. `rows-codegen.md` — **include** — 736B stack frame、跨 call spill/reload（`sd a7,-664(s0)`/`ld a7,-664(s0)` 族）、栈上 gf limb 反复装载（`ld a5,-448(s0)` 7.04%）等 codegen 形态证据。
6. `rows-offload.md` — **exclude** — X448 无矩阵引擎/packed-SIMD 参与。
7. `rows-crypto.md` — **include（评估后排除）** — X448 为密码原语，但该 class 4 个 row 的 gate 全部不成立：Curve448 为素数域 p=2^448−2^224−1 的 56-bit limb 折叠（add/sub/shift/and/mul），非 GF(2^k)（无 clmul/vclmul 语义 → carry-less row 排除）；无 AES/SHA/SM3/SM4/GHASH/CRC 语义（dedicated-vector-crypto、cipher-mode 排除）；无 lookup-table 热点（polynomial-table row 排除）。
8. `rows-runtime-os.md` — **exclude** — 非 RTOS/kernel/CSR/timer 热点。

### Classes scanned: `rows-operator-rvv.md`, `rows-vectorized-tuning.md`, `rows-codegen.md`, `rows-crypto.md`

### Local performance pattern scan: `ossl_x448_int`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | 热点已向量化但 `vsetivli zero,2,e64,m1`（vl=2，VLEN=256 下仅用半宽）且每 2-limb chunk 用 3 个 `vrgather.vv` 反转；8-limb（64B）数组本可 m1/vl=4 两段或 m2 一段覆盖；weak_reduce 每 bit 调用 ~8 次 × 448 bit | High | Low | `patterns/rvv_register_group_utilization.md` |
| RVV Contiguous Elementwise Arithmetic Kernels（independent） | `gf_sub_RAW`/`gf_cond_swap` 8-limb scalar loop（`ld/sub/addiw/bne`、`ld/xor/and/xor/sd`）未向量化，样本在标量 load/算术/循环控制 | High | Low | `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` |

#### Finding 1（primary）：weak_reduce 向量化欠利用 + vrgather 公式化

**(a) 逐字 evidence 引用**（local period，全部位于 weak_reduce 向量循环 interval，8 次内联实例）：
```
8.45 :  15ddc2: vle64.v v5,(s2)      # weak_reduce(z2) 实例（15ddba–15dde6 区间）
8.45 :  15de88: vle64.v v5,(a5)      # weak_reduce(z2) 实例（15de80–15deaa 区间）
5.63 :  15df82: vle64.v v5,(a5)      # weak_reduce(z3) 实例（15df7a–15dfa4 区间）
5.63 :  15e048: vle64.v v5,(a7)      # weak_reduce(z2) 实例（15e040–15e06a 区间）
4.23 :  15e146: vle64.v v5,(t3)      # weak_reduce(t2) 实例（15e13e–15e168 区间）
4.23 :  15e1f8: vle64.v v5,(s7)      # weak_reduce(t1) 实例（15e1f0–15e21a 区间）
1.41 :  15e058: vand.vv v3,v3,v2
7.04 :  15ddf2: ld a5,-448(s0)       # weak_reduce 标量 epilogue 栈载
```
加上 `15dc20/15dce8/15e036: lbu a3,-19x(s0)`（tmp = limb[7]>>56）、`15e03a: add a5,a5,a2` 等，weak_reduce 全机制局部占比 ≈ 70%。逐字证明向量体为每 2-limb chunk：`vle64.v v3,(a5)`+`vle64.v v5,(s1)` → `vrgather.vv v4,v3,v1`+`vrgather.vv v3,v5,v1` → `vsrl.vx v4,v4,s11` → `vand.vv v3,v3,v2` → `vadd.vv v3,v3,v4` → `vrgather.vv v4,v3,v1` → `vse64.v v4,(s1)`，`vsetivli zero,2,e64,m1` 令 vl=2（VLEN=256 下 e64/m1 满宽 4 lane，此处只用 2 lane）；同一 interval 还有 `15dc38/15dd04: 2.82%` 两个 `vle64.v v5` 实例。同机制补充锚点：`gf_add_RAW` 向量循环（15dc02–15dc18、15de54–15de6a、15e018–15e02e，含 `15e02e: bne s6,a5,15e018` 1.41%）同样使用 `vsetivli zero,2,e64,m1` 半宽迭代（8 limb → 4 次迭代）。
**(b) 互斥邻居排除**：
- `rvv_vector_state_management`（重复建立兼容 vl/vtype）：该 row 行内判据要求「中间无 call/inline-assembly/state clobber」——本区间每次重新建立 `vsetivli zero,2,e64,m1`+`vmv.v.x v2,a5`+`vle64.v v1,(a3)`（从 -688(s0) 栈槽重载 index）都发生在 `jal <ossl_gf_mul/ossl_gf_sqr>` **真实调用之后**，vtype/vl/v0-31 按 psABI 被 call clobber，重建是 ABI 强制而非过早失效 → 该 row 不命中（引用行 `15de2a: li a3,-1`…`15de3a: vle64.v v1,(a3)`）。
- `no-vectorization`：该 row 要求 hot main loop 全 scalar、zero `v*`——本热点区间存在 `vle64.v/vrgather.vv/vadd.vv`（引用行 `15ddc2`），且硬件/构建均支持 `v` → 排除。
- `rvv_operand_form_selection`：该 row 要求 scalar/immediate 先经 `vmv.v.x` 成 temp 再被**单条** op 消费；本区间 `vmv.v.x v2,a5`（56-bit mask）与 `vid.v+vrsub.vi`（index v1）为**循环不变量**、跨多次 weak_reduce 复用（每 bit 仅随 call clobber 重建），不是 per-op splat → 排除。
- `rvv_register_budgeted_loop_unrolling`：按 arbitration「缩短 live range 后更大 LMUL 合法、无 spill 且能直接减少迭代数时，由 register-group sizing 认领、unroll 延后」→ 本 finding 由 register-group row 认领，unroll 不独立命中。
- `register_pressure_and_save_restore`（rows-codegen）：栈上 gf 数组是 7 个 × 64B 长生命周期 temporary 且横跨真实调用，内存驻留由调用边界 + 数组总量强制（RA 无合法寄存器化余地）；spill/reload（`15dfd0: sd t3,-656(s0)`、`15e00c: ld t3,-656(s0)`）是调用边界的指针保存，不是 RA 可消除的局部 spill → 该 row 的「RA/live 可改进」gate 不成立，排除（作为机制背景记录）。
- `kernel_operation_fusion`（跨 pass 融合）：weak_reduce 是 gf op 语义合同的一部分（`gf_add_RAW`/`gf_sub_RAW` 内联调用），非独立可融合 pass → 排除。
**(c) 双 Confidence 推导式**：`route: 热点已向量化 + 非 .S compiler-generated + 当前 LMUL(m1/vl=2) 低于合法边界(m1/vl=4、m2/vl=8) + live set 允许 m2 无 spill（峰值 live ≤ v1,v2+3 temp ≈ 5 group ≤ 32 寄存器）+ 行内互斥逐项排除（vector-state/operand-form/no-vectorization/unroll）→ High；impact: 局部 sample share 充分（~70%）但采样语义缺 global-period 与函数级贡献（baseline_gap: sampling metadata）、IP precision 未知 → Low`。

#### Finding 2（independent）：gf_sub_RAW / gf_cond_swap 标量 8-limb 循环未向量化

**(a) 逐字 evidence 引用**（local period）：
```
1.41 :  15dcac: ld a7,0(a2)          # gf_sub_RAW(t2) 循环体（15dcaa–15dce0 区间）
1.41 :  15dc9e: addiw a4,a4,1        # gf_sub_RAW 循环计数
1.41 :  15df36: ld a3,0(a1)          # gf_sub_RAW(z3) 循环体（15df34–15df64 区间）
1.41 :  15df48: sd a5,-8(a0)
1.41 :  15df4c: ld a5,8(a1)
1.41 :  15e0fe: ld a3,0(a1)          # gf_sub_RAW(t2)（15e0fc–15e12c 区间）
1.41 :  15e10e: add a5,a5,a4
4.23 :  15dbca: sd a4,-8(a1)         # gf_cond_swap(x2,x3) 循环（15dbb2–15dbf2 区间）
1.41 :  15dbb4: ld a4,0(a1)
1.41 :  15dbee: sd a4,-8(a1)         # gf_cond_swap(z2,z3) 收尾
```
两族标量循环局部占比 ≈ 18.3%（sub 11.3% + cond_swap 7.0%）。`gf_sub_RAW` 源语义（field.h §arch_64）：`out->limb[i] = a->limb[i] - b->limb[i] + ((i == NLIMBS/2) ? co2 : co1)`，每输出只依赖同索引输入 + 每 lane 常量；`gf_cond_swap` 源语义（field.h）：`*a ^= xor; *b ^= xor; xor = (*a ^ *b) & value_barrier_64(mask)`，为寄存器级两源选择。
**(b) 互斥邻居排除**：
- `no-vectorization`：该 row 只认领「未被更具体 semantic/codegen row 解释的 generic scalar main loop」；本循环是具名 gf 域算术（elementwise sub + lane 常量 bias；select）→ 更具体的 elementwise row 认领，no-vectorization 排除。
- `rvv_register_group_utilization`：该 row 只认领已向量化区间的 LMUL/live-set 根因；sub/cond_swap 区间 **zero `v*`**（纯标量）→ 排除。
- `rvv_layout_and_channel_packing` / `rvv_gather_indexed_memory_access`：本区间为 unit-stride 连续 8-limb 访问，非 packing/gather → 排除。
- `rvv_vector_state_management`：标量区间无 vtype 状态 → 排除。
- `zero_based_comparison` / `loop_induction_variable_strength_reduction`：sub 循环已用指针递增 + `li a4,0` 计数比较（`beq a4,a6`），非 index-scaling 主导；`li a4,0` 每轮循环初始化一次非热路径形态 → 不作为独立命中，排除。
**(c) 双 Confidence 推导式**：`route: compiler-generated scalar loop + 同索引依赖 + unit-stride + 语义为 elementwise sub/select + hardware/build 有 v + 无行内互斥残留 → High；impact: 局部 sample share ~18% 但采样语义缺 global/贡献（baseline_gap: sampling metadata）、IP precision 未知 → Low`。

#### 多命中仲裁

- **Finding 1 = primary**：weak_reduce 向量区间（15dc1c–15e24a 族）局部样本 ≈ 70%，机制为 GCC 对 8-limb 弱归约的 `vl=2`+vrgather 公式化欠利用；因果层次 L3/L4（向量配置/微结构），机制层归属 `rvv_register_group_utilization.md`（rows-vectorized-tuning Row 5）。
- **Finding 2 = independent**：地址不相交（15dbb2–15dbf2、15dc9e–15dce0、15df2c–15df64、15e0e2–15e12c 标量区间 vs 向量区间），机制可分离（「未向量化的具名 elementwise 循环」vs「已向量化区间的 LMUL/gather 公式化」），修复对象与验证方法各自独立 → 并列输出，不做主从合并。
- supporting：`gf_add_RAW` 向量循环（`15e02e: bne s6,a5,15e018` 1.41%）的半宽 `vsetivli zero,2,e64,m1` 与 Finding 1 同一机制（GCC 对本 TU 固定尺寸循环统一选 vl=2），只作 Finding 1 的 supporting evidence，不单独成 finding。
- 未列名机制兜底：无。
- 排序：入口模式 A 有局部 sample share（71 samples，local period），但采样语义 gate 未过（非 global-period、贡献未知）→ 收益上界按「当前 sampled event 下的局部样本份额」表述，不称 workload 级 Amdahl 上界。

## Phase 4 — Root-cause blueprint / 根因蓝图：ossl_x448_int

### Finding 1（primary）— `patterns/rvv_register_group_utilization.md`（对应 rows-vectorized-tuning Row 5：RVV Register-Group Utilization and LMUL Sizing）

1. **Root cause**：X448 ladder 每 bit 执行 ~8 次 `gf_weak_reduce`（`gf_add_nr`×4 + `gf_sub_nr`×3 + `gf_mulw`×1，每次 add/sub 尾部内联），每次对 8×56-bit limb（64B）做纯并行弱归约。GCC autovec 将其生成为 `vsetivli zero,2,e64,m1`（vl=2，**只用 VLEN=256 的一半**）的 2-limb chunk 循环，并用 `vrgather.vv` 反转索引实现相邻 limb 移位窗口——每 chunk 3 个 vrgather（反转输入×2 + 反反转输出），8 limb 共 9 个 vrgather + 3 次循环迭代 + 标量 epilogue（`lbu`/`ld`/`and`/`add`/`sd`）。依据 `patterns/rvv_register_group_utilization.md` §Why this is slow 第 1 条「当前 LMUL 小于合法且无 spill 的候选边界时，单次迭代的有效元素数偏低，循环、vsetvl 和分支开销按更多迭代重复发生」；§Why this is slow 第 3 条「widen/narrow、gather 和类型转换要求 source 与 destination 的 LMUL 成固定倍率…抵消放大 LMUL 的收益」；§典型优化场景表「gather/转换的 index 与 data 宽度不匹配 → 为 index 选择匹配的 fractional LMUL/EEW」。weak_reduce 全机制局部样本 ≈ 70%（六处 `vle64.v v5` 锚点合计 42.3% + 同区间标量 epilogue/tmp 段 ≈ 26.8% + 循环分支 1.4%），为函数内最大单一机制。
2. **The fix / 修复方式**（依据 pattern §The fix 1/2/3：先做 `LMUL * peak_live_vectors <= 32` 预算核对，再枚举合法 LMUL 候选；本 kernel 峰值 live ≈ index v1 + mask v2 + 2 load + 1 结果 ≈ 5 group，m1 时 5×1=5≤32、m2 时 5×2=10≤32 → m2/vl=8 合法）：
   - 修正对象：GCC 对 `gf_weak_reduce` 的向量化公式化（vl=2 + vrgather 反转），不改变弱归约算法语义。
   - 修复前（当前生成形态，每 2-limb chunk，×3 次/调用）：
     ```asm
     vsetivli zero,2,e64,m1
     vle64.v  v3,(a5)          # limbs[i-1..i]
     vle64.v  v5,(s1)          # limbs[i..i+1]
     vrgather.vv v4,v3,v1      # 反转
     vrgather.vv v3,v5,v1      # 反转
     vsrl.vx  v4,v4,s11        # >>56
     vand.vv  v3,v3,v2         # & mask
     vadd.vv  v3,v3,v4
     vrgather.vv v4,v3,v1      # 反反转
     vse64.v  v4,(s1)
     addi s1,s1,-16 ; bne …
     ```
   - 修复后（RVV 公式化：全宽 + vslide1up 替代 vrgather；m2/vl=8 一次覆盖 8 limb，或 m1/vl=4 两段；按 pattern §The fix 2/3 的 LMUL 枚举与 §7 接入）：
     ```asm
     # 每次调用：
     lbu   t, 63(x)            # tmp = limb[7] >> 56
     ld    t2, 32(x); add t2,t2,t; sd t2,32(x)   # limb[4] += tmp
     # 主循环外不变量：v2 = 56-bit mask（vmv.v.x 一次）；vl=8（m2）
     vle64.v    v3,(x)         # 全 8 limb
     vslide1up.vx v4,v3,t      # v4 = [tmp, limb0..limb6]（lane0 即 limb[0] 的 +tmp）
     vsrl.vx    v4,v4,56
     vand.vv    v3,v3,v2
     vadd.vv    v3,v3,v4
     vse64.v    v3,(x)
     ```
     每调用向量 op 数：~27+3 迭代 → ~6（m2）；移除全部 9 个 vrgather；`gf_add_RAW` 向量循环同法从 vl=2 提到 vl=4/m2（supporting）。
   - 适用前提：硬件/构建支持 `v`（✓）；VLEN≥256（✓，vlenb=32）；固定 8-limb 数组（NLIMBS=8）与 56-bit mask 语义；常数时间要求不被破坏（vslide1up/vle/vse 无数据相关分支）。
   - correctness contract：`gf_weak_reduce` 语义——先 `limb[4] += tmp`，随后所有 `limb[i] = (limb[i]&mask) + (limb[i-1]>>56)` 读到的都是更新后的旧值（倒序循环无跨迭代依赖，limb[i-1] 在迭代 i 执行时尚未被写），`limb[0] = (limb[0]&mask) + tmp`。vslide1up 的 lane0=tmp 恰好实现 limb[0] 路径；其余 lane j 得到 limb[j-1]>>56。LMUL 改变不改变整数数值结果。
   - 限制/风险：vrgather 与 vslide1up 的实机吞吐需在 X100 上 A/B（部分核 slide 也是多 uop）；m1/vl=4 与 m2 两候选必须实测与 spill 检查；GCC 自动向量化不保证采纳该形态——需以 intrinsic kernel 显式表达或调整源码形态，并验证生成代码；若走 intrinsic 变体，须保留 generic C fallback（pattern §The fix 5/7：autovec/generic 保留为 fallback，不无条件替换）。
   - 预期 Profile signals：weak_reduce 区间 `vrgather.vv` 全部消失；`vle64.v/vse64.v` 每调用降为 1–2 次；`vsetivli` 出现 vl=8（m2）或 vl=4（m1）；`15ddf2: ld a5,-448(s0)` 等标量 epilogue 栈载显著减少；局部样本份额大头（vle64.v v5 六处 42.3%）预期明显缩小。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcvh_…_v_zvbb_zvbc_zve64d…（含 `v`）；build ISA=热点 object libcrypto.so.4 含 `v`（annotate 证实；精确 attribute `baseline_gap`）；VLEN=256（vlenb=32）；bound type=compute-bound（IPC 2.338，L1 miss 0.212%）。
4. **收益上界**：当前 sampled event（cpu-clock）下本函数局部样本份额 ≈ 70%（weak_reduce 全机制）。采样语义四条未全过（percent type=local、函数贡献未知）→ 只表述方向性收益，不估算 workload 级 Amdahl 倍数。`dynamic priority unavailable` 不适用（模式 A，但收益不可称全局）。
5. **三维路由判定**：
   - current source：compiler-generated C（`curve448.c`/`field.h` 内联，GCC autovec 生成 `v*`），非 `.S`、非 JIT。
   - implementation existence/reachability：无专用 RVV kernel、无 dispatch slot（field ops 直接调用）；当前路径即该编译产物（annotate 证实执行 `v*`）→ 修正对象是编译产物/源码形态，不涉及 kernel-selection。
   - function-level policy：OpenSSL curve448/x448 全架构无 asm（crypto/ec/asm/ 仅 ecp_sm2p256-riscv64.pl 等，x448 无任何 `.S`）→ missing-`.S` 四证中 policy/existence gate 不成立，不走 assembly-primary 分支；fix 载体为 C/intrinsic 层修正。
6. Implementation-shape proof：不适用（非 policy-backed missing `.S`）。
7. **Related PRs**：`patterns/rvv_register_group_utilization.md` §Related PRs——OpenCV [#26318](https://github.com/opencv/opencv/pull/26318)、[5be158a2b6ed](https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5)、[#25586](https://github.com/opencv/opencv/pull/25586)、Linux Kernel [a4348546332c](https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c)、[a894e8ed09c6](https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0)、[c2a658d41924](https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05)、OpenJDK [bdd37b0e5eaa](https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e)、OpenBLAS [cc1b5794a040](https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870)、[d69be17b6ff7](https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b)、[4a12cf53ec11](https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3b604ca6cb17)、[240695862984](https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946df9327)、V8 [384433993606](https://github.com/v8/v8/commit/3844339936068c529170dcb4f1602aa654d25943)、vLLM [#47538](https://github.com/vllm-project/vllm/pull/47538)。Related PRs：13 条 URL。

### Finding 2（independent）— `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md`（对应 rows-operator-rvv Row 1：RVV Contiguous Elementwise Arithmetic Kernels）

1. **Root cause**：`gf_sub_RAW`（`out->limb[i] = a->limb[i] - b->limb[i] + ((i == NLIMBS/2) ? co2 : co1)`）与 `gf_cond_swap`（常数时间掩码交换 `xor/&/xor/xor`）是纯 elementwise、unit-stride、无跨元素依赖的 8-limb 循环，GCC 却保留标量形态（sub 因 i==4 的 bias 分支未向量化；cond_swap 因常数时间掩码序列未向量化）。依据 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` §Why this is slow 第 1 条「标量循环每次只产生一个结果，同一组指针更新、边界判断和回跳分支需要按元素重复执行。RVV 可以在一次迭代中处理当前 vl 个元素，将固定控制开销分摊到多个结果上」；§Why this is slow 第 2 条「标量实现为每个输入和输出元素分别发出 load/store…增加前端取指、译码和地址生成压力」。局部样本 ≈ 18.3%（sub 11.3% + cond_swap 7.0%）。
2. **The fix / 修复方式**（依据 pattern §The fix 2/6：VLEN-agnostic strip-mined loop；mask compare/select 直接 lowering）：
   - 修正对象：两个 8-limb 标量循环 → RVV strip-mined loop（`vle64`×2 + `vsub` + lane 常量 bias；`vle64`×2 + `vxor` + `vand`(mask) + `vxor` + `vse64`×2）。
   - 修复前（sub，当前形态）：8 次 `ld/sub/addiw/bne`（含 i==4 展开分支，引用行 `15dcac: ld a7,0(a2)`、`15dc9e: addiw a4,a4,1`）；修复后（sub）：`vsetvli a5,8,e64,m1/m2` + `vle64.v v3,(a); vle64.v v4,(b); vsub.vv v3,v3,v4; vadd.vv v3,v3,v5`（v5 = 每 lane bias 常量向量：i==4 lane 为 co2、其余 co1，一次构建复用）`+ vse64.v v3,(out)`——8 limb 一次覆盖。bias 向量每调用不变可循环外构建；或 vmv.v.x 两常数 + vrgather/select 组合（pattern §The fix 6「在语义明确时使用比较和 mask 完成逐元素选择」）。
   - 修复后（cond_swap）：`vle64.v v3,(x); vle64.v v4,(y); vxor.vv v5,v3,v4; vand.vv v5,v5,v6`（v6 = 广播 swap mask，`vmv.v.x`）`+ vxor.vv v3,v3,v5; vxor.vv v4,v4,v5; vse64.v v3,(x); vse64.v v4,(y)`。
   - 适用前提：硬件/构建 `v`（✓）；unit-stride 连续 8-limb（✓）；无符号回绕减法语义（field.h 的 uint64_t limb，模 2^64 回绕 ✓）；常数时间合同——掩码选择无分支、与数据无关（✓，vector 指令与标量同保 CT）。
   - correctness contract：`co1=((1ULL<<56)-1)*2`、`co2=co1-2` 逐 lane 常量；i==4 的 co2 分支必须经 lane 常量精确还原；swap 掩码经 `value_barrier_64` 语义保持。
   - 限制/风险：bias 常量向量构建若每调用重建会引入额外开销（应循环外/每 bit 一次）；VLEN 无关性（8 limb 在 VLEN<128 时需 tail 处理——目标 VLEN=256 无需）；实测 sub 的 i==4 分支消除是否值得（8-limb 短循环，RVV 收益在中大循环更显著，本处为固定 8 元素，仍需 A/B）。
   - 预期 Profile signals：`15dc9e/15df2c/15df48/15df4c/15e0fe/15e10e` 等标量 sub 行消失，出现 `vle64.v/vsub.vv/vse64.v`；`15dbb4/15dbca/15dbee` 标量 cond_swap 行消失，出现 `vle64.v/vxor.vv/vand.vv/vse64.v`。
3. **Baseline facts 回填**：同 Finding 1（hardware `v` ✓；libcrypto.so.4 含 `v`；VLEN=256；compute-bound）。
4. **收益上界**：本函数局部样本份额 ≈ 18.3%（sub 11.3% + cond_swap 7.0%）；采样语义 gate 未全过 → 方向性表述，不称全局收益。
5. **三维路由判定**：current source=compiler-generated C（field.h）；implementation existence/reachability=无专用 RVV kernel/无 dispatch slot；function-level policy=同 Finding 1（无 curve448 asm policy，走 C/intrinsic 层修正）。
6. Implementation-shape proof：不适用。
7. **Related PRs**：`patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` §Related PRs——OpenBLAS [45fd2d9b0790](https://github.com/OpenMathLib/OpenBLAS/commit/45fd2d9b0790c5ca3698502d65d59d38d911ef4f)、MNN [#3913](https://github.com/alibaba/MNN/pull/3913)、[5376580ba19a](https://github.com/alibaba/MNN/commit/5376580ba19ac4034dc373f566fca846326fd612)、[#3779](https://github.com/alibaba/MNN/pull/3779)、[b35da1022747](https://github.com/alibaba/MNN/commit/b35da10227477e90d5e5be55e1e5e646d4af41e4)、[815ed5d6cb70](https://github.com/alibaba/MNN/commit/815ed5d6cb7052e2294d9eb6298e9e23b2fdf91d)、oneDNN [2b1dfe2d6233](https://github.com/uxlfoundation/oneDNN/commit/2b1dfe2d6233696bc2c803b48c06c00b29f7f866)、[#5265](https://github.com/uxlfoundation/oneDNN/pull/5265)、[1184757c2868](https://github.com/uxlfoundation/oneDNN/commit/1184757c286814e552c137ca354060a30a9872bb)、[#5079](https://github.com/uxlfoundation/oneDNN/pull/5079)、[de1342a9d1df](https://github.com/uxlfoundation/oneDNN/commit/de1342a9d1dfe8419dd5159a16013529e567a6bc)、[580b9c80484f](https://github.com/uxlfoundation/oneDNN/commit/580b9c80484f5df175ad36870280703ea5767cbe)、[a0961ab37e4c](https://github.com/uxlfoundation/oneDNN/commit/a0961ab37e4ccf7dec0a0fd05fab92c1fc812e38)、[1147a0739a1f](https://github.com/uxlfoundation/oneDNN/commit/1147a0739a1fa1ee881075ac1cb8dd8f05e26cb5)、[595fc3b9bf46](https://github.com/uxlfoundation/oneDNN/commit/595fc3b9bf46a5381d337af4ae0d7c529e2a9bcd)、OpenJDK [6700baa50520](https://github.com/openjdk/jdk/commit/6700baa5052046f53eb1b04ed3205bbd8e9e9070)、[885be2efa6b1](https://github.com/openjdk/jdk/commit/885be2efa6b1359a7c7ab36882e19a7eaba77fb3)、[9b61a7608eff](https://github.com/openjdk/jdk/commit/9b61a7608efff13fc3685488f3f54a810ec0ac22)、V8 [2b368def4848](https://github.com/v8/v8/commit/2b368def484809ad8d35b0c5d5f913bd95ad23ef)、[56dd6a2f1ee2](https://github.com/v8/v8/commit/56dd6a2f1ee28b2d37989a7888ad178a89f4f5ea)、MNN [#4042](https://github.com/alibaba/MNN/pull/4042)、[672c586](https://github.com/alibaba/MNN/commit/672c5862392393c171f1513bf7994d3b95e2a6a1)。Related PRs：22 条 URL。

## Phase 5 — Verification forecast / 验证预测：ossl_x448_int

按收益上界（局部样本份额）顺序逐项验证；supporting（gf_add_RAW 半宽）跟随 Finding 1 验证，不单独预测。

**Finding 1（primary，weak_reduce 向量公式化）**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`15ddc2: vle64.v v5,(s2)`（8.45%）、`15de88: vle64.v v5,(a5)`（8.45%）、`15df82: vle64.v v5,(a5)`（5.63%）、`15e048: vle64.v v5,(a7)`（5.63%）、`15e146: vle64.v v5,(t3)`（4.23%）、`15e1f8: vle64.v v5,(s7)`（4.23%）、`15e058: vand.vv v3,v3,v2`、`15ddf2: ld a5,-448(s0)`（7.04%）——weak_reduce 向量循环与标量 epilogue 样本应显著缩小；同区间 `vrgather.vv` 序列应消失。
- 应出现侧（锚定 pattern §Verification「候选 frontier 验证/指令验证/register spill 验证」）：重建（同 `-march` 含 v）后 `perf annotate` 该区间 `vsetvli`/`vsetivli` 显示候选 LMUL（m1/vl=4 或 m2/vl=8）；`vle64.v/vse64.v` 每调用 1–2 次覆盖 8 limb；无 `vlmul_ext/vlmul_trunc` 插入、无 vector spill/reload 回归（不同 VLEN 与编译器复核）；逐元素输出与 baseline 一致（整数弱归约无浮点归约顺序问题）。
- 验证脚本形态：`perf annotate --stdio -l -s ossl_x448_int` 对 `openssl speed x448`（同 workload）重采；正确性对照用 `openssl speed -evp X448` / KAT 向量。

**Finding 2（independent，标量 8-limb 循环）**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`15dcac: ld a7,0(a2)`（1.41%）、`15dc9e: addiw a4,a4,1`（1.41%）、`15df36: ld a3,0(a1)`（1.41%）、`15df48: sd a5,-8(a0)`、`15df4c: ld a5,8(a1)`、`15e0fe: ld a3,0(a1)`、`15e10e: add a5,a5,a4`、`15dbca: sd a4,-8(a1)`（4.23%）、`15dbb4: ld a4,0(a1)`、`15dbee: sd a4,-8(a1)`——sub/cond_swap 标量循环样本应消失。
- 应出现侧（锚定 pattern §Verification「指令验证/正确性对照/长度与 tail 验证」）：`vle64.v/vsub.vv/vse64.v`（sub，含 i==4 lane bias 常量精确还原）与 `vle64.v/vxor.vv/vand.vv/vse64.v`（cond_swap）出现；8-limb 一次覆盖；常数时间属性复查（无数据相关分支）。
- 可选补充（升级采样语义）：`perf record` 同窗口 + `perf annotate --percent-type=global-period` 获取函数级 workload 贡献与全局份额，以便把收益上界从局部份额升级为 workload 级表述。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（含 Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ 1/1 组；[ossl_x448_int] |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 两 L0 gate + bound gate + Sampling IP precision） | ✅ 7 行；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace、Classes scanned、顶层 finding 数、evidence 锚点、supporting/排除/推导） | ✅ 8 项 trace（rows-asm/operator-rvv/string-memory/vectorized-tuning/codegen/offload/crypto/runtime-os 各 1 项）；`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md, rows-crypto.md`；顶层 finding=2（primary 1 + independent 1）；each finding 带 (a) 逐字锚点（如 `15ddc2: vle64.v v5,(s2)` 8.45%、`15dcac: ld a7,0(a2)` 1.41%）；supporting=1（gf_add_RAW 半宽，`15e02e: bne s6,a5,15e018`）；排除条数=10（vector-state/operand-form/no-vectorization×2/unroll/register-pressure/fusion/layout/gather/IVSR）；推导式=2（route High×2 / impact Low×2，均点名 `baseline_gap: sampling metadata`） |
| 4 | Phase 4 输出要求满足（已读 pattern + 命中 row + 引用短语 + The fix 四要素 + Related PRs） | ✅ 已读 `rvv_register_group_utilization.md`（Row 5，引用「当前 LMUL 小于合法且无 spill 的候选边界」+「gather/转换的 index 与 data 宽度不匹配」）、`rvv_contiguous_elementwise_arithmetic_kernels.md`（Row 1，引用「标量循环每次只产生一个结果…RVV 可以在一次迭代中处理当前 vl 个元素」）；The fix 均含 before/after、correctness、风险、预期 Profile 锚点；Related PRs：13+22 条 URL |
| 5 | 路径合规（8 项 trace 扫描集、仲裁、leaf 来自 gate 内 row、模式 A 局部份额排序、th.v* 未停扫） | ✅ 模式 A（profile-backed）；primary/independent/supporting 仲裁合规；leaf 均来自通过 gate 的 row；`th.v*` 无 mismatch 未停扫；按局部样本份额排序 |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧对 Phase 3 引用行（Finding 1：`15ddc2/15de88/15df82/15e048/15e146/15e1f8/15ddf2`；Finding 2：`15dcac/15dc9e/15df36/15df48/15df4c/15e0fe/15e10e/15dbca/15dbb4/15dbee`）；出现侧标注 pattern §Verification |
| 7 | 契约边界合规（无实施询问/代码修改/补丁生成；交付止于证据、蓝图、The fix、验证预测） | ✅ 无实施询问、无直接编辑/补丁；仅诊断蓝图 |

修正记录：无