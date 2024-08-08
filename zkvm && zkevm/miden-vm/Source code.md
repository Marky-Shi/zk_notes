## Source code

### example

```rust
use miden_vm::{execute_iter, prove, verify, Assembler, DefaultHost, ProvingOptions, StackInputs};

fn main() {
    let assember = Assembler::default();
    let program = assember
        .compile(
            r#"
            begin push.2 push.7 add end
        "#,
        )
        .unwrap();

    let stack_input = StackInputs::default();

    let mut host = DefaultHost::default();

    //try use hash_FN rpo256 but slowly and Blake3_192 by default
    let prove_options = ProvingOptions::with_96_bit_security(true);
  	// let prove_options = ProvingOptions::default();

    let (program_outputs, proofs) =
        prove(&program, stack_input.clone(), &mut host, prove_options).unwrap();

    println!("program_outputs: {:?}", &program_outputs);
    // println!("proofs: {:?}", &proofs.proof.fri_proof);

    let security_level =
        verify(program.clone().into(), stack_input.clone(), program_outputs, proofs).unwrap();

    println!("security level: {}", security_level);

    /*
    VmState { clk: 0, ctx: ContextId(0), op: None, asmop: None, fmp: 1073741824, stack: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 1, ctx: ContextId(0), op: Some(Span), asmop: None, fmp: 1073741824, stack: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 2, ctx: ContextId(0), op: Some(Push(2)), asmop: None, fmp: 1073741824, stack: [2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 3, ctx: ContextId(0), op: Some(Push(7)), asmop: None, fmp: 1073741824, stack: [7, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 4, ctx: ContextId(0), op: Some(Add), asmop: None, fmp: 1073741824, stack: [9, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 5, ctx: ContextId(0), op: Some(Noop), asmop: None, fmp: 1073741824, stack: [9, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    VmState { clk: 6, ctx: ContextId(0), op: Some(End), asmop: None, fmp: 1073741824, stack: [9, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], memory: [] }
    */

    for vm_state in execute_iter(&program, stack_input, host) {
        match vm_state {
            Ok(vm_state) => println!("{:?}", vm_state),
            Err(_) => println!("something went terribly wrong!"),
        }
    }
}

```

从这个例子看来，polygon的miden-vm 的工作流程还是比较清晰的，编译自定义的程序为miden-vm可以执行的字节码，

构建proving-options，这里使用不是默认方式，这是因为

```rust
fn default() -> Self {
        Self::with_96_bit_security(false)
    }

pub fn with_96_bit_security(recursive: bool) -> Self {
  // 如果启用递归的属性，则使用递归友好的hash fucntion       
  if recursive {
            Self {
                exec_options: ExecutionOptions::default(),
                proof_options: Self::RECURSIVE_96_BITS,
                hash_fn: HashFunction::Rpo256,
            }
        } else {
            Self {
                exec_options: ExecutionOptions::default(),
                proof_options: Self::REGULAR_96_BITS,
                hash_fn: HashFunction::Blake3_192,
            }
        }
}
```

这两个主要的区别就是hash-function了，选择Rpo256是因为它比Blake3_192 更加适用于递归场景下，代价就是证明生成速度会很慢。

### prove

```rust
pub fn prove<H>(
    program: &Program,
    stack_inputs: StackInputs,
    host: H,
    options: ProvingOptions,
) -> Result<(StackOutputs, ExecutionProof), ExecutionError>
where
    H: Host,
{
  ... do something ...
  
   // generate STARK proof
    let proof = match hash_fn {
        HashFunction::Blake3_192 => ExecutionProver::<Blake3_192, WinterRandomCoin<_>>::new(
            options,
            stack_inputs,
            stack_outputs.clone(),
        )
        .prove(trace),
        HashFunction::Blake3_256 => ExecutionProver::<Blake3_256, WinterRandomCoin<_>>::new(
            options,
            stack_inputs,
            stack_outputs.clone(),
        )
        .prove(trace),
        HashFunction::Rpo256 => {
            let prover = ExecutionProver::<Rpo256, RpoRandomCoin>::new(
                options,
                stack_inputs,
                stack_outputs.clone(),
            );
            #[cfg(all(feature = "metal", target_arch = "aarch64", target_os = "macos"))]
            let prover = gpu::MetalRpoExecutionProver(prover);
            prover.prove(trace)
        }
    }
    .map_err(ExecutionError::ProverError)?;
    let proof = ExecutionProof::new(proof, hash_fn);

    Ok((stack_outputs, proof))
}
```

由于咱们的例子是使用的是 Rpo256 这个hash function，因此proof 走的分支就是`HashFunction::Rpo256 => {......}`

这时候就分两种情况了，假定我们在macos 上且是arm64 架构的cpu时，在执行prove 的时候启用了 `metal` 这个特性，则会使用gpu进行证明加速，否则全部都会使用cpu加速。执行的时候会启动 metal 这个特性，然后咱们继续往下看。

```rust
fn prove(&self, trace: Self::Trace) -> Result<StarkProof, ProverError> {
        match self.options().field_extension() {
            FieldExtension::None => self.generate_proof::<Self::BaseField>(trace),
            FieldExtension::Quadratic => {
                if !<QuadExtension<Self::BaseField>>::is_supported() {
                    return Err(ProverError::UnsupportedFieldExtension(2));
                }
                self.generate_proof::<QuadExtension<Self::BaseField>>(trace)
            }
            FieldExtension::Cubic => {
                if !<CubeExtension<Self::BaseField>>::is_supported() {
                    return Err(ProverError::UnsupportedFieldExtension(3));
                }
                self.generate_proof::<CubeExtension<Self::BaseField>>(trace)
            }
        }
    }
```

上述的代码则是依据不同的域大小，分别调用各自的方法来生成proof。 对应到咱们的代码里，FieldExtension 就是Quadratic（对这个困惑的可以查看一下proving-option的构建过程）。也就是说此时代码运行的分支就是`FieldExtension::Quadratic => {`

核心技术&& 关键步骤：

* **将计算轨迹转化为多项式：** 通过在 LDE 域上进行插值，将执行轨迹表示为一个多项式。
* **使用随机挑战：** 通过引入随机的挑战值，使得攻击者很难构造一个假的证明。
* **利用 Merkle 树：** 使用 Merkle 树来对数据进行承诺，并高效地验证数据的正确性。
* **应用 FRI 协议：** 使用 FRI 协议来验证多项式的正确性，从而验证计算的正确性。

```rust
fn generate_proof<E>(&self, mut trace: Self::Trace) -> Result<StarkProof, ProverError>{
  //1. 初始化AIR 和 prover channel
  // AIR 代数中间表示，区别于snark的 r1cs stark 则使用的是代数中间表示的形式来表示证明过程
 	// prover channel 用于模拟证明者和验证者之间的交互；该通道将用于提交值并提取应来自验证者的随机挑战。
  
  let air = Self::Air::new(...);
  let mut channel = ProverChannel::<Self::Air, E, Self::HashFn, Self::RandomCoin>::new(
            &air,
            pub_inputs_elements,
        );
  
  //2. 承诺执行的轨迹
  //构建计算域。为之后的多项式评估做准备
  let domain = info_span!("build_domain", trace_length, lde_domain_size)
            .in_scope(|| StarkDomain::new(&air));
  
  // 承诺主轨迹段，为通过prover channel 发送承诺值
  let (mut trace_lde, mut trace_polys) = {
            let span = info_span!("commit_to_main_trace_segment").entered();
            let (trace_lde, trace_polys) =
                self.new_trace_lde(&trace.get_info(), trace.main_segment(), &domain);

            // get the commitment to the main trace segment LDE
            let main_trace_root = trace_lde.get_main_trace_commitment();

            channel.commit_trace(main_trace_root);
            drop(span);
            (trace_lde, trace_polys)
        };
  
  
  // 如果有其他的轨迹段，则依次承诺每个辅助轨迹段。
  for i in 0..trace.layout().num_aux_segments() {
    let (aux_segment, rand_elements) = {
                let span = info_span!("build_aux_trace_segment", num_columns).entered();
                let rand_elements = channel.get_aux_trace_segment_rand_elements(i);
                let aux_segment = trace
                    .build_aux_segment(&aux_trace_segments, &rand_elements)
                    .expect("failed build auxiliary trace segment");
                drop(span);
                (aux_segment, rand_elements)
            };
    let aux_segment_polys = {
      			let span = info_span!("commit_to_aux_trace_segment").entered();
                let (aux_segment_polys, aux_segment_root) =
                    trace_lde.add_aux_segment(&aux_segment, &domain);
      			channel.commit_trace(aux_segment_root);

            drop(span);
            aux_segment_polys
    }
    trace_polys.add_aux_segment(aux_segment_polys);
    aux_trace_rand_elements.add_segment_elements(rand_elements);
    aux_trace_segments.push(aux_segment);
  }
  
  //3.评估约束
  // 根据约束评估域，评估 AIR 指定的约束，并使用从 ProverChannel 获取的系数计算这些评估的随机线性组合。
  let composition_poly_trace =
            info_span!("evaluate_constraints", ce_domain_size).in_scope(|| {
                self.new_evaluator(
                    &air,
                    aux_trace_rand_elements,
                    channel.get_constraint_composition_coeffs(),
                )
                .evaluate(&trace_lde, &domain)
            });
  
  //4.提交约束承诺 
  let (constraint_commitment, composition_poly) = {
            let span = info_span!("commit_to_constraint_evaluations").entered();

           // 建立对约束组合多项式列的评估的承诺，这一步则会用到gpu加速
            let (constraint_commitment, composition_poly) = self.build_constraint_commitment::<E>(
                composition_poly_trace,
                air.context().num_constraint_composition_columns(),
                &domain,
            );

            // 将约束 Merkle 树的根写入通道来承诺对约束的评估
            channel.commit_constraints(constraint_commitment.root());
            drop(span);
            (constraint_commitment, composition_poly)
        };
  // 5. 构建 DEEP 组合多项式
  // 从扩展域或基域抽取一个域外点 (z)。
  let z = channel.get_ood_point();
  
  //在域外点 z 处评估轨迹多项式和约束多项式，并将结果发送给验证者。
  // 在 OOD 点 z 处评估迹和约束多项式，并将结果发送给验证器。迹多项式实际上是在两个点上进行评估的：z 和 z * g，其中 g 是迹域的生成器。
  let ood_trace_states = trace_polys.get_ood_frame(z);
  channel.send_ood_trace_states(&ood_trace_states);
  
  let ood_evaluations = composition_poly.evaluate_at(z);
  channel.send_ood_constraint_evaluations(&ood_evaluations);
  
  // 将所有多项式组合在一起并合并为 DEEP 组合多项式
  deep_composition_poly.add_trace_polys(trace_polys, ood_trace_states);
  deep_composition_poly.add_composition_poly(composition_poly, ood_evaluations);
  
   event!(Level::DEBUG, "degree: {}", deep_composition_poly.degree());
  
  //6.在 LDE 域上评估 DEEP 组合多项式
  let deep_evaluations = {
            let span = info_span!("evaluate_deep_composition_poly").entered();
            let deep_evaluations = deep_composition_poly.evaluate(&domain);
            // we check the following condition in debug mode only because infer_degree is an
            // expensive operation
            debug_assert_eq!(trace_length - 2, infer_degree(&deep_evaluations, domain.offset()));

            drop(span);
            deep_evaluations
        };
  
  //7. 计算组合多项式的 FRI 层
  // 使用 ProverChannel 和深度评估值构建 FRI 层
  let mut fri_prover = FriProver::new(fri_options);
        info_span!("compute_fri_layers", num_layers)
            .in_scope(|| fri_prover.build_layers(&mut channel, deep_evaluations));
  
  //8. 确定查询位置
  // 应用工作量证明 (PoW) 到查询种子
  // 从 ProverChannel 获取伪随机查询位置。
  let query_positions = {
            let grinding_factor = air.options().grinding_factor();
            let num_positions = air.options().num_queries();
            let span =
                info_span!("determine_query_positions", grinding_factor, num_positions,).entered();

            // apply proof-of-work to the query seed
            channel.grind_query_seed();

            // generate pseudo-random query positions
            let query_positions = channel.get_query_positions();
            event!(Level::DEBUG, "query_positions_len: {}", query_positions.len());

            drop(span);
            query_positions
        };
  
  // 构建证明
  // 使用 FRI 证明者对象构建 FRI 证明。
  //在选定的位置查询执行轨迹和约束承诺，并获取相应的 Merkle 认证路径。
  //使用 ProverChannel 和查询结果构建最终的证明对象。
  let proof = {
            let span = info_span!("build_proof_object").entered();
            // generate FRI proof
            let fri_proof = fri_prover.build_proof(&query_positions);
            let trace_queries = trace_lde.query(&query_positions);
            let constraint_queries = constraint_commitment.query(&query_positions);
            let proof = channel.build_proof(
                trace_queries,
                constraint_queries,
                fri_proof,
                query_positions.len(),
            );

            drop(span);
            proof
        };
  
}
```

* 代码使用了不同的域（基域、扩展域）来进行计算。
* 证明过程中使用了 ProverChannel 模拟证明者和验证者之间的交互。
* FRI 层用于增强证明的可验证性。
* 查询位置的选取至关重要，需要通过 PoW 机制来保证随机性。

> 补充知识点
>
> **LDE 域**，即**低度扩展域**（Low Degree Extension domain），在 STARK 证明系统中扮演着非常重要的角色。它是一个有限域，用于将原始的计算轨迹扩展到一个更大的域上，从而便于进行多项式插值和评估。
>
> LDE 域的作用
>
> - **多项式插值：** 将原始的计算轨迹看作是一组点，通过在 LDE 域上进行多项式插值，可以将这些点表示为一个多项式。这个多项式包含了原始计算的所有信息。
> - **快速傅里叶变换（FFT）：** LDE 域的特殊结构使得可以在其上高效地进行 FFT，从而加速多项式的评估和乘法。
> - **错误扩散：** 如果计算过程中存在错误，那么在 LDE 域上进行多项式插值后，这个错误会扩散到整个多项式，从而更容易被检测到。
>
> 为什么在 LDE 域上评估 DEEP 组合多项式？
>
> - **隐藏信息：** DEEP 组合多项式包含了关于计算轨迹和约束的很多信息。通过在 LDE 域上评估这个多项式，可以将这些信息隐藏起来，使得攻击者很难直接从评估结果中提取出有用的信息。
> - **错误检测：** 如果计算过程中存在错误，那么在 LDE 域上评估 DEEP 组合多项式时，会产生不一致的结果，从而可以检测出错误。
> - **FRI 证明：** FRI 证明是一种高效的交互式证明系统，它通过对多项式进行多次采样和比较来验证多项式的正确性。LDE 域为 FRI 证明提供了良好的基础。

















