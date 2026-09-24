# 无同方向的优化前（craft）补丁；最近前驱是 PR #5535（仅 decode 方向）

两个来源仓均无 Lloyd-Max **编码**方向的 craft 候选：craft/faiss 的 20 个目录只覆盖 DCTemplate / PQ / fvec_add / Clustering（其中 17 个目录内容还是同一份累积补丁），yuansheng-patches 的 sq-encode / sq-decode / sq-accuracy 蓝图亦无 `QuantizerLloydMax`；trace 的 6 个 testcase 中也没有 LloydMax 用例。

最近的前驱（同目标、不同方向）是 **PR #5535**：它为 `QuantizerLloydMax<1/2/3/4/8, RISCV_RVV>` 建立了 RISCV_RVV 特化，但只含 `reconstruct_m8_components` / `decode_vector`——`encode_vector` 出现次数为 0。本补丁（encode 方向）正建立在该特化之上，且按提交说明「scalar base 的 `encode_vector` 本就可重写」，无需改动 quantizers.h。

复核范围：项目内全部 `*.patch`/`*.diff`（675 个，排除 `.git/`、`node_modules/`）按目标函数 `QuantizerLloydMax` 检索，命中 6 个文件，**无一是独立的 encode 方向 craft**：

- `pr-5535.patch` ×2（hebo、jinge 两次 screening 各存一份副本）—— 同目标、仅 decode 方向；
- `10-rvv-sq-encode-lloydmax.patch`（提交包 pack 10）与本记录 `review/patch-opt.diff`（两个 worktree 各一份）—— 即本补丁自身；
- `faiss/patch.sq-encode3-on-pr5535.diff` —— 本补丁的源件。

（`_wt_agentslog_osslpr/` 下那份是本记录在另一 worktree 的检出，不是第三方来源。）
