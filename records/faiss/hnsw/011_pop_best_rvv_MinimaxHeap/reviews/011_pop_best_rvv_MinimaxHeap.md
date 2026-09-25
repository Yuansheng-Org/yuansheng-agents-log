## Title

```
feat(rvv): add RVV kernel for HNSW MinimaxHeap pop_min
```

## Body

````markdown
`MinimaxHeapT::pop_min` is a linear scan over the candidate heap, and the RVV build currently inherits the scalar NONE implementation. This adds `pop_min_tpl<HC, SIMDLevel::RISCV_RVV>` for both CMax and CMin, mirroring `pop_best_avx2` in `hnsw/avx2.cpp` (sentinels, tie handling, and the returned worst-value semantics are identical).

`pop_min` shows up in two hot paths on riscv64: the HNSW greedy-search candidate queue, and the beam-search encode steps of the additive quantizers (it measures at ~0.5% of a residual-quantizer train+search benchmark on SG2044, all of it scalar today).

## What this adds

- `faiss/impl/hnsw/rvv.cpp`: `pop_best_rvv<HC>` plus the two explicit  specializations, resolved at link time like the AVX2/AVX512 ones.
- `faiss/CMakeLists.txt`: register the new TU in `FAISS_SIMD_RVV_SRC`.
- `faiss/impl/hnsw/MinimaxHeap.cpp`: extend `MINIMAX_HEAP_SIMD_LEVELS` with the `RISCV_RVV` bit under `#ifdef COMPILE_SIMD_RISCV_RVV` — the guard matches the `#ifdef`'d `RISCV_RVV` case in `with_selected_simd_levels`, so the extra bit is inert on every other architecture.

## Testing

- Frontend-checked with clang 22 (`--target=riscv64-linux-gnu`, `-fsyntax-only -Wall -Wextra`) at `rv64gcv` and `zvl128b/zvl256b/zvl512b`.