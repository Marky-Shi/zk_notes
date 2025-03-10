# Bellman 库分析

[Bellman](https://github.com/zkcrypto/bellman.git) 是一个用 Rust 实现的 zk-SNARK 库，主要实现了 Groth16 算法。本文档分析其内部实现原理和关键代码流程。

## 1. 总体流程

Bellman 库实现 Groth16 算法的流程可以分为以下几个步骤：

1. **电路构建**：将问题多项式拍平（flatten），构建对应的电路（Circuit）
   （这一步是由上层应用程序配置的）
2. **R1CS 生成**：根据电路生成 R1CS（Rank 1 Constraint System）
3. **QAP 转换**：将 R1CS 转化为 QAP（Quadratic Arithmetic Program）
   传统做法是通过拉格朗日插值，但为了降低计算复杂度，Bellman 通过快速傅里叶变换来实现
4. **参数初始化**：初始化 QAP 问题的参数，即 CRS（Common Reference Strings）
5. **证明生成**：根据 CRS 和输入创建 proof
6. **证明验证**：验证 proof 的有效性

## 2. 核心原理

Groth16 的核心是验证等式 `A(x) * B(x) - C(x) = t(x) * h(x)`，这一验证能够一次完成全部 R1CS 约束的验证。一旦这个验证通过，就可以相信输入为真。

zk-SNARK 要做的工作就是帮助验证 `A(x) * B(x) - C(x) = t(x) * h(x)`。

**推理式证明变成了交互式证明**：verifier 在一个随机点上提出挑战，prover 给出这个点上的解响应挑战；prover 需要有「知识」才能计算出随机点上的解，但这个解本身不会泄露「知识」。

### 2.1 多项式验证原理

为什么能通过验证多项式上的一个点来确定两个多项式 `A(x) * B(x) - C(x)` 与 `t(x) * h(x)` 是否相等？

这是由多项式的特性决定的，「一个多项式在任意点的计算结果都可以看做是其唯一身份的表示」。更准确地说，这基于以下原理：

1. **Schwartz-Zippel 引理**：如果两个不同的多项式在一个随机选择的点上取值相同的概率非常小
2. **知识提取**：如果 prover 能够在随机点上给出正确的多项式值，那么它很可能知道整个多项式
3. **零知识性**：通过适当的随机化，可以确保 prover 的响应不会泄露关于原始 witness 的信息

## 3. 代码结构分析

### 3.1 Setup 阶段

Setup 阶段最主要的工作是生成 CRS 数据。

#### 3.1.1 参数结构

Bellman 中定义了两个主要的参数结构：`VerifyingKey` 和 `Parameters`。

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
### 3.6 验证阶段

验证阶段的核心是检查配对等式是否成立。Bellman 中的验证代码位于 `bellman/src/groth16/verifier.rs`：

```rust
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
这个验证等式对应了 Groth16 论文中的验证公式：

$$e(A, B) \cdot e(\text{acc}, -\gamma) \cdot e(C, -\delta) = e(\alpha, \beta)$$

其中：

- $A$, $B$, $C$ 是证明中的三个元素
- $\text{acc}$ 是公开输入的线性组合
- $\alpha$, $\beta$, $\gamma$, $\delta$ 是 setup 阶段生成的参数

## 4. 实际应用示例

### 4.1 简单电路示例

以下是一个简单的电路示例，验证 $a \cdot b = c$：

```rust
use bellman::{
    Circuit, ConstraintSystem, SynthesisError,
    groth16::{
        create_random_proof, generate_random_parameters,
        prepare_verifying_key, verify_proof,
    },
};
use ff::Field;
use pairing::bls12_381::{Bls12, Fr};
use rand::rngs::OsRng;

// 定义电路结构
struct MultiplyDemo<Fr: Field> {
    a: Option<Fr>,
    b: Option<Fr>,
    c: Option<Fr>,
}

// 实现 Circuit trait
impl<Fr: Field> Circuit<Fr> for MultiplyDemo<Fr> {
    fn synthesize<CS: ConstraintSystem<Fr>>(
        self,
        cs: &mut CS,
    ) -> Result<(), SynthesisError> {
        // 分配私有输入变量
        let a = cs.alloc(|| "a", || self.a.ok_or(SynthesisError::AssignmentMissing))?;
        let b = cs.alloc(|| "b", || self.b.ok_or(SynthesisError::AssignmentMissing))?;
        
        // 分配公开输出变量
        let c = cs.alloc_input(
            || "c",
            || self.c.ok_or(SynthesisError::AssignmentMissing),
        )?;
        
        // 添加约束: a * b = c
        cs.enforce(
            || "a * b = c",
            |lc| lc + a,
            |lc| lc + b,
            |lc| lc + c,
        );
        
        Ok(())
    }
}

// 使用示例
fn main() {
    // 创建随机数生成器
    let mut rng = OsRng;
    
    // 生成参数
    let params = {
        let circuit = MultiplyDemo::<Fr> {
            a: None,
            b: None,
            c: None,
        };
        
        generate_random_parameters::<Bls12, _, _>(circuit, &mut rng).unwrap()
    };
    
    // 准备验证密钥
    let pvk = prepare_verifying_key(&params.vk);
    
    // 创建实际电路实例
    let a = Fr::from(3);
    let b = Fr::from(5);
    let c = a * b; // c = 15
    
    let circuit = MultiplyDemo {
        a: Some(a),
        b: Some(b),
        c: Some(c),
    };
    
    // 生成证明
    let proof = create_random_proof(circuit, &params, &mut rng).unwrap();
    
    // 验证证明
    let result = verify_proof(&pvk, &proof, &[c]).unwrap();
    
    println!("Verification result: {}", result);
}
```

## 5. Bellman 内部实现细节
### 5.1 多项式表示与操作
Bellman 使用 EvaluationDomain 结构来表示和操作多项式：

```rust
pub struct EvaluationDomain<Fr: PrimeField> {
    coeffs: Vec<(Fr, Fr)>,
    exp: u32,
    omega: Fr,
    omega_inv: Fr,
    geninv: Fr,
    minv: Fr,
}
 ```

这个结构支持以下操作：

1. FFT 变换 ：将多项式系数转换为点值表示
2. 逆 FFT 变换 ：将点值表示转换回系数表示
3. 多项式乘法 ：在点值表示下高效执行
4. 多项式除法 ：特别是除以消失多项式 $Z(X)$

### 5.2 多指数运算优化
多指数运算是 Groth16 中的计算瓶颈之一。Bellman 使用以下技术优化多指数运算：
```rust
pub fn multiexp<G: CurveAffine>(
    pool: &Worker,
    bases: &[G],
    density: Density,
    exponents: Arc<Vec<<G::Scalar as PrimeField>::Repr>>,
) -> G::Projective {
    // 分块并行计算
    let mut acc = G::Projective::zero();
    let chunk_size = if exponents.len() < 32 {
        1
    } else {
        exponents.len() / pool.cpus
    };
    
    pool.scope(exponents.len(), |scope, chunk| {
        for (bases, exponents) in bases
            .chunks(chunk_size)
            .zip(exponents.chunks(chunk_size))
        {
            scope.spawn(move |_| {
                // 计算当前块的多指数运算
                let mut acc = G::Projective::zero();
                for (base, exp) in bases.iter().zip(exponents.iter()) {
                    let mut tmp = *base;
                    // 执行标量乘法
                    tmp = tmp.mul(*exp);
                    acc.add_assign(&tmp);
                }
                acc
            });
        }
    });
    
    acc
}
```

### 5.3 FFT 实现
Bellman 使用快速傅里叶变换（FFT）来加速多项式操作，特别是在 QAP 转换过程中：
```rust
// 快速傅里叶变换
pub fn fft(a: &mut [Fr], omega: &Fr, log_n: u32) {
    let n = 1 << log_n;
    
    // 比特反转排序
    for k in 0..n {
        let rk = bitreverse(k, log_n);
        if k < rk {
            a.swap(k, rk);
        }
    }
    
    // 蝶形运算
    let mut m = 1;
    for _ in 0..log_n {
        let w_m = omega.pow(&[(n / (2 * m)) as u64]);
        
        for k in (0..n).step_by(2 * m) {
            let mut w = Fr::one();
            for j in 0..m {
                let t = w * a[k + j + m];
                a[k + j + m] = a[k + j] - t;
                a[k + j] = a[k + j] + t;
                w = w * w_m;
            }
        }
        
        m *= 2;
    }
}
```

## 6. 性能考虑
### 6.1 计算复杂度
Groth16 算法的主要计算瓶颈在于：

1. Setup 阶段 ：$O(n \log n)$ 复杂度，其中 $n$ 是约束数量
2. Prove 阶段 ：$O(n \log n)$ 复杂度
3. Verify 阶段 ：常数时间复杂度，与电路大小无关
### 6.2 内存使用
对于大型电路，内存使用是一个重要考虑因素：

1. 参数大小 ：与约束数量成线性关系
2. 证明大小 ：恒定大小（3 个群元素）
3. 验证密钥大小 ：与公开输入数量成线性关系
### 6.3 优化策略
Bellman 采用以下策略优化性能：

1. 并行计算 ：利用多核 CPU 加速计算
2. 批处理 ：对多个证明进行批量验证
3. 预计算 ：对固定基点的多指数运算进行预计算
4. 内存管理 ：使用 Arc 智能指针共享大型数据结构
