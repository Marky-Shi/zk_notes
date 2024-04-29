## Jolt zkvm implement

### Sart by demo

在jolt中新建一个目录一般会生成一下的文件

```shell
ls -l
total 136
-rw-r--r--  1 scc  staff  53766 Apr 18 15:05 Cargo.lock
-rw-r--r--@ 1 scc  staff    612 Apr 24 16:41 Cargo.toml
drwxr-xr-x  4 scc  staff    128 Apr 18 15:05 guest
-rw-r--r--  1 scc  staff     83 Apr 18 15:05 rust-toolchain.toml
-rw-r--r--  1 scc  staff    111 Apr 18 15:07 rustfmt.toml
drwxr-xr-x  3 scc  staff     96 Apr 18 15:05 src
drwxr-xr-x@ 6 scc  staff    192 Apr 24 20:47 target
```

* `guest` 中可以定制化自己的业务，方法的编写格式则是如下：

  * ```rust
    #![cfg_attr(feature = "guest", no_std)]
    #[jolt::provable(max_input_size = 10000, max_output_size = 10000)]
    fn sum(input: &[u8]) -> u32 {
        let mut sum = 0u32;
    
        input.iter().for_each(|x| sum += *x as u32);
        sum
    }
    ```

* `src` 则是生成证明，验证证明的验证程序。

  * ```rust
    fn main(){
    		let (prove_sum, verify_sum) = guest::build_sum();
    
        let (output, proof) = prove_sum(&[5u8; 100]);
        let is_valid = verify_sum(proof);
    }
    ```

对比与sp1，先把程序编译为elf，然后在验证程序中将elf加载到内存中，进行参数获取、证明生成、证明验证的工作。jolt则是通过 `jolt::provable` 这个属性达到程序编译的目的，而证明的生成和验证都是在`src` 中进行的。

### Jolt::provable attribute

```rust
// jolt-sdk/macro/lib.rs line 23
#[proc_macro_attribute]
pub fn provable(attr: TokenStream, item: TokenStream) -> TokenStream {
    let attr = parse_macro_input!(attr as AttributeArgs);  // 解析自定程序中设置的属性， 包含stack-size memory-size  max_input_size min_input_size 
    let func = parse_macro_input!(item as ItemFn);  // fn 相关的设置
    MacroBuilder::new(attr, func).build() // build fn
}
```



```rust
struct MacroBuilder {
    attr: AttributeArgs,
    func: ItemFn,
    std: bool,
    func_args: Vec<(Ident, Box<Type>)>,
}
```

```rust
fn build(&self) -> TokenStream {
        let build_fn = self.make_build_fn();
        let execute_fn = self.make_execute_function();
        let analyze_fn = self.make_analyze_function();
        let preprocess_fn = self.make_preprocess_func();
        let prove_fn = self.make_prove_func();

        let main_fn = if let Some(func) = self.get_func_selector() {
            if *self.get_func_name() == func {
                self.make_main_func()
            } else {
                quote! {}
            }
        } else {
            self.make_main_func()
        };

        quote! {
            #build_fn
            #execute_fn
            #analyze_fn
            #preprocess_fn
            #prove_fn
            #main_fn
        }
        .into()
    }
```

```rust
// lib.rs
fn make_prove_func(&self) -> TokenStream2 {
	... do some thing
  
  let (jolt_proof, jolt_commitments) = RV32IJoltVM::prove(
                    io_device,
                    bytecode_trace,
                    memory_trace,
                    instruction_trace,
                    circuit_flags,
                    preprocessing,
                );

                #handle_return

                let proof = jolt::Proof {
                    proof: jolt_proof,
                    commitments: jolt_commitments,
                };
  ... do some thing
}
```

```rust
fn make_build_fn(&self) -> TokenStream2 {
  ... do something
  qute!{
    ... do something
    let prove_closure = move |#inputs| {
             let program = (*program).clone();
             let preprocessing = (*preprocessing).clone();
             #prove_fn_name(program, preprocessing, #(#input_names),*)
                };


     let verify_closure = move |proof: jolt::Proof| {
               let program = (*program_cp).clone();
               let preprocessing = (*preprocessing_cp).clone();
                    RV32IJoltVM::verify(preprocessing, proof.proof, proof.commitments).is_ok()
               };

     (prove_closure, verify_closure)
    ...
  }
  ...
}
```

* `make_prove_fn`  主要核心则在 `RV32IJoltVM::prove`
* `make_build_fn` 的核心则是在`RV32IJoltVM::verify`
* 并不是说其他的几个方法不重要，要是想专心研究编译器的玩家，可以自行食用。

而关于`prove`  `verify` 这两个核心方法则是在：`jolt-core/src/jolt/vm/mod.rs` 下的 jolt strutct中

开始prove 的工作

```rust
#[tracing::instrument(skip_all, name = "Jolt::prove")]
    fn prove(
        program_io: JoltDevice,
        bytecode_trace: Vec<BytecodeRow>,
        memory_trace: Vec<[MemoryOp; MEMORY_OPS_PER_INSTRUCTION]>,
        instructions: Vec<Option<Self::InstructionSet>>,
        circuit_flags: Vec<F>,
        preprocessing: JoltPreprocessing<F, PCS>,
    ) -> (
        JoltProof<C, M, F, PCS, Self::InstructionSet, Self::Subtables>,
        JoltCommitments<PCS>,
    ) {
  	
      // 准备多项式
      let (memory_polynomials, read_timestamps) = ReadWriteMemory::new(
            &program_io,
            load_store_flags,
            &preprocessing.read_write_memory,
            padded_memory_trace,
        );

        let (bytecode_polynomials, range_check_polys) = rayon::join(
            || BytecodePolynomials::<F, PCS>::new(&preprocessing.bytecode, bytecode_trace),
            || RangeCheckPolynomials::<F, PCS>::new(read_timestamps),
        );

        let jolt_polynomials = JoltPolynomials {
            bytecode: bytecode_polynomials,
            read_write_memory: memory_polynomials,
            timestamp_range_check: range_check_polys,
            instruction_lookups: instruction_polynomials,
        };
      	// 封装多项式 
      	let mut jolt_commitments = jolt_polynomials.commit(&preprocessing.generators);
      
        // 多项式=====> r1cs 
      	let (spartan_key, witness_segments, r1cs_commitments) = Self::r1cs_setup(
            padded_trace_length,
            RAM_START_ADDRESS - program_io.memory_layout.ram_witness_offset,
            &instructions,
            &jolt_polynomials,
            circuit_flags,
            &preprocessing.generators,
        );
      	
      	// vk 和其他的输出结果
      	transcript.append_scalar(b"spartan key", &spartan_key.vk_digest);

        jolt_commitments.r1cs = Some(r1cs_commitments);

        jolt_commitments.append_to_transcript(&mut transcript);	
      
      // 生成 prove 
       let bytecode_proof = BytecodeProof::prove_memory_checking(
            &preprocessing.bytecode,
            &jolt_polynomials.bytecode,
            &mut transcript,
        );

        let instruction_proof = InstructionLookupsProof::prove(
            &jolt_polynomials.instruction_lookups,
            &preprocessing.instruction_lookups,
            &mut transcript,
        );

        let memory_proof = ReadWriteMemoryProof::prove(
            &preprocessing.read_write_memory,
            &jolt_polynomials,
            &program_io,
            &mut transcript,
        );
      
      let r1cs_proof =
            R1CSProof::prove(spartan_key, witness_segments, &mut transcript).expect("proof failed");
      
      // 整合
      let jolt_proof = JoltProof {
            trace_length,
            program_io,
            bytecode: bytecode_proof,
            read_write_memory: memory_proof,
            instruction_lookups: instruction_proof,
            r1cs: r1cs_proof,
        };
      
      (jolt_proof, jolt_commitments)
      
	}
```

在R1CS 生成proof的时候底层用到了spartan进行多项式承诺（和sum-check有关）。



verify,prove的底层与 hyrax有关

```rust
#[tracing::instrument(skip_all, name = "UniformSpartanProof::prove_precommitted")]
    pub fn prove_precommitted(
        key: &UniformSpartanKey<F>,
        witness_segments: Vec<Vec<F>>,
        transcript: &mut ProofTranscript,
    ) -> Result<Self, SpartanError> {
      
      ... do something 
      let opening_proof = C::batch_prove(
            &witness_segment_polys_ref,
            r_y_point,
            &witness_evals,
            BatchType::Big,
            transcript,
        );
      ...
}
```



```rust
 #[tracing::instrument(skip_all, name = "SNARK::verify")]
    pub fn verify_precommitted(
        &self,
        witness_segment_commitments: Vec<&C::Commitment>,
        key: &UniformSpartanKey<F>,
        io: &[F],
        generators: &C::Setup,
        transcript: &mut ProofTranscript,
    ) -> Result<(), SpartanError> {
      ... do something
      C::batch_verify(
            &self.opening_proof,
            generators,
            r_y_point,
            &self.claimed_witnesss_evals,
            &witness_segment_commitments,
            transcript,
        )
        .map_err(|_| SpartanError::InvalidPCSProof)?;
		Ok(()) 
}
```

而 `C::branch_verify` `C::branch_prove`  源码则在 jolt-core/src/poly/commitment/hyrax.rs 中 
