## 开源的ZKVM 框架

https://github.com/succinctlabs/sp1.git 

```shell
curl -L https://sp1.succinct.xyz | bash

source ~/.bashrc

// mac os
source /Users/scc/.zshenv

sp1up

///log
sp1up: installed - cargo-prove sp1 (69a3681 2024-04-14T00:21:16.525365000Z)
sp1up: installing rust toolchain
Successfully cleaned up ~/.sp1 directory.
Successfully created ~/.sp1 directory.
Downloading https://github.com/succinctlabs/rust/releases/download/succinct-v2024-02-24.1/rust-toolchain-aarch64-apple-darwin.tar.gz

⠄ [00:00:54] [#######################################################>---------] 265.89MB/308.96MB (4.92MB/s, 9s



cargo prover --version

// mac上 sp1up 就会自动进行 install-toolchain的操作，因此不需要执行这一步。
cargo prover install-toolchain


// build from source code 
git clone git@github.com:succinctlabs/sp1.git
cd sp1
cd cli
cargo install --locked --path .
cd ~
cargo prove build-toolchain
```





```
cargo prove new program_name

.
├── program
│   ├── Cargo.toml
│   ├── elf
│   └── src
│       └── main.rs
└── script
    ├── Cargo.toml
    ├── rust-toolchain
    └── src
        └── main.rs

5 directories, 5 files
```

- `program` 程序源代码，会在 **zkVM** 中执行.
- `script`  包含**生成证明**以及**验证**的代码.

### Build program

```
cd program 
cargo prove build
```

执行这个命令会生成 ELF（Executable and Linkable Format），包含了SP1 zkvm 执行的字节码



### Generate Proof

ELF 文件放入 ZKVM中执行，而 script 中的代码则是生成证明、证明存储以及验证证明的逻辑。

```
cd ../script
RUST_LOG=info cargo run --release
```

烤机开始 ☺

完成后，证明将保存在proof-with-io.json 文件中，并验证正确性。
