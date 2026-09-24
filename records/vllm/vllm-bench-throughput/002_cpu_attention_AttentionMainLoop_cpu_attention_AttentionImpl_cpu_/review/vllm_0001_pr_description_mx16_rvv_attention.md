# [CPU][RISC-V] Attention: use a 16-column RVV tile for the narrow-M GEMMs

## Summary

The RISC-V RVV CPU attention kernel currently drives every output tile with an
8-column (LMUL_256) B-row load and one accumulator per row. For the narrow-M
shapes that dominate decode and GQA (`m_size <= 4`), that shape has too little
independent work to keep the core busy and re-issues the same scalar A loads
twice as often as necessary.

This PR adds a 16-column micro kernel that keeps **two independent 8-lane (m2)
accumulator chains per row** and calls it from the macro kernel when
`N % 16 == 0 && mb <= 4`. M >= 8 (prefill) keeps the existing Mx8 tile.

Single file, no API change, no new symbol outside `csrc/cpu/cpu_attn_rvv.hpp`.

```text
 csrc/cpu/cpu_attn_rvv.hpp | 156 +++++++++++++++++++++++++++++++++++++++++++++-
 1 file changed, 155 insertions(+), 1 deletion(-)
```

## Motivation

`gemm_macro_rvv_fma_Mx8_Ku4` is used by both attention phases via `TileGemmRVV`
(`BLOCK_SIZE_ALIGNMENT = HEAD_SIZE_ALIGNMENT = 32`), so `N` is a compile-time
32 and `M` is `q_head_num` for the iteration:

- decode / GQA: `M` is small (typically 1-4), `K` is `head_dim` (QK) or the
  token count in the block (PV);
- prefill: `M` is 8-16.

With `mb <= 4` the existing kernel issues, per k step, one 8-column B load and
then only `mb` scalar-broadcast `vfmacc.vf`. The scalar A loads (`flw`) are
therefore re-issued once per 8 output columns even though the same `a[i][k]`
value feeds the whole 16-column row width that the block already holds, and the
per-row dependency chain is a single accumulator.

## What changed

1. **New micro kernel** `gemm_micro_rvv_fma_Mx16_Ku4<M, kv_cache_t>`
   (`M` in `[1,4]`, `K` unrolled by 4 plus a tail loop). Each row keeps two
   `fixed_fp32x8_t` accumulators, `accl<i>` for output columns `[0,8)` and
   `acch<i>` for `[8,16)`. Register budget at VLEN=128: `4 * m2` accumulators
   (16 regs) + 2 `m2` B rows (4 regs) = 20 of 32.
2. **Macro kernel dispatch**: when `N % 16 == 0` and `mb <= 4`, step `n` by 16
   and call the new kernel; otherwise fall through to the unchanged 8-column
   path. `mb == 8` (prefill) still uses `gemm_micro_rvv_fma_Mx8_Ku4`.
3. Corrected the now-stale section comment describing the N step.

Nothing else changes: paged KV layout, stride contract, mask / softmax / alibi /
softcap semantics, OpenMP structure, and the QK/PV public API are untouched.

## Why two 8-lane chains, and not one wider one

The obvious alternative — one 16-element (`LMUL_512`) accumulator per row and a
single 16-element B load — was implemented and measured, and is **clearly
worse**. Three variants were compared on the target machine (details in
"Performance"):

| variant | per-row accumulators | B-row load | speedup at M=1, K=256 |
|---|---|---|---|
| Mx8 baseline | 1 x m2 | 1 x m2 (8 cols) | 1.00x |
| **this PR** | **2 x m2** | **2 x m2** | **1.49-1.56x** |
| LMUL_512 variant | 1 x m4 | 1 x m4 (16 cols) | 1.11-1.13x |
| LMUL_512 load only | 2 x m2 | 1 x m4 (+ `vget` split) | 1.10-1.13x |

Widening a vector register does not create an independent dependency chain — it
makes one chain longer over more lanes. What helps here is having two
independent accumulator chains per row, which the core can overlap. Note that
the register accounting is identical for both designs (20 of 32), so the wider
tile buys no register headroom either.

`M >= 8` is deliberately left on the Mx8 tile: `4` rows x `m4` would need 32
accumulator registers, and prefill shapes are not the bottleneck.

The kernel comment in the patch states this mechanism explicitly, so the code
and the measured result agree; an earlier draft of the rationale attributed the
gain to the wider B access, which the measurements above do not support.

## Correctness

**Bit-identical to the existing kernel.** For a fixed output element, `k` still
ascends in steps of one and every step is a single `vfmacc.vf` into the same
element, so the FP32 accumulation order is unchanged; the 16-column tile only
changes which accumulators live side by side. The claim was checked
empirically rather than argued:

- the kernel from this patch was compiled against the real
  `csrc/cpu/cpu_types_riscv_defs.hpp` (`RVVI` / `LMUL_*`) at the target VLEN,
  together with the unmodified `gemm_micro_rvv_fma_Mx8_Ku4`;
- sweep `M in {1,2,3,4,8,16} x N in {8,16,24,32,64} x K in {1,3,4,7,31,64,128,
  255,256} x accumulate in {0,1}` = 540 cases;
- result: **540/540 byte-identical** (`memcmp` of the full output tile) versus
  the Mx8 kernel, and within `1e-4` relative of a scalar reference.

The code inside this patch is the code that was compiled for that test (the
kernel body is textually identical apart from line wrapping and comments).

**Existing tests already exercise the new path.** In
`tests/kernels/attention/test_cpu_attn.py`,
`test_varlen_with_paged_kv_normal_rvv` parametrizes
`num_heads` over `(4, 4)`, `(8, 2)`, `(9, 3)` and `SEQ_LENS` contains
`q_len = 1` decode batches, so both the narrow-M (`mb <= 4`, new tile) and the
wide-M (`mb == 8`, unchanged tile) dispatch are covered, across
`HEAD_SIZES = [96, 128, 512]` and `block_size in {96, 128}`.

## Performance

Measured on the target SoC, single core pinned, 400 warmup iterations,
25 samples per round, rotating variant order, median reported, three
independent processes. Kernel-level, `N = 32`, `-O3`, GCC 15.1, VLEN=128:

| shape | speedup vs Mx8 (3 runs) |
|---|---|
| M=1 K=256 | 1.563 / 1.543 / 1.485 |
| M=2 K=256 | 1.423 / 1.425 / 1.340 |
| M=1 K=128 | 1.199 / 1.258 / 1.474 |
| M=2 K=128 | 1.134 / 1.290 / 1.369 |
| M=4 K=256 | 1.154 / 1.156 / 1.154 |
| M=4 K=128 | 1.058 / 1.184 / 1.145 |

Two caveats stated plainly:

- **Noise floor.** Rows with `M >= 8` are byte-identical code in all variants
  and measured **0.87x-1.22x** on this (busy, multi-tenant) machine. The M=1 and
  M=2 rows are well outside that band and reproduce across runs; the M=4 rows
  overlap it and should not be quoted on their own.
- **Kernel-level, not end-to-end.** These numbers are the GEMM tile in
  isolation. Real model throughput will be lower, since softmax, KV movement,
  scheduling and the non-GEMM parts of the kernel are unchanged.

## Not verified in this PR

Requires a full CPU build of vLLM, which was not available when this was
prepared. The following must be done before merge:

- [ ] `pip wheel .` on riscv64 with `-DVLLM_RVV_VLEN=128` and run
      `tests/kernels/attention/test_cpu_attn.py -k rvv`;
- [ ] end-to-end `vllm bench latency` / `vllm bench throughput` before and
      after, on SG2044 with VLEN=128 (ideally also on a VLEN=256 part, where
      `LMUL_512` maps to `m2`, and on VLEN=128 with `VLLM_RVV_VLEN=256`);
- [ ] confirm no register spill in the new kernel (`-fopt-info` / `objdump`);
- [ ] K = 96 and K = 512 were not microbenchmarked (K = 128, 256 were); the
      kernel is K-agnostic (unroll by 4 + tail, tails covered by the sweep).

## Risk

Low and well contained. One translation unit, no ABI or API change, the new
tile is only reachable for `mb <= 4` with `N % 16 == 0`, and `N` is the
compile-time 32 for both phases, so the wide/prefill path is untouched. The
fallback for any `N % 16 != 0` is the existing code path verbatim.

## Attribution

Patch derived from the Yuansheng trace/craft pipeline
(`craft/vllm/002_cpu_attention_AttentionMainLoop_cpu_attention_AttentionImpl_cpu_`,
blueprint `bp-vllm-vllm-bench-throughput-002`); correctness and performance
verification on real hardware performed independently. Author/sign-off to be
filled in by the submitter.

---

### Diff

```text
diff --git a/csrc/cpu/cpu_attn_rvv.hpp b/csrc/cpu/cpu_attn_rvv.hpp
index 396cc55c5..a455cb30c 100644
--- a/csrc/cpu/cpu_attn_rvv.hpp
+++ b/csrc/cpu/cpu_attn_rvv.hpp
@@ -183,7 +183,132 @@ FORCE_INLINE void gemm_micro_rvv_fma_Mx8_Ku4(
@@ -194,11 +319,40 @@ FORCE_INLINE void gemm_macro_rvv_fma_Mx8_Ku4(const float* __restrict A,
```

Base: upstream `main` at `0aee727ff6131a1b647941186ceef8f4777bdac2`
(2026-09-21). `csrc/cpu/cpu_attn_rvv.hpp` is unchanged between v0.24.0 and this
commit (`396cc55c59`), so the patch applies to either.
