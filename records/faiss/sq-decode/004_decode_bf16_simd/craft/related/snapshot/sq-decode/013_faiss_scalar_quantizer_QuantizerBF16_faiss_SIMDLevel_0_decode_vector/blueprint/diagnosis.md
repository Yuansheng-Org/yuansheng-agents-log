# Diagnosis

Software: faiss
Testcase: sq-decode
Function: faiss\_scalar\_quantizer\_QuantizerBF16\_faiss\_SIMDLevel\_0\_decode\_vector
Verdict: `confirmed`

Summary: QuantizerBF16\<SIMDLevel::NONE\>::decode\_vector executes a fully scalar BF16-\>FP32 conversion loop (disasm 0x3896ee-0x3896fe: lhu; slliw a5,a5,0x10; sw a5,-4(a2); bne, 100% of function samples at the loop) because decode\_bf16\_simd in faiss/utils/bf16.h has only an \_\_AVX512F\_\_-guarded vector fast path and no RISC-V/RVV path, so on the rv64gcv build the scalar fallback decode\_bf16((uint32\_t(v))\<\<16) runs for every element.

## Findings
- `confirmed` (root cause, high) QuantizerBF16\<SIMDLevel::NONE\>::decode\_vector executes a fully scalar BF16-\>FP32 conversion loop (disasm 0x3896ee-0x3896fe: lhu; slliw a5,a5,0x10; sw a5,-4(a2); bne, 100% of function samples at the loop) because decode\_bf16\_simd in faiss/utils/bf16.h has only an \_\_AVX512F\_\_-guarded vector fast path and no RISC-V/RVV path, so on the rv64gcv build the scalar fallback decode\_bf16((uint32\_t(v))\<\<16) runs for every element.
  Reasoning: The annotate shows the entire function body is the 6-instruction scalar loop with zero v\*/th.v\* instructions, and the only hotspot (0x3896fe, 100%) is its back-branch. Source confirms decode\_bf16\_simd guards its SIMD widening path with \_\_AVX512F\_\_ only and otherwise falls to the per-element scalar decode\_bf16. Metadata confirms libfaiss.so Tag\_RISCV\_arch includes V 1.0 (zve32f/zve64d/zvl128b). The operation is a required, lane-independent precision conversion (zero-extend u16 to u32 then shift left 16), directly expressible with RVV intrinsics, matching the rvv\_precision\_conversion\_kernels pattern whose AVX512 twin (\_mm512\_cvtepu16\_epi32+\_mm512\_slli\_epi32) is literally in the same file.
- `probable` (contributing factor, medium) The scalar loop structure costs 6 instructions and one branch per converted element (u16 load, two pointer adds, slliw, f32 store, back-branch) with per-element loop control; with n=d elements and 2d input / 4d output bytes streamed, dynamic instructions grow ~6d and branches ~d for the decode, so no unrolling, widening vector load, or vector store amortizes loop control at any VLEN.
  Reasoning: The disassembly shows exactly 5 arithmetic/memory instructions plus the branch inside the one-element trip loop (cf97f02c). The whole-run perf stat branches=104,108,126 and L1\_dcache\_loads=137,292,750 (39aa17cb) are consistent with branch- and load-per-element patterns but cannot be isolated to this function across the mixed 4/6/8-bit/bf16/fp16 benchmark run, so the claim is deliberately limited to the per-element structural cost evidenced in disassembly. RVV would replace d branches/element and ~d\*6 instructions with ~4-5 vector instructions amortized over the vector length.

## Gaps
- `baseline_gap_measurement`: perf stat cpu\_cycle and instruction values both read ~9.22e18 (~2^63), i.e., counter overflow/saturation, so the reported IPC=1 and absolute cycle/instruction counts are artifacts and cannot bound the loop's cost.
- `baseline_gap_measurement`: The cpu-clock annotate for the target function captured only 1 sample (100% local, all on the back-branch 0x3896fe), which is too sparse to characterize per-instruction distribution within the loop.
- `source_context_gap_analysis`: d (the number of bf16 elements per decode\_vector call) used by the QT\_bf16 benchmark is not established from this evidence, so the per-call setup-vs-loop tradeoff for vectorization cannot be sized.
- `baseline_gap_measurement`: The function's share of the whole 0.267 s benchmark run is unknown; the run-level perf stat also covers the 4/6/8-bit/fp16 quantizer sub-benchmarks, so benchmark-level benefit cannot be attributed to this function.

## Actions
- `conditional`: Add an RVV fast path to decode\_bf16\_simd in faiss/utils/bf16.h, guarded by \_\_riscv\_vector, mirroring the existing \_\_AVX512F\_\_ widening sequence: per chunk load u16 with vle16.v, widen with unsigned vzext.vf2 (e32), shift left by 16 with vsll.vi, store with vse32.v, then fall through to the existing scalar tail for the remainder; leave the AVX512 path and scalar fallback intact.
- `conditional`: Replace the one-element-per-iteration loop-control structure with a vector main loop that carries a single back-branch per vector chunk plus a runtime-VL tail, cutting dynamic taken branches from ~d to ~d/vl per call; keep a minimal scalar unrolled tail for the last \<vl elements.
