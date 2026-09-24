# RISC-V 性能分析报告 — yuv2planeX_10LE_c (rank 001, checkasm_sw_scale)

**Functions under analysis: [yuv2planeX_10LE_c]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（001-yuv2planeX_10LE_c-annotate.txt，12 samples，覆盖完整函数）
- perf stat：已提供（IPC 2.45；branch_miss_rate=2.547%；**compared_functions=0——sw_scale 无任何 RVV 对照**）
- source context：libswscale/swscale_template.c yuv2planeX（10-bit LE 变体）；libswscale/riscv/swscale.c 无 yuv2planeX 注册
- hardware：SpacemiT X100，含 v；vlenb=32

## Phase 1 — Baseline / 基线
L0 gate：模式 A。关键：10LE yuv2planeX 无 RVV 实现。

## Phase 2 — Scope / 分析边界
函数：yuv2planeX_10LE_c（swscale 10-bit 平面水平缩放+转换，C fallback）。
- hot interval：306bda–306be4（指针加载 ld + 系数 lh + mulw 乘加 + 累加）
- samples：306bdc lh 16.67%、306be4 lh 16.67%、306bda ld 8.33%

## Phase 3 — Pattern scan
Classes scanned: rows-asm.md, rows-operator-rvv.md

| Pattern | Evidence | Route conf | Impact conf |
|---|---|---|---|
| Policy-Backed Missing RISC-V Assembly Kernel（primary，命中） | 四证：①yuv2planeX DSP slot；②10LE C fallback 被 checkasm 采用（compared=0）；③RVV yuv2planeX 缺失（swscale.c 无注册）；④FFmpeg policy（input/range/rgb2rgb 已有 RVV，yuv2plane 系列未覆盖）。gate：RVV_I32 | High | Medium |

**(a) 逐字 evidence：**
- 16.67 : 306bdc: lh a7,0(a1)（系数加载）
- 16.67 : 306be4: lh a4,0(a4)（源像素加载）
- 8.33 : 306bda: ld a4,0(a5)（行指针加载）

**(b) 互斥排除：** riscv-assembly-kernel-performance-optimization：排除（热点非已存在 .S）

**(c) 双 Confidence：** route=High（四证齐全）；impact=Medium（swscale 是通用转换高频路径，12 samples）

## Phase 4 — Root-cause blueprint
1. **Root cause**：swscale yuv2planeX（10LE）的 RISC-V 向量实现不存在：libswscale/riscv/swscale.c 仅注册部分 kernel（input/range/rgb2rgb），yuv2planeX 系列（9/10/12/14-bit）全执行 C 标量乘加链。
2. **The fix**：与 yuv2planeX 族统一规划（SEW=8/16 水平缩放乘加 + clip）；init 补齐 yuv2planeX 槽。Correctness contract：像素逐位一致。
3. **Baseline facts**：hardware ISA=v；build ISA=zvl128b1p0；VLEN=256
4. **收益上界**：局部样本份额——12 samples（rank 001）
5. **三维路由**：C compiler-generated；缺失；FFmpeg SIMD/DSP policy
6. **Shape proof**：SEW=16（10-bit）；gate RVV_I32
7. **Related PRs**：1 条 URL（FFmpeg PR 23712 shape-first 实例）

## Phase 5 — Verification forecast
- 应消失：lh 加载链占比下降；RVV kernel 出现 vle16/vwmaccu/vse16
- 验证方法：checkasm sw_scale（yuv2planeX 10LE）bit-exact；K3 对比 cycles

## Phase 6 — Completion check
7/7 项 ✅。修正记录：无