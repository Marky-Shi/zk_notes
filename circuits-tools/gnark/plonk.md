### generate witness full/public

所有的曲线都是基于Bn 254的

在gnark中，witness的生成需要用到的条件：

* 椭圆曲线的选择
* 实例化自定的电路
* 配置选项（是否只是公共的witness prover/verifier 都可以查看的）

```go
var w types.Plonk_Circuit
		w.X = 2// public	
		w.E = 2 // private
		w.Y = 4 //public 

		witnessFull, err := frontend.NewWitness(&w, ecc.BN254.ScalarField())

    witnessPublic, err := frontend.NewWitness(&w, ecc.BN254.ScalarField(), frontend.PublicOnly())
```

```go
type witness struct {
	vector             any
	nbPublic, nbSecret uint32
}
```

```go
func NewWitness(assignment Circuit, field *big.Int, opts ...WitnessOption) (witness.Witness, error) {
	opt, _ := options(opts...)

	// 迭代电路，计算叶子结点的信息（类型，数量，可以理解为public，secret的数量）
	s, err := schema.Walk(assignment, tVariable, nil)
	// 如果是要生成public witness，那么这部分是无需向witness共享的，因此设置为默认值。
	if opt.publicOnly {
		s.Secret = 0
	}

	// 根据提供的椭圆曲线 ScalarField  初始化witness
	w, err := witness.New(field)

	// write the public | secret values in a chan
	chValues := make(chan any)
	go func() {
		defer close(chValues)
		schema.Walk(assignment, tVariable, func(leaf schema.LeafInfo, tValue reflect.Value) error {
			if leaf.Visibility == schema.Public {
				chValues <- tValue.Interface()
			}
			return nil
		})
		if !opt.publicOnly {
			schema.Walk(assignment, tVariable, func(leaf schema.LeafInfo, tValue reflect.Value) error {
				if leaf.Visibility == schema.Secret {
					chValues <- tValue.Interface()
				}
				return nil
			})
		}
	}()
  // 填充witness
	if err := w.Fill(s.Public, s.Secret, chValues); err != nil {
		return nil, err
	}

	return w, nil
}
```

### generate pk，vk

```go
// 和kzg多项式承诺密切相关
type VerifyingKey struct {
	// Size circuit
	Size              uint64
	SizeInv           fr.Element
	Generator         fr.Element
	NbPublicVariables uint64

	// Commitment scheme that is used for an instantiation of PLONK
	Kzg kzg.VerifyingKey

	// cosetShift generator of the coset on the small domain
	CosetShift fr.Element

	// S commitments to S1, S2, S3
	S [3]kzg.Digest

	// Commitments to ql, qr, qm, qo, qcp prepended with as many zeroes (ones for l) as there are public inputs.
	// In particular Qk is not complete.
	Ql, Qr, Qm, Qo, Qk kzg.Digest
	Qcp                []kzg.Digest

	CommitmentConstraintIndexes []uint64
}


// ProvingKey stores the data needed to generate a proof
type ProvingKey struct {
	Kzg, KzgLagrange kzg.ProvingKey

	// Verifying Key is embedded into the proving key (needed by Prove)
	Vk *VerifyingKey
}

// ProvingKey used to create or open commitments
type ProvingKey struct {
	G1 []bn254.G1Affine // [G₁ [α]G₁ , [α²]G₁, ... ]
}
```



```go
func Setup(spr *cs.SparseR1CS, srs, srsLagrange kzg.SRS) (*ProvingKey, *VerifyingKey, error) {
  // 设置FFT 的域
  domain := initFFTDomain(spr)
  // 确保域中的元素是>2,zk-snark是基于多项式承诺的，如果元素为1，那么zkp就没有意义，且很容易被攻击。
  
  // 检查可信参数 (SRS) 的大小，以确保其足够大以支持给定的电路。如果 SRS 太小，则会返回错误信息。
  if len(srs.Pk.G1) < (int(domain.Cardinality) + 3){
    return error
  }
  
  // 确保 Kate-Shanahan zk-SNARK 系统的标准和 Lagrange 形式可信参数 (SRS) 都具有足够的大小来支持给定的电路并实现正确的验证
  if len(srsLagrange.Pk.G1) != int(domain.Cardinality) {
    return error 
  }
  
  // 设置VK
  vk.CosetShift.Set(&domain.FrMultiplicativeGen)
	vk.Size = domain.Cardinality
	vk.SizeInv.SetUint64(vk.Size).Inverse(&vk.SizeInv)
	vk.Generator.Set(&domain.Generator)
	vk.NbPublicVariables = uint64(len(spr.Public))

	pk.Kzg.G1 = srs.Pk.G1[:int(vk.Size)+3]
	pk.KzgLagrange.G1 = srsLagrange.Pk.G1
	vk.Kzg = srs.Vk
	
  // ql, qr, qm, qo, qk, qcp in Lagrange Basis
  // commit to s1, s2, s3, ql, qr, qm, qo, and (the incomplete version of) qk
  //跟踪电路中的多项式转换为承诺，这些承诺将作为验证密钥的一部分，用于后续的证明验证过程。
  trace := NewTrace(spr, domain)
  vk.commitTrace(trace, domain, pk.KzgLagrange)
  
  return &pk,&vk,nil
}
```

### prove

```go
type Proof struct {

	// Commitments to the solution vectors
	LRO [3]kzg.Digest

	// Commitment to Z, the permutation polynomial
	Z kzg.Digest

	// Commitments to h1, h2, h3 such that h = h1 + Xⁿ⁺²*h2 + X²⁽ⁿ⁺²⁾*h3 is the quotient polynomial
	H [3]kzg.Digest

	Bsb22Commitments []kzg.Digest

	// Batch opening proof of linearizedPolynomial, l, r, o, s1, s2, qCPrime
	BatchedProof kzg.BatchOpeningProof

	// Opening proof of Z at zeta*mu
	ZShiftedOpening kzg.OpeningProof
}
```

prover 

这个结构体就是prover ，算是为后续的多个grotutine进行计算做铺垫，设计还是比较巧妙的。使用channel进行多个groutine之间的通信（更多的算是个信号，来确定协程执行的先后顺序）。这部分代码可以说是go异步编程的范例。

```go
type instance struct {
	ctx context.Context

	pk    *ProvingKey
	proof *Proof
	spr   *cs.SparseR1CS
	opt   *backend.ProverConfig

	fs             *fiatshamir.Transcript
	kzgFoldingHash hash.Hash // for KZG folding
	htfFunc        hash.Hash // hash to field function

	// polynomials
	x        []*iop.Polynomial // x stores tracks the polynomial we need
	bp       []*iop.Polynomial // blinding polynomials
	h        *iop.Polynomial   // h is the quotient polynomial
	blindedZ []fr.Element      // blindedZ is the blinded version of Z

	linearizedPolynomial       []fr.Element
	linearizedPolynomialDigest kzg.Digest

	fullWitness witness.Witness

	// bsb22 commitment stuff
	commitmentInfo constraint.PlonkCommitments
	commitmentVal  []fr.Element
	cCommitments   []*iop.Polynomial

	// challenges
	gamma, beta, alpha, zeta fr.Element

	// channel to wait for the steps
	chLRO,
	chQk,
	chbp,
	chZ,
	chH,
	chRestoreLRO,
	chZOpening,
	chLinearizedPolynomial,
	chGammaBeta chan struct{}

	domain0, domain1 *fft.Domain

	trace *Trace
}
```

```go
func Prove(spr *cs.SparseR1CS, pk *ProvingKey, fullWitness witness.Witness, opts ...backend.ProverOption) (*Proof, error) {
  // init instance
	g, ctx := errgroup.WithContext(context.Background())
	instance, err := newInstance(ctx, spr, pk, fullWitness, &opt)
  
  // solve constraints
	g.Go(instance.solveConstraints)  //等待 case <-s.chbp 任务完成  close  chLRO

	// complete qk
	g.Go(instance.completeQk) // 等待case <-s.chLRO 执行完成，close chQk

	// init blinding polynomials
	g.Go(instance.initBlindingPolynomials) // close chbp

	// derive gamma, beta (copy constraint)
	g.Go(instance.deriveGammaAndBeta) // 等待 case <-s.chLRO: 执行完成，close chGammaBeta

	// compute accumulating ratio for the copy constraint
	g.Go(instance.buildRatioCopyConstraint) //等待 case <-s.chGammaBeta: 执行完成，close chZ

	// compute h
	g.Go(instance.computeQuotient) //等待case <-s.chLRO: 以及 case <-s.chZ: case <-s.chRestoreLRO case <-s.chQk:  执行完成， close chH 

	// open Z (blinded) at ωζ (proof.ZShiftedOpening)
	g.Go(instance.openZ) // 等待case <-s.chH:执行完成，close chZOpening

	// linearized polynomial
	g.Go(instance.computeLinearizedPolynomial)//等待 case <-s.chH case <-s.chZOpening 执行完成，
  // close  chLinearizedPolynomial

	// Batch opening
	g.Go(instance.batchOpening)//等待 case <-s.chLRO case <-s.chLinearizedPolynomial 执行完。
  
  return instance.proof, nil

}
```

prove的过程则是拆分成多个协程来处理，之间使用channel进行通讯，代码的可读性还是极佳的（该说不说确实比rust/C++ 实现的prove过程观感要舒服一些，最起码没有一个prove代码两千多行 ）

从chan 关闭以及获取信号的顺序可以看出，prove过程中协程执行的先后顺序是：

`initBlindingPolynomials ` -----> `solveConstraints` -----> `completeQk` ----->`deriveGammaAndBeta` -----> `buildRatioCopyConstraint` -----> `computeQuotient` -----> `openZ`  -----> `computeLinearizedPolynomial`----->`batchOpening` 。

#### initBlindingPolynomials

```go
func (s *instance) initBlindingPolynomials() error {
	s.bp[id_Bl] = getRandomPolynomial(order_blinding_L)//1
	s.bp[id_Br] = getRandomPolynomial(order_blinding_R)//1
	s.bp[id_Bo] = getRandomPolynomial(order_blinding_O)//1
	s.bp[id_Bz] = getRandomPolynomial(order_blinding_Z)//2
	close(s.chbp) // 通道关闭，发送信号，告知监听s.chbp的协程，初始化多项式已经完成了，可以进行下一步的操作了。
	return nil
}
```

该方法是用于初始化盲化多项式（随机生成的多项式，用于隐藏敏感多项式的系数），之后备注的1,1,1,2 则是盲化多项式的度/阶

#### solveConstraints

```go
func (s *instance) solveConstraints() error {
  // 求解约束
	_solution, err := s.spr.Solve(s.fullWitness, s.opt.SolverOpts...)

	solution := _solution.(*cs.SparseR1CSSolution)
	evaluationLDomainSmall := []fr.Element(solution.L)
	evaluationRDomainSmall := []fr.Element(solution.R)
	evaluationODomainSmall := []fr.Element(solution.O)
	
  // 并发生成多项式
  wg.Add(2)
	go func() {
    // 指定拉格朗日基 (Lagrange basis) 和常规布局 (Regular layout) 来创建多项式
		s.x[id_L] = iop.NewPolynomial(&evaluationLDomainSmall, iop.Form{Basis: iop.Lagrange, Layout: iop.Regular})
		wg.Done()
	}()
	go func() {
		s.x[id_R] = iop.NewPolynomial(&evaluationRDomainSmall, iop.Form{Basis: iop.Lagrange, Layout: iop.Regular})
		wg.Done()
	}()

	s.x[id_O] = iop.NewPolynomial(&evaluationODomainSmall, iop.Form{Basis: iop.Lagrange, Layout: iop.Regular})

	wg.Wait()

	// 承诺 l, r, o 
	if err := s.commitToLRO(); err != nil {
		return err
	}
	close(s.chLRO)// 关闭chan，发送信号，告知下一个协程这部分已经处理完毕，进行进一步的处理
	return nil
}
```

```go
func (s *instance) commitToLRO() error {
	// 监听信号，确定之前的初始化多项式的步骤已经完成
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chbp:
	}

	g := new(errgroup.Group)

  // 并发生成 L、R、O 的承诺
	g.Go(func() (err error) {
		s.proof.LRO[0], err = s.commitToPolyAndBlinding(s.x[id_L], s.bp[id_Bl])
		return
	})

	g.Go(func() (err error) {
		s.proof.LRO[1], err = s.commitToPolyAndBlinding(s.x[id_R], s.bp[id_Br])
		return
	})

	g.Go(func() (err error) {
		s.proof.LRO[2], err = s.commitToPolyAndBlinding(s.x[id_O], s.bp[id_Bo])
		return
	})

	return g.Wait()
}
```

`solveConstraints` 和 `commitToLRO` 方法共同协作，通过求解约束、生成多项式和创建承诺，为后续证明步骤做好准备



#### completeQk

```go
func (s *instance) completeQk() error {
	// 获取Qk多项式的系数
  qk := s.trace.Qk.Clone()
	qkCoeffs := qk.Coefficients()

  // 获取witness
	wWitness, ok := s.fullWitness.Vector().(fr.Vector)

	copy(qkCoeffs, wWitness[:len(s.spr.Public)])

	// 等待上一步执行完成
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chLRO:
	}
  
	// 填充qk系数
	for i := range s.commitmentInfo {
		qkCoeffs[s.spr.GetNbPublicVariables()+s.commitmentInfo[i].CommitmentIndex] = s.commitmentVal[i]
	}

	s.x[id_Qk] = qk
	close(s.chQk)// 关闭chan，发送信号，告知下一个协程

	return nil
}
```

这一步主要是构建 Qk 多项式



#### deriveGammaAndBeta

```go
// deriveGammaAndBeta (copy constraint)
func (s *instance) deriveGammaAndBeta() error {
	... do something ...

	// wait for LRO to be committed
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chLRO:
	}
	
  
  // challenge gamma bate
	gamma, err := deriveRandomness(s.fs, "gamma", &s.proof.LRO[0], &s.proof.LRO[1], &s.proof.LRO[2])
	if err != nil {
		return err
	}

	bbeta, err := s.fs.ComputeChallenge("beta")
	if err != nil {
		return err
	}
	s.gamma = gamma
	s.beta.SetBytes(bbeta)

	close(s.chGammaBeta)// 关闭chan，告知下一个协程，挑战点已经生成。

	return nil
}
```

这一步就很简单了，生成ß γ 两个挑战点



#### buildRatioCopyConstraint

```go
func (s *instance) buildRatioCopyConstraint() (err error) {
	// 等待随机挑战点生成完毕
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chGammaBeta:
	}
  
  // 构建Z多项式
	s.x[id_Z], err = iop.BuildRatioCopyConstraint(
		[]*iop.Polynomial{
			s.x[id_L],
			s.x[id_R],
			s.x[id_O],
		},
		s.trace.S,
		s.beta,
		s.gamma,
		iop.Form{Basis: iop.Lagrange, Layout: iop.Regular},
		s.domain0,
	)

	// 生成Z多项式承诺
	s.proof.Z, err = s.commitToPolyAndBlinding(s.x[id_Z], s.bp[id_Bz])

	close(s.chZ)// 关闭chanel，发送信号

	return
}
```

构建 Z 多项式，它是证明过程中用于复制约束



#### computeQuotient

计算商多项式

```go
func (s *instance) computeQuotient() (err error) {
	... 加载信息 ...

	// wait for solver to be done && wait for Z to be committed or context done
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chLRO:
  
    case <-s.chZ:
	}
  
	// 计算å 
	if err = s.deriveAlpha(); err != nil {
		return err
	}
  

	// 准备特殊多项式
	identity := make([]fr.Element, n)
	identity[1].Set(&s.beta)

	s.x[id_ID] = iop.NewPolynomial(&identity, iop.Form{Basis: iop.Canonical, Layout: iop.Regular})
	s.x[id_LOne] = iop.NewPolynomial(&lone, iop.Form{Basis: iop.Lagrange, Layout: iop.Regular})
	s.x[id_ZS] = s.x[id_Z].ShallowClone().Shift(1)
	
  
  // 计算分子多项式 等待 chQk 通道关闭
	numerator, err := s.computeNumerator()
	if err != nil {
		return err
	}

	s.h, err = divideByXMinusOne(numerator, [2]*fft.Domain{s.domain0, s.domain1})
	if err != nil {
		return err
	}

	// 计算商多项式的承诺
	if err := commitToQuotient(s.h1(), s.h2(), s.h3(), s.proof, s.pk.Kzg); err != nil {
		return err
	}

	// 等待清理 LRO 相关资源的通道 (s.chRestoreLRO) 关闭，确保资源释放完成。
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chRestoreLRO:
	}

	close(s.chH)//通知之后的协程，商多项式计算/承诺已经全部完成

	return nil
}
```

computeNumerator 涉及到多个多项式的运算和变换，最终计算出分子多项式，为后续的 H/商多项式计算提供基础

* 该方法涉及大量复杂的计算，包括 FFT、多项式乘法、加法等。
* 采用了并行计算（`wgBuf`）来优化性能。
* 使用了 `scalingVector` 和 `scalingVectorRev` 进行高效的缩放操作。
* 约束条件的计算涉及多个多项式的组合和运算。



#### openZ

在 ωζ 处打开 Z 多项式

```go
func (s *instance) openZ() (err error) {
	// wait for H to be committed and zeta to be derived (or ctx.Done())
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chH:
	}
  
  // 计算 ζ 的偏移值
	var zetaShifted fr.Element
	zetaShifted.Mul(&s.zeta, &s.pk.Vk.Generator)
  // 获取Z的系数
	s.blindedZ = getBlindedCoefficients(s.x[id_Z], s.bp[id_Bz])
	// open z at zeta
	s.proof.ZShiftedOpening, err = kzg.Open(s.blindedZ, zetaShifted, s.pk.Kzg)
	if err != nil {
		return err
	}
	close(s.chZOpening)
	return nil
}
```

Tips： 打开承诺的多项式意味着揭示其在特定点 ωζ 处的真实值。该过程依赖于 KZG 承诺方案的打开操作。ζ 的偏移值用于指定打开的位置。

允许验证者在不泄露整个 Z 多项式的情况下，验证证明者是否知道 Z 多项式在特定点 ωζ 处的真实值。



#### computeLinearizedPolynomial

过并行计算和 KZG 承诺的方式，高效地计算并提交了线性多项式

```go
func (s *instance) computeLinearizedPolynomial() error {

	// wait for H to be committed and zeta to be derived (or ctx.Done())
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chH:
	}

  // 计算 Q 多项式在 ζ 处的取值 
	qcpzeta := make([]fr.Element, len(s.commitmentInfo))
	var blzeta, brzeta, bozeta fr.Element
	var wg sync.WaitGroup
	wg.Add(3 + len(s.commitmentInfo))
  
	// 并行 (goroutine) 遍历所有 Q 多项式 (Ql, Qr, Qm, Qk, Qcp) 并计算其在挑战点 ζ 处的取值
	for i := 0; i < len(s.commitmentInfo); i++ {
		go func(i int) {
			qcpzeta[i] = s.trace.Qcp[i].Evaluate(s.zeta)
			wg.Done()
		}(i)
	}
	
  // 并行计算 L, R, O 多项式在 ζ 处的估值
	go func() {
		blzeta = evaluateBlinded(s.x[id_L], s.bp[id_Bl], s.zeta)
		wg.Done()
	}()

	go func() {
		brzeta = evaluateBlinded(s.x[id_R], s.bp[id_Br], s.zeta)
		wg.Done()
	}()

	go func() {
		bozeta = evaluateBlinded(s.x[id_O], s.bp[id_Bo], s.zeta)
		wg.Done()
	}()

	// wait for Z to be opened at zeta (or ctx.Done())
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chZOpening:
	}
  // 获取Z多项式
	bzuzeta := s.proof.ZShiftedOpening.ClaimedValue

	wg.Wait()
	
  // 计算线性化多项式系数
	s.linearizedPolynomial = s.innerComputeLinearizedPoly(
		blzeta,
		brzeta,
		bozeta,
		s.alpha,
		s.beta,
		s.gamma,
		s.zeta,
		bzuzeta,
		qcpzeta,
		s.blindedZ,
		coefficients(s.cCommitments),
		s.pk,
	)
 // 将计算得到的线性化多项式系数 (blindedZCanonical) 提交到一个新的 KZG 承诺中，
	s.linearizedPolynomialDigest, err = kzg.Commit(s.linearizedPolynomial, s.pk.Kzg, runtime.NumCPU()*2)
	
	close(s.chLinearizedPolynomial)// 线性多项式生成完毕，可以执行下一步的证明操作了
	return nil
}
```

#### batchOpening

```go
func (s *instance) batchOpening() error {

	// wait for LRO to be committed (or ctx.Done())
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chLRO:
	}

	// wait for linearizedPolynomial to be computed (or ctx.Done())
	select {
	case <-s.ctx.Done():
		return errContextDone
	case <-s.chLinearizedPolynomial:
	}

  // 准备待打开的多项式以及承诺摘要
	polysQcp := coefficients(s.trace.Qcp)
	polysToOpen := make([][]fr.Element, 6+len(polysQcp))
	copy(polysToOpen[6:], polysQcp)

	polysToOpen[0] = s.linearizedPolynomial
	polysToOpen[1] = getBlindedCoefficients(s.x[id_L], s.bp[id_Bl])
	polysToOpen[2] = getBlindedCoefficients(s.x[id_R], s.bp[id_Br])
	polysToOpen[3] = getBlindedCoefficients(s.x[id_O], s.bp[id_Bo])
	polysToOpen[4] = s.trace.S1.Coefficients()
	polysToOpen[5] = s.trace.S2.Coefficients()

	digestsToOpen := make([]curve.G1Affine, len(s.pk.Vk.Qcp)+6)
	copy(digestsToOpen[6:], s.pk.Vk.Qcp)

	digestsToOpen[0] = s.linearizedPolynomialDigest
	digestsToOpen[1] = s.proof.LRO[0]
	digestsToOpen[2] = s.proof.LRO[1]
	digestsToOpen[3] = s.proof.LRO[2]
	digestsToOpen[4] = s.pk.Vk.S[0]
	digestsToOpen[5] = s.pk.Vk.S[1]

  // 批量打开多项式，使用Fiat Shamir 使其成为非交互式协议
	var err error
	s.proof.BatchedProof, err = kzg.BatchOpenSinglePoint(
		polysToOpen,
		digestsToOpen,
		s.zeta,
		s.kzgFoldingHash,
		s.pk.Kzg,
		s.proof.ZShiftedOpening.ClaimedValue.Marshal(),
	)

	return err
}
```

批量打开一组多项式，这是零知识证明过程的最后一步。它允许验证者在不泄露原始多项式的情况下，验证证明者知道这些多项式在特定点处的真实值。该过程依赖于 KZG 承诺方案的批量打开操作。

##### BatchOpenSinglePoint

```go
func BatchOpenSinglePoint(polynomials [][]fr.Element, digests []Digest, point fr.Element, hf hash.Hash, pk ProvingKey, dataTranscript ...[]byte) (BatchOpeningProof, error) {
	... do some check...
  
	// 计算多项式的预期值
	res.ClaimedValues = make([]fr.Element, len(polynomials))
	var wg sync.WaitGroup
	wg.Add(len(polynomials))
	for i := 0; i < len(polynomials); i++ {
		go func(_i int) {
			res.ClaimedValues[_i] = eval(polynomials[_i], point)
			wg.Done()
		}(i)
	}

	// wait for polynomial evaluations to be completed (res.ClaimedValues)
	wg.Wait()

	// 生成挑战点 γ
	gamma, err := deriveGamma(point, digests, res.ClaimedValues, hf, dataTranscript...)

  
	// ∑ᵢγⁱf(a)
  // 计算折叠评估值，并行计算所有承诺多项式的取值乘以对应的 gamma 次方之和
	var foldedEvaluations fr.Element
	chSumGammai := make(chan struct{}, 1)
	go func() {
		foldedEvaluations = res.ClaimedValues[nbDigests-1]
		for i := nbDigests - 2; i >= 0; i-- {
			foldedEvaluations.Mul(&foldedEvaluations, &gamma).
				Add(&foldedEvaluations, &res.ClaimedValues[i])
		}
		close(chSumGammai)
	}()

	// compute ∑ᵢγⁱfᵢ
  // 计算折叠多项式
	// 并行计算每个承诺多项式乘以对应的 gamma 次方，累加到相应的 foldedPolynomials 系数上。
	foldedPolynomials := make([]fr.Element, largestPoly)
	copy(foldedPolynomials, polynomials[0])
	gammas := make([]fr.Element, len(polynomials))
	gammas[0] = gamma
	for i := 1; i < len(polynomials); i++ {
		gammas[i].Mul(&gammas[i-1], &gamma)
	}

	for i := 1; i < len(polynomials); i++ {
		i := i
		parallel.Execute(len(polynomials[i]), func(start, end int) {
			var pj fr.Element
			for j := start; j < end; j++ {
				pj.Mul(&polynomials[i][j], &gammas[i-1])
				foldedPolynomials[j].Add(&foldedPolynomials[j], &pj)
			}
		})
	}

	// 计算最终承诺 (H):
	<-chSumGammai
	h := dividePolyByXminusA(foldedPolynomials, foldedEvaluations, point)
	foldedPolynomials = nil // same memory as h

	res.H, err = Commit(h, pk)
	if err != nil {
		return BatchOpeningProof{}, err
	}

	return res, nil
}
```

* 挑战 (gamma) 的生成过程使用了 **Fiat-Shamir** 转换，将交互式协议转换为非交互式协议。
* 折叠评估值和折叠多项式的计算利用了多项式乘法和累加的并行化，提高了效率。
* 最终承诺 (H) 包含了所有打开信息，可以用于验证证明的正确性。
* KZG 批量打开可以同时高效地验证多个多项式在同一个点。
* 该过程依赖于挑战的生成、折叠评估值和折叠多项式的计算，以及最终承诺的生成。

### verify

```go
func Verify(proof *Proof, vk *VerifyingKey, publicWitness fr.Vector, opts ...backend.VerifierOption) error {


	//生成挑战点，使用 Fiat-Shamir 哈希函数派生挑战值
	fs := fiatshamir.NewTranscript(cfg.ChallengeHash, "gamma", "beta", "alpha", "zeta")

	// The first challenge is derived using the public data: the commitments to the permutation,
	// the coefficients of the circuit, and the public inputs.
	// derive gamma from the Comm(blinded cl), Comm(blinded cr), Comm(blinded co)
	if err := bindPublicData(fs, "gamma", vk, publicWitness); err != nil {
		return err
	}
	gamma, err := deriveRandomness(fs, "gamma", &proof.LRO[0], &proof.LRO[1], &proof.LRO[2])
	if err != nil {
		return err
	}

	// derive beta from Comm(l), Comm(r), Comm(o)
	beta, err := deriveRandomness(fs, "beta")
	if err != nil {
		return err
	}

	// derive alpha from Com(Z), Bsb22Commitments
	alphaDeps := make([]*curve.G1Affine, len(proof.Bsb22Commitments)+1)
	for i := range proof.Bsb22Commitments {
		alphaDeps[i] = &proof.Bsb22Commitments[i]
	}
	alphaDeps[len(alphaDeps)-1] = &proof.Z
	alpha, err := deriveRandomness(fs, "alpha", alphaDeps...)
	if err != nil {
		return err
	}

	// derive zeta, the point of evaluation
	zeta, err := deriveRandomness(fs, "zeta", &proof.H[0], &proof.H[1], &proof.H[2])
	if err != nil {
		return err
	}

	// 多项式求值 zhZeta=ζⁿ-1
	var zetaPowerM, zhZeta, lagrangeOne fr.Element
	var bExpo big.Int
	one := fr.One()
	bExpo.SetUint64(vk.Size)
	zetaPowerM.Exp(zeta, &bExpo)
	zhZeta.Sub(&zetaPowerM, &one) // ζⁿ-1
	lagrangeOne.Sub(&zeta, &one). // ζ-1
					Inverse(&lagrangeOne).         // 1/(ζ-1)
					Mul(&lagrangeOne, &zhZeta).    // (ζ^n-1)/(ζ-1)
					Mul(&lagrangeOne, &vk.SizeInv) // 1/n * (ζ^n-1)/(ζ-1)

	// PI = ∑_{i<n} Lᵢ*wᵢ
  // PI 是线性化多项式的系数之和，它表示在评估点 zeta 处的多项式值
	var pi fr.Element
	var accw fr.Element
	{
		// [ζ-1,ζ-ω,ζ-ω²,..]
		dens := make([]fr.Element, len(publicWitness))
		accw.SetOne()
		for i := 0; i < len(publicWitness); i++ {
			dens[i].Sub(&zeta, &accw)
			accw.Mul(&accw, &vk.Generator)
		}

		// [1/(ζ-1),1/(ζ-ω),1/(ζ-ω²),..]
		invDens := fr.BatchInvert(dens)

		accw.SetOne()
		var xiLi fr.Element
		for i := 0; i < len(publicWitness); i++ {
			xiLi.Mul(&zhZeta, &invDens[i]).
				Mul(&xiLi, &vk.SizeInv).
				Mul(&xiLi, &accw).
				Mul(&xiLi, &publicWitness[i]) // Pi[i]*(ωⁱ/n)(ζ^n-1)/(ζ-ω^i)
			accw.Mul(&accw, &vk.Generator)
			pi.Add(&pi, &xiLi)
		}

		if cfg.HashToFieldFn == nil {
			cfg.HashToFieldFn = hash_to_field.New([]byte("BSB22-Plonk"))
		}
		var hashedCmt fr.Element
		nbBuf := fr.Bytes
		if cfg.HashToFieldFn.Size() < fr.Bytes {
			nbBuf = cfg.HashToFieldFn.Size()
		}
		var wPowI, den, lagrange fr.Element
		for i := range vk.CommitmentConstraintIndexes {
			cfg.HashToFieldFn.Write(proof.Bsb22Commitments[i].Marshal())
			hashBts := cfg.HashToFieldFn.Sum(nil)
			cfg.HashToFieldFn.Reset()
			hashedCmt.SetBytes(hashBts[:nbBuf])

			// Computing Lᵢ(ζ) where i=CommitmentIndex
			wPowI.Exp(vk.Generator, big.NewInt(int64(vk.NbPublicVariables)+int64(vk.CommitmentConstraintIndexes[i])))
			den.Sub(&zeta, &wPowI) // ζ-wⁱ
			lagrange.SetOne().
				Sub(&zetaPowerM, &lagrange). // ζⁿ-1
				Mul(&lagrange, &wPowI).      // wⁱ(ζⁿ-1)
				Div(&lagrange, &den).        // wⁱ(ζⁿ-1)/(ζ-wⁱ)
				Mul(&lagrange, &vk.SizeInv)  // wⁱ/n (ζⁿ-1)/(ζ-wⁱ)

			xiLi.Mul(&lagrange, &hashedCmt)
			pi.Add(&pi, &xiLi)
		}
	}

	var _s1, _s2, tmp fr.Element
	l := proof.BatchedProof.ClaimedValues[1]
	r := proof.BatchedProof.ClaimedValues[2]
	o := proof.BatchedProof.ClaimedValues[3]
	s1 := proof.BatchedProof.ClaimedValues[4]
	s2 := proof.BatchedProof.ClaimedValues[5]

	// Z(ωζ)
	zu := proof.ZShiftedOpening.ClaimedValue

	// α²*L₁(ζ)
	var alphaSquareLagrangeOne fr.Element
	alphaSquareLagrangeOne.Mul(&lagrangeOne, &alpha).
		Mul(&alphaSquareLagrangeOne, &alpha) // α²*L₁(ζ)

	// 计算全代数关系的常数系数，对应于线性化多项式的值
	//ζ PI(ζ) - α²*L₁(ζ) + α(l(ζ)+β*s1(ζ)+γ)(r(ζ)+β*s2(ζ)+γ)(o(ζ)+γ)*z(ωζ)
	var constLin fr.Element
	constLin.Mul(&beta, &s1).Add(&constLin, &gamma).Add(&constLin, &l)       // (l(ζ)+β*s1(ζ)+γ)
	tmp.Mul(&s2, &beta).Add(&tmp, &gamma).Add(&tmp, &r)                      // (r(ζ)+β*s2(ζ)+γ)
	constLin.Mul(&constLin, &tmp)                                            // (l(ζ)+β*s1(ζ)+γ)(r(ζ)+β*s2(ζ)+γ)
	tmp.Add(&o, &gamma)                                                      // (o(ζ)+γ)
	constLin.Mul(&tmp, &constLin).Mul(&constLin, &alpha).Mul(&constLin, &zu) // α(l(ζ)+β*s1(ζ)+γ)(r(ζ)+β*s2(ζ)+γ)(o(ζ)+γ)*z(ωζ)

	constLin.Sub(&constLin, &alphaSquareLagrangeOne).Add(&constLin, &pi) // PI(ζ) - α²*L₁(ζ) + α(l(ζ)+β*s1(ζ)+γ)(r(ζ)+β*s2(ζ)+γ)(o(ζ)+γ)*z(ωζ)
	constLin.Neg(&constLin)                                              // -[PI(ζ) - α²*L₁(ζ) + α(l(ζ)+β*s1(ζ)+γ)(r(ζ)+β*s2(ζ)+γ)(o(ζ)+γ)*z(ωζ)]

	// 验证线性多项式的开点 是否与 -constLin 相等
	openingLinPol := proof.BatchedProof.ClaimedValues[0]
	if !constLin.Equal(&openingLinPol) {
		return errAlgebraicRelation
	}

	// 计算多项式的摘要
	// α²*L₁(ζ)*[Z] +
	// _s1*[s3]+_s2*[Z] + l(ζ)*[Ql] +
	// l(ζ)r(ζ)*[Qm] + r(ζ)*[Qr] + o(ζ)*[Qo] + [Qk] + ∑ᵢQcp_(ζ)[Pi_i] -
	// Z_{H}(ζ)*(([H₀] + ζᵐ⁺²*[H₁] + ζ²⁽ᵐ⁺²⁾*[H₂])
	// where
	// _s1 =  α*(l(ζ)+β*s1(ζ)+γ)*(r(ζ)+β*s2(ζ)+γ)*β*Z(μζ)
	// _s2 = -α*(l(ζ)+β*ζ+γ)*(r(ζ)+β*u*ζ+γ)*(o(ζ)+β*u²*ζ+γ)

	// _s1 = α*(l(ζ)+β*s1(β)+γ)*(r(ζ)+β*s2(β)+γ)*β*Z(μζ)
	_s1.Mul(&beta, &s1).Add(&_s1, &l).Add(&_s1, &gamma)                   // (l(ζ)+β*s1(β)+γ)
	tmp.Mul(&beta, &s2).Add(&tmp, &r).Add(&tmp, &gamma)                   // (r(ζ)+β*s2(β)+γ)
	_s1.Mul(&_s1, &tmp).Mul(&_s1, &beta).Mul(&_s1, &alpha).Mul(&_s1, &zu) // α*(l(ζ)+β*s1(β)+γ)*(r(ζ)+β*s2(β)+γ)*β*Z(μζ)

	// _s2 = -α*(l(ζ)+β*ζ+γ)*(r(ζ)+β*u*ζ+γ)*(o(ζ)+β*u²*ζ+γ)
	_s2.Mul(&beta, &zeta).Add(&_s2, &gamma).Add(&_s2, &l)                                                     // (l(ζ)+β*ζ+γ)
	tmp.Mul(&beta, &vk.CosetShift).Mul(&tmp, &zeta).Add(&tmp, &gamma).Add(&tmp, &r)                           // (r(ζ)+β*u*ζ+γ)
	_s2.Mul(&_s2, &tmp)                                                                                       // (l(ζ)+β*ζ+γ)*(r(ζ)+β*u*ζ+γ)
	tmp.Mul(&beta, &vk.CosetShift).Mul(&tmp, &vk.CosetShift).Mul(&tmp, &zeta).Add(&tmp, &o).Add(&tmp, &gamma) // (o(ζ)+β*u²*ζ+γ)
	_s2.Mul(&_s2, &tmp).Mul(&_s2, &alpha).Neg(&_s2)                                                           // -α*(l(ζ)+β*ζ+γ)*(r(ζ)+β*u*ζ+γ)*(o(ζ)+β*u²*ζ+γ)

	// α²*L₁(ζ) - α*(l(ζ)+β*ζ+γ)*(r(ζ)+β*u*ζ+γ)*(o(ζ)+β*u²*ζ+γ)
	var coeffZ fr.Element
	coeffZ.Add(&alphaSquareLagrangeOne, &_s2)

	// l(ζ)*r(ζ)
	var rl fr.Element
	rl.Mul(&l, &r)

	// -ζⁿ⁺²*(ζⁿ-1), -ζ²⁽ⁿ⁺²⁾*(ζⁿ-1), -(ζⁿ-1)
	nPlusTwo := big.NewInt(int64(vk.Size) + 2)
	var zetaNPlusTwoZh, zetaNPlusTwoSquareZh, zh fr.Element
	zetaNPlusTwoZh.Exp(zeta, nPlusTwo)
	zetaNPlusTwoSquareZh.Mul(&zetaNPlusTwoZh, &zetaNPlusTwoZh)                          // ζ²⁽ⁿ⁺²⁾
	zetaNPlusTwoZh.Mul(&zetaNPlusTwoZh, &zhZeta).Neg(&zetaNPlusTwoZh)                   // -ζⁿ⁺²*(ζⁿ-1)
	zetaNPlusTwoSquareZh.Mul(&zetaNPlusTwoSquareZh, &zhZeta).Neg(&zetaNPlusTwoSquareZh) // -ζ²⁽ⁿ⁺²⁾*(ζⁿ-1)
	zh.Neg(&zhZeta)

	var linearizedPolynomialDigest curve.G1Affine
	points := append(proof.Bsb22Commitments,
		vk.Ql, vk.Qr, vk.Qm, vk.Qo, vk.Qk,
		vk.S[2], proof.Z,
		proof.H[0], proof.H[1], proof.H[2],
	)

	qC := make([]fr.Element, len(proof.Bsb22Commitments))
	copy(qC, proof.BatchedProof.ClaimedValues[6:])

	scalars := append(qC,
		l, r, rl, o, one,
		_s1, coeffZ,
		zh, zetaNPlusTwoZh, zetaNPlusTwoSquareZh,
	)
	if _, err := linearizedPolynomialDigest.MultiExp(points, scalars, ecc.MultiExpConfig{}); err != nil {
		return err
	}

	// 折叠证明，将初始证明折叠成更小的证明，以便于批量验证
	digestsToFold := make([]curve.G1Affine, len(vk.Qcp)+6)
	copy(digestsToFold[6:], vk.Qcp)
	digestsToFold[0] = linearizedPolynomialDigest
	digestsToFold[1] = proof.LRO[0]
	digestsToFold[2] = proof.LRO[1]
	digestsToFold[3] = proof.LRO[2]
	digestsToFold[4] = vk.S[0]
	digestsToFold[5] = vk.S[1]
	foldedProof, foldedDigest, err := kzg.FoldProof(
		digestsToFold,
		&proof.BatchedProof,
		zeta,
		cfg.KZGFoldingHash,
		zu.Marshal(),
	)
	if err != nil {
		return err
	}

	// 批量验证，使用 KZG 算法批量验证折叠后的证明和针对 Z 的证明
  // 需要提供评估点 zeta 和其偏移版本 (shiftedZeta)
	var shiftedZeta fr.Element
	shiftedZeta.Mul(&zeta, &vk.Generator)
	err = kzg.BatchVerifyMultiPoints([]kzg.Digest{
		foldedDigest,
		proof.Z,
	},
		[]kzg.OpeningProof{
			foldedProof,
			proof.ZShiftedOpening,
		},
		[]fr.Element{
			zeta,
			shiftedZeta,
		},
		vk.Kzg,
	)

	log.Debug().Dur("took", time.Since(start)).Msg("verifier done")

	return err
}
```

`Verify` 函数通过检查挑战响应、评估线性化多项式以及使用 KZG 算法确保一致性来验证 Plonk 证明的有效性。
