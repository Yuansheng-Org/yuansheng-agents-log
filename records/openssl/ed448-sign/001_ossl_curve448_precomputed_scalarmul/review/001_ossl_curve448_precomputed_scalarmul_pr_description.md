# [RISC-V] Curve448: vectorize `constant_time_lookup` with RVV

## Summary

`ossl_curve448_precomputed_scalarmul()` spends most of its runtime inside
`constant_time_lookup()`, which it reaches through `constant_time_lookup_niels()`.
The scalar implementation walks the table row by row and accumulates one byte at a
time. This patch adds an RVV fast path that loads a whole vector group of a row,
applies the (row-invariant) constant-time mask as a scalar operand, and ORs it into
the accumulator — bit-exact with the scalar path for every mask value.

## Motivation

`constant_time_lookup()` is a generic constant-time table lookup in
`include/internal/constant_time.h`. For Curve448 scalarmul it is entered 90 times
per operation, each call scanning 16 rows of `sizeof(niels_s) == 192` bytes, i.e.
16 x 192 masked byte-select iterations per lookup. Profiling on SG2044 attributes
the bulk of `x448` and `ed448-sign` to this function (see Performance).

The scalar loop's per-byte masked-select/OR is exactly the shape RVV handles well:
the mask is loop-invariant within a row, so it can be applied with `vand.vx` while
the row bytes are loaded and stored in full vector groups.

## Implementation

Two hunks, one file (`include/internal/constant_time.h`):

1. Include `<riscv_vector.h>` under `#if defined(__riscv_vector)`.
2. Add an RVV path in `constant_time_lookup()` that strip-mines the row with
   `vsetvli` at `e8/m8`:

```c
size_t vl = __riscv_vsetvl_e8m8(rowsize);
if (vl > 0) {
    for (i = 0; i < numrows; i++, idx--) {
        mask = (unsigned char)constant_time_is_zero_s(idx);
        const unsigned char *row = tablec + i * rowsize;
        for (j = 0; j < rowsize; j += vl) {
            vl = __riscv_vsetvl_e8m8(rowsize - j);
            vuint8m8_t vr = __riscv_vle8_v_u8m8(row + j, vl);
            vuint8m8_t vm = __riscv_vand_vx_u8m8(vr, mask, vl);
            vuint8m8_t vo = __riscv_vle8_v_u8m8(outc + j, vl);
            __riscv_vse8_v_u8m8(outc + j, __riscv_vor_vv_u8m8(vo, vm, vl), vl);
        }
    }
    return;
}
```

Design notes:

- **Strip-mined, VLEN-agnostic.** `vsetvl` recomputes `vl` per group, so a partial
  final group is handled by a shorter vector. `VLMAX` is 128 bytes at VLEN=128 and
  256 at VLEN=256 — the same code covers both without a `-march`-specific build.
- **Mask as a scalar operand.** The mask is invariant within a row, so it is
  applied with `vand.vx` rather than materialised as a vector.
- **Bit-exact with the scalar path.** Every row is still loaded unconditionally and
  the mask only ever feeds data operands, so neither the control flow nor the set
  of addresses touched depends on the secret index. The `vl > 0` guard means the
  original scalar loop remains the fallback for a zero-length row.
- **No dispatch-level change.** The path is compiled in only when the translation
  unit is built with the vector extension, and returns early; all other targets and
  build configurations keep the existing scalar code verbatim.

## Performance

SG2044, riscv64, VLEN=128. Baseline (`-nopatch`) vs. this patch, same
YuanshengCI runner (`container-v4`), image digest
`sha256:4d92def334be5ad02440683a7075904a80327fd99b0f3dc52c303c6a5bfcdf1c`,
kernel build-id `375b8f2e9e900beb73ea2b497ed5067608c5a3b0`, `perf record` at
`cpu-clock` 99 Hz, 0 lost samples.

| Case | Iterations | Baseline | Patched | Duration | Throughput |
|---|---:|---:|---:|---:|---:|
| `x448` | 3000 | 3.624422 s | 2.656165 s | **-26.715%** | 827.718 -> 1129.448 ops/s (**+36.453%**) |
| `ed448-sign` | 5000 | 2.690212 s | 1.142295 s | **-57.539%** | 1858.590 -> 4377.153 ops/s (**+135.509%**) |
| `ed448-verify` | 2000 | 1.453125 s | 1.431324 s | -1.500% | 1376.344 -> 1397.307 ops/s (+1.523%) |
| `x25519` | 5000 | 1.260010 s | 1.259512 s | -0.040% | 3968.221 -> 3969.791 ops/s (+0.040%) |

Target hotspot (`perf report --no-children`, self overhead):

| Case | Baseline | Patched |
|---|---:|---:|
| `x448` | 28.81% | 3.80% |
| `ed448-sign` | 66.04% | 10.81% |

Consistent secondary signal: `branches` falls 65.690% on `x448` and 87.256% on
`ed448-sign`, matching the removal of the per-byte loop and its loop-control
overhead.

Run-to-run control: across the 51 non-target cases (excluding `x448`,
`ed448-sign`, `ed448-verify`), the paired duration change has median **+0.079%**
(P10 -0.981%, P90 +1.007%, MAD 0.246%). The `x448` and `ed448-sign` gains are far
outside that band, so they cannot be explained by the machine simply running
faster. `x25519`, which does not reach this hotspot, is flat (-0.040%).

## Correctness and evidence boundaries

- Both baseline and patched runs cover 58 benchmark cases; both report 55
  `PASSED` / 3 `FAILED`. The three shared failures (`aes-128-xts`,
  `aes-256-xts`, `cmac-aes-128-cbc`) are pre-existing and not introduced here.
  `x448`, `ed448-sign` and `ed448-verify` all return `PASSED`.
- Disassembly of the patched target function shows an `e8m8` loop
  (`vsetvli ...,e8,m8` / `vle8.v` / `vand.vx` / `vor.vv` / `vse8.v`) that is
  absent from the baseline.
- **Not yet verified:** a full OpenSSL functional-test log was not available for
  this run, so API-level `PASSED` alone does not prove cryptographic correctness.
  `constant_time_lookup()` is side-channel sensitive: the patch still visits every
  table entry and uses masked selection, but the constant-time property needs its
  own review, which performance data cannot answer.
- PMU `cpu_cycle` / `instruction` and some cache events show platform counting
  anomalies (1e13-scale cycles; cache misses above references) and are **not** used
  as evidence here. The conclusions rest on wall-clock, task-clock, throughput,
  hotspot share and the non-target control distribution.

## Upstream status

The same hotspot is already addressed upstream by **PR #32835**
("riscv: vectorize constant_time_lookup for Curve448 scalarmul", open), which is
the upstream-tracked submission for this craft/review pair. That PR implements the
same algorithm via raw `.word` encodings emitted through
`crypto/perlasm/riscv.pm`, rather than inline intrinsics, so that the generated
object does not depend on the assembler accepting the vector mnemonics under a
default `linux64-riscv64` build (which passes no `-march`).

This record is the perf-verified artifact for that hotspot; the two differ only in
how the vector instructions are emitted. Please review the upstream PR for the
submission-form discussion.

## Risk

Low, contained to one header's helper.

- Behaviour is bit-exact with the scalar path for all mask values, by construction
  (unconditional row load, mask only in data operands).
- The RVV path is guarded by `#if defined(__riscv_vector)`; all other targets are
  untouched.
- The final partial group is handled by `vsetvl`, so no assumption is made about
  `rowsize` being a multiple of `VLMAX`.

## Attribution

Patch derived from the Yuansheng trace/craft pipeline
(`craft/openssl/001_ossl_curve448_precomputed_scalarmul`,
blueprint `bp-openssl-ed448-sign-001`); correctness and performance verification on
real hardware performed independently. Author/sign-off to be filled in by the
submitter.

---

### Diff

```text
diff --git a/include/internal/constant_time.h b/include/internal/constant_time.h
index ddb15d7b6f..1dbb0975e3 100644
--- a/include/internal/constant_time.h
+++ b/include/internal/constant_time.h
@@ -15,6 +15,10 @@
+#if defined(__riscv_vector)
+# include <riscv_vector.h>
+#endif
@@ -463,6 +467,36 @@ static ossl_inline void constant_time_lookup(void *out,
+#if defined(__riscv_vector)
+    /* e8/m8 masked-select OR accumulation; strip-mined, VLEN-agnostic */
+#endif
```

Blob hashes as recorded in the patch header: `ddb15d7b6f` (pre-image) → `1dbb0975e3`
(post-image). These are the hashes of the worktree the patch was generated in; against
upstream `master` at the time of writing the file hashes to
`168b3e8880f5135ca18f58309fe1703f80310d49`. The patch applies cleanly to upstream
`master` regardless — the header hashes are informational, not a base requirement.
