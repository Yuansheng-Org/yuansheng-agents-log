# AI 补丁审核报告：VP9 idct_idct 8x8 add RVV

- **reviewId**: rv-ffmpeg-vp9-idct8-005
- **reviewResult**: **pass**

## 审核范围

新增 `libavcodec/riscv/vp9itxfm_rvv.S`（`ff_vp9_idct_idct_8x8_add_rvv`）+ `vp9dsp_init.c` 注册 `itxfm_add[TX_8X8][DCT_DCT]`。

## RISC-V 架构审核

- DC-only 路径（eob==1）：标量计算 t，向量 splat 累加，与 C `has_dconly && eob==1` 分支逐位一致。
- 完整路径：pass 1 以行向量加载 8x8（vle16 stride 16），lane-wise 蝶形实现 C 的列变换（IN(x)=block[x*8+i] 分布在 lane i 上），列向写回（vsse16 stride 16）到栈上 tmp[]，精确复现 C 的 row-major 中间块；随后清零输入 block（调用方契约）。
- pass 2 按行向量重载，同一蝶形宏，`(out+(1<<4))>>5` 累加到 dst 行并 clip [0,255]。
- e32 运算，(x+8192)>>14 舍入用 vadd.vx+vsra.vi；vnsrl.wi 截断到 int16 匹配 dctcoef 语义；vwcvt/vzext 在源宽度模式执行。
- arch-scan 0 findings。

## 审核结果

**pass**

## 发现问题

无（review-validate 机器复核 findings=0）。

## 幻觉自检

- **blueprint-anchor PASS**：修改文件与 blueprint candidateFiles（libavcodec/riscv/vp9dsp_init.c）一致。
- **root-cause-boundary PASS**：仅新增 RVV 内核与注册。
- **verification-evidence PASS**：模拟器 20 用例 × 3 种 eob = 0/7680 逐字节一致（含 dst 与 block 清零契约）。
- **diff-anchor PASS**：review-validate 0 findings。

## 结论

通过。补丁与 C 参考逐字节一致，架构与锚点校验全部通过。
