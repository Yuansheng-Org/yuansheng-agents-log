Functions under analysis: [cgemm_oncopy]（1 个）→ 本输出含 1 组 Phase 3–5

# RISC-V 性能诊断：OpenBLAS `cgemm_oncopy`（cblas_cgemm_512x512，SpacemiT X100 / VLEN=256 / RVV 1.0）

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`003-cgemm_oncopy-annotate.txt`，**5 samples**，event=cpu-clock，percent=local period，覆盖 0x22206–0x22436 全部地址）
- perf stat（可选 bound/context）：已提供（全局 `14-openblas-benchmark-riscv-cblas_cgemm_512x512.txt`：IPC 1.026、L1_dcache_load_miss_rate 1.806%、branch_miss_rate 0.255%）
- workload/binary/DSO/source context：已部分提供（annotate 内嵌源码行 `ctemp01 = *(aoffset1 +  0);` 等，证明为 `__riscv_v*` intrinsic 编写的 C 代码；无 DWARF 二进制）
- readelf -A（build ISA）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo / hwprobe）：已提供（metadata snapshot：`rv64imafdcvh_...`，含 `v`、`zve64d`）
- `vlenb`：已提供（vlenb=32 → VLEN=256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cpu-clock`、`local period`、本函数 5 samples；函数级贡献≈极小——本函数样本数仅为 cgemm_kernel_n（472 samples）的约 1%）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`（含 `v`、`zve64d`；无 `th.v*`） |
| Build ISA | `baseline_gap: build ISA`（metadata `binaries:{}`，无 ELF；annotate 的 `v*` mnemonics 表明实际以含 `v` 的 `-march` 构建） |
| Vector flavor | 标准 RVV 1.0 `v*`（`vle32.v`/`vse32.v`/`vsetivli`），与硬件一致，无 mismatch |
| VLEN | vlenb=32 → VLEN=256 bits；`vsetivli zero,2,e32,mf2,ta,ma` → e32,mf2 的 VLMAX=4，实际 VL=2 |
| Bound type | 全局 IPC≈1.03、L1 miss≈1.81%、branch miss≈0.26% → compute/issue-bound；本函数为打包型 memcpy 结构（窄 vector load/store + 地址算术），样本极少（5） |
| Sampling semantics | event=cpu-clock；percent type=local period；单窗口；函数级贡献≈1%（5/472）→ 只能表述为局部样本份额，禁止 Amdahl 上界；`baseline_gap: sampling metadata` |
| Sampling IP precision | `precise_ip`/Exact-IP 未知 → `baseline_gap: sampling IP precision`；单行占比只锚定 basic block / loop interval |

L0 baseline gate：hardware 含 `v`、annotate 含 `v*` → 无 hardware/build `v` mismatch；无 `th.v*` → flavor gate 不触发。Bound-type gate：compute/issue-bound，本地布局/指令形态修复是合理杠杆，但因本函数样本量极小，impact 强制 Low。

## Phase 2 — Scope / 分析边界
- 函数清单（与承诺声明一致）：`cgemm_oncopy`（1 个）。
- 语义角色：OpenBLAS CGEMM **A 面板打包器**（repack 层），`int CNAME(BLASLONG m, BLASLONG n, FLOAT *a, BLASLONG lda, FLOAT *b)`——把 A 的列面板按 8 行块打包进 packed buffer；其输出布局**直接决定** `cgemm_kernel_n`（本批次 rank 001，472 samples）在 k-loop 中必须以 stride=8B 的 `vlse32` gather 读取 A0r/A0i（见该函数蓝图 Finding 1）。
- hot loop 锚点：8 列打包 i-loop（0x2238e–0x2241e），最高占比行原文：
  - `40.00 :  223b4:  add     t3,a5,t2`（aoffset5 地址生成）
  - `20.00 :  223a4:  add     t3,a5,t0`（aoffset3 地址生成）
  - `20.00 :  223bc:  add     t3,a5,t4`（aoffset6 地址生成）
  - `20.00 :  223d0:  vle32.v v2,(s8)`（aoffset7 装载，即 ctemp13）
- 样本归属：4/5（80%）落在该 i-loop，其中 3/5 落在 `add` 地址生成指令、1/5 落在 `vle32.v` 装载；0x222ce 分支的 N&4 打包循环（vle32/vse32 + `addi` 偏移序列）0 样本。
- Sampling IP precision 不足 → 锚定 interval（i-loop 的「每行装载前的 base+index 地址加法 + 装载」组），不做单指令 latency 归因。

## Phase 3 — Pattern scan / 模式扫描：cgemm_oncopy

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — intrinsic C 代码（annotate 内嵌 `ctemp01 = *(aoffset1 + 0);` 源码行），非手写 `.S`；无 policy/existence 四证。
2. `rows-operator-rvv.md` — **include** — 打包语义（interleaved/panel packing）；检查 layout-packing row 与 no-vectorization。
3. `rows-string-memory.md` — **exclude** — 虽有拷贝形态，但语义是 GEMM 面板重排（numeric panel packing），归 layout/repack 类 row，不归 copy/fill 类。
4. `rows-vectorized-tuning.md` — **include** — 已向量化（vle32/vse32）；检查 operand-form、register-group、vector-state。
5. `rows-codegen.md` — **include** — 每内存 op 前的 `add`/`addi` 地址生成序列（addressing-mode fusion、induction variable strength reduction）。
6. `rows-offload.md` — **include** — 本函数即权重重排预处理层（weight-repack row）。
7. `rows-crypto.md` — **exclude** — 无密码学原语。
8. `rows-runtime-os.md` — **exclude** — 非 kernel/RTOS 上下文。

Classes scanned: `rows-operator-rvv.md`、`rows-vectorized-tuning.md`、`rows-codegen.md`、`rows-offload.md`（全文件逐行评估）；其余 4 类按 class 一级判据整组排除。

### Local performance pattern scan: `cgemm_oncopy`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Weight Repacking for Vectorized GEMM（primary） | 本函数即 repack 层；i-loop 打包 8 行×1 复数到 packed buffer，输出 **interleaved (r,i)** 布局；该布局是 `cgemm_kernel_n` 主 k-loop 两条 `vlse32` gather（23.09% 单行）的直接成因 | High | Low | `patterns/weight_repacking_for_vectorized_risc_v_gemm.md` |
| Load/Store Addressing-Mode Fusion（independent，次级） | i-loop 每个 vle32/vse32 前均有只服务该访存的 `add t3,a5,<off>` / `addi t3,a6,<off>` 序列（4/5 样本落在地址生成）；store 偏移 8/16/…/56 未折叠进 `vse32` 的 imm12 | Medium | Low | `patterns/load_store_addressing_mode_fusion.md` |

### Finding 1（primary）：repack 层的 interleaved 布局强制内核 gather（Weight Repacking）

**(a) 逐字 evidence 引用**（8 列打包 i-loop 0x2238e–0x2241e）：
```
20.00 :  223a4:  add     t3,a5,t0            ← aoffset3 地址生成
40.00 :  223b4:  add     t3,a5,t2            ← aoffset5 地址生成
20.00 :  223bc:  add     t3,a5,t4            ← aoffset6 地址生成
20.00 :  223d0:  vle32.v v2,(s8)             ← aoffset7 装载（ctemp13）
```
store 侧（同循环）：`vse32.v v8,(a6)` + 7 条 `addi t3,a6,{8..56}; vse32.v vX,(t3)`——写入 packed buffer 的是 8 行 × (r,i) 交错的 16 floats/块。该输出布局与 `cgemm_kernel_n` 的读取方式配对（内核 `vlse32 v2,(a1),a4`，stride=8B=2 floats，source 行 `__riscv_vlse32_v_f32m1( &A[ai+0*gvl*2], sizeof(FLOAT)*2, gvl )`），证实 oncopy 产出 interleaved (r,i) 面板、内核被迫跨步 gather。

**(b) 互斥邻居排除**：
- layout/channel-packing row（rows-operator）——行内判据要求 **scalar hot path** 的窄 load/store/register reorder 主导；本函数已向量化（vle32/vse32），且 row 未明确接纳 already-vectorized 失效形态；而 weight-repack row 明确接纳「已向量化 GEMM/GEMV 中……跨 block 重排主导」，故归 weight-repack；
- no-vectorization row——主循环已向量化，不适用；
- strided-layout row——本函数访存为连续 2-float 对（非固定 stride 搬运），且语义是 panel repack 而非 layout transform；
- cache-aware-blocking row——打包成本按 GEMM 调用摊薄（每次调用一次、被 j-loop 复用 N/8 次），非 tile-residency/refill 断崖问题。

**(c) 双 Confidence 推导式**：
- `route: intrinsic C 源码 + annotate 打包循环 + 与 cgemm_kernel_n 的 vlse32 stride-2 读取构成严格配对（def-use/布局合同证据）→ High`；
- `impact: 本函数仅 5 samples（函数级贡献≈1%）→ Low；真正收益在 cgemm_kernel_n 侧（其 A-gather ≈23.1% 局部份额），属跨函数联动，已计入该函数蓝图`。

### Finding 2（independent，次级）：地址生成未折叠进访存操作数（Load/Store Addressing-Mode Fusion）

**(a) 逐字 evidence 引用**（同 i-loop，store 段 0x223d4–0x2240c）：
```
0.00 :  223d4:  addi    t3,a6,8
0.00 :  223d8:  vse32.v v7,(t3)       ← 应为 vse32.v v7,8(a6)（imm12 可折叠）
0.00 :  223e0:  addi    t3,a6,16
0.00 :  223e4:  vse32.v v6,(t3)
...
```
load 侧同理：`add t3,a5,<off>; vle32.v vX,(t3)` ×8。i-loop 每轮 ≈35 条指令搬 64B，其中地址生成 15 条（8 add + 7 addi）≈43%；样本 4/5 落在地址生成组（3×`add` + 1×`vle32`）。

**(b) 互斥邻居排除**：
- loop-induction-variable row——a5 已用指针递增（`addi a5,a5,8`），问题不在 a5 的 index scaling，而在 8 个行基址的 base+index 重建与 store 偏移未折叠；归 addressing-fusion row；
- register-pressure row——hot loop 无 spill/reload，非 RA/live-set 问题；
- operand-form row——vle32/vse32 直接访存，无 vmv 物化；
- resource-aware-scheduling row——缺 X100 调度模型证据，且机制是「缺失指令形态」而非调度顺序。

**(c) 双 Confidence 推导式**：
- `route: annotate 直接显示只服务该访存的 add/addi 序列 + RISC-V vse32/vle32 的 imm12 寻址合法（offset 0–56 全部可折叠）→ Medium（compiler 行为，无 backend dump 佐证，故不 High）`；
- `impact: 本函数 5 samples、函数级贡献≈1%，且收益仅为 35→28 条/轮（约 -20% 指令）的局部改进 → Low`。

### 多命中仲裁小段（依据 triggers/arbitration.md）
- 两 finding 机制可分账：Finding 1 是布局/repack 合同问题（修复对象=打包输出布局，收益主要在 cgemm_kernel_n）；Finding 2 是访存指令形态问题（修复对象=本函数循环的地址生成）。修复对象、验证方法均不同 → **independent**，并列输出。
- 因果消除测试：仅折叠 store 偏移不会改变输出布局；仅改布局不消除地址生成——双向均不吞并。
- 归属按地址分账：4/5 样本在 i-loop；两个 finding 共享同一 interval，但样本行可区分（3/5 落在 `add` 地址生成=Finding 2 的形态证据；`vle32`/`vse32` 与布局=Finding 1 的载体）。
- 与 cgemm_kernel_n 蓝图的关系：本函数 Finding 1 即该函数蓝图 Finding 1（Weight Repacking）的**修复落点**（candidateFiles 指向 cgemm_oncopy）；本蓝图为该修复提供函数内视角与独立样本证据。
- Evidence-mechanism layer：Finding 1 归 L2（data movement / packing 合同）；Finding 2 归 L4（codegen micro-structure）。
- 排序（入口 A，按 local sample share）：Finding 1 与 Finding 2 共享同一小样本集（5 samples），不做份额排序；同层 independent findings unranked；两 finding 的 impact 均 Low。

## Phase 4 — Root-cause blueprint / 根因蓝图：cgemm_oncopy

### Finding 1（Weight Repacking；pattern：`patterns/weight_repacking_for_vectorized_risc_v_gemm.md`，对应 rows-offload.md 命中 row）

**1. Root cause**
本函数以 **interleaved (r,i) 复数对** 将 A 面板写入 packed buffer（i-loop store 段：8 行 × (r,i) 连续 16 floats）。依 pattern §Why this is slow 第 1 条「非交织布局强制 gather……向量单元空等数据」：该布局迫使消费方 `cgemm_kernel_n` 每个 k 步用两条 stride=8B 的 `vlse32` gather 取 A0r/A0i（该内核单条 23.09% 局部样本，472 samples，为全 workload 第一热点）。依 §Why 第 3 条「一次性重排摊薄到多次复用」：repack 在每次 GEMM 调用只执行一次、输出被内核 j-loop 复用 N/8 次，正是本 pattern 认领的「独立、可摊薄的预处理层」——其布局决策错误在 kernel 侧放大为每 k 步 gather 代价。

**2. The fix / 修复方式**
修复对象：本函数 8 列打包 i-loop（0x2238e–0x2241e）的输出布局，与 `cgemm_kernel_n` 的 A 装载形态**配对修改**（pattern §Verification「打包/使用一致性：重排后的布局与 GEMM/GEMV kernel 的读取顺序严格匹配」）：
- Before（当前）：每 i 步写 `[row0r,row0i,row1r,row1i,…,row7r,row7i]`（interleaved，16 floats/块）；内核以 `vlse32`（stride=8B）×2 读取。
- After（目标）：每 i 步按 **split-plane** 写 `[row0r,…,row7r]`（8 floats）随后 `[row0i,…,row7i]`（8 floats）；内核改为两条 unit-stride `vle32`：
```c
// oncopy store 侧（伪代码，每 i 步）
vse32.v v8, 0(a6);  vse32.v v7, 8(a6);  … vse32.v v2, 56(a6);   // 实部平面 8×4B
vse32.v v8, 64(a6); vse32.v v7, 72(a6); … vse32.v v2, 120(a6);  // 虚部平面 8×4B
// kernel 侧（配对修改，见 cgemm_kernel_n 蓝图）
vfloat32m1_t A0r = __riscv_vle32_v_f32m1(&Ar[ai], gvl);   // 连续 32B
vfloat32m1_t A0i = __riscv_vle32_v_f32m1(&Ai[ai], gvl);
```
- 适用前提：必须同步更新本批次 `cgemm_kernel_n` 内所有消费 oncopy 布局的段（main M×8 loop、M&4/M&2/N&4/N&2/M&1 尾块）与其它消费同一打包器的 kernel；`ai=m_top*K*2` 与 k 步指针步长按新布局重算。
- correctness contract：复数 (r,i) 配对语义不变；packed buffer 总大小不变（每块仍 128B）；打包器与所有 kernel 的读取顺序严格一致；M/N 余数路径逐条验证。
- 限制/风险：改动跨 oncopy+kernel 两文件；L1 行为变化（实/虚部平面分占 cache line）；本函数自身样本仅 5，函数内收益可忽略，收益主体在 cgemm_kernel_n（≈23.1% A-gather 份额）。
- 修复后预期 Profile signals：本函数 i-loop 布局改为两段 8×4B 连续 store；`cgemm_kernel_n` 的 `17c5e vlse32.v v1,(t2),a4`（23.09%）与配对 `17c5a vlse32.v v2,(a1),a4` 消失、变为 `vle32.v`。

**3. Baseline facts 回填**：hardware ISA=rv64imafdcvh_…（含 v，无 th.v*）；build ISA=`baseline_gap: build ISA`；VLEN=256 bits；bound type=compute/issue-bound（全局）。

**4. 收益上界**：本函数内局部样本份额——i-loop 占本函数样本 4/5（80%）；本函数占 workload 样本 ≈1%（5 vs cgemm_kernel_n 472）。布局修复的直接收益（A-gather 消除 ≈23.1% 局部份额）**归属 cgemm_kernel_n 蓝图**；本函数自身仅反映为 store 形态变化。因 local period + 无全局份额，不构成 Amdahl 上界。

**5. 三维路由判定**：current source=intrinsic C；existence/reachability=repack 层已存在且执行（本函数即活动打包器），缺失的是 split-plane 布局变体；function-level policy=OpenBLAS 允许 intrinsic 内核，无 policy 冲突。

**6. Implementation-shape proof**：不适用（非 missing `.S` 分支）。

**7. Related PRs**：Related PRs：5 条 URL（`https://github.com/ggml-org/llama.cpp/pull/19121`、`https://github.com/alibaba/MNN/pull/3813`、`https://github.com/alibaba/MNN/commit/6afcf99fedaec90b5b363ce4c78317800f291443`、`https://github.com/alibaba/MNN/pull/4426`、`https://github.com/alibaba/MNN/pull/4433`）。

### Finding 2（Load/Store Addressing-Mode Fusion；pattern：`patterns/load_store_addressing_mode_fusion.md`，对应 rows-codegen.md 命中 row）

**1. Root cause**
i-loop 的 8 条 `vse32` 前各有一条只服务该访存的 `addi t3,a6,{8..56}`，RISC-V `vse32` 的 base+imm12 寻址可合法表达 0–2047B 偏移，这些 `addi` 属「offset 未折叠进合法 memory operand」的地址生成冗余（依 pattern §The fix：把只服务该访存的 address-generation 序列折叠进 memory operand）。load 侧 8 条 `add t3,a5,<off>` 同理可经 8 个行指针递增（pointer induction）消除依赖 a5 的 base+index 重建。每轮 ≈35 条指令搬 64B，地址生成占 ≈43%，且 4/5 样本落在地址生成组。

**2. The fix / 修复方式**
修复对象：8 列打包 i-loop 的访存指令形态：
```c
// Before（store 侧）
addi t3,a6,8;  vse32.v v7,(t3);
addi t3,a6,16; vse32.v v6,(t3);
...
// After（偏移折叠进 imm12，消除 7 条 addi）
vse32.v v8, 0(a6);
vse32.v v7, 8(a6);
vse32.v v6,16(a6);
...
vse32.v v1,56(a6);
// load 侧（可选）：8 个行基址指针 t3X 各 +8 递增，vle32.v vX,0(t3X)，消除 8 条 add（等量换成 8 条指针 addi，但断开 a5 依赖链）
```
- 适用前提：store 偏移 0–56B 全部在 vse32 imm12 范围；load 侧指针递增需 8 个额外 GPR 的 live set 评估（本循环 GPR 预算充足，无 spill）。
- correctness contract：打包顺序与地址语义完全不变；只改寻址形态，不改变写入字节序列。
- 限制/风险：编译器若自行折叠则无收益（当前 annotate 证明未折叠）；load 侧指针归纳的寄存器压力需复核。
- 修复后预期 Profile signals：i-loop 中 `addi t3,a6,N` 序列消失；每轮指令 35→28（约 -20%）；该函数样本量过小，效果以指令计数/`perf stat` 指令数为准。

**3. Baseline facts 回填**：同 Finding 1（hardware 含 v；build ISA=`baseline_gap`；VLEN=256；bound=compute/issue-bound）。

**4. 收益上界**：本函数局部：i-loop 地址生成组 4/5 样本（其中 3/5 为 `add`、store 侧 `addi` 0 样本但指令计数占比 ≈20%）；函数级贡献≈1% → 收益上界 Low；「当前 sampled event 下局部份额」，无 Amdahl。

**5. 三维路由判定**：current source=intrinsic C（GCC 生成该指令形态）；existence/reachability=无替代实现问题；function-level policy=无政策冲突。

**6. Implementation-shape proof**：不适用。

**7. Related PRs**：pattern 本地 Related PRs 表为「无」（该文件未含关联提交表）→ 按规范填 `无关联提交`。注：`patterns/load_store_addressing_mode_fusion.md` 本地表无条目，故本小节如实记录。

### 排除项（逐 row，供复核）
- `rvv_register_group_utilization.md` / `rvv_register_budgeted_loop_unrolling.md`：hot loop 无 vector spill、无 vsetvl churn；VL=2/mf2 虽窄，但由「每行单复数」的打包粒度决定，非 LMUL 选型错误 → 排除；
- `rvv_operand_form_selection.md`：vle32/vse32 直接访存，无 vmv/vfmv 物化 → 排除；
- `rvv_vector_state_management.md`：`vsetivli` 在 j-loop 外一次建立，循环内无 vsetvl → 排除；
- `kernel_selection_and_runtime_specialization.md`：打包器已被选中执行 → 排除；
- `no-vectorization.md`：主循环已向量化 → 排除；
- `rvv_intrinsic_kernel_autovectorization_control.md`：无 autovec 扰动证据 → 排除。

## Phase 5 — Verification forecast / 验证预测：cgemm_oncopy

**Finding 1（布局 split-plane）**——修复对象：本函数打包输出布局 + 内核 A 装载（配对）。
- 应消失/缩小：`cgemm_kernel_n` 锚点行 `23.09 : 17c5e: vlse32.v v1,(t2),a4` 与 `0.00 : 17c5a: vlse32.v v2,(a1),a4` 被 `vle32.v` 取代、份额显著下降（这是本 finding 的主验证信号）；
- 应出现：本函数 i-loop store 变为两段连续 8×4B 平面写入；`cgemm_kernel_n` 出现 unit-stride `vle32.v`；
- 额外验证：打包器-内核布局严格匹配（pattern §Verification「打包/使用一致性」）；512×512 及 M/N 余数形状正确性测试全绿；其余消费 oncopy 布局的 kernel 无回归；
- 升级所需最小补采：global-period 采样（`perf annotate --percent-type=global-period`）与 `readelf -A`。

**Finding 2（寻址折叠）**——修复对象：本函数 i-loop 访存指令形态。
- 应消失/缩小：i-loop 的 `addi t3,a6,{8..56}` 与 `add t3,a5,<off>` 序列消失，`vse32.v vX,N(a6)`/`vle32.v vX,0(t3X)` 直接出现；
- 应出现：每轮指令 35→28；本函数整体指令数下降；
- 额外验证：打包输出字节级一致（与 reference 打包结果逐字节对比）；M/N 尾块路径不受影响；
- 升级所需最小补采：本函数样本量太小，需更长时间/多次运行的采样窗口或函数级 `perf stat` 指令计数。

**组合验证**：两修复分别验证后，重跑 cblas_cgemm_512x512 全链路：`cgemm_kernel_n` A-gather 消失 + 本函数指令数下降，GFLOPS（16.57）提升。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现（载荷：1/1 组；cgemm_oncopy） | ✅（1/1 组；`Functions under analysis: [cgemm_oncopy]`） |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 表 + L0 gate ×2 + bound gate；gap：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`） | ✅（含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md、rows-offload.md；顶层 finding=2（independent）；evidence 锚点=223a4/223b4/223bc/223d0/223d4；supporting=0；排除条数=9 条逐 row + 4 类整组；推导式=2 组） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern=weight_repacking_for_vectorized_risc_v_gemm.md、load_store_addressing_mode_fusion.md + 命中 row + 引用短语=「非交织布局强制 gather」/「offset 未折叠进合法 memory operand」；The fix 含 before/after、correctness、风险、Profile 信号；Related PRs：weight-repack 5 条 URL、addressing-fusion 无关联提交） | ✅ |
| 5 | 路径合规（载荷：模式 A（profile_backed，5 samples）；路径=intrinsic→offload/codegen；两 finding 机制可分账 → independent；样本 5 个、无排序；th.v* 不适用） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧=17c5e/17c5a（跨函数锚点）、223d4 addi/223b4 add；出现侧=weight_repacking §Verification 打包一致性、load_store_addressing_mode_fusion §The fix） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问、无代码修改、无补丁生成；交付止于证据+蓝图+The fix+验证预测） | ✅ |

修正记录：无
