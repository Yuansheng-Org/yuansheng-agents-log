# Diagnosis

Software: onednn
Testcase: softmax\_f32\_upstream
Function: dnnl\_impl\_cpu\_rv64\_anonymous\_namespace\_compute\_softmax\_f32\_rvv\_float\_const\_float\_long\_bool\_7bb41bca
Verdict: `confirmed`

Summary: In softmax\_f32\_upstream, compute\_softmax\_f32\_rvv executes the scalar fallback exp/sum loop (d25a30-d25a54) that calls expf@plt once per element and accumulates sum\_exp through a serial fadd chain with a per-iteration stack store; this loop accounts for ~70% of the function's samples (36% d25a38, 20% d25a40, 13% d25a50) because the jit\_exp gate (axis\_size\>=16, inner\_size\<=8192) keeps the existing RVV exp\_sub\_sum JIT kernel unreachable for the measured shape.

## Findings
- `confirmed` (root cause, high) In softmax\_f32\_upstream, compute\_softmax\_f32\_rvv executes the scalar fallback exp/sum loop (d25a30-d25a54) that calls expf@plt once per element and accumulates sum\_exp through a serial fadd chain with a per-iteration stack store; this loop accounts for ~70% of the function's samples (36% d25a38, 20% d25a40, 13% d25a50) because the jit\_exp gate (axis\_size\>=16, inner\_size\<=8192) keeps the existing RVV exp\_sub\_sum JIT kernel unreachable for the measured shape.
  Reasoning: Profile hotspots concentrate in the non-jit scalar exp/sum loop: d25a38 36%, d25a40 20% (auipc for expf@plt), d25a50 13%, d25a4c 1%. Source shows this loop is the else branch of jit\_exp, which is gated off by axis\_size\>=16 && inner\_size\<=8192 (exp\_jit\_min\_len=16). The benchmark executes expf@plt per element with a serialized sum\_exp dependency that stores to stack each iteration, producing ~70% of the function's samples. A validated RVV JIT kernel (jit\_rvv\_softmax\_f32\_exp\_sub\_sum) exists at c77dc6 in the same binary but is not reached for this shape.
- `confirmed` (contributing factor, high) The Stage-1 max reduction in compute\_softmax\_f32\_rvv is always a scalar value-only maximum loop (d25838-d25850; beqz d25844 at 15%) because the f32 softmax path, unlike the f16 path, provides no vector reduce-max kernel; it is a secondary scalar full scan that contributes roughly 15% of the function's samples.
  Reasoning: The disassembly d25838-d25850 is a scalar value-only max reduction: flw fa5, flt.s, beqz (15% at d25844), fmv.s select, addi/bne loop control. It always runs regardless of jit\_exp because the f32 path has no JIT/vector reduce-max (only the f16 path has jit\_rvv\_softmax\_f16\_reduce\_max). It is a contributing, not primary, cost: 15%+ of samples and a second scalar full scan in the softmax pipeline.

## Gaps
- `baseline_gap_measurement`: build\_isa metadata is null, and the sampled benchdnn-elf-A attribute string lists rv64i2p1... without v, so it cannot be confirmed whether the translation unit containing compute\_softmax\_f32\_rvv was compiled with V; the scalar loops may partly stem from a non-V build rather than gating.
- `source_context_gap_analysis`: The exact softmax\_f32\_upstream benchdnn shape (axis\_size and inner\_size) was not located in tests/benchdnn/inputs (no literal 'upstream' match), so the jit\_exp gate decision (axis\_size\>=16 && inner\_size\<=8192) is inferred from the profile showing the scalar path, not confirmed from the test config.

## Actions
- `conditional`: In execute\_forward (src/cpu/rv64/rvv\_softmax.cpp:217-220) extend the jit\_exp predicate so the validated jit\_rvv\_softmax\_f32\_exp\_sub\_sum kernel (present at c77dc6) is selected for the measured shape instead of the scalar else-branch loop; where the JIT is not reachable, replace the scalar exp/sum loop in compute\_softmax\_f32\_rvv with an RVV intrinsic kernel (vsetvl + vle32 + vfsub + project-validated vector exp + vfredusum with scalar seed) following the rvv\_normalization\_kernels Stage-2/Stage-3 dataflow, retaining the scalar path only below an empirically measured crossover.
- `conditional`: Add an f32 vector reduce-max path for Stage 1 in compute\_softmax\_f32\_rvv, mirroring jit\_rvv\_softmax\_f16\_reduce\_max (jit\_rvv\_softmax\_kernel.cpp) or using RVV intrinsics: per-chunk vsetvl, vle32, vfredmax\_vs seeded with the current scalar max (starting -INFINITY), NaN lanes merged to the seed to reproduce the scalar \`\> max\_val\` contract, then vfmv\_f\_s to the scalar accumulator; keep the scalar loop only for lengths below a measured crossover.
