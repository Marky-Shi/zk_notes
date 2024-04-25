https://github.com/zkcrypto/bellman.git

https://learnblockchain.cn/article/1662

实现了Groth16算法

总体流程大致可以分为以下几个步骤：
1.将问题多项式拍平（flatten），构建对应的电路（Circuit）
（这一步是由上层应用程序配置的）
2.根据电路生成R1CS（Rank 1 Constraint System）
3.将R1CS转化为QAP（Quadratic Arithmetic Program）。传统做法是通过拉格朗日插值，但是为了降低计算复杂度，可以通过快速傅立叶变换来实现。
4.初始化QAP问题的参数，即CRS（Common Reference Strings）
5.根据CRS和输入创建proof
6.验证proof



验证A(x) * B(x)–C(x) = t(x) * h(x)，就能够一次完成全部RICS 约束的验证；而一旦这个验证通过，就可以相信输入为真。

zk-SNARK 要做的工作就是帮助验证 A(x) * B(x)–C(x) = t(x) * h(x)。

**推理式证明变成了交互式证明：verifier 在一个随机点上提出挑战，prover 给出这个点上的解响应挑战；prover 需要有「知识」才能计算出随机点上的解，但这个解本身不会泄露「知识」**。

为什么能通过验证多项式上的一个点来确定两个多项式A(x) * B(x)–C(x) 与t(x) * h(x)是否相等？

这是由多项式的特性决定的，「一个多项式在任意点的计算结果都可以看做是其唯一身份的表示」。



### SetUp 阶段

Setup阶段最主要的工作是生成CRS数据

- 参数结构  bellman/src/groth16/mod.rs

  ```rust
  #[derive(Clone)]
  pub struct VerifyingKey<E: Engine> {
      // alpha in g1 for verifying and for creating A/C elements of
      // proof. Never the point at infinity.
      pub alpha_g1: E::G1Affine,
  
      // beta in g1 and g2 for verifying and for creating B/C elements
      // of proof. Never the point at infinity.
      pub beta_g1: E::G1Affine,
      pub beta_g2: E::G2Affine,
  
      // gamma in g2 for verifying. Never the point at infinity.
      pub gamma_g2: E::G2Affine,
  
      // delta in g1/g2 for verifying and proving, essentially the magic
      // trapdoor that forces the prover to evaluate the C element of the
      // proof with only components from the CRS. Never the point at
      // infinity.
      pub delta_g1: E::G1Affine,
      pub delta_g2: E::G2Affine,
  
      // Elements of the form (beta * u_i(tau) + alpha v_i(tau) + w_i(tau)) / gamma
      // for all public inputs. Because all public inputs have a dummy constraint,
      // this is the same size as the number of inputs, and never contains points
      // at infinity.
      pub ic: Vec<E::G1Affine>,
  }
  ```

  ```rust
  #[derive(Clone)]
  pub struct Parameters<E: Engine> {
      pub vk: VerifyingKey<E>,
  
      // Elements of the form ((tau^i * t(tau)) / delta) for i between 0 and
      // m-2 inclusive. Never contains points at infinity.
      pub h: Arc<Vec<E::G1Affine>>,
  
      // Elements of the form (beta * u_i(tau) + alpha v_i(tau) + w_i(tau)) / delta
      // for all auxiliary inputs. Variables can never be unconstrained, so this
      // never contains points at infinity.
      pub l: Arc<Vec<E::G1Affine>>,
  
      // QAP "A" polynomials evaluated at tau in the Lagrange basis. Never contains
      // points at infinity: polynomials that evaluate to zero are omitted from
      // the CRS and the prover can deterministically skip their evaluation.
      pub a: Arc<Vec<E::G1Affine>>,
  
      // QAP "B" polynomials evaluated at tau in the Lagrange basis. Needed in
      // G1 and G2 for C/B queries, respectively. Never contains points at
      // infinity for the same reason as the "A" polynomials.
      pub b_g1: Arc<Vec<E::G1Affine>>,
      pub b_g2: Arc<Vec<E::G2Affine>>,
  }
  ```

  最后3个参数a, b_g1, b_g2,后面在计算proof的时候会用到。

- 变量类型

  - Variable 类型代表输入数据中的每一个值，分为公开的statement 数据和私有的witness。

    - input 类型 即 statement, Aux 即 witness类型

      ```rust
      #[derive(Copy, Clone, PartialEq, Debug)]
      pub enum Index {
          Input(usize),
          Aux(usize),
      }
      ```

- ConstrainSystem是一个接口，表示可以具有新变量的约束系统，分配和他们直接形成的约束  bellman/groth16/src/lib.rs

  - ```rust
    // 返回input类型的变量，索引为0
    fn one() -> Variable {
            Variable::new_unchecked(Index::Input(0))
       }
    ```

  - ```rust
     // 在约束系统中分配私有变量(aux)
     fn alloc<F, A, AR>(&mut self, annotation: A, f: F) -> Result<Variable, SynthesisError>
        where
            F: FnOnce() -> Result<Scalar, SynthesisError>,
            A: FnOnce() -> AR,
            AR: Into<String>;
    ```

  - ```rust
    // 在约束系统中分配一个 input (public var),用于确定变量的赋值
    fn alloc_input<F, A, AR>(&mut self, annotation: A, f: F) -> Result<Variable, SynthesisError>
        where
            F: FnOnce() -> Result<Scalar, SynthesisError>,
            A: FnOnce() -> AR,
            AR: Into<String>;
    ```

  - ```rust
    // 强制执行 A*B = C
    fn enforce<A, AR, LA, LB, LC>(&mut self, annotation: A, a: LA, b: LB, c: LC)
        where
            A: FnOnce() -> AR,
            AR: Into<String>,
            LA: FnOnce(LinearCombination<Scalar>) -> LinearCombination<Scalar>,
            LB: FnOnce(LinearCombination<Scalar>) -> LinearCombination<Scalar>,
            LC: FnOnce(LinearCombination<Scalar>) -> LinearCombination<Scalar>;
    ```

  - example

    ```rust
    let a = cs.alloc(...)
    let b = cs.alloc(...)
    let c = cs.alloc_input(...)
    cs.enforce(
        || "a*b=c",
        |lc| lc + a,
        |lc| lc + b,
        |lc| lc + c
    );
    ```

    c是statement，a和b是witness，需要验证a * b = c这个Circu

    若想验证 a+ b =c  

    ```rust
    cs.enforce(
        || "a*b=c",
        |lc| lc + a + b,
        |lc| lc + CS::one(),
        |lc| lc + c
    );
    ```

- **构建R1CS**

  - Circuit的synthesize()会调用ConstraintSystem的enforce()构建R1CS。(可查看位于 bellam/src/groth16/mod.rs 中的 test case )
    **KeypairAssembly**是**ConstraintSystem**的一个实现，R1CS的参数会保存在其成员变量中：（bellman/src/groth16/generator.rs）

  - ```rust
    struct KeypairAssembly<Scalar: PrimeField> {
        num_inputs: usize,
        num_aux: usize,
        num_constraints: usize,
        at_inputs: Vec<Vec<(Scalar, usize)>>,
        bt_inputs: Vec<Vec<(Scalar, usize)>>,
        ct_inputs: Vec<Vec<(Scalar, usize)>>,
        at_aux: Vec<Vec<(Scalar, usize)>>,
        bt_aux: Vec<Vec<(Scalar, usize)>>,
        ct_aux: Vec<Vec<(Scalar, usize)>>,
    }
    ```

- 构建QAP

  - R1CS 转换QAP，其中一步会利用逆离散快速傅立叶变换实现拉格朗日插值 //    bellman/src/groth16/generator.rs

    ```rust
    // Use inverse FFT to convert powers of tau to Lagrange coefficients
    powers_of_tau.ifft(&worker);
    let powers_of_tau = powers_of_tau.into_coeffs();
    ```

- 准备验证参数

  ```rust
  pub fn prepare_verifying_key<E: MultiMillerLoop>(vk: &VerifyingKey<E>) -> PreparedVerifyingKey<E> {
      let gamma = vk.gamma_g2.neg();
      let delta = vk.delta_g2.neg();
  
      PreparedVerifyingKey {
          alpha_g1_beta_g2: E::pairing(&vk.alpha_g1, &vk.beta_g2),
          neg_gamma_g2: gamma.into(),
          neg_delta_g2: delta.into(),
          ic: vk.ic.clone(),
      }
  }
  ```

- **创建proof**  bellman/src/groth16/mod.rs

  ```rust
  #[derive(Clone, Debug)]
  pub struct Proof<E: Engine> {
      pub a: E::G1Affine,
      pub b: E::G2Affine,
      pub c: E::G1Affine,
  }
  ```

  bellman/src/groth16/prover.rs 

  ```rust
  pub fn create_proof<E, C, P: ParameterSource<E>>(
      circuit: C,
      mut params: P,
      r: E::Fr,
      s: E::Fr,
  ) -> Result<Proof<E>, SynthesisError>
  where
      E: Engine,
      E::Fr: PrimeFieldBits,
      C: Circuit<E::Fr>,
  {
  ```

  `circuit.synthesize(&mut prover)?;` 生成proof的时候使用的是ConstraintSystem的另外一个实现ProvingAssignment来调用Circuit的synthesize()函数。 和setup阶段基本类似

  - alloc_input  : public input 放在input_assignment

  - alloc : witness 存放在 aux_assignment

  - enforce()/eval()

  - 获取VK    params.get_vk(prover.input_assignment.len())?;

  - 计算h(x) 

    ```rust
    let h = {
    		// 获取之前的到的结果
            let mut a = EvaluationDomain::from_coeffs(prover.a)?;
            let mut b = EvaluationDomain::from_coeffs(prover.b)?;
            let mut c = EvaluationDomain::from_coeffs(prover.c)?;
            
        	// 通过3次 iFFT 获取多项式系数，计算在coset上的值，通过3次iFFT 获取偏移后的点
        	a.ifft(&worker);
            a.coset_fft(&worker);
            b.ifft(&worker);
            b.coset_fft(&worker);
            c.ifft(&worker);
            c.coset_fft(&worker);
    		
        	// 进行计算
            a.mul_assign(&worker, &b);
            drop(b);
            a.sub_assign(&worker, &c);
            drop(c);
            a.divide_by_z_on_coset(&worker);
            
        	// 获取多项式系数
        	a.icoset_fft(&worker);
            let mut a = a.into_coeffs();
            let a_len = a.len() - 1;
            a.truncate(a_len);
            // TODO: parallelize if it's even helpful
            let a = Arc::new(a.into_iter().map(|s| s.0.into()).collect::<Vec<_>>());
    		
        	// 进行多次幂运算
            multiexp(&worker, params.get_h(a.len())?, FullDensity, a)
        };
    ```
- **验证阶段**  bellman/src/groth16/verifier.rs

  - ```rsut
    let mut acc = pvk.ic[0].to_curve();
    
        for (i, b) in public_inputs.iter().zip(pvk.ic.iter().skip(1)) {
            AddAssign::<&E::G1>::add_assign(&mut acc, &(*b * i));
        }
    ```

  - ```rust
    if pvk.alpha_g1_beta_g2
            == E::multi_miller_loop(&[
                (&proof.a, &proof.b.into()),
                (&acc.to_affine(), &pvk.neg_gamma_g2),
                (&proof.c, &pvk.neg_delta_g2),
            ])
            .final_exponentiation()
        {
            Ok(())
        } else {
            Err(VerificationError::InvalidProof)
        }
    ```
  
    
  
  
  
  
  
  
  
  
  
  

