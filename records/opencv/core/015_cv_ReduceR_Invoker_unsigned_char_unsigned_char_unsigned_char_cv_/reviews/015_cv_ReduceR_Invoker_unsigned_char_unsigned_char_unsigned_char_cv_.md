Add an RVV 1.0 kernel for cv::reduce with dim=0 and CV_32F input/output
through the CPU dispatch mechanism.

Process four source rows per vector iteration to reduce intermediate
buffer traffic while preserving source-row accumulation order. Use
LMUL=4 with dynamic vector lengths for tails, and defer output writes
until all input rows have been read to support overlapping matrices.

The implementation is VLEN-agnostic.

Functional test:

    ./bin/opencv_test_core \
        --gtest_filter='*Reduce*:*reduce*' \
        --test_threads=1

Performance test:

    ./bin/opencv_perf_core \
        --gtest_filter='*reduceR*' \
        --perf_threads=1

Performance was measured on SpacemiT K3 with VLEN=256, GCC 14.3.0,
a Release build, and a single thread.

CV_32FC1 REDUCE_SUM2 median time (ms), 10 samples per case:

    Size         Baseline    Patched    Speedup
    640x480         0.20       0.08       2.50x
    1280x720        0.64       0.29       2.21x
    1920x1080       1.33       1.05       1.27x

Speedups are approximate and calculated from the rounded benchmark
output.