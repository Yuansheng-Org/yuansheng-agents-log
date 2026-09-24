Functions under analysis: [ossl_ec_GFp_simple_ladder_step]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`009-ossl_ec_GFp_simple_ladder_step-annotate.txt`，libcrypto.so.4，cpu-clock，18 samples，percent: local period；覆盖 0x179716–0x179a8a 全函数约 220 条指令）
- perf stat（可选 bound/context）：已提供（`1-openssl-benchmark-riscv-ecdhp256.txt`：IPC 2.606、branch_miss_rate 0.686%、L1_dcache_load_miss_rate 0.038%）
- workload/binary/DSO/source context：已提供（OpenSSL master commit 2924476b5591e691e904c4baf57894c526c4b8de；`ossl_ec_GFp_simple_ladder_step` 源码 = `crypto/ec/ecp_smpl.c:1560`，compiler-generated C；ECDH P-256 Montgomery ladder 步函数；**源码链路证据**：`ecp_mont.c:76`/`ecp_nist.c:78`/`ecp_smpl.c:77` 将其注册为通用方法 ladder_step；`ec_local.h:776-788` 显示仅当 `group->meth->ladder_step != NULL` 时调用它，否则 fallback 是 `EC_POINT_add/dbl`；`ec_curve.c:2575-2585` P-256 条目在 `ECP_NISTZ256_ASM`/S390X/`EC_NISTP_64_GCC_128` 均不成立时为 method=0 → `ec_group_new_from_data`（ec_curve.c:2917）回退 `EC_GROUP_new_curve_GFp` = EC_GFp_mont_method；`Configure:621-638` `ec_nistp_64_gcc_128` 为 default 禁用项）
- readelf -A（`Tag_RISCV_arch`）：已提供（metadata：`rv64i2p1_m2p0_..._zcd1p0`——无 `v`）
- hardware ISA：已提供（`rv64imafdcvh_...` 含 `v` = RVV 1.0 及 zvk*/zvbb/zvbc）
- `vlenb`：已提供（vlenb=32 → VLEN=256 bits）
- 采样元数据：已提供（event=`cpu-clock`；percent type=`local period`；同一窗口；函数 workload 级贡献未知 → `baseline_gap: sampling metadata`）
- Sampling IP precision：缺失 → `baseline_gap: sampling IP precision`
- 构建 configdata：**缺失**（未直接核对本 build 的 `OPENSSL_NO_EC_NISTP_64_GCC_128` 状态；由"profile 中出现 simple ladder"这一事实 + Configure default 推断 → `source_context_gap: build configdata`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...` 含 `v`（RVV 1.0），并含 Zvbb/Zvbc/Zvkn*/Zvks*/Zvkt |
| Build ISA | `Tag_RISCV_arch` 无 `v`（rv64i2p1_m2p0_..._zcd1p0） |
| Vector flavor | 全 scalar（无 `v*`/`th.v*`）；build 无 `v` |
| VLEN | vlenb=32 → VLEN=256 bits |
| Bound type | IPC=2.606、branch_miss 0.686%、L1D miss 0.038% → compute/latency-bound |
| Sampling semantics | event=`cpu-clock` ✓；percent=`local period` ✗；同一窗口 ✓；workload 贡献未知 ✗ → `baseline_gap: sampling metadata` |
| Sampling IP precision | `precise_ip` 未知 → `baseline_gap: sampling IP precision` |
| 构建 configdata | 缺失（`source_context_gap: build configdata`；可选命令：核对 configdata.pm 中 `ec_nistp_64_gcc_128` 与 `OPENSSL_NO_EC_NISTP_64_GCC_128` 定义） |

L0 baseline gate 1（hardware 有 `v`，build 无 `v`）：成立——置顶 baseline finding，不停止扫描。
L0 baseline gate 2（`th.v*`）：不适用。
Bound-type gate：compute/latency-bound（IPC 2.606）→ 本地实现选择/codegen fix 的 impact 不被 memory-bound 反驳。

## Phase 2 — Scope / 分析边界
函数清单：`[ossl_ec_GFp_simple_ladder_step]`，与承诺一致。

该函数是 XZ-coordinate Montgomery ladder 单步（EFD ladder-mladd-2002-it-4），以 `t0–t6` 七个 BN_CTX 临时 BIGNUM 为中间态，通过 `group->meth->field_mul/field_sqr` 函数指针（248/256(a5) 偏移）与 `BN_mod_add_quick/sub_quick/lshift1_quick`（直接 jal）串联约 23 次 field 运算调用。**函数自身无循环、无算术内核**——它是通用方法（EC_GFp_mont/nist/simple）的编排层，实际 limb 运算全部下放到 BN 库（bn_mul_mont、BN_usub、bn_sub_words 等，均为本批次其它 rank）。

hot interval：整个函数体。样本分布（18 samples）：
- prologue callee-saved 保存：`179724 sd s5` 5.56 + `17972e sd s11` 11.11 + `17978a sd s3` 5.56 + `1797d6 sd s9` 11.11 = **33.3%（6/18）**（共保存 13 槽：ra+s0..s11）
- epilogue 恢复：`17981a ld s0` 5.56 + `179828 ld s10` 5.56 = **11.1%（2/18）**
- 调用参数搬运（结构体 load + 寄存器 move）：`179730 mv s2,a0`、`1797cc ld a3,16(s5)`、`179800 mv a2,s10`、`179884 ld a5,0(s2)`、`17993e ld a3,96(s2)`、`179956 ld a3,64(s2)`、`179986 mv a2,s7` 各 5.56 = **38.9%（7/18）**
- 调用点/结果检查：`17999e jal BN_mod_sub_quick` 5.56 + `179a48 beqz` 11.11 = **16.7%（3/18）**

最高行 trace anchors（各 11.11% = 2/18）：`17972e: sd s11,24(sp)`（prologue 保存）、`1797d6: sd s9,40(sp)`（prologue 保存，与第 4 次 field_mul 参数装载交错）、`179a48: beqz a0,17980c`（调用链错误检查）。

Sampling IP precision 未确认 → 锚定区间（prologue/epilogue 组、marshalling 组、调用点组）。

**关键结构性事实**：该函数出现在 P-256 ecdh 热点中，本身即证明 group 走通用方法——`ec_local.h:776-788` 规定只有 `meth->ladder_step != NULL`（ecp_mont/nist/smpl 通用方法）才调用 simple ladder；专用方法 `EC_GFp_nistp256_method`（`ecp_nistp256.c:1826` ladder_step=0）与 `EC_GFp_nistz256_method`（`ecp_nistz256.c:1622` ladder_step=0）都走 `EC_POINT_add/dbl` fallback 或自带 points_mul，不会调本函数。

## Phase 3 — Pattern scan / 模式扫描：ossl_ec_GFp_simple_ladder_step

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude**：compiler-generated C（`ecp_smpl.c:1560`）；无 `.S` provenance；RISC-V 无 ladder `.S` 且无 policy 四证要求。
2. `rows-operator-rvv.md` — **include**：第一级 compiler-generated scalar；逐行核对（本函数无自含 main loop，field 运算下放库调用；预期零命中——见排除表）。
3. `rows-string-memory.md` — **exclude**：无 copy/fill/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — **exclude**：zero `v*`，build 无 `v`。
5. `rows-codegen.md` — **include**：第二级——prologue/epilogue 保存恢复组（44.4%）+ 调用参数搬运（38.9%）主导样本；且方法选择（generic vs 专用）属 kernel-selection 形态。
6. `rows-offload.md` — **exclude**：无矩阵引擎/packed-SIMD 上下文。
7. `rows-crypto.md` — **include**：第二级——workload 为 ECDH P-256（crypto benchmark），本函数是 ECC 点运算编排；逐行核对 crypto 四行（预期零命中）。
8. `rows-runtime-os.md` — **exclude**：用户态。

Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`、`rows-crypto.md`

### Local performance pattern scan: `ossl_ec_GFp_simple_ladder_step`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Kernel Selection and Runtime Specialization（primary） | P-256 ecdh 请求进入 generic EC_GFp_mont_method（profile 中 simple ladder 热 = `ecp_mont.c:76` 注册；`ec_local.h:776-788` 证明仅通用方法调用它）；仓库存在满足 P-256 合同的专用实现 `EC_GFp_nistp256_method`（`ecp_nistp256.c`，`ec_curve.c:2581` 指定，需 `enable-ec_nistp_64_gcc_128`——Configure:638 default 禁用）与 nistz256 asm（RISC-V 未提供） | High | Low（`baseline_gap: sampling metadata` + `baseline_gap: sampling IP precision` + `source_context_gap: build configdata`，无 A/B 实测） | `patterns/kernel_selection_and_runtime_specialization.md` |
| Register Pressure and Save/Restore Optimization（supporting，见 primary 行） | 13 槽 callee-saved 保存/恢复组占 44.4% 样本（`17972e sd s11` 11.11 + `1797d6 sd s9` 11.11 + `179724 sd s5` 5.56 + `17978a sd s3` 5.56 + epilogue `17981a`/`179828` 各 5.56）；7 个 BN 临时 + 4 个点指针跨 ~23 次调用 live；另有 38.9% 样本在参数装载/move | — | — | `patterns/register_pressure_and_save_restore.md` |

排除行：
| Row | 排除观察 |
|---|---|
| RVV Contiguous Elementwise / No vectorization（`rows-operator-rvv.md`） | 本函数无自含 main loop——无 elementwise 语义循环；热点是调用编排（save/restore + marshalling），不是 scalar compute loop；`no-vectorization` 行互斥点名 compiler/JIT codegen 形态归 codegen 行 |
| Dedicated Vector Crypto / Cipher-Mode / Carry-less / Polynomial Table（`rows-crypto.md` 四行） | 本函数非命名 crypto 原语（无 vaes*/vsha2*/vghsh 语义、非 GF(2^k) 乘法、无 lookup table）；ECC GF(p) 点运算不落入四行 |
| Kernel Operation Fusion（`rows-codegen.md`） | t0–t6 中间值"先写回再读入"且无外部观察者的观察部分成立，但其 canonical fix（fused ladder kernel）与 kernel-selection primary 的修复对象完全重合——启用/移植专用实现即提供 fused 实现；机制无法与 primary 分账 → 并入 primary，不另立 finding |
| Forced Inlining for Hot Specialization Helpers（`rows-codegen.md`） | 调用目标是跨 TU 全局函数（BN_mod_add_quick 等）与方法函数指针（field_mul jalr），非 translation-unit-local static helper |
| Control-Flow Layout / Resource-Aware Scheduling / 其余 `rows-codegen.md` 行 | `179a48 beqz` 是错误检查（每调用一次）；无 branch diamond/RAS/独立 PMU 证据；marshalling 的 move 序列是调用约定的直接产物 |

### 三件套：Kernel Selection and Runtime Specialization（primary）
(a) 逐字 evidence 引用：
- `11.11 :  17972e: sd s11,24(sp)`（prologue 保存，2/18）；`11.11 :  1797d6: sd s9,40(sp)`（prologue 保存，2/18）；`5.56 :  179724: sd s5,72(sp)`；`5.56 :  17978a: sd s3,88(sp)`；epilogue `5.56 :  17981a: ld s0,112(sp)`、`5.56 :  179828: ld s10,32(sp)` —— 保存/恢复组合计 44.4%
- 参数搬运：`5.56 :  1797cc: ld a3,16(s5)`（读 s->X）、`5.56 :  17993e: ld a3,96(s2)`（读 group->a）、`5.56 :  179956: ld a3,64(s2)`（读 group->field）、`5.56 :  179800: mv a2,s10`、`5.56 :  179986: mv a2,s7`、`5.56 :  179730: mv s2,a0`、`5.56 :  179884: ld a5,0(s2)` —— 合计 38.9%
- 调用点：`5.56 :  17999e: jal 116cb0 <BN_mod_sub_quick>`、`11.11 :  179a48: beqz a0,17980c`
- 结构性（非 annotate 行，源码级）：`ecp_mont.c:76` 注册 `ossl_ec_GFp_simple_ladder_step`；`ec_local.h:780-781` `if (group->meth->ladder_step != NULL) return group->meth->ladder_step(...)`；`ec_curve.c:2575-2585` P-256 条目 method 选择链；`Configure:638` `"ec_nistp_64_gcc_128" => "default"`
(b) 互斥邻居排除：
- no-vectorization（operator class）：排除——hot interval 非 scalar main loop（编排直列块）；field 算术在库函数内（各有自身 annotate）
- runtime-ISA-dispatch（codegen）：排除——非 hwprobe/feature-detection 缺口，是 build 配置选择（`ec_nistp_64_gcc_128` default 禁用）
- policy-backed missing `.S`（asm class）：排除——四证不齐（无 dispatch slot、无强制独立 `.S` 的函数合同；专用实现以 C 方法形式存在）
- fusion：排除为独立 finding——修复对象与 primary 重合（见上方排除表）
(c) 双 Confidence 推导式：
- route: profile 直接证明 generic ladder 被 P-256 workload 执行（该函数热 = 通用方法被选中）+ 源码直接证明专用实现存在且为 P-256 指定方法（ecp_nistp256.c + ec_curve.c 条目）+ 互斥排除成立；build configdata 未直接核对（推断性）→ **High**
- impact: 方向明确（启用/移植专用方法后本函数不再被调用、其 18 samples 与下游 BN 链收缩），但 `baseline_gap: sampling metadata`（local period、workload 贡献未知）+ `baseline_gap: sampling IP precision` + 无本 target 上 nistp256 vs mont 的 A/B benchmark + `source_context_gap: build configdata` → **Low**

### 三件套：Register Pressure and Save/Restore Optimization（supporting）
(a) 逐字引用（同上方 prologue/epilogue 行）：`17972e`/`1797d6`/`179724`/`17978a`/`17981a`/`179828` 六条 sd/ld 占 44.4%；13 槽保存区 = 128 字节帧。
(b) 互斥排除：hot-helper-inlining 排除（跨 TU 全局/函数指针调用，非 TU-local static helper）；LMUL/register-group 排除（零 `v*`）；assembly 排除（compiler-generated）。
(c) 双 Confidence：route — compiler-generated + 保存/恢复组主导（44.4% 区间份额）+ RA/live 证据（7 临时 + 4 点指针跨 23 调用）→ High；impact — 方向明确但采样缺口 → Low。

多命中仲裁：primary = Kernel Selection and Runtime Specialization（L1 层：vectorization/semantic dispatch 之下的实现选择）；supporting = Register Pressure and Save/Restore Optimization（L4 层，因果消除测试成立——方法切换后 simple ladder 不再执行，其保存/恢复与 marshalling 样本全部消失；supporting 自身 row gate 与直接反汇编证据均成立，允许作为 supporting）。无 independent/companion。

## Phase 4 — Root-cause blueprint / 根因蓝图：ossl_ec_GFp_simple_ladder_step
对应 row：`rows-codegen.md` 的 Kernel Selection and Runtime Specialization（Phase 3 primary）+ supporting `register_pressure_and_save_restore.md`。

1. **Root cause**：P-256 ECDH 在 RISC-V 上落入通用 `EC_GFp_mont_method` 路径，未选择仓库中存在的专用 `EC_GFp_nistp256_method`。依据 `patterns/kernel_selection_and_runtime_specialization.md` §Why this is slow 独有机制句："优化实现不可达：实现选择的优先级、注册条件或属性匹配决定了后续所有执行。高性能实现存在但未被选中时，其内部优化完全无法产生收益"。证据链：`Configure:638` 将 `ec_nistp_64_gcc_128` 置为 default 禁用 → `ec_curve.c:2575-2585` P-256 条目（无 ECP_NISTZ256_ASM、无 S390X、EC_NISTP_64_GCC_128 禁用）method=0 → `ec_group_new_from_data`（ec_curve.c:2917）回退 `EC_GROUP_new_curve_GFp` = EC_GFp_mont_method → `ecp_mont.c:76` ladder_step = 本函数 → 每 ladder bit 执行约 23 次经函数指针/BN 库的 field 运算（generic bn_mul_mont/bn_mod_* round-trip）。该函数的 18 samples 全部是这条 generic 路径的编排开销（prologue/epilogue 44.4% + marshalling 38.9% + 调用点 16.7%）。
2. **The fix / 修复方式**（与 pattern §The fix 一致）：
   - **Fix 1（构建配置，主修复）**：以 `enable-ec_nistp_64_gcc_128` 重建，使 P-256 使用 `EC_GFp_nistp256_method`（`ec_curve.c:2581` 分支）——其 field/point 运算在 4×64 felem 表示上直接计算（`ecp_nistp256.c`），ladder 走 `EC_POINT_add/dbl` fallback（`ec_local.h:783-784`）→ nistp256 专用 point ops，本函数不再被调用，generic BN 链（bn_sub_words/BN_usub/bn_wexpand 等）在 P-256 路径上消失。修复前/后伪代码：
     ```
     // Before（配置）：./Configure ...（ec_nistp_64_gcc_128 默认禁用）
     //   P-256 → ec_curve.c method=0 → EC_GROUP_new_curve_GFp → EC_GFp_mont_method
     //   ladder_step = ossl_ec_GFp_simple_ladder_step（每 bit ~23 次 BN 调用）
     // After（配置）：./Configure enable-ec_nistp_64_gcc_128 ...
     //   P-256 → ec_curve.c → EC_GFp_nistp256_method → ladder 经 EC_POINT_add/dbl fallback
     //   （nistp256 专用 point ops，无 BN round-trip）
     ```
   - **Fix 2（实现选择替代路径）**：移植 `ecp_nistz256.c` 式 fused ladder kernel 到 RISC-V（nistz256 是 OpenSSL 在 x86_64/ARM 上 P-256 的 canonical 实现，`ladder_step=0` 用 point_add/dbl 或自带 points_mul）；或为该 RISC-V target 提供专用 GFp ladder `.S`——均属 kernel-selection 的"实现存在性"增强方向，需先完成四证与基准验证。
   - **Correctness contract（不可破坏）**：ladder 的常量时间结构（ECDH 私钥标量处理，`ec_mult.c` ladder 循环 + CSWAP 逻辑）；EC 点运算语义与 P-256 曲线参数；`EC_GROUP_new_by_curve_name` 返回的 group 行为一致性；nistp256 方法依赖 `__uint128_t`（RV64 GCC 支持）与小端序。
   - **限制/风险**：Fix 1 是构建级变更（需重跑全部 EC 测试）；nistp256 的 points_mul 为变时间窗口 comb，若 ECDH 走 `ossl_ec_scalar_mul_ladder`（常量时间要求）则由 `EC_POINT_add/dbl` fallback 承担，仍需实测其性能；RVV 构建（build 无 `v`）不直接受益于本 finding，但方法切换不依赖 RVV。
   - **修复后预期 Profile signals**：`ossl_ec_GFp_simple_ladder_step` 的 18 samples 归零（不再被调用）；profile 中 `bn_sub_words`/`BN_usub`/`bn_wexpand`/`BN_get_flags` 的份额显著下降（P-256 路径不再走 generic BN field 算术）；ecdhp256 吞吐（519.78 ops/s）上升。
3. **Baseline facts 回填**：hardware ISA 含 `v`（RVV 1.0，VLEN=256）；build ISA 无 `v`（L0 finding 置顶）；bound type = compute/latency-bound（IPC 2.606）。
4. **收益上界**：本函数 18/18 samples（100%）均属 generic 路径编排开销（方法切换后函数整体消失）——局部样本份额表述为 1.0；supporting 的保存/恢复组占 44.4%。**`baseline_gap: sampling metadata`**（percent=`local period`、workload 级贡献未知）→ 只允许局部份额表述，禁止 workload 级 Amdahl 上界；未实测 nistp256 vs mont 在本 target 的差值。
5. **三维路由判定**：
   - current source：compiler-generated C（`crypto/ec/ecp_smpl.c:1560`）。
   - implementation existence/reachability：**专用实现存在但当前不可达**——`EC_GFp_nistp256_method`（C）存在且为 `ec_curve.c` 指定方法，但需 `enable-ec_nistp_64_gcc_128`；nistz256 asm 无 RISC-V 版本；`ec_nistp_64_gcc_128` 为 Configure default 禁用 → 构建未选择。
   - function-level policy：无要求独立 `.S` 的函数合同 → **不进入 policy-backed missing-`.S` 分支**（专用 C 方法已存在，四证不齐）。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs**：按 primary pattern `kernel_selection_and_runtime_specialization.md` 本地 `## Related PRs / 关联提交` 表——Related PRs：19 条 URL（oneDNN：cc92e4b29240、9233aa44f246、e19dc70ec1fb、276cb7cd00e8、b2f18637a7da、ec2bfa125433 + PR #4463、#4363、#5453、#4620、#4945、#5403；OpenJDK：PR #21083 → 580eb62dc097；OpenBLAS：ef8e7d0279df、bef47917bd72、03a83778bb9b、64401b441758）；supporting pattern `register_pressure_and_save_restore.md`：25 条 URL（V8/QEMU/Zephyr，见前序函数）。

## Phase 5 — Verification forecast / 验证预测：ossl_ec_GFp_simple_ladder_step
- 应消失/缩小（锚定 Phase 3(a) 引用行）：`17972e: sd s11,24(sp)`、`1797d6: sd s9,40(sp)`、`179724`/`17978a`、`17981a`/`179828` 保存/恢复组与 `1797cc`/`17993e`/`179956` 等 marshalling 行的样本归零（方法切换后整个 `ossl_ec_GFp_simple_ladder_step` symbol 从 P-256 annotate 中消失）。
- 应出现（依据 primary pattern §Verification）：真实 workload 进入预期实现——P-256 的 ladder 路径改用 `EC_GFp_nistp256_method` 的 point ops（或经 `EC_POINT_add/dbl` fallback），`perf annotate` 中 nistp256 field/point 函数（如 `ec_GFp_nistp256_point_add`/`field_mul`）出现样本；"确认热点不再逐 lane 调用标量库函数"；ecdhp256 吞吐上升；`openssl speed ecdhp256` + EC 测试套件（ectest/ec_internal_test/bntest）无回归。依据 supporting pattern §Verification：RA dump 确认 13 槽保存区随函数消失。
- 升级到定量结论所需补采数据：① 本 build 的 configdata.pm（确认 `OPENSSL_NO_EC_NISTP_64_GCC_128`）——消解 `source_context_gap: build configdata`；② `enable-ec_nistp_64_gcc_128` 变体构建后同 workload 的 A/B 吞吐与 annotate；③ `perf record --percent-type=global-period` + `precise_ip` 消解采样缺口。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅ | 1/1 组；[ossl_ec_GFp_simple_ladder_step] |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline + build-configdata 行；gap：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`、`source_context_gap: build configdata`（含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`、`rows-crypto.md`；顶层 finding 1（Kernel Selection and Runtime Specialization）+ supporting 1（Register Pressure）；evidence 锚点 `17972e: sd s11,24(sp)`（11.11%）、`1797d6: sd s9,40(sp)`（11.11%）、`179a48: beqz`（11.11%）、`1797cc`（5.56%）、源码锚点 `ecp_mont.c:76`、`ec_curve.c:2575-2585`、`Configure:638`；排除 7 条；推导式 2 条 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件 `kernel_selection_and_runtime_specialization.md`（primary）+ `register_pressure_and_save_restore.md`（supporting）；命中 row：rows-codegen.md 两行；引用短语首词：`优化实现不可达`、`Spill/reload 和过宽保存恢复`；The fix 含 before/after（配置级）、correctness（常量时间 ladder）、风险与 Profile signals 锚点；非 missing `.S`；Related PRs：primary 19 条 + supporting 25 条 URL |
| 5 | 路径合规 | ✅ | 入口模式 A；8 类 trace 完整；primary/supporting 仲裁合规（因果消除测试成立）；leaf 均来自通过 gate 的 row；`th.v*` 未触发停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`17972e/1797d6/179724/17978a/17981a/179828` + marshalling 行；出现侧：`kernel_selection_and_runtime_specialization.md` §Verification + `register_pressure_and_save_restore.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付止于 Profile 证据、根因蓝图、The fix 与验证预测 |

修正记录：无
