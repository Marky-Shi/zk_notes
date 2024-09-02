## Groth16
grath16 零知识简洁非交互知识证明（zk-snarks）

1. **电路描述**：
   1. 所有电路都有一个专业术语 Relation（变量之间的关系描述）
   2. 电路通常使用R1CS(Rank-1 Constraint System) 语言进行描述，类似于线性方程组。
   3. 需要将 R1CS 描述的电路转换为QAP(Quadratic Arithmetic Program)
2. **QAP 转化**：
   1. 在R1CS 和 QAP 之间的转化称为 Reduction
   2. QAP描述使用**拉格朗日基函数**，而不是多项式系数，以更好地适应Groth16算法的要求。
3. **Domain**选择：
   1. 选择合适的domain(域)对计算性能至关重要
   2. domain的选择影响拉格朗日插值和**FFT/iFFT**的计算。
   3. 不同的domain适用于不同的输入个数。
4. **Setup 计算**
   1. 生成 证明私钥（PK） 验证私钥（VK）
5. **Prove 计算**
   1. 在给定 **witness** 或 **statement** 的情况下，生成证明。
   2. 涉及多次FFT/iFFT 计算和 MultiExp（多点乘法）操作
6. **Verify 计算**
   1. 在已知证明和验证密钥的情况下，通过配对函数验证证明的正确性。

[零知识证明 - Groth16计算详解 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/2019/12/19/zkp-Groth16)

[零知识证明 - libsnark源代码分析 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/2019/08/15/libsnark-source/)
