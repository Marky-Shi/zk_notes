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

#### GPU加速 for mac

让我们看看启用gpu加速之后会进行那些方面的加速吧

```rust
fn build_constraint_commitment<E: FieldElement<BaseField = Felt>>(
        &self,
        composition_poly_trace: CompositionPolyTrace<E>,
        num_trace_poly_columns: usize,
        domain: &StarkDomain<Felt>,
    ) -> (ConstraintCommitment<E, Self::HashFn>, CompositionPoly<E>) {
      // 初始化并评估组合多项式列
      let composition_poly =
            CompositionPoly::new(composition_poly_trace, domain, num_trace_poly_columns);
      let blowup = domain.trace_to_lde_blowup();
      // 计算LDE域的膨胀因子和评估偏移量。
      let offsets =
            get_evaluation_offsets::<E>(composition_poly.column_len(), blowup, domain.offset());
      // 为之后的GPU处理做准备
      let segments = Self::build_aligned_segements(
            composition_poly.data(),
            domain.trace_twiddles(),
            &offsets,
        );
      
      // 构建约束评估承诺
      // 对求值结果的每一行进行哈希计算，得到一系列哈希值。
      // 将这些哈希值构建成一棵 Merkle 树。树根就是最终的承诺（这部分实在gpu上完成的）
      // 处理每个segment
      for (segment_idx, segment) in segments.iter().enumerate() {
            // 检查segment 是否需要填充
            if rpo_padded_segment_idx.map_or(false, |pad_idx| pad_idx == segment_idx) {
                // 复制并修改最后一个段，使用 Rpo256 的填充规则（“1”后跟“0”）。此处填充为1
                rpo_padded_segment = unsafe { page_aligned_uninit_vector(lde_domain_size) };
                rpo_padded_segment.copy_from_slice(segment);
                
                if self.metal_hash_fn == HashFn::Rpo256 {
                    let rpo_pad_column = num_base_columns % RATE;
                    rpo_padded_segment.iter_mut().for_each(|row| row[rpo_pad_column] = ONE);
                }
                row_hasher.update(&rpo_padded_segment);
                assert_eq!(segments.len() - 1, segment_idx, "padded segment should be the last");
                break;
            }
            row_hasher.update(segment);
        }
      
       // 使用GPU 生成merkle tree
        let composed_evaluations = RowMatrix::<E>::from_segments(segments, num_base_columns);
        let nodes = block_on(tree_nodes).into_iter().map(|dig| H::Digest::from(&dig)).collect();
        let leaves = row_hashes.into_iter().map(|dig| H::Digest::from(&dig)).collect();
        let commitment = MerkleTree::<H>::from_raw_parts(nodes, leaves).unwrap();
        let constraint_commitment = ConstraintCommitment::new(composed_evaluations, commitment);
      (constraint_commitment, composition_poly)
      
}
```

这个方法主要用于构建约束评估的承诺。简单来说，它将一个复杂的数学问题（约束系统）转化为一个更小的、可验证的数字指纹。这个指纹可以用来验证原始数据的完整性，而无需重新计算整个复杂的系统。

步骤：

**评估约束多项式:**

- 将表示约束系统的多项式在特定的域上进行求值。
- 将求值结果按照特定的方式组织成矩阵或向量。

**构建承诺:**

- 对求值结果的每一行进行哈希计算，得到一系列哈希值。
- 将这些哈希值构建成一棵 Merkle 树。
- Merkle 树的根节点就是最终的承诺。

而segment 它表示约束多项式在 LDE 域上求值的结果的一个片段。

segment 的处理过程

1. **分段求值:** 为了提高计算效率，通常将整个约束多项式分成多个片段，分别进行求值。每个片段对应一个 `segment`。
2. **对齐:** 为了满足 GPU 处理的要求，这些 `segment` 需要进行对齐处理。这可能涉及填充、重新排列数据等操作。
3. **哈希计算:** 每个 `segment` 都会被哈希成一个固定长度的值。这些哈希值随后会用来构建 Merkle 树。
4. **构建 Merkle 树:** 将所有 `segment` 的哈希值作为叶子节点，构建一棵 Merkle 树。Merkle 树的根节点就是最终的承诺。

分段处理的好处：

* **提高并行性:** 将数据分段可以更好地利用 GPU 的并行计算能力，提高计算效率。
* **降低内存占用:** 分段处理可以减少一次性加载到内存的数据量，从而降低内存占用。
* **优化缓存利用率:** 分段处理可以更好地利用 CPU 和 GPU 的缓存，提高数据访问速度。



### Verify

该函数的主要目的是验证一个给定的程序执行是否正确，并返回其安全级别。

```rust
pub fn verify(
    program_info: ProgramInfo,
    stack_inputs: StackInputs,
    stack_outputs: StackOutputs,
    proof: ExecutionProof,
) -> Result<u32, VerificationError> {
  
  // 从proof中获取安全级别
  let security_level = proof.security_level();
  //构建公共输入
  let pub_inputs = PublicInputs::new(program_info, stack_inputs, stack_outputs);
  let (hash_fn, proof) = proof.into_parts();
  
  // 根据不用的hash——fn 进行对应的验证
  match hash_fn {
    HashFunction::Blake3_192 => {
      verify_proof::<ProcessorAir, Blake3_192, WinterRandomCoin<_>>(proof, pub_inputs, &opts)
    }
    HashFunction::Blake3_256 => {
      verify_proof::<ProcessorAir, Blake3_256, WinterRandomCoin<_>>(proof, pub_inputs, &opts)
    }
    // 由于咱们之前使用的his Rpo256 hash-func 因此解析也是从这个匹配臂开始
    HashFunction::Rpo256 => {
      let opts = AcceptableOptions::OptionSet(vec![
                ProvingOptions::RECURSIVE_96_BITS,
                ProvingOptions::RECURSIVE_128_BITS,
            ]);
      verify_proof::<ProcessorAir, Rpo256, RpoRandomCoin>(proof, pub_inputs, &opts)
    }
    
    HashFunction::Rpx256 => {
      verify_proof::<ProcessorAir, Rpx256, RpxRandomCoin>(proof, pub_inputs, &opts)
    }
    
  }
  
  Ok(security_level)
}
```

```rust
pub fn verify<AIR, HashFn, RandCoin>(
    proof: Proof,
    pub_inputs: AIR::PublicInputs,
    acceptable_options: &AcceptableOptions,
) -> Result<(), VerifierError>
where
    AIR: Air,
    HashFn: ElementHasher<BaseField = AIR::BaseField>,
    RandCoin: RandomCoin<BaseField = AIR::BaseField, Hasher = HashFn>,
{
  // 验证证明的参数
  acceptable_options.validate::<HashFn>(&proof)?;
  
  // 构建公共随机数，初始种子是证明上下文和公共输入的哈希值，但随着协议的进展，币将使用从证明者收到的信息重新计算
  let mut public_coin_seed = proof.context.to_elements();
  public_coin_seed.append(&mut pub_inputs.to_elements());
  
  // 构建AIR实例，用于指定证明的计算
  let air = AIR::new(proof.trace_info().clone(), pub_inputs, proof.options().clone());

 	//根据不同的域扩展字段选择不同的验证方式
  match air.options().field_extension() {
     FieldExtension::None => {
       perform_verification::<AIR, AIR::BaseField, HashFn, RandCoin>(air, channel, public_coin)
    },
    // ^2
    FieldExtension::Quadratic => {
            if !<QuadExtension<AIR::BaseField>>::is_supported() {
                return Err(VerifierError::UnsupportedFieldExtension(2));
            }
            let public_coin = RandCoin::new(&public_coin_seed);
            let channel = VerifierChannel::new(&air, proof)?;
            perform_verification::<AIR, QuadExtension<AIR::BaseField>, HashFn, RandCoin>(
                air,
                channel,
                public_coin,
            )
        },
    
    
    // ^3
    FieldExtension::Cubic => {
            if !<CubeExtension<AIR::BaseField>>::is_supported() {
                return Err(VerifierError::UnsupportedFieldExtension(3));
            }
            let public_coin = RandCoin::new(&public_coin_seed);
            let channel = VerifierChannel::new(&air, proof)?;
            perform_verification::<AIR, CubeExtension<AIR::BaseField>, HashFn, RandCoin>(
                air,
                channel,
                public_coin,
            )
        },
  }
}
```

由于咱们在例子中构建的时候选择是RPO256，因此这时候的匹配臂就是` FieldExtension::Quadratic => {`



验证由 `VerifierChannel` 传递过来的证明数据是否正确。

```rust
fn perform_verification<A, E, H, R>(
    air: A,
    mut channel: VerifierChannel<E, H>,
    mut public_coin: R,
) -> Result<(), VerifierError>
where
    E: FieldElement<BaseField = A::BaseField>,
    A: Air,
    H: ElementHasher<BaseField = A::BaseField>,
    R: RandomCoin<BaseField = A::BaseField, Hasher = H>,
{
  //1.证明者发送的对 LDE 域上的迹多项式求值的承诺。这些承诺用于更新公共硬币，并从public_cion中获取随机元素集。 每个先前的承诺都用于抽取构建下一个迹段所需的随机元素。最后一个迹承诺用于抽取一组随机系数，证明者使用这些系数来计算约束组合多项式。
  const MAIN_TRACE_IDX: usize = 0;
  const AUX_TRACE_IDX: usize = 1;
  let trace_commitments = channel.read_trace_commitments();
  public_coin.reseed(trace_commitments[MAIN_TRACE_IDX]);
  
  // 验证GRK 承诺包括验证GKR证明并生成所需的随机元素
  // 处理辅助跟踪段（如果有），为每个段构建一组随机元素
  let aux_trace_rand_elements = if air.trace_info().is_multi_segment() {
    
    if air.context().has_lagrange_kernel_aux_column() {
      let gkr_proof = {
                  let gkr_proof_serialized = channel
                      .read_gkr_proof()
      };
      let lagrange_rand_elements = air
                  .get_auxiliary_proof_verifier::<E>()
                  .verify::<E, _>(gkr_proof, &mut public_coin)

      let rand_elements = air.get_aux_rand_elements(&mut public_coin);
      public_coin.reseed(trace_commitments[AUX_TRACE_IDX]);
    }else{
      let rand_elements = air.get_aux_rand_elements(&mut public_coin)
      public_coin.reseed(trace_commitments[AUX_TRACE_IDX]);
    }  
  }
  // 为组合多项式构建系数
  let constraint_coeffs = air
        .get_constraint_composition_coefficients(&mut public_coin)
        .map_err(|_| VerifierError::RandomCoinError)?;
  
  // 2. 约束承诺
  // 证明者发送的对 LDE 域上的约束组合多项式的求值的承诺，使其来更新public-coin，并从public-coin中获取一个域外点Z，在协议的交互式版本中，验证者将此点 z 发送给证明者，证明者在 z 处求迹和约束组合多项式，并将结果发送回验证者。
  channel.read_constraint_commitment();
  public_coin.reseed(constraint_commitment);
  let z = public_coin.draw::<E>().map_err(|_| VerifierError::RandomCoinError)?;
  
  //3 一致性检查 检查在域外点评估的约束是否一致。
  let ood_trace_frame = channel.read_ood_trace_frame();
  let ood_constraint_evaluation_1 = evaluate_constraints(
        &air,
        constraint_coeffs,
        &ood_main_trace_frame,
        &ood_aux_trace_frame,
        ood_lagrange_kernel_frame,
        aux_trace_rand_elements.as_ref(),
        z,
    );
    public_coin.reseed(ood_trace_frame.hash::<H>());
  
  // H(X) = \sum_{i=0}^{m-1} X^{i * l} H_i(X).
  let ood_constraint_evaluations = channel.read_ood_constraint_evaluations();
    let ood_constraint_evaluation_2 =
        ood_constraint_evaluations
            .iter()
            .enumerate()
            .fold(E::ZERO, |result, (i, &value)| {
                result + z.exp_vartime(((i * (air.trace_length())) as u32).into()) * value
            });
    public_coin.reseed(H::hash_elements(&ood_constraint_evaluations));
  
  assert_eq!(ood_constraint_evaluation_1,ood_constraint_evaluation_2);
  
  //4 创建 FRI 证明验证器，用于后续的 FRI 验证。
  let fri_verifier = FriVerifier::new(
        &mut channel,
        &mut public_coin,
        air.options().to_fri_options(),
        air.trace_poly_degree(),
    )
    .map_err(VerifierError::FriVerificationFailed)?;
  
  // 5 生成查询位置: 生成随机的查询位置
  // 获取prover 发送的pow 随机数
  channel.read_pow_nonce();
  
  // 从public_coin中获取 LDE 域的伪随机查询位置
  public_coin
        .draw_integers(air.options().num_queries(), air.lde_domain_size(), pow_nonce)
        .map_err(|_| VerifierError::RandomCoinError)?;
  
  // 获取查询位置处的迹和约束组合多项式的评估值
  channel.read_queried_trace_states(&query_positions)?;
  channel.read_constraint_evaluations(&query_positions)?;
  
  //6 根据查询位置和之前计算的数据，计算 DEEP 组合多项式的评估值。
  let composer = DeepComposer::new(&air, &query_positions, z, deep_coefficients);
    let t_composition = composer.compose_trace_columns(
        queried_main_trace_states,
        queried_aux_trace_states,
        ood_main_trace_frame,
        ood_aux_trace_frame,
        ood_lagrange_kernel_frame,
    );
    let c_composition = composer
        .compose_constraint_evaluations(queried_constraint_evaluations, ood_constraint_evaluations);
    let deep_evaluations = composer.combine_compositions(t_composition, c_composition);
  
  //7 验证 FRI 证明，验证deep多项式的低度性
  // FriVerifier.verify：这个方法是 FRI 协议的验证器，通过迭代地降低多项式的度数，并检查每一层的正确性，最终验证多项式的低度性。
  fri_verifier
        .verify(&mut channel, &deep_evaluations, &query_positions)
        .map_err(VerifierError::FriVerificationFailed)
}
```

验证FRI证明中的每一层和余项多项式是否符合预期，通过递归检查每层的评估值、插值多项式以及线性组合。

```rust
 fn verify_generic<const N: usize>(
        &self,
        channel: &mut C,
        evaluations: &[E],
        positions: &[usize],
    ) -> Result<(), VerifierError> {
      // 参数准备
      ... do something ...
      
      // 迭代 FRI 层: 对于每一层：
      
      //计算折叠后的查询位置。
      let mut folded_positions =
                fold_positions(&positions, domain_size, self.options.folding_factor());
      
      //从承诺中读取对应位置的评估值。
      let layer_commitment = self.layer_commitments[depth];
      let layer_values = channel.read_layer_queries(&position_indexes, &layer_commitment)?;
      let query_values =
                get_query_values::<E, N>(&layer_values, &positions, &folded_positions, domain_size);
      
      //验证评估值的一致性。
      if evaluations != query_values {
                return Err(VerifierError::InvalidLayerFolding(depth));
            }
      
      // 为每个行多项式构建一组x坐标
      let xs = folded_positions.iter().map(|&i| {
                let xe = domain_generator.exp_vartime((i as u64).into()) * self.options.domain_offset();
                folding_roots.iter()
                    .map(|&r| E::from(xe * r))
                    .collect::<Vec<_>>().try_into().unwrap()
            })
            .collect::<Vec<_>>();
      
      
      // 插值x和y值到行多项式
      polynom::interpolate_batch(&xs, &layer_values);
      
      //计算用于层折叠中的线性组合的伪随机值
      let alpha = self.layer_alphas[depth];
      
      
      // 检查在alpha处评估多项式时，结果是否等于相应的列值
      evaluations = row_polys.iter().map(|p| polynom::eval(p, alpha)).collect();
      
      
      //确保下一次度数减少不会导致度数截断
      max_degree_plus_1 % N != 0 
      
    
      domain_generator = domain_generator.exp_vartime((N as u32).into());
            max_degree_plus_1 /= N;
            domain_size /= N;
            mem::swap(&mut positions, &mut folded_positions);
    
     
     // 验证最终层的余项多项式是否符合度数要求。
     // 从channel读取余项多项式，并确保其与上一层的评估一致。
     channel.read_remainder()?;
     if remainder_poly.len() > max_degree_plus_1 {
            return Err(VerifierError::RemainderDegreeMismatch(max_degree_plus_1 - 1));
       } 
     
      for (&position, evaluation) in positions.iter().zip(evaluations) {
            let comp_eval = eval_horner::<E>(
                &remainder_poly,
                offset * domain_generator.exp_vartime((position as u64).into()),
            );
            if comp_eval != evaluation {
                return Err(VerifierError::InvalidRemainderFolding);
            }
        }  
}
```

细节

- **折叠操作:** 使用 `fold_positions` 函数将原始查询位置映射到折叠后的域上的位置。
- **Merkle 树查询:** 使用 `channel.read_layer_queries` 从承诺中读取对应位置的评估值。
- **多项式插值:** 使用 `polynom::interpolate_batch` 将评估点插值成多项式。
- **度数检查:** 验证每一层的度数是否符合要求。
- **余项验证:** 检查最终层的余项多项式是否满足度数限制。

关键概念

- **FRI 协议:** 一种用于验证多项式低度性的交互式证明协议。
- **折叠操作:** 将多项式在更小的域上进行评估，以降低度数。
- **查询点:** 验证者随机选择的点，用于验证多项式的正确性。
- **Merkle 树:** 用于存储多项式评估值的承诺。

































