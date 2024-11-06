# Sp1  machine workflow

[日志详情](https://github.com/Chengcheng-S/sp1_demo/blob/master/fibo/run.log)

本节主要探究sp1 程序编译且验证过程中各个模块之间协作。上述的日志则是参考标准，这是一个简单的fibonacci 程序，日志详细的记录了每一步的操作。在自定义的程序编译为elf文件之后，会很好奇proof是如何生成以及被验证的，之前的笔记中已经记录了prove/verify 的各个步骤（当然这只是zk后端的事情）。因为sp1 是个zkvm，还有前端是如何处理这个elf文件的，是由哪几个组件组成的？他们之间又是如何协作的呢？

> 可以这样简化地理解 SP1 zkVM。SP1 zkVM 本身就像一个大型的电路应用，而其中的多个 chip 相当于电路的不同部分，它们共同完成用户程序的执行。这些 chip 通过相互协作，处理用户程序中的操作、数据流和状态变化等。而零知识证明（ZKP）则是用来证明整个执行过程的可信性和正确性。
>
> 具体来说：
>
> ​	•	**SP1 zkVM 是一个大型电路应用**：它的任务是通过电路模拟程序执行，而 chip 就是构建这个电路的基本单元。
>
> ​	•	**chip 之间的协作**：每个 chip 负责处理特定的任务，例如算术操作、哈希运算等，它们共同构成了程序的执行路径。
>
> ​	•	**ZKP 证明可信性**：零知识证明确保即使外界无法直接看到内部 chip 如何具体运作，也能相信这个过程是正确且可信的。
>
> 
>
> **chip 保存关键信息**：在 chip 之间，用户程序的执行轨迹、状态转移、输入输出等关键信息会被记录和传递，这些信息对于生成最终的零知识证明至关重要。

## initialize prover

sp1prover ：

* CoreMachine
* CompressMachine
* WrapMachine
* ShrinkMachine

```rust
// crates/recursion/core/src/stark/mod.rs
pub type RecursionAirWideDeg3<F> = RecursionAir<F, 3>;
pub type RecursionAirWideDeg9<F> = RecursionAir<F, 9>;
pub type RecursionAirWideDeg17<F> = RecursionAir<F, 17>;
```

> Tip:
> 这里的3,9,17 针对的是**递归电路的宽度**。**宽度** 在 AIR 电路中通常指电路中每个约束（constraint）所涉及的变量的数量。或者更直观地，可以理解为电路中每个“门”所连接的“线”的数量。



```rust
/// A STARK for proving RISC-V execution.
pub struct StarkMachine<SC: StarkGenericConfig, A> {
    /// The STARK settings for the RISC-V STARK.
    config: SC,
    /// The chips that make up the RISC-V STARK machine, in order of their execution.
    chips: Vec<Chip<Val<SC>, A>>,

    /// The number of public values elements that the machine uses
    num_pv_elts: usize,
}
```

### init Chips

```rust
// crates/core/machine/src/riscv/mod.rs
pub enum RiscvAir<F: PrimeField32> {
    /// An AIR that containts a preprocessed program table and a lookup for the instructions.
    Program(ProgramChip),
    /// An AIR for the RISC-V CPU. Each row represents a cpu cycle.
    Cpu(CpuChip),
    /// RISC-V operation instructions.
    Add(AddSubChip),
    Bitwise(BitwiseChip),
    Mul(MulChip),
    DivRem(DivRemChip),
    Lt(LtChip),
    ShiftLeft(ShiftLeft),
    ShiftRight(ShiftRightChip),
    ByteLookup(ByteChip<F>),
    /// Memory table relation.
    MemoryInit(MemoryChip),
    MemoryFinal(MemoryChip),
    /// A table for initializing the program memory.
    ProgramMemory(MemoryProgramChip),
    /// precompile && EC operate instructions
    Sha256Extend(ShaExtendChip),
    Sha256Compress(ShaCompressChip),
    Ed25519Add(EdAddAssignChip<EdwardsCurve<Ed25519Parameters>>),
    Ed25519Decompress(EdDecompressChip<Ed25519Parameters>),
    K256Decompress(WeierstrassDecompressChip<SwCurve<Secp256k1Parameters>>),
    Secp256k1Add(WeierstrassAddAssignChip<SwCurve<Secp256k1Parameters>>),
    Secp256k1Double(WeierstrassDoubleAssignChip<SwCurve<Secp256k1Parameters>>),
    /// A precompile for the Keccak permutation.
    KeccakP(KeccakPermuteChip),
    Bn254Add(WeierstrassAddAssignChip<SwCurve<Bn254Parameters>>),
    Bn254Double(WeierstrassDoubleAssignChip<SwCurve<Bn254Parameters>>),
    Bls12381Add(WeierstrassAddAssignChip<SwCurve<Bls12381Parameters>>),
    Bls12381Double(WeierstrassDoubleAssignChip<SwCurve<Bls12381Parameters>>),
    Uint256Mul(Uint256MulChip),
    Bls12381Decompress(WeierstrassDecompressChip<SwCurve<Bls12381Parameters>>),
    Bls12381Fp(FpOpChip<Bls12381BaseField>),
    Bls12381Fp2Mul(Fp2MulAssignChip<Bls12381BaseField>),
    Bls12381Fp2AddSub(Fp2AddSubAssignChip<Bls12381BaseField>),
    Bn254Fp(FpOpChip<Bn254BaseField>),
    Bn254Fp2Mul(Fp2MulAssignChip<Bn254BaseField>),
    Bn254Fp2AddSub(Fp2AddSubAssignChip<Bn254BaseField>),
}
```

 

```rust
pub enum RecursionAir<F: PrimeField32 + BinomiallyExtendable<D>, const DEGREE: usize> {
    Program(ProgramChip),
    Cpu(CpuChip<F, DEGREE>),
    MemoryGlobal(MemoryGlobalChip),
    Poseidon2Wide(Poseidon2WideChip<DEGREE>),
    FriFold(FriFoldChip<DEGREE>),
    RangeCheck(RangeCheckChip<F>),
    Multi(MultiChip<DEGREE>),
    ExpReverseBitsLen(ExpReverseBitsLenChip<DEGREE>),
}
```

BaseAir<plonky3里的trait，也就是说这个machineAir 是基于Plonky3 实现的>

```rust
// crates/stark/src/air/machine.rs
/// An AIR that is part of a multi table AIR arithmetization.
pub trait MachineAir<F: Field>: BaseAir<F> + 'static + Send + Sync {
    /// The execution record containing events for producing the air trace.
    type Record: MachineRecord;

    /// The program that defines the control flow of the machine.
    type Program: MachineProgram<F>;

    /// A unique identifier for this AIR as part of a machine.
    fn name(&self) -> String;

    /// Generate the trace for a given execution record.
    ///
    /// - `input` is the execution record containing the events to be written to the trace.
    /// - `output` is the execution record containing events that the `MachineAir` can add to the
    ///   record such as byte lookup requests.
    fn generate_trace(&self, input: &Self::Record, output: &mut Self::Record) -> RowMajorMatrix<F>;

    /// Generate the dependencies for a given execution record.
    fn generate_dependencies(&self, input: &Self::Record, output: &mut Self::Record) {
        self.generate_trace(input, output);
    }

    /// Whether this execution record contains events for this air.
    fn included(&self, shard: &Self::Record) -> bool;

    /// The width of the preprocessed trace.
    fn preprocessed_width(&self) -> usize {
        0
    }

    /// Generate the preprocessed trace given a specific program.
    fn generate_preprocessed_trace(&self, _program: &Self::Program) -> Option<RowMajorMatrix<F>> {
        None
    }

    /// Specifies whether it's trace should be part of either the global or local commit.
    fn commit_scope(&self) -> InteractionScope {
        InteractionScope::Local
    }
}
```

