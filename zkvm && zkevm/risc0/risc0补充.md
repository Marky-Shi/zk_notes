## 递归证明

![workflow](./workflow-for-risc0.svg)

开始之前补充一下，在最新版本的代码中，官方对gpu算法进行了优化，性能提升了20%～30%

https://github.com/risc0/risc0/pull/2054

RISC Zero 的 zkVM 使用递归证明来实现无限制的计算规模、恒定的证明规模、证明聚合和证明组合。

自定义的程序到被验证的过程是：

1. 程序*执行*后，产生多个的*segment*。
2. 每个*Segment*都经过证明，产生一个*SegmentReceipt*。
3. 每个*SegmentReceipt* 聚合，产生一个*SuccinctReceipt*。
4. 将多对*SuccinctReceipt**连接起来*，产生另一个*SuccinctReceipt*。此过程持续到剩下一个*SuccinctReceipt*为止。
5. 最后的*SuccinctReceipt*经过*identity_p254*传递，为 Groth16 证明做准备。
6. SuccinctReceipt*被*压缩*，*生成*Groth16Receipt*。

Groth16Receipt现在可以发布到链上并由RISC0 合约进行验证。

### 递归电路

Risc0 zkvm 系统由三个电路组成

1. RISC-V 电路是一个 STARK 电路，可证明 RISC-V 程序的正确执行。
2. 递归电路是一个单独的 STARK 电路，旨在高效生成用于验证 STARK 证明的证明，并支持将自定义加速器电路集成到 zkVM 中。该电路的架构与 RISC-V 电路类似，但列数较少，且指令集针对加密进行了优化。RISC -V 电路和递归电路使用相同的证明系统。
3. STARK-to-SNARK 电路是一个 R1CS 电路，用于验证来自递归电路的证明。

### 递归程序

递归电路支持许多程序，包括 lift、join、resolve 和 Identity_p254。这些在 Prover 实现内部使用来生成 SuccinctReceipt 和 Groth16Receipt。

* `lift` 程序使用递归证明程序（Recursion Prover）验证来自 RISC-V Prover 的 STARK 证明。 相对于原始段长度，递归证明只有一个恒定时间验证过程，然后被用作所有其他递归程序（如 join、resolve 和 identity_p254）的输入。
* `join`程序使用递归证明器验证来自递归证明器的两个证明。通过重复应用join，同一session内执行跨度的任意数量的receipt都可以压缩为整个session的单个receipt。
* Identity_p254 程序使用递归证明器和 Poseidon254 哈希函数来验证来自递归证明器的证明。在运行 Groth16 证明者之前，identity_p254 程序用作证明者管道中的最后一步。

