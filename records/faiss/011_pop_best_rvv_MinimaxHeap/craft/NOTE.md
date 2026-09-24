# 无优化前（craft）补丁，相似的也没有；本补丁新增文件而非改写既有实现

HNSW MinimaxHeap 方向在全部来源中都没有 craft 候选，逐个排除如下：

- **craft/faiss**（20 个目录）无 `hnsw` 目录，也无 `MinimaxHeap` / `pop_best`；yuansheng-patches 的 `faiss_new_0901` 各蓝图同样没有。
- **PR #5539**（同在 sq-rvv.cpp 上、与 #5535 同批的 RVV 距离补丁）0 命中 `hnsw|pop_min|MinimaxHeap|pop_best`，其 hunk 全部是 `DCTemplate` / `DistanceComputerByte`。
- **trace**（`yuansheng-agent-debug/trace/faiss/`，6 个 case 目录、**60 个 annotate 文件**；整棵 trace 树 179 个）0 命中 `lloyd|hnsw|minimax|pop_best|pop_min`，6 个 case 中没有 hnsw 用例。
- 文件名最接近的候选是 `rcq-search/018 candidate-6c0fb0b9`（同样改 `CMakeLists.txt` 并沿用 ResultHandler/riscv.cpp 的写法），但它新增的是 `impl/result_handler/riscv.cpp`，**不是** `impl/hnsw/`，目标函数也不同。

结构上也决定了不会有"优化前补丁"：本补丁并非改写已有标量实现，而是**新增** `faiss/impl/hnsw/rvv.cpp`（`new file mode 100644`，116 行），只在 `MinimaxHeap.cpp` 的 `MINIMAX_HEAP_SIMD_LEVELS` 里加一个 `RISCV_RVV` 位。优化前状态即上游 main 原样（`MinimaxHeap.cpp` 的 pre-image blob `44a970cdb` 与上游 `e683890`/`80a1656` 完全一致），没有对应的候选补丁。
