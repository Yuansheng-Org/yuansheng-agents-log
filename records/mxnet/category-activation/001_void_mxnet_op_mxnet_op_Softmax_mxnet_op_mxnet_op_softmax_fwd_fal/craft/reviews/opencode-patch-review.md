# AI 补丁审核报告

## 审核范围

- 审核对象：`pc-bp-mxnet-category-activation-001`（PatchCandidate）及其 `patch.diff`
- 对应蓝图：`bp-mxnet-category-activation-001`（`.yuansheng/trace/mxnet/category-activation/001_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_softmax_fwd_fal/blueprint_mxnet_category-activation_001.json`）
- 对应计划：`pp-bp-mxnet-category-activation-001`（`craft/patch-plan.json`）
- 变更文件：`CMakeLists.txt`（+17）、`src/operator/nn/softmax-inl.h`（+148）
- 事实核对（只读）：
  - `patch.diff` 与目标仓库 `git diff` 逐字节一致（已用 `diff` 比对，输出 `PATCH.DIFF == GIT DIFF (exact)`）
  - `patch-candidate.json` 的 `gitDiff` 字段与 `patch.diff` 逐字节一致（`equal: true`，长度均 9434）
  - diff 头部 blob 索引 `53a6978f4` / `3023cd717` 与 `HEAD` 对应文件哈希一致
  - `patch.diff` 仅含项目源码（两个文件），不含 `.yuansheng/` 协议产物
- 审核方式：机器架构扫描（`arch-scan`）+ 通用质量/约束审核 + 蓝图与 diaglog 证据溯源
- 只读边界：本审核未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate

## RISC-V 架构审核

机器扫描 `arch-scan` 结果：`archSpecific: true`，命中规则 `riscv-macro` 与 `march-flag`（锚点行 `+    check_cxx_compiler_flag("-march=rv64gcv_zfhmin_zvfhmin" MXNET_RISCV_HAS_ZFHMIN)`）。因此执行 RISC-V 专项审核。

### ISA 成员资格（对照 blueprint.source.targetHardware 与 diaglog Build ISA）

- 目标硬件（blueprint.source.targetHardware / diaglog Phase 1）：`SOPHGO SG2044 (XuanTie C920v2, RVV 1.0, VLEN=128)`；`cpuinfo.isa` 含 `v`、`zve32f`、`zve32x`、`zve64f`、`zve64d`、`zfh`、`zfhmin`、`zvfh`、`zvfhmin`。
- 基线 Build ISA（diaglog Phase 1 / metadata `Tag_RISCV_arch`）：`rv64...v1p0...zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`，含 `v` 与 f32 向量能力，**不含** `zfh/zfhmin`。
- 补丁新增指令的 ISA 归属（依据上述证据，不引入模型自有知识之外的断言）：
  - `vle32/vse32`、`vfsub.vf`、`vfmul.vf/vv`、`vfmv_s_f`、`vfmv_f_s`、`vfredmax.vs`、`vfredusum.vs`、`vfcvt_x_f`、`vfcvt_f_x`、`vmfne/vmflt`、`vmerge` → 需 `Zve32f`/`Zve32x`，基线 Build ISA 已具备（`zve32f1p0_zve32x1p0`）。
  - `vfncvt.f.f.w`（`__riscv_vfncvt_f_f_w_f16mf2`）→ 需 `zvfhmin`，硬件具备（`zvfhmin`），基线 build 不具备，由补丁 CMake 追加 `zvfhmin` 补足 → 与 `recommendedFirstAction`「构建配置增补 zfhmin 以启用原生 FP16 转换」一致。
- 结论：新增指令集成员资格与 targetHardware 一致；对 zfhmin/zvfhmin 的依赖由补丁的构建参数显式补足，符合蓝图 Finding 2 的 L0 类修正方向。

### vsetvl / vtype（LMUL/SEW）一致性

- 全路径统一 `SEW=32, LMUL=1`：`vsetvl_e32m1`、`vfloat32m1_t`、`vint32m1_t`、`vbool32_t`。
- FP16 收窄结果为 `vfloat16mf2_t`（`f32m1` 收窄后 EMUL=1/2 → `mf2`），`vse16_v_u16mf2` 与之一致。
- 未出现 LMUL/SEW 混用或跨组错配。

### tail / mask 策略

- 所有 intrinsic 均未带 `_tu`/`_tamu` 后缀 → 默认 tail-agnostic；reduction/merge 只依赖第 0 元素与显式 mask，不读取未定义 tail 元素。
- `vsetvl` 每块按 `M - j` 计算 `vl`，尾部由 `vl` 截断而非依赖 mask 读取越界元素，`vse16/vse32` 存储长度受 `vl` 约束。

### scalar ↔ vector 边界

- 标量↔向量桥接仅用 `__riscv_vfmv_s_f_f32m1`（seed 入向量）与 `__riscv_vfmv_f_s_f32m1_f32`（reduction 结果出标量），逐块以标量 seed 串联，跨块 max/sum 语义正确。
- `vfredmax`/`vfredusum` 参数序为 `(vs2=v, vs1=seed, vl)`，与 RVV `vfred*.vs`（vs1[0] 为初值）约定一致。

### register-group 重叠

- LMUL=1，单寄存器组，无寄存器组重叠；reduction/转换结果 EMUL≤1，无 EMUL 放大导致的重叠风险。与 diaglog §2 预算（LMUL×peak_live≤32）无冲突。

### 运行时 / 编译期派发自一致性

- 派发为**编译期**：`#if defined(__riscv) && defined(__riscv_v)` + `if constexpr`（类型）+ 运行时 `temperature == 1.0 && sa == 1` 条件；无 hwprobe 运行时特性探测。
- 与蓝图/diaglog 证据自一致：Build ISA 含 V，故 `__riscv_v` 成立；非 RVV 构建回退到未改动的 scalar 循环。
- 未使用任何 x86/ARM 指令或 intrinsic；扫描范围内无跨架构指令混入。

### 证据缺口

- 未在本环境执行 RISC-V 工具链编译（无交叉工具链/构建目录），故 intrinsic 的**可编译性未动态验证**；`CMakeLists.txt` 的 `check_cxx_compiler_flag` 探测结果也未实测。该项按「无法验证」处理并降级为 suggestion（见 F7），不作事实断言。
- 未提供 ARM 侧对比 perf 数据与 `precise_ip`，故不评估收益幅度，仅审核机制与语义正确性。

## 审核结果

- reviewResult：**pass**
- 结论：补丁聚焦蓝图已诊断根因（rank-001 `Softmax<softmax_fwd,false,double,float,half_t,int,2>` 的未向量化 3-pass scalar 归一化 + 软件 float2half），以 `vfredmax`/向量 exp/`vfredusum`/`vfmul` 向量化主循环，并以构建参数补足 `zfhmin/zvfhmin` 启用原生 FP16 收窄；未改动公共 API/模板签名与 temperature 分支；变更范围聚焦（构建系统 + 目标源码）。未发现 critical/major 问题。
- 发现数量：8（minor ×5，suggestion ×3）。

## 发现问题

### F1（minor，numerical-semantics）float 累加替换 AType=double 累加

- file：`src/operator/nn/softmax-inl.h`
- evidence：diff 新增 `+          float sum = 0.0f;`、`+          const float inv_sum = 1.0f / sum;`；蓝图 `constraints.mustPreserve` 要求「Softmax 数值语义」；diaglog Phase 4 Finding 1 明示「本项目 AType=double 累加若需 bit-exact 或误差合同必须单独验证」。
- 说明：原标量路径 `AType sum`（本例 AType=double）逐元素累加 `std::exp`，归一化除法在 double 域；新快路径改为 float 累加 + float 除法，并将 `vfredusum`（无序）结果作为 seed。对 half_t 输出（~1e-3 相对精度）影响可忽略，对 OType=float 输出为末位 ULP 级差异；但这是相对蓝图「double 累加合同」的精度下调，需以回归测试确认容差。
- suggestion：在 category-activation 回归中显式核对 softmax 输出和/误差容差；若项目要求与 double 累加 bit-exact，则改用有序归约（`vfredosum`）或保留 double 累加路径，并在注释/文档中记录该精度取舍。

### F2（minor，numerical-semantics）向量 exp 将次正规（subnormal）结果下溢为 0

- file：`src/operator/nn/softmax-inl.h`
- evidence：diff 新增 `+  const float kExpUnderflow = -87.33654475f;  // logf(FLT_MIN)`、`+  const vbool32_t is_small = __riscv_vmflt_vf_f32m1_b32(x, kExpUnderflow, vl);`、`+  res = __riscv_vmerge_vvm_f32m1(res, __riscv_vfmv_v_f_f32m1(0.0f, vl), is_small, vl);`；蓝图 `constraints.mustPreserve` 含「溢出下溢处理」。
- 说明：阈值取 `logf(FLT_MIN)`，对 `x < -87.3365` 直接置 0；而标量 `std::exp` 在约 `[-103.97, -87.3365]` 区间会返回次正规数。差值绝对值 ≤ FLT_MIN（~1.18e-38），且稳定减 max 后 sum≥1，对归一化结果影响可忽略，但确为下溢语义的轻微偏移。
- suggestion：以极端 logits/下溢用例对拍 reference，确认次正规区差异可接受；若需严格保持标量下溢语义，可对 `is_small` 区改用渐近下溢近似或扩大保护阈值，并在报告中记录该取舍。

### F3（minor，build-portability）zvfhmin 编译期门控过宽

- file：`src/operator/nn/softmax-inl.h`
- evidence：diff 新增 `+#if defined(__riscv_zvfhmin) || defined(__riscv_zfhmin)` 与 `+    const vfloat16mf2_t h = __riscv_vfncvt_f_f_w_f16mf2(v, vl);`。
- 说明：`vfncvt.f.f.w` 及 `vfloat16mf2_t` 属**向量**半精度能力，需 `zvfhmin`；仅 `zfhmin`（标量）时 `__riscv_zfhmin` 为真但向量收窄类型/内建不可用，可能触发编译失败。当前 CMake 同时追加 `zfhmin` 与 `zvfhmin`，故默认路径不触发；但手工 `-march=rv64gcv_zfhmin`（仅标量）会命中该分支。
- suggestion：向量收窄分支仅以 `__riscv_zvfhmin` 门控（或 `__riscv_v_intrinsic` + `zvfhmin` 组合），标量 `zfhmin` 路径如需保留应使用标量 `fcvt.h.s` 而非向量内建。

### F4（minor，build-portability）`-march` 仅按编译器接受度追加，未按目标 CPU 能力判定

- file：`CMakeLists.txt`
- evidence：diff 新增 `+    check_cxx_compiler_flag("-march=rv64gcv_zfhmin_zvfhmin" MXNET_RISCV_HAS_ZFHMIN)`、`+      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=rv64gcv_zfhmin_zvfhmin")`、`+      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=rv64gcv_zfhmin_zvfhmin")`；diaglog Build ISA 显示原 build 为 `rv64...v1p0...`（无 zfh）。
- 说明：`check_cxx_compiler_flag` 只验证工具链接受该 `-march`，不验证运行目标 CPU 是否支持 `zfhmin/zvfhmin`。该改动作用于整个 riscv64 库的 C/CXX flags，若在同族但缺 `zfhmin/zvfhmin` 的 RISC-V 机器上运行，自动向量化产生的半精度指令可能触发非法指令；同时末尾追加的 `-march` 会覆盖工具链/环境中更具体（可能含 `zba/zbb/zbs/zicbom` 等）的既有 `-march`。
- suggestion：将 zfhmin/zvfhmin 作为可选 `MXNET_RISCV_*` 开关或按目标 CPU 能力/hwprobe 决定；或仅在 softmax 相关 TU 上启用并保留运行时回退，避免对全库强加扩展要求。

### F5（suggestion，performance）pass 3 重新计算向量 exp

- file：`src/operator/nn/softmax-inl.h`
- evidence：pass 3 新增 `+            vfloat32m1_t e = softmax_rvv_vexp_f32m1(v, vl);`（在 pass 2 已计算过同值 exp 后再次计算）。
- 说明：3-pass 设计避免了中间指数值的内存往返，但第三遍重算 exp 增加一次超越函数运算；与 PatchPlan 指令「pass 3 as ... vector exp -> vfmul」一致，属实现选择。
- suggestion：若寄存器/带宽允许，可将 pass 2 的指数向量缓存到临时缓冲或融合 pass 2/3，减少一次 exp；以 category-activation 的 forward 延迟实测决定是否值得。

### F6（suggestion，portability）fallback 临时缓冲假定 VLEN≤1024

- file：`src/operator/nn/softmax-inl.h`
- evidence：diff 新增 `+    float tmp[32];  // VLMAX for SEW=32/LMUL=1 is 32 at VLEN=1024`。
- 说明：`tmp[32]` 仅在无 `zfhmin/zvfhmin` 时编译（即 CMake 回退到 `rv64gcv` 的构建）；对目标 VLEN=128（VLMAX=4）安全，但若未来 VLEN>1024，`vse32` 会按 `vl` 越界写入。属潜在可移植性问题，不影响本目标。
- suggestion：以 `__riscv_vsetvlmax_e32m1()` 派生上界，或按 VLMAX 分块转换存储，避免对 VLEN 的隐式上限假设。

### F7（suggestion，verification-gap）构建与回归未在本环境实测

- file：`src/operator/nn/softmax-inl.h`
- evidence：本环境无 RISC-V 交叉工具链/构建目录，未执行编译；蓝图 `validation.regressionCommand` 为 `agent1 patch_regression --case category-activation`。
- 说明：intrinsic 拼写/类型、CMake 探测分支与 FP16 收窄路径均未经动态编译验证；该缺口不阻断补丁生成，但需在交付说明中披露。
- suggestion：在 SG2044/RVV 工具链上完成一次配置与编译，随后按蓝图执行 category-activation 回归，并重新 `perf annotate` 核对 `18dea7c/18deafe` 的 expf@plt 样本下降、出现 `vle32/vfredmax/vfredusum/vse16` 与 `fcvt.h.s`。

### F8（minor，numerical-semantics）FP16 存储路径由软件 float2half 改为硬件收窄

- file：`src/operator/nn/softmax-inl.h`
- evidence：diff 新增 `+    const vfloat16mf2_t h = __riscv_vfncvt_f_f_w_f16mf2(v, vl);`、`+    __riscv_vse16_v_u16mf2(reinterpret_cast<uint16_t*>(out),`；蓝图 `constraints.mustPreserve` 要求「FP16 输出 rounding 语义（RNE，与 MSHADOW_HALF_ROUND_TO_NEAREST 对齐）」；diaglog Finding 2 明示需验证 `fcvt.h.s` 的 frm 与 mshadow RNE 对齐。
- 说明：原路径经 `mshadow::half::half_t`（`MSHADOW_HALF_ROUND_TO_NEAREST=1`，软件 RNE）从 double 结果转换；新路径由 `vfncvt.f.f.w` 按默认 `frm=RNE` 从 float 结果收窄。舍入模式一致，但次正规/NaN/溢出边界的具体行为（如 mshadow 对超范围置 inf、对 NaN 归一化）与硬件收窄的位级结果可能有细微差别。
- suggestion：对 FP16 边界用例（`0x7c00` inf、`0x8000` ±0、`0x7e00` NaN、次正规、RNE 半值）与 reference/软件路径对拍；必要时显式设置 `frm` 或在文档中记录硬件收窄与 mshadow 语义的对应关系。

## 幻觉自检

- 技术精度：所有结论均基于 `patch.diff` 原文、`blueprint_*.json`、`diaglog_*.md`、`metadata.json` 与机器扫描输出；未对未提供的动态编译/性能结果作事实断言。 [PASS]
- 声明溯源：每条 finding 的 evidence 均逐字引用 `patch.diff` 新增行或蓝图/diaglog 具体字段；文件路径均真实存在于 diff 的 `+++ b/` 头。 [PASS]
- 可解释性：findings 均给出机制说明、影响范围与可执行建议，未使用模糊断言。 [PASS]
- 内部一致性：reviewResult 与 finding 严重度一致（无 critical/major，故 pass）；`archReview.status=passed` 与 `archReviewWarning` 为空及 `arch-scan` 的 `archSpecific:true` 相符。 [PASS]
- 安全：`patch.diff` 无硬编码密钥/凭据，无 `system()/popen/rm -rf/sudo/网络下载` 等危险命令或路径。 [PASS]

## 结论

补丁正确对应蓝图根因（未向量化 3-pass scalar softmax 的 expf 主导热点 + 软件 FP16 转换），并以与目标硬件一致的 RVV 1.0（SEW=32/LMUL=1）向量化实现和 `zfhmin/zvfhmin` 构建参数完成修复；公共 API/模板签名、temperature 分支与既有标量回退均保留。RISC-V 架构专项审核通过（ISA 成员资格、vsetvl/vtype、tail/mask、scalar↔vector 边界、寄存器组、派发一致性均自洽，无跨架构指令）。发现 8 项，均为 minor/suggestion，不构成阻断。审核结果：**pass**。残余风险：float 累加/次正规下溢/FP16 硬件收窄的精度取舍需以回归与边界用例确认；本环境未做 RISC-V 编译与 perf 实测，相关收益与可编译性待目标机验证。
