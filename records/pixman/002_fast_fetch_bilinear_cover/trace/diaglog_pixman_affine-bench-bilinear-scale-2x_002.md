Functions under analysis: [fast_fetch_bilinear_cover]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 perf annotate：已提供（`002-fast_fetch_bilinear_cover-annotate.txt`；DSO `libpixman-1.so.0.46.5`；event=`cycles:u`；120 samples；`percent: local period`；覆盖完整 hot loop 3cd80–3cdfa）
- perf stat（bound/context）：已提供（`21-pixman-benchmark-riscv-affine-bench-bilinear-scale-2x.txt`；duration=3.36 s；cycles=7.395 G；instructions=20.387 G；IPC=2.756689）
- workload/binary/DSO/source context：已提供（pixman `affine-bench-bilinear-scale-2x`，commit 14735ced17e0053abbb925f9cf18c05ed9f52378；热点 DSO=libpixman-1.so.0.46.5；pixman fast-path 注册/源码细节不在 workspace 中 → `source_context_gap`）
- readelf -A（承载热点地址 object 的 `Tag_RISCV_arch`）：部分 — 冻结 metadata 快照提供 bench binaries 的 `Tag_RISCV_arch: rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`（无 `v`）；hot DSO 自身 attribute 未在快照内（详见 Phase 1）
- hardware ISA（冻结快照 cpuinfo）：已提供（SpacemiT X100，`rv64imafdcvh_...`，含标准 `v` 及 Zbb/Zba）
- `vlenb`：已提供（冻结快照：vlenb=32 → VLEN=256 bits）
- 采样元数据：已提供（event=`cycles:u`；percent type=`local period`；同一采样窗口；函数 workload 贡献未知）
- Sampling IP precision：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`（含 `v`，RVV 1.0；另含 zba/zbb/zvbb/zvk*）；来源：冻结 metadata 快照 cpuinfo。V-capable target 确认 |
| Build ISA | bench binaries `Tag_RISCV_arch: rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` — **无 `v`**；hot DSO 自身 attribute 未直接捕获 → `baseline_gap: build ISA（hot DSO attribute 未在快照；annotate 全 scalar 佐证 build 无 v）` |
| Vector flavor | annotate 内无 `v*` 也无 `th.v*`（hot loop 全 scalar）→ 无 `vector_flavor_mismatch`；存在 L0 finding：hardware 有 `v`、build 无 `v` |
| VLEN | vlenb=32 bytes → VLEN=256 bits（冻结快照） |
| Bound type | 仅 counting 级 IPC=2.756689；无 cache/memory/branch counters → `baseline_gap: bound type` |
| Sampling semantics | event=`cycles:u`；percent type=`local period`；同一采样窗口；函数级 workload 贡献未知 → 收益上界只能表述为「当前 sampled event 下函数内局部样本份额」 |
| Sampling IP precision | `precise_ip`/Exact-IP 未知 → `baseline_gap: sampling IP precision`；单指令高占比只能锚定 basic block / loop interval |

L0 baseline gate：**hardware 有 `v`，build 无 `v`** → 最高优先级 baseline finding（置顶保留，不停止其它 row 扫描）；无 `th.v*`。Bound-type gate：部分 gap → 命中 performance-impact confidence 依缺项降级。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`fast_fetch_bilinear_cover`（libpixman-1.so.0.46.5，地址 3ccc2–3ce1e）。

hot loop interval：`3cd80`–`3cdfa`（垂直插值主循环，40 条指令/轮，每轮 1 个输出像素，`sw` 4 字节 store；setup 段 3ccc2–3cd50 内含两次 `jal 3cc34 <fetch_horizontal.isra.0>` 调用，均为 0.00%）。最高占比行（trace anchor）：
- `14.96 :  3cdee:  or      a5,t3,a5`（字节打包，loop interval 内）
- `11.61 :  3cdd4:  and     t3,t3,a6`（通道字节提取）
- ` 8.27 :  3cdde:  add     a4,a4,t6`（插值链加法）
- ` 6.16 :  3cdf8:  lw      a5,24(s1)`（循环内不变式 limit 重载）

annotate 覆盖完整（含全部 hot loop body）。Sampling IP precision 未知 → 归因收敛到 interval-level mechanism；hot loop 占该函数局部样本 ≈100%（loop 行样本和 ≈100%，120 samples；setup/calls/epilogue 均 0.00%）。

## Phase 3 — Pattern scan / 模式扫描：fast_fetch_bilinear_cover

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源 compiler-generated scalar（GCC 风格 codegen），无 `.S`/DWARF provenance；policy-backed missing `.S` 四证无 pixman 源码证据，不成立
2. `rows-operator-rvv.md` — **include** — compiler-generated scalar loop 且反汇编直接显示 separable bilinear 的垂直插值合同（预计算权重、top/bottom 双行 load、插值链、字节打包）→ RVV Resampling row；zero-`v*` 触发 No-vectorization row（supporting 候选）
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel/checksum/string 语义；是像素重采样垂直 pass
4. `rows-vectorized-tuning.md` — **exclude** — annotate 无任何 `v*` 指令
5. `rows-codegen.md` — **include** — 连续固定 stride 循环每轮 index scaling（`slli`+`add`）与 limit reload（`lw 24(s1)`）→ Loop Induction Variable Strength Reduction row；Eliminate Redundant Sign/Zero Extensions row 评估后不命中（`sext.w` 为计数器符号比较语义、`zext.b` 为字节提取语义，均非冗余扩展）；Load/Store Addressing-Mode Fusion row 不命中（offset 已折叠，问题是不变式 load 未提升，无独立 row gate → symptom）
6. `rows-offload.md` — **exclude** — 无矩阵引擎 / packed-SIMD 证据
7. `rows-crypto.md` — **exclude** — 非密码原语热点
8. `rows-runtime-os.md` — **exclude** — 非 RTOS/kernel 侧热点

Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`

### Local performance pattern scan: `fast_fetch_bilinear_cover`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | hot main loop 为 separable bilinear 的垂直插值 pass：权重预计算 `srli a0,s4,0x8`+`andi a0,a0,254`；top/bottom 双行 load `ld t3,0(a4)`/`ld a5,0(a5)`；插值链 `sub`(3cdaa) → `mul`(3cdb4/3cdca) → `add`(3cdde)；字节提取/打包 `and`(11.61%)/`or`(14.96%)；hardware `v` + build 无 `v` | High | Medium | `patterns/rvv_resampling_kernels.md` |
| No vectorization（supporting） | 主循环全 scalar、interval 内 zero `v*`；hardware 暴露 `v`；build 无 `v`；被更具体 resampling row 认领 → 只作 supporting | — | — | `patterns/no-vectorization.md` |
| Loop Induction Variable Strength Reduction（supporting） | 连续固定 stride 循环每轮 index scaling（`slli t5,a3,0x2`/`slli t3,a3,0x3` + `add`）+ 计数器演进（`addi a3,a3,1`+`sext.w t5,a3`）+ limit reload（`lw a5,24(s1)`，6.16%） | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

### Primary finding 三件套（RVV Resampling Kernels）

**(a) 逐字 evidence 引用**（loop interval `3cd80`–`3cdfa`）：
- ` 5.79 :  3cd84:  ld      a5,8(a2)` — bottom 行缓冲基址（循环不变式，未提升）
- ` 4.14 :  3cd90:  ld      t3,0(a4)` / ` 2.48 :  3cd8e:  ld      a5,0(a5)` — top/bottom 行像素（64-bit 插值结果，每行来自 `fetch_horizontal` 的 `sd`）
- ` 5.94 :  3cdaa:  sub     a5,a5,t3` — 通道差（delta）
- ` 0.83 :  3cdb4:  mul     a5,a5,a0` / ` 2.48 :  3cdca:  mul     a4,a4,a0` — 垂直权重乘（a0=(y>>8)&0xfe，权重在 loop 外预计算：` 0.00 :  3cd62:  srli    a0,s4,0x8` + ` 0.00 :  3cd6e:  andi    a0,a0,254`）
- ` 8.27 :  3cdde:  add     a4,a4,t6` — top*256 + delta*w
- `11.61 :  3cdd4:  and     t3,t3,a6` / `14.96 :  3cdee:  or      a5,t3,a5` — 通道字节提取与打包（8→16→8 位合同）
- ` 0.00 :  3cdf4:  sw      a5,0(t4)` — 32-bit 像素 store

**(b) 互斥邻居排除**：
- 非 Spatial Convolution：tap 为运行期权重合同（a0 来自 y 定点小数）的双行插值，非固定 stencil tap
- 非 Indexed Gather：两行缓冲为顺序 unit-stride 访问（`ld 0(a4)`/`ld 0(a5)` + 步进 8），无数据相关索引
- 非 Layout/Color Conversion：字节提取/打包是插值算术（16-bit 展开合同）的收尾，无颜色矩阵/chroma 常量、无固定 stride 变换
- 非 No-vectorization primary：本行（resampling）按 rows-operator-rvv 行内判据更具体认领该 loop

**(c) 双 Confidence 推导式**：
- `route: compiler-generated provenance + 反汇编直接显示 vertical bilinear 插值合同（权重预计算、sub/mul/add 插值链、字节打包）+ hardware V + 三项语义互斥排除 → High`
- `impact: hot interval 局部样本份额 ≈100%（120 samples 全在 loop）+ VLEN=256 已知 + 函数为批次 rank 002，但 bound type gap（无 cache/mem counters）+ 采样语义四条件不全（local period、函数 workload 贡献未知）→ Medium`

### Supporting evidence

- **No vectorization**：(a) ` 4.14 :  3cd90:  ld      t3,0(a4)`（loop interval 内全部为 scalar `ld/slli/add/sub/mul/srli/and/or/sw`，zero `v*`）；supporting because: build ISA 无 `v` 使该 loop 无法发出任何 RVV 指令，解释"为何 scalar"，与 resampling 同一机制（同一 hot interval）
- **Loop Induction Variable Strength Reduction**：(a) ` 6.16 :  3cdf8:  lw      a5,24(s1)`（limit reload：循环不变式迭代界值每轮重载）+ ` 3.31 :  3cdc4:  addi    a3,a3,1` / ` 1.66 :  3cdc6:  sext.w  t5,a3`（计数器演进）+ ` 1.66 :  3cdbe:  slli    t5,a3,0x2`（每轮 index scaling；行缓冲侧另有 ` 0.00 :  3cd86:  slli    t3,a3,0x3` + 两次 `add`）；supporting because: 三处访问（top/bottom 行 64-bit 缓冲、输出 32-bit 行）均连续固定 stride，本可用指针递增 + 预计算 end-pointer（pattern §1/§2「把索引归纳变量替换成指针递增 + 预计算 end-pointer 比较」），每轮 `slli+add`、计数器 `sext.w` 与 limit 重载属循环级归纳变量冗余（相关行局部样本 ≈13.6%）；RVV 重写后整个标量循环消失

### 多候选仲裁

因果消除测试：上层修复（resampling 垂直 pass 向量化 kernel 重写该 hot interval）会自然消除下层信号（scalar 指令全部消失，含 index scaling 与 limit reload）→ **resampling = primary（L1）**；no-vectorization 与 LISR 均解释同一 hot interval 的机制侧面 → **supporting**（LISR 属 L4 循环控制层）。L0 baseline finding（build 无 v）置顶（Phase 1 已报告）。入口模式 A：primary 覆盖该函数 ≈100% 局部样本（supporting 不单独排序）。同层无 independent findings。循环不变式基址 load（`3cd80` 0.83% + `3cd84` 5.79% + `3cd94` 2.49% ≈ 9.1%）记录为 symptom：编译器因潜在 alias 保守每轮重载 top/bottom/dest 基址；无对应 row gate 成立（offset 已折叠，非 addressing-mode-fusion 场景），不作 finding，在 The fix 中一并消除。

## Phase 4 — Root-cause blueprint / 根因蓝图：fast_fetch_bilinear_cover

**纳入蓝图的 pattern 及对应通过 gate 的 row**：primary = `rows-operator-rvv.md`「RVV Resampling Kernels」row → `patterns/rvv_resampling_kernels.md`；supporting = 同文件「No vectorization」row → `patterns/no-vectorization.md`；`rows-codegen.md`「Loop Induction Variable Strength Reduction」row → `patterns/loop_induction_variable_strength_reduction.md`。

1. **Root cause**：该 hot interval 是 pixman bilinear 重采样管线的垂直插值 pass（fast path `_cover` 变体）：对 `fetch_horizontal.isra.0`（rank 001）预插值的 top/bottom 两行 64-bit 像素（4 通道 × 16-bit）做垂直定点插值并按 8→16→8 位合同打包输出。每像素 ≈40 条指令：两次 64-bit 行 load、通道 sub/mul/add 插值链、srli/and/zext.b/or 字节提取打包、sw 32-bit store，外加每轮 index scaling、计数器 sext.w 与不变式 limit/基址重载（≈22.7% 局部样本落在寻址/循环控制类指令）。依据 `patterns/rvv_resampling_kernels.md` §Why this is slow：重采样根因含「bilinear/cubic 权重与边界处理割裂成多遍」与逐输出重复计算——本 pass 与 horizontal pass 分裂为两遍标量循环；且因 build ISA 无 `v`（L0 finding），vector unit 无法处理该 loop 的并行元素（依据 `patterns/no-vectorization.md` §Why this is slow："vector unit 没有处理 hot main-loop 的并行元素"）。次要机制：依据 `patterns/loop_induction_variable_strength_reduction.md` §Why this is slow 第 1/2 条——索引归纳变量「每轮重复 scale + base+offset 计算」且「上界比较依赖乘法结果」，本 loop 每轮 3 处 `slli+add` 重建地址 + limit 重载，形成较长的循环控制依赖链。
2. **The fix / 修复方式**：
   - **修复对象**：pixman bilinear `_cover` fetch 路径的垂直插值 pass 与构建配置。先修 L0：以硬件与工具链共同支持的**精确** `-march`（含 `v`）重建 libpixman。
   - **before（现状标量循环，每迭代 ≈40 条指令 / 1 像素）**：
     ```c
     // topBuf/botBuf: fetch_horizontal 预插值行（每像素 4×16-bit 通道，64-bit）；w=(y>>8)&0xfe
     for (i = 0; i < n; i++) {
         vtop = *(u64*)(topBuf + i*8);          // ld 0(a4)
         vbot = *(u64*)(botBuf + i*8);          // ld 0(a5)
         d    = (vbot & M) - (vtop & M);        // M=0x000fffff000fffff；sub
         v    = ((vtop & M) << 8) + d * w;      // mul + add（上下两半各一组）
         px   = extract_bytes(v);               // srli/and/zext.b/or 打包 4 字节
         *(u32*)(dst + i*4) = px;               // sw
     }  // 循环控制：slli+add 寻址、sext.w、lw 24(s1) limit 重载
     ```
   - **after（RVV 形态，按 `patterns/rvv_resampling_kernels.md` §7 向量化插值 + §8 separable 流水约定）**：垂直 pass 为**顺序 unit-stride** 访问，无需 gather（与 horizontal 不同），直接用向量 load 双行：
     ```c
     // 每像素通道在 64-bit lane 内为 4×16-bit；或用 e16 视图按通道处理
     while (o < n) {
         vl = __riscv_vsetvl_e64m1(n - o);              // fixed-VL 主循环（kernel-conventions §3）
         vtop = vle64(topBuf + o*8);   vbot = vle64(botBuf + o*8);   // unit-stride 双行
         vd   = vsub(vand(vbot, M), vand(vtop, M));      // 保持 M 掩码与 sub 顺序
         v    = vadd(vsll(vand(vtop, M), 8), vmul(vd, w));            // top*256 + delta*w（w 广播）
         vpx  = pack_bytes(v);                            // vsrl/vand/vor（或 vnsrl+vnsrl+vzext）保持字节提取顺序
         vse32(dst + o*4, vpx);                           // 32-bit 像素连续 store
         o += vl;
     }  // tail: runtime-VL（kernel-conventions §3）；迭代界值外提、基址指针递增（LISR §1/§2）
     ```
     LMUL 预算（`kernel-conventions.md` §2）：vtop/vbot/delta/result e64m1×4 或合并为 m2×2；pack 阶段 e32/e8 级；按 `LMUL * peak_live_vectors <= 32` 与逐指令 live interval 校验选 m1/m2 + unroll 1/2/4，不默认 m8。vertical 与 horizontal pass 均为 unit-stride 后可考虑两遍间直接传递向量行缓冲（separable row-buffer 布局合同，pattern §8），减少中间 store-load 往返。
   - **适用前提**：build 含 `v`；`_cover` 变体保证坐标/权重在界内（无 border 分支）；pixman fast-path dispatch 注册 RVV fetch（workspace 无源码证据 → `source_context_gap`）；保留 scalar fallback。
   - **不可破坏的 correctness contract**：pixman 垂直定点合同 — 权重 `(y>>8)&0xfe`、掩码 `0x000fffff000fffff` 的低 20 位/32-bit 半、`top*256 + delta*w` 算术顺序、字节提取位置（`>>40`&0xff00、`>>16`&0xff、`&0xff00ff`、`>>24`&0xff000000）与 or 打包顺序、`_cover` 无 border 检查语义、输出 32-bit 像素连续布局。
   - **限制/风险**：字节打包的 RVV 表达（narrow + 移位 + mask）指令数可能仍偏高，需与 scalar 打包对比 A/B；SEW 选择（e64 lane 内 4×16-bit 通道 vs 按通道 e16 处理）影响 pack 形状；LISR 修复（指针递增 + end-pointer）若单独应用需证明 `end=base+n*stride` 不溢出且 n==0 零次执行（pattern §3）；两遍融合涉及行缓冲布局合同，不得与既有 arithmetic order 冲突（pattern §8）。
   - **预期 Profile signals**：`3cd90/3cd8e` 标量 `ld`、`3cdb4/3cdca` 标量 `mul`、`3cdd4/3cdee` 打包指令、`3cdf4` 标量 `sw` 样本消失或骤降；出现 `vsetvli`、`vle64`、`vse32`、`vsll/vsub/vmul/vand/vor` 向量指令；`3cdf8` limit 重载与 `3cdc4/3cdc6/3cdbe` 循环控制指令消失。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（RVV 1.0，含 zba/zbb/zvbb）；build ISA=bench binaries `rv64i2p1...zca1p0_zcd1p0` 无 `v`（hot DSO attribute gap）；VLEN=256 bits（vlenb=32）；bound type=`baseline_gap: bound type`（IPC 2.756689 仅 counting 级）。
4. **收益上界**：入口模式 A — primary finding 的 evidence 局部样本份额 ≈100%（120 samples 全在 loop）；表述为「当前 sampled event 下的函数内局部样本份额」。采样语义四条件不全 → 不表述为 workload 级 Amdahl 上界。supporting 不单独排序。
5. **三维路由判定**：
   - `current source`：compiler-generated scalar（libpixman-1.so.0.46.5 `.text`，GCC 风格 codegen；无 `.S`/DWARF 证据）
   - `implementation existence/reachability`：无 RVV kernel 证据（annotate 全 scalar）；pixman fast-path dispatch 集成细节不在 workspace → `source_context_gap`
   - `function-level policy`：pixman per-arch fast-path 惯例无本次源码证据 → policy-backed missing `.S` 四证不齐，**不进入 missing-`.S` 分支**；修复载体建议 intrinsic kernel + fast-path 注册
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`；四证不齐）。RVV intrinsic kernel 的 LMUL/tail 形状已在 The fix 中按 `kernel-conventions.md` §2/§3 给出候选（e64 m1/m2 + fixed-VL main loop + runtime-VL tail），属候选 leverage，待目标机 A/B 验证。
7. **Related PRs 小节**：
   - `patterns/rvv_resampling_kernels.md`：Related PRs：2 条 URL — https://github.com/alibaba/MNN/pull/4053 ；https://github.com/alibaba/MNN/commit/f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d
   - `patterns/no-vectorization.md`：Related PRs：10 条 URL — https://github.com/opencv/opencv/pull/22179 ；https://github.com/opencv/opencv/pull/22520 ；https://github.com/opencv/opencv/pull/23980 ；https://github.com/opencv/opencv/pull/24058 ；https://github.com/opencv/opencv/pull/24132 ；https://github.com/opencv/opencv/pull/24166 ；https://github.com/opencv/opencv/pull/24301 ；https://github.com/opencv/opencv/pull/24325 ；https://github.com/opencv/opencv/pull/27160 ；https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d
   - `patterns/loop_induction_variable_strength_reduction.md`：Related PRs：3 条 URL — https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d ；https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df ；https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec

## Phase 5 — Verification forecast / 验证预测：fast_fetch_bilinear_cover

**Primary — RVV Resampling Kernels（垂直 pass）**（supporting 跟随本预测验证）：
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`3cdee: or a5,t3,a5`（14.96%）、`3cdd4: and t3,t3,a6`（11.61%）、`3cdde: add a4,a4,t6`（8.27%）、`3cd84: ld a5,8(a2)`（5.79%）、`3cdf8: lw a5,24(s1)`（6.16%）样本消失或骤降。
- 应出现侧（锚定 `patterns/rvv_resampling_kernels.md` §Verification）：annotate 出现 unit-stride 向量 load/store（`vle64`/`vse32`）与 `vsetvli`、`vsll/vsub/vmul/vand/vor` 插值打包指令；「逐 lane coordinate/index arithmetic 与 scalar loads 下降」「cycles/output 改善」。正确性合同覆盖 half-pixel/align-corners 坐标定义、floor/rounding、权重与 integer fixed-point scale 与 reference 一致（§Verification 正确性合同）。
- 辅证（`patterns/no-vectorization.md` §Verification）：`readelf -A libpixman-1.so.0.46.5` 确认重建后 object 含 `v`；与 scalar reference 对比覆盖空、小 n、main-loop 整倍数与全部 tail。
- LISR 侧（若 scalar 路径保留）：`patterns/loop_induction_variable_strength_reduction.md` §Verification — 循环体内 `slli`/`add` index scaling 与 limit reload 减少，替换为指针递增与指针比较；`n==0` 零次执行、地址上界无溢出、逐元素等价比对无 off-by-one。
- 收益排序（入口 A）：按局部样本份额逐项验证 primary；supporting 不单独验证。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（载荷） |
|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅（1/1 组；`fast_fetch_bilinear_cover`） |
| 2 | Phase 1 输出要求满足 | ✅（7 行 baseline 表 + L0 gate 判定；gap 标签：`baseline_gap: build ISA（hot DSO attribute）`、`baseline_gap: bound type`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足 | ✅（8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding=1（primary resampling）；evidence 锚点：`14.96 : 3cdee: or a5,t3,a5`、`11.61 : 3cdd4: and t3,t3,a6`、`8.27 : 3cdde: add a4,a4,t6`、`5.79 : 3cd84: ld a5,8(a2)`、`0.00 : 3cd62: srli a0,s4,0x8`；supporting=2；排除=6（含互斥邻居 3 项判别）；推导式=1（route High / impact Medium）） |
| 4 | Phase 4 输出要求满足 | ✅（已读 pattern 文件：`patterns/rvv_resampling_kernels.md`（row: RVV Resampling Kernels；引用短语首词：bilinear/cubic 权重与边界处理割裂成多遍）、`patterns/no-vectorization.md`（row: No vectorization；引用：vector unit 没有处理）、`patterns/loop_induction_variable_strength_reduction.md`（row: Loop Induction Variable Strength Reduction；引用：每轮重复 scale + base+offset 计算）；`The fix` 含 before/after、correctness、风险与 Profile signals 锚点；非 missing-`.S` 分支；Related PRs：rvv_resampling=2 条、no-vectorization=10 条、loop_induction=3 条） |
| 5 | 路径合规 | ✅（模式 A profile-backed；8 项 trace 扫描集；多命中按因果消除测试仲裁为 primary+2 supporting；blueprint leaf 均来自通过 gate 的 row；L0 baseline finding 未停扫；入口 A 按局部样本份额排序） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧：`3cdee: or a5,t3,a5` / `3cdd4: and t3,t3,a6` / `3cdde: add a4,a4,t6` / `3cd84: ld a5,8(a2)` / `3cdf8: lw a5,24(s1)`；出现侧：`patterns/rvv_resampling_kernels.md` §Verification「indexed vector loads…cycles/output 改善」（垂直 pass 为 unit-stride `vle64/vse32`）） |
| 7 | 契约边界合规 | ✅（无实施询问、无代码修改、无补丁生成；无向用户追问；交付物止于证据、蓝图、The fix 与验证预测） |

修正记录：无