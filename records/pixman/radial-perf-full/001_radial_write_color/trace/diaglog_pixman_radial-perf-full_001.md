Functions under analysis: [radial_write_color]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`radial_write_color`，libpixman-1.so.0.46.5，204 samples，event=`cycles:u`，percent type=local period，含完整 hot loop body）
- perf stat（可选 bound/context）：已提供（`21-pixman-benchmark-riscv-radial-perf-full.txt`：duration 4,664,234,052 ns；task_clock 4,677,380,363；cpu_cycle 10,286,327,313；instruction 13,996,423,021；IPC 1.360682）
- workload/binary/DSO/source context：已提供（pixman master @ 14735ced17e0053abbb925f9cf18c05ed9f52378；热点 object `libpixman-1.so.0.46.5`；源码 `pixman/pixman/pixman-radial-gradient.c:70` 静态函数，compiler-generated C，无任何 `.S` provenance）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0` —— 不含 `v`）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（SpacemiT X100/K3 快照：`rv64imafdcvh_..._zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_..._zvbb_zvbc_...`，RVV 1.0，OoO 乱序执行）
- `vlenb`：已提供（32 bytes → VLEN 256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cycles:u`；percent type=local period；单次运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为函数内局部份额）
- Sampling IP precision：缺失（未提供 precise_ip/Exact-IP/skid 信息，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v` 存在：RVV 1.0；硬件快照含 `zve64d`（SEW=64 double vector FP 可用）、`zve32f`、`zvbb`、`zbb/zba/zbc/zbs`、`zfh` 等 |
| Build ISA | `baseline_finding（L0）`：`Tag_RISCV_arch` 为 `rv64i2p1_..._zcd1p0`，**不含 `v`** —— 二进制不可能发出任何 RVV 指令；hardware 有 `v` 而 build 无 `v` 为最高优先级 baseline finding |
| Vector flavor | annotate 全 scalar（`fmul.d`/`fsqrt.d`/`fdiv.d`/`fmadd.d`/`fcvt.l.d`），zero `v*`、zero `th.v*`；无 flavor mismatch（无 `th.v*`） |
| VLEN | 256 bits（vlenb=32，来自冻结硬件 profile 快照） |
| Bound type | IPC=1.360682（cycles 10.29G / instructions 14.00G）；热区间指令组合为标量 double FP（fsqrt/fmul/fdiv/fmadd）→ compute/latency-bound 分类；缺 cache/memory/branch counters，该分类无 counter 交叉确认 |
| Sampling semantics | event=`cycles:u`；percent type=local period（非 global）；同一运行窗口；radial_write_color 占整个 workload 的贡献未知 → 不满足 workload 级 Amdahl 上界四条，收益上界仅限函数内局部样本份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP 证据）→ 单指令高占比只锚定所属 basic block / loop interval，不承担单指令 latency 归因 |

L0 baseline gate 1（hardware 有 `v`、build 无 `v`）：命中并置顶 —— `vector_flavor` 相关 route 不冻结（build 缺 `v` 不是 flavor 错配，是构建目标缺能力），仅作为 baseline finding 报告并继续扫描。
L0 baseline gate 2（`th.v*`）：不适用（annotate 无 `th.v*`）。
Bound-type gate：compute/latency-bound（IPC 1.36 + FP 指令组合），无 memory-bound 证据 → 本地 compute 向量化 fix 的 performance-impact confidence 不因 memory 降级。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`radial_write_color`（1 个）。

hot loop 区间划分（函数 4357c–4366c，annotate 覆盖完整）：
- Interval A（a≠0 求解区间）`435ca–43632`：discr = b²−a·c（435ca fneg → 435ce fmul → 435d2 fmadd → 435d6 fadd +0.0）→ `435f0 fsqrt.d` → `435f4/435f8 fadd.d/fsub.d`（b±sqrtdiscr）→ `435fc/43600 fmul.d`（t0/t1 = (b±sqrtdiscr)·inva）→ 根选择与边界检查（43604 bnez a1,4364e；43606–43632 [0,1] 检查）。
- Interval B（repeat 模式根有效检查）`4364e–43666`：t0·dr ≥ mindr（43656 bnez 转写 t0）→ t1·dr ≥ mindr（43658 fmul.d → 43666 转写 t1）。
- Interval C（write 尾调用）`435bc–435c8` / `43634–4363c`：`fcvt.l.d a1,fa2,rtz`（double→int64 fixed-point 索引）→ 恢复 s0/ra → `jr a3` 间接尾调用 write_pixel。
- Interval D（memset 路径）`435e2–435ec`：无效根 → `memset(buffer,0,Bpp)`（0 samples）。

hot loop trace anchor（最高行）：`40.82 :   435fc:  fmul.d  fa2,fa0,fa3`（Interval A 内，t0=(b+sqrtdiscr)·inva，位于 fsqrt 依赖链下游）。

区间聚合样本份额（204 samples 全局加总）：
- Interval A+B（求解+根选择计算体）：0.97（435ca）+ 12.62（435f0）+ 40.82（435fc）+ 6.29（43600）+ 19.94（43656）+ 7.74（43658）+ 0.97（43666）= **89.35%**
- Interval C+D（调用结构 prologue/epilogue/tail-call）：2.90（43580）+ 2.42（43588）+ 3.87（435c2）+ 1.45（435c8）= **10.64%**

Sampling IP precision 不足 → 以上均按区间聚合解读；40.82% 的单行份额不单独承担 instruction-latency 根因，只作为区间 trace anchor。

provenance 判定（Step 0c 三维）：
1. 当前代码来源：compiler-generated C（`pixman-radial-gradient.c:70` 静态函数；树内无对应 `.S`；`pixman/pixman/*.S` 仅 MIPS/ARM/ARM64）。
2. 实现存在性/可达性：**无任何 RVV/汇编 radial gradient kernel 存在** —— `pixman-rvv.c`（`_pixman_implementation_create_rvv`，3096 行起）只注册 combine fast-path（OVER/SRC/…）与 combine_float/combine_float_ca 表，全树 grep 无 `radial`/`gradient` 相关 RVV kernel；gradient 走通用 iterator（`_pixman_radial_gradient_iter_init` → `radial_get_scanline_narrow/wide`），无 per-arch 覆盖层。
3. 函数级 policy：pixman 对 gradient 无 assembly-default policy（gradients 在所有架构均为 C-only 实现），不存在"要求独立 `.S`"的函数合同 → policy/existence 四证第 4 项失败。

## Phase 3 — Pattern scan / 模式扫描：radial_write_color

### Class selection trace（8 项）

1. `rows-asm.md` — **include** — 需评估 policy-backed missing `.S` 四证：pixman 存在 RVV 实现层与 dispatch 链（`pixman-riscv.c` + `_pixman_implementation_create_rvv`），但 radial gradient 无任何 vector kernel，须按行内判据判定四证是否齐全。
2. `rows-operator-rvv.md` — **include** — 当前代码来源为 compiler-generated scalar loop，hardware V-capable，主循环 zero `v*`（必选 class）。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel/compare/checksum/back-reference 语义。
4. `rows-vectorized-tuning.md` — **exclude** — annotate 零 `v*`，无已向量化代码可调（无 RVV 配置/寄存器/LMUL/policy 修正对象）。
5. `rows-codegen.md` — **include** — 分派可达性维度需判定 kernel-selection（现有 RVV 层是否应覆盖 radial）；且 prologue/epilogue/tail-call 区间有独立样本（register save/restore、call 开销候选）。
6. `rows-offload.md` — **exclude** — 无矩阵引擎/GEMM/packed-SIMD 语义（每像素二次方程求根不是 GEMM 卸载对象）。
7. `rows-crypto.md` — **exclude** — 无密码学原语证据。
8. `rows-runtime-os.md` — **exclude** — 用户态图形合成负载，无 timer/CSR/ISR 特权路径。

### Classes scanned: rows-asm.md, rows-operator-rvv.md, rows-codegen.md

### Local performance pattern scan: `radial_write_color`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| No vectorization（primary） | hot main loop（Interval A+B，89.35%）全为标量 double FP（`fsqrt.d`/`fmul.d`/`fmadd.d`/`fdiv.d`/`fcvt.l.d`），zero `v*`；hardware 暴露 RVV 1.0（zve64d，VLEN=256）；build ISA 无 `v`；无更具体 semantic/codegen row 认领 | High | Medium | `patterns/no-vectorization.md` |
| Policy-backed missing `.S`（排除） | 四证失败：dispatch slot 证弱（gradient 走 iterator，无 per-arch 覆盖槽）、policy 证**失败**（gradients 全 arch C-only，无 assembly-default 合同） | — | — | `patterns/policy_backed_missing_riscv_assembly_kernel.md` |
| Kernel Selection（排除） | 无"已有专用实现未选中"：`pixman-rvv.c` 全树无 radial/gradient kernel，不存在可修正的 selection/registration/gate | — | — | `patterns/kernel_selection_and_runtime_specialization.md` |
| Register Pressure / Save-Restore（supporting） | Interval C 的 `43580 addi sp,sp,-16`（2.90）、`43588 sd ra,8(sp)`（2.42）、`435c2 ld s0,0(sp)`（3.87）、`435c8 jr a3`（1.45）≈10.6% 逐像素调用结构开销；非 spill 主导，无 RA/live-set 主导证据 | — | — | `patterns/register_pressure_and_save_restore.md` |

### 顶层命中三件套 — No vectorization（primary）

**(a) 逐字 evidence 引用**（Interval A+B，solve 基本块区间）：

```
40.82 :   435fc:  fmul.d  fa2,fa0,fa3        # t0 = (b+sqrtdiscr) * inva —— 区间最高行
12.62 :   435f0:  fsqrt.d fa2,fa2            # sqrtdiscr
19.94 :   43656:  bnez    a5,435bc           # repeat 模式：t0*dr >= mindr → 尾调写像素
 7.74 :   43658:  fmul.d  fa4,fa1,fa4        # t1 * dr
 6.29 :   43600:  fmul.d  fa1,fa1,fa3        # t1 = (b - sqrtdiscr) * inva
```

整个 84 行 annotate 中零条 `v*` 指令（全为 scalar double FP + 整数搬迁/分支）。

**(b) 互斥邻居排除**：
- **elementwise row**（`rvv_contiguous_elementwise_arithmetic_kernels.md`）：本函数不是对连续索引数组做同型算术——每像素独立执行"二次方程求根（`fsqrt`）+ 双根选择分支（`bnez`/`beqz`）+ 区间判定 + 间接尾调用"，lane-independent 简单算术合同不成立。
- **resampling row**：无分数坐标插值/indexed load（t 由解析解直接求出，非 bilinear/cubic 采样合同）。
- **color-conversion row**：无颜色矩阵/chroma sampling。
- **precision-conversion row**：`fcvt.l.d`（double→int64）是 walker 的 `pixman_fixed_48_16_t` 合同必需转换，无往返、无冗余。
- **vectorized-tuning rows**：zero `v*`，无 LMUL/vsetvl/inactive-lane 修正对象。
- **kernel-selection row**：无"现有 RVV kernel 未选中"（树中根本不存在 radial RVV kernel）。
- **policy-backed missing `.S`**：policy 证失败（见上文 Step 0c 维度 3）。
- **floating-point semantic lowering row**：`435d6 fadd.d fa2,fa2,ft0`（fdot 的 0·0 项）有冗余潜力，但 0.00% samples，非主导，不命中（evidence-first）。

**(c) 双 Confidence 推导式**：
- `route: 源码 provenance 直接（pixman-radial-gradient.c:70 静态 C 函数）+ hardware V 直接暴露（zve64d）+ main loop 全 scalar zero v* + build 缺 v 解释成因 + 无更具体 semantic row 认领 → High`
- `impact: hot interval 局部样本份额 89.35% 充足 + VLEN=256 已知 + bound 分类 compute/latency（IPC 1.36，但缺 cache/memory/branch counters）+ 采样语义仅 local period、workload 贡献未知 → Medium（不得称 workload 级 Amdahl 上界）`

### 多命中仲裁小段

- 顶层 finding 唯一：**No vectorization**（primary）。evidence 与机制归属 L1（vectorization/semantic dispatch 层）。
- **Register Pressure / Save-Restore 为 supporting**：逐像素调用结构（prologue/epilogue + `jr a3` 间接尾调）的 10.6% 区间样本，其行内 gate（spill/reload 主导 + RA/live/clobber 证据）不成立（无 spill、无冗余 mv，仅 s0/ra 保存）；且按仲裁因果消除测试——向量化批量重写会消除逐像素调用结构，上层（L1）改写使下层 signal 消失 → supporting，不计顶层命中数，不单独验证。
- **L0 baseline finding**（build 无 `v`、hardware 有 `v`）先于一切 pattern 报告：zero `v*` 的直接原因是构建目标不含 V；它解释"为什么没向量化"，不停止其它 row 扫描。
- evidence 分账：Interval A+B 的 89.35% 归属 No vectorization；Interval C 的 10.64% 归属 supporting 的调用结构；无跨地址重叠。
- 收益上界（仅 primary）：`当前 sampled event（cycles:u）下 radial_write_color 函数内局部样本份额 89.35%`——采样语义四条不齐（local period、workload 贡献未知），**不**称 workload 级 Amdahl 上界。

## Phase 4 — Root-cause blueprint / 根因蓝图：radial_write_color

**对应 Phase 3 通过 gate 的 row**：`No vectorization`（rows-operator-rvv.md 末行；primary）。

1. **Root cause**：每像素以标量 double FP 求解径向渐变二次方程：`discr = b²−a·c → sqrt(discr) → t0/t1 = (b±sqrt)·inva → repeat 模式根有效性检查 → fcvt.l.d → 间接尾调用 write_pixel`。hot interval（435f0–43666）内每条 FP 指令只服务 1 个像素的 1 个 double，vector FP 单元完全不参与。引用 `patterns/no-vectorization.md` §Why this is slow 独有内容："vector unit 没有处理 hot main-loop 的并行元素"、`VLMAX = LMUL × VLEN / SEW`（VLEN=256、SEW=64 → 单条 m1 指令可并行 4 像素）；且该文件明确 "根因可能在 build target … 或缺失专用 kernel"。build `Tag_RISCV_arch` 无 `v`（L0）是 zero `v*` 的直接成因：即使硬件暴露 zve64d，`rv64gc` 构建的 object 也不能发出 RVV 指令。IP precision 未知 → 40.82% 仅锚定区间：聚类于 fsqrt（高延迟）→ fadd/fsub → fmul 依赖链下游，与 FP latency-chain-bound 假设一致（bound 分类 compute/latency，IPC 1.36）。

2. **The fix / 修复方式**（与 no-vectorization.md §The fix 第 1/2/3 步一致）：
   - **步骤 1（L0 前置，必须先行）**：用硬件与工具链共同支持的**精确** `-march`（含 `v`，如 `rv64gcv` 或含 `zve64d` 的最小 superset）clean rebuild libpixman。这是消除"硬件有 V、build 无 V"的直接动作；修复后编译器 autovec 与现有 `pixman-rvv.c` combine 层均可生效。
   - **步骤 2（主体，vectorize 求解区间）**：沿 scanline 方向把逐像素二次求解改写为跨像素向量 kernel（每迭代 4 像素 @ SEW=64/m1，VLEN=256 上界）：b、c 逐像素满足线性/二次递推（`pixman-radial-gradient.c:373-375`：`b += db; c += dc; dc += ddc`），可向量生成像素索引到 (a,b,c) 的向量；随后同型替换：
     ```
     # Before（scalar，每像素一轮）
     fneg.d   fa2,fa2;  fmul.d fa2,fa2,fa0;  fmadd.d fa2,fa1,fa1,fa2
     fsqrt.d  fa2,fa2;  fadd.d fa0,fa1,fa2;  fsub.d fa1,fa1,fa2
     fmul.d   fa2,fa0,fa3;  fmul.d fa1,fa1,fa3
     # After（期望形态示意，非固定实现处方；SEW=64，LMUL 按 live-vector budget 选 m1/m2）
     vfneg.v  v4,v4;   vfmul.vv v4,v4,v0;   vfmadd.vv v4,v1,v1,v4
     vfsqrt.v v4,v4;   vfadd.vv v0,v1,v4;   vfsub.vv v1,v1,v4
     vfmul.vv v4,v0,v3; vfmul.vv v1,v1,v3
     ```
     双根选择与 repeat 边界检查改为 lane 掩码（`vmfle`/掩码 `vmv` 选择有效根），写像素从逐像素间接尾调用改为批量写；无效根 lane 保持"写 0"语义。
   - **正确性 contract（不可破坏）**：源码 `pixman-radial-gradient.c:82-97` 注释明确的误差传播边界 —— discr 符号错误与 b²−a·c 病态、非稳定求根顺序（a>0 时 t0 最大优先）、repeat 模式 `t*dr >= mindr` 判定、`PIXMAN_REPEAT_NONE` 的 `0<=t<=pixman_fixed_1` 判定，全部必须按 IEEE double 逐 lane 保持；`fcvt.l.d` 的 rtz 舍入合同不变。
   - **限制/风险**：数据相关双根选择在向量形态下需要双路径求值 + 掩码（计算量上界翻倍风险）；masked lane 写 0（原 `memset` 路径）与 write_pixel 每 lane 调用契约需重构；SEW=64 下 VLEN=256 仅 4 lanes/迭代，收益受 lane 数限制；需验证 repeat=NONE 与 repeat≠NONE 两种路径。
   - **修复后预期 Profile signals**：Interval A+B 的 `fsqrt.d`/`fmul.d`/`fmadd.d` 份额骤降或消失；annotate 出现 `vsetvli`、`vfsqrt.v`、`vfadd.vv`、`vfmul.vv`、`vmfle` 等 `v*` 指令；Interval C 的逐像素调用样本（43580/43588/435c2/435c8 ≈10.6%）随批量写消失而下降。

3. **Baseline facts 回填**：hardware ISA = RVV 1.0（zve64d 等，SpacemiT X100 OoO）；build ISA = `rv64i2p1_..._zcd1p0`（**无 v**，L0 mismatch）；VLEN = 256 bits（vlenb=32）；bound type = compute/latency（IPC 1.3607，缺 cache/memory/branch counters）。

4. **收益上界**：`当前 sampled event（cycles:u）下，radial_write_color 函数内局部样本份额 89.35%（Interval A+B 加总）`。采样语义四条不齐（percent type=local period、函数 workload 贡献未知）→ 不称 workload 级 Amdahl 上界；VLEN=256/SEW=64 的 4-lane 上界与双路径求值会侵蚀实际收益，需 A/B 验证。

5. **三维路由判定**：
   - `current source`：compiler-generated C（`pixman-radial-gradient.c:70`，无 `.S` 关联）—— 证据：源码树 + annotate 全 scalar。
   - `implementation existence/reachability`：无替代实现（`pixman-rvv.c` 全树无 radial/gradient kernel）—— 证据：grep `radial|gradient` 于 `pixman-rvv.c` 零命中；`.S` 目录仅 MIPS/ARM/ARM64。
   - `function-level policy`：无 assembly-default policy（gradients 全 arch C-only）→ **不进 policy-backed missing `.S` 分支**。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**（`patterns/no-vectorization.md` §Related PRs）：
   - `Related PRs：19 条 URL`
   - https://github.com/opencv/opencv/pull/22179
   - https://github.com/opencv/opencv/pull/22520
   - https://github.com/opencv/opencv/pull/23980
   - https://github.com/opencv/opencv/pull/24058
   - https://github.com/opencv/opencv/pull/24132
   - https://github.com/opencv/opencv/pull/24166
   - https://github.com/opencv/opencv/pull/24301
   - https://github.com/opencv/opencv/pull/24325
   - https://github.com/opencv/opencv/pull/27160
   - https://github.com/opencv/opencv/pull/27119
   - https://github.com/opencv/opencv/pull/27097
   - https://github.com/opencv/opencv/pull/27007
   - https://github.com/opencv/opencv/pull/26958
   - https://github.com/opencv/opencv/pull/26865
   - https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d
   - https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a
   - https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706
   - https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f
   - https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a

## Phase 5 — Verification forecast / 验证预测：radial_write_color

primary finding（No vectorization）修复对象 = Interval A+B 求解区间 + Interval C 调用结构：

- **应消失/缩小**（锚定 Phase 3(a) 引用行）：
  - `435fc: fmul.d fa2,fa0,fa3`（40.82%）与 `435f0: fsqrt.d fa2,fa2`（12.62%）、`43600: fmul.d fa1,fa1,fa3`（6.29%）、`43658: fmul.d fa4,fa1,fa4`（7.74%）、`43656: bnez a5,435bc`（19.94%）所在标量区间不再主导 annotate；
  - `43580/43588/435c2/435c8` 的逐像素 prologue/epilogue/尾调样本随批量写像素而下降。
- **应出现**（对齐 `patterns/no-vectorization.md` §Verification）：同一函数（或替代的 scanline kernel）annotate 出现 RVV 指令 `vsetvli`、`vle64`/`vse64`、`vfadd.vv`、`vfmul.vv`、`vfsqrt.v`、`vfmacc.vv`、`vmfle` 等；scalar 指令不再主导 hot loop；`readelf -A` 确认承载 object 的 `Tag_RISCV_arch` 含 `v`。
- 对比验证：同一 radial-perf-full 输入下重跑 benchmark，对比 cycles/instruction 与每像素开销；覆盖 repeat=NONE 与 repeat≠NONE（PAD/NORMAL/REFLECT）两种路径、masked 全无效根区间（memset 语义）与正常根区间；FP 语义（IEEE double 逐 lane、rtz 舍入）不因向量化改变。
- supporting（register save-restore 调用结构）不单独验证，跟随 primary 预测：批量重写后逐像素调用样本消失即为该 supporting 的消失侧信号。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅ | `1/1 组；radial_write_color` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表 + 2 个 L0 gate + bound gate；gap 标签：`baseline_gap: sampling IP precision`（含 `Sampling IP precision` 行）；无 `th.v*` |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-asm.md, rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（No vectorization）+ supporting 1（Register Pressure，不计命中）；evidence 锚点：`40.82 : 435fc: fmul.d fa2,fa0,fa3`、`12.62 : 435f0: fsqrt.d fa2,fa2`、`19.94 : 43656: bnez a5,435bc`；排除条数 8（elementwise/resampling/color-conversion/precision-conversion/vectorized-tuning/kernel-selection/policy-backed .S/FP-semantic-lowering）；推导式 confidence：route High / impact Medium |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`patterns/no-vectorization.md`；命中 row：No vectorization；引用短语：`vector unit 没有处理 hot main-loop 的并行元素`、`VLMAX = LMUL × VLEN / SEW`、`根因可能在 build target … 或缺失专用 kernel`；The fix 含 before/after（标量 435f0-43600 序列 vs vfneg/vfmul/vfmadd/vfsqrt/vfadd/vfsub/vfmul 形态）、correctness contract（误差传播/根顺序/repeat 判定/rtz）、风险（双路径求值、masked lane 写 0、4-lane 上界）、预期 Profile signals；missing `.S` 分支未进入（四证失败）；`Related PRs：19 条 URL` |
| 5 | 路径合规 | ✅ | 模式 A（profile_backed）+ 路径：compiler-generated scalar → rows-operator-rvv.md（必选）→ rows-asm.md/rows-codegen.md（include 评估）→ 单 primary 命中；L0（build 无 v）先于 pattern 报告但未停扫；区间分账（Interval A+B 89.35% / Interval C 10.64%）；仅 A 模式按动态份额排序（primary 89.35%，supporting 不排序） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`435fc: fmul.d fa2,fa0,fa3`（40.82）、`435f0: fsqrt.d fa2,fa2`（12.62）、`43600`（6.29）、`43658`（7.74）、`43656`（19.94）、`43580/43588/435c2/435c8`；出现侧：`patterns/no-vectorization.md` §Verification（vsetvli/vle64/vse64/vfadd/vfmul/vfsqrt/vfmacc/vmfle、readelf -A 含 v） |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix、验证预测；无向用户追问（无 object-clarification 需要） |

修正记录：无