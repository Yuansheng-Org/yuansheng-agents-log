# RISC-V: vectorize float CPU im2col with RVV

## Motivation

CPU `im2col` gathers input pixels into a contiguous column buffer. The existing inner loop checks image bounds and computes a source address for every output element. On RISC-V with RVV, a row can instead be split into padding and an in-bounds span, then copied with vector loads and stores.

## Implementation

- Factor the output-row operation into a helper. The scalar helper remains available for other data types.
- For `float` on RVV, calculate the in-bounds column interval once per row. Write zero to the padded prefix and suffix, use unit-stride loads when `stride_w == 1`, and strided loads otherwise.
- Use variable vector lengths with e32,m4 so the same loop handles different row widths and tails. The public `im2col_cpu` interface is unchanged.

## Testing

Built from Caffe commit `9b891540183ddc834a02b2bd81b31afae71b2153` in Release, CPU-only mode with OpenBLAS and `-O3 -DNDEBUG -march=rv64gcv`. The unpatched build and this patch used the same settings. On the RISC-V test host, each build was measured three times with:

```bash
taskset -c 0 env OPENBLAS_NUM_THREADS=1 OMP_NUM_THREADS=1 \
  <build>/tools/caffe time \
  --model=models/bvlc_reference_caffenet/deploy.prototxt --iterations=5
```

## Performance

`Average Forward-Backward` for the complete CaffeNet model, in milliseconds; lower is better.

| Build | Three runs | Median |
| --- | --- | ---: |
| Upstream | 2239.6, 2243.8, 2245.6 | 2243.8 |
| RVV im2col | 2201.2, 2201.2, 2202.0 | 2201.2 |

The median improved by **1.90%**. All three patched runs were faster than the fastest upstream run.
