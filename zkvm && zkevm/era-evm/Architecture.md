## Architecture

Zksync-era 中的的zk-evm 是兼容evm的，区别于给予堆栈的evm，zksync-evm是给予寄存器的。纯rust实现则是 https://github.com/matter-labs/era-zk_evm.git。 而zksync-evm 则是执行zkEVM assembly，类似于evm的opcode。

```shell
repo:  https://github.com/matter-labs/era-zk_evm.git
commit: 9ddd65ce2910049bae402aff0230e6e1bfed578a
```

Path: era-zk_evm/src/vm_state/cycle.rs

```rust
pub fn cycle<DT: tracing::Tracer<N, E, SupportedMemory = M>>(
    &mut self,
    tracer: &mut DT,
) -> anyhow::Result<()>{
  // 验证代码哈希
  //在执行指令之前，会验证默认的AA代码哈希和EVM模拟器代码哈希的格式是否正确。这是为了确保代码的完整性和安全性。
  self.block_properties
            .default_aa_code_hash
            .to_big_endian(&mut buffer);
        assert!(
            zkevm_opcode_defs::definitions::versioned_hash::ContractCodeSha256Format::is_valid(
                &buffer
            ),
            "default AA bytecode hash is malfored: {:?}",
            &buffer,
        );
        self.block_properties
            .evm_simulator_code_hash
            .to_big_endian(&mut buffer);
        assert!(
            zkevm_opcode_defs::definitions::versioned_hash::ContractCodeSha256Format::is_valid(
                &buffer
            ),
            "EVM simulator bytecode hash is malfored: {:?}",
            &buffer,
        );
  
 
  //decode opcode
  // 从VM的状态中读取并解码下一条指令
  let (after_masking_decoded, delayed_changes, skip_cycle) = read_and_decode(
            &self.local_state,
            &mut self.memory,
            &mut self.witness_tracer,
            tracer,
        );
  delayed_changes.apply(&mut self.local_state);
  
  
  // memory process
  // 进行内存操作
  MemOpsProcessor::<N, E> {
            sp: self.local_state.callstack.get_current_stack().sp,
        };
  // compute  address && select operands
  // 根据指令的操作数类型和寄存器值，计算出内存访问的地址。
  // 从寄存器或内存中读取操作数的值。
  // 如 堆栈上的pop/push/offset等操作。
  let (src0_reg_value, mut src0_mem_location) = mem_processor
    .compute_addresses_and_select_operands(
        self,
        after_masking_decoded.src0_reg_idx,
        after_masking_decoded.imm_0,
        after_masking_decoded.variant.src0_operand_type,
        false,
    );
	
  // 特殊处理 Nop 指令
  // 如果指令是NOP（无操作），则将src0的内存位置设置为None。
  if after_masking_decoded.variant.opcode == zkevm_opcode_defs::Opcode::Nop(NopOpcode) {
    src0_mem_location = None;
  }
  
  //内存读取
  let src0_mem_value = if let Some(src0_mem_location) = src0_mem_location {
    ...
    PrimitiveValue {
       value: u256_word,
       is_pointer,
     }
  }else {
     PrimitiveValue::empty()
  };
  
  // 操作数选择和交换
  let src0 = match after_masking_decoded.variant.src0_operand_type { ... };
	let src1 = self.select_register_value(after_masking_decoded.src1_reg_idx);

  // PC program counter 
  let mut new_pc = self.local_state.callstack.get_current_stack().pc;
  if !skip_cycle {
      new_pc = new_pc.wrapping_add(E::PcOrImm::from_u64_clipped(1u64));
  }
	
  // 构建prestate
  ...
  let prestate = PreState {
      src0,
      src1,
      dst0_mem_location,
      new_pc,
      is_kernel_mode,
   };
  // 执行指令  apply中涵盖了对应的指令集如Add,Sub,Ptr,Shitf etc 
  after_masking_decoded.apply(self, prestate)?;
  
  // 更新时间戳和周期计数器 && 记录执行的过程中的信息
  if DT::CALL_AFTER_EXECUTION {
    tracer.after_execution(local_state, data, &mut self.memory);
	}

  if !skip_cycle {
    self.increment_timestamp_after_cycle();
	}
	self.local_state.monotonic_cycle_counter += 1;
  
}
```

`cycle`函数模拟了虚拟机的一个周期，处理一条指令并更新机器状态。它包括内存操作、操作数选择、执行前后的钩子，以及对特殊情况（如NOP指令和指针元数据）的处理。

