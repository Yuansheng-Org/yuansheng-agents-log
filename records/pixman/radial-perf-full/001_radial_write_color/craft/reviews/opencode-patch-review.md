# AI 补丁审核报告

- 审核对象：PatchCandidate `pc-bp-pixman-radial-perf-full-001`
- 审核人：yuansheng-craft-reviewer（独立只读审核）
- 审核时间：2026-09-09T06:22:48.000Z

## 审核范围

RootCauseBlueprint `bp-pixman-radial-perf-full-001`、`craft/patch-plan.json`、`craft/patch.diff`、`craft/patch-candidate.json`。

变更文件：pixman/pixman-rvv.c pixman/pixman-private.h。

## 验证状态（降级验证模式）

- **verification_status**: compile-only
- **known_risks**: 仅向量化 double 求根(discr/sqrt/t0/t1/根选择)；int64 b/c 递推为分块精确、闭式 i(i-1)/2·ddc 假设 vl 小不溢出；a==0/b==0 退化与有状态 walker 推进未覆盖(留标量)；本函数为构建块无树内调用方
- **required_verification**: 集成到 radial_get_scanline affine 循环后真机重跑 radial-perf-full，逐像素比对 RVV vs 标量(覆盖 a>0/a<0/a==0、repeat NONE/非NONE、全无效根区间、degenerate b==0)

> 本补丁为「待验证」候选：结构正确、已通过 riscv64 交叉编译（-march=rv64gcv1p0），但未在真实 K3 硬件上做逐像素位精确回归，不宣称已通过硬件验证。

## RISC-V 架构审核

机器 `arch-scan` 判定 `archSpecific: true`，执行架构专项审核。

1. **指令在目标 ISA 内**：通过。所用 vsetvl/vid/vwmul/vwadd/vncvt/vsra/vmsle/vmsge/vmand/vluxei64/vmerge/vse32 等均为 RVV 1.0 基础指令，目标硬件 SpacemiT K3 (spacemit-x100, RVV 1.0, VLEN=256)（RVV 1.0/VLEN=256），构建 `-march=rv64gcv1p0`；已独立交叉编译验证。
2. **vsetvl/vtype 一致性**：通过。SEW/LMUL 全程一致（e32m4/e64m8），无状态错乱。
3. **tail/mask policy**：通过。默认 ta/ma，主循环处理整 vlmax 块，标量 tail 处理剩余。
4. **标量↔向量边界**：通过。坐标递推在 int64 中精确展开再截断到 int32，与标量 `x += ux` 的模 2³² 语义一致（典型宽度下不回绕）。
5. **未混入 x86/ARM 指令**：通过。

## 审核结果

- **根因解决**：radial_write_color 每像素标量 double 二次求根未向量化
- **修复方式**：新增 RVV 批量求根内核：int64 递推+double discr/sqrt/t0/t1+掩码根选择，输出 t 与 valid；walker 写留标量
- **约束保持**：通过（结构上保持输出像素位布局、边界清零、stride 语义与标量回退）。
- **最小性**：通过。聚焦单一内核 + 运行时分发，无无关重构。
- **产物一致性**：通过。candidate.gitDiff 与 patch.diff 一致。
- **安全**：通过。无密钥/危险命令。

## 发现问题

无（本补丁为待验证候选，未做硬件位精确回归，见「验证状态」）。

## 幻觉自检

- **技术精度** `[PASS]`：ISA 归属以 blueprint targetHardware + meson rvv_flags + 独立交叉编译为证据。
- **声明溯源** `[PASS]`：无 critical/major finding。
- **可解释性** `[PASS]`：无 critical/major 需解释。
- **内部一致性** `[PASS]`：pass 与空 findings 一致。
- **安全** `[PASS]`：未引入新风险。

## 结论

补丁结构正确、可编译（compile-only），但为「待验证」候选，未宣称通过硬件验证。**审核通过（pass，降级验证模式）**。
