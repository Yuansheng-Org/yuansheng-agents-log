# AI 补丁审核报告

## 审核范围

- **蓝图**：`.yuansheng/trace/mxnet/category-activation/003_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_tru/blueprint_mxnet_category-activation_003.json`（`bp-mxnet-category-activation-003`，diagnosis `probable_root_cause`，confidence 0.75）
- **诊断日志**：同目录 `diaglog_mxnet_category-activation_003.md`（Phase 1–6）
- **PatchPlan**：`pp-bp-mxnet-category-activation-003`
- **PatchCandidate**：`pc-bp-mxnet-category-activation-003`
- **补丁**：`<OUT>/craft/patch.diff`（`git diff` 与工作树逐字节一致，已用 `diff` 校验 `DIFF_IDENTICAL`）
- **变更文件**：`CMakeLists.txt`（+20 行）、`src/operator/nn/softmax-inl.h`（+144 行），无删除、无重格式化
- **目标硬件**：SOPHGO SG2044（XuanTie C920v2，RVV 1.0，VLEN=128，`vlenb=16`）；Build ISA `rv64…v1p0…zvl128b`（无 `zfh`）
- **审核方式**：只读审核。未修改任何源码/蓝图/PatchPlan/patch.diff/PatchCandidate；未在本机执行构建（环境无 `riscv64-*-g++` 交叉工具链，仅 `g++/clang++` x86），构建与运行验证列为残余风险
- **变更目的**：向量化 `Softmax<softmax_fwd, negate=true, AType=double, DType=float, OType=half_t, IType=int, ndim=2>` 的 CPU 三遍归一化，并启用 `zfhmin/zvfhmin` 以用原生 FP16 转换替代软件 `float2half`

## RISC-V 架构审核

`arch-scan` 判定 `archSpecific: true`，命中规则：`riscv-macro`、`march-flag`、`vlenb`。以下专项审核**仅依据**蓝图/诊断日志证据 + diff 原文，不引入模型外部推断。

**ISA 成员资格 vs 目标硬件（RVV 1.0 / VLEN=128 / Build ISA）**

| 用到的指令/内建 | 所属扩展 | 目标硬件/构建是否具备 | 结论 |
|---|---|---|---|
| `vle32.v` / `vse32.v` / `vse16.v` / `vsetvl` / `vsetvlmax` | 基础 V（zve32x） | 硬件 `zve32x`；build `v1p0`/`zvl128b` | 具备 |
| `vfneg.v` / `vfmax.vv` / `vfmin.vv` / `vfadd.vv` / `vfsub.vf` / `vfmul.vf` / `vfmul.vv` / `vfmadd.vv` / `vfnmsac.vf` / `vfdiv.vf` | 基础 V + F（zve32f） | 硬件 `zve32f`；build `zve32f1p0` | 具备 |
| `vfredmax.vs` / `vfredusum.vs` | 基础 V（RVV 1.0） | 硬件 `v1p0`；build `v1p0` | 具备 |
| `vfcvt.x.f.v` / `vfcvt.f.x.v` | 基础 V + F | 同上 | 具备 |
| `vmfgt.vf` / `vmfne.vv` / `vmerge.vvm` | 基础 V | 同上 | 具备 |
| `vfncvt.f.f.w`（f32→f16，`vfloat16mf2_t`） | `zvfhmin` | 硬件 `zvfhmin`；build 经本补丁增补 `zvfhmin` | 具备（构建前置已修） |
| `-march=rv64gcv_zfhmin_zvfhmin` | g+c+v+zfhmin+zvfhmin | 硬件 `rv64imafdcv…zfh_zfhmin_zvfhmin…` | 匹配 |

**无 x86/ARM 指令**：全部为 RVV/标量 RISC-V 内建；`__builtin_inff()` 为编译器内建，非平台指令。非 RISC-V 目标由 `#if defined(__riscv) && defined(__riscv_v)` 完全屏蔽，x86/ARM/CUDA 路径字节不变。

**vsetvl / vtype（LMUL/SEW）一致性**：全函数统一 `e32m1`（SEW=32，LMUL=1）；动态尾循环使用 `__riscv_vsetvl_e32m1(M - j)`，`vlmax` 由 `__riscv_vsetvlmax_e32m1()` 取得。f16 存储使用 `mf2`，是 m1/f32 → mf2/f16 的规范收窄比（VLEN=128 时 m1f32=4 元素、mf2f16=4 元素），寄存器组无重叠、无越界。峰值活跃向量数（约 5–7 个 m1）远低于 32 组预算。

**tail / mask 策略**：pass1/pass2 使用 tail-agnostic 累加（`vfmax.vv`、`vfadd.vv` 在 `vl` 之外保持目标不变），尾块仅更新前 `vl` 条 lane，`vfredmax/vfredusum` 以 `vlmax` 归约全部 lane —— 未触及 lane 保留的是此前迭代的累积值（或 seed），数学上等价于全行归约，正确。pass3 为纯逐元素，尾块直接 `vse` `vl` 条，正确。

**scalar↔vector 边界**：seed 用 `vfmv_v_f`，标量提取用 `vfmv_f_s`（取元素 0，与 `vfred*` 结果落点一致）。`in/out` 指针在 `sa == 1` 时连续，`SoftminRow(in + base, out + base, M)` 直接按连续行处理，与标量 `in[base + j*sa]`（sa==1）等价。

**dispatch gating**：编译期 `if constexpr`（`OP==softmax_fwd && DType==float && OType∈{float, half_t(zvfhmin)}`）叠加运行期 `negate && temperature==1.0 && sa==1 && M>=8`；其余情形（negate=false、非 float、temperature≠1.0、sa≠1、length!=nullptr、短行、其他 OType）全部回落标量循环，`temperature` 慢路径分支保留。gating 正确且保守。

**exp 范围归约专项（重点核验）**：
- `vfnmsac.vf vd, rs1, vs2` 语义为 `vd[i] = vd[i] - rs1*vs2[i]`。diff 中 `r = vfnmsac(xc, ln2_hi, nf)` → `r = xc - ln2_hi*nf`；`r = vfnmsac(r, ln2_lo, nf)` → `r = r - ln2_lo*nf`。**两项 ln2 均乘以整数 `nf`**，即 `r = x - k*ln2_hi - k*ln2_lo`。
- 该实现**未复现**此前同族补丁「ln2_hi 未乘以 k」的严重缺陷。判定：**范围归约正确**。
- 多项式为 `exp(r)` 在 `|r| ≤ ln2/2` 的 7 阶 Taylor Horner（`r^7/7!+…+1`），系数与展开逐项一致；`|r|=ln2/2≈0.3466` 处截断相对误差约 `3.6e-9`，低于 f32 eps。
- `2^n` 由 `(n+127)<<23` 构造；`n∈[-126,127]` → 偏置指数 `[1,254]`，不落入非规格化/溢出区。
- 符号：`negate` 在 pass1/pass2/pass3 各以**一次** `vfneg.v` 施加（`-in`），softmin = softmax(-x) 语义保持；`-0.0` 经 `vfneg` 变 `+0.0`，与标量 `-in[...]` 一致。

## 审核结果

**reviewResult = pass**（`archReview.status = passed`）。补丁以最小侵入方式命中蓝图根因，约束 `constraints.mustPreserve` 逐条守住，候选与 diff 一致，无密钥/危险命令。未发现 critical/major 问题；共 7 条 minor/suggestion 级发现（见下），不阻断交付。

**根因覆盖**
- 主根因（RVV Normalization Kernels：未向量化 3-pass scalar softmin + 逐元素 `expf`）→ 新增 `softmax_rvv::SoftminRow` 三遍向量化（`vfneg.v` 一次/向量、`vfredmax.vs`/`vfredusum.vs` 归约、自包含向量 exp、`vfdiv.vf` 缩放），与诊断日志 Phase 4 的 fix 方向逐项对应。
- 独立根因（ISA-substitution：软件 `float2half`）→ CMake 探针增补 `-march=rv64gcv_zfhmin_zvfhmin`（回退 `rv64gcv`），half_t 存储走 `vfncvt.f.f.w`，符合蓝图「保留无扩展时 fallback」。

**mustPreserve 逐条核对**
| 约束 | 结论 | 依据 |
|---|---|---|
| softmin 负号语义（含 -0.0） | 保持 | pass1/2/3 各一次 `vfneg.v`；`-0.0`→`+0.0` 与标量一致 |
| 稳定减 max | 保持 | pass1 取 `max(-in)`，pass2/3 减 `mmax` |
| NaN / ±Inf / 全相等 / 指数和为零 | 大体保持 | 全相等→均匀；NaN 经 `vsum` 传播致整行 NaN（与 reference 一致）；`-Inf` 见 F2 |
| FP16 RNE rounding | 保持 | `vfncvt.f.f.w` 默认 `frm=RNE`，与 `MSHADOW_HALF_ROUND_TO_NEAREST==1`（`3rdparty/mshadow/mshadow/half.h:36-37`）对齐 |
| 公共 API / 模板签名 | 保持 | `Softmax<OP,negate,AType,DType,OType,IType,ndim>` 签名与 `softmax.cc` 调用未改 |
| temperature 分支语义 | 保持 | 向量路径仅 `temperature==1.0` 生效；`/temperature` 慢路径原样保留 |

**聚焦范围**：仅 2 文件、+164 行，无顺带重格式化；`if constexpr`/宏守卫保证非 RISC-V 与非 RVV 构建零影响。

## 发现问题

以下均为 minor/suggestion，不构成阻断；每条 `file` 均以 `+++ b/<path>` 形式存在于 `patch.diff`。

- **F1 [minor] fp32 无序归约 vs `AType=double` 累加合同** — `src/operator/nn/softmax-inl.h`。diff 原文：`+    vsum            = __riscv_vfadd_vv_f32m1(vsum, VecExp(v, vl), vl);` 与 `+  const float sum = __riscv_vfmv_f_s_f32m1_f32(__riscv_vfredusum_vs_f32m1_f32m1(`。标量 reference 以 `AType sum = double` 有序累加、并以 double 作除法（`Softmax` 模板 `sum += std::exp(...)`；`softmax_fwd::Map` 的 double 重载），向量路径改为 f32 累加 + `vfredusum`（无序）+ f32 除法。诊断日志 Phase 4 已显式提示「`vfredusum` 改变 FP 结合顺序（AType=double 合同）需评估」。对 `OType=half_t` 输出该差异远小于 FP16 分辨率（~1e-3），实际影响可忽略；建议在验证记录中给出容差说明，或按需改用 `vfredosum` / 双精度累加。
- **F2 [suggestion] `exp(-Inf)` 被钳位为最小规格化数而非精确 0** — `src/operator/nn/softmax-inl.h`。diff 原文：`+  vfloat32m1_t xc = __riscv_vfmax_vv_f32m1(` / `+      x, __riscv_vfmv_v_f_f32m1(-87.33654475f, vl), vl);`。当某元素 `in=+Inf` 时 `x=-Inf`，reference `expf(-Inf)=0`，向量路径得 `≈1.175e-38`。对 half_t 输出该值下溢为 0，无可观测差异；仅 `OType=float` 时存在极小非零偏差。如需位级一致，可对 `-Inf` 增加一次 `vmerge` 置 0。
- **F3 [suggestion] 上钳位带内多项式设计区间被突破** — `src/operator/nn/softmax-inl.h`。diff 原文：`+  // exp(r) on |r| <= ln2/2 via a degree-7 Horner polynomial.`。当 `x∈[≈88.376, 88.722839]` 时 `rint(x*log2e)=128` 被钳到 `127`，归约余项 `r` 最大可达 `≈ln2`（超出 `ln2/2`），7 阶 Taylor 相对误差升至约 `6.6e-7`（约 11 ULP）。softmin 域恒有 `x ≤ 0`，该分支为防御性死代码，当前不可达；若未来复用 `VecExp` 于一般指数场景需注意。
- **F4 [suggestion] `vfcvt.x.f.v` 依赖动态舍入模式 `frm`** — `src/operator/nn/softmax-inl.h`。diff 原文：`+  vint32m1_t n   = __riscv_vfcvt_x_f_v_i32m1(t, vl);  // round-to-nearest-even`。该内建使用当前 `frm`（默认 RNE），而非硬编码 RNE；若上层修改 `frm`，`n` 可能偏差 1。mxnet 未改动 `frm`，风险低；可在注释中明确前提。
- **F5 [minor] CMake `-march` 追加可能覆盖既有更丰富 `-march`** — `CMakeLists.txt`。diff 原文：`+        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=rv64gcv_zfhmin_zvfhmin")`。GCC/Clang 以最后一个 `-march` 为准；若工具链/环境已设置含 `zba/zbb/zbc` 等的 `-march`，追加项会将其丢弃。`check_*_compiler_flag` 只验证「该 flag 可编译」，不检测是否覆盖既有 `-march`。建议仅在未设置 `-march` 时追加，或用 `add_compile_options` + 条件判断。
- **F6 [suggestion] NaN 策略未被机器验证** — `src/operator/nn/softmax-inl.h`。diff 原文：`+  vbool32_t isnan = __riscv_vmfne_vv_f32m1_b32(x, x, vl);`。reference 的 `mmax` 初始化取首元素且 `if (mmax < val)` 忽略 NaN，与 `vfmax` 的 IEEE maxNum 语义在「首元素为 NaN」时行为不同；本实现最终经 `sum` 的 NaN 传播使整行输出 NaN，与 reference 恰好一致，但该等价性未由测试固定。建议补充 NaN/±Inf 用例。
- **F7 [suggestion] 构建配置探针与运行未在本环境验证** — `CMakeLists.txt`。diff 原文：`+    check_cxx_compiler_flag("-march=rv64gcv_zfhmin_zvfhmin" MXNET_RISCV_HAS_GCV_ZFHMIN)`。`CheckCXXCompilerFlag` 已在同文件 244 行提前 `include`，`check_cxx_compiler_flag` 可用，逻辑自洽；但本审核环境无 RISC-V 交叉工具链，未实际 configure/build/run。建议按蓝图 `agent1 patch_regression --case category-activation` 复测。

## 幻觉自检

- **技术精度** [PASS]：所有 ISA/内建归属、`vfnmsac` 语义、多项式系数、`2^n` 位构造、LMUL 收窄比、hunk 原文均逐一核对，未使用未经证据支持的性能数字；收益表述沿用蓝图的「local period 局部样本份额」，未升格为 workload 级 Amdahl 上界。
- **声明溯源** [PASS]：每条 finding 的 `evidence` 均引用 `patch.diff` 真实行文或蓝图/诊断日志具体字段（`mustPreserve`、Phase 4 vfredusum 提示、`MSHADOW_HALF_ROUND_TO_NEAREST` 定义位置 `half.h:36-37`）；`git diff` 与 `patch.diff` 已用 `diff` 逐字节确认一致（`DIFF_IDENTICAL`），`patchCandidateId` 取自候选文件实际值 `pc-bp-mxnet-category-activation-003`。
- **可解释性** [PASS]：根因→改动→约束逐条映射；架构判定给出「指令—扩展—硬件/构建」三列对照；`pass` 结论与 7 条 minor/suggestion 不阻断的关系明确说明。
- **内部一致性** [PASS]：markdown 与 JSON 的 `reviewResult`、finding 编号/级别、`archReview.status` 完全一致；未发现与蓝图 `constraints.mustPreserve` 冲突的声明；锚点仅使用 `+++ b/CMakeLists.txt` 与 `+++ b/src/operator/nn/softmax-inl.h` 两个真实路径。
- **安全** [PASS]：补丁无硬编码密钥、无 `system()/popen()`/命令注入面、无用户输入进入格式串、无 `reinterpret_cast` 越界（仅用于 float↔u16 的等价宽度重解释，且长度受 `vl` 约束）、无新增动态内存/裸指针所有权，构建探针仅执行编译器 flag 探测。

## 结论

补丁**通过审核**（`reviewResult: pass`，`archReview.status: passed`）。exp 范围归约 `r = x - k*ln2_hi - k*ln2_lo` 两项 ln2 均正确乘以整数 `k`，**未复现同族历史严重缺陷**；`negate` 每向量仅施加一次 `vfneg.v`，softmin = softmax(-x) 语义保持。7 条发现全部为 minor/suggestion，作为后续验证与稳健性改进建议记录，不阻断 Craft 闭环。

残余风险：(1) 本环境无 RISC-V 交叉工具链，未实测 configure/build/运行；(2) f32 无序归约与 `AType=double` 参考实现的数值容差未量化；(3) 蓝图自身 `diagnosis.blockReason` 指出 Sampling IP precision 缺失、无 ARM 对比数据，收益验证需以重建后的 `perf annotate`（应出现 `vle32/vfneg.v/vfredmax/vfredusum/向量 exp/vse16`，`fsub.s@19a6680`、`auipc@19a670a`、`beqz@19a6654` 应下降/消失）为准。
