## zk-stark

zk-STARK（Zero-Knowledge Succinct Transparent Argument of Knowledge）并不直接依赖于哈希碰撞。它的证明机制基于多项式和线性代数，尤其是FRI（Fast Reed-Solomon Interactive Oracle Proof）子协议，用于确保**证明的完整性和正确性**。FRI阶段涉及对多项式的度进行**递归约束**，而不是寻找或避免哈希碰撞。

在STARK中，**计算过程被转换成一组多项式方程，然后通过傅里叶变换进行处理**。这个过程可以产生一个简洁的证明，该证明可以被任何人验证，而且不需要信任设置（与zk-SNARKs不同，后者通常需要一个初始化的信任设置阶段）。虽然哈希函数在STARKs中仍然有作用，例如用于随机化或者作为部分输入的指纹，但它们并不是核心的证明机制。

所以，STARK的核心不是基于哈希碰撞的，而是基于**多项式和代数结构**来构建透明和高效的零知识证明。

### 底层实现：

#### 多项式承诺和多项式校验：

ZK-Starks 将计算转换为多项式，利用多项式在有限域上的运算性质，通过多项式承诺和验证来实现证明的生成和验证。这种方法的优势在于多项式运算的**高效性和确定性**。

* **计算过程被转换为一组多项式方程**，这些方程描述了计算的每一个步骤（计算过程则是通过对转换后的多项式进行算术运算来模拟进行）。 这些多项式计算通常在一个**大的素数**有限域上进行，用有限域的性质来简化计算并确保计算的确定性。
* 使用多项式的特性进行编码，通过**快速傅里叶变换（FFT）**处理这些多项式。
  * FFT 用于多项式的高效乘法和插值，在Stark 中加速设计的关键。
* 通过多项式的低度性和约束性来保证计算的正确性。

#### FRI 子协议

* FRI 是 ZK-STARK 中用于多项式检验的核心协议。
* 用于验证一个给定的多项式是**否具有低度**。
* 它通过递归地将多项式分解并检验其低阶系数，以此来检测潜在的欺诈。
* 这一过程**无需可信设置**，保证了系统的**透明性**。

**ZK-STARKs 通过 FRI 和其他优化技术实现了高效的验证过程。FFT（Fast Fourier Transform） 的使用使多项式运算更加高效，而 GPU 加速进一步提升了计算性能。这些优化使得 ZK-STARKs 能够处理大规模数据和复杂计算，同时保持高效的证明生成和验证过程。**



### 步骤：

1. 将计算表示为多项式约束/方程系统。这可能是**代数中间表示 (AIR) 或 Plonkish 算术化**
2. 根据计算过程生成一个执行轨迹，执行轨迹是一个矩阵，每一行代表一个计算步骤，每一列代表一个寄存器或状态变量。
3. 多项式插值，将执行轨迹的每一列看作是一个多项式的取值点集，通过插值算法（如**拉格朗日插值**）构建出这些多项式。
4. 多项式评估 和 merkle tree
   1. 将这些多项式在更大的域 D0 上进行评估
   2. 使用评估结果构建一棵默克尔树，确保数据的完整性
5. 约束检查
   1. 验证多项式是否满足之前定义的约束条件。
   2. 通过将多项式**除以一个特定的“消失多项式”（vanishing polynomial）来检查约束是否成立**。如果结果也是一个多项式，则约束成立。
6. 线性组合和约束检查
   1. 如果有多个多项式，则随机选择一些系数，对多项式进行线性组合。
   2. 检查线性组合后的结果是否仍是一个多项式。如果是的，则有很高的概率所有多项式都满足约束
7. 最终验证
   1. 将步骤6的结果在 D0 上进行评估，并构建另一棵默克尔树。
   2. 使用 FRI 协议验证这些评估结果是否对应一个低度多项式，以确保没有作弊。

> tips 
>
> 消失多项式（Vanishing polynomials）： 表示一个约束条件不满足的区域。一组约束，对每个约束都构建一个多项式，当且仅当**这个多项式约束不满足时，取值为0**。这个多项式就叫做**消失多项式**
>
> 用途：
>
> 1. **约束检查：** 如果一个多项式能够**被消失多项式整除**，那么它就**满足了对应的约束条件**。
> 2. **零知识证明：** 在零知识证明系统中，消失多项式可以用来验证计算的正确性。如果一个计算的结果**不满足约束条件**，那么对应的多项式就**不能**被消失多项式整除。
>
> 为什么满足约束时，多项式能被消失多项式整除？
>
> * **多项式的整除性：** 如果一个多项式 `f(x)` 能被另一个多项式 `g(x)` 整除，那么 `f(x)` 的根一定包含 `g(x)` 的所有根。
> * **消失多项式的根：** 消失多项式的根就是约束不满足的点。
> * **满足约束的多项式：** 如果一个多项式 `f(x)` 满足约束，那么它的**根一定不在消失多项式的根集上**。因此，`f(x)` 不能被消失多项式整除。

### 梅森素域

[Circle STARKs: Part I, Mersenne - ZKSECURITY](https://www.zksecurity.xyz/blog/posts/circle-starks-1/)

梅森素域是基于梅森素数构建的有限域。

* **梅森素数:** 形如 2^p - 1 的素数，其中 p 本身也是一个素数。例如，3, 7, 31 都是梅森素数。
* **有限域:** 一个有限集合，在这个集合上定义了加法、减法、乘法和除法运算，并且这些运算满足一定的规则。

为什么选用梅森素域

1. **高效的算术运算：**
   - **模二的幂运算：** 梅森素域上的运算本质上是模 2^p 的运算。计算机硬件对二进制运算有天然的优势，这使得在梅森素域上进行算术运算非常高效。
   - **移位操作：** 在梅森素域上，乘法运算可以转化为移位和异或操作，而这些操作在计算机中是极其快速的。
2. **特殊的数学性质：**
   - **有限域的良好性质：** 梅森素域作为一个有限域，具有许多良好的代数性质，这为构建高效的密码学协议提供了坚实的基础。
   - **与椭圆曲线的兼容性：** 梅森素域上的椭圆曲线具有特殊的性质，可以用于构建高效的椭圆曲线密码系统。



### Hash function

**哈希函数的作用**：

- 虽然 ZK-STARK 的核心不是基于哈希碰撞，但哈希函数在系统中仍然扮演重要角色，例如用于**随机化和生成指纹。**
- 哈希函数帮助增强系统的安全性和随机性，但不作为核心证明机制。

####  随机预言机（Random Oracle Model）：

- 虽然 ZK-STARKs 尽量减少对外部随机预言机的依赖，但在生成证明过程中会用到随机性。例如，在 FRI 中用于生成挑战，以确保证明的不可伪造性和安全性。

#### Turbo/GPU 优化

* 在生成证明时，zk-STARKs的实现经常利用并行计算技术，如GPU加速，以及算法上的优化（如Turbo Proofs），来减少计算时间和资源消耗。

####  透明性和无需可信设置

* zk-STARKs的设计确保了证明过程不需要任何信任设置阶段，所有必要的参数都可以公开生成，增加了系统的透明度和去中心化特性。



### 如何实现隐私保护

1. **零知识特性**：
   - ZK-STARK 提供零知识证明，即证明者可以证明某个声明的正确性而不泄露任何额外信息。
   - 证明的生成和验证过程保证了隐私性，验证者无法从证明中提取出计算的任何细节。
2. **非交互性和透明性**：
   - ZK-STARK 是非交互的，意味着证明者可以独立生成证明，验证者无需与证明者互动即可验证证明。
   - 透明性保证了证明过程不依赖任何可信第三方，进一步增强了隐私和安全性。
3. **高效性**：
   - ZK-STARK 的证明生成和验证过程相对高效，适合大规模数据和复杂计算的场景。
   - 通过多项式和线性代数的优化，ZK-STARK 可以生成简洁且高效的证明。



#### 总结： 

ZK-STARK 通过将**计算转换成多项式方程**，并利用 **FRI 子协议进行多项式检验**，实现了高效、透明和零知识的证明过程。虽然哈希函数在系统中有辅助作用，但 ZK-STARK 的核心机制是基于**多项式和代数结构**来构建的。这使得 ZK-STARK 能够在保证隐私和安全性的同时，提供高效的证明和验证方法。

### stark  snark的区别

zk-SNARK（Zero-Knowledge Succinct Non-Interactive Argument of Knowledge）和zk-STARK（Zero-Knowledge Scalable Transparent Argument of Knowledge）都是零知识证明（Zero-Knowledge Proof, ZKP）的两种不同实现，它们允许一方（证明者）向另一方（验证者）证明某些知识是真实的，而无需透露任何具体信息。以下是它们的主要区别：

1. **信任设置（Trust Setup）**：
   - **zk-SNARKs**：需要一个初始的可信设置（trusted setup）阶段，这个阶段可能产生一些潜在的脆弱性，如如果参与者之一是恶意的，可能会破坏整个系统的安全性。
   - **zk-STARKs**：无需可信设置，因此减少了信任假设，提供了更强的安全性保证。
2. **证明的长度和效率**：
   - **zk-SNARKs**：生成的证明通常较短，但涉及复杂的数学结构，包括非线性的伽罗华域运算，这可能使证明的构造和验证复杂。
   - **zk-STARKs**：证明通常比zk-SNARKs更大，但验证过程更为直接和透明，基于**线性代数和多项式**，更容易理解和实现。
3. **可验证性和透明度**：
   - **zk-STARKs**：因为没有信任设置，它们是**透明的**，意味着任何人都可以检查和验证证明的每个步骤，增加了公众的可审计性。
   - **zk-SNARKs**：由于其内在的复杂性，证明的验证过程不如zk-STARKs透明。
4. **计算需求**：
   - **zk-STARKs**：通常需要更多的计算资源来生成证明，但验证过程相对更简单和高效。
   - **zk-SNARKs**：生成证明可能更快，但验证过程可能需要更多步骤。
5. **后量子安全性**：
   - **zk-STARKs**：被认为在量子计算机时代可能更具抵抗力，因为它们不依赖于易受量子攻击的数学结构。
   - **zk-SNARKs**：某些类型的zk-SNARKs依赖于某些假设，如RSA或椭圆曲线，这些假设在量子计算面前可能较弱。

总之，zk-STARKs提供了更高的透明度和更强的安全性，而zk-SNARKs则以其简洁性和效率著称。



## ZK-starks 如何做到无需可信设置的？

### 透明性

zk-starks 不依赖于任何可信第三方来生成系统参数。对比于zk-snarks 必须通过一个可信的设置阶段生成公共参数，这个过程需要依赖于一个信任的第三方或者多方计算协议。如果这些参数生成过程中存在恶意行为，整个系统的安全性就会受到威胁。

* **公共随机数**：zk-STARKs 使用**公共随机数**（如 Fiat-Shamir 变换）来替代可信设置阶段。这些随机数可以通过常见的随机信标或其他公开的随机数生成方法来获得。
* **公开参数生成**：所有需要的系统参数都可以公开生成，并且可以被任何人验证，从而确保了系统的透明性和安全性。

### 代数中间表示（AIR Algebraic Intermediate Representation）

zk-starks 使用代数中间表示来描述计算问题，AIR 是一种能够将任意计算过程转化为多项式方程的方法，这种方法有助于证明者生成符合这些方程的证明，而无需依赖可信设置。具体来说，AIR 通过将计算转化为关于多项式的约束问题，使得证明者和验证者可以通过简单的代数操作来完成证明过程。

### 多项式承诺

在 zk-STARKs 中，多项式承诺机制用于确保数据的**完整性和正确性**。多项式承诺的构建不依赖于可信设置，可以通过以下方式实现：

* Merkle tree ： 利用 Merkle 树结构来**承诺多项式的评估值**。Merkle 树允许高效地证明和验证多项式在某些点上的值，并且这些证明是透明的，不需要可信设置。（完整性）
* FRI ：FRI 用来**验证多项式的低度性**，不依赖于可信第三方，可以通过递归的方式逐步减少多项式的度数，确保多项式的低度性。

### 随机挑战

zk-STARKs 通过使用随机挑战来确保证明的安全性和完整性。随机挑战可以由验证者生成，也可以通过公共随机数生成方法来获得。这些随机挑战在整个证明过程中起到至关重要的作用：

* **Fiat-Shamir变换**：通过 Fiat-Shamir 变换，将交互式证明转化为非交互式证明，使得整个过程无需可信设置。
* **随机性保护**：使用公共随机数生成方法，确保了随机挑战的不可预测性和安全性。

zk-STARKs使用线性的计算模型，这使得证明过程可以被分解为一系列简单的操作，这些操作可以独立验证。这样，证明者只需提供关于**计算过程的证据**，**而不是整个计算的完整状态**。








