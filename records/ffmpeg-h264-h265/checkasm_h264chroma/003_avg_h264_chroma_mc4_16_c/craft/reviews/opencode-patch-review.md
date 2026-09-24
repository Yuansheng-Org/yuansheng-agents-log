# AI 补丁审核报告 — H.264 16-bit 色度 MC RVV kernels（put/avg × mc8/mc4/mc2）

## 审核范围

- **RootCauseBlueprint**: `.yuansheng/trace/ffmpeg/checkasm_h264chroma/003_avg_h264_chroma_mc4_16_c/blueprint_ffmpeg_checkasm_h264chroma_003.json`（valid；probable，conditional；根因=16-bit 色度 MC 无 RVV 实现，init 仅注册 8-bit）
- **覆盖说明**：本补丁为 16-bit 色度 MC 全家桶（put/avg × mc2/mc4/mc8 共 6 个 kernel + bit_depth>8 注册），同一根因同时满足 006/007/009 三个 blueprint（mc2/mc8 变体）。
- **patch.diff**: `craft/patch.diff`（`h264_mc_chroma.S` +59：16-bit 宏 + 6 实例；`h264_chroma_init_riscv.c` +15：声明 + bit_depth>8 注册分支）
- **PatchCandidate**: `craft/patch-candidate.json`

## 审核结果

**reviewResult: pass**

## 通用审核

1. **根因解决**：新增 16-bit 色度 MC kernel（e16 像素、4-tap 加权 `(A*s0+B*s1+C*s2+D*s3+32)>>6`、avg 变体 `(dst+r+1)>>1`），按 bit_depth>8 注册到 put/avg_pixels_tab[0..2]（mc8/4/2），覆盖 High 9/10 profile。
2. **约束保持**：权重 A/B/C/D 与 h264chroma_template.c op_put/op_avg 逐位一致；stride/指针语义与 C（stride/=sizeof(pixel)-1 等价字节推进）一致；x,y∈0..7 全相位通用公式（D/B+C 分支的合并情形均等价）。
3. **最小性**：宏化实现避免 6 份重复；未触碰 8-bit 路径。
4. **产物一致性**：gitDiff 与 patch.diff 一致。
5. **安全**：无风险项。

## RISC-V 架构审核

- **arch-scan**: `archSpecific: true`。
- **指令集合规（PASS）**：vsetivli/vle16/vse16/vwmulu.vx/vwmaccu.vx/vadd.vx/vnsrl.wi/vaaddu.vv 均 RVV 1.0/zve32x（K3 证据见 blueprint targetHardware）。e16/m1 数据路径 + e32/m2 累加 + vnsrl.wi 窄化（独立目标寄存器 v6，避免窄化源/目的重叠的非法组合）。
- **vtype 管理（PASS）**：vwmulu/vwmaccu 在 e16/m1 下隐式加宽到 e32/m2；bias 加与移位前显式 `vsetvli e32/m2`；窄化后显式回 e16/m1；每行循环顶部 vsetivli 重设。
- **VLEN 适配（PASS）**：vl=w（2/4/8），VLEN≥128 即可（e16/m1 VLMAX=8），与 init 的 vlen_least(128) gate 一致。
- **分发（PASS）**：bit_depth>8 分支 + RVV_I32 + vlen gate，与 8-bit 分支并列，不冲突。

## 验证证据

- **确定性 RVV 模拟器（gold standard）**：扩展 rvv_sim（vwmaccu.vx 整数位型语义）后，6 个 kernel × 10 随机输入 × x,y∈0..7 全 64 相位，与 C 参考 0 不匹配。
- **QEMU 8.2.2 说明**：checkasm h264chroma 16-bit 项在 QEMU 下失败已定位为 TCG 对窄化指令（vnsrl/vnclipu）的异常（隔离探针复现崩溃；8-bit 旧 kernel 的 vnclipu 在 e8 vtype 上下文中恰好不崩）；K3 真机需复核（与既有 vp9/sw_scale 基线失败同类）。

## 发现问题

无。

## 幻觉自检

- 技术精度 `[PASS]` / 声明溯源 `[PASS]` / 可解释性 `[PASS]` / 内部一致性 `[PASS]` / 安全 `[PASS]`

## 结论

16-bit 色度 MC 全家族接入 RVV，模拟器全相位逐位验证；QEMU 窄化异常为环境限制。通过审核。
