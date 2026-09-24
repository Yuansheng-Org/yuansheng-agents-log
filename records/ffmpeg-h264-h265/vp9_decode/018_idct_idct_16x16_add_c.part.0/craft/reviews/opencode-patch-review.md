# AI 补丁审核报告：VP9 idct_idct 16x16 add RVV

- **reviewId**: rv-ffmpeg-vp9-idct16-018
- **reviewResult**: **pass**

## 审核范围

`libavcodec/riscv/vp9itxfm_rvv.S` 新增 `ff_vp9_idct_idct_16x16_add_rvv`，并在 `vp9dsp_init.c` 注册 `itxfm_add[TX_16X16][DCT_DCT]`。

## RISC-V 架构审核

- 分两个 8 列半块处理，e32 m1（8 lanes）lane-wise 16 点蝶形（stage1 t*a / stage2 t / stage3 旋转 / stage4 / stage5 / 输出，逐段与 C 数据流核对）。
- Pass 1：每半块加载 16 个行向量（vle16，行步长 32 字节），运行 vp9_idct16_1d —— lane-wise 实现 C 的列变换（IN(x)=block[x*16+i] 分布在行向量的 lane i 上）；e32 模式 vnsrl.wi 截断，列向 vsse16（步长 32）写入栈 tmp[]；输入 block 以 vl=16 存储清零（调用方契约）。
- Pass 2：从 tmp[] 按行向量重载，同一蝶形，`(out+32)>>6` 累加到 dst 行并 clip [0,255]。
- DC-only（eob==1）：标量 t + 向量 splat 累加。
- 模拟器 20 用例 × 3 种 eob = 0/15360 逐字节一致（dst 块区域 + block 清零契约）。
- arch-scan 0 findings。

## 审核结果

**pass**

## 发现问题

无（review-validate 机器复核 findings=0）。

## 幻觉自检

- **blueprint-anchor PASS**：修改文件与 blueprint candidateFiles 一致。
- **root-cause-boundary PASS**：仅新增 RVV 内核与注册。
- **verification-evidence PASS**：模拟器 0/15360 逐字节一致。
- **diff-anchor PASS**：review-validate 0 findings。

## 结论

通过。补丁与 C 参考逐字节一致，架构与锚点校验全部通过。
