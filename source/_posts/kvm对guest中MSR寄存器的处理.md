---
title: kvm对guest中MSR寄存器的处理
date: 2019-01-18 00:07:49
tags:
- kvm
- msr
categories: kvm
---

本文介绍kvm中对MSRs相关的处理，包括MSR bitmap, rdmsr和wrmsr相关的trap和处理等。

# MSR bitmap
如果处理器支持 1-setting of the "Use MSR bitmaps" VM-execution control, 那么VM-execution control fields会包含4个连续的MSR bitmaps的64位的物理地址，每个连续的MSR bitmaps为1 KByte的大小。
如果处理器不支持1-setting of the "Use MSR bitmaps",那么不会存在这4个连续的区域。
- Read bitmap for low MSRs(locate at the MSR-bitmap address). This contains one bit for each MSR address in the range 0000 0000H to 0000 1FFFH. The bit determines whether an execution of RDMSR applied to that MSR causes a VM exit.
即从 [MSR-bitmap address ~ MSR-bitmap address+1023] 共1KB(8192 bits)的大小，每个bit对应0000 0000H - 0000 1FFFH中的一个MSR。如果对应的bit为1，那么RDMSR相应地址的MSR则会VM exit.
- Read bitmap for high MSRs (located at the MSR-bitmap address plus 1024). This contains one bit for each MSR address in the range C000 0000H to C000 1FFFH. The bit determines whether an execution of RDMSR applied to that MSR causes a VM exit.
即从 [MSR-bitmap address+1024 ~ MSR-bitmap address+2047] 共1KB(8192 bits)的大小，每个bit对应C000 0000H - C000 1FFFH中的一个MSR。如果对应的bit为1，那么RDMSR相应地址的MSR则会VM exit.
- Write bitmap for low MSRs (located at the MSR-bitmap address plus 2048). 同理控制地址范围0000 0000H - 0000 1FFFH的MSR的WRMSR是否会VM exit.
- Write bitmap for high MSRs (located at the MSR-bitmap address plus 3072). 同理控制地址范围C000 0000H - C000 1FFFH的MSR的WRMSR是否会VM exit.


# rdmsr & wrmsr 指令
在`SDM.Vol3.25.1.3 Instructions That Cause VM Exits Conditionally`中关于rdmsr和wrmsr指令，是这样说的：
RDMSR. The RDMSR instruction causes a VM exit if any of the following are true: 
- The "Use MSR bitmaps" VM-execution control is 0.
- The value of ECX is not in the ranges 0000 0000H - 0000 1FFFH and C000 0000H - C000 1FFFH.
- The value of ECX is in the ranges 0000 0000H - 0000 1FFFH and bit n in read bitmap for low MSRs is 1, where n is the value of ECX.
- The value of ECX is in the ranges C000 0000H - C000 1FFFH and bit n in read bitmap for high MSRs is 1, where n is the value of ECX & 0000 1FFFH.

WRMSR同上可以类推。

---
当guest在运行时，如果不产生VM EXIT，即运行VM non-root mdoe的时候。如果RDMSR/WRMSR不会产生VM exit,那么会直接对物理的MSR进行读写，
如果产生了VM EXIT，那么会VM EXIT，到VM root mode，根据KVM code来决定如何处理，下面我们来看kvm code是如何实现的。
//应该放到总结去。

# KVM如何处理rdmsr VM exit.

# KVM中MSR相关的处理。
1. kvm_init() -> kvm_arch_init();
2. kvm_init() -> kvm_arch_hardware_setup(); 

## kvm_arch_init() 
在x86.c中的
```
static struct kvm_shared_msrs __percpu * shared_msrs;
...

int kvm_arch_init(void *opaque){
...
shared_msrs = alloc_percpu(struct kvm_shared_msrs);
...
}
```
为每个CPU分配了一个percpu变量，为kvm_shared_msrs结构体：
```
#define KVM_NR_SHARED_MSRS 16

struct kvm_shared_msrs {
	struct user_return_notifier run;
	bool registered;
	struct kvm_shared_msr_values {
		u64 host;
		u64 curr;
	} values[KVM_NR_SHARED_MSRS];		//values[16]数组的大小为16，最多可以存16个msr寄存器的值
}; 
```

## kvm_arch_hardware_setup()
1. kvm_x86_ops->hardware_setup();
```static __init int hardware_setup(void)
{
	...
	rdmsrl_safe(MSR_EFER, &host_efer);
	
	for (i = 0; i < ARRAY_SIZE(vmx_msr_index); ++i)
		kvm_define_shared_msr(i, vmx_msr_index[i]);
	
	if(setup_vmcs_config(&vmcs_config, &vmx_capability) < 0)
		return -EIO;

	if (boot_cpu_has(X86_FEATURE_NX))
		kvm_enable_efer_bits(EFER_NX);	//将efer_reserved_bits中除了X86_FEATURE_NX外的所有bit都置上
	...
}

其中kvm_enable_efer_bits()为：
void kvm_enable_efer_bits(u64 mask)
{
	efer_reserved_bits &= ~mask;
}
```
	
2. kvm_init_msr_list();
