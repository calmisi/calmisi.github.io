---
title: kvm对guest中MSR寄存器的处理（四）-- wrmsr
date: 2019-01-25 22:19:35
tags:
- MSR
categories:
- KVM
---

本文为kvm操作MSR寄存器的第四部分，主要介绍kvm trap guest中的wrmsr指令的处理过程。

当guest中因为执行wrmsr指令产生vm exit时，其exit reason为32，相应地最后会调用KVM中的handle_wrmsr()函数。
其函数实现如下：
```C
static int handle_wrmsr(struct kvm_vcpu *vcpu)                                                                                                              
{
        struct msr_data msr; 
        u32 ecx = vcpu->arch.regs[VCPU_REGS_RCX];
        u64 data = (vcpu->arch.regs[VCPU_REGS_RAX] & -1u) 
                | ((u64)(vcpu->arch.regs[VCPU_REGS_RDX] & -1u) << 32); 

        msr.data = data;
        msr.index = ecx; 
        msr.host_initiated = false;
        if (kvm_set_msr(vcpu, &msr) != 0) { 
                trace_kvm_msr_write_ex(ecx, data);
                kvm_inject_gp(vcpu, 0);
                return 1;
        }    

        trace_kvm_msr_write(ecx, data);
        return kvm_skip_emulated_instruction(vcpu);
}
```
<!-- more -->
该handler的处理过程基本上与handle_rdmsr一样，具体可以参考[kvm对guest中MSR寄存器的处理（二）][handle_rdmsr]。
通过vcpu-arch获取到guest中的ECX寄存器的值和要写入MSR的data后，将它们存入struct msr_data msr中。
然后调用kvm_set_msr(vcpu, &msr)进行wrmsr的模拟。

> 注意
> 这里和handle_rdmsr()有点不同，handle_msr直接调用vmx_get_msr()函数进行模拟，而handle_wrmsr()会先调用kvm_set_msr(),
> 这是因为在kvm_set_msr()中需要对和地址相关的MSR，进行canonical检测或canonical调整。对其他的MSR没有上面的处理。
> 然后调用kvm_x86_ops->set_msr(vcpu, msr)即vmx_set_msr()进行wrmsr的模拟。
> 即handle_wrmsr时，x86平台多了这一步验证。

## kvm_set_msr(struct kvm_vcpu * vcpu, struct msr_data *msr)
该函数对于和地址相关的MSR，进行canonical检测，或canonical调整。
对其他的MSR没有上面的处理。
最后调用kvm_x86_ops->set_msr(vcpu, msr)进行wrmsr的模拟。
```C
/*
 * Writes msr value into into the appropriate "register".
 * Returns 0 on success, non-0 otherwise.
 * Assumes vcpu_load() was already called.
 */
int kvm_set_msr(struct kvm_vcpu *vcpu, struct msr_data *msr)                                                                                                
{
        switch (msr->index) {
        case MSR_FS_BASE:
        case MSR_GS_BASE:
        case MSR_KERNEL_GS_BASE:
        case MSR_CSTAR:
        case MSR_LSTAR:
                if (is_noncanonical_address(msr->data, vcpu))
                        return 1;
                break;
        case MSR_IA32_SYSENTER_EIP:
        case MSR_IA32_SYSENTER_ESP:
                /*   
                 * IA32_SYSENTER_ESP and IA32_SYSENTER_EIP cause #GP if
                 * non-canonical address is written on Intel but not on
                 * AMD (which ignores the top 32-bits, because it does
                 * not implement 64-bit SYSENTER).
                 *
                 * 64-bit code should hence be able to write a non-canonical
                 * value on AMD.  Making the address canonical ensures that
                 * vmentry does not fail on Intel after writing a non-canonical
                 * value, and that something deterministic happens if the guest
                 * invokes 64-bit SYSENTER.
                 */
                msr->data = get_canonical(msr->data, vcpu_virt_addr_bits(vcpu));
        }    
        return kvm_x86_ops->set_msr(vcpu, msr);
}
EXPORT_SYMBOL_GPL(kvm_set_msr);
```

#vmx_set_msr()
intel的CPU对kvm_x86_ops->set_msr()函数的实现为vmx_set_msr()













[handle_rdmsr]: kvm对guest中MSR寄存器的处理（二）-rdmsr.html