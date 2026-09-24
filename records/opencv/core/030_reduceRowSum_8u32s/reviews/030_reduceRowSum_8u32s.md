Optimize the RVV path of reduceRowSum_8u32s by using the native
widening add instruction vwaddu.wv.

The generic scalable-vector path expands each u8m1 input vector into
two u16m1 halves and performs two separate load/add/store sequences
for the u16 accumulation buffer.

On RVV, an u8m1 vector and an u16m2 vector have the same number of
elements. Use an u16m2 accumulator directly and combine widening and
addition with vwaddu.wv:

    u16m2 = u16m2 + u8m1

This removes the explicit widening and low/high u16 accumulator split,
reducing the vector operations in the hot accumulation loop while
keeping the existing scalar tail and 256-row u16-to-u32 flush logic
unchanged.

The implementation is VLEN-agnostic.

Functional test:

    ./bin/opencv_test_core \
        --gtest_filter='*Reduce*:*reduce*'

Performance test:

    OPENCV_FOR_THREADS_NUM=1 ./bin/opencv_perf_core \
        --gtest_filter='*Reduce*:*reduce*'

Performance was measured on SpacemiT K3 / X100 with RVV 1.0,
VLEN=256, GCC 14.3.0, a Release build, and a single thread.

Median execution time:

    Size / type       Baseline    RVV vwaddu.wv    Speedup
    640x480  8UC1       0.09          0.05          1.80x
    640x480  8UC4       0.36          0.16          2.25x
    1280x720 8UC1       0.27          0.12          2.25x
    1280x720 8UC4       1.08          0.56          1.93x
    1920x1080 8UC1      0.62          0.29          2.14x
    1920x1080 8UC4      2.43          1.11          2.19x

Geometric mean speedup: ~2.09x.

REDUCE_MIN, REDUCE_MAX, and REDUCE_SUM2 performance remains
essentially unchanged, indicating that the speedup is localized to
the optimized 8-bit row SUM path.