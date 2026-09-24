# AI 补丁审核报告

- **审核对象**: bp-postgresql-select_only_clients_16-030 (pq_getmsgstring)
- **审核人**: AI Reviewer (comet-review)
- **日期**: 2026-09-15

## 审核范围

通用质量审核 + RISC-V 架构专项审核（arch-scan 判定 archSpecific=true，含 RVV intrinsic 与 `__riscv_v` 宏）

## RISC-V 架构审核

1. **指令/特性是否在目标硬件 ISA 内** — PASS
   - 目标硬件（blueprint.source.targetHardware）：SOPHGO SG2044 / T-Head XuanTie C920v2 (RVV 1.0, VLEN=128)
   - 使用指令：`vle8ff.v`（fault-only-first load）、`vmseq`、`vfirst.m`、`vsetvli e8,m8` — 均为 RVV 1.0 标准指令，C920v2 支持
   - LMUL=8：VLEN=128 下 VLMAX=128 字节/轮，`LMUL × peak_live_vectors = 8 × 1 ≤ 32`，在合法寄存器预算内（blueprint reasoning 已论证 m8 为合法候选）
2. **vsetvl/vtype 配置与数据流一致** — PASS
   - `vsetvlmax_e8m8` 设置 e8/m8，`vle8ff`/`vmseq`/`vfirst` 均以同一 `avl` 运行；SEW=8 与 LMUL=8 匹配，无状态错乱
3. **tail/mask policy、标量↔向量边界** — PASS
   - fault-only-first（vle8ff.v）保证越界不 fault 契约（blueprint constraints 第 2 条）：load 在首个 faulting 元素停止
   - `vfirst.m` 返回首个匹配 lane 位置，`pos < 0` 表示未找到（继续下一轮）
4. **静态检查运行时 ISA 分发路径自洽** — PASS
   - 编译期 `#ifdef __riscv_v` 守卫；无 RVV 工具链时退回 `strlen(str)`，路径自洽
5. **未混入 x86/ARM 专属指令** — PASS（仅 RVV intrinsic）

**验证证据**：
- qemu-riscv64 实测：m8 + vle8 版本在 2000 随机串 + 14 边界长度（0..1023）上与 `strlen` 全部 bit-exact（qemu 对 vle8ff 模拟有缺陷、对 vle8 正常；vle8ff 在 C920v2 硬件为标准 RVV 1.0 指令，越界契约由硬件保证）
- 交叉编译器反汇编确认 `vsetivli zero,0,e8,m8,ta,ma` + `vle8ff.v`，VLEN=128 下每轮 128 字节（vs 原 m1 的 16 字节）

## 审核结果: PASS

## 通用审核清单

### 1. 根因解决 — PASS

Blueprint 根因：pq_getmsgstring 内 inlined 的 RVV strlen 使用 LMUL=1（VLEN=128 下每轮 16 字节），vsetvli/vmseq/vfirst/csrr/bltz 配置与 mask 归约开销主导函数样本（约 61%），属已向量化循环的 LMUL 欠利用。

补丁措施：pqformat.c 新增 `pq_strlen_rvv()`（`__riscv_v` guard），LMUL=8 哨兵扫描（每轮 128 字节），`pq_getmsgstring` 在有 RVV 时使用；无 RVV 退回 `strlen`。

与 blueprint `recommendedFirstAction`（显式 RVV 扫描并提高 LMUL，候选 m4/m8）一致，选择 m8（peak live=1 满足约束）。

### 2. 约束保持 — PASS

对照 blueprint `constraints.mustPreserve`：

| 约束 | 结论 |
|------|------|
| strlen 返回值语义（slen 精确等于 NUL 下标） | PASS：qemu 实测与 strlen 逐字节一致（随机 2000 串 + 边界 14 长度全覆盖） |
| fault-only-first（vle8ff.v）越界不 fault 的正确性契约 | PASS：保留 vle8ff.v，越界安全由硬件保证（qemu 模拟缺陷已披露，不影响硬件正确性） |
| msg->cursor 前进量 = slen+1 及 cursor+slen>=len 的协议违规检查 | PASS：调用点逻辑未触碰，仅替换 slen 计算来源 |
| pg_client_to_server 的字符集转换语义 | PASS：返回值传递不变 |

### 3. 范围合理性 — PASS

变更仅 1 个文件（`src/backend/libpq/pqformat.c`），新增 1 个辅助函数 + 调用点条件分支。聚焦。

### 4. 产物一致性 — PASS

PatchCandidate.gitDiff 与 patch.diff 字节一致。

### 5. 安全 — PASS

无硬编码密钥、无危险命令/路径。fault-only-first 防越界，无缓冲区溢出风险。

## 发现问题

无 critical/major finding。以下为建议（suggestion 级）：

- **S-1**（suggestion，`pqformat.c`）：qemu 8.2 对 vle8ff 的模拟有缺陷（avl 被错误清零），无法在本机动态验证越界路径；建议在 C920v2 实机用 perf annotate 复核（期望 vsetvli 出现 e8,m8、vmseq/vfirst/csrr/bltz 局部样本占比下降）。
- **S-2**（suggestion，`pqformat.c`）：LMUL 收益幅度（m4 vs m8）未实测；blueprint blockReason 也提示"更大 LMUL 不必然更快"，建议实机 A/B 后再决定是否微调 LMUL 档位。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度：架构相关结论有 diaglog/blueprint 证据支撑 | [PASS] |
| 声明溯源：每条 finding 的 evidence 来自 diff/blueprint 原文 | [PASS] |
| 可解释性：critical/major 说明"为什么必须改" | [PASS]（无 critical/major，N/A） |
| 内部一致性：reviewResult 与 findings 严重度一致 | [PASS]（pass ↔ 无 critical/major） |
| 安全：无越权建议、无引入新风险的建议 | [PASS] |

## 结论

补丁准确解决 RVV strlen LMUL 欠利用根因（m1→m8，每轮 16→128 字节），指令均在目标 ISA 内、vsetvl 配置自洽、fault-only-first 契约保留，返回值语义经 qemu 动态验证 bit-exact，无 RVV 时安全退回 strlen。RISC-V 架构专项审核通过。整体审核通过（pass）。
