# Example && Source code

## Example 

```rust
cargo nexus host project-name 
host-project ±master⚡ » tree 
.
├── Cargo.lock
├── Cargo.toml
└── src
    ├── guest
    │   ├── Cargo.toml
    │   ├── rust-toolchain.toml
    │   └── src
    │       └── main.rs  // user defined program 
    └── main.rs // prove && verify

4 directories, 6 files

```

类似于jolt 这个zkvm的项目结构

```shell
tree
.
├── Cargo.lock
├── Cargo.toml
├── guest
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── index.html
├── rust-toolchain.toml
├── rustfmt.toml
└── src
    └── main.rs 

4 directories, 8 files
```



```shell
cd guest && cargo nexus prove 
Setting up public parameters for IVC ... 14.7s
    Finished in 14.7s
  Loading public parameters ... 3.8s
 Finished in 3.8s
  Computing step 0 ... 834ms
  Computing step 1 ... 973ms
  Computing step 2 ... 973ms
  Computing step 3 ... 1.0s
  Computing step 4 ... 988ms
  Computing step 5 ... 1.0s
  Computing step 6 ... 1.0s
  Computing step 7 ... 1.0s
  Computing step 8 ... 1.0s
  Computing step 9 ... 1.0s
  Computing step 10 ... 1.0s
  Computing step 11 ... 1.0s
  Computing step 12 ... 995ms
  Computing step 13 ... 1.0s
  Saving proof ... 25ms
Finished in 25ms
```



guest/src/main.rs 这是用户自定义的程序

```rust
#![no_std]
#![no_main]

#[nexus_rt::profile]
fn fibonacci(n: u32) -> u32 {
    fib(n)
}

/// profile macro 用来分析guest中的 `fibonacci` 函数的执行情况
/// ```shell
/// benchmark/src/guest ±master⚡ » cargo nexus run
///  Execution Summary:
/// └── Total program cycles: 1073
/// └──  'src/guest/src/main.rs:fibonacci': 895 cycles (83% of total)
/// ```
fn fib(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fib(n - 1) + fib(n - 2),
    }
}

#[nexus_rt::main]
fn main() {
    let n = 5;
    assert_eq!(5, fibonacci(n));
}

```

Project/src/main.rs 这是用来验证的程序，类似于 sp1 中的 scripts/main.rs   或者jolt中的 src/main.rs

```rust
use nexus_sdk::{
    compile::CompileOpts,
    nova::seq::{Generate, Nova, PP},
    Local, Prover, Verifiable,
};

const PACKAGE: &str = "guest";

fn main() {
    println!("Setting up Nova public parameters...");
    let pp: PP = PP::generate().expect("failed to generate parameters");

    let mut opts = CompileOpts::new(PACKAGE);
    opts.set_memlimit(8); // use an 8mb memory

    println!("Compiling guest program...");
    let prover: Nova<Local> = Nova::compile(&opts).expect("failed to compile guest program");

    println!("Proving execution of vm...");
    let proof = prover.prove(&pp).expect("failed to prove program");

    println!(">>>>> Logging\n{}<<<<<", proof.logs().join(""));

    print!("Verifying execution...");
    proof.verify(&pp).expect("failed to verify proof");

    println!("  Succeeded!");
}
```

```
cd /src/guest && cargo nexus prove
若是全局执行则是 cargo run -r 
```

按照目前nexus-vm的设计有三种prover server

```rust
pub enum ProverImpl {
    Jolt,
    Nova(NovaImpl),
    HyperNova, // Nova 的加强版，用于生成更加紧凑简洁的proof
}
```

```rust
/// Prover for the Nexus zkVM using Nova.
pub struct Nova<C: Compute = Local> {
    vm: NexusVM<MerkleTrie>,
    _compute: PhantomData<C>,
}


/// A verifiable proof of a zkVM execution. Also contains a view capturing the output of the machine.
pub struct Proof {
    proof: IVCProof,
    view: UncheckedView,
}

/// Prover for the Nexus zkVM using HyperNova.
pub struct HyperNova<C: Compute = Local> {
    vm: NexusVM<MerkleTrie>,
    _compute: PhantomData<C>,
}

/// A verifiable proof of a zkVM execution. Also contains a view capturing the output of the machine.
pub struct Proof {
    proof: IVCProof,
    view: UncheckedView,
}


/// Prover for the Nexus zkVM using Jolt.
pub struct Jolt<C: Compute = Local> {
    vm: JoltVM<MerkleTrie>,
    _compute: PhantomData<C>,
}

/// A Jolt proof (and auxiliary information needed for verification).
pub struct Proof {
    proof: JoltProof,
    pre: JoltPreprocessing,
    commits: JoltCommitments,
}
```

在咱们刚才的例子中使用到的则是 nova，其他两种prover 在sdk中也提供了，咱们首先以nova为主，其余两种方式随后也会提到。

## Nova prover

### generate public params

```rust
let pp: PP = PP::generate().expect("failed to generate parameters");
```

PP  *Public parameters*，用于proof 生成和验证

```rust
// 每个递归步骤要打包的 vm 指令的硬编码数量
const K: usize = 16;

fn generate() -> Result<Self, Self::Error> {
        Ok(gen_vm_pp(K, &()).map_err(ProofError::from)?)
}
```

```rust
pub fn gen_vm_pp<C, SP>(k: usize, aux: &C::SetupAux) -> Result<PP<C, SP>, ProofError>
where
    SP: SetupParams<G1, G2, C, C2, RO, SC>,
    C: CommitmentScheme<P1>,
{
    let tr = nop_circuit(k)?;
    gen_pp(&tr, aux)
}
```

gen-vm-pp 首先根据实现划定的vm指令数生成对应数的nop电路。`nop_circuit(k)` 函数通过创建一个最小化的空操作电路，为后续的零知识证明系统提供了基础。这个电路的规模 `k` 事先约定好，决定了生成的执行轨迹的长度。 参数 `k` 控制了电路的规模，也就是执行轨迹的长度。

```rust
fn setup(
        ro_config: <RO as CryptographicSponge>::Config,
        step_circuit: &SC,
        aux1: &C1::SetupAux,
        aux2: &C2::SetupAux,
    ) -> Result<public_params::PublicParams<G1, G2, C1, C2, RO, SC, Self>, cyclefold::Error> {
        
        let input = NovaAugmentedCircuitInput::<G1, G2, C1, C2, RO>::Base {
            i,
            z_i,
            vk: G1::ScalarField::ZERO,
        };
      // 构建约束系统
        let circuit = NovaAugmentedCircuit::new(&ro_config, step_circuit, input);
        let _ = NovaConstraintSynthesizer::generate_constraints(circuit, cs.clone())?;

        cs.finalize();

        let shape = R1CSShape::from(cs);
        let shape_secondary = cyclefold::secondary::setup_shape::<G1, G2>()?;

      // 设置承诺方案
        let pp = C1::setup(
            shape.num_vars.max(shape.num_constraints),
            b"nova_pcd_primary_curve",
            aux1,
        );
        let pp_secondary = C2::setup(
            shape_secondary
                .num_vars
                .max(shape_secondary.num_constraints),
            b"nova_pcd_secondary_curve",
            aux2,
        );
				
      // 生成public params
        let mut params = public_params::PublicParams {
            ro_config,
            shape,
            shape_secondary,
            pp,
            pp_secondary,
            digest: G1::ScalarField::ZERO,

            _step_circuit: PhantomData,
            _setup_params: PhantomData,
        };
        let digest = params.hash();
        params.digest = digest;
        Ok(params)
    }
```

### Prover 

例子中实例生成prover则是通过`Nova::compile` 

```rust
let prover: Nova<Local> = Nova::compile(&opts).expect("failed to compile guest program");
```

而在local nova中的 compile 方法则做了两件事情

```rust
impl Prove for Nova<Loval>{
  ...  some inner types ... 
  
  fn new(elf_bytes: &[u8]) -> Result<Self, Self::Error> {
        Ok(Nova::<Local> {
            vm: parse_elf::<Self::Memory>(elf_bytes).map_err(ProofError::from)?,
            _compute: PhantomData,
        })
    }
  
  fn compile(opts: &compile::CompileOpts) -> Result<Self, Self::Error> {
   
    // 若未设置内存，则设置为4mb
    if iopts.memlimit.is_none() {
            iopts.set_memlimit(4);
        }
    
    let elf_path = iopts
            .build(&compile::ForProver::Default)
            .map_err(BuildError::from)?;

        Self::new_from_file(&elf_path)
  }
  
  
  ... other function ...
}
```

可能会有疑问，compile 明明是生成prover ，但是为什么最后执行的是 `Self::new_from_file(&elf_path)`

实际上在最后调用的`new_from_file` 中，底层调用的 Nove<local> 中的 new 方法,大致顺序则是：

`compile--> new_from_file--> new-->parse_elf----> init_vm(elf_file,bytes)`

```rust
pub fn init_vm<M: Memory>(elf: &ElfBytes<LittleEndian>, data: &[u8]) -> Result<NexusVM<M>> {
  let e_entry = elf.ehdr.e_entry as u32;
  
  let load_phdrs = elf
        .segments()
        .unwrap()
        .iter()
        .filter(|phdr| phdr.p_type == PT_LOAD);
  
  let mut vm = NexusVM::new(e_entry);
    for p in load_phdrs {
        let s = p.p_offset as usize;
        let e = (p.p_offset + p.p_filesz) as usize;
        let bytes = &data[s..e];
        vm.init_memory(p.p_vaddr as u32, bytes)?;
    }
  
}
```

*  根据elf文件e_entry字段来确定虚拟机的初始指令地址。
* **加载段数据:** 将 ELF 文件中的各个段（例如代码段、数据段）加载到虚拟机的内存中，每个段被加载到指定的位置。
* **创建虚拟机实例:** 创建一个虚拟机实例，并设置初始状态（如程序计数器指向入口地址）。



### generate proof

```rust
let proof = prover.prove(&pp).expect("failed to prove program");
```

nexus-zkvm/sdk/src/traits.rs

```rust
fn prove(self, pp: &Self::Params) -> Result<Self::Proof, Self::Error>
    where
        Self: Sized,
    {
        Self::prove_with_input::<()>(self, pp, &())
    }
```

> Tips 这里prover实际上是调用的prove_with_inputs 但是这里实际传入的数据时`()` 当时在外部也是可以调用这个方法，比如说我程序的public params 是u8 或者其他类型 
>
> ```rust
> let input: Input = (3, 5);
>  print!("Proving execution of vm...");
> let proof = prover
>         .prove_with_input::<(u32,u32)>(&pp, &input)
>         .expect("failed to prove program");
> ```
>
> 这时候我们PI 则是 `(u32,u32)` 类型

 

```rust
 fn prove_with_input<T>(
        mut self,
        pp: &Self::Params,
        input: &T,
    ) -> Result<Self::Proof, Self::Error>
    where
        T: Serialize + Sized,
{
	
  ... do some initliaized works ...
  
  Ok(Self::Proof {
            proof: prove_seq(pp, tr).map_err(ProofError::from)?,
            view: Self::View {
                output: self.vm.syscalls.get_output(),
                logs: self
                    .vm
                    .syscalls
                    .get_log_buffer()
                    .into_iter()
                    .map(String::from_utf8)
                    .collect::<Result<Vec<_>, _>>()
                    .map_err(TapeError::from)?,
            },
        })
}
```

如之前看到的nova proof 包含两部分

* IVCProof：proof 则是通过 `prove_seq `方法生成
* UncheckedView
  * Output 
  * logs



```rust
pub fn prove_seq(pp: &SeqPP, trace: Trace) -> Result<IVCProof, ProofError> {
    let tr = init_circuit_trace(trace)?;

    let mut proof = prove_seq_step(None, pp, &tr)?;
    for _ in 1..tr.steps() {
        proof = prove_seq_step(Some(proof), pp, &tr)?;
    }

    Ok(proof)
}
```

```rust
pub fn prove_step(
        self,
        params: &PublicParams<G1, G2, C1, C2, RO, SC>,
        step_circuit: &SC,
    ) -> Result<Self, cyclefold::Error> {
      ... do some thing for init ...
      // 是否有non—base,提取上一次的证明和承诺等中间状态数据
      if let Some(non_base) = non_base {
        // 提取中间的承诺值 U, W 以及对应的 u 和 w
        let IVCProofNonBase {
            U, W, U_secondary, W_secondary, u, w, i, z_i,
        } = non_base;
				
        // NIMFSProof 生成新的递归证明
        let proof. = NIMFSProof::<G1, G2, C1, C2, RO>::prove(...);
        
        //构造input 用于下一个递归步骤。该输入包括验证密钥 vk、初始输入 z_0、中间步骤的证明 z_i 等
        let input =. NovaAugmentedCircuitInput::NonBase(NovaAugmentedCircuitNonBaseInput {...};
          
     else{
     // 如果不是non——base，则进行基础步骤处理，构建用于下一步递归证明的参数
       let U = RelaxedR1CSInstance::<G1, C1>::new(&params.shape);
       let W = RelaxedR1CSWitness::zero(&params.shape);
       ...
       
     }
          
     // 创建约束系统并生成电路
     let cs = ConstraintSystem::new_ref();
     let circuit = NovaAugmentedCircuit::new(&params.ro_config, step_circuit, input);  
     
          
     // 生成电路的约束并确保满足性，得到中间证明 z_i
     let z_i = NovaConstraintSynthesizer::generate_constraints(circuit, cs.clone())?;
     
     // 提交，并生成承诺
     let commitment_W = w.commit::<C1>(&params.pp);
     let u = R1CSInstance::<G1, C1> { commitment_W, X: pub_io };
}
```

> prove_step在递归证明系统中进行单步证明生成的过程,它根据之前的状态或初始状态生成新的证明，处理基础步骤和非基础步骤，并在生成约束和承诺后返回更新的 `IVCProof`。

#### NIMFSProof 

Prove 生成一个递归证明。它涉及对电路输入、见证数据的折叠操作和证明生成，主要是通过随机预言机和承诺方案的组合来完成。

```rust
pub fn prove(
        pp: &C1::PP,
        pp_secondary: &C2::PP,
        config: &RO::Config,
        vk: &G1::ScalarField,
        (shape, shape_secondary): (&R1CSShape<G1>, &R1CSShape<G2>),
        (U, W): (&RelaxedR1CSInstance<G1, C1>, &RelaxedR1CSWitness<G1>),
        (U_secondary, W_secondary): (&RelaxedR1CSInstance<G2, C2>, &RelaxedR1CSWitness<G2>),
        (u, w): (&R1CSInstance<G1, C1>, &R1CSWitness<G1>),
    ) -> Result<
        (
            Self,
            (RelaxedR1CSInstance<G1, C1>, RelaxedR1CSWitness<G1>),
            (RelaxedR1CSInstance<G2, C2>, RelaxedR1CSWitness<G2>),
        ),
        Error,
    > {
      // 将vk，以及主要电路的承诺 U 和 u ,承诺喂给预言机，为后续随机数生成做准备。
      let mut random_oracle = RO::new(config);
      
      // 提交承诺
      let (T, _commitment_T) = r1cs::commit_T(shape, pp, U, W, u, w)?;
     	random_oracle.absorb(&vk,&U,&u,&U_secondary,&_commitment_T.into());
      
      // 使用预言机生成随机数，这将用于后续的折叠操作。
      let r_0: G1::BaseField =
      random_oracle.squeeze_field_elements_with_sizes(&[SQUEEZE_ELEMENTS_BIT_SIZE])[0];
      let r_0_scalar: G1::ScalarField =
          unsafe { cast_field_element::<G1::BaseField, G1::ScalarField>(&r_0) };
      
      // 折叠操作
      // 将 U 和 W 与 u 和 w 进行折叠操作，生成新的折叠实例 folded_U 和折叠见证 folded_W，通过承诺 T 和随机数 r_0_scalar 完成折叠。
      let folded_U = U.fold(u, &_commitment_T, &r_0_scalar)?;
			let folded_W = W.fold(w, &T, &r_0_scalar)?;
			
      // secondary Circuit的合成和验证
      // 调用 synthesize，基于主要电路的折叠承诺和随机数生成一个次要电路的证明。E_comm_trace 包含了次要电路的承诺 E。
      let E_comm_trace = secondary::synthesize::<G1, G2, C2>(
          secondary::Circuit {
              g1: U.commitment_E.into(),
              g2: _commitment_T.into(),
              g_out: folded_U.commitment_E.into(),
              r: r_0,
          },
          pp_secondary,
      )?;
      
      // secondary circuit 提交，并喂给预言机
      let (T, commitment_T) = r1cs::commit_T(
            shape_secondary,
            pp_secondary,
            U_secondary,
            W_secondary,
            &E_comm_trace.0,
            &E_comm_trace.1,
        )?;
      
      
			random_oracle.absorb(&commitment_T.into_affine());
      
      
      // 再次生成随机数，用于下一次折叠操作
      let r_1: G1::BaseField =
    		random_oracle.squeeze_field_elements_with_sizes(&[SQUEEZE_ELEMENTS_BIT_SIZE])[0];
			
      // secondary circuit fold 
      // 将 U_secondary 和 W_secondary 进行折叠，生成临时的次要实例 U_secondary_temp 和见证 W_secondary_temp。
      let U_secondary_temp = U_secondary.fold(&E_comm_trace.0, &commitment_T, &r_1)?;
			let W_secondary_temp = W_secondary.fold(&E_comm_trace.1, &T, &r_1)?;

      
      // 生成最终的证明
      // 最终生成两个次要电路的证明 commitment_E_proof 和 commitment_W_proof，包含了次要电路的承诺和中间结果。
      let commitment_E_proof = secondary::Proof { commitment_T, U: E_comm_trace.0 };
			let commitment_W_proof = secondary::Proof { commitment_T, U: W_comm_trace.0 };
     	Ok((proof, (folded_U, folded_W), (U_secondary, W_secondary)))
}
```

* 这个 `prove` 方法通过折叠和承诺操作来生成递归证明，涉及两个电路（主要电路和次要电路）的证明生成和合成。
* 它利用随机预言机生成随机数，确保电路的输入和见证数据经过正确的折叠，并生成最终的递归证明。



## Verify

```rust
pub fn verify(
        &self,
        params: &PublicParams<G1, G2, C1, C2, RO, SC>,
    ) -> Result<(), cyclefold::Error> {
        self.verify_steps(params, self.step_num() as _)
    }
```

```rust
pub fn verify_steps(
        &self,
        params: &PublicParams<G1, G2, C1, C2, RO, SC>,
        num_steps: usize,
    ) -> Result<(), cyclefold::Error> {
      //验证Non_base，non_base 是否存在。如果 non_base 不存在，则说明这是一个基本证明，并返回错误。
      let Some(non_base) = &self.non_base else {
            return Err(NOT_SATISFIED_ERROR);
        };
      
      // num_steps 是用户传入的步骤数量。代码验证这个数量是否与 non_base 中的 i 相等。如果不相等，则返回错误。
      let num_steps = num_steps as u64;
        if num_steps != *i {
            return Err(NOT_SATISFIED_ERROR);
        }
      
      // 生成随机预言机并喂数据
      let mut random_oracle = RO::new(&params.ro_config);

        random_oracle.absorb(&params.digest，&G1::ScalarField::from(*i)，&self.z_0&z_i，U,U_secondary);

        let hash: &G1::ScalarField =
            &random_oracle.squeeze_field_elements(augmented::SQUEEZE_NATIVE_ELEMENTS_NUM)[0];
		//验证从随机预言机生成的 hash 是否与 u.X[1] 相等。如果不相等，则返回错误，说明证明验证失败。
        if hash != &u.X[1] {
            return Err(NOT_SATISFIED_ERROR);
        }
      
      // 验证松弛的 R1CS 约束是否满足，即 U 和 W 是否满足 params.pp，以及 U_secondary 和 W_secondary 是否满足 params.pp_secondary，
      params.shape.is_relaxed_satisfied(U, W, &params.pp)?;
      params.shape_secondary.is_relaxed_satisfied(U_secondary, W_secondary, &params.pp_secondary)?;
      params.shape.is_satisfied(u, w, &params.pp)?;
	
}
```

总结：

* `verify_steps` 方法验证了递归证明的正确性。
* 首先检查步骤数量是否匹配。
* 使用随机预言机生成哈希值，并验证其正确性（确保证明的完整性）。
* 验证松弛 R1CS 约束是否满足，实际上是检查压缩后的证明是否有效
* 如果任何一个步骤失败，都会返回错误，表明证明无效。
