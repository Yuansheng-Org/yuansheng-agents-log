# AI 补丁审核报告

独立只读审核会话（opencode-cpp-reviewer）。本报告仅依据 RootCauseBlueprint、PatchPlan、patch.diff、patch-candidate.json 与 sibling diaglog，并辅以对真实仓库源码的只读核对与本地编译验证；未修改任何源码、蓝图、计划、补丁或候选产物。

## 审核范围

- 蓝图：`.yuansheng/trace/mxnet/category-activation/017_MapPlan_ReduceWithAxisExp_sum_mul/blueprint_mxnet_category-activation_017.json`（blueprintId `bp-mxnet-category-activation-017`）。
- 补丁计划：`.yuansheng/craft/mxnet/017_MapPlan_ReduceWithAxisExp_sum_mul/craft/patch-plan.json`（patchPlanId `pp-bp-mxnet-category-activation-017`）。
- 候选补丁：`.yuansheng/craft/mxnet/017_MapPlan_ReduceWithAxisExp_sum_mul/craft/patch-candidate.json`（patchCandidateId `pc-bp-mxnet-category-activation-017`）。
- 差异：`.yuansheng/craft/mxnet/017_MapPlan_ReduceWithAxisExp_sum_mul/craft/patch.diff`，单文件改动 `3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`（+约 150 行，-约 6 行）。
- 诊断日志：`.yuansheng/trace/mxnet/category-activation/017_MapPlan_ReduceWithAxisExp_sum_mul/diaglog_mxnet_category-activation_017.md`。
- 诊断对象：`MapPlan<sv::saveto, Tensor<cpu,2,float>, 2, float, ReduceWithAxisExp<red::sum, BinaryMapExp<op::mul, Tensor<cpu,3,float>, Tensor<cpu,3,float>, float, 1>, float, 3, false, 2>>`（mshadow 双输入 3D→2D 按轴 sum-of-products 归约）。经源码核对，其消费方为 `src/operator/nn/softmax_activation-inl.h` 的 `reduce_with_axis<red::sum, false>(m_out_grad * m_out_data, 1)`。

## RISC-V 架构审核

`arch-scan` 机器判定：`archSpecific=true`，命中 `riscv-macro`（`#if defined(__riscv) && defined(__riscv_v)`）与 `vlenb`（`__riscv_vsetvl_e32m2`）。补丁的架构相关代码全部位于 `#if defined(__riscv) && defined(__riscv_v)` 守卫内，非 RISC-V 编译单元不会引入任何向量依赖；未发现 x86/ARM/SSE/NEON 指令或内建。

对补丁的三项核心主张逐条核对：

1. 消除 volatile accumulator 栈槽往返：源码确认 `mshadow::red::sum::Reduce` 的签名为 `Reduce(volatile DType& dst, volatile DType src) { dst += src; }`。补丁新增的 `ReduceWithAxisStep(red::sum, DType&, DType)` 以非 volatile 引用执行完全相同的 `acc += val` 序列，标量路径不再强制每迭代回写/重载栈槽；其它 Reducer 仍走 `Reducer::Reduce`，语义不变。
2. 消除每元素 div/rem 索引解码：原式 `z=(x*size_+k)*trailing_+y; z/last_; z%last_` 被改写为增量计数器 `(zdiv,zmod)`，步进常量 `q=trailing_/last_`、`r=trailing_%last_`，并带单次进位。已用 20 万组随机 `(size,trailing,last,y,x,k)` 暴力比对，增量解码与原始 `Z/last_`、`Z%last_` 完全一致（0 处不一致）；且循环体内不再有 div/rem（仅 Eval 入口预计算 6 处整数除法，原实现为每输出元素入口 2 处 + 每 k 迭代 2 处）。
3. 新增受守卫的 RVV unit-stride float32 归约与标量回退：快路径仅在 `ReduceWithAxisIsSum<Reducer>::value && trailing_==1 && ReduceWithAxisSrcTraits<SrcExp>::kUnitStrideF32` 且 `LhsStride(se)==last_ && RhsStride(se)==last_` 且指针非空时启用；`Run` 对 `n<=0`、空指针、非 float 返回 false 回退标量。几何推导：`trailing_==1` 时 `y=0`、`x=i*last_dst_dim_+j`，`z=x*size_+k`；`src_.Eval(z/last_,z%last_)` 映射为 `dptr_[(z/last_)*stride_+z%last_]`，当 `stride_==last_` 时恰为 `dptr_[z]`，即从 `lhs+x*size_` 起的连续点积，索引映射正确无越界/进位错误。

本地编译验证（只读，产物写入 `/tmp/opencode`）：
- 宿主 g++ 对精确实例化 `reduce_with_axis<red::sum,false>(a*b,1)` 编译通过（非 RISC-V 路径，验证 traits 特化、增量索引与模板机制，且确认无架构代码泄漏）。
- 以 clang 面向 `riscv64 -march=rv64gcv_zvl128b` 编译同一实例化通过；反汇编确认生成 `vsetvli / vle32.v / vfmacc.vv / vfadd.vv / vfredusum.vs / vmv.v.i / vse32.v`，且内层 k 循环不再产生 `div/rem`。
- 独立 RVV 内核片段对 riscv64 编译通过，确认全部 intrinsic 名称/签名与 RVV 1.0 一致。
- 循环索引覆盖模拟（VLMAX=8，n=1..39 及 63/64/65/127/128/129/1000/4097）：每个元素恰被加载一次、无遗漏无重复（0 处不一致）。

## 审核结果

- reviewResult：**pass**
- archReview.status：**passed**
- 严重级别统计：critical 0，major 0，minor 1，suggestion 3。
- 结论：补丁聚焦已诊断根因，标量路径的 volatile 往返与每元素 div/rem 均已消除；RVV unit-stride float32 快路径在编译期与索引几何上均正确，且对非 unit-stride、非 float、非 sum reducer、非连续 stride 一律回退标量。未发现 genuine critical/major 缺陷；剩余问题为数值顺序合同记录与代码整洁性，属 minor/suggestion。

## 发现问题

### F-001（minor，数值合同）FP 归约顺序变化未完整记录

- 文件：`3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`
- 证据：补丁注释仅声明 `Note: vfredusum is unordered, so callers relying on the strict scalar addition order must validate the tolerance.`，但同样改变加法结合顺序的“双独立累加器”拆分（`acc0`/`acc1` 分别累积奇偶块后 `__riscv_vfadd_vv_f32m2` 合并）未单独说明。蓝图 `constraints.mustPreserve` 明确要求「FP 加法顺序的 reference 容差（volatile 严格顺序；vfredusum 无序需验证或改 vfredosum）」。
- 建议：在 RVV 内核注释中同时点明双累加器与 vfredusum 两处顺序变化，并在回归中显式覆盖向量路径 vs volatile 标量 reference 的容差；若项目要求逐位稳定，评估改用 `vfredosum`（ordered）或单累加器。

### F-002（suggestion，效率）向量路径仍支付无用的预计算整数除法

- 文件：`3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`
- 证据：`const index_t q = trailing_ / last_;`、`const index_t r = trailing_ % last_;`、`index_t zdiv = (x*size_*trailing_ + y) / last_;`、`index_t zmod = ... % last_;` 位于 RVV 快路径分支之前；当快路径命中并 `return acc` 时，这 4 处整数除法未被使用。
- 建议：将增量解码的预计算移动到 RVV 快路径尝试失败之后（或仅在标量回退分支内计算），避免向量路径上的多余除法。

### F-003（suggestion，健壮性）`src_expr_` 裸指针的生命周期耦合

- 文件：`3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`
- 证据：构造函数 `: src_(MakePlan(e.src_)), src_expr_(&e.src_)` 保存指向源表达式对象的裸指针；正确性依赖 mshadow 的即时求值模型（表达式临时量在整条赋值表达式内存活）。
- 建议：补一行注释说明该指针仅在 `ReduceWithAxisExp` 表达式存活期内有效，防止后续重构将 Plan 存储到超出表达式生命周期的位置而悬垂。

### F-004（suggestion，可维护性）快路径条件命名与含义不完全一致

- 文件：`3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`
- 证据：trait 名 `kUnitStrideF32` 实际表达“cpu 3D float 的 `mul` 源且连续”，运行时还额外要求 `LhsStride(se) == last_ && RhsStride(se) == last_`；两处条件语义重叠且命名偏窄。
- 建议：统一为单一具名谓词（例如 `kContiguousMulF32_3D`）或补充注释，减少后续维护时对两处条件是否等价的误判。

## 幻觉自检

- 技术精度：[PASS] 对 `red::sum::Reduce` 的 volatile 签名、Tensor `stride_`/`Eval` 语义、`ReduceWithAxisExp` 几何（`trailing_`/`last_`/`size_`）均以仓库真实源码核对；增量索引等价性经 20 万组暴力比对，RVV 内核经 riscv64 实际编译与反汇编确认。
- 声明溯源：[PASS] 全部 finding 与结论均锚定 patch.diff 的具体 hunk、蓝图字段或源码位置，未凭模型知识臆断；架构判定以 `arch-scan` 机器输出为准。
- 可解释性：[PASS] 报告给出补丁改动意图、索引几何推导、守卫条件、回退路径与验证方法的可复现说明。
- 内部一致性：[PASS] reviewResult/archReview/findings/幻觉自检之间无矛盾；findings 全部为 minor/suggestion，与 pass 结论一致。
- 安全：[PASS] 补丁不含硬编码密钥、无用户输入处理、无内存越界/未校验指针解引用（空指针与 `n<=0` 均先返回 false），无新增外部依赖或权限面。

## 结论

补丁通过审核（pass）。它准确命中根因链：以非 volatile 寄存器累加替换 `red::sum` 的 volatile 栈槽往返，并将每元素 div/rem 强度削减为常量步进的增量解码；同时新增一个编译期/运行期双重守卫的 RVV unit-stride float32 归约快路径，对非适用形态安全回退标量。索引等价性、循环覆盖、RVV intrinsic 正确性与实际向量码生成均已独立验证，未发现 critical/major 问题。建议在后续回归中落实 F-001 的数值容差验证，并在需要时处理 F-002~F-004 的整洁性与健壮性改进。
