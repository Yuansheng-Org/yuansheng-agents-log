# RISC-V: vectorize float CPU max pooling with RVV

## Motivation

CPU max pooling currently walks each output window with scalar loads, bounds checks, comparisons, and mask-index updates. For windows fully inside the image, adjacent output columns follow the same access pattern and can be reduced together with RVV.

## Implementation

- Vectorize the interior columns of `float` MAX pooling across output positions. Strided vector loads read each window tap; strict-greater masks update the running value and its source index together.
- Keep the scalar path for border rows and columns, and for data types without the RVV overload. The AVE and STOCH pooling branches are unchanged.
- Calculate horizontal interior bounds once per channel plane. The common three-column kernel width has an unrolled horizontal tap loop; other widths use the general loop. The vector helper uses e32,m4 and remains out of line to limit growth of `Forward_cpu<float>`.

## Testing

Built from Caffe commit `9b891540183ddc834a02b2bd81b31afae71b2153` in Release, CPU-only mode with OpenBLAS and `-O3 -DNDEBUG -march=rv64gcv`. The unpatched build and this patch used the same settings. The final comparison alternated the two builds on the RISC-V test host, with three runs of each:

```bash
taskset -c 0 env OPENBLAS_NUM_THREADS=1 OMP_NUM_THREADS=1 \
  <build>/tools/caffe time \
  --model=models/bvlc_reference_caffenet/deploy.prototxt --iterations=5
```

## Performance

`Average Forward-Backward` for the complete CaffeNet model, in milliseconds; lower is better.

| Build | Three runs | Median |
| --- | --- | ---: |
| Upstream | 2244.6, 2242.8, 2234.6 | 2242.8 |
| RVV max pooling | 2204.4, 2204.0, 2206.8 | 2204.4 |

The median improved by **1.71%**. The slowest patched run was still 27.8 ms faster than the fastest upstream run.
