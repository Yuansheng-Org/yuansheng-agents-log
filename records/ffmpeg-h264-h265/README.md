# FFmpeg/H264/H265：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`FFmpeg/H264/H265`；目录标识：`ffmpeg-h264-h265`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 21     | 314    | 22               | 12     | 3             | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-ffmpeg` 已筛选的 11 个函数目录，按 `records/ffmpeg-h264-h265/<测试用例>/<函数>/` 归档。来源为 `yuansheng-patches/ffmpeg-1d7b14f61d66fdf18f15204c613df9d65396c319/20260914` 的同名测例与函数目录；`20260915` 批次没有本分支已筛选的函数。原有专家复核文件保留。

| 测试用例 | 函数目录 | Blueprint ID | 来源交付候选（未重复归档） |
| --- | --- | --- | --- |
| `aac_decode_sbr_ps` | [001_ff_tx_mdct_inv_float_c](aac_decode_sbr_ps/001_ff_tx_mdct_inv_float_c/) | `bp-ffmpeg-aac_decode_sbr_ps-001` | `candidate-027a7c08ee.patch` |
| `checkasm_h264chroma` | [003_avg_h264_chroma_mc4_16_c](checkasm_h264chroma/003_avg_h264_chroma_mc4_16_c/) | `bp-ffmpeg-checkasm_h264chroma-003` | `candidate-7a8e946abb.patch` |
| `checkasm_hevc_pel` | [001_put_hevc_qpel_bi_w_hv_8](checkasm_hevc_pel/001_put_hevc_qpel_bi_w_hv_8/) | `bp-ffmpeg-checkasm_hevc_pel-001` | `candidate-501082bb6b.patch` |
| `checkasm_sw_scale` | [001_yuv2planeX_10LE_c](checkasm_sw_scale/001_yuv2planeX_10LE_c/) | `bp-ffmpeg-checkasm_sw_scale-001` | `candidate-52464f8bdb.patch` |
| `checkasm_vf_blend` | [002_blend_subtract_16bit](checkasm_vf_blend/002_blend_subtract_16bit/) | `bp-ffmpeg-checkasm_vf_blend-002` | `candidate-fb38f3ce36.patch` |
| `filter_bwdif_deinterlace` | [001_ff_bwdif_filter_line_c](filter_bwdif_deinterlace/001_ff_bwdif_filter_line_c/) | `bp-ffmpeg-filter_bwdif_deinterlace-001` | `candidate-82d6f32273.patch` |
| `mpeg4_encode` | [003_ff_jpeg_fdct_islow_8](mpeg4_encode/003_ff_jpeg_fdct_islow_8/) | `bp-ffmpeg-mpeg4_encode-003` | `candidate-b5cbe0bd4b.patch` |
| `mpeg4_encode` | [006_pix_abs16_xy2_c](mpeg4_encode/006_pix_abs16_xy2_c/) | `bp-ffmpeg-mpeg4_encode-006` | `candidate-ca11d05058.patch` |
| `vp8_decode` | [003_vp8_v_loop_filter16_inner_c](vp8_decode/003_vp8_v_loop_filter16_inner_c/) | `bp-ffmpeg-vp8_decode-003` | `candidate-4b62b51adc.patch` |
| `vp9_decode` | [005_idct_idct_8x8_add_c](vp9_decode/005_idct_idct_8x8_add_c/) | `bp-ffmpeg-vp9_decode-005` | `candidate-6292f0ace8.patch` |
| `vp9_decode` | [018_idct_idct_16x16_add_c.part.0](vp9_decode/018_idct_idct_16x16_add_c.part.0/) | `bp-ffmpeg-vp9_decode-018` | `candidate-f3c9afcaf1.patch` |

`trace/` 保留来源 `blueprint/` 的全部内容；每个函数的 `craft/` 只保留一份 `patch.diff`。上表列出的交付候选文件仍在上述 `yuansheng-patches` 来源批次的同名测例与函数目录，不在本仓库重复归档。若来源函数目录存在逐函数 `craft/`，则 `craft/patch.diff`、Plan、Candidate 和 AI 审核文件也原样归档。`006_pix_abs16_xy2_c` 的来源目录没有逐函数 Craft 协议文件，因此本目录仅保留原有 `patch.diff` 和 Trace，不补造 Plan/Candidate/AI 审核。

**补丁版本溯源：**来源 `summary.json.delivery_refs` 的 Blueprint 与交付候选 SHA-256 已逐项核验；对 10 项有逐函数 Craft 协议的记录，`patch-candidate.json.gitDiff` 与归档的 `craft/patch.diff` 完全一致。下列三项在 `add-ffmpeg` 中的 `craft/patch.diff` 与该原始 Craft diff 不同；原 `add-ffmpeg` 文件已另存到各目录的 `provenance/add-branch-patch.diff`，没有丢弃。

- `018_idct_idct_16x16_add_c.part.0`：原 `add-ffmpeg` diff 在补丁尾部提前结束（26,969 字节）；来源 Craft diff 与交付候选相同（35,856 字节）。
- `001_put_hevc_qpel_bi_w_hv_8`、`002_blend_subtract_16bit`：原 `add-ffmpeg` diff 与来源交付候选相同，但与逐函数 Craft diff 不同；两份补丁各自保留，不能混称为同一版本。

FFmpeg 的部分补丁是共享内核的族交付，同一交付候选可能出现在多个函数目录；此处仅归档 `add-ffmpeg` 已筛选的函数。历史统计保持原值；AI 审核或交付候选不自动证明真机编译、回归或上游接收。
