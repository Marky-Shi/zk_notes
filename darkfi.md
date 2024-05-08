## darkfi

[DarkFi - The DarkFi Book (darkrenaissance.github.io)](https://darkrenaissance.github.io/darkfi/index.html)

> 纯zk挖矿项目，感觉和aleo差不多，但是只针对其中的zkvm进行研究，其他部分不做涉猎。

mac os 

```shell
rustup toolchain install nightly
rustup target add wasm32-unknown-unknown
rustup target add wasm32-unknown-unknown --toolchain nightly

brew install wabt && berw install cmake 

git clone https://github.com/darkrenaissance/darkfi.git
cd darkfi 
make zkas //only zk other modules doesn't 
```



darkfi zk底层则是halo2实现的。

```shell
~/code/rustcode/darkfi ±master » ./zkas proof/opcodes.zk
Wrote output to proof/opcodes.zk.bin

./zkas -i -e proof/opcode.zk. //将会返回vm中每一个字节码的执行过程。 -e 则是数结构
```

### zk file 结构

```shell
k = 11;
field = "pallas";

constant "Opcodes" {
	EcFixedPointShort VALUE_COMMIT_VALUE,
	EcFixedPoint VALUE_COMMIT_RANDOM,
	EcFixedPointBase NULLIFIER_K,
}

witness "Opcodes" {
	Base value,
	Scalar value_blind,
	
	...
	EcNiPoint pubkey,
	Base ephem_secret,

	Uint32 leaf_pos,
	MerklePath path,

	Base cond,
}

circuit "Opcodes" {
	
	range_check(64, a);
	range_check(253, b);
	less_than_strict(a, b);
	less_than_loose(a, b);

	root = merkle_root(leaf_pos, path, c);
	constrain_instance(root);

}
```

* `k`  表示row 为 2的k次幂，k越大，证明速度越慢，若k太低，则会直接导致证明失败。通常是 `11 --- 13`
* `filed = "pallas"`  base field.
* `const xxx { }`  常量定义部分
* `witness xxxx{ }`  生成证明时需要提供的信息
* `circuit xxx{ }`  电路逻辑的定义部分。

### 生成zkp 

Darkfi/tests/zkvm_opcode.rs 参考这部分代码。

```rust
fn zkvm_opcodes()->Result<()>{
	// load zkfile and transfer to zkbin
	let bincode = include_bytes!("../proof/opcodes.zk.bin");
  let zkbin = ZkBinary::decode(bincode)?;
	
  // value for proof 
  ...
  let root = tree.root(0).unwrap();
  let merkle_path = tree.witness(leaf_pos, 0).unwrap();
  let leaf_pos: u64 = leaf_pos.into();
  ...
  
  let prover_witnesses = vec![.......];
  
  // pbulic_inputs
  let public_inputs = vec![
        *value_coords.x(),
        *value_coords.y(),
        c2,
        d,
        root.inner(),
        pub_x,
        pub_y,
        ephem_x,
        ephem_y,
        a,
        pallas::Base::ZERO,
    ];
  
  let circuit = ZkCircuit::new(prover_witnesses, &zkbin);
  
  let proving_key = ProvingKey::build(zkbin.k, &circuit);
  let proof = Proof::create(&proving_key, &[circuit], &public_inputs, &mut OsRng)?;

  let verifier_witnesses = empty_witnesses(&zkbin)?;
  let circuit = ZkCircuit::new(verifier_witnesses, &zkbin);
  let verifying_key = VerifyingKey::build(zkbin.k, &circuit);
  proof.verify(&verifying_key, &public_inputs)?;
}
```

整体的生成proof 以及验证，和sp1 很类似 **先加载zkfile，准备参数然后生成证明，最后验证。**

### zk circuit 

ZK电路有布局。空闲空间越少，电路的效率就越高。通常它只是意味着减小指定的 k 值。电路中的行数是 2 k ，因此将该值减 1 将使行数减半

```shell
./bin/zkrunner/zkrender.py -w src/contract/dao/proof/witness/exec.json src/contract/dao/proof/exec.zk /tmp/layout.png
```



### Zkas bincode 

程序由四个部分组成：`constant`  `literal`  ,`witness` , `circuit` 。我们的二进制代码表示相同。此外，还有一个名为 `.debug` 的可选部分，它可以保存与二进制文件相关的调试信息。

目前，将所有变量保存在一个堆上，将`literal`保存在另一堆上。因此，在每个` HEAP_INDEX` 之前，在前面添加 `HEAP_TYPE`，以便虚拟机能够知道应该从哪个堆进行查找。

bincode 结构

```shell
MAGIC_BYTES
BINARY_VERSION
K
NAMESPACE
.constant
CONSTANT_TYPE CONSTANT_NAME 
CONSTANT_TYPE CONSTANT_NAME 
...
.literal
LITERAL
LITERAL
...
.witness
WITNESS_TYPE
WITNESS_TYPE
...
.circuit
OPCODE ARG_NUM HEAP_TYPE HEAP_INDEX ... HEAP_TYPE HEAP_INDEX
OPCODE ARG_NUM HEAP_TYPE HEAP_INDEX ... HEAP_TYPE HEAP_INDEX
...
.debug
TBD
```

* `MAGIC_BYTES`:  是由四个字节组成的文件签名，用于识别 zkas 二进制代码

  ```shell
   0x0b 0x01 0xb1 0x35
  ```

* `BINARY_VERSION`:  binary 文件格式，为之后的迭代做准备。

  ```shell
  0x02
  ```

* `K`: 这是一个 32 位无符号整数，表示了解电路需要多少行所需的 k 参数。

* `NAMESPACE`: 该区域包含代码的引用命名空间。这是源代码中使用的命名空间。

  ```shell
  constant "MyNamespace" { ... }
  witness  "MyNamespace" { ... }
  circuit  "MyNamespace" { ... }
  ```

* `.constant` 常量用其类型和名称进行声明，以便 VM 知道如何搜索内置常量并将其添加到堆中。

* `literal`:`.literal` 部分中的文字当前是无符号整数，会在 VM 内解析为 u64 类型。将来这可以用有符号整数和字符串来扩展。

* `.witness`: `.witness` 部分以 `WITNESS_TYPE` 的形式保存电路`witness `值。每个`witness` 的`堆索引`都会**递增**，因为它们像源文件中一样按顺序保持。 与电路本身类型相同的`witness`（通常是 Base）将使用 `Halo2 load_private API` 作为`private`加载到电路中。

* `.circuit`: 保存了 ZK 证明的程序逻辑。在这里，我们有带有操作码的语句，这些操作码按照虚拟机的理解执行

  ```shell
  OPCODE ARG_NUM HEAP_TYPE HEAP_INDEX ... HEAP_TYPE HEAP_INDEX
  ```

  | Element    | **Description**                                            |
  | ---------- | ---------------------------------------------------------- |
  | Opcode     | execute opcode                                             |
  | ARG_NUM    | the number of arguments that given to opcode               |
  | HEAP_TYPE  | Type of the heap to do lookup from (variables or literals) |
  | HEAP_INDEX | The location of the argument on the heap.                  |

  若操作码执行之后有返回值，应将返回值返回到堆上，以供后续使用。

* `.debug`



### ZKVM

Darkfi zkvm 是基于 halo2 的单个ZKSNARK电路，无需可信设置，并且能够执行和证明编译后的 zkas bincode。 

> zkVM 的设计方式使其能够根据 bincode 选择代码路径，从而创建特定于 zkas 电路的 zkSNARK 证明

整个VM可以被认为是一台可以堆访问ZK电路内构造的值（变量）的机器。初始化时，VM 实例化两个堆，其中一个保存文字（当前支持 u64），另一个保存 HeapVar 中定义的任意类型。

一旦堆被实例化，电路就会初始化所有的可用的 halo2- gadgets，且可以创建和访问任何查找表。



















