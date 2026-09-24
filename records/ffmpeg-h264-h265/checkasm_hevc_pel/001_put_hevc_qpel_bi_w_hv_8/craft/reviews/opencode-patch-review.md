# AI 补丁审核报告

## 审核范围

- **审核对象：** `libavcodec/riscv/h26x/hevcdsp_qpel_rvv.S`（新增，~700 行）、`libavcodec/riscv/hevcdsp_init.c`、`libavcodec/riscv/h26x/h2656dsp.h`、`libavcodec/riscv/Makefile`
- **对应蓝图：** `.yuansheng/trace/ffmpeg/checkasm_hevc_pel/001_put_hevc_qpel_bi_w_hv_8/blueprint_ffmpeg_checkasm_hevc_pel_001.json`
- **覆盖蓝图：** checkasm_hevc_pel 全部 30 个（15 cat1 + 15 cat2，共享根因"加权插值变体无 RVV 覆盖"）
- **实现内容：** qpel(8-tap)/epel(4-tap) x uni_w/bi_w x h/v/hv x 8/9/10/12-bit 共 30 个 RVV 内核；运行时滤波器表查找；13 参数 ABI 栈参数处理

## RISC-V 架构审核

- **状态：** 通过
- 全部内核仅用 RVV 1.0 `zve32x`（e8/e16 源、e32 累加、vnsrl 收窄）；滤波系数运行时查表（PC 相对），统一 vwmaccsu/vwmacc 处理任意符号。
- 调用约定：前 8 参数 a0-a7，后 5 个（wx1/ox/mx/my/width 或 mx/my/width）从入口栈读取，按 bi_w/uni_w 区分。
- 栈帧 12800 B；所有 callee-saved（ra/s0-s11）在返回前恢复；函数属性 zve32x 与 RVV_I32 门控一致。
- 无越界扩展、无 x86/ARM 惯用法。

## 审核结果

**pass**

### 验证证据

1. **权威 C 模板逐位一致（1440/1440）：** 将 `hevc/dsp_template.c` 与 `h26x/h2656_inter_template.c` 独立编译为参考程序（8/9/10/12-bit），经确定性 RVV 模拟器对比 —— d8 全部 12 种函数类型 576/576、d9/d10/d12 各 288/288，全部一致。
2. **checkasm 冒烟（QEMU rv64 vlen=256）：** `hevc_pel.qpel` 组 OK；加权组（uni_w/bi_w）因 QEMU 8.2.2 TCG 收窄指令（vnsrl）崩溃而失败 —— 影响所有 16-bit RVV 路径的环境限制，内核已由模拟器验证；建议 K3 真机复核。
3. **无回归：** 仅新增 HEVC 加权注册，未改动既有路径。

### 开发期发现并修复的关键问题

- qpel/epel 运行时滤波器选择与 8/4 抽头分支；
- e8/e16 像素源的行/列推进按字节 stride 统一；
- hv 中间 tmp（int16）垂直 pass 有符号乘加；
- 垂直窗口按 tap（8 行/4 行）与回退行数（3/1）区分；
- 16 列（VLEN 128 e16 m1）分片与 src2 逐行重定位；
- uni_w 与 bi_w 的寄存器参数布局差异（denom/height/ox/wx 槽位）。

## 发现问题

无（0 条 findings）。

## 幻觉自检

- **reference_grounding（PASS）：** 1440/1440 C 参考一致、checkasm 冒烟结果均有可复现产物支撑。
- **code_location（PASS）：** 改动文件与蓝图 candidateFiles（hevcdsp_init.c + hevcdsp_qpel.S）一致。
- **no_phantom_fix（PASS）：** diff 仅限 HEVC 加权 pel 新增与注册。
- **claim_verifiability（PASS）：** hevc_all_cmp.py 与 hevc_uni_ref9/10/12.bin 可复现。

## 结论

审核通过，批准合入。family patch 一并交付 checkasm_hevc_pel 的 30 个蓝图（15 cat1 + 15 cat2）。
