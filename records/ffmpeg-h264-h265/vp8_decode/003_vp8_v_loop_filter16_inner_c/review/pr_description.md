## 性能数据

平台：SpacemiT X100, RV64, VLEN=256, CPU 0, about 2.199 GHz。checkasm 为 7 次独立运行的中位数，越低越好。

| 函数 | C | RVV | speedup |
|---|---:|---:|---:|
| `vp8_loop_filter16y_inner_v` | 130.756 | 77.985 | 1.677x |
| `vp8_loop_filter16y_v` | 146.775 | 97.532 | 1.505x |
| `vp8_loop_filter8uv_inner_v` | 139.968 | 92.108 | 1.520x |
| `vp8_loop_filter8uv_v` | 175.507 | 115.337 | 1.522x |

端到端 VP8 解码的 5 轮中位数：

| 构建 | task-clock | cycles | speedup vs baseline |
|---|---:|---:|---:|
| baseline | 3.27994 s | 7.213 B | 1.000x |
| v2-style eight-function candidate | 2.74440 s | 6.036 B | 1.195x |
| this patch | 2.54105 s | 5.588 B | 1.291x |

最终源码清理后的复测中位数为 2.56064 s / 5.631 B cycles；收益保持在约 1.28–1.29 倍范围。
