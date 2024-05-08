## RICS0

jolt/sp1 等收到RICS0的启发实现的zkvm，但是不同于sp1/jolt的是，rics0是基于zk-starks协议实现的。

科普一下zk-starks：

ZK-STARKs，全称为Zero-Knowledge Scalable Transparent Argument of Knowledge（零知识可扩展透明知识论证），是一种零知识证明系统。就像ZK-SNARKs一样，ZK-STARKs可以验证一个声明的有效性，而不透露任何关于声明本身的信息

特性：

1. **无需可信设置**：ZK-STARK不需要可信设置即可运行，而是依赖于公共随机性。这减少了用户的信任假设并提高了基于STARK的协议的安全性。
2. **可扩展**：相比与snarks，STARK的计算和验证速度更快。更重要的是，即使底层计算的复杂性呈指数级增长，ZK-STARKs的证明和验证时间仍然很短。
3. **吞吐量大**：与SNARKs一样，STARKs可以通过启用安全且可验证的链下计算来扩展区块链。提交到L1链的单个STARK证明可以验证在主链外进行的数千笔交易。
4. **更高的安全保障**：ZK-STARKs使用抗碰撞哈希（collision-resistant hashes）进行加密，而不是ZK-SNARKs中使用的椭圆曲线方案（elliptic curve schemes）。这被认为可以抵抗量子计算攻击，使其比SNARK中使用的椭圆曲线更安全。

缺点：

* 产生更大的证明(proof size)

**递归ZK-SNARK是一种特殊的ZK-SNARK，它可以验证其他ZK-SNARK的证明。这意味着，我们可以并行地为不同的交易区块生成证明，并将它们聚合成一个提交到主区块链的单个区块证明**

s

> dev tools
>
> ```shell
> go install github.com/google/pprof@latest
> brew install graphviz
> ```



### setup

```rust
cargo install cargo-binstall
cargo binstall cargo-risczero
cargo risczero install 
```

### update

```shell
cargo binstall cargo-risczero
cargo risczero install
```



![ricso-setp](./risc0-step.png)

program project 

```shell
total 168
-rw-r--r--  1 scc  staff  53794  5  5 12:00 Cargo.lock
-rw-r--r--  1 scc  staff    216  5  5 11:59 Cargo.toml
-rw-r--r--  1 scc  staff  11357  5  5 11:59 LICENSE
-rw-r--r--  1 scc  staff   4399  5  5 11:59 README.md
drwxr-xr-x  4 scc  staff    128  5  5 11:59 host
drwxr-xr-x  6 scc  staff    192  5  5 11:59 methods
-rw-r--r--  1 scc  staff     88  5  5 11:59 rust-toolchain.toml
drwxr-xr-x@ 7 scc  staff    224  5  5 13:17 target

//methods
total 16
-rw-r--r--  1 scc  staff  167  5  5 11:59 Cargo.toml
-rw-r--r--  1 scc  staff   48  5  5 11:59 build.rs
drwxr-xr-x  5 scc  staff  160  5  5 12:00 guest
drwxr-xr-x  3 scc  staff   96  5  5 12:03 src
```

* zkVM 应用程序的核心是guest 目录下的程序。guest是应用程序中经过验证的部分。
  * 将guest程序编译为ELF文件，
  * executor 运行ELF文件，标记为一个session
  * 然后prover 检查并证明ELF的有效性，并输出recepit
* ![program build && run && prove](./from-rust-to-receipt-23117368c4f46d78c8cac3b753245a5a.png)

任何拥有receipt 副本的人都可以验证guest的执行情况并读取其public的输出。验证算法接收`ImageID`作为参数； ImageID 用作预期 ELF 二进制文件的加密标识符。

receipt 给出了程序的结果以及这些结果是`诚实生成的证据`

> 当执行 zkVM 应用程序时，应用程序的输出将包含在receipt中。作为执行程序的简洁有效性证明。可以传递给第三方并进行验证，以便以加密方式证明应用程序输出的有效性。
>
> Receipt由journal和seal组成。journal证明了程序的公开输出，而seal是隐私的，以密码方式证明了收据的有效性。



### Host

在 zkVM 应用程序中，host是运行 zkVM 中。host是一个不受信任的代理，用于设置 zkVM 环境和 在执行过程中处理输入/输出。

host负责构建和运行 executor && prover

```rust
use risc0_zkvm::{default_prover, ExecutorEnv};

let env = ExecutorEnv::builder().build().unwrap();
let prover = default_prover();
let receipt = prover.prove(env, METHOD_NAME_ELF).unwrap();
```

#### Verify receipt

```rust
receipt.verify(METHOD_NAME_ID).unwrap();
```



使用dev mode 绕过耗时的证明生成，激活开发模式

```shell
RISC0_DEV_MODE=1 RUST_LOG="[executor]=info" cargo run --release
RISC0_DEV_MODE=1 cargo run --release
```

若是要生成证明则使用

```shell
RISC0_DEV_MODE=0 cargo run --release
```



### RICS0中用于生成proof的选项

* local： 在用户自己的 CPU/GPU 上生成证明。
* dev-mode：不生成任何证明，从而实现快速原型设计。
* remote（Bonsai）：证明由 Bonsai 生成，Bonsai 是一种高度并行、高性能的证明服务。



### 程序分析与优化

性能分析工具（如 pprof 和 perf）允许收集性能 有关程序整个执行过程的信息，并帮助创建 程序性能的可视化效果。RISC Zero具有实验性 支持生成用于周期计数的 PPROF 文件。

本地需要安装golang，以及 pprof

```shell
RISC0_PPROF_OUT=./profile.pb cargo run

go tool pprof -http=127.0.0.1:8000 profile.pb
```

