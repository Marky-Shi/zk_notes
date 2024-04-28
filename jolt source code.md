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
fn fn make_prove_func(&self) -> TokenStream2 {
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

