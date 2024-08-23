## circom

### setup

rust tool-chain

```
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

nodejs  yarn

```
curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update && sudo apt-get install yarn
```

```
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs



//update node version to stable
sudo npm install -g n
sudo n stable 
```



### build form source code

```
git clone https://github.com/iden3/circom.git

cd circom && cargo build --release
//will generate binary at target/release 

 ls -l target/release/circom
-rwxr-xr-x 2 root root 14532448 Mar  4 17:51 target/release/circom
```

### install snarkjs

```
npm install -g snarkjs@latest
```



## demo

crate a new file that include myself demo circuit named `circuit.circom`

```
pragma circom 2.1.8;

template Multiply(){
	signal input a;
	signal input b;
	signal output out;
	
	out <== a*b;
}

component main = Myltiply();
```

> 使用此电路，我们将能够证明我们知道两个数字（a和b）相乘得到c，而不会显示a和b。
>
> 这个电路有2个 private 输入信号，名为 `a` 和 `b` ，还有一个输出 `out`.
>
> 输入和输出使用`<==`运算符进行关联。 在circom中，<==运算符做两件事。 首先是连接信号。 第二个是施加约束。
>
> 在本例中，我们使用`<==`将`cout` 连接到`a`和`b`，同时将`cout`约束为`a * b`的值，即电路做的事情是让强制信号 `c` 为 `a*b` 的值。
>
> 在声明 `Multiplier` 模板之后, 我们使用名为`main`的组件实例化它。
>
> 注意：编译电路时，必须始终有一个名为`main`的组件。
>
> **有效的 R1CS 必须每个约束（在 R1CS 中的一行，在 Circom 中 `<==`）只有一个乘法。**

### 编译电路

```
circom circuit.circom --r1cs --sym --wasm

/// log
template instances: 1
non-linear constraints: 1
linear constraints: 0
public inputs: 0
private inputs: 2
public outputs: 1
wires: 4
labels: 4
Written successfully: ./circuit.r1cs
Written successfully: ./circuit.sym
Written successfully: ./circuit_js/circuit.wasm
Everything went okay
```

- `--r1cs`: 生成 `circuit.r1cs` ( [r1cs](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649) 电路的二进制格式约束系统).
- `--wasm`: 生成 `circuit.wasm` ( wasm 代码用来生成见证 witness ).
- `--sym`: 生成 `circuit.sym` (以注释方式调试和打印约束系统所需的符号文件）

### 将编译好的电路载入snarkjs

- 查看电路信息

  - ```shell
    snarkjs info -r circuit.r1cs
    [INFO]  snarkJS: Curve: bn-128
    [INFO]  snarkJS: # of Wires: 4
    [INFO]  snarkJS: # of Constraints: 1
    [INFO]  snarkJS: # of Private Inputs: 2
    [INFO]  snarkJS: # of Public Inputs: 0
    [INFO]  snarkJS: # of Labels: 4
    [INFO]  snarkJS: # of Outputs: 1
    ```

  - 打印电路约束

    ```shell
    snarkjs r1cs print circuit.r1cs  circuit.sym
    [INFO]  snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.a ] * [ main.b ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.out ] = 0
    
    // 既  a*b -c =0 
    ```

### Comput the witness

- 创建一个 input.json 

  ```
  {"a": "3", "b": "11"}
  ```

  通过wasm 生成witness

  ```shell
  node generate_witness.js multiplier2.wasm input.json witness.wtns
  ```

  通过C++ 生成

  ```
  make
  
  ./xxxname input.json witness.wtns
  ```

  这两个命令将生成相同的ẁitness.wtns 文件。该文件以与 snarkjs 兼容的二进制格式编码，snarkjs 是我们用来创建实际证明的工具。`wtns`中包含所有计算出的信号，以及一个扩展名为 .r1cs 的文件，其中包含描述电路的约束。这两个文件将用于创建我们的证明。

  tip： 对于大型电路 C++ 的速度会快于wasm

### 可信设置 trusted setup



Groth16需要每个电路的可信设置。更详细地说，可信设置由两部分组成：

- The **powers of tau**, 与电路无关。
- `phase2` 依赖电路



### power of tau

- new power of tau  ceremony

  - ```
    snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
    
    [DEBUG] snarkJS: Calculating First Challenge Hash
    [DEBUG] snarkJS: Calculate Initial Hash: tauG1
    [DEBUG] snarkJS: Calculate Initial Hash: tauG2
    [DEBUG] snarkJS: Calculate Initial Hash: alphaTauG1
    [DEBUG] snarkJS: Calculate Initial Hash: betaTauG1
    [DEBUG] snarkJS: Blank Contribution Hash:
                    786a02f7 42015903 c6c6fd85 2552d272
                    912f4740 e1584761 8a86e217 f71f5419
                    d25e1031 afee5853 13896444 934eb04b
                    903a685b 1448b755 d56f701a fe9be2ce
    [INFO]  snarkJS: First Contribution Hash:
                    9e63a5f6 2b96538d aaed2372 481920d1
                    a40b9195 9ea38ef9 f5f6a303 3b886516
                    0710d067 c09d0961 5f928ea5 17bcdf49
                    ad75abd2 c8340b40 0e3b18e9 68b4ffef
    ```

  - ```shell
    snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v
    Enter a random text. (Entropy): ******
    [DEBUG] snarkJS: Calculating First Challenge Hash
    [DEBUG] snarkJS: Calculate Initial Hash: tauG1
    [DEBUG] snarkJS: Calculate Initial Hash: tauG2
    [DEBUG] snarkJS: Calculate Initial Hash: alphaTauG1
    [DEBUG] snarkJS: Calculate Initial Hash: betaTauG1
    [DEBUG] snarkJS: processing: tauG1: 0/8191
    [DEBUG] snarkJS: processing: tauG2: 0/4096
    [DEBUG] snarkJS: processing: alphaTauG1: 0/4096
    [DEBUG] snarkJS: processing: betaTauG1: 0/4096
    [DEBUG] snarkJS: processing: betaTauG2: 0/1
    [INFO]  snarkJS: Contribution Response Hash imported: 
                    ad89125b 5d69df55 67ece43f 5da6f782
                    057d0447 74bdcdb4 65d9b3b1 95a20d7f
                    dccbfb99 2b38d5d1 ab0734ff dc8d7105
                    b73583ee e5d88ffb e2fc46ff c6153e9e
    [INFO]  snarkJS: Next Challenge Hash: 
                    a26e380e 7238338d 26c90ee0 849328f3
                    ce2180cd 920340ee e67befb1 3b771842
                    bd219957 9a2b7adc ca7d0a73 94962888
                    9c3cc50e 50f42235 163d48a4 27b31f4d
    ```

### phase2

phase2 针对特定电路，执行如下命令开始该阶段的生成

```shell
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v

[36;22m[DEBUG] [39;1msnarkJS[0m
: Starting section: tauG1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 0 mix start: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 0 mix end: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 1 mix start: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 1 mix end: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 2 mix start: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 2 mix end: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 3 mix start: 0/1
[36;22m[DEBUG] [39;1msnarkJS[0m: tauG1: fft 3 mix end: 0/1
.................

[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft  12  join: 12/12
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 1/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 0/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 4/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 6/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 2/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 5/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 3/8
[36;22m[DEBUG] [39;1msnarkJS[0m: betaTauG1: fft 12 join  12/12  1/1 7/8
```

**生成.zkey 文件，包含证明和验证密钥以及所有phase2的内容**

```shell
snarkjs groth16 setup multiplier2.r1cs pot12_final.ptau multiplier2_0000.zkey

[INFO]  snarkJS: Reading r1cs
[INFO]  snarkJS: Reading tauG1
[INFO]  snarkJS: Reading tauG2
[INFO]  snarkJS: Reading alphatauG1
[INFO]  snarkJS: Reading betatauG1
[INFO]  snarkJS: Circuit hash: 
                d56ff0d8 cea93e0b 37d3df09 f81225fa
                44f43c6b e252a563 12f371dc 2d85427e
                d21f68e8 50cff65b dce88ffd ea8e0825
                abe792f1 11d89a70 6cda2dd4 82f9b074
```

**Contribute to the phase 2 of the ceremony:**

```
snarkjs zkey contribute multiplier2_0000.zkey multiplier2_0001.zkey --name="1st Contributor Name" -v
Enter a random text. (Entropy): xxxxxx
[DEBUG] snarkJS: Applying key: L Section: 0/2
[DEBUG] snarkJS: Applying key: H Section: 0/4
[INFO]  snarkJS: Circuit Hash: 
                d56ff0d8 cea93e0b 37d3df09 f81225fa
                44f43c6b e252a563 12f371dc 2d85427e
                d21f68e8 50cff65b dce88ffd ea8e0825
                abe792f1 11d89a70 6cda2dd4 82f9b074
[INFO]  snarkJS: Contribution Hash: 
                ec239892 8a43a19e 6a9a7390 5796c4c6
                78f40d42 07063366 07ab6e05 1fe4b6f7
                63afb457 eebe90b3 0bfeb8c1 5b9865dd
                321b6297 cddb21b9 29dd3ca9 2490e4d1
```

**Export the verification key:**

```
snarkjs zkey export verificationkey multiplier2_0001.zkey verification_key.json

[INFO]  snarkJS: EXPORT VERIFICATION KEY STARTED
[INFO]  snarkJS: > Detected protocol: groth16
[INFO]  snarkJS: EXPORT VERIFICATION KEY FINISHED
```

### Generating Proof

一旦计算出witness并执行了可信设置，就可以生成与电路和见证相关的 zk 证明：

```
snarkjs groth16 prove multiplier2_0001.zkey witness.wtns proof.json public.json
```

该命令会生成一个 groth16 proof 并且输出两个文件：

- `proof.json` 
- `public.json`  它包含公共输入和输出的值。

### verify  proof

```
snarkjs groth16 verify verification_key.json public.json proof.json

///log
snarkjs groth16 verify verification_key.json public.json proof.json 
[INFO]  snarkJS: OK!
```

该命令使用我们之前导出的文件verification_key.json、proof.json和public.json来检查证明是否有效。如果证明有效，该命令将输出 OK。

有效的证明不仅证明我们知道一组满足电路的信号，而且证明我们使用的公共输入和输出与 public.json 文件中描述的相匹配.



### verify from a Smart contract

还可以生成一个 Solidity 验证器，允许在以太坊区块链上验证证明。

- #### generate the sol code

  - ```shell
    snarkjs zkey export solidityverifier multiplier2_0001.zkey verifier.sol
    /// log
    [INFO]  snarkJS: EXPORT VERIFICATION KEY STARTED
    [INFO]  snarkJS: > Detected protocol: groth16
    [INFO]  snarkJS: EXPORT VERIFICATION KEY FINISHED
    ```

    该命令将验证密钥multiplier2_0001.zkey作为输入，并在名为verifier.sol的文件中输出Solidity代码。可以从该文件中复制代码，然后粘贴到Remix中。您会发现代码包含两个合约：Pairing和Verifier。**只需要部署Verifier合约**。

    Verifier有一个名为**`verifyProof`**的视图函数，当且仅当证明和输入有效时返回**TRUE**。为了方便调用，可以使用snarkJS生成调用的参数。

    ```
    snarkjs generatecall
    
    ["0x268b0b62df8c1b06924fdc17cb33265f0b4d4b37a564225737b3c80ef05b8e89", "0x2740f68c607ee863f2ad5519c78c08f4bb2b26d16a50d282fc7415e5dc2183b5"],[["0x2ca8a4a50a47aaca9deba5b09eefa4fb0ba66926a1f752f7c1f912e1ed7f106c", "0x143a9d66d107b10cbf65e4454b6ca4df793c2a790db7e063420895f44925703f"],["0x0fd2fd968bd0b8cd5bc8337918062ab6fefe6966d01ee2865d1fbc69aae21324", "0x1837d5580f4ab82858827cb38e83eddbbe410d525f79a40fefc84dac20ea4f9d"]],["0x1a1c5a993cf17e506aeab665f59843b35f0ecd0a5339392181500d91c4a15f5e", "0x291f8833c67dcf45858f18c7bb4197794d919a24c47941032063496b5ee625b9"],["0x0000000000000000000000000000000000000000000000000000000000000021"]
    ```

    将命令的输出剪切并粘贴到 Remix 中 verifyProof 方法的参数字段。如果一切顺利，该方法应返回 TRUE。您可以尝试更改参数中的一个位，您会发现结果是无法验证的 FALSE。



### 进阶

[Circom 语言教程与 circomlib 演示 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/6811)



#### circom 公共输入

```shell
pragma circom 2.1.8;

template Somepublic(){
    signal input a;
    signal input b;
    signal input c;
    signal v;
    signal output out;

    v <== a*b;
    out <== v*c;
}

component main {public [a,b]} = Somepublic();
```

a 和 c 是公共输入，但 b 保持隐藏。注意当实例化主组件时需要 **`public`** 关键字。

```
snarkjs info -r public_input.r1cs 
[INFO]  snarkJS: Curve: bn-128
[INFO]  snarkJS: # of Wires: 6
[INFO]  snarkJS: # of Constraints: 2
[INFO]  snarkJS: # of Private Inputs: 1 // b
[INFO]  snarkJS: # of Public Inputs: 2  // a,c
[INFO]  snarkJS: # of Labels: 6
[INFO]  snarkJS: # of Outputs: 1
```

#### 数组

```shell
pragma circom 2.1.8;

template powers(n){
    signal input a;
    signal output powers[n];

    powers[0] <== a;

    for (var i =1 ; i<n ; i++){
        powers[i] <== powers[i-1]*a;
    }
}

component main = powers(6);  /// 必须是固定的。
```

##### 模板参数和变量

template 使用 **n** 作为参数，`Powers(n)`。

一个R1CS 必须是**固定不变**的，即一旦定义了行数或者列数，就不可再更改，也不能更改矩阵或证明的值。

`要重用这段代码来支持不同大小的电路，那么让模板能够根据需要改变大小会更加方便。因此，组件可以接受参数来参数化控制流和数据结构，但这必须是每个电路固定的。`



**变量可构建存在于 R1CS 之外的辅助代码。它们有助于定义电路，但它们不是电路的一部分。**

变量 `var i`只是用于跟踪循环迭代的记账变量，在构建电路时使用，它不是约束的一部分。

```shell
template instances: 1
non-linear constraints: 5
linear constraints: 0
public inputs: 0
private inputs: 1 (none belong to witness)
public outputs: 6    // arr[6]
wires: 7
labels: 8
```



#### signal  VS  variable

signal 是**不可变的**，旨在成为R1CS的**列**之一。 变量只是R1CS的一部分，用于在R1CS之外计算值，用来帮助定义R1CS。

signal不可变的原因是因为 **R1CS 中的见证条目具有固定的值**。R1CS 中的解向量改变值是没有意义的，因为无法为它创建证明。

`<--`、`<==`和`===`操作符用于Signal，而不是变量。



`<==`操作符**先计算，然后赋值，然后添加约束**。如果只想约束，使用`===`。

试图强制`<--`分配正确的值时，可以使用`===`操作符

以下的电路是等价的：

```shell
pragma circom 2.1.8;

template Multiply() {
    signal input a;
    signal input b;
    signal output c;
    
    c <-- a * b;
    c === a * b;
}

template MultiplySame() {
    signal input a;
    signal input b;
    signal output c;
    
    c <== a * b;
}
```

#### template 组合

Circom 模板是可重用和可组合的，

```shell
pragma circom 2.1.8;

template Square(){
    signal input in;
    signal output out;

    out <== in * in; 
}


template SumOfSquare(){
    signal input a;
    signal input b;
    signal output out;

    component sq1 = Square();
    component sq2 = Square();


    sq1.in <== a;
    sq2.in <== b;

    out <== sq2.out + sq1.out;
}

component main  = SumOfSquare();
```

`<==`运算符视为通过引用它们的输入或输出将组件“连接”在一起。

#### 组件的多个输入

如果一个组件接受多个输入，通常将其指定为一个数组 in[n]。

```
pragma circom 2.1.8;

template Square(){
    signal input in[2];
    signal output out;

    out <== in[0] * in[1]; 
}

template SumOfSquare(){
    signal input Sumin[4];
    
    signal output out;

    component sq1 = Square();
    component sq2 = Square();

    sq1.in[0] <== Sumin[0];
    sq1.in[1] <== Sumin[1];

    sq2.in[0] <== Sumin[2];
    sq2.in[1] <== Sumin[3];

    out <== sq2.out + sq1.out;
}

component main  = SumOfSquare();
```



只有一个约束，证明者只需正确设置数组中的第一个元素，但可以为其他 5 个元素设置任何值！**你不能相信从这样的电路中产生的证明！**

欠约束（Underconstraints）是零知识应用中安全漏洞的主要来源，因此请再三检查约束是否按照你的预期在 R1CS 中生成！

这就是为什么我们强调在学习 Circom 语言之前要先理解 R1CS（Rank 1 Constraint Systems），否则会有一整类难以检测的错误！



**在编写 circom 代码时，这是另一个容易犯的错误：信号不能作为 if 语句或循环的输入。**

```
template IsOver21() {
    signal input age;
    signal output oldEnough;
    
    if (age >= 21) {
        oldEnough <== 1;
    } else {
        oldEnough <== 0;
    }
}

```

信号是域元素，不能在信号上使用变量的运算符。



### TOOLS

社区中circom 开发的用到的工具

Circomspect 是 Circom 编程语言的静态分析器和 linter

https://github.com/trailofbits/circomspect

```shell
cargo install circomspect

useage:
circomspect path/circuit.circom  --curve BN254 --level debug

A static analyzer and linter for Circom programs

USAGE:
    circomspect [OPTIONS] [INPUT]...

ARGS:
    <INPUT>...    Initial input file(s)

OPTIONS:
    -a, --allow <ID>             Ignore results from given analysis passes
    -c, --curve <NAME>           Set curve (BN254, BLS12_381, or GOLDILOCKS) [default: BN254]
    -h, --help                   Print help information
    -l, --level <LEVEL>          Output level (INFO, WARNING, or ERROR) [default: WARNING]
    -L, --library <LIBRARIES>    Library file paths
    -s, --sarif-file <OUTPUT>    Output analysis results to a Sarif file
    -v, --verbose                Enable verbose output
```



circom 在线编译器 https://zkrepl.dev/ 

