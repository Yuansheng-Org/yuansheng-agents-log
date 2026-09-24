# AI 补丁审核报告

- reviewId: `pr-caffe-bvlc-alexnet-006-20260921`
- patchCandidateId: `pc-bp-caffe-bvlc-alexnet-006`
- 审核对象: blueprint `bp-caffe-bvlc_alexnet-006`（caffe `PoolingLayer<float>::Forward_cpu` MAX 池化）
- 审核模式: 独立只读审核（未修改任何源码或补丁产物）
- 审核结论: **pass**

## 审核范围

审核读取的冻结产物：

- `craft/patch-plan.json`（patchPlanId `pp-bp-caffe-bvlc-alexnet-006`）
- `craft/patch.diff`（唯一改动文件 `src/caffe/layers/pooling_layer.cpp`）
- `craft/patch-candidate.json`（patchCandidateId `pc-bp-caffe-bvlc-alexnet-006`）
- `.yuansheng/trace/caffe/bvlc_alexnet/006_caffe_PoolingLayer_float_Forward_cpu/blueprint_caffe_bvlc_alexnet_006.json`
- `.yuansheng/trace/caffe/bvlc_alexnet/006_caffe_PoolingLayer_float_Forward_cpu/diaglog_caffe_bvlc_alexnet_006.md`
- `.yuansheng/trace/caffe/bvlc_alexnet/006_caffe_PoolingLayer_float_Forward_cpu/metadata.json`
- 当前 `src/caffe/layers/pooling_layer.cpp`（只读对照，未编辑）

**根因契合度**：blueprint `rootCause.summary` 指出 MAX 池化以标量逐元素执行固定 stencil 邻域归约（每 tap `flw+flt.s+条件 fsw/sw`，每轮 `mulw` 重建 `h*width_+w`），`candidateFiles` 唯一为 `src/caffe/layers/pooling_layer.cpp`。补丁在同一文件 MAX 分支引入 RVV interior fast path（`vlse32` 跨步 gather + `vmfgt.vv`/`vmerge.vvm` 同步维护最大值与索引），与根因和 `recommendedFirstAction` 一致。

**mustPreserve 逐项核对**：

- MAX/AVE 数值输出与首胜 tie（严格大于才更新）：`vmfgt` 为严格大于，`vmerge` 仅在 tap > vmax 时替换，tap 迭代顺序 `h` 外 `kw` 内与标量 `h` 外 `w` 内一致 → 首个（行主序）最大值获胜。PASS。
- NCHW 布局与 `offset(0,1)` 通道步进：补丁未触碰 `bottom_data/top_data/top_mask/mask += offset(0,1)` 的循环（`Forward_cpu` 调用处保留原步进）。PASS。
- `use_top_mask`/`max_idx_` 选择与公共 API：`top_mask != NULL` 分支写 `Dtype`，否则写 `int`；函数签名与层接口未改。PASS。
- NaN 语义：`vmfgt` 对 NaN 返回假，vmax/vindex 不更新，与标量 `>` 对 NaN 为假一致；vmax 初值 `-FLT_MAX` 非 NaN，故永不引入 NaN。PASS。
- AVE divisor/rounding 合同：AVE 分支（`pool_size` 与 `/ pool_size`）整段未改。PASS。

**语义等价论证核对**（interior 检测 / 每 lane 索引 / mask / 初值 / border-tail 回退）：

- interior 行：`row_interior = (hstart_raw >= 0) && (hstart_raw + kernel_h <= height)`，此时 `hstart == hstart_raw` 且 `hend == hstart + kernel_h`，与标量 `hend = min(hstart_raw + kernel_h, height)` 相同。
- interior 列：`pwl = ceil(pad_w/stride_w)`（`pad_w==0` 时为 0），`pwr = (width + pad_w - kernel_w)/stride_w + 1`（`limit<0` 时为 0），均 clamp 到 `pooled_width` 且 `pwl <= pwr`。对 `pw∈[pwl,pwr)` 有 `wstart = pw*stride_w - pad_w ∈ [0, width-kernel_w]`，窗口完全落在 `[0,width)`。
- 每 lane 索引：`wstart0 = pw0*stride_w - pad_w`，`lane_off = lane*stride_w`，`cand = lane_off + h*width + wstart0 + kw = h*width + w`（lane j 对应 `w = wstart0 + j*stride_w + kw`），与标量 `index = h*width + w` 一致；tap 地址 `bottom_data + h*width + wstart0 + kw` 以 `byte_stride = stride_w*4` 跨步 gather，恰为同 lane 的 tap。
- 初值：`vmax = -FLT_MAX`、`vindex = -1`，与 `caffe_set(top_count, Dtype(-FLT_MAX), top_data)` 和 `caffe_set(top_count, -1, mask/top_mask)` 一致；未更新 lane 写回 `-FLT_MAX` / `-1`，与标量保持同值。
- border/tail：行 border（`row_interior==false`）与列 prefix `[0,pwl)`、suffix `[pwr,pooled_width)` 全部走同一 `max_pool_scalar_range`（原逐像素循环抽出的等价实现）；行内前缀/后缀列与向量列互不重叠，各自读到的 `top_data[pool_index]` 仍为 `caffe_set` 的 `-FLT_MAX`。
- 边界：`n==0` 时 `if (n>0)` 跳过向量路径，不会以 0 长度 `vsetvl` 误写。

**范围合理性**：仅改 `src/caffe/layers/pooling_layer.cpp`，新增 `__riscv_v` 保护的 float 重载与泛型标量模板，AVE 分支与公共 API 未动，聚焦根因。

**产物一致性**：`patch-candidate.gitDiff` 与 `patch.diff` 逐字节相等（8488 bytes，已用脚本比对 `byte-equal: true`）；`changedFiles == ["src/caffe/layers/pooling_layer.cpp"]`，与 diff 中唯一 `+++ b/` 路径一致。

**安全性**：无新增 I/O、无外部输入解析、无内存越界（tap 索引均在行内 `[0,width)`、`h∈[0,height)`）；整数 `index`/`cand` 在 `int32` 范围；无密钥/日志泄露。

## RISC-V 架构审核

`arch-scan` 结果：`archSpecific: true`（命中规则 `rvv-intrinsic`、`riscv-macro`）。目标：SpacemiT X100 / K3，RVV 1.0，VLEN=256，`vlenb=32`（metadata `Tag_RISCV_arch` 含 `v1p0`、`zve*`、`zvl128b1p0`；diaglog Phase 1 记录 vlenb=32）。

- **vsetvl/vtype 一致性**：全段使用动态 `__riscv_vsetvl_e32m4(n-i)`，后续 intrinsic 后缀统一 `_e32m4`（`vfmv_v_f_f32m4`、`vmv_v_x_i32m4`、`vid_v_u32m4`、`vmul_vx_u32m4`、`vlse32_v_f32m4`、`vmfgt_vv_f32m4_b8`、`vmerge_vvm_f32m4`、`vadd_vx_i32m4`、`vse32_v_f32m4`/`vse32_v_i32m4`、`vfcvt_f_x_v_f32m4`），SEW=e32、LMUL=m4 自洽；尾长度由 `vsetvl` 裁剪，`i += vl` 前进，尾部无越界。PASS。
- **mask 类型**：SEW/LMUL=32/4=8 → `vbool8_t`，`__riscv_vmfgt_vv_f32m4_b8` 返回类型正确。PASS。
- **标量↔向量边界**：interior 列向量化，prefix/suffix/行 border 走标量；跨步 `byte_stride = stride_w*4` 与 `vlse32` 契约一致。PASS。
- **编译期 gate + 重载分派自洽**：`#if defined(__riscv_v)` 包裹 `<riscv_vector.h>` 与 float 非模板重载；`float*` 调用优先匹配非模板（RVV），`double` 及其它 Dtype 走泛型模板（标量）；无 RVV 时仅泛型模板。build ISA 含 `v1p0`，`__riscv_v` 在本构建下成立。PASS。
- **无 x86/ARM 指令**：diff 无任何 x86/ARM 专属指令或头文件。PASS。
- **寄存器组/spill**：补丁同时持有 `vmax/vindex/tap/lane/lane_off/cand` 及掩码，e32m4 下约 5 个 4 寄存器组；**artifact 中无编译产物或反汇编，无法核实是否发生 spill** → 降级为 suggestion（见 S2），不影响通过结论。

## 审核结果

- reviewResult: **pass**
- archReview.status: **passed**
- 通用质量：根因契合、mustPreserve 全部保留、语义等价论证成立、范围聚焦、产物一致、安全。
- RISC-V 架构：vsetvl/vtype/mask 自洽，gate 与重载分派自洽，无跨架构指令；仅 spill 一事无法从现有证据核实，已降级为 suggestion。

## 发现问题

| id | 严重度 | 类别 | 摘要 |
|---|---|---|---|
| S1 | suggestion | RISC-V 向量化调优 | e32m4 在 VLEN=256 下 32 lane，而 AlexNet interior 长度仅 27/13/6，短向量下 m4 利用率低，可评估 m1/m2 |
| S2 | suggestion | RISC-V 寄存器压力 | 无编译产物/反汇编，无法核实是否发生 vector spill，建议反汇编确认 |

均无 critical/major；两个 suggestion 的 `evidence` 均引用 patch.diff hunk 原文，属提示性、不阻断通过。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 证据锚点真实性 | PASS | 全部结论引用 patch.diff hunk 原文或 blueprint/diaglog/metadata 具体字段；arch-scan 结果由工具机器产出 |
| 无虚构文件/行号 | PASS | 仅引用 diff 中唯一文件 `src/caffe/layers/pooling_layer.cpp`；suggestion 未给出行号，无越界锚点 |
| 语义等价推断有据 | PASS | interior 检测、lane 索引、严格大于/首胜 tie、NaN、初值、border/tail 回退均逐项由 diff 源码与 blueprint 约束推导，未凭模型知识断言 |
| 架构结论基于提供证据 | PASS | 目标 VLEN/vlenb/ISA 均来自 metadata `Tag_RISCV_arch` 与 diaglog Phase 1；无法核实项（spill）明确标注并降级 |
| 只读与产物边界 | PASS | 仅读取冻结产物与源码，仅写入两份 review 文件，未修改源码或补丁产物 |

## 结论

补丁正确解决了 blueprint 根因（MAX 池化标量固定邻域归约未向量化），严格保留 MAX/AVE 输出、首胜 tie、NCHW 布局、`use_top_mask`/`max_idx_` 选择、公共 API、NaN 语义与 AVE divisor 合同；RVV 路径的 interior 检测、per-lane 索引、`vmfgt` 严格大于与 `vmerge` 双更新、初值与 border/tail 回退均与标量语义等价。RISC-V 架构专项审核通过（spill 一项无法从现有证据核实，已降级为 suggestion）。产物一致、范围聚焦、安全。**审核通过（pass）**。
