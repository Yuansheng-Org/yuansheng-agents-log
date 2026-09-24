# Diagnosis

Software: onednn
Testcase: softmax\_f16\_upstream
Function: dnnl\_impl\_cpu\_rv64\_jit\_rvv\_softmax\_f16\_scatter\_dnnl\_impl\_float16\_t\_const\_dnnl\_impl\_float16\_t
Verdict: `confirmed`

Summary: jit\_rvv\_softmax\_f16\_scatter spends the dominant within-function sample share (97.9%) in the thread-safe static-local guard check of dispatch\_f16\_strided\<false\>() — including a per-call fence r,rw — which is redundant synchronization in the uncontended benchmark context, executed once per softmax block via rvv\_softmax.cpp:321.

## Findings
- `confirmed` (root cause, high) jit\_rvv\_softmax\_f16\_scatter spends the dominant within-function sample share (97.9%) in the thread-safe static-local guard check of dispatch\_f16\_strided\<false\>() — including a per-call fence r,rw — which is redundant synchronization in the uncontended benchmark context, executed once per softmax block via rvv\_softmax.cpp:321.
  Reasoning: 140/143 samples (97.9%) concentrate at c73a3c in the thread-safe-static guard block (lb a5,0(a5); fence r,rw; zext.b a5,a5; beqz) of dispatch\_f16\_strided\<false\>(). Source (jit\_rvv\_softmax\_kernel.cpp:55-60, 171-177) shows jit\_rvv\_softmax\_f16\_scatter builds a call\_params\_t and calls dispatch\_f16\_strided\<false\>(&p), which constructs a function-local static kernel (thread-safe guard + fence) and then indirectly calls it via operator(). rvv\_softmax.cpp:321 calls the scatter once per softmax block inside a loop over outer\_size\*inner\_size blocks, so the guard+decorated-fence sequence executes once per block per execution. The JIT kernel is constructed exactly once (guard byte constant afterward), and benchdnn softmax\_f16\_upstream is a single-threaded benchmark, so no concurrent writer exists; the per-call fence r,rw and guard check are redundant synchronization in an uncontended context. Pattern profile signal 'fence in a provably nonconcurrent path' and mechanism 'fence drains the pipeline / high frequency magnifies small costs' match the observed hotspot and the 0.80 IPC / 0.184% branch-miss counters.

## Gaps
- `baseline_gap_measurement`: Sampling IP precision (precise\_ip) is not reported in the annotate or metadata; without it the 97.9% single-instruction share at c73a3c may include skid from the preceding fence r,rw and the out-of-line JIT call, so the guard block's true cost is an interval-level attribution.
- `source_context_gap_analysis`: The JIT-emitted RVV scatter kernel body is out-of-line and not covered by this annotate; the annotate captures only the host wrapper symbol, so the relative cost of the wrapper vs the JIT loop cannot be directly measured from this single-function annotate.
- `evidence_gap_confidence`: The uncontended-context proof relies on benchdnn's default single-threaded execution; no explicit thread-count or parallel\_nd concurrency evidence for the softmax\_f16\_upstream run is recorded in the input, so the no-concurrent-writer claim is indirect.
- `source_context_gap_missing_source`: Local source evidence is unavailable at src/cpu/rv64/jit\_rv64\_softmax.cpp.

## Actions
- `conditional`: Hoist the strided JIT kernel construction and dispatch out of the per-block hot path. In rvv\_softmax.cpp, mirror the existing affine\_kernel\_ member pattern: add a member (e.g., std::unique\_ptr\<jit\_rvv\_softmax\_f16\_strided\_kernel\_t\> strided\_kernel\_) created once in the rvv\_softmax\_fwd\_t constructor (single-threaded primitive creation), and in execute\_forward call (\*strided\_kernel\_)(&p) directly for the f16 gather/scatter instead of dispatch\_f16\_strided\<false\>(&p) / dispatch\_f16\_strided\<true\>(&p). This removes the per-block thread-safe-static guard sequence (lb guard; fence r,rw; zext.b; beqz; indirect call) that currently executes for every block; the kernel object is immutable after construction so no per-call synchronization is needed. Keep the jit\_rvv\_softmax\_f16\_gather/scatter free functions and dispatch\_f16\_strided helper intact for any other callers.
