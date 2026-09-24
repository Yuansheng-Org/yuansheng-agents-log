# AI 补丁审核报告：ff_jpeg_fdct_islow_8 RVV

- **reviewId**: rv-ffmpeg-mpeg4-fdct-003
- **patchCandidateId**: pc-bp-ffmpeg-mpeg4_encode-003
- **reviewResult**: **pass**

## 审核范围

新增 `libavcodec/riscv/fdctdsp_rvv.S`（`ff_jpeg_fdct_islow_8_rvv`）+ `fdctdsp_init.c`，在 `libavcodec/fdctdsp.c` 注册 ARCH_RISCV 钩子。

## RISC-V 架构审核

- 两遍 8 点 LL&M 蝶形均为 8 向量 lane-wise 运算。
- Pass 1 以列向量方式加载（vlse16 stride=16），行蝶形（x0+x7、x1+x6 等对）跨列向量逐元素完成；结果按列写回（vsse16 stride=16），原地复现 C 的 row-major 中间块。
- Pass 2 按行重载（vle16）复用同一蝶形宏。
- 乘法全部在 e32 mod 2^32；DESCALE=(x+(1<<(n-1)))>>n 用 vadd.vx+vsra.vi；int16 截断用 vnsrl.wi；pass1 偶输出为 <<PASS1_BITS（vsll.vi）。
- 与 jfdctint_template.c 逐位一致；arch-scan 0 findings。
- 注册门禁：8-bit、RVV_I32、vlen>=128、dct_algo 非 FASTINT/FAAN。

## 审核结果

**pass**

## 发现问题

无（review-validate 机器复核 findings=0）。

## 幻觉自检

- **blueprint-anchor PASS**：修改文件与 blueprint candidateFiles 一致。
- **root-cause-boundary PASS**：仅新增 RVV 内核与注册。
- **verification-evidence PASS**：模拟器 30 用例 × 64 系数 = 0/1920 逐位一致。
- **diff-anchor PASS**：review-validate 0 findings。

## 结论

通过。补丁与 C 参考逐位一致（1920/1920），架构与锚点校验全部通过。
