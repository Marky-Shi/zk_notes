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



### setup

选择一个椭圆曲线 
$$
E
$$
及其循环群
$$
G_1、G_2、G_T
$$
 和 ，具有素数阶 
$$
p   
$$
。

选取随机生成元 
$$
g_1 \in G_1  ||   g_2 \in G_2
$$
选择双线性配对
$$
e: G_1 \times G_2 \to G_T
$$
配对的性质：
$$
e(g_1^a, g_2^b) = e(g_1, g_2)^{ab}
$$

#### **公共参考字符串（CRS）**

* 证明密钥 
  $$
  Pk = \{g^\alpha, g^\beta, g^\delta, \{g^{a_i(\tau)}, g^{\beta a_i(\tau)}, h^{b_i(\tau)}, g^{c_i(\tau)}\}_{i \in [m]}, \{g^{\tau^i}\}_{i \in [d]}\}
  $$
  包含多项式评估和随机数相关的值。

* 验证密钥 
  $$
  Vk = \{g^\gamma, h^\beta, e(g^\alpha, h^\beta), \{h^{\tau^i}\}_{i \in [d]}\}
  $$
   包含验证需要的常数，如 
  $$
  e(g^\alpha, h^\beta)
  $$
  。

其中m是约束数量，d是多项式的阶数。



### R1CS

R1CS 将计算分解成一系列形如
$$
(a · w) * (b · w) = (c · w)
$$
用于表示约束。这里的 w 是 witness 向量，包含所有变量，包括：

​	•	常量（通常为 1 ，表示偏移量）；

​	•	输入变量；

​	•	中间变量；

​	•	输出变量。

对于 
$$
z = x \cdot y
$$
，R1CS 的表示：
$$
w = \begin{pmatrix} 1 \\ x \\ y \\ z \end{pmatrix}, \quad

a = \begin{pmatrix} 0 \\ 1 \\ 0 \\ 0 \end{pmatrix}, \quad

b = \begin{pmatrix} 0 \\ 0 \\ 1 \\ 0 \end{pmatrix}, \quad

c = \begin{pmatrix} 0 \\ 0 \\ 0 \\ 1 \end{pmatrix}
$$


约束矩阵形式正确，且约束等式为：
$$
(a \cdot w) \cdot (b \cdot w) = (c \cdot w) \implies x \cdot y = z
$$


> 对于更复杂的计算，例如 z = x^2 + y ，需要多个约束。R1CS 将每一步分解为独立的乘法约束。
>
> 生成形如 
> $$
> n \cdot m
> $$
> 格式的矩阵，其中n 行是 约束的数量，m 列是witness 的数量



### **QAP (Quadratic Arithmetic Program)**

QAP 将 R1CS 转换为多项式形式。对于每个约束 i ，定义多项式
$$
a_i(x), b_i(x), c_i(x)
$$
 ，它们的值满足：


$$
a_i(i) = a \cdot w, \quad b_i(i) = b \cdot w, \quad c_i(i) = c \cdot w
$$


这些多项式用于插值，将离散约束点转化为多项式。

全局多项式表示为：


$$
A(x) = \sum_{i=1}^{m} w_i \cdot a_i(x), \quad

B(x) = \sum_{i=1}^{m} w_i \cdot b_i(x), \quad

C(x) = \sum_{i=1}^{m} w_i \cdot c_i(x)
$$
这里 
$$
w_i
$$
是 witness 向量中的第 i 个值，它结合 
$$
a_i(x), b_i(x), c_i(x)
$$
来计算全局多项式。

目标多项式 t(x) 定义为：
$$
t(x) = \prod_{i=1}^{m} (x - i)
$$
如果所有约束都满足，则存在一个多项式 h(x)，使得：
$$
A(x)B(x) - C(x) = h(x)t(x)
$$
h(x) 是商多项式，表明约束一致性。



### prove 

* **计算 witness 多项式 w(x)**：根据 witness 向量 w 计算。实际上，计算的是 
  $$
  A(τ), B(τ), C(τ)
  $$
  的值，这些值可以表示为 
  $$
  Σᵢwᵢaᵢ(τ), Σᵢwᵢbᵢ(τ), Σᵢwᵢcᵢ(τ)
  $$
  。

* **计算 h(x)**：计算 h(x) 使得 
  $$
  A(x)B(x) - C(x) = h(x)t(x)
  $$
  。

* **选择随机数**：选择随机数 
  $$
  r, s ∈ z_p
  $$
  。

* **计算证明 π (Proof)**：


$$
\pi_a = g^{A(\tau)} \cdot g^{r \cdot \delta}, \quad
\pi_b = h^{B(\tau)} \cdot h^{s \cdot \delta}, \quad
\pi_c = g^{C(\tau)} \cdot g^{r \cdot \alpha} \cdot g^{s \cdot \beta}
$$

### verify

验证公式
$$
e(\pi_a, \pi_b) = e(g^\alpha, h^\beta) \cdot e(\pi_c, g^\gamma)
$$




* 验证者检查 A(x), B(x), C(x) 是否满足 h(x)t(x) 的约束。
* 双线性配对用于高效验证。





