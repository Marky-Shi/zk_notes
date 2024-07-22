## Sp1 细节补充

之前的所有的笔记都是基于zkp后端，也就是单纯的prove流程的，但是会存在一定的割裂感，完全不清晰 `业务===>电路 ====> 业务`的这个流程， 这里就会进行相应的补充。

之前提到sp1由原先的groth16 切换到了plonk，证明的后端也则是由gnark完成的，这里补充一下，由于macos 是基于arm的，会导致rust ----ffi----> go 的时候会出现link cc的error，强调一下，用的是这个`bindgen` 这个rust库，当时确实没少被折腾，hhhhh。

熟悉gnark的朋友肯定知道，使用这个库一般就是定义电路的结构体，然后定义电路要做的事情，如：

```go
type Circuit struct{
  X frontend.Variable `gnark:",public"`
	Y frontend.Variable `gnark:",public"` 
}
// X**3 +x + 5 = Y
func (circuit *Circuit) Define(api frontend.API) error {
	x3 := api.Mul(circuit.X, circuit.X, circuit.X)
	api.AssertIsEqual(circuit.Y, api.Add(x3, circuit.X, 5))
	return nil
}
```

那咱们看看sp1使用gnark，到底做了什么事情吧。

首先需要看一下PK，vk的产生过程，或者PK，vk中包含了什么信息

core/src/stark/machine.rs 

```rust
#[instrument("setup machine", level = "debug", skip_all)]
    pub fn setup(&self, program: &A::Program) -> (StarkProvingKey<SC>, StarkVerifyingKey<SC>) {
       let mut named_preprocessed_traces = tracing::debug_span!("generate preprocessed traces")
            .in_scope(|| {
                self.chips()
                    .iter()
                    .map(|chip| {
                      do something
              });
        // 这两步就是根据program 生成trace，并根据height进行排序。   
        named_preprocessed_traces.sort_by_key(|(_, trace)| Reverse(trace.height()));
              
       // 处理chip 以及trace信息
        let (chip_information, domains_and_traces): (Vec<_>, Vec<_>) = named_preprocessed_traces
            .iter()
            .map(|(name, trace)| {
              do something
              }).unzip();
      // 提交预处理的traces
      let (commit, data) = tracing::debug_span!("commit to preprocessed traces")
            .in_scope(|| pcs.commit(domains_and_traces));
      // chip 进行排序
      let chip_ordering = named_preprocessed_traces
            .iter()
            .enumerate()
            .map(|(i, (name, _))| (name.to_owned(), i))
            .collect::<HashMap<_, _>>();
              
     // 处理trace
     // Get the preprocessed traces
        let traces = named_preprocessed_traces
            .into_iter()
            .map(|(_, trace)| trace)
            .collect::<Vec<_>>(); 
    // 自定义程序的起始堆栈
     let pc_start = program.pc_start();
			// 构建Pk，VK
        (
            StarkProvingKey {
                commit: commit.clone(),
                pc_start,
                traces,
                data,
                chip_ordering: chip_ordering.clone(),
            },
            StarkVerifyingKey {
                commit,
                pc_start,
                chip_information,
                chip_ordering,
            },
        )
}
```

从setup的过程可以看出，pk，vk生成的过程实际上就是记录自定义程序执行过程堆栈信息，以及多项式承诺，chip处理的顺序。换言之就是将自定义的程序转换为只有zkp（此处是plonk）可以构建的zk电路并且用来生成proof。理解这个前提之后，咱们继续往下看。

接着setup完了之后，接着就到了prove阶段了，这个还是很有意思的，sp1中包含三种类型的proof（想啥呢，不是stark/snark之分啊，那是next next next step了）由于之前总结过了证明生成的整个过程，因此本片中会略过这一步骤的。

以local prover 为例

```rust
fn prove<'a>(
        &'a self,
        pk: &SP1ProvingKey,
        stdin: SP1Stdin,
        opts: SP1ProverOpts,
        context: SP1Context<'a>,
        kind: SP1ProofKind,
    ) -> Result<SP1ProofWithPublicValues> {
      let proof = self.prover.prove_core(pk, &stdin, opts, context)?;
      //1.sp1 core proof 
      if kind == SP1ProofKind::Core {
            return Ok(SP1ProofWithPublicValues {
                proof: SP1Proof::Core(proof.proof.0),
                stdin: proof.stdin,
                public_values: proof.public_values,
                sp1_version: self.version().to_string(),
            });
        }
      
      //2. sp1 compress proof
      let deferred_proofs = stdin.proofs.iter().map(|p| p.0.clone()).collect();
      let public_values = proof.public_values.clone();
      let reduce_proof = self.prover.compress(&pk.vk, proof, deferred_proofs, opts)?;
      if kind == SP1ProofKind::Compressed {
            return Ok(SP1ProofWithPublicValues {
                proof: SP1Proof::Compressed(reduce_proof.proof),
                stdin,
                public_values,
                sp1_version: self.version().to_string(),
            });
        }
      // 3. plonk proof
      let compress_proof = self.prover.shrink(reduce_proof, opts)?;
      let outer_proof = self.prover.wrap_bn254(compress_proof, opts)?;

      let plonk_bn254_aritfacts = if sp1_prover::build::sp1_dev_mode() {
            sp1_prover::build::try_build_plonk_bn254_artifacts_dev(
                &self.prover.wrap_vk,
                &outer_proof.proof,
            )
        } else {
            try_install_plonk_bn254_artifacts()
        };
      let proof = self
            .prover
            .wrap_plonk_bn254(outer_proof, &plonk_bn254_aritfacts);
      if kind == SP1ProofKind::Plonk {
            return Ok(SP1ProofWithPublicValues {
                proof: SP1Proof::Plonk(proof),
                stdin,
                public_values,
                sp1_version: self.version().to_string(),
            });
        }
      
}
```

这里着重说一下，想要本地运行sp1 生成plonk proof，需要确保你有足够强悍的硬件性能（1.目前这个项目并未实现GPU加速，所以你的CPU要是分的硬核；2.若是本地生成plonk出了构建电路之外（当然sp1也提供了下载电路配置的选项）面还有进行FFT，插值，置换等多个步骤，这对内存也是极大的考验，因此，没有128G的内存，就不要考虑了。）

到了plonk 这里没看到PK，VK，最起码明面上没看到PK，VK，那你就好奇了，咋前面还说了那么多的PK，VK，纯纯浪费时间不是？

```rust
let plonk_bn254_aritfacts = if sp1_prover::build::sp1_dev_mode() {
            sp1_prover::build::try_build_plonk_bn254_artifacts_dev(
                &self.prover.wrap_vk,
                &outer_proof.proof,
            )
        } else {
            try_install_plonk_bn254_artifacts()
        };
```

这一步是用来本地生成/下载plonk电路组件的，需要用到的一个是prover.wrap_vk(StarkProveKey，sp1是基于starkmachine的)，一个则是ouer_proof (proof压缩之后的变种)，需要这两个关键的变量来生成plonk用到的组件。

这里就需要回到prover构建的时候了，实际上就是Sp1Prover::new

```rust
 #[instrument(name = "initialize prover", level = "debug", skip_all)]
 pub fn new() -> Self {
 	//recursive verifier setup pk.vk，递归电路部分
    let recursion_program =
            SP1RecursiveVerifier::<InnerConfig, _>::build(core_prover.machine());
    let compress_machine = ReduceAir::machine(InnerSC::default());
    let compress_prover = C::CompressProver::new(compress_machine);
    let (rec_pk, rec_vk) = compress_prover.setup(&recursion_program);
	
   // deferred program and keys.
   let deferred_program =
            SP1DeferredVerifier::<InnerConfig, _, _>::build(compress_prover.machine());
   let (deferred_pk, deferred_vk) = compress_prover.setup(&deferred_program);
	
   
   //reduce program and keys
   let compress_program = SP1CompressVerifier::<InnerConfig, _, _>::build(
            compress_prover.machine(),
            &rec_vk,
            &deferred_vk,
        );
    let (compress_pk, compress_vk) = compress_prover.setup(&compress_program);

    // compress program, machine, and keys.
     let shrink_program = SP1RootVerifier::<InnerConfig, _, _>::build(
            compress_prover.machine(),
            &compress_vk,
            RecursionProgramType::Shrink,
        );
     let shrink_machine = CompressAir::wrap_machine_dyn(InnerSC::compressed());
     let shrink_prover = C::ShrinkProver::new(shrink_machine);
     let (shrink_pk, shrink_vk) = shrink_prover.setup(&shrink_program);

     // wrap program, machine, and keys.
     let wrap_program = SP1RootVerifier::<InnerConfig, _, _>::build(
            shrink_prover.machine(),
            &shrink_vk,
            RecursionProgramType::Wrap,
        );
     let wrap_machine = WrapAir::wrap_machine(OuterSC::default());
     let wrap_prover = C::WrapProver::new(wrap_machine);
     let (wrap_pk, wrap_vk) = wrap_prover.setup(&wrap_program);

  	Self{
      ...
      wrap_pk,
      wrap_vk,
      ...
   }   
}
```

可以看到，prover创建的过程就是层层递进（套娃，极度依赖于上一步的program），而这里每一步keys的生成，又回到了上一步的setup了。

在下一步gnark侧，plonk电路的定义用到了一个名为`constraints.json`的文件，那你就会好奇，说了那那么多也没提到"constraints.json"这个咋来的啊。别急之前都铺垫好了，就差最后一步了。

```rust
/// Builds the PLONK circuit locally.
    pub fn build<C: Config>(constraints: Vec<Constraint>, witness: Witness<C>, build_dir: PathBuf) {
        let serialized = serde_json::to_string(&constraints).unwrap();

        // Write constraints.
        let constraints_path = build_dir.join("constraints.json");
        let mut file = File::create(constraints_path).unwrap();
        file.write_all(serialized.as_bytes()).unwrap();

        // Write witness.
        let witness_path = build_dir.join("witness.json");
        let gnark_witness = GnarkWitness::new(witness);
        let mut file = File::create(witness_path).unwrap();
        let serialized = serde_json::to_string(&gnark_witness).unwrap();
        file.write_all(serialized.as_bytes()).unwrap();

        build_plonk_bn254(build_dir.to_str().unwrap());

        // Write the corresponding asset files to the build dir.
        let sp1_verifier_path = build_dir.join("SP1Verifier.sol");
        let vkey_hash = Self::get_vkey_hash(&build_dir);
        let sp1_verifier_str = include_str!("../assets/SP1Verifier.txt")
            .replace("{SP1_CIRCUIT_VERSION}", SP1_CIRCUIT_VERSION)
            .replace(
                "{VERIFIER_HASH}",
                format!("0x{}", hex::encode(vkey_hash)).as_str(),
            );
        let mut sp1_verifier_file = File::create(sp1_verifier_path).unwrap();
        sp1_verifier_file
            .write_all(sp1_verifier_str.as_bytes())
            .unwrap();
    }
```

看不懂没关系，网上再倒一步直接豁然开朗了。

```rust
pub fn build_plonk_bn254_artifacts(
    template_vk: &StarkVerifyingKey<OuterSC>,
    template_proof: &ShardProof<OuterSC>,
    build_dir: impl Into<PathBuf>,
) {
    let build_dir = build_dir.into();
    std::fs::create_dir_all(&build_dir).expect("failed to create build directory");
    let (constraints, witness) = build_constraints_and_witness(template_vk, template_proof);
    PlonkBn254Prover::build(constraints, witness, build_dir);
}
```

一句话概括，这里做了个转换，使用之前构建的的warp_vk,以及proof 为依据，构建plonk的**constraints system**以及**witness**，存储到指定路径下，是为 `constraints.json`。

下一步 plonk proof，因为这里才是ffi 调用gnark的部分。 将 plonk bn254 工件构建到给定的目录中。
这可能需要一段时间，因为它需要先生成一个虚拟证明，然后需要编译电路。这一步也走完之后，就看时plonk proof的生成了

recursion/gnark-ffi/go/sp1.go

```go
func (circuit *Circuit) Define(api frontend.API) error {
	// Get the file name from an environment variable.
	fileName := os.Getenv("CONSTRAINTS_JSON")
	if fileName == "" {
		fileName = "constraints.json"
	}
	// Read the file.
	data, err := os.ReadFile(fileName)
	if err != nil {
		return fmt.Errorf("failed to read file: %w", err)
	}

	// Deserialize the JSON data into a slice of Instruction structs.
	var constraints []Constraint
	err = json.Unmarshal(data, &constraints)
	if err != nil {
		return fmt.Errorf("error deserializing JSON: %v", err)
	}
  
  ......
  // Iterate through the instructions and handle each opcode.
	for _, cs := range constraints {
		switch cs.Opcode {
		case "ImmV":
			vars[cs.Args[0][0]] = frontend.Variable(cs.Args[1][0])
		case "ImmF":
			felts[cs.Args[0][0]] = babybear.NewF(cs.Args[1][0])
		case "ImmE":
			exts[cs.Args[0][0]] = babybear.NewE(cs.Args[1])
		case "AddV":
			vars[cs.Args[0][0]] = api.Add(vars[cs.Args[1][0]], vars[cs.Args[2][0]])
		case "AddF":
			felts[cs.Args[0][0]] = fieldAPI.AddF(felts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "AddE":
			exts[cs.Args[0][0]] = fieldAPI.AddE(exts[cs.Args[1][0]], exts[cs.Args[2][0]])
		case "AddEF":
			exts[cs.Args[0][0]] = fieldAPI.AddEF(exts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "SubV":
			vars[cs.Args[0][0]] = api.Sub(vars[cs.Args[1][0]], vars[cs.Args[2][0]])
		case "SubF":
			felts[cs.Args[0][0]] = fieldAPI.SubF(felts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "SubE":
			exts[cs.Args[0][0]] = fieldAPI.SubE(exts[cs.Args[1][0]], exts[cs.Args[2][0]])
		case "SubEF":
			exts[cs.Args[0][0]] = fieldAPI.SubEF(exts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "MulV":
			vars[cs.Args[0][0]] = api.Mul(vars[cs.Args[1][0]], vars[cs.Args[2][0]])
		case "MulF":
			felts[cs.Args[0][0]] = fieldAPI.MulF(felts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "MulE":
			exts[cs.Args[0][0]] = fieldAPI.MulE(exts[cs.Args[1][0]], exts[cs.Args[2][0]])
		case "MulEF":
			exts[cs.Args[0][0]] = fieldAPI.MulEF(exts[cs.Args[1][0]], felts[cs.Args[2][0]])
		case "DivE":
			exts[cs.Args[0][0]] = fieldAPI.DivE(exts[cs.Args[1][0]], exts[cs.Args[2][0]])
		case "NegE":
			exts[cs.Args[0][0]] = fieldAPI.NegE(exts[cs.Args[1][0]])
		case "InvE":
			exts[cs.Args[0][0]] = fieldAPI.InvE(exts[cs.Args[1][0]])
		case "Num2BitsV":
			numBits, err := strconv.Atoi(cs.Args[2][0])
			if err != nil {
				return fmt.Errorf("error converting number of bits to int: %v", err)
			}
			bits := api.ToBinary(vars[cs.Args[1][0]], numBits)
			for i := 0; i < len(cs.Args[0]); i++ {
				vars[cs.Args[0][i]] = bits[i]
			}
		case "Num2BitsF":
			bits := fieldAPI.ToBinary(felts[cs.Args[1][0]])
			for i := 0; i < len(cs.Args[0]); i++ {
				vars[cs.Args[0][i]] = bits[i]
			}
		case "Permute":
			state := [3]frontend.Variable{vars[cs.Args[0][0]], vars[cs.Args[1][0]], vars[cs.Args[2][0]]}
			hashAPI.PermuteMut(&state)
			vars[cs.Args[0][0]] = state[0]
			vars[cs.Args[1][0]] = state[1]
			vars[cs.Args[2][0]] = state[2]
		case "PermuteBabyBear":
			var state [16]babybear.Variable
			for i := 0; i < 16; i++ {
				state[i] = felts[cs.Args[i][0]]
			}
			hashBabyBearAPI.PermuteMut(&state)
			for i := 0; i < 16; i++ {
				felts[cs.Args[i][0]] = state[i]
			}
		case "SelectV":
			vars[cs.Args[0][0]] = api.Select(vars[cs.Args[1][0]], vars[cs.Args[2][0]], vars[cs.Args[3][0]])
		case "SelectF":
			felts[cs.Args[0][0]] = fieldAPI.SelectF(vars[cs.Args[1][0]], felts[cs.Args[2][0]], felts[cs.Args[3][0]])
		case "SelectE":
			exts[cs.Args[0][0]] = fieldAPI.SelectE(vars[cs.Args[1][0]], exts[cs.Args[2][0]], exts[cs.Args[3][0]])
		case "Ext2Felt":
			out := fieldAPI.Ext2Felt(exts[cs.Args[4][0]])
			for i := 0; i < 4; i++ {
				felts[cs.Args[i][0]] = out[i]
			}
		case "AssertEqV":
			api.AssertIsEqual(vars[cs.Args[0][0]], vars[cs.Args[1][0]])
		case "AssertEqF":
			fieldAPI.AssertIsEqualF(felts[cs.Args[0][0]], felts[cs.Args[1][0]])
		case "AssertEqE":
			fieldAPI.AssertIsEqualE(exts[cs.Args[0][0]], exts[cs.Args[1][0]])
		case "PrintV":
			api.Println(vars[cs.Args[0][0]])
		case "PrintF":
			f := felts[cs.Args[0][0]]
			api.Println(f.Value)
		case "PrintE":
			e := exts[cs.Args[0][0]]
			api.Println(e.Value[0].Value)
			api.Println(e.Value[1].Value)
			api.Println(e.Value[2].Value)
			api.Println(e.Value[3].Value)
		case "WitnessV":
			i, err := strconv.Atoi(cs.Args[1][0])
			if err != nil {
				panic(err)
			}
			vars[cs.Args[0][0]] = circuit.Vars[i]
		case "WitnessF":
			i, err := strconv.Atoi(cs.Args[1][0])
			if err != nil {
				panic(err)
			}
			felts[cs.Args[0][0]] = circuit.Felts[i]
		case "WitnessE":
			i, err := strconv.Atoi(cs.Args[1][0])
			if err != nil {
				panic(err)
			}
			exts[cs.Args[0][0]] = circuit.Exts[i]
		case "CommitVkeyHash":
			element := vars[cs.Args[0][0]]
			api.AssertIsEqual(circuit.VkeyHash, element)
		case "CommitCommitedValuesDigest":
			element := vars[cs.Args[0][0]]
			api.AssertIsEqual(circuit.CommitedValuesDigest, element)
		case "CircuitFelts2Ext":
			exts[cs.Args[0][0]] = babybear.Felts2Ext(felts[cs.Args[1][0]], felts[cs.Args[2][0]], felts[cs.Args[3][0]], felts[cs.Args[4][0]])
		default:
			return fmt.Errorf("unhandled opcode: %s", cs.Opcode)
		}
	}
  return nill 
}
```

说白了就两件事情：

1. 获取 CONSTRAINTS_JSON 源数据
2. 根据CONSTRAINTS_JSON 的指令集分别处理对应的操作码

也就是使用CONSTRAINTS_JSON 来构建约束系统。之后就是proof生成以及验证的过程了（纯密码学的东西了）。

再提一嘴，plonk 端的pk，vk 则是基于BN_256 曲线生成的，至于witness 直接从 CONSTRAINTS_JSON 获取就行了。

```go
func Prove(dataDir string, witnessPath string) Proof {
	// Sanity check the required arguments have been provided.
	if dataDir == "" {
		panic("dataDirStr is required")
	}
	os.Setenv("CONSTRAINTS_JSON", dataDir+"/"+constraintsJsonFile)

	// Read the R1CS.
	scsFile, err := os.Open(dataDir + "/" + circuitPath)
	if err != nil {
		panic(err)
	}
	scs := plonk.NewCS(ecc.BN254)
	scs.ReadFrom(scsFile)
	defer scsFile.Close()

	// Read the proving key.
	pkFile, err := os.Open(dataDir + "/" + pkPath)
	if err != nil {
		panic(err)
	}
	pk := plonk.NewProvingKey(ecc.BN254)
	bufReader := bufio.NewReaderSize(pkFile, 1024*1024)
	pk.UnsafeReadFrom(bufReader)
	defer pkFile.Close()

	// Read the verifier key.
	vkFile, err := os.Open(dataDir + "/" + vkPath)
	if err != nil {
		panic(err)
	}
	vk := plonk.NewVerifyingKey(ecc.BN254)
	vk.ReadFrom(vkFile)
	defer vkFile.Close()

	// Read the file.
	data, err := os.ReadFile(witnessPath)
	if err != nil {
		panic(err)
	}

	// Deserialize the JSON data into a slice of Instruction structs
	var witnessInput WitnessInput
	err = json.Unmarshal(data, &witnessInput)
	if err != nil {
		panic(err)
	}

	// Generate the witness.
	assignment := NewCircuit(witnessInput)
	witness, err := frontend.NewWitness(&assignment, ecc.BN254.ScalarField())
	if err != nil {
		panic(err)
	}
	publicWitness, err := witness.Public()
	if err != nil {
		panic(err)
	}

	// Generate the proof.
	proof, err := plonk.Prove(scs, pk, witness)
	if err != nil {
		panic(err)
	}

	// Verify proof.
	err = plonk.Verify(proof, vk, publicWitness)
	if err != nil {
		panic(err)
	}

	return NewSP1PlonkBn254Proof(&proof, witnessInput)
}
```

