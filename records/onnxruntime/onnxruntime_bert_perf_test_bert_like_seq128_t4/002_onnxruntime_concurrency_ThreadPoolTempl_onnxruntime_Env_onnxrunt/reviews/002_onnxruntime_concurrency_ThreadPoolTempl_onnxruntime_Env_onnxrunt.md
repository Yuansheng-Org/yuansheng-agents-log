

### 构建命令

```bash
cd /home/yw/onnxruntime
/home/yw/.venv/bin/python tools/ci_build/build.py \
  --config Release \
  --build_dir build/riscv_patch_validation \
  --update --build --parallel 8 \
  --target onnxruntime_perf_test \
  --skip_tests --skip_pip_install --skip_submodule_sync \
  --no_sve --enable_rvv --allow_running_as_root \
  --cmake_generator Ninja
```



- 正式命令（基线和补丁只替换 `--perf_test` 二进制路径）：

```bash
nice -n -20 taskset -c 0-7 /usr/bin/python3 \
  tools/perftest/benchmark_spin_settings.py \
  --perf_test <onnxruntime_perf_test> \
  --model onnxruntime/test/testdata/squeezenet/model.onnx \
  --intra_op 8 --duration 3 --repeats 1 --configs default
```

该命令独立重复 5 次。以平均推理时延的 5 次中位数为主指标，吞吐中位数为辅。



汇总：

| 指标               |      基线 |    补丁后 |           相对变化 |
| ------------------ | --------: | --------: | -----------------: |
| 平均推理时延中位数 |  6.819 ms |  6.736 ms | -1.22%（越低越好） |
| 吞吐中位数         | 146.6 IPS | 148.4 IPS | +1.23%（越高越好） |