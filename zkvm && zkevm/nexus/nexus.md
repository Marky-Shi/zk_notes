## nexus zkvm

一个基于RISC-V 的zkvm，后端为snark

Nexus zkVM 架构有两个主要组件：Nexus 虚拟机和 Nexus Proof 系统。

setup

```shell
rustup target add riscv32i-unknown-none-elf

cargo install --git https://github.com/nexus-xyz/nexus-zkvm cargo-nexus --tag 'v0.2.3'
```

### VM Architecture

Nexus虚拟机采用冯诺依曼架构，将指令和数据存储在**同一读写内存空间**中。该机器有32个寄存器和一个读写存储器，地址范围为{0.. 2^32 -1} ，机器状态定义为四元祖(pc,M,R,I):

* Pc 表示程序计数寄存器
* M 表示机器的内存
* R 表示状态寄存器
* I 表示私有输入

![nexus-architecture](./nvm-architecture.01227859.svg)



Nexus VM 指令集共包含 41 条指令。 每条指令通过一个助记符指定，并可接受一些参数，通常是寄存器选择器和一个立即值，格式如下：

*  **mnemonic** 指令名
* `rd` 是指定目标寄存器的寄存器选择器；
* `rs_1`  指定第一个操作数的寄存器选择器；
* `rs_2`  指定第二操作数的寄存器选择器；
* `i`  是一个立即数，其大小根据指令而变化

每条指令被编码为一个 32 位长的字符串，以 7 位长结束 操作码 操作码字符串，在许多情况下前面是 5 位长的寄存器选择器，以及其他数据（取决于操作）。目前Nexus 虚拟机使用通用电路来模拟整个 CPU。但是每条附加指令都增加了 Nexus Proof 系统的复杂性。



### Nexus Proof

* zkvm initialization
* Program execution
* proof accumulation
* proof compression

#### zkvm initialization

在加载之前 𝑃rogram 进入从地址开始的内存 0 𝑥 0000，Nexus VM 最初将其全局寄存器的内容设置为 0，并且假定每个内存位置的值未定义。

为了确保执行过程中内存访问的一致性，nexus 采用了merkle tree 和 Poseidon hash function以及进行内存检查。计算与内存当前内容相关的 Merkle 根，并在内存内容发生变化时更新其值。

一旦计算出初始状态的 Merkle Root，Nexus zkVM 就会**开始加载程序 𝑃 一次执行一条指令，在每次内存更新后相应地更新 Merkle 树根。**

一旦 P 完全加载到内存中，使用内存当前状态的 Merkle Root 作为证明系统执行步骤的公共输入。

##### Public paramaters in nexus zkvm

PP 在整个系统用于生成、验证和管理证明的基本加密值和结构。这些参数是在受信任的设置（或其他初始化过程）期间生成的，并在零知识证明系统的所有参与者之间共享。

##### component of PP 

* Elliptic Curve Parameters (G1,G2):Nexus ZKVM 使用椭圆曲线加密技术，公共参数包括曲线特定的细节，例如椭圆曲线 G1 和 G2 上的生成器和群元素。这些通常用于基于配对的加密技术，其中涉及两个组（G1 和 G2）的曲线。
* Commitment Scheme Parameters(C1,C2):承诺方案用于隐藏某些值，但仍允许稍后验证它们。这些承诺方案的参数（G1 ---> C1、G2 -----> C2）是PP的一部分。
* Random Oracle Configuration 涉及加密哈希函数或随机预言机，它们充当可公开验证的函数，可确定性地将输入映射到看似随机的输出。随机预言机的配置（例如，使用 SHA-256 或其他哈希函数）是PP的一部分.确保证明的不可延展性，并可用于 Fiat-Shamir 转换，使交互式协议变得非交互式。
* Circuit Shape/Description 参数定义了计算的结构（例如，约束如何构建，输入和输出如何排列），这对于证明者和验证者在系统内运行都是必要的。
* Secondary Circuit Parameters 在处理多个电路的递归证明系统中（例如 Nova 和 HyperNova），通常存在与证明的不同层相对应的次级电路参数。这些参数有助于管理递归证明中各个步骤之间的转换。

```rust
pub struct PublicParams<G1, G2, C1, C2, RO, SC, SP>
where
    G1: SWCurveConfig,
    G2: SWCurveConfig,
    C1: PolyCommitmentScheme<Projective<G1>>,
    C2: CommitmentScheme<Projective<G2>>,
    RO: CryptographicSponge + Sync,
    RO::Config: CanonicalSerialize + CanonicalDeserialize + Sync,
    SC: StepCircuit<G1::ScalarField>,
    SP: Send + Sync,
{
    pub ro_config: RO::Config,
    pub shape: CCSShape<G1>,
    pub shape_secondary: R1CSShape<G2>,
    pub ck: C1::PolyCommitmentKey,
    pub pp_secondary: C2::PP,
    pub digest: G1::ScalarField,

    pub _step_circuit: PhantomData<SC>,
    pub _setup_params: PhantomData<SP>,
}
```

#### Program execution

在完成上述初始化之后，Nexus zkVM 在 Nexus VM 上运行该程序并为其生成完整的执行跟踪。然后，该执行跟踪被传递到证明累积步骤，

![execution-sequence](./execution-sequence.d111c904.svg)

#### Proof Accumulation

Nexus zkVM 使用Proof Accumulation（无 SNARK），如上图所示。使用这种方案的主要优点是它实现了**增量可验证计算**（IVC）的概念及其证明携带数据（PCD）对分布式设置的推广

Nova 的一个主要优点是它具有较小的递归开销，这表示除了证明调用之外，证明者还必须证明的步骤数 𝐹 。特别是，当用于实现IVC方案时，Nova的递归开销主要由两组标量乘法和三个哈希计算组成。当在 PCD 中使用时，此开销稍高。

就证明者成本而言，证明者在每个折叠步骤中最昂贵的工作是**计算两个多标量乘法（MSM）**，其大小与  F. 这是因为证明者需要使用 Pedersen 承诺方案在每个折叠步骤中承诺两个大见证向量。尽管是证明生成中计算成本最高的部分，但该计算是高度可并行的。

Nexus zkVM 使用配对友好的 bn254（作为主曲线（也称为 bn128 或 bn256）来实现 Nova 折叠方案。因为后者在以太坊上有预编译支持。此外，由于 bn254 形成 2-cycle 与 grumpkin曲线一起，grumpkin 用作辅助曲线。



#### IVC Incrementally Verifiable Computation Proof

IVC Nexus ZKVM 中用于表示递归计算过程的证明结构。

* 在逐步执行计算时逐步生成证明的技术。IVC 不是在最后为整个计算生成一个证明，而是允许为每个步骤生成一个证明，并且可以递归地组合这些证明。
* 封装计算的状态，包括中间步骤，并允许逐步证明和验证这些步骤。每次计算新步骤时，都会更新证明，无需重新执行整个计算即可验证结果。



##### Hyper Nova && Nova 

* Nova 是一种基础递归 SNARK协议，允许增量生成证明。它旨在实现增量可验证计算 (IVC)，其中计算的每个步骤都会生成一个证明，该证明可以与之前步骤的证明相结合
  * 在 Nexus ZKVM 中，Nova 用于处理递归证明，将大型计算拆分为多个步骤。每个步骤都会为该计算部分生成一个证明，并且这些证明可以有效地组合成一个证明。即使底层计算很长或很复杂，Nova 的方法也能确保最终证明保持简洁。
* HyperNova 是 Nova 的扩展或改进。它利用更先进的技术进一步提高了证明效率和可扩展性，有可能实现更紧凑的证明和更快的验证时间。
  * HyperNova 通常专注于优化证明系统，以用于更大或更复杂的应用程序。在 Nexus ZKVM 中，HyperNova 将应用于 Nova 性能可能不足的场景，提供改进的证明压缩和验证速度，尤其是在**处理递归电路或嵌套计算**时。

总结：

Nova 是用于增量证明生成的基础递归 SNARK，而 HyperNova 则在 Nova 的基础上进行了进一步的优化和提升。IVCProof 是 Nexus ZKVM 中的核心证明对象，它封装了递归证明过程，从而实现了可扩展和可验证的计算。



#### Proof Compression

生成的proof 相当大，导致验证起来很消耗资源，证明生成的最后一步是用一系列zk-snark 来压缩累积的证明。

目前的设计中，proof cimpression 包含两个阶段：

* IVC阶段生成的累积证明被转化为有效IVC证明的知识证明，其大小是底层Nova IVC证明大小的对数。
* 这些证明被转换为恒定大小的snark proof，以便于在L1 上进行验证。

压缩步骤的第一阶段，当前设计使用两种不同的证明系统， `bn254`  ` grumpkin` 椭圆曲线。

对于 bn254 曲线相关的 IVC 证明，Nexus zkVM 使用 Nova 友好的 Spartan SNARK 以及 Zeromorph多项式承诺方案。这种组合不仅允许 Nexus zkVM 生成与底层 Nova IVC 证明大小成对数的证明，而且还避免了 SNARK 证明器在 R1CS 电路内计算多标量乘法的需要。（这部分就与jolt和类似）

由于grumpkin 不是配对友好的椭圆曲线，因此nexus 使用基于KZG 多项式承诺的 halo2方案，由于次级 IVC 电路的尺寸较小，电路尺寸的放大是可以控制的。（所以这部分还是有很大的优化空间的）

压缩阶段的第二部分，当前是 Nexus zkVM 使用 Groth16 证明。压缩的最后阶段将递归地验证第一阶段的 Spartan 证明，将日志大小的证明减少到恒定大小。



### Memory Checking

由于 Nexus 虚拟机使用的外部内存被认为是不可信的，因此 Nexus zkVM 必须确保整个程序执行过程中的读写一致性。通俗地说，这要求读取内存任何单元的内容应始终返回最后写入该单元的值。

之前提到的使用merkle tree 和 poseidon hash func  内存的内容首先与 Merkle 树的叶子相关联，然后通过 Merkle 哈希处理生成 Merkle 根。后者 Merkle 根是对内存的约束性承诺，每当内存内容发生变化时都需要更新。

#### Merkle tree  setup

内存大小设置为2^32字节 ,将叶子与256位长的字符串（即32字节）关联，然后使用17层的Merkle二叉树来描述其内容，

最初，假设每个存储单元是0字符串。因此，为了计算初始内存的 Merkle 根，我们首先计算树的每个级别的默认哈希值，从叶子开始，到 Merkle 根结束。对于叶子，默认值只是0字符串。对于任何后续级别，它们的默认值将成为其下一级的两个默认值串联的哈希值。

#### Update && prove

为内存设置初始 Merkle 根后，每次访问内存时都会使用或更新该值，具体取决于操作是读取还是写入。

##### Read 

![merkle-hash-tree-path](./merkle-hash-tree-path.57b68586.svg)



##### write

* 首先执行读操作获取 中号  M[64] 以及它的 Merkle 开局证明（下图中红色框突出显示)
* 更新值 m[64]至 内存段中的0xFF **MS**=**M**[64…95]  以及路径中的所有节点{*h*170,…,1,0,…,*h*10,*h*0} ,从关联的叶子MS开始
* 最后保留更新后的值 h0作为新的 Merkle 根。

![merkle-hash-tree-path-update](./merkle-hash-tree-path-update.7f0f7c14.svg)



















