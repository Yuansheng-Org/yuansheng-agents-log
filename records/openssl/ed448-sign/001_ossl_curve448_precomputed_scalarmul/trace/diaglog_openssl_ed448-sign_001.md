Functions under analysis: [ossl_curve448_precomputed_scalarmul]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 `perf annotate`：已提供（`001-ossl_curve448_precomputed_scalarmul-annotate.txt`；DSO=libcrypto.so.4；event=cpu-clock，157 samples，percent: local period；prologue 到 epilogue 及全部循环体覆盖完整）
- `perf stat`：已提供（`1-openssl-benchmark-riscv-ed448-sign.txt`，整 run counting，5000 iterations）
- workload/binary/DSO/source context：已提供（openssl ed448-sign benchmark；metadata commit 2924476b5591e691e904c4baf57894c526c4b8de；annotate 内嵌 constant_time.h 源码行，provenance 可判定）
- `readelf -A`（热点 object）：部分提供（openssl_bench ELF attribute：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`，无 `v`；libcrypto.so.4 自身 attribute 缺失——但**同 DSO 内 function 002–005 的 annotate 直接包含 RVV 1.0 指令**（`vsetivli/vle64.v/vadd.vv/vrgather.vv/vse64.v` 等），证明 libcrypto.so.4 已按含 `v` 构建；attribute 是 object 级而非函数级）
- hardware ISA：已提供（metadata cpuinfo：`rv64imafdcvh_...`，含 `v`；SpacemiT X100 / K3）
- `vlenb`：已提供（RVV 1.0，vlen_bits=256，vlenb=32）
- 采样元数据：部分提供（event=cpu-clock；percent-type=local period（非 global-period）；单次运行窗口；函数级 workload 贡献=testcase 热点 rank 001）→ 限制 workload 级上界
- Sampling IP precision：缺失（`baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v` 存在（RVV 1.0）；crypto 相关：`zvbb/zvbc/zvkb/zvkg/zvkned/zvknha/zvknhb/zvksed/zvksh/zvkt`；出处：metadata cpuinfo（SpacemiT X100） |
| Build ISA | **DSO 已启用 `v`（直接证据，跨函数确认）**：同 DSO 内 function 002（ossl_gf_mul）含 `vsetivli zero,2,e64,m1`+`vle64.v/vadd.vv/vse64.v`、003（ossl_gf_sqr）含 `vle64.v/vslidedown.vi/vmv.x.s`、004/005 含 `vrgather.vv` 等 RVV 1.0 指令 → libcrypto.so.4 已按含 `v` 的 `-march` 构建；openssl_bench 的 attribute 无 `v` 仅代表该可执行文件对象。`baseline_gap: build ISA`（libcrypto.so.4 精确 extension 集仍待 `readelf -A` 确认） |
| Vector flavor | annotate 内 function 001 无 `v*`，但同 DSO 已含 RVV 1.0 `v*` → 无 flavor mismatch（无 xtheadvector 参与） |
| VLEN | 256 bits（vlenb=32） |
| Bound type | compute/latency-bound：IPC=2.829；L1_dcache_load_miss_rate=0.017%；branch_miss_rate=0.537% → 非 memory/branch-bound，本地 compute/instruction-throughput 修复是正确杠杆 |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent-type=local period（**非 global-period**）；同一运行窗口；函数级贡献=testcase rank 001 → 只可表述「当前 event 下的局部样本份额」，不得给 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`） |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP 证据）→ 高占比行只锚定 loop interval，不做单指令 latency 归因 |

L0 baseline gate 1（hardware 有 `v` 而 build 无 `v`）：**不成立**——hardware 有 `v`，build（libcrypto.so.4）也已启用 `v`（function 002–005 annotate 直接证据）。function 001 的 constant_time_lookup 循环零 `v*` 属 **autovec 覆盖缺口**（compiler 未向量化该循环；constant_time.h 的 value_barrier 空 asm 与循环结构是候选阻碍因素），**不是 build-ISA 缺失**。RVV 依赖的 route 不需要重建前提。
L0 baseline gate 2（`th.v*` flavor gate）：不触发（无 `th.v*` 指令）。
Bound-type gate：compute-bound（IPC 2.829、miss 率极低）→ 本地 compute-vectorization 修复不因 memory-bound 降 impact。

## Phase 2 — Scope / 分析边界

- 函数清单与承诺声明一致：`ossl_curve448_precomputed_scalarmul`（1 个）。annotate 覆盖完整（含全部循环体）→ 入口条件 A（`profile_backed`）。
- 函数结构：ed448 comb 预计算标量乘（`constant_time_lookup_niels` 查表 + `niels_to_pt`/`add_niels_to_pt` + `point_double_internal` 折半）。全部 157 个局部样本落在 `constant_time_lookup` 的**内层 j-loop**（0x15d662–0x15d676）；外层 row-loop（0x15d64c–0x15d67e）的 mask 计算与 memset 均为 0.00%。
- hot loop interval：0x15d662–0x15d676（每字节 8 条指令：2×`lbu` + `addi` + `and` + `or` + `sb` + `addi` + `bne`；每字节 2 load + 1 store）。
- 最高行原文（trace anchor）：`17.83 :   15d66e: or      a5,a5,a1`；该 interval 各行合计 ≈ **100%**（157/157 局部样本）。
- Sampling IP precision 未确认 → 归因收敛到 interval 级机制（该循环整体承担全部局部样本），不对单条指令做 latency/cycle 归因。

## Phase 3 — Pattern scan / 模式扫描：ossl_curve448_precomputed_scalarmul

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源是 compiler-generated（annotate 内嵌 constant_time.h C 源行 339/353/469–475，非 `.S` symbol）；无 policy-backed missing-`.S` 四证（无 dispatch slot、无 scalar-fallback 注册证据、无要求独立 `.S` 的函数级 policy 证据）
2. `rows-operator-rvv.md` — **include** — compiler-generated scalar loop；hot interval 由 unit-stride load、简单 select+OR 算术、store 与循环控制主导，符合 elementwise select 语义合同
3. `rows-string-memory.md` — **include** — 192 字节整行逐字节 masked OR-accumulate（类 copy/fill 数据流 + 宽访问问题需判别）
4. `rows-vectorized-tuning.md` — **exclude** — 本函数执行代码零 `v*`，无 RVV 配置/寄存器/展开可调（同 DSO 其它函数含 `v*` 属各自函数范畴）
5. `rows-codegen.md` — **include** — compiler-generated 指令形态（addressing-mode、register pressure、ISA substitution、native-width）需判别
6. `rows-offload.md` — **exclude** — 无矩阵引擎/packed-SIMD 证据
7. `rows-crypto.md` — **include** — workload 是 crypto（ed448-sign），需对 4 个 crypto row 逐一判别
8. `rows-runtime-os.md` — **exclude** — 用户态 crypto benchmark，无 RTOS/kernel/CSR 热点

### Classes scanned: `rows-operator-rvv.md`、`rows-string-memory.md`、`rows-codegen.md`、`rows-crypto.md`

### Local performance pattern scan

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Contiguous Elementwise Arithmetic Kernels（primary） | `constant_time_lookup` 内层 j-loop（0x15d662–0x15d676）100% 局部样本落在逐字节 `lbu`/`and`/`or`/`sb` + 指针递增 + 回边；每字节 8 指令、3 内存操作 | High | Medium | `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` |
| No vectorization（supporting） | 同一 interval zero `v*`；hardware 有 `v`（RVV 1.0）；build（libcrypto.so.4）也已启用 `v`（function 002–005 直接证据）→ autovec 覆盖缺口 | — | — | `patterns/no-vectorization.md` |

#### Primary：RVV Contiguous Elementwise Arithmetic Kernels

**(a) 逐字 evidence 引用**（基本块 0x15d662–0x15d676，`constant_time_lookup` 内层 j-loop，rowsize=192）：

```
15.29 :   15d662: lbu     a5,0(a3)      ; tablec 行字节 unit-stride load
14.65 :   15d666: lbu     a1,0(a4)      ; outc 字节 load（读-改-写）
 9.55 :   15d66a: addi    a4,a4,1       ; out 指针递增
 0.00 :   15d66c: and     a5,a5,a2      ; 与行内不变量 mask（constant_time_select_8）
17.83 :   15d66e: or      a5,a5,a1      ; OR 累加进 out
12.10 :   15d670: sb      a5,-1(a4)     ; 逐字节写回
16.56 :   15d674: addi    a3,a3,1       ; table 指针递增
14.01 :   15d676: bne     s10,a4,15d662 ; 192 字节行内回边
```

interval 合计 ≈100%（157/157 局部样本）；`and` 行 0.00% 属 sampling 分布（同一基本块，skid/IP 精度未确认），interval 级归因不受影响。外层 row-loop 每次查表 16 行（`bne a6,s9,15d64c`，每行 +192B）；comb 阶梯 18(i)×5(j)=90 次 `constant_time_lookup_niels`（ed448 COMBS_S=18/COMBS_N=5，disassembly 常量 s9=18、s2=5 相符）。

**(b) 互斥邻居排除**（逐条判别性观察）：
- elementwise-activation row：interval 内只有整数 `lbu/and/or/sb/addi/bne`，无任何 FP activation/math helper → 排除
- normalization row：mask（a2）为外层 row-loop 不变量、非跨 lane 统计；每字节输出只依赖同 index table 字节 → 排除
- extrema / arg-extrema / widening-reduction rows：无 loop-carried accumulator；`out` 字节由内存读-改-写携带、逐元素独立；OR 累加发生在跨 row（同 lane），非跨 lane → 排除
- strided-layout / layout-packing / color / gather rows：访存为 `lbu 0(a3)`+`addi a3,a3,1` unit-stride 顺序流，无固定 stride、无 data-dependent index（gather 要求 `dst[i]=table[idx[i]]`，本循环 idx 与索引无关）→ 排除
- assembly rows（`riscv-assembly-kernel-performance-optimization` / `hand_tuned_assembly_unrolling_and_register_management`）：provenance 是 compiler-generated C inline（constant_time.h 源行可见），非手写 `.S` → 排除
- copy/fill row（`rvv_memory_copy_fill_implementation.md`）：非纯 copy——每字节执行 `and`+`or` 算术，`out` 为读-改-写而非整行覆写，且 OR 跨 16 行累加；更具体的 select 语义由 elementwise row 认领 → 排除
- wide-scalar-memory row（`wide_scalar_memory_access_codegen.md`）：行内判据互斥「完整 buffer traversal → RVV copy/fill」，本循环是 192 字节整行遍历而非 small aggregate 物化 → 排除
- kernel-selection row：仓库中无面向 `constant_time_lookup` 的现成 RVV 专用实现（无 dispatch 证据、执行代码零 `v*`）→ 排除
- crypto rows（dedicated vector crypto / cipher-mode / carry-less / polynomial-table-reuse）：本循环是 comb 预计算表的 **constant-time 查表**，非 AES/SHA/SM3/SM4/GHASH 原语；ed448 为素域（2^448−2^224−1）算术、非 GF(2^k) 二元域乘法（域乘在独立函数 `ossl_gf_mul`）；无 CRC/polynomial table ownership/reuse 问题 → 全部排除
- codegen rows（addressing-mode fusion / register pressure / ISA substitution / native-width）：interval 内 load 均用 `0(a3)`/`0(a4)` 直接寻址（无独立 address-gen 序列）；hot loop 仅用 a1–a5 等 4–5 个 GPR、无 spill/reload；`and`+`or` 非 Zbb `andn`/`orn` 可替形态（后者是 `(~a)&b` 形态）；字节宽度是源码语义宽度、无 truncate/extend churn → 全部排除

**(c) 双 Confidence 推导式**：
- route: compiler-generated provenance（annotate C 源行直接证据）+ elementwise select+OR 语义合同成立 + unit-stride 连续证据 + 行内互斥全部由具体指令判别 → **High**
- impact: interval sample share=100%（157/157 局部）+ testcase rank 001 + VLEN=256 已知 + bound=compute（IPC 2.829）→ 但 percent-type=local period（采样语义四条不全）→ **Medium**

#### Supporting：No vectorization（rows-operator-rvv.md 行内判据）

**(a) 逐字 evidence 引用**：本函数 hot interval 全部 8 行均为标量 `lbu/and/or/sb/addi/bne`（见上），零 `v*` 与零 `th.v*`；hardware ISA 含 `v`；同 DSO function 002–005 annotate 含 `vsetivli/vle64.v/vadd.vv/vrgather.vv/vse64.v`（build 已启用 `v`）。
**supporting because**: 解释同一 hot interval「缺少向量执行」的载体事实——build 与硬件均支持 RVV，但 autovec 未覆盖该 constant-time 逐字节循环（覆盖缺口），不决定贡献载体与修复机制；自身 row gate 成立（generic scalar main loop + zero-`v*` + hardware `v`，且更具体 semantic row 已认领）。
两种 confidence 保持 `—`，不另起顶层 row、不计入命中数。

### 多命中仲裁小段

- 顶层 finding 1 个（primary：RVV Contiguous Elementwise Arithmetic Kernels），supporting 1 个（No vectorization），无 companion（glibc companion 白名单不适用）、无 independent。
- 因果消除测试：对同一 hot interval、同一机制（逐字节 select+OR 循环），elementwise semantic row 认领唯一 primary；no-vectorization 只解释零 `v*` 载体。L0 baseline finding 修正：**build 已启用 v（跨函数证据），非 build-ISA 缺失**——本函数为 autovec 覆盖缺口，L1 判定不受影响。
- Evidence-mechanism layer：L1 = vectorization/semantic dispatch（elementwise select+OR + no-vectorization supporting）。
- 收益上界排序（入口条件 A）：单一顶层 finding，evidence sample share 加总 = 100%（157/157 局部样本，interval 0x15d662–0x15d676）。

## Phase 4 — Root-cause blueprint / 根因蓝图：ossl_curve448_precomputed_scalarmul

**对应 Phase 3 通过 gate 的 row**：primary = `RVV Contiguous Elementwise Arithmetic Kernels`（rows-operator-rvv.md 首行）；supporting = `No vectorization`（同文件末行）。

**1. Root cause**：`constant_time_lookup_niels` → `constant_time_lookup(out, table, 192, 16, idx)` 的**内层 j-loop 是逐字节 scalar 的 elementwise masked select + OR 累加**。每处理一个字节都要执行「加载、算术、写回、地址更新和循环分支」（依据 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` §Why this is slow intro），「同一组指针更新、边界判断和回跳分支需要按元素重复执行」（§Why this is slow #1），且「RVV unit-stride `vle*` 和 `vse*` 使用更少的架构指令描述同一连续区间」（§Why this is slow #2）。8 条动态指令/字节 × 16 行 × 192 字节 × 90 次查表 ≈ 221 万条指令/标量乘，全部局部样本（157/157）集中于此。目标硬件 RVV 1.0、VLEN=256，SEW=8/LMUL=8 时 VLMAX=256 ≥ 192，**一整行只需一次向量迭代**即可完成同样语义；同 DSO 已按含 `v` 构建（function 002–005 annotate 直接证据），但本循环未被编译器自动向量化（autovec 覆盖缺口），向量执行资源全程闲置（§Why this is slow #3「Vector execution resources remain unused」）。正确性前提全部满足：每字节输出只依赖同 index 输入（无跨元素依赖）、unit-stride 连续、mask 为行内不变量（§When to apply 前提清单）。

**2. The fix / 修复方式**（依据 elementwise pattern §The fix；不构成直接编辑或补丁）

前置（已确认，无需重建）：build（libcrypto.so.4）已含 `v`（function 002–005 annotate 的 `vsetivli/vle64.v/vadd.vv/vrgather.vv/vse64.v` 直接证据）→ RVV 改写不需要重建前提；`readelf -A` 仅用于确认精确 extension 集。

主修复（elementwise §The fix #1/#2/#9——VLEN-agnostic RVV strip-mined masked-select loop）：

```
Before（当前 annotate 等价形态，scalar per-byte，8 instr/byte）:
for (j = 0; j < rowsize; j++)
    outc[j] |= (unsigned char)(tablec[j] & mask);   /* mask 行内不变量 */
// ≈ lbu / lbu / and / or / sb ×192

After（RVV 形态，SEW=8，动态 vl，unit-stride）:
size_t remaining = rowsize;
const uint8_t *t = tablec; uint8_t *o = outc;
while (remaining > 0) {
    size_t vl = __riscv_vsetvl_e8m8(remaining);          /* VLEN=256 时 VLMAX(e8m8)=256 ≥ 192 */
    vuint8m8_t tb = __riscv_vle8_v_u8m8(t, vl);          /* 行字节 load */
    vuint8m8_t ob = __riscv_vle8_v_u8m8(o, vl);          /* out 读-改-写 load */
    tb = __riscv_vand_vx_u8m8(tb, mask, vl);             /* mask 标量广播：vand.vx（mask 0x00/0xff 已为 e8 正确宽度） */
    ob = __riscv_vor_vv_u8m8(ob, tb, vl);                /* OR 累加 */
    __riscv_vse8_v_u8m8(o, ob, vl);                      /* 向量写回 */
    t += vl; o += vl; remaining -= vl;
}
```

LMUL 选择（§The fix #9）：峰值 live 向量 = tb(m8) + ob(m8) = 16 组 ≤ 32（m8 合法）；保守取 m2/m4 亦可，按 X100 实测与 live-set 预算决定，不得默认 m1 最优或 m8 最优。

标量中间步（不依赖 autovec 行为、保持 constant-time 的即时收益）：64-bit word-wide masked select——`mask64 = neg(andi(mask,1))`（0 或 −1）一次构造；每 8 字节 `ld table / ld out / and / or / sd`（rowsize=192=24×8，行内对齐）→ 迭代数降至 1/8、内存操作降至每 8 字节 3 次。

Correctness contract（不可破坏）：① constant-time 不变量——无秘密依赖分支/索引，位运算无 timing 侧信道，mask 全 0/全 1 路径逐位一致；② 语义——`out` 先 memset 后逐行 OR 累加，RVV/word 版本保持逐位相同结果；③ 边界——动态 `vl` 保证不越界（tail 由 `vsetvl` 覆盖，本场景 VLEN=256 单次覆盖 192 字节无 tail）；④ 别名——table 行与 `out` 为不同缓冲区（constant_time_lookup 合同），无 overlap。
限制/风险：RVV 改写需显式 intrinsic（或调整 autovec 可达性）；若项目按 runtime dispatch 接 RVV，需确保真实 workload 到达该路径；LMUL/展开需在 X100（OoO）实测；intrinsic API 名称随工具链版本调整。
预期 Profile signals：出现 `vsetvli`/`vle8.v`/`vand.vx`/`vor.vv`/`vse8.v`；`15d662–15d676` 的 `lbu/and/or/sb` 逐字节循环局部样本份额降至 ~0。

**3. Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（含 `v`，RVV 1.0，VLEN=256）；build ISA=**DSO 已含 `v`**（function 002–005 annotate 直接证据；精确 extension 集 gap）；VLEN=256 bits；bound type=compute（IPC 2.829、L1 miss 0.017%、branch miss 0.537%）。

**4. 收益上界**：当前 sampled event（cpu-clock）下该 finding 的**局部样本份额 = 100%**（157/157，interval 0x15d662–0x15d676）；因 percent-type=local period 且函数 workload 贡献仅有 rank 001 定性信息，**不得表述为 workload 级 Amdahl 上界**（`baseline_gap: sampling metadata`）。

**5. 三维路由判定**：
- current source：compiler-generated scalar C（constant_time.h inline，annotate 内嵌 C 源行 339/353/469–475；非 `.S`）——直接证据
- implementation existence/reachability：无现成 RVV `constant_time_lookup` kernel 证据；本函数执行代码零 `v*`，但同 DSO 已含 RVV 代码（build 支持）→ RVV 实现不存在于该路径，需显式改写（intrinsic 或 autovec 可达性调整）
- function-level policy：OpenSSL 有 RVV/汇编优化的 runtime 检测惯例，但本路径无独立 `.S` policy 证据 → 不进入 policy-backed missing-`.S` 分支（policy/existence 四证缺：无 dispatch-slot 证据、无 scalar-fallback 注册证据、无要求独立 `.S` 的函数级 policy、无源码级目标 `.S` 缺失记录）

**6. Implementation-shape proof**：不适用（未进入 policy-backed missing-`.S` 分支）。

**7. Related PRs 小节**：
- `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` → `Related PRs：22 条 URL`
  - https://github.com/OpenMathLib/OpenBLAS/commit/45fd2d9b0790c5ca3698502d65d59d38d911ef4f
  - https://github.com/alibaba/MNN/pull/3913
  - https://github.com/alibaba/MNN/commit/5376580ba19ac4034dc373f566fca846326fd612
  - https://github.com/alibaba/MNN/pull/3779
  - https://github.com/alibaba/MNN/commit/b35da10227477e90d5e5be55e1e5e646d4af41e4
  - https://github.com/alibaba/MNN/commit/815ed5d6cb7052e2294d9eb6298e9e23b2fdf91d
  - https://github.com/uxlfoundation/oneDNN/commit/2b1dfe2d6233696bc2c803b48c06c00b29f7f866
  - https://github.com/uxlfoundation/oneDNN/pull/5265
  - https://github.com/uxlfoundation/oneDNN/commit/1184757c286814e552c137ca354060a30a9872bb
  - https://github.com/uxlfoundation/oneDNN/pull/5079
  - https://github.com/uxlfoundation/oneDNN/commit/de1342a9d1dfe8419dd5159a16013529e567a6bc
  - https://github.com/uxlfoundation/oneDNN/commit/580b9c80484f5df175ad36870280703ea5767cbe
  - https://github.com/uxlfoundation/oneDNN/commit/a0961ab37e4ccf7dec0a0fd05fab92c1fc812e38
  - https://github.com/uxlfoundation/oneDNN/commit/1147a0739a1fa1ee881075ac1cb8dd8f05e26cb5
  - https://github.com/uxlfoundation/oneDNN/commit/595fc3b9bf46a5381d337af4ae0d7c529e2a9bcd
  - https://github.com/openjdk/jdk/commit/6700baa5052046f53eb1b04ed3205bbd8e9e9070
  - https://github.com/openjdk/jdk/commit/885be2efa6b1359a7c7ab36882e19a7eaba77fb3
  - https://github.com/openjdk/jdk/commit/9b61a7608efff13fc3685488f3f54a810ec0ac22
  - https://github.com/v8/v8/commit/2b368def484809ad8d35b0c5d5f913bd95ad23ef
  - https://github.com/v8/v8/commit/56dd6a2f1ee28b2d37989a7888ad178a89f4f5ea
  - https://github.com/alibaba/MNN/pull/4042
  - https://github.com/alibaba/MNN/commit/672c5862392393c171f1513bf7994d3b95e2a6a1
- `patterns/no-vectorization.md`（supporting，不单独出蓝图）：`Related PRs：19 条 URL`（OpenCV scalable RVV 系列，见该文件表格）

## Phase 5 — Verification forecast / 验证预测：ossl_curve448_precomputed_scalarmul

primary finding 的修复对象：`constant_time_lookup` 内层 j-loop 的逐字节 scalar masked-select+OR。

- 应消失/缩小侧（锚 Phase 3(a) 逐字引用行）：`15d662: lbu a5,0(a3)`、`15d66e: or a5,a5,a1`、`15d670: sb a5,-1(a4)`、`15d676: bne s10,a4,15d662` 所在的逐字节循环局部样本份额应从 100% 降至 ~0（scalar `lbu/or/sb` 不再主导 hot loop）。
- 应出现侧（锚 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` §Verification）：同一函数 annotate 出现 `vsetvli`/`vsetivli`、unit-stride `vle8.v`、`vand.vx`、`vor.vv`、`vse8.v`（§指令验证）；`readelf -A libcrypto.so.4` 的 attribute 包含 `v`（§ISA 属性验证——当前已由同 DSO 内 v* 指令直接证明，此项作精确 extension 集确认）；与 scalar reference 逐位一致对比，覆盖 mask=全 0、全 1、交错/随机（§mask/select 验证）；长度与 tail 验证覆盖 0、1、<VLMAX、=VLMAX、>VLMAX（§长度和 tail 验证——本函数行恒为 192 字节，需额外验证 VLEN<1536bit 平台之外 VLMAX(e8m8)<192 时的 tail 行为）；别名验证（table 行与 out 不同缓冲区，§别名验证）；内存边界（§内存边界验证）；实现可达性（确认真实 workload 进入 RVV 路径而非继续 scalar fallback，§实现可达性验证）；性能验证（ed448-sign benchmark 对比 cycles/instructions/throughput，§性能验证）；带宽分析（§带宽分析——192 字节行驻留 L1，无 cache 拐点风险）。
- supporting（no-vectorization）不单独验证，跟随 primary 预测。
- L0 前置确认（已满足）：build（libcrypto.so.4）已含 `v`（function 002–005 annotate 直接证据），无需重建（no-vectorization §The fix #1 的 build 前置在本环境已成立）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor payload |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅ | 1/1 组；[ossl_curve448_precomputed_scalarmul] |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表齐；gap 标签：`baseline_gap: build ISA`（仅精确 extension 集）、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；L0 gate 1 判定「不成立（DSO 已含 v，跨函数证据）」+ 修正说明；bound-type gate 已写 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md、rows-string-memory.md、rows-codegen.md、rows-crypto.md`；顶层 finding 1 个 + supporting 1 个；evidence 锚点：`15.29 : 15d662: lbu a5,0(a3)`、`17.83 : 15d66e: or a5,a5,a1`、`12.10 : 15d670: sb a5,-1(a4)`、`14.01 : 15d676: bne s10,a4,15d662`；互斥排除 11 条；双 confidence 推导式已写 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`patterns/rvv_contiguous_elementwise_arithmetic_kernels.md`（命中 row：RVV Contiguous Elementwise Arithmetic Kernels；引用短语：「加载、算术、写回、地址更新和循环分支」「同一组指针更新、边界判断和回跳分支需要按元素重复执行」「Vector execution resources remain unused」）、`patterns/no-vectorization.md`（supporting）；The fix 含 before/after、correctness contract（constant-time/语义/边界/别名）、风险、预期 Profile signals；`Related PRs：22 条 URL`（primary） |
| 5 | 路径合规 | ✅ | 模式 A（profile_backed）；8 类 trace 可解释；primary/supporting 分账正确；L0 修正（build 已含 v → autovec 覆盖缺口）已并入；按局部样本份额排序（100%）；`th.v*` 未触发、未全局停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`15d662: lbu a5,0(a3)` / `15d66e: or a5,a5,a1` / `15d670: sb a5,-1(a4)` / `15d676: bne s10,a4,15d662`；出现侧：`patterns/rvv_contiguous_elementwise_arithmetic_kernels.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 |

修正记录：Phase 1/Phase 4/Phase 5 依据同 DSO function 002–005 的 RVV 指令直接证据，将「build 未启用 v」修正为「build 已启用 v、本循环为 autovec 覆盖缺口」，移除重建前提；primary root cause（逐字节 scalar masked select+OR）与 The fix 主体不变。