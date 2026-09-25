# AI 补丁审核报告

## 审核范围
- 蓝图：bp-faiss-sq-distance-001（DCTemplate<Codec6bit,Scaling1,L2>::query_to_code 每迭代 vfredosum 归约 + LMUL=1）
- 补丁：sq-rvv.cpp 通用 RVV DCTemplate（持久向量累加器 + 单次归约，LMUL=8）

## RISC-V 架构审核
- 指令均 RVV 1.0；运行时 VL；LMUL=8 且 live 集合在 32 寄存器预算内；4bit-uniform 既有特化不受影响；无 x86/ARM 指令

## 审核结果：pass

## 发现问题
- F1（suggestion）：向量归约顺序与标量不同，距离最低位差异需回归标定。

## 幻觉自检
- 技术精度 PASS / 声明溯源 PASS / 可解释性 PASS / 内部一致性 PASS / 安全 PASS

## 结论
补丁准确消除根因（每迭代标量往返归约 + LMUL 欠利用），转为持久向量累加 + 单次归约 + LMUL=8；约束保持，无阻塞缺陷。
