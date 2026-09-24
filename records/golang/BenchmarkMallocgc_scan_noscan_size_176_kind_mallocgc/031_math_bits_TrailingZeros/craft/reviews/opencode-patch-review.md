# AI 补丁审核报告

## 审核范围

- patchCandidateId: `pc-bp-go-benchmarkmallocgc_scan_noscan_size_176_kind_mallocgc-028`
- 蓝图: `.yuansheng/trace/go/BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc/028_nextFreeFast/blueprint_go_BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc_028.json`
- 补丁文件: `src/cmd/compile/internal/ssagen/intrinsics.go`（唯一改动文件）
- 根因（blueprint.rootCause.summary）: 默认 `GORISCV64=rva20u64` 构建未启用硬件 Zbb，`sys.TrailingZeros64` 以 10 条 De Bruijn 软件序列实现；编译器未把已接线的 runtime Zbb dispatch 扩展到 TrailingZeros64。

补丁内容：把已有的 `makeOnesCountRISCV64` 泛化为 `makeRISCV64ZbbDispatch`（同名逻辑，改名 + 注释），并把原先仅 `cfg.goriscv64 >= 22` 才注册的 `math/bits.TrailingZeros{8,16,32,64}` riscv64 intrinsic 改为无条件注册、复用该 dispatch helper。rva22u64+ 仍直接发 CTZ；rva20u64 走 `runtime.riscv64HasZbb` 运行时分支。

## RISC-V 架构审核

`arch-scan` 结果：`{"archSpecific": true, "matches": [{"rule":"riscv-macro","line":"+\t// makeRISCV64ZbbDispatch returns ..."}]}` —— 唯一命中来自新增注释里的 "RISCV64" 字样，补丁本身不含内联汇编/RVV intrinsic。

按蓝图 `source.targetHardware`（SOPHGO SG2044 / XuanTie C920v2）与 diaglog build-ISA 证据逐项核对：

- **扩展可用性**：目标硬件 cpuinfo 含 `zbb`（diaglog Phase 1 直接引用），因此新增的 `CTZ`/`CTZW` 指令在目标上可执行。
- **非 Zbb 目标安全**：CTZ 不在无条件路径上执行——生成的代码先读 `runtime.riscv64HasZbb`（`internal/cpu.RISCV64.HasZbb`，`src/runtime/proc.go:797` 初始化），为假时调用纯 Go `math/bits.TrailingZeros*` 回退。这与既有 `OnesCount` 的 Zbb dispatch（同一 helper）完全同构，不引入非法指令执行。
- **build ISA 兼容**：`cfg.goriscv64 >= 22` 分支保留原有直接 intrinsic 行为；rva20u64 分支只新增受运行时门控的指令，不改变无 Zbb 目标的语义。
- **实测确认**：用改后编译器交叉编译 riscv64 调用者，`go tool objdump` 显示 `MOVBU -2005(X9),X9`（读 riscv64HasZbb）→ `BEQZ` → `CTZ X10,X10` → `CALL math/bits.TrailingZeros64`（回退），证明 ISA 门控与回退路径均正确。

## 审核结果

`pass`。

补丁直接实现蓝图 `recommendedFirstAction` 的第二种落地方式（“在 intrinsics.go:962 处为 TrailingZeros64 补齐 Zbb 运行时 dispatch，仿照 makeOnesCountRISCV64”），且是语义保持的：`CTZ` 对 `x==0` 返回 XLEN=64，与 `TrailingZeros64(0)=64` 逐位一致（`(Ctz64 ...) => (CTZ ...)`，RISCV64.rules:221）；32/16/8 位经 `CTZW(ORI 1<<N)` 与 Go 语义一致。回退路径调用原纯 Go 实现，保证无 Zbb 硬件结果不变。

## 发现问题

无 `critical`/`major`/`minor` finding。补丁将 `TrailingZeros64` 的收益从“需全量 rva22u64 重建”降为“默认 rva20u64 构建即可在 Zbb 硬件受益”，因此不依赖工具链级重建决策。

## 幻觉自检

- 技术精度：PASS —— 改名、注册项、`cfg.goriscv64 >= 22` 分支、`ir.Syms.RISCV64HasZbb`、`runtime.riscv64HasZbb` 均来自实际源码；objdump 证据来自本次实测。
- 声明溯源：PASS —— “De Bruijn 软件实现”溯源到 blueprint/annotate；“CTZ 对 0 返回 64”溯源到 `_gen/RISCV64.rules:221` 与 RISC-V Zbb 语义；“既有 OnesCount dispatch”溯源到 `intrinsics.go:1153-1185`。
- 可解释性：PASS —— 补丁动机、门控机制、回退路径、验证命令均可复核。
- 内部一致性：PASS —— pass 结论与“无 finding”“archReview=passed”一致。
- 安全：PASS —— 未弱化任何内存序；未删除 fallback；未影响非 riscv64 目标；`TrailingZeros(0)` 语义保持。

## 结论

补丁安全、最小、语义保持，并在默认 rva20u64 构建下即可让 Zbb 硬件使用单指令 ctz，直接命中蓝图根因。构建验证：`GOOS=linux GOARCH=riscv64 go build cmd/compile/` 通过，`gofmt -l` 为空；交叉编译 riscv64 调用者并用 objdump 确认 `CTZ` + 运行时门控 + 回退路径。审核通过。
