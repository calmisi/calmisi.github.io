---
title: kvm指令模拟-kvm_emulate_instruction
date: 2019-07-06 19:36:53
tags:
- KVM
- Instruction emulation
categories:
- KVM
---

kvm_emualte_instruction(struct kvm_vcpu *vcpu, int emulation_type)
根据当前vcpu的状态和指定的emulation_type来进行当前vcpu所执行的指令进行模拟。

<!-- more -->

# Emulation type & Emulation result
```C
enum emulation_result {
	EMULATE_DONE,		/* no further processing */
	EMULATE_USER_EXIT,	/* kvm_run ready for userspace exit */
	EMULATE_FAIL,		/* can't emulate this instruction */
};

#define EMULTYPE_NO_DECODE		(1 << 0)
#define EMULTYPE_TRAP_NO		(1 << 1)
#define EMULTYPE_SKIP			(1 << 2)
#define EMULTYPE_ALLOW_RETRY	(1 << 3)
#define EMULTYPE_NO_UD_ON_FAIL	(1 << 4)
#define EMULTYPE_VMWARE			(1 << 5)
```

# x86_emulate_instruction
`kvm_emulate_instruction()`函数实际是对`x86_emulate_instruction()`函数的封装：
```C
int kvm_emulate_instruction(struct kvm_vcpu *vcpu, int emulate_type)
{
	return x86_emulate_instruction(vcpu, 0, emulation_type, NULL, 0);
}
```

```C
int x86_emulate_instruction(struct kvm_vcpu *vcpu,
							unsigned long cr2,
							int emulation_type,
							void *insn,
							int insn_len)
{
	...
}
```
该函数有5个参数：
1. vcpu：需要进行指令模拟的vcpu,
2. cr2: 当前vcpu的cr2;
3. emulation_type: 模拟的类型；
4. insn: 需要模拟的指令的指针；
5. insn_len:需要模拟的指令的长度；

该函数在kvm中有三个地方会被调用：
1. in arch/x86/kvm/mmu.c-> kvm_mmu_page_fault()->x86_emulate_instruction(vcpu, cr2, emulation_type, insn, insn_len);
2. in arch/x86/kvm/x86.c-> kvm_emulate_instruction()->x86_emulate_instruction(vcpu, 0, emulation_type, NULL, 0);
3. in arch/x86/kvm/x86.c-> kvm_emulate_instruction_from_buffer()->x86_emulate_instruction(vcpu, 0, 0, insn, insn_len);

我们重点关注case 2， ji`kvm_emulate_instruction()`

# kvm_emulate_instruction
其调用`x86_emulate_instruction()`传入的cr2为0，insn为NULL,insn_len为0，即只有`vcpu`,`emulation_type`有效。

# x86_emulate_instruction
```C
int x86_emulate_instruction(struct kvm_vcpu *vcpu,                                  
                            unsigned long cr2,                                      
                            int emulation_type,                                     
                            void *insn,                                             
                            int insn_len)                                           
{                                                                                   
        int r;                                                                      
        struct x86_emulate_ctxt *ctxt = &vcpu->arch.emulate_ctxt;                   
        bool writeback = true;                                                      
        bool write_fault_to_spt = vcpu->arch.write_fault_to_shadow_pgtable;         
                                                                                    
        vcpu->arch.l1tf_flush_l1d = true;                                           
                                                                                    
        /*                                                                          
         * Clear write_fault_to_shadow_pgtable here to ensure it is                 
         * never reused.                                                            
         */                                                                         
        vcpu->arch.write_fault_to_shadow_pgtable = false;                           
        kvm_clear_exception_queue(vcpu);                                            
                                                                                    
        if (!(emulation_type & EMULTYPE_NO_DECODE)) {                               
                init_emulate_ctxt(vcpu);                                            
                                                                                    
                /*                                                                  
                 * We will reenter on the same instruction since                    
                 * we do not set complete_userspace_io.  This does not              
                 * handle watchpoints yet, those would be handled in                
                 * the emulate_ops.                                                 
                 */                                                                 
                if (!(emulation_type & EMULTYPE_SKIP) &&                            
                    kvm_vcpu_check_breakpoint(vcpu, &r))                            
                        return r;                                                   
                                                                                    
                ctxt->interruptibility = 0;                                         
                ctxt->have_exception = false;                                       
                ctxt->exception.vector = -1;                                        
                ctxt->perm_ok = false;                                              
                                                                                    
                ctxt->ud = emulation_type & EMULTYPE_TRAP_UD;                       
                                                                                    
                r = x86_decode_insn(ctxt, insn, insn_len);                          
                                                                                    
                trace_kvm_emulate_insn_start(vcpu);                                 
                ++vcpu->stat.insn_emulation;                                        
                if (r != EMULATION_OK)  {                                           
                        if (emulation_type & EMULTYPE_TRAP_UD)                      
                                return EMULATE_FAIL;                                
                        if (reexecute_instruction(vcpu, cr2, write_fault_to_spt,
                                                emulation_type))                    
                                return EMULATE_DONE;                                
                        if (ctxt->have_exception && inject_emulated_exception(vcpu))
                                return EMULATE_DONE;                                
                        if (emulation_type & EMULTYPE_SKIP)                         
                                return EMULATE_FAIL;                                
                        return handle_emulation_failure(vcpu, emulation_type);  
                }                                                                   
        }                                                                           
                                                                                    
        if ((emulation_type & EMULTYPE_VMWARE) &&                                   
            !is_vmware_backdoor_opcode(ctxt))                                       
                return EMULATE_FAIL;                                                
                                                                                    
        if (emulation_type & EMULTYPE_SKIP) {                                       
                kvm_rip_write(vcpu, ctxt->_eip);                                    
                if (ctxt->eflags & X86_EFLAGS_RF)                                   
                        kvm_set_rflags(vcpu, ctxt->eflags & ~X86_EFLAGS_RF);    
                return EMULATE_DONE;                                                
        }                                                                           
                                                                                    
        if (retry_instruction(ctxt, cr2, emulation_type))                           
                return EMULATE_DONE;                                                
                                                                                    
        /* this is needed for vmware backdoor interface to work since it            
           changes registers values  during IO operation */                         
        if (vcpu->arch.emulate_regs_need_sync_from_vcpu) {                          
                vcpu->arch.emulate_regs_need_sync_from_vcpu = false;                
                emulator_invalidate_register_cache(ctxt);                           
        }                                                                           
                                                                                    
restart:                                                                            
        /* Save the faulting GPA (cr2) in the address field */                      
        ctxt->exception.address = cr2;                                              
                                                                                    
        r = x86_emulate_insn(ctxt);                                                 
                                                                                    
        if (r == EMULATION_INTERCEPTED)                                             
                return EMULATE_DONE;                                                
                                                                                    
        if (r == EMULATION_FAILED) {                                                
                if (reexecute_instruction(vcpu, cr2, write_fault_to_spt,            
                                        emulation_type))                            
                        return EMULATE_DONE;                                        
                                                                                    
                return handle_emulation_failure(vcpu, emulation_type);              
        }                                                                           
                                                                                    
        if (ctxt->have_exception) {                                                 
                r = EMULATE_DONE;                                                   
                if (inject_emulated_exception(vcpu))                                
                        return r;                                                   
        } else if (vcpu->arch.pio.count) {                                          
                if (!vcpu->arch.pio.in) {                                           
                        /* FIXME: return into emulator if single-stepping.  */  
                        vcpu->arch.pio.count = 0;                                   
                } else {                                                            
                        writeback = false;                                          
                        vcpu->arch.complete_userspace_io = complete_emulated_pio;
                }                                                                   
                r = EMULATE_USER_EXIT;                                              
        } else if (vcpu->mmio_needed) {                                             
                if (!vcpu->mmio_is_write)                                           
                        writeback = false;                                          
                r = EMULATE_USER_EXIT;                                              
                vcpu->arch.complete_userspace_io = complete_emulated_mmio;          
        } else if (r == EMULATION_RESTART)                                          
                goto restart;                                                       
        else                                                                        
                r = EMULATE_DONE;                                                   
                                                                                    
        if (writeback) {                                                            
                unsigned long rflags = kvm_x86_ops->get_rflags(vcpu);               
                toggle_interruptibility(vcpu, ctxt->interruptibility);              
                vcpu->arch.emulate_regs_need_sync_to_vcpu = false;                  
                kvm_rip_write(vcpu, ctxt->eip);                                     
                if (r == EMULATE_DONE && ctxt->tf)                                  
                        kvm_vcpu_do_singlestep(vcpu, &r);                           
                if (!ctxt->have_exception ||                                        
                    exception_type(ctxt->exception.vector) == EXCPT_TRAP)           
                        __kvm_set_rflags(vcpu, ctxt->eflags);                       
                                                                                    
                /*                                                                  
                 * For STI, interrupts are shadowed; so KVM_REQ_EVENT will          
                 * do nothing, and it will be requested again as soon as            
                 * the shadow expires.  But we still need to check here,            
                 * because POPF has no interrupt shadow.                            
                 */                                                                 
                if (unlikely((ctxt->eflags & ~rflags) & X86_EFLAGS_IF))             
                        kvm_make_request(KVM_REQ_EVENT, vcpu);                      
        } else                                                                      
                vcpu->arch.emulate_regs_need_sync_to_vcpu = true;                   
                                                                                    
        return r;                                                                   
}
```

1. 首先判断是否需要decode instruction.即如果没有置上EMULTYPE_NO_DECODE,那么就需要指令译码。
	- init_emulate_ctxt(vcpu)： 初始化vcpu->arch.tmulate_ctxt；
		- 初始化ctxt的一些变量
			- 获取当前vcpu的CS段描述符的DB位（bit22） 和L位（bit21）
			- ctxt->eflag
			- ctxt->tf;
			- ctxt->rip;
			- ctxt->mode;可选的值有
				- X86EMUL_MODE_REAL
				- X86EMUL_MODE_VM86
				- X86EMUL_MODE_PROT64
				- X86EMUL_MODE_PROT32
				- X86EMUL_MODE_PROT16
		- init_decode_cache(ctxt);
			- memset(&ctxt->rip_relative, 0 , (void *)&ctxt->modrm - (void *)&ctxt->rip_relative);
			- ctxt->io_read.pos = 0;
			- ctxt->io_read.end = 0;
			- ctxt->mem_read.end = 0;
		- vcpu->arch.emulate_regs_need_sync_from_vcpu = false;
	- 如果当前模拟没有设置EMULTYPE_SKIP并且当前指令设置了breakpoint，那么直接返回，因为还没有set_complete_userspace_io,所以还会再进入到当前指令。
	- 继续初始化ctxt的一些域：
		- ctxt->interruptibility = 0;
		- ctxt->have_exception = false;
		- ctxt->exception.vector = -l;
		- ctxt->perm_ok = false;
		- ctxt->ud = emulation_type & EMULTYPE_TRAP_UD;
	- x86_decode_insn(ctxt, insn, insn_len)进行指令decode;
	  在此处会有tracepoint来trace kvm_emulate_insn(failed=0),并且将vcpu->stat.insn_emulation加1；
	  如果返回值为EMULATION_OK,说明指令译码过程没有出问题。
2. 如果emulation_type置上了EMULTYPE_VMWARE,但是该指令不是vmware_backdoor_opcode，那么return EMULATE_FAIL；
3. 如果置上了EMULTYPE_SKIP，那么更新vcpu_rip和EFLAGS_RF；
4. 




## x86_decode_insn
```C
int x86_decode_insn(struct x86_emulate_ctxt *ctxt, void *insn, int insn_len)
```
初始状态：
- op_prefix = false;
- has_seg_override = false;
- ctxt->memop.type = OP_NONE;
- ctxt->memopp = NULL;
- ctxt->_eip = ctxt->eip;
- ctxt->fetch.ptr = ctxt->fetch.data;
- ctxt->fetch.end = ctxt->fetch.data + insn_len; 如果传入的insn为NULL，insn_len为0，那么ctxt->fetch.ptr == ctxt->fetch.end;
- ctxt->opcode.len = 1;
 
1. 如果传入的insn != NULL, 那么将insn地址开始的insn_len长度的指令memcpy到ctxt->fetch.data；
   如果传入的insn == NULL, 那么调用`__do_insn_fetch_bytes(ctxt,1)`。该函数会设置ctxt->fetch.end;

2. 根据ctxt->mode来设置def_op_bytes和def_ad_bytes的长度，最终会赋值到
- ctxt->op_bytes = def_op_bytes; 操作数长度
- ctxt->ad_bytes = def_ad_bytes; 地址长度

3. for()循环进行指令前缀处理，ctxt->b = insn_fetch(u8, ctxt), 取出一个字节，根据不同的值进行不同的处理，比如有前缀，段超越，address-size override等等。
- 设置op_prefix,更新ctxt->op_bytes;				0x66： operand-size override;
- 更新ctxt->ad_bytes;							0x67: address-size override;
- 设置has_seg_override, ctxt->seg_override;		0x26,0x2e,0x36,0x3e,0x64,0x65: Segment override;
- 设置ctxt->rex_prefix;							0x40...0x4f: /*REX*/
- 设置ctxt->lock_prefix;							0xf0: lock;
- 设置ctxt->rep_prefix;							0xf2, 0xf3: REP

4. 上一步处理完指令前缀后，ctxt->b指向前缀后的指令的第一个字节，
然后根据ctxt->b的值，来检查是几字节指令，将指令译码出入opcode,并设置ctxt->opcode_len= 1/2/3;

5. retry_instruction(),很多情况不执行，具体还不清楚这一步是在什么场景需要。

6. restart开始指令执行的模拟：
	- ctxt->exception.address = cr2;
	- x86_emulate_insn(ctxt)进行指令执行模拟