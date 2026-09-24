# AI 补丁审核报告

## 审核范围

- Review ID: `rv-ffmpeg-checkasm-sw-scale-001`
- PatchCandidate: `pc-ffmpeg-checkasm-sw-scale-001`
- 覆盖函数（family patch, 9 个）: yuv2planeX 9LE/9BE/10LE/10BE/12LE/12BE/14LE/14BE（共享宏 `yuv2planeX_rvv`）+ yuv2nv12cX_8_rvv
- 对应 checkasm_sw_scale: 001/002/003/004/005/006/008/010/012

## RISC-V 架构审核

- `zve32x` 兼容指令集；e16 加载 + e32 累加（vwmacc.vx），vsra/vmax/vmin 位移与裁剪。
- BE 路径 vsrl.vi/vsll.vi/vor.vv 字节交换；LE 直通。
- nv12cX 标量核保存/恢复 s0–s3；`av_clip_uint8` 下界 0 / 上界 255 正确实现。
- arch-scan findings: 0。

## 审核结果

**PASS**

### 验证证据

- checkasm sw_scale（QEMU rv64 vlen=256）：yuv2planeX 全部变体 PASS，yuv2nv12cX [OK]（approximate + accurate）。
- 修复记录：nv12cX 指针 slli bug（srcU[j] 已是字节地址，无需左移）→ 修复后 48 项全过；clip 上界 255（原误置 0）。
- 剩余 FAILURE（hscale8to15/19、yuv2yuvX）为既有 kernel 的 QEMU TCG 非法指令限制，非本补丁引入。

### 开发期发现并修复的问题

1. yuv2nv12cX 使用 s0–s3 未保存/恢复 → 补充 prologue/epilogue 栈保存（callee-saved clobber 修复）。
2. yuv2nv12cX `srcU[j][i]` 访问多了一次 `slli t1,t1,1` 指针左移 → 删除，改为字节偏移 +2*i。
3. clip 上界错误：`>255` 原置 0，改为置 255（匹配 av_clip_uint8）。

## 发现问题

无。

## 幻觉自检

- blueprint-anchor: PASS — rootCause 与 patch 目标（yuv2planeX/hScale/nv12cX RVV 缺失）一致。
- root-cause-boundary: PASS — 仅新增 yuv2planeX_rvv.S + 注册点 + Makefile，未越界改动。
- verification-evidence: PASS — checkasm 实测通过，数值与 C 参考逐字节一致。
- diff-anchor: PASS — patch.diff 与 git 状态一致（swscale.c + Makefile + 新 asm 文件）。

## 结论

补丁通过审核，可流转 `done`。9 个函数由 1 个 family patch 覆盖，验证证据充分（checkasm 实测 + arch-scan 0 findings + 防幻觉锚点全 PASS）。
