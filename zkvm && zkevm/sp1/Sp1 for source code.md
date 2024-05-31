## SP1 for example 

SP1 commit : ab0efc008737407e609439a4d362395bb2e325b1

![sp1-workflow](./Sp1 overflow.jpg)

PS: 官方目前只实现了groth16 的prove/verify，plonk的并未实现。

Program/src

```rust
#![no_main]
sp1_zkvm::entrypoint!(main);

pub fn main() {
    // NOTE: values of n larger than 186 will overflow the u128 type,
    // resulting in output that doesn't match fibonacci sequence.
    // However, the resulting proof will still be valid!
    let n = sp1_zkvm::io::read::<u32>();   // set in memory when run scripts

    // Write n to public input
    sp1_zkvm::io::commit(&n);

    // Compute the n'th fibonacci number,

    let mut nums = vec![1, 1];

    for _ in 0..n {
        let mut c = nums[nums.len() - 1] + nums[nums.len() - 2];
        c %= 7919;
        nums.push(c);
    }
    // Write the output of the program.

    sp1_zkvm::io::commit(&nums[nums.len() - 2]);  // return to memory that verifier can readß
    sp1_zkvm::io::commit(&nums[nums.len() - 1]);
}

```

Script/src

```rust
use sp1_sdk::{utils, ProverClient, SP1Stdin};


// read program elf 
const FIBO_ELF: &[u8] = include_bytes!("../../program/elf/riscv32im-succinct-zkvm-elf");

fn main() {
    utils::setup_logger();
    // Generate proof.

    let n = 500u32;
    let expected_a = 1926u32;
    let expected_b: u32 = 3194u32;

    let mut stdin = SP1Stdin::new();
    stdin.write(&n);

    let client = ProverClient::new();
    let (pk, vk) = client.setup(FIBO_ELF);
    let mut proof = client.prove(&pk, stdin).expect("proving failed");
    println!("generated proof");

    // Read and verify the output.
    let _ = proof.public_values.read::<u32>();
    let a = proof.public_values.read::<u32>();
    let b = proof.public_values.read::<u32>();
    assert_eq!(a, expected_a);
    assert_eq!(b, expected_b);

    println!("a: {}", a);
    println!("b: {}", b);

    // Verify proof.
    client.verify(&proof, &vk).expect("verification failed");

    // Save proof.
    proof
        .save("proof-with-io.json")
        .expect("saving proof failed");

    println!("succesfully generated and verified proof for the program!")
}

```

从script 的rs文件中，可以看出，sp1 需要讲自定的程序编译为`elf`文件，然后根据生成的elf进行 proof的生成/验证。

* `stdin.write(xxx)` 这个则是作为我们程序的public inputs。
* `let (pk, vk) = client.setup(FIBO_ELF);`  这一步则是会生成pk，vk 在后续proof 生成以及验证的时候是用得到的。
* `client.prove(&pk, stdin)`  proof 生成
* `client.verify(&proof, &vk)` proof verify

在sp1 中支持三种方式的proof 生成/验证

* mockprover ：并不实际执行证明过程或计算。它主要用于模拟证明器的行为，以便在开发和测试过程中使用。mockprover 可以生成假定的证明结果，用于测试系统的其他部分而不进行真正的证明计算。
* local： 在本地机器上运行证明过程。所有的证明计算都在用户的本地环境中进行，而不需要网络连接或远程计算资源。
* remote：在远程服务器上运行证明过程。用户通过网络将证明任务发送到远程服务器，服务器完成计算后将结果返回给用户

### 以localprover 为例

sdk/src/provers/mod.rs

```rust
pub trait Prover: Send + Sync {
    fn id(&self) -> String;

    fn sp1_prover(&self) -> &SP1Prover;

    fn setup(&self, elf: &[u8]) -> (SP1ProvingKey, SP1VerifyingKey);

    /// Prove the execution of a RISCV ELF with the given inputs.
    fn prove(&self, pk: &SP1ProvingKey, stdin: SP1Stdin) -> Result<SP1Proof>;

    /// Generate a compressed proof of the execution of a RISCV ELF with the given inputs.
    fn prove_compressed(&self, pk: &SP1ProvingKey, stdin: SP1Stdin) -> Result<SP1CompressedProof>;

    /// Given an SP1 program and input, generate a Groth16 proof that can be verified on-chain.
    fn prove_groth16(&self, pk: &SP1ProvingKey, stdin: SP1Stdin) -> Result<SP1Groth16Proof>;

    /// Given an SP1 program and input, generate a PLONK proof that can be verified on-chain.
    fn prove_plonk(&self, pk: &SP1ProvingKey, stdin: SP1Stdin) -> Result<SP1PlonkProof>;

    /// Verify that an SP1 proof is valid given its vkey and metadata.
    fn verify(
        &self,
        proof: &SP1Proof,
        vkey: &SP1VerifyingKey,
    ) -> Result<(), MachineVerificationError<CoreSC>> {
        self.sp1_prover()
            .verify(&SP1CoreProofData(proof.proof.clone()), vkey)
    }

    /// Verify that a compressed SP1 proof is valid given its vkey and metadata.
    fn verify_compressed(&self, proof: &SP1CompressedProof, vkey: &SP1VerifyingKey) -> Result<()> {
        self.sp1_prover()
            .verify_compressed(
                &SP1ReduceProof {
                    proof: proof.proof.clone(),
                },
                vkey,
            )
            .map_err(|e| e.into())
    }

    /// Verify that a SP1 Groth16 proof is valid. Verify that the public inputs of the Groth16Proof match
    /// the hash of the VK and the committed public values of the SP1ProofWithPublicValues.
    fn verify_groth16(&self, proof: &SP1Groth16Proof, vkey: &SP1VerifyingKey) -> Result<()> {
        let sp1_prover = self.sp1_prover();

        let groth16_aritfacts = if sp1_prover::build::sp1_dev_mode() {
            sp1_prover::build::groth16_artifacts_dev_dir()
        } else {
            sp1_prover::build::groth16_artifacts_dir()
        };
        sp1_prover.verify_groth16(&proof.proof, vkey, &proof.public_values, &groth16_aritfacts)?;

        Ok(())
    }

    /// Verify that a SP1 PLONK proof is valid given its vkey and metadata.
    fn verify_plonk(&self, _proof: &SP1PlonkProof, _vkey: &SP1VerifyingKey) -> Result<()> {
        Ok(())
    }
}
```

在sp1中提供了 groth16 以及plonk 证明生成和验证。

#### setup

sdk/src/provers/local.rs

```rust
#[instrument(name = "setup", level = "info", skip_all)]
    pub fn setup(&self, elf: &[u8]) -> (SP1ProvingKey, SP1VerifyingKey) {
        let program = Program::from(elf);
        let (pk, vk) = self.core_machine.setup(&program);
        let vk = SP1VerifyingKey { vk };
        let pk = SP1ProvingKey {
            pk,
            elf: elf.to_vec(),
            vk: vk.clone(),
        };
        (pk, vk)
    }
```

PK

```rust
pub struct SP1ProvingKey {
    pub pk: StarkProvingKey<CoreSC>,
    pub elf: Vec<u8>,
    /// Verifying key is also included as we need it for recursion
    pub vk: SP1VerifyingKey,
}
```

VK

```rust
#[derive(Clone)]
pub struct SP1VerifyingKey {
    pub vk: StarkVerifyingKey<CoreSC>,
}
```



#### Prove

```rust
fn prove(&self, pk: &SP1ProvingKey, stdin: SP1Stdin) -> Result<SP1Proof> {
        let proof = self.prover.prove_core(pk, &stdin);
        Ok(SP1ProofWithPublicValues {
            proof: proof.proof.0,
            stdin: proof.stdin,
            public_values: proof.public_values,
        })
    }
```

将核心证明分片，然后设置挑战节点，最后递归生成证明，验证之后再将这些分片链接在一起。



#### Verify

```rust
 fn verify(
        &self,
        proof: &SP1Proof,
        vkey: &SP1VerifyingKey,
    ) -> Result<(), MachineVerificationError<CoreSC>> {
        self.sp1_prover()
            .verify(&SP1CoreProofData(proof.proof.clone()), vkey)
    }
```

