Add an RVV HAL implementation for CV_32F to CV_16S conversion.

The RISC-V convertScale HAL already handles several source/destination
depth combinations, but CV_32F to CV_16S currently returns
CV_HAL_ERROR_NOT_IMPLEMENTED and falls back to the generic core path.

Implement the missing conversion using native RVV intrinsics. The
identity-scale case skips the multiply-add, while scaled conversions
apply alpha and beta before narrowing to signed 16-bit output.

The implementation is VLEN-agnostic and uses vsetvl for tail handling.

Functional test:

```text
./build-rvv/bin/opencv_test_core \
    --gtest_filter='*ConvertScale*'
```

Performance was measured with opencv_perf_core on SpacemiT K3
(RVV 1.0, VLEN=256), comparing against an unmodified 5.x baseline.

CV_32F -> CV_16S median time:

```text
1920x1080 C1, alpha=1:
    26.16 ms -> 1.46 ms  (17.92x)

1920x1080 C1, alpha=1/255:
    24.10 ms -> 1.45 ms  (16.62x)

1920x1080 C4, alpha=1:
    103.76 ms -> 6.06 ms  (17.12x)

1920x1080 C4, alpha=1/255:
    96.03 ms -> 5.68 ms  (16.91x)