## lamdbaworks

Repo: [lambdaclass/lambdaworks: lambdaworks offers implementations for both SNARKs and STARKs provers, along with the flexibility to leverage their individual components for constructing customized SNARKs. (github.com)](https://github.com/lambdaclass/lambdaworks?tab=readme-ov-file)

一个提供snark/stark prover 实现的**低级**repo（并不是low，是很底层的实现）。用于构建证明系统的加密原语的有效实现。同时，还提供了用于证明系统的多种后端，并支持与不同前端的兼容性。相比于gnark，这个库的实现更底层，需要开发者对snark/stark的底层十分熟悉（如 stark中的边界约束，轨迹多项式，AIR等），而gnark（zk-snark）则是封装好了prove/verify，开发者只需要自定义电路逻辑即可。

现已支持了 circom/miden-vm/cairo 等多种前端。



