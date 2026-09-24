# AI 补丁审核报告

- reviewId: `rev-bp-go-BenchmarkAddMulVVWW-026-01`
- patchCandidateId: `pc-bp-go-benchmarkaddmulvvww-words-100000-impl-asm-026`
- reviewer: opencode-craft-independent-reviewer（只读独立审核）
- blueprint: `.yuansheng/trace/go/BenchmarkAddMulVVWW_words_100000_impl_asm/026_internal_runtime_maps.memHashFallback/blueprint_go_BenchmarkAddMulVVWW_words_100000_impl_asm_026.json`
- 目标函数: `internal/runtime/maps.memHashFallback`（`runtime_hash64.go:21-65`），经 `r8/r4` → `readUnaligned64/32`（`runtime_alg.go:79-93`）

## 审核范围

被审对象为 `craft/patch.diff`，含三个文件：

1. `src/internal/runtime/maps/runtime_alg.go`：移除 `readUnaligned32/64`（连同 `internal/byteorder`
   导入），其余不变。
2. `src/internal/runtime/maps/runtime_alg_le.go`（新增，`//go:build !riscv64`）：
   `readUnaligned32/64` 的原实现（`byteorder.LEUint32/64` / BE 分支）。
3. `src/internal/runtime/maps/runtime_alg_riscv64.go`（新增，`//go:build riscv64`）：
   `readUnaligned32/64` 在 LE 且地址自然对齐时用单条原生 `lwu`/`ld`，否则回退 `byteorder.LEUint32/64`；
   riscv64 仅小端，故省略死代码 BE 分支。

审核依据：blueprint（`problem`、`rootCause`、`constraints.mustPreserve`、`diagnosis`）、
sibling diaglog（Phase 3 hot line `140b0`/`141a2`、Phase 4 `The fix`、Phase 5）、annotate、
以及 `craft/patch-candidate.json`。

## RISC-V 架构审核

- `node tools/yuansheng-craft-tools.js arch-scan craft/patch.diff` 结果：
  `{"archSpecific": false, "matches": []}` → 按 runbook `archReview.status = "not-applicable"`。
- 补充说明（不改变结论）：补丁不含汇编 / inline-asm / RVV intrinsic / ISA 特性宏；riscv64 分支仅用
  RV64I 基础 `lwu`/`ld` 做对齐快路径，未对齐回退字节拼装；无 `vsetvl`/`vtype`，无 x86/ARM 专有指令。
  （本补丁不改变 `memHashAESImplemented=false` 的事实，riscv64 仍走 fallback 路径。）

## 审核结果

**reviewResult: pass**

判定依据：

1. **语义保持（mustPreserve）**：
   - 对齐路径的原生 `lwu`/`ld` 零扩展后按小端解释，与 `byteorder.LEUint32/64` 拼装值逐位一致
     （riscv64 为小端）；哈希数值输出不变。
   - 每段读取宽度与语义宽度完全一致（`*[4]byte` / `*[8]byte`），不 over-read；未对齐时完全走原
     `byteorder.LEUint32/64`（含其 bounds check），可恢复 bounds-panic 行为不变。
   - BE 分支在 riscv64 为编译期死代码（`goarch.BigEndian=false`），省略不改变行为；非 riscv64
     保留原实现（`runtime_alg_le.go`），无跨架构行为变化。
2. **根因对应（本地可复现）**：以仓库自带工具链（`GOROOT=/root/go-workspace/go`，
   `GOOS=linux GOARCH=riscv64`）编译 `internal/runtime/maps` 并读取 `-S` 输出：
   - `readUnaligned32` 被内联进 `memHashFallback`，出现
     `ANDI $3, X10, X14; BNEZ X14, +12; MOVWU (X13), X14` —— 对齐时**单条原生 `lwu`**。
   - `readUnaligned64`（86B，LEAF）出现 `ANDI $7, X10, X11; BNEZ X11, +10; MOV (X9), X10`
     —— 对齐时**单条原生 `ld`**；未对齐保留 `byteorder` 字节拼装回退。
   即诊断的「8/4 字节读取付出 ~22/~10 条 lbu+shift+OR」在对齐路径上被消解。
3. **构建/格式**：`GOOS=linux GOARCH=riscv64 go build ./src/internal/runtime/maps/` 退出码 0；
   `gofmt -l src/internal/runtime/maps/` 为空。
4. **行为回归**：`go test internal/runtime/maps` 通过（`ok 0.340s`）；
   `go test -run 'TestMap|TestHash' runtime` 通过（`ok runtime 7.354s`）。

### 未验证项与权衡（已披露，不夸大收益）

- **收益以关键字地址对齐为前提**：`memHashFallback` 读取 `p` 与 `add(p, s-8)`；map 键的数据指针
  对齐取决于键类型/长度（例如字符串数据指针常对齐，而 `p+s-8` 可能不对齐）。仅对齐读取走单条
  `lwu`/`ld`，未对齐读取仍走字节拼装。blueprint 记载 `readUnaligned64` 命名契约显式许可未对齐读取，
  但真正「无条件折叠」会要求在任意地址发宽加载；Go riscv64 后端 `Config.unalignedOK=false` 不允许
  编译器主动引入未对齐访问。本补丁采用「调用点显式对齐判定 + 原生加载」的安全等价形态
  （与同批 023/024/025 及既有 030 一致）。
- **内联变化（权衡）**：改后 `readUnaligned64` 不再被内联进 `memHashFallback`（`memHashFallback`
  size 1460/leaf → 806 + 对 `readUnaligned64` 的调用；`readUnaligned64` 为 86B LEAF）。即对齐读取
  由「内联单条 ld」变为「leaf 调用 + 单条 ld」，未对齐读取由「内联字节拼装」变为「leaf 调用 +
  字节拼装」。该变化是否净收益取决于实际对齐命中率与调用开销，本机无目标硬件**未**实测。
- 本机无 SG2044 目标硬件，**未**重跑 benchmark；样本份额 2/264≈0.76%（link 进程构建阶段），
  blueprint `recommendToCraft=conditional`、`needsHumanReview=true`。本审核只声明「对齐路径下的
  字节拼装被折叠为单条 lwu/ld」，不声明 workload 级性能收益。

## 发现问题

无 critical / major / minor 问题。`findings: []`。

（「收益以对齐为前提」「64 位读取由内联变为 leaf 调用」为证据边界/权衡说明，不构成 `patch.diff`
缺陷锚点。）

## 幻觉自检

1. **技术精度 — PASS**：`readUnaligned32/64` 的字段宽度、对齐判定（`&3` / `&7`）、原生加载
   `*(*uint32/64)(unsafe.Pointer(&q[0]))` 与小端 `byteorder.LEUint32/64` 位一致；build tag 互斥
   （`!riscv64` / `riscv64`），非 riscv64 行为与原实现逐字一致（含 BE 分支）；riscv64 省略 BE 分支
   正确（riscv64 仅小端）。导入调整（`runtime_alg.go` 去掉 `byteorder`）后仍能编译。
2. **声明溯源 — PASS**：全部结论锚定于 `craft/patch.diff`、`026-...-annotate.txt`
   （hot line `140b0`/`141a2`）、blueprint/diaglog 字段，或本地可复现的
   `GOOS=linux GOARCH=riscv64 go build -gcflags=-S internal/runtime/maps` 输出
   （`MOVWU (X13),X14` 内联、`MOV (X9),X10` 单条 `ld`、`memHashFallback`/`readUnaligned64` size 变化）；
   无未执行却声称执行的验证。
3. **可解释性 — PASS**：改动为按架构拆分两个读取函数 + 对齐快路径；`runtime_alg.go` 仅删除函数与
   导入；无无关重构/格式噪音；`readUnaligned32/64` 仍为包内私有。
4. **内部一致性 — PASS**：`changedFiles` 与 `patch.diff` 三文件一致；`archReviewWarning` 与
   `arch-scan` 的 `archSpecific=false` 一致；plan/blueprint id 一致。
5. **安全 — PASS**：不改变哈希数值、内存布局与调用语义；不引入数据竞争；新增 `unsafe` 仅在 riscv64
   且对齐时生效，读宽与语义宽度一致；未对齐回退保证严格对齐核不产生非法访问；不涉及 AES/zk* 路径。

## 结论

补丁安全、最小、语义保持，并在本地可复现地把 `readUnaligned32/64`（`memHashFallback` 的
字节拼装来源）在自然对齐路径上折叠为单条原生 `lwu`/`ld`。`archReview = not-applicable`，幻觉自检
5 项全 PASS。判定 **pass**，进入 `done`。收益以关键字地址对齐为前提、未在目标硬件实测，
且 64 位读取由内联变为 leaf 调用，均已按 runbook 披露。