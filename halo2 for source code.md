## Halo2 for source code

> 了解个概念
>
> Halo2电路组件以及用途：
>
> * Columns：我们可以将电路的输入和输出概念化为给定有限域F上m列n行的矩阵1。列可以分为三种类型：
>
>   * Instance columns：包含了Prover/Verifier之间共享的输入，通常用于公共输入 (public inputs)，例如SHA256的结果，Merkle Tree的根。
>
>   * Advice columns：包含了private input & 电路运行中所需的中间变量，即witness，这部分只有Prover知道。
>
>   * Fixed columns：包含在key generation阶段设置的preprocessed values，可以视为是电路结构固定的一部分，也是可以被pre-compute的。
>
> * Rows：矩阵中的行数通常是2的幂，受有限域F的大小限制；行数对应于Plonkish算术化(arithmetization)中的n-th单位根(nth root of unity)
>
> * Gate：门通常是由一组约束构成，这组约束通常受selector控制。Halo2提供两种类型的门：
>
>    * Standard gate：标准门支持通用算术，例如域乘法和除法。
>
>   * Custom gate：自定义门更具表现力，能够支持电路中的专门操作。
> 
> * Region：在Halo2中，电路被划分为由相邻的行和列组成的region，在region中可以采用相对偏移(relative offsets)的方式访问Cell



Halo2 example

```rust
fn main() {
    //  The number of rows in our circuit cannot exceed 2^k  
    let k = 5;

    let constant = Fp::from(7);

    let a = Fp::from(2);
    let b = Fp::from(3);
    let c: Fp = constant * a.square() * b.square();

    let circuit = MyCiruit {  // ouerself defined circuit
        a: Value::known(a),
        b: Value::known(b),
        constant,
    };

    let mut public_inputs = vec![c];  // circuit public inputs

    let prover = MockProver::run(k, &circuit, vec![public_inputs.clone()]).unwrap();  // gerenate proof
    assert_eq!(prover.verify(), Ok(()));
}
```

以上是halo2 官方给出的一个实例代码的入口。接下来要从几个方面来看

* 自定义电路，即上述事例中的MyCircuit
  * 如何定义电路的逻辑
  * 如何设置public inputs
  * 如何设置 private inputs
  * 如何设置constant
  * etc
* 生成证明
  * `Mockprover:: run`
  * 用的什么算法生成的proof
* 验证证明
  * `prover.verify`
  * 如何验证的proof

### 自定义电路

需求，定义一个`a*b =c` 的电路

```rust
trait NumericInstructions<F: Field>: Chip<F> {
    /// variable representing a number
    type Num;

    fn load_private(&self, layouter: impl Layouter<F>, vale: Value<F>) -> Result<Self::Num, Error>;

    fn load_constant(&self, layouter: impl Layouter<F>, constant: F) -> Result<Self::Num, Error>;

    ///  c = a*b
    fn mul(
        &self,
        layouter: impl Layouter<F>,
        a: Self::Num,
        b: Self::Num,
    ) -> Result<Self::Num, Error>;

    fn expose_public(
        &self,
        layouter: impl Layouter<F>,
        num: Self::Num,
        row: usize,
    ) -> Result<(), Error>;
}
```

`NumericInstructions`  这是自定义的一个接口，里边定义了：

1. `Load_private ` 加载数据至电路，作为private inputs
2. `load_constant` 加载constant 至电路，在halo2中称为`fixed`
3. `mul` 这是实现 `a*b =c` 逻辑方法
4. `expose_public`  电路中的 public_inputs

```rust
// halo2中的特定模块，电路实现的基础模块。
struct FieldChip<F: Field> {
    config: FieldConfig,
    _marker: PhantomData<F>,
}

impl<F: Field> Chip<F> for FieldChip<F> {
    type Config = FieldConfig;
    type Loaded = ();
    fn config(&self) -> &Self::Config {
        &self.config
    }

    fn loaded(&self) -> &Self::Loaded {
        &()
    }
}
```



Chip需要配置实现所有所需指令所需的`columns`、`permutations`和`gates`。

chip state 存储在config结构中。这是chip在配置过程中生成的，然后存储在chip内部。

```rust
#[derive(Clone, Debug)]
struct FieldConfig {
    // private inputs
    advice: [Column<Advice>; 2],

    //public inputs / instance columns
    instance: Column<Instance>,

    //启用 S_mul gates 无需未使用“NumericInstructions：：mul”的单元格施加任何约束。 在构建较大的电路时很重要，因为在电路中，列由多组指令使用。  确定激活门的条件
    s_mul: Selector,
}


impl<F: Field> FieldChip<F> {
	fn construct(config: FieldConfig) -> Self{}
  
  fn configure(
        meta: &mut ConstraintSystem<F>,
        advices: [Column<Advice>; 2],
        instance: Column<Instance>,
        constant: Column<Fixed>,
    ) -> <Self as Chip<F>>::Config{
    		// 对电路输入信息的检查
      	meta.enable_equality(instance);
        meta.enable_constant(constant);
        for cloum in &advices {
            meta.enable_equality(*cloum);
        }
      	// 创建selector，启用mul 约束
      	let s_mul = meta.selector();
      	
      	//创建乘法门
      	meta.create_gate("mul", |meta| {
        	  // | advive 1  | advice 2  | selector |
            // |-----------|-----------|----------|
            // | lhs 			 | rhs 			 | s_mul 		|
            // | out 			 |     			 |       		|  
        		let lhs = meta.query_advice(advices[0], Rotation::cur());
            let rhs = meta.query_advice(advices[1], Rotation::cur());
            let out = meta.query_advice(advices[0], Rotation::next());
            let s_mul = meta.query_selector(s_mul);
          	
          	
           // lhs * rhs  = out
          	vec![s_mul * (lhs * rhs - out)]
        }
        FieldConfig {
            advice: advices,
            instance,
            s_mul,
        }
  	}
  	
}
```

> Gate 可以引用我们想要的任何相对偏移量，但每个不同的偏移量都会**增加证明的成本**。最常见的偏移量是 **0（当前行**）、**1（下一行）和 -1（上一行**），其中“Rotation”具有特定的构造函数。
>
> 从“create_gate”返回的多项式表达式将被证明系统约束为等于零。于是就有了 lhs * rhs - out =0 



FieldChip 的具体实现，（自定义电路的借口实现，为下一步实现自定义电路做铺垫）

```rust
impl<F: Field> NumericInstructions<F> for FieldChip<F> {
    type Num = Number<F>;

    fn load_constant(
        &self,
        mut layouter: impl Layouter<F>,
        constant: F,
    ) -> Result<Self::Num, Error> {
        let config = self.config();

        layouter.assign_region(
            || "load constant",
            |mut region| {
                region
                    .assign_advice_from_constant(|| "constant input", config.advice[0], 0, constant)
                    .map(Number)
            },
        )
    }
  	
    fn load_private(
          &self,
          mut layouter: impl Layouter<F>,
          value: Value<F>,
      ) -> Result<Self::Num, Error> {
          let config = self.config();

          layouter.assign_region(
              || "load private",
              |mut region| {
                  region
                      .assign_advice(|| "private input", config.advice[0], 0, || value)
                      .map(Number)
              },
          )
      }
  	
        fn mul(
          &self,
          mut layouter: impl Layouter<F>,
          a: Self::Num,
          b: Self::Num,
      ) -> Result<Self::Num, Error> {
          let config = self.config();

          layouter.assign_region(
              || "mul",
              |mut region: Region<'_, F>| {
                  config.s_mul.enable(&mut region, 0)?;
                  a.0.copy_advice(|| "lhs", &mut region, config.advice[0], 0)?;
                  b.0.copy_advice(|| "rhs", &mut region, config.advice[1], 0)?;
                  let value = a.0.value().copied() * b.0.value();
                  region
                      .assign_advice(|| "mul", config.advice[0], 1, || value)
                      .map(Number)
              },
          )
      }
			
      fn expose_public(
          &self,
          mut layouter: impl Layouter<F>,
          num: Self::Num,
          row: usize,
      ) -> Result<(), Error> {
          let config = self.config();

          layouter.constrain_instance(num.0.cell(), config.instance, row)
      }
  	
}
```

上述方法是自定义NumericInstructions 接口实现。仔细观察发现这几个方法的有几个共同的特点

* 形参中的`layouter` 这个代表着halo2 电路中的布局，将Chip添加到一个电路上需要布局。Layouter就是用来实现布局，对电路进行分配，分配到一张表中，方便之后的读取进而进行其他的操作。
*  `assign_xxxx`  
  * `assign_constrain_instance`  约束 ['Cell'] 在绝对位置等于实例列的行值。
  * `assign_region`  定一个区域，在这个区域内进行各种`assignment` (任务)
    * `assign_advice_from_constant`  在此区域的`advice` 或`fixed` 分配一个常量。 *ConstraintSystem::enable_constant*
    * `assign_advice`   分配witness 列
    * `assign_advice_from_instance` 
  * `assgin_table`:  获取一个table，并设置table

自定义电路具体的实现

```rust
#[derive(Default)]
struct MyCiruit<F: Field> {
    constant: F,
    a: Value<F>,
    b: Value<F>,
}

//必须实现 Circuit接口
impl<F: Field> Circuit<F> for MyCiruit<F> {
    type Config = FieldConfig;
    type FloorPlanner = SimpleFloorPlanner;

    fn without_witnesses(&self) -> Self {
        Self::default()
    }

    fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
        let advices = [meta.advice_column(), meta.advice_column()];
        let instance = meta.instance_column();
        let constant = meta.fixed_column();
        FieldChip::configure(meta, advices, instance, constant)
    }

    fn synthesize(
        &self,
        config: Self::Config,
        mut layouter: impl Layouter<F>,
    ) -> Result<(), Error> {
        let field_chip = FieldChip::<F>::construct(config);

        // Load our private values into the circuit.
        let a = field_chip.load_private(layouter.namespace(|| "load a"), self.a)?;
        let b = field_chip.load_private(layouter.namespace(|| "load b"), self.b)?;

        // Load the constant factor into the circuit.
        let constant =
            field_chip.load_constant(layouter.namespace(|| "load constant"), self.constant)?;

        // but it's more efficient to implement it as:
        //     ab   = a*b
        //     absq = ab^2
        //     c    = constant*absq

        let ab = field_chip.mul(layouter.namespace(|| "ab"), a, b)?;
        let ab_sq = field_chip.mul(layouter.namespace(|| "ab_sq"), ab.clone(), ab)?;
        let c = field_chip.mul(layouter.namespace(|| "c"), constant, ab_sq)?;

        // Expose the result as a public input to the circuit.
        field_chip.expose_public(layouter.namespace(|| "expose c"), c, 0)
    }
}
```

* `type FloorPlanner = SimpleFloorPlanner` 电路布局器。
* `without_witness`  电路副本
* `configure`  电路的配置
* `synthesize`  电路核心逻辑的入口，根据给定的`CS` (ConstraintSystem) 合成电路。

至此一个halo2_demo代码已经解析完成。然后进入核心部分，proof 生成和验证。

### Prover 

```rust
fn main(){
  ...
  MockProver::run(k, &circuit, vec![public_inputs.clone()]).unwrap();
  ...
}
```

```rust
pub fn run<ConcreteCircuit: Circuit<F>>(
        k: u32,
        circuit: &ConcreteCircuit,
        instance: Vec<Vec<F>>,
    ) -> Result<Self, Error> {
      ..... 
      ConcreteCircuit::FloorPlanner::synthesize(&mut prover, circuit, config, constants)?;
      let (cs, selector_polys) = prover.cs.compress_selectors(prover.selectors.clone());
      prover.cs = cs;
      prover.fixed.extend(selector_polys.into_iter().map(|poly| {
            let mut v = vec![CellValue::Unassigned; n];
            for (v, p) in v.iter_mut().zip(&poly[..]) {
                *v = CellValue::Assigned(*p);
            }
            v
        }));
     Ok(prover) 
}
```

* `ConcreteCircuit::FloorPlanner::synthesize(&mut prover, circuit, config, constants)?;` 这部分则是将自定义的电路给实例化
* `prover.cs.compress_selectors` 将提供的selector ，压缩优化，转换为多项式，以及CS
* `prover.fixed.extend` 这部分代码则是在填充fixed column



### Verify

```rust
fn main(){
  ...
  assert_eq!(prover.verify(), Ok(()));
  ...
}
```

```rust
pub fn verify(&self) -> Result<(), Vec<VerifyFailure>> {
  	
  // check selector_errors  which not in region
  let selector_errors = self.regions.iter().enumerate().flat_map(|(r_i, r)| {
    ... do some check...
    
     match cell.column.column_type() {
                                    Any::Instance => {
                                        // Handle instance cells, which are not in the region.
                                        let instance_value =
                                            &self.instance[cell.column.index()][cell_row];
                                        match instance_value {
                                            InstanceValue::Assigned(_) => None,
                                            _ => Some(VerifyFailure::InstanceCellNotAssigned {
                                                gate: (gate_index, gate.name()).into(),
                                                region: (r_i, r.name.clone()).into(),
                                                gate_offset: *selector_row,
                                                column: cell.column.try_into().unwrap(),
                                                row: cell_row,
                                            }),
                                        }
                                    }
                                    _ => {
                                        // Check that it was assigned!
                                        if r.cells.contains(&(cell.column, cell_row)) {
                                            None
                                        } else {
                                            Some(VerifyFailure::CellNotAssigned {
                                                gate: (gate_index, gate.name()).into(),
                                                region: (r_i, r.name.clone()).into(),
                                                gate_offset: *selector_row,
                                                column: cell.column,
                                                offset: cell_row as isize
                                                    - r.rows.unwrap().0 as isize,
                                            })
                                        }
                                    }
                                }
  });
  // Check that all gates are satisfied for all rows.
  let gate_errors = 
		self.cs
                .gates
                .iter()
                .enumerate()
                .flat_map(|(gate_index, gate)| {
                   gate.polynomials().iter().enumerate().filter_map(
                            move |(poly_index, poly)| match poly.evaluate(...){
                            Value::Real(x) if x.is_zero_vartime() => None,
                            Value::Real(_) => Some(VerifyFailure::ConstraintNotSatisfied {...}
                            Value::Poison => Some(VerifyFailure::ConstraintPoisoned {...}
                     }
                  };
  // check lookup error                       
  let lookup_errors = 
      ......
      Some(VerifyFailure::Lookup {
                                    lookup_index,
                                    location: FailureLocation::find_expressions(
                                        &self.cs,
                                        &self.regions,
                                        *input_row,
                                        lookup.input_expressions.iter(),
                                    ),
                                })
                            } else {
                                None
                            }
 let perm_errors = {
     ...... 
     
     }
                     
}
```

* `selector_error` : 检查每个region内，实例化门中使用的所有单元都已被分配 .
  * 迭代每一个region，对于每个region，迭代selecor，对于每个启用的selector，找出由该选择器启用的所有gate，对于每个门，遍历其查询的所有cell。若cell没有被正确地分配，那么就生成一个`VerifyFailure::CellNotAssigned`或`VerifyFailure::InstanceCellNotAssigned`错误，表示单元格没有被分配。
* `gate_error` :  检查所有的gate 是否都满足约束条件，遍历多项式，并计算值，如果不为0，则约束不满足，生成`VerifyFailure::ConstraintNotSatisfied` 错误。这个错误包含了约束的详细信息，包括约束的名称、位置以及在该位置的单元格值。
  * 若为 `Value::Poison`   返回`VerifyFailure::ConstraintPoisoned` 
  * 这段代码的目的是找出所有不满足约束的门，并生成相应的错误信息。这是零知识证明中非常重要的一步，因为它确保了证明的正确性。如果所有的门都满足约束，那么证明就是有效的。否则，证明就是无效的。
* `lookup_error` 检查所有的查找（lookups）是否满足约束条件。
  * 对于每个查找，计算其在表格中的所有行的值（`lookup.table_expressions.iter().map(move |c| load(c, self.usable_rows.end - 1))`）并将其存储在`fill_row`中。
  * 对于每个查找，遍历其在可用行中的所有行（`self.usable_rows.clone().filter_map(|table_row| {...})`），并将不等于`fill_row`的行的值存储在`table`中。
  * 对于每个查找，遍历其在可用行中的所有行（`self.usable_rows.clone().filter_map(|input_row| {...})`），并将不等于`fill_row`的行的值以及原始输入行的索引存储在`inputs`中。
  * 对 `inputs`  `table` 进行排序，对于`inputs`中的每个元素，如果它不在`table`中，那么就生成一个`VerifyFailure::Lookup`错误，表示查找失败。
* `perm_error` 检查所有的排列（permutations）是否满足约束条件
  * 遍历所有的排列列（`self.permutation.mapping.iter().enumerate()`）。
  * 对于每个列，遍历其所有的行（`values.iter().enumerate()`）并检查单元格的值是否被映射保留。
  * 如果原始单元格的值和排列后的单元格的值不相等（`if original_cell == permuted_cell {...} else {...}`），那么就生成一个`VerifyFailure::Permutation`错误，表示排列失败。









