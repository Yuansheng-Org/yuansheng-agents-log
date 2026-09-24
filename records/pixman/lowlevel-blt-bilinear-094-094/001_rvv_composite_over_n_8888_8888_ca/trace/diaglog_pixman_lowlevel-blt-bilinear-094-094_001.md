Functions under analysis: [rvv_composite_over_n_8888_8888_ca]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-rvv_composite_over_n_8888_8888_ca-annotate.txt`，1436 samples，event=`cycles:u`，percent: local period，覆盖 hot loop body 55586–55630 与 setup/epilogue，覆盖完整）
- perf stat（bound/context）：已提供（`21-pixman-benchmark-riscv-lowlevel-blt-bilinear-094-094.txt`：duration 18.63s、cycles 40.95G、instructions 22.39G、IPC 0.546855）
- workload/binary/DSO/source context：已提供（libpixman-1.so.0.46.5，symbol `rvv_composite_over_n_8888_8888_ca`；源码 `pixman-rvv.c:2415-2455` + 宏 913-1043 + `RVV_FOREACH_2` 61-63；perf report 显示该函数占全程序 **78.49%** samples）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：部分缺失 — metadata 只含 bench 可执行文件的 attribute（`rv64i2p1_m2p0_..._zcd1p0`，无 `v`），不含热点 object libpixman-1.so 的 attribute（详见 Phase 1）
- hardware ISA：已提供（frozen hardware_profile_snapshot：SpacemiT X100，`rv64imafdcvh_zicbom_..._zve64d_..._zvbb_...`，含标准 `v` 与 zvk* 系）
- `vlenb`：已提供（VLEN=256 bits，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cycles:u`，percent: **local period**，perf_cpu=2，perf_freq=99，单次运行同一窗口；annotate 百分比为函数内局部份额）
- Sampling IP precision：缺失（frozen 快照无 `precise_ip` / Exact-IP 信息，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcvh_..._zve32x/zve32f/zve64x/zve64f/zve64d/zvfh/zvbb/zvkb/zvk*...`，暴露标准 RVV 1.0 `v` 与向量 crypto 扩展（frozen snapshot） |
| Build ISA | 部分：热点 object libpixman-1.so.0.46.5 的 disassembly 本身含 RVV 1.0 `v*` 指令（`vsetvli`/`vle32.v`/`vwmaccu.vx` 等），证明其按含 `v` 的 `-march` 构建；metadata 中 bench ELF 的 attribute（`rv64...zcd1p0`，无 `v`）属于 benchmark 可执行文件（harness），非热点 object → 不构成 build/hardware mismatch，但热点 object 的 `Tag_RISCV_arch` 未直接提供，记 `baseline_gap: build ISA（热点 object attribute 未直接提供）` |
| Vector flavor | annotate 全部为 RVV 1.0 `v*` mnemonic，无 `th.v*` → 无 flavor mismatch |
| VLEN | 已提供：vlenb=32 → VLEN=256 bits（frozen snapshot） |
| Bound type | IPC=0.546855（cycles 40.95G / instructions 22.39G，whole-test perf stat）→ 低 IPC、stall/latency-prone；无 cache/memory/branch counter → `baseline_gap: cache/memory counters`，无法完全分类 memory-bound；样本集中在 `vsetvli`（状态重配置）与一次 stack reload，而非 load/store 本体 → 第一杠杆不是 memory bandwidth |
| Sampling semantics | event=`cycles:u`（可解释为时间）；percent type=**local period**（annotate 内函数局部份额）；同一运行窗口（单次运行）；函数 workload 贡献已知（perf report 78.49%）→ 采样语义四条**未全部满足**（percent type≠global-period），记 `baseline_gap: sampling metadata`；收益上界只能表述为「当前 sampled event 下函数内局部样本份额」 |
| Sampling IP precision | 缺失：`baseline_gap: sampling IP precision`（无 `precise_ip`/Exact-IP 证据）→ 单条指令占比只锚定所属 basic block / loop interval（55586–55630），不承担 instruction-latency 级归因 |

L0 baseline gate：hardware 有 `v`、热点 object 实际发射 RVV 1.0 指令 → 无 build/hardware mismatch；无 `th.v*` → 无 flavor gate 冻结；annotate 完整 → 入口条件 A（profile-backed）。
Bound-type gate：无 cache/memory counter 无法判定 memory-bound；样本分布（vsetvli 主导、访存指令近零占比）指示**配置/重配置 stall-bound**，与 IPC 0.55 一致；本轮命中不因 memory-bound 下调 route confidence。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`rvv_composite_over_n_8888_8888_ca`（pixman-rvv.c:2415，OVER/solid→a8r8g8b8 mask→a8r8g8b8/x8r8g8b8 dest，`_ca` 变体；注册于 pixman-rvv.c:3038-3042 `PIXMAN_STD_FAST_PATH_CA`）。

hot basic block / loop interval：内层 chunk 循环 **55586–55630**（单一 straight-line basic block + 唯一 back-edge `55630: bnez a2,5558a`，区间内无 `jal`/`call`/inline-asm，无 vector-state clobber）。每 chunk 处理 32 pixels（VLEN=256 下 `vsetvli t0,s1,e8,m1` VLMAX=32 作为 min() 推导，见 55578/5562c）。

trace anchor（最高占比行）：
- `50.46 : 555da: vsetvli zero,a5,e8,m4,ta,ma`
- `25.74 : 55592: vsetvli zero,a5,e16,m8,ta,ma`
- `14.83 : 555ce: vl4r.v v20,(t2)`

Sampling IP precision：未确认 → 上述单行占比**只锚定 interval 55586–55630**（chunk 级机制结论），不将单条指令占比升级为该指令的 latency/cycle 成本；区间内 vsetvli 序列、spill 指令的存在性与频次是反汇编架构事实（非采样推导）。annotate 覆盖完整，无 `annotate_gap`。

## Phase 3 — Pattern scan / 模式扫描：rvv_composite_over_n_8888_8888_ca

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源是 compiler-generated RVV（C intrinsics 源码 pixman-rvv.c，反汇编含 `vsetvli zero,zero`、`vmv8r.v`、`vs4r.v`/`vl4r.v` 等 compiler 生成特征）；无 policy-backed missing-`.S` 情形（RVV 实现存在、注册、且在运行）。
2. `rows-operator-rvv.md` — **include**：算子语义合同（UN8x4 逐字节通道合成算术：widening MAC + rounding + clamp/saturation），需逐 row 评估是否接纳 already-vectorized 失效形态。
3. `rows-string-memory.md` — **exclude**：无 copy/fill/sentinel/compare/checksum/back-reference 语义；访问为 unit-stride 32-bit 像素流。
4. `rows-vectorized-tuning.md` — **include（必选）**：完整 annotate 已有 `v*` 且非手写 `.S`，修正对象候选为 RVV 配置/寄存器分组/policy/vector-state。
5. `rows-codegen.md` — **include**：独立信号 — hot interval 内出现 vector spill/reload（`vs4r.v` 55570 / `vl4r.v` 555ce）与重复 vlenb/VLMAX 派生（5555c/5559a csrr vlenb、5562c vsetvli-avl），指向 register-allocation / save-restore 类 codegen row。
6. `rows-offload.md` — **exclude**：无矩阵引擎 / packed-SIMD / 权重重排信号，纯 RVV 1.0 kernel。
7. `rows-crypto.md` — **exclude**：无密码学原语证据。
8. `rows-runtime-os.md` — **exclude**：用户态图形库合成路径，无 timer/ISR/特权 CSR 热点。

### Classes scanned: rows-vectorized-tuning.md, rows-codegen.md, rows-operator-rvv.md

### Local performance pattern scan

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Vector-State Management（independent，rank #1） | 内层 chunk 每 32 pixels 发射 **15 条 vsetvli**；完全相同状态被重复建立：`e16/m8,vl=128` 建立 4 次（55592/555b0/555f6/5560a），`e8/m4,vl=128` 建立 6 次（555a4/555c6/555da/555ee/55602/55616）；如 555f6 与 5560a 之间仅隔 1 条破坏状态指令 55602；区间为无 clobber 的 straight-line block；vsetvli 家族合计 ≈79.1% 函数样本（top-2：555da 50.46% + 55592 25.74% = 76.20%） | Medium | Medium | `patterns/rvv_vector_state_management.md` |
| RVV Register-Group Utilization and LMUL Sizing（independent，rank #2） | 峰值 live set 36 groups > 32（3× e16/m8 累加器 v16/v8/v0 + 3× m4 v24/v28/v20 同时存活，逐指令 live interval 核算见下）；出现 vector spill/reload：`vs4r.v v4,(a5)` 55570（每行）+ `vl4r.v v20,(t2)` 555ce（每 chunk，14.83%）= ≈15.1% 函数样本 | High | Medium | `patterns/rvv_register_group_utilization.md` |

### Finding 1 — RVV Vector-State Management（三件套）

**(a) 逐字 evidence 引用**（interval 55586–55630 内）：
- `25.74 : 55592: vsetvli zero,a5,e16,m8,ta,ma`（进入 16-bit widening 域，vl=a5=4·vl_pixels）
- `50.46 : 555da: vsetvli zero,a5,e8,m4,ta,ma`（回到 8-bit 域，vl=a5=128）
- 同一基本块内相同状态重建序列：`0.28 : 555b0: vsetvli zero,zero,e16,m8,ta,ma` → `0.07 : 555c6: vsetvli zero,zero,e8,m4,ta,ma` → … → `0.00 : 555f6: vsetvli zero,zero,e16,m8,ta,ma`（与 555b0 完全相同状态，仅隔 7 条指令）→ `0.07 : 55602: vsetvli zero,zero,e8,m4,ta,ma` → `0.62 : 5560a: vsetvli zero,zero,e16,m8,ta,ma`（再次重建，间隔 2 条指令）
- 每 chunk 的 next-vl 派生：`0.07 : 5562c: vsetvli a4,a2,e8,m1,ta,ma`（以 vsetvli-rs1 形式做 min(a2,VLMAX) 的 VL 派生，与 55578 同类派生每行重复）
- 该基本块（55586–55630）内**无任何 call/inline-asm/CSR 写** → vector-state 无外部 clobber，重建纯属重复建立兼容且可复用的状态。

**(b) 互斥邻居排除**：
- register-group row（LMUL/SEW 主因）：排除 — 判别性观察：被重复重建的 `e16/m8`、`e8/m4` 是 128-byte chunk 的**正确最小合法 LMUL**（128 个 e8 lane / 128 个 e16 lane 必须 m4/m8），改变 LMUL 不减少重建次数；证据是「同一状态建立多次」而非「LMUL 与 live set 不匹配」。
- inactive-lane row：排除 — interval 内全部 `ta,ma`，无 `vmset`/`vmclr.m`/masked op（0 条 mask 指令）。
- copy/fill 或 operator semantic row：排除 — 非复制/搬运 kernel；vtype 切换服务于 UN8x4 算术的 e8↔e16 域与 e32 内存视图，不是固定 stride deinterleave/packing 的数据访问症状（改写访存算法不消除 e8↔e16 切换，仅消除 e32 视图；该子项并入本 finding 的 fix 而非转 semantic row）。
- assembly row：排除 — compiler-generated（源码 pixman-rvv.c intrinsics）。
- operand-form row：排除 — 本 finding 的对象是状态重建本身，与 broadcast 形态无关。

**(c) 双 Confidence 推导式**：
- route：compiler-generated RVV provenance（源码+反汇编直接证据）+ 同一无 clobber 基本块内相同 vl/vtype 重复建立（反汇编状态数据流直接证据）+ 互斥排除完成 → 判据成立；但无 compiler pass/emitter-internal 证据（无 -fopt-info / backend dump），且 pattern §Verification 要求 emitter invalidation point 未证实时 route 低于 High → **Medium**。
- impact：hot interval sample share 充分（vsetvli 家族 ≈79.1%，top-2=76.20%）+ VLEN 已知（256）+ bound 分类（IPC 0.547 stall-prone）+ build/dispatch baseline 成立（kernel 在跑）；但采样语义四条未全满足（percent type=local-period）且存在竞争瓶颈（spill，见 Finding 2）→ **Medium**。

### Finding 2 — RVV Register-Group Utilization and LMUL Sizing（三件套）

**(a) 逐字 evidence 引用**（interval 内）：
- `0.28 : 55570: vs4r.v v4,(a5)`（vsrc 全宽 128B group 存栈，每行一次）
- `14.83 : 555ce: vl4r.v v20,(t2)`（每 chunk 从栈重载 128B，v20 随后 `0.00 : 555e2: vwmaccu.vv v0,v24,v20` 使用）
- 峰值 live-set 核算（指令级，555e2 时刻）：v16(m8, 555ac 产出、555ea 消费) + v24(m4, 5558e 载入) + v20(m4, 555ce 重载) + v0(m8, 正在写) + v8(m8, 555a0 产出、555f2 消费) = 8+4+4+8+8 = **32 groups**；编译器还把 `s_wide>>8`（`0.62 : 5560e: vsrl.vi v16,v0,8` 二次使用 v0）提升到两条 rounding 链共享，使 v0(m8) 活到 5561a → 5560e 时刻 v0+v8+v16+v28+v24 = 8+8+8+4+4 = 32 groups 整满 → v4(vsrc, 4 groups) 无寄存器可放，被迫 spill。

**(b) 互斥邻居排除**：
- rows-codegen register-pressure row：排除 — 该 row 行内互斥「LMUL/EMUL 主因 → register-group row」；判别性观察：spill 对象是 vector register-group（`vs4r.v`/`vl4r.v` of v4/v20），由 3 个并发的 e16/m8 widening 累加器（EMUL=8）直接造成，区间内无 scalar spill。
- operand-form row：排除 — vsrc 不能用 `vx` 形态替代：`vmv.v.x v4,a0`(55562) 是 e32 全宽 broadcast，reinterpret 为 4 个不同字节（RR/GG/BB/00）逐像素重复；`vwmaccu.vx` 在 SEW=8 下只 splat 标量低 8 位，语义不等价。128 常量（`vmv.v.x v8,a6` 55596）被 3 条 op 复用（vmv8r 555a0/555de、vwmaccu 555f2）→ 按 row 行内规则仅低置信候选，不列顶层。
- unroll row：排除 — back-edge `55630: bnez` 与指针/计数更新合计 ≈0.07% 样本，loop-control 不主导；无串行 recurrence 证据。
- no-vectorization / operator rows：排除 — main loop 完全向量化（`vle32.v`/`vse32.v`/`vwmaccu`/`vsaddu`）。
- maximal-LMUL row：排除 — 非 byte copy/compare 路径。
- layout/quantized rows：排除 — 无低比特 encoding/layout/quantization 语义（UN8x4 全字节通道算术）。

**(c) 双 Confidence 推导式**：
- route：spill 指令存在（直接反汇编事实）+ 逐指令 live interval 核算（36>32，架构上限）直接证明 LMUL/live-set 不匹配；行内互斥（LMUL/EMUL 主因）以判别性观察排除 codegen register-pressure 行 → **High**。
- impact：样本份额 ≈15.1%（vl4r.v 14.83% + vs4r.v 0.28%）；VLEN 已知；采样语义四条未全满足 + IP precision 缺失（单行占比只锚定 interval）→ **Medium**。

### 多候选仲裁小段

两 finding 均命中同一 hot loop 55586–55630，但按证据与机制**可分账**：Finding 1 锚定 vsetvli 地址段（55592/555da/…，状态重建机制），Finding 2 锚定 vs4r.v/vl4r.v 地址段（55570/555ce，spill 机制）。因果消除测试：批处理 vtype 阶段（Fix 1）不改变 live set → spill 仍在；缩短 live range/降低并发 m8 group（Fix 2）不减少 vtype 重建次数 → vsetvli churn 仍在。→ 两 finding 为 **independent**，均列顶层。Evidence-mechanism layer：两者同属 **L3**（vector/runtime configuration：`rvv_vector_state_management.md` 与 `rvv_register_group_utilization.md` 均为 L3 层点名 pattern）。入口条件 A：按 evidence sample share 加总排序 — Finding 1 ≈79.1%（top-2 76.20%）> Finding 2 ≈15.1%；top-3 指令合计 91.03%，其余 ≈9% 分布在 setup/epilogue（`1.04 : 5564c: ld ra,72(sp)` 等）。非顶层 supporting：无（operand-form 仅低置信候选，不列）。

## Phase 4 — Root-cause blueprint / 根因蓝图：rvv_composite_over_n_8888_8888_ca

### Finding 1 — RVV Vector-State Management（独立，rank #1）

**Root cause**：GCC 对 pixman-rvv.c 的 UN8x4 intrinsic 序列逐宏逐阶段发射 vsetvli，每个 `rvv_UN8_MUL_UN8_*` helper（pixman-rvv.c:921-948）内部需要 e8（wmaccu 源）→ e16/m8（累加器）→ e8（narrow）三次域切换，三次 helper 调用（pixman-rvv.c:2450-2451 链）使每 32-pixel chunk 重建 15 次 vtype，其中 **完全相同状态**（`e16/m8,vl=128`、`e8/m4,vl=128`）在无任何 clobber 的同一基本块内被重复建立 4 次与 6 次（如 555f6→55602→5560a 相隔 2 条指令即重建同一状态）。依据 `patterns/rvv_vector_state_management.md`：核心机制句 — 「重复设置相同状态或在每个局部分支里重复标记会增加循环控制指令、translator work 和生成代码膨胀」（§Why this is slow）；其 fix #3「只把重复 vector-state 维护提升到安全公共路径」与 fix #4「用支配与 VL 无关证明合并 compiler 发出的 VSETVLI」正是本 kernel 的结构性出路。SpacemiT X100（OoO）上每次 vtype 重配置使向量数据通路重排，区间 76.20% 的局部样本落在两次紧邻访存后的 vsetvli（55592/555da），与 IPC 0.547 的 stall-bound 一致。

**The fix / 修复方式**：重构 `rvv_composite_over_n_8888_8888_ca` 内层循环为**按 vtype 分相**结构（fix #3 的源码侧落地），把每 chunk 的 vsetvli 从 15 条降到 ≈6 条，并消除同块内相同状态重建：

```c
// Before（现状，pixman-rvv.c:2443-2453 的宏展开每 chunk 发射 15 条 vsetvli，
// 其中 e16/m8,vl=128 与 e8/m4,vl=128 各被重复建立 4 次 / 6 次）：
RVV_FOREACH_2 (width, vl, e32m4, mask, dst) {
    vuint32m4_t m = __riscv_vle32_v_u32m4 (mask, vl);
    __riscv_vse32 (dst, rvv_UN8x4_MUL_UN8x4_ADD_UN8x4_vvv_m4 (
        __riscv_vle32_v_u32m4 (dst, vl),
        __riscv_vnot (rvv_UN8x4_MUL_UN8_vx_m4 (m, srca, vl), vl),
        rvv_UN8x4_MUL_UN8x4_vv_m4 (m, vsrc, vl), vl), vl);
}

// After（蓝图：每相一个 vtype，同相内不切换；依赖链允许的乘法先批处理）：
//  相 A（e32 一次）：  m  = vle32(mask, vl);   d = vle32(dst, vl)
//  相 B（e8→e16 一次）：sm_wide = vwmaccu(m, srca, 128);  s_wide = vwmaccu(m, vsrc, 128)
//                       /* 两个乘法只依赖 m，可在同一 e8 相内连续发射，编译器只
//                          需要在相入口/出口各一次 vsetvli */
//  相 C（e16 一次）：  sm = (sm_wide + sm_wide>>8)>>8;  s = (s_wide + s_wide>>8)>>8
//  相 D（e8 一次）：   da = vnot(sm);  d_out_wide = vwmaccu(d, da, 128)
//  相 E（e16 一次）：  d_out = (d_out_wide + d_out_wide>>8)>>8
//  相 F（e8 一次）：   out = vsaddu(s, d_out);  vse32(dst, out, vl)
// → 每 chunk ≈6 条 vsetvli（相入口/出口），不再重建同块内已存在的相同状态；
//   vl 派生（原 5562c 的 vsetvli-rs1 min()）改为循环外一次计算 + 指针差 tail 判定。
```

- 适用前提：kernel 内无 call/clobber（已证），依赖链允许同相批处理（sm、s 仅依赖 m）。
- 不可破坏的 correctness contract：逐字节 UN8x4 语义、`(x + x>>8 + ONE_HALF)>>8`（G_SHIFT=8, ONE_HALF=128，pixman-rvv.c:926-933）除法近似、`vsaddu` 饱和、像素内字节序（A/R/G/B 端序）、chunk vl 精确（ta 尾随 lane 不被读取）。
- 限制/风险：m 的两次 wmaccu 若编译器仍各自重建 vtype，需要把乘法改写为同一 vtype 下的连续 intrinsic（或借助 kernel-scope 调优）；`vnot(sm)` 依赖 sm 完成 narrow，相 D 无法并入相 B — 这是本 fix 的天然下限（每 chunk ≥4 次 vtype 切换），不承诺消除全部切换。
- 预期 Profile signals：55592/555da 两行占比显著下降；hot interval 内 vsetvli 计数从 15 → ≈6；vsetvli 家族 ≈79% 份额向算术/访存行再分配。

**Baseline facts 回填**：hardware ISA=`rv64imafdcvh_..._zvbb_...`（标准 V）；build ISA=热点 object 含 RVV 1.0（disassembly 证明；object attribute 未直接提供）；VLEN=256；bound type=stall-prone（IPC 0.547，无 cache counter）。

**收益上界**：入口条件 A，但采样语义四条未全满足 → 「当前 sampled event（cycles:u）下的函数内局部样本份额」≈79.1%（vsetvli 家族；top-2 指令 76.20%）；不称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`）；参考上下文：perf report 显示函数占全程序 78.49% samples。

**三维路由判定**：current source = compiler-generated RVV（intrinsics C，pixman-rvv.c:2415-2455，直接源码证据）；implementation existence/reachability = 存在且可达（`PIXMAN_STD_FAST_PATH_CA` 注册 pixman-rvv.c:3038-3042，运行中 78.49%）；function-level policy = 无独立 `.S` policy 参与（非 missing-assembly；修正对象是既有 intrinsic kernel 的 vector-state 结构）。

**Related PRs：12 条 URL**（`patterns/rvv_vector_state_management.md` §Related PRs）：https://github.com/v8/v8/commit/c81ffb7a356dc408940d479fbbec1d048181be71 ；https://github.com/qemu/qemu/commit/d57dfe4b37ae542cec84a0cf751ecef313614cb6 ；https://github.com/qemu/qemu/commit/944b6dfd3d67236882f2bc09d1d30ed923268e16 ；https://github.com/qemu/qemu/commit/25669d275ce70346b94e3d5e4475d619eb979f5e ；https://github.com/qemu/qemu/commit/bd2c82283d21e3400d7d89676a221935904c2fe6 ；https://github.com/qemu/qemu/commit/81b9ef995a3b2fa5b08fab0615a1c9ed7cbe053e ；https://github.com/qemu/qemu/commit/949b6bcb27295eb04350afac32a45b698fc50104 ；https://github.com/qemu/qemu/commit/b8e1f32cda7805236c2bd497106a9356431c2d60 ；https://github.com/llvm/llvm-project/pull/148246 ；https://github.com/llvm/llvm-project/commit/d0554ae4cf264dd05024a753c66e15e4d16bf6e8 ；https://github.com/llvm/llvm-project/pull/118285 ；https://github.com/llvm/llvm-project/pull/123878 ；https://github.com/llvm/llvm-project/commit/f59307bfdc01c584bfa7cd31a55226831bf5590f

### Finding 2 — RVV Register-Group Utilization and LMUL Sizing（独立，rank #2）

**Root cause**：UN8x4 widening 结构（每个 255-division 需要 e16 累加器）在 e32m4 chunk（32 pixels=128 bytes）下强制 e16/m8 累加器（EMUL=8），编译器又把 `s_wide>>8` 提升为两条 rounding 链共享（`5560e: vsrl.vi v16,v0,8` 二次使用 v0），使 v0/v8/v16 三个 m8 group 在区间内持续并存；逐指令 live 核算峰值 32 groups 整满（555e2 与 5560e 两处），vsrc（v4, 4 groups）无处安放 → 编译器以 `vs4r.v`/`vl4r.v` 把它逐行存栈、逐 chunk 重载（128B 栈往返，14.83% 局部样本）。依据 `patterns/rvv_register_group_utilization.md`：核心机制句 —「选得过小会浪费寄存器带宽和循环开销；选得过大或跨宽度不匹配会造成 spill 或额外转换」（§Why this is slow）；以及「提高 LMUL 或增加 register-block accumulator 后，若 instruction-level live interval、EMUL 或 group 对齐超过分配预算，应缩短中间值 live range、把 load 移近 consumer」（§The fix #4）。`LMUL * peak_live_vectors <= 32`（§The fix #1）在此被违反（本 kernel 当前为 e16/m8 下的峰值 36 需求含被 spill 的 vsrc）。

**The fix / 修复方式**：在不改 32-pixel chunk 的前提下把峰值 live set 压回预算，消除 vsrc 的栈往返（fix #4 的 live-range 控制）：

```c
// Before：s_wide（v0, m8）被编译器提升为两条 rounding 链共享，从 555e2 活到 5561a，
// 与 v8/v16 两个 m8 并发 → 峰值 32 groups 整满 → vsrc 被 spill 到栈
// （vs4r.v 55570 / vl4r.v 555ce，每 chunk 128B 栈流量）。
//   sm = (wmaccu(m, srca, 128)  round-chain)  → 8-bit v16   [链 1]
//   s  = (wmaccu(m, vsrc, 128)  round-chain)  → 8-bit v0    [链 2]
//   d_out = (wmaccu(d, vnot(sm), 128)  round-chain)         [链 3，复用 v0>>8]

// After（蓝图：链 2 的 e16 中间值提前收敛，不再跨链共享）：
//   sm = (wmaccu(m, srca, 128)  round-chain)  → 8-bit       [链 1]
//   s  = (wmaccu(m, vsrc, 128)  round-chain)  → 8-bit        [链 2，narrow 立即完成]
//   d_out = (wmaccu(d, vnot(sm), 128)  round-chain)          [链 3，独立 e16 中间值]
// → 任一时刻最多 2 个 m8 group 并发（链 1+2 后链 3），峰值 live ≤ 20 groups < 32，
//   vsrc 常驻 v4 寄存器，vs4r.v/vl4r.v 从 hot interval 消失。
// 备选 shape（需 frontier 实测，不预设）：
//   (a) chunk 拆 16 pixels → e16/m4 累加器（3×m4+3×m4=24<32），迭代数 ×2；
//   (b) 保留 32-pixel chunk，禁止共享 v0>>8（多一次 vsrl 换回 8 groups 余量）。
// 候选须按 m1/m2/m4/m8 × unroll=1/2/4/8 frontier 枚举并 benchmark 选定
// （patterns/rvv_register_group_utilization.md §Verification）。
```

- 适用前提：live-range 收缩不改变数据流语义；chunk 尺寸不变时 iteration 开销不变。
- 不可破坏的 correctness contract：UN8x4 逐字节结果必须逐元素一致；rounding 链若因去共享而改变中间舍入，需与 pixman 对 fast-path 的容差约定核对（像素级输出一致）。
- 限制/风险：备选 (a) 增加 vsetvli/循环开销，与 Finding 1 的 fix 方向张力需 A/B；备选 (b) 以 1 条额外 vsrl 换 8 groups 余量，推荐首选。
- 预期 Profile signals：`vl4r.v 555ce`（14.83%）与 `vs4r.v 55570` 行消失或骤减；hot interval 内无 `vl*r.v`/`vs*r.v`；spill 栈流量（128B/chunk）归零。

**Baseline facts 回填**：同 Finding 1（hardware ISA=标准 V；build ISA=热点 object 含 RVV 1.0；VLEN=256；bound=stall-prone）。

**收益上界**：「当前 sampled event（cycles:u）下的函数内局部样本份额」≈15.1%（vl4r.v 14.83% + vs4r.v 0.28%）；不称 Amdahl 上界（`baseline_gap: sampling metadata`）。

**三维路由判定**：current source = compiler-generated RVV（同 Finding 1）；implementation existence/reachability = 存在且可达（同 Finding 1）；function-level policy = 无独立 `.S` policy；修正对象是 intrinsic kernel 的 LMUL/live-set 选型。

**Related PRs：13 条 URL**（`patterns/rvv_register_group_utilization.md` §Related PRs）：https://github.com/opencv/opencv/pull/26318 ；https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5 ；https://github.com/opencv/opencv/pull/25586 ；https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c ；https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0 ；https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05 ；https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e ；https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870 ；https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b ；https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3bca6046cb17 ；https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946932df7 ；https://github.com/v8/v8/commit/3844339936068c529170dcb4f2aa160654d25943 ；https://github.com/vllm-project/vllm/pull/47538

## Phase 5 — Verification forecast / 验证预测：rvv_composite_over_n_8888_8888_ca

**Finding 1（vector-state）**：
- 应消失/缩小侧：Phase 3(a) 引用行 `55592: vsetvli zero,a5,e16,m8,ta,ma`（25.74%）与 `555da: vsetvli zero,a5,e8,m4,ta,ma`（50.46%）占比显著下降；hot interval 内 vsetvli 计数 15 → ≈6；`555f6`/`5560a` 相邻同状态重建对不再出现。
- 应出现侧：`patterns/rvv_vector_state_management.md` §Verification —「count `vsetvl*` in the exact hot interval before and after」「Re-run annotate; repeated setup or bookkeeping should shrink without changing vector results, exception order, vstart, dirty-state visibility, or tail/mask behavior」→ 以相同 `-march`/`-O2` 重建 pixman，对同一 workload 重跑 `perf annotate`，验证 vsetvli 份额下移且 `rvv_composite_over_n_8888_8888_ca` 输出逐像素一致；补采说明：当前 `baseline_gap: sampling IP precision` — 若需指令级归因，用 `perf record -e cycles:u --precise-ip=2`（或核对目标 PMU Exact-IP 能力）重采后再确认 55592/555da 的单指令成本。

**Finding 2（register-group）**：
- 应消失/缩小侧：Phase 3(a) 引用行 `555ce: vl4r.v v20,(t2)`（14.83%）与 `55570: vs4r.v v4,(a5)` 消失或骤减；hot interval 内零 `vl*r.v`/`vs*r.v`。
- 应出现侧：`patterns/rvv_register_group_utilization.md` §Verification —「register spill 验证：在不同 VLEN、编译器和微架构上确认提高 LMUL 后没有出现 vector spill/reload 回归」「LMUL 候选比较：比较多个合法 LMUL 的生成指令、寄存器压力、spill、代码尺寸和 benchmark」→ 对 `m1/m2/m4/m8 × unroll=1/2/4/8` frontier 构建合法候选，逐候选记录 live interval、group 对齐、EMUL 与 spill，实测选定；输出与 baseline 逐元素一致（LMUL 改变不得改变数值结果）。

两条 independent finding 分别验证（先单假设归因，再做组合验证）：Fix 1 只应改变 vsetvli 分布，Fix 2 只应消除 spill 指令；组合后 top-3 指令（91.03% 局部份额）全部塌缩。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（含 Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ — 1/1 组；`rvv_composite_over_n_8888_8888_ca` |
| 2 | Phase 1 输出要求满足 | ✅ — 7 行 baseline 表 + L0 gate ×2 + bound gate；gap 标签：`baseline_gap: build ISA（热点 object attribute 未直接提供）`、`baseline_gap: cache/memory counters`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足 | ✅ — 8 项 `Class selection trace`；`Classes scanned:` rows-vectorized-tuning.md, rows-codegen.md, rows-operator-rvv.md；顶层 finding 2 个（vector-state、register-group）；evidence 锚点：`50.46 : 555da: vsetvli zero,a5,e8,m4,ta,ma`、`25.74 : 55592: vsetvli zero,a5,e16,m8,ta,ma`、`14.83 : 555ce: vl4r.v v20,(t2)`；supporting 0；排除条数：Finding1 5 条（register-group/inactive-lane/operator-semantic/assembly/operand-form）、Finding2 5 条（codegen-register-pressure/operand-form/unroll/no-vectorization/maximal-LMUL+layout-quantized）；推导式 2 条（route: Medium/High；impact: Medium/Medium） |
| 4 | Phase 4 输出要求满足 | ✅ — 已读 pattern 文件：`patterns/rvv_vector_state_management.md`（命中 row：RVV Vector-State Management；引用短语首词：「重复设置相同状态」「只把重复 vector-state 维护提升到安全公共路径」「用支配与 VL 无关证明合并」；The fix 含 before/after、correctness、风险、Profile signals）与 `patterns/rvv_register_group_utilization.md`（命中 row：RVV Register-Group Utilization and LMUL Sizing；引用短语首词：「选得过小会浪费」「LMUL * peak_live_vectors <= 32」「缩短中间值 live range」；The fix 含 before/after、correctness、风险、Profile signals）；非 missing-`.S` 分支（无 shape class 要求）；Related PRs：12 条 URL（vector-state）+ 13 条 URL（register-group） |
| 5 | 路径合规 | ✅ — 入口模式 A（profile-backed）；8 类 trace 扫描集完整；2 顶层 independent finding 按 evidence sample share 排序（≈79.1% > ≈15.1%）；每个 blueprint leaf 均来自已通过 gate 的 row（rows-vectorized-tuning.md 两行）；L3 层归属（vector/runtime configuration）；无 `th.v*`（RVV 1.0） |
| 6 | Phase 5 两侧锚定 | ✅ — 消失侧：`55592: vsetvli zero,a5,e16,m8,ta,ma`、`555da: vsetvli zero,a5,e8,m4,ta,ma`、`555ce: vl4r.v v20,(t2)`、`55570: vs4r.v v4,(a5)`；出现侧：`patterns/rvv_vector_state_management.md` §Verification、`patterns/rvv_register_group_utilization.md` §Verification |
| 7 | 契约边界合规 | ✅ — 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 |

修正记录：无