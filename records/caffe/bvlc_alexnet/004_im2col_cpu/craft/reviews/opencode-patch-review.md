# AI 补丁审核报告

- reviewId: review-bp-caffe-bvlc-alexnet-004-r2
- patchCandidateId: pc-bp-caffe-bvlc-alexnet-004
- blueprintId: bp-caffe-bvlc_alexnet-004
- 审核时间: 2026-09-21T06:19:18Z
- 审核者: opencode independent read-only reviewer
- 审核模式: 只读，不修改源码与补丁产物
- 轮次: 第 2 轮（针对上一轮 F1 注释 lane 数修订后的复审）

## 审核范围

- RootCauseBlueprint: `.yuansheng/trace/caffe/bvlc_alexnet/004_im2col_cpu/blueprint_caffe_bvlc_alexnet_004.json`
- 诊断日志: `.yuansheng/trace/caffe/bvlc_alexnet/004_im2col_cpu/diaglog_caffe_bvlc_alexnet_004.md`
- Metadata / build ISA: `.yuansheng/trace/caffe/bvlc_alexnet/004_im2col_cpu/metadata.json`
- PatchPlan: `.yuansheng/craft/caffe/004_im2col_cpu/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/caffe/004_im2col_cpu/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/caffe/004_im2col_cpu/craft/patch-candidate.json`
- 现网源码（只读核对）: `src/caffe/util/im2col.cpp`

机器架构扫描（`arch-scan`）结果：`archSpecific=true`，命中 `rvv-intrinsic`（`vle32.v`）与 `riscv-macro`（`#if defined(__riscv_v)`），因此执行 RISC-V 架构专项审核。

### 通用审核结论

1. 是否解决 blueprint rootCause：是。根因为 `im2col_cpu<float>` 内层固定 stride gather（`input_col += stride_w`）被编译为全标量、每元素约 8 条指令（`addw+slli+add` 地址重建 + `flw/fsw` + `bgeu` + 归纳变量）。补丁将该内层循环抽为 `im2col_cpu_row`：float 重载按仿射区间 `[lclip, rclip)` 三段处理，头/尾 `vse32` 零填、主段 `stride_w==1` 用 `vle32.v + vse32.v`（unit-stride）、`stride_w>1` 用 `vlse32.v`（byte stride = `stride_w*4`）+ `vse32.v`。与 blueprint.diagnosis.recommendedFirstAction 及 pattern §The fix 一致。
2. 约束保持（blueprint.constraints.mustPreserve）：
   - 列布局逐位一致：`lclip = ceil(-base_col/stride_w)`、`rclip = min(output_w, floor((width-1-base_col)/stride_w)+1)` 与标量 `0 <= base_col+oc*stride_w < width` 的判定等价（`input_col` 随 `oc` 单调增 → in-bounds 为连续区间）；主段源地址 `row + base_col + lclip*stride_w` 与标量 `data_im[input_row*width + input_col]` 一致（现网源码 `src/caffe/util/im2col.cpp:93-120`）。以 AlexNet conv1（width=227, pad=0, kernel=11, stride=4, output_w=55）核对：base_col=kc∈[0,10]，lclip=0、rclip=55，全 in-bounds，与标量一致；conv2–5（stride=1, pad=2）kc=0 → base_col=-2、lclip=2，与标量首个 in-bounds 列一致。
   - bit-preserving：向量路径仅 32-bit 加载/存储，无 FP 算术/归约重排，NaN payload / signed zero 原样搬运。
   - padding 精确 `+0.0f`：零填用 `__riscv_vfmv_v_f_f32m4(0.0f, vl)` + `vse32`，与标量 `*(data_col++) = 0;`（`+0.0f`）位型一致。
   - 函数签名与调用约定不变：`im2col_cpu` 12 参数模板与 `im2col_cpu<float>/<double>` 显式实例化（`src/caffe/util/im2col.cpp:157-167`）未改动；整行越界 `input_row` 的零填路径保持原样。
   - 非 RVV 标量 fallback：float 专用重载仅在 `#if defined(__riscv_v)` 内定义；非 RVV 构建下 float 回落到泛型标量模板，行为与原文一致。
3. 改动范围：仅 `src/caffe/util/im2col.cpp`，无无关重构/格式化噪声；`#include <cstddef>` 为 `ptrdiff_t/size_t` 所需。
4. 产物一致性：`PatchCandidate.gitDiff` 与 `patch.diff` 逐字节相同（Node `Buffer.equals` 为 `true`，长度均 5375 字节）；`changedFiles == ["src/caffe/util/im2col.cpp"]`，与 diff 中唯一 `+++ b/src/caffe/util/im2col.cpp` 一致；`git status` 显示该文件为已修改（工作树与 diff 一致）。
5. 上一轮 F1（minor，注释 lane 数）：已修复。现 patch.diff 第 58 行为 `// LMUL * VLEN / 32 float lanes per iteration (32 lanes at VLEN=256).`，与 VLEN=256、e32、m4 的 VLMAX=32 自洽；全文已无 “16 lane” 残留。
6. 安全性：无密钥、无危险命令、无外部网络/系统调用；补丁为纯计算路径改动。

## RISC-V 架构审核

- 目标 ISA：SpacemiT X100 (K3), RVV 1.0, VLEN=256, vlenb=32（blueprint.source.targetHardware；diaglog Phase 1）。构建侧证据：metadata `binaries.caffe-elf-A` 的 `Tag_RISCV_arch` 含 `v1p0` 与 `zvl128b1p0/zvl64b1p0/zvl32b1p0`，与 RVV 1.0 一致。注意：frozen `metadata.json` 的 `vector` / `cpuinfo` 为空对象，VLEN=256/vlenb=32 无法由 metadata 二次确认，仅见于 blueprint 与 diaglog（cannot verify → 不影响补丁正确性判定）。
- 指令在目标 ISA 内：`vle32.v` / `vse32.v` / `vlse32.v` 与 `vsetvl` 均为 RVV 1.0 基础指令；无 x86/ARM 指令混入（arch-scan 仅命中 RVV intrinsic 与 RISC-V 宏）。
- SEW/LMUL 与数据流一致：float 为 32-bit，使用 `e32`（`__riscv_vsetvl_e32m4`）匹配；LMUL=m4 合法候选，kernel live vector 仅一组（`vfloat32m4_t v`），无寄存器组重叠。
- vsetvl/vtype 一致性：三处循环均使用动态 `__riscv_vsetvl_e32m4(remaining)`，未硬编码 lane 数，VLEN-agnostic；SEW/LMUL 在整段保持一致（e32/m4），无 vtype 混用。
- tail/mask 策略：`vse32` 仅写 `vl` 个元素，不依赖 tail-agnostic 语义；零填 `vfmv` 与 `vse32` 使用同一 `vl`，尾部元素不会被误写；标量↔向量边界由 `lclip/rclip` 精确切分，无重叠/漏写。
- 编译期 gate + 重载分派自洽性：`#if defined(__riscv_v)` gate + 非模板 float 重载优先于泛型模板（float→RVV，double/其它→标量模板），与 blueprint 建议一致。无法由既有证据确认构建编译器是否定义 `__riscv_v`（metadata 未含编译器版本/构建参数；部分工具链只定义 `__riscv_v_intrinsic`）→ 标记 cannot verify，降级为 suggestion（F1），不阻塞。
- 未发现架构级正确性缺陷；架构审核结论为 passed。

## 审核结果

- reviewResult: pass
- archReview.status: passed
- 阻塞问题: 无（无 critical / major）
- 非阻塞问题: 3 个 suggestion（F1 架构宏可验证性 cannot verify、F2 区间裁剪前置条件、F3 编译期 gate 与约束措辞）

## 发现问题

| id | severity | category | 摘要 |
|---|---|---|---|
| F1 | suggestion | arch-portability | `__riscv_v` 编译期 gate 是否被该构建编译器定义无法由冻结证据确认（cannot verify），部分工具链仅定义 `__riscv_v_intrinsic` |
| F2 | suggestion | edge-case-robustness | 区间裁剪 `lclip/rclip` 的整除隐含 `stride_w > 0` 与 `width >= 0`；非法输入下与标量 unsigned 比较语义不同 |
| F3 | suggestion | constraint-wording | blueprint 约束写「不支持 v 的运行环境保留标量 fallback」，补丁仅做编译期 gate，无运行时 hwprobe/ifunc 降级 |

详情：

- F1（suggestion，cannot verify）：`patch.diff` 原文 `+#if defined(__riscv_v)`；blueprint.diagnosis.recommendedFirstAction 原文「以 `__riscv_v` / `__riscv_v_intrinsic` 宏做 RVV build gate 并保留标量 fallback」。冻结 metadata 未提供编译器版本/构建参数，无法确认 `__riscv_v` 一定被定义；该点无法由既有证据验证，故降级为 suggestion。建议可同时 gate `#if defined(__riscv_v) || defined(__riscv_v_intrinsic)` 或记录构建编译器版本以增强跨工具链稳健性。
- F2（suggestion）：`patch.diff` 原文 `+    lclip = (-base_col + stride_w - 1) / stride_w;` 与 `+    const int r = last_in / stride_w + 1;` 依赖 `stride_w > 0`（否则除零）与 `width >= 0`；原标量 `is_a_ge_zero_and_a_lt_b` 注释声明 b 始终为正，caffe im2col 调用合同亦保证 stride 为正，故对合法输入无语义差异。可作为防御性 `DCHECK_GT(stride_w, 0)` 或前置条件注释。
- F3（suggestion）：`patch.diff` 原文 `+#if defined(__riscv_v)` / `+#endif  // defined(__riscv_v)`；blueprint.constraints.mustPreserve 第 4 条原文「非 RVV 构建 / 不支持 v 的运行环境保留原标量 fallback 行为」。补丁以编译期 gate 覆盖「非 RVV 构建」，未提供无 V 硬件上的运行时降级；这与 recommendedFirstAction 指定的 build gate 一致，且目标 object 的 `Tag_RISCV_arch` 已含 `v1p0`，不构成缺陷。若未来需单二进制覆盖无 V 硬件，应另加 hwprobe/ifunc 运行时 dispatch。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| technical precision | PASS | 结论均基于 diff 原文与 blueprint/diaglog/metadata 字段；RVV 1.0 与 VLEN 引用蓝图/metadata 的 `Tag_RISCV_arch`，未凭模型知识断言硬件能力。 |
| claim traceability | PASS | 每条 finding 的 evidence 均引用 patch.diff hunk 原文或 blueprint 具体字段；findings 未给 line，避免锚点越界。 |
| explainability | PASS | 区间等价性、bit-preserving、零填 `+0.0f`、SEW/LMUL、gate/fallback 均给出可复核推理与 diff/源码位置。 |
| internal consistency | PASS | `gitDiff == patch.diff` 逐字节相同、`changedFiles` 与 diff 唯一文件一致；reviewResult 与 findings 严重度一致（无 critical/major 故 pass）；上一轮 F1 已复核修复。 |
| safety | PASS | 无密钥、无危险命令、无网络/系统副作用；补丁为纯计算路径改动。 |

## 结论

补丁正确且聚焦地解决了 blueprint 根因：将 `im2col_cpu<float>` 的固定 stride 标量 gather 改写为 RVV 1.0 向量访存（`vle32/vlse32 + vse32`），动态 `vsetvl`、SEW=e32、LMUL=m4，并保留非 RVV 标量 fallback；列布局逐位一致、bit-preserving、`+0.0f` 零填与签名/实例化约束均满足，产物一致性通过。上一轮 minor（注释 lane 数）已修复。架构专项审核通过。仅余 3 个非阻塞 suggestion，不构成阻塞。

最终审核：**pass**。
