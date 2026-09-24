# 无优化前补丁

本补丁是 005（decode,codec-template）与 006（decode,raw-codec）的合并增量：其 11 个 hunk 恰等于两者之和（quantizers.h 5 处 = 2+3，sq-rvv.cpp 6 处 = 2+4）。它不是独立方向，因此没有单独的优化前补丁；优化前状态见 005/006 的 craft。
