# 无独立优化前（craft）补丁；优化前状态 = 005 与 006 的 craft 叠加

本补丁是 005（decode, codec-template）与 006（decode, raw-codec）的合并增量：11 个 hunk 恰等于两者之和（`quantizers.h` 5 处 = 2+3、`sq-rvv.cpp` 6 处 = 2+4），逐字节比对两者新增行的并集与本补丁新增行的并集完全一致。

因此它不是独立优化方向，没有单独的 craft 补丁。其优化前状态就是 005+006 的 post-image，也就是把本补丁的 hunk 行号平移回 005+006 未合并时的基线：

- 以 **PR #5535** 为基线时，本补丁声明 `sq-rvv.cpp` 的 hunk 起于第 137 行；而 005 自身的 hunk 起于第 112 行，两者相差正好是 005 插入的 23 行——即本补丁的基线就是「PR #5535 + 005 + 006」。
- 该叠加状态下 `quantizers.h` 的 post-image 为 006 的 post-image（`8e05a7709`），与本补丁 5 处 `quantizers.h` hunk 的上下文完全吻合。

复核范围：全语料 358 个 patch/diff 文件按目标函数与本补丁 hunk 上下文检索，无第三方独立候选。
