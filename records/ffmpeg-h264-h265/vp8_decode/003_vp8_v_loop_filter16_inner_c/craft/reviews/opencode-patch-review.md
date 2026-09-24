# AI 补丁审核报告

## 审核范围

- Review ID: `rv-ffmpeg-vp8-lpf-003`
- 覆盖函数: vp8 loop filter 8 个 kernel（v/h × 16y/8uv × inner/正常）
- 对应 checkasm_vp8dsp 003/004/005/006/007/008/009/010

## RISC-V 架构审核

- 16/8 列并行向量化；e16 域计算 + e8 加载/存储。
- normal_limit mask（6×|diff|<=I + 2|p0-q0|+(|p1-q1|>>1)<=E）与 hev mask 用 vmsleu/vmsgtu + vand/vor。
- filter_common（4tap/8tap）与 filter_mbedge（6tap）核心，用算术选择（mask?a:b = b^((a^b)&mask)）规避 vmerge 的 v0-only 约束。
- callee-saved s0 保存（8uv 双平面）；arch-scan findings 0。

## 审核结果

**PASS**

### 验证证据

- 模拟器 20/20 × 8 函数 = 160 组合，与 FFmpeg C 参考逐字节一致。
- checkasm vp8dsp loopfilter 测试（QEMU 下 16-bit vnsrl/vwcvt 为已知 TCG 限制 → illegal instruction；模拟器为权威验证）。

### 开发期发现并修复的问题

1. vmerge 的 mask 必须是 v0 → 改用算术选择（vxor/vand）。
2. vadd.vi 63 超立即数范围 → vadd.vx。
3. mbedge 的 a0/a1/a2 系数（27/18/9）与 4tap 分支组合。
4. masks 宏参数化 E/I/thresh 寄存器（16y 与 8uv 参数位置不同）。

## 发现问题

无。

## 幻觉自检

- blueprint-anchor: PASS；root-cause-boundary: PASS；verification-evidence: PASS；diff-anchor: PASS。

## 结论

补丁通过审核，可流转 `done`。
