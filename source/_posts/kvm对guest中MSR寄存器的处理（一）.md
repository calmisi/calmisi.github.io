---
title: kvm中MSR的处理(1) -- percpu变量shared_msrs
date: 2019-01-18 00:07:49
tags:
- MSR
categories: KVM
---

本文介绍KVM中percpu变量shared_msrs，以及其相关的一些的数据结构。
其最终是为了在VM entry和VM exit的时候切换host和guest之间几个重要的MSR的值。

<!-- more -->
 
# 数据结构1: vmx_msr_index[]数组
`vmx_msr_index[]`数组定义需要percpu变量中需要维护哪些MSR，这些MSR在host和guest中是独立的值。
## 1. 定义
```C
/*
 * Though SYSCALL is only supported in 64-bit mode on Intel CPUs, kvm
 * will emulate SYSCALL in legacy mode if the vendor string in guest
 * CPUID.0:{EBX,ECX,EDX} is "AuthenticAMD" or "AMDisbetter!" To
 * support this emulation, IA32_STAR must always be included in
 * vmx_msr_index[], even in i386 builds.
 */
const u32 vmx_msr_index[] = {                                                                                                             
#ifdef CONFIG_X86_64
        MSR_SYSCALL_MASK, MSR_LSTAR, MSR_CSTAR,
#endif
        MSR_EFER, MSR_TSC_AUX, MSR_STAR,
};
```

---

# 数据结构2: kvm_shared_msrs，__percpu变量 
## 1. 定义
```C
#define KVM_NR_SHARED_MSRS 16

struct kvm_shared_msrs {
	struct user_return_notifier run;
	bool registered;
	struct kvm_shared_msr_values {
		u64 host;
		u64 curr;
	} values[KVM_NR_SHARED_MSRS];		//values[16]数组的大小为16，最多可以存16个msr寄存器的值
}; 

static struct kvm_shared_msrs __percpu * shared_msrs;
```

## 2. 创建
在x86.c中的kvm_arch_init()函数中，为每个CPU分配了一个percpu变量shared_msrs，为kvm_shared_msrs结构体
```C
int kvm_arch_init(void *opaque){
	...
	shared_msrs = alloc_percpu(struct kvm_shared_msrs);
	...
}
```

## 3. 初始化

---

# 数据结构3-- shared_msrs_global
## 1. 定义
在x86.c中，定义了一个结构体struct kvm_shared_msrs_global和其一个static实例shared_msrs_global;
```C
#define KVM_NR_SHARED_MSRS	16
struct kvm_shared_msrs_global {
	int nr;
	u32 msrs[KVM_NR_SHARED_MSRS];
}

static struct kvm_shared_msrs_global __read_mostly shared_msrs_global;
```

## 2. 初始化
在hardware_setup()函数对`数据结构1 -- vmx_msr_index[]`数组每一个成员调用kvm_define_shared_msr()函数对shared_msrs_global静态变量进行初始化，具体地：
1. 将vmx_msr_index[]数组中每一个成员MSR存入shared_msrs_global.msrs[]数组中，
2. 使用shared_msrs_global.nr记录shared_msrs_global.msrs[]数组的大小，即为vmx_msr_index[]数组的大小。

## 总结
shared_msrs_global为一个静态变量，其中所包含的MSR成员(shared_msrs_global.msrs[])与`数据结构1 -- vmx_msr_index[]`数组是一模一样的，并且顺序是一一对应的，只是shared_msrs_global中多了一个成员nr表示该数组的大小。

---

# 数据结构4 -- vmx->guest_msrs
## 1. 定义
`vmx->guest_msrs`为strcut vcpu_vmx结构体中定义的一个struct shared_msr_entry的指针。其实质为struct shared_msr_entry的数组。
```C
struct shared_msr_entry {
	unsigned index;
	u64 	data;
	u64	mask;
}

struct vcpu_vmx {
	...
	struct shared_msr_entry * guest_msrs;
	int	nmsrs;
	int 	save_nmsrs;
	bool 	guest_msrs_dirty;
	...
}
```

## 2. 创建
在`vmx_create_vcpu()`函数中，vmx->guest_msrs = kmalloc(PAGE_SIZE, GFP_KERNRL)为其分配了一个4KB的page页。

## 3. 初始化
在vmx.c中的`vmx_vcpu_setup()`函数中，初始化vmx->guest_msrs[]数组，
```C
static void vmx_vcpu_setup(struct vcpu_vmx *vmx)
{
	...
	for (i = 0; i < ARRAY_SIZE(vmx_msr_index); ++i) {
	    u32 index = vmx_msr_index[i];
	    u32 data_low, data_high;
	    int j = vmx->nmsrs;
	
	    if (rdmsr_safe(index, &data_low, &data_high) < 0)
	            continue;
	    if (wrmsr_safe(index, data_low, data_high) < 0)
	            continue;
	    vmx->guest_msrs[j].index = i;
	    vmx->guest_msrs[j].data = 0;
	    vmx->guest_msrs[j].mask = -1ull;
	    ++vmx->nmsrs;
	}
	...
}
```
 初始时，vmx->nmsrs=0，然后根据`数据结构1 -- vmx_msr_index[]`数组定义的不同的MSR，来填充vmx->guest_msrs[]结构：
 具体地，使用rdmsr_safe()和wrmsr_safe()验证在实际的物理平台上可以读取对应的MSR，然后将其加入vmx->guest_msrs[]数组。
 > **注意
 > 存入vmx->guest_msrs[]中每一个元素的index不是MSR的index,而是该MSR在vmx_msr_index[]数组中的序号；data的初始值为0；mask的初始值为全1；**
 
 最后使用vmx->nmsrs记录vmx->guest_msrs[]数组的大小。

## 4. 操作/使用
### find_msr_entry()
```C
struct shared_msr_entry *find_msr_entry(struct vcpu_vmx *vmx, u32 msr)
{
    int i;

    i = __find_msr_index(vmx, msr);
    if (i >= 0)
        return &vmx->guest_msrs[i];
    return NULL;
}
```
 该函数查询指定的msr是否在vmx->guest_msrs[]数组中有表示，如果有则返回对应的元素的指针，如果没有则返回NULL；
其中参数2：u32 msr，为要查找的MSR的 index编号值。
返回值： NULL，vmx->guest_msrs[]没有对应的MSR；！NULL， vmx->guest_msrs[]中MSR对应的元素的指针。

### setup_msrs()
```C
/*
 * Set up the vmcs to automatically save and restore system
 * msrs.  Don't touch the 64-bit msrs if the guest is in legacy
 * mode, as fiddling with msrs is very expensive.
 */
static void setup_msrs(struct vcpu_vmx *vmx)
{
    int save_nmsrs, index;

    save_nmsrs = 0;
#ifdef CONFIG_X86_64
    /*
     * The SYSCALL MSRs are only needed on long mode guests, and only
     * when EFER.SCE is set.
     */
    if (is_long_mode(&vmx->vcpu) && (vmx->vcpu.arch.efer & EFER_SCE)) {
        index = __find_msr_index(vmx, MSR_STAR);
        if (index >= 0)
            move_msr_up(vmx, index, save_nmsrs++);
        index = __find_msr_index(vmx, MSR_LSTAR);
        if (index >= 0)
            move_msr_up(vmx, index, save_nmsrs++);
        index = __find_msr_index(vmx, MSR_SYSCALL_MASK);
        if (index >= 0)
            move_msr_up(vmx, index, save_nmsrs++);
    }
#endif
    index = __find_msr_index(vmx, MSR_EFER);
    if (index >= 0 && update_transition_efer(vmx, index))
        move_msr_up(vmx, index, save_nmsrs++);
    index = __find_msr_index(vmx, MSR_TSC_AUX);
    if (index >= 0 && guest_cpuid_has(&vmx->vcpu, X86_FEATURE_RDTSCP))
        move_msr_up(vmx, index, save_nmsrs++);

    vmx->save_nmsrs = save_nmsrs;
    vmx->guest_msrs_dirty = true;

    if (cpu_has_vmx_msr_bitmap())
        vmx_update_msr_bitmap(&vmx->vcpu);
}
```
 该函数setup vmx->guest_msrs[]数组，当更新了vmx->guest_msrs[]数组中元素的data,mask之后，会调用该函数来重新setup，
其会按照MSR_STAR, MSR_LSTAR, MSR_SYSCALL_MASK, MSR_EFER, MSR_TSC_AUX的顺序进行处理，
**注意 该顺序和初始化时使用的vmx_msr_index[]数组中顺序不一样**
具体地，每一步处理首先会判断是否支持该MSR，然后判断vmx->guest_msrs[]数组中是否包含该MSR，如果包含则将MSR从vmx->guest_msrs[]中向前移，最后处理完后，vmx->guest_msrs[]数组中的MSR如果被支持，肯定是和前面的顺序是一致的。
并且，vmx->save_nmsrs保存该setup_msrs()函数处理的其中的几个MSR，即setup的有效的MSR的个数，以及将vmx->guest_msrs_dirty=true;表示调用过setup_msrs()函数。
最后调用vmx_update_msr_bitmap()，其检查msr_bitmap_mode是否改变，如果改变则更新vmx->vmcs01.msr_bitmap。

### vmx_prepare_switch_to_guest()
```C
void vmx_prepare_switch_to_guest(struct kvm_vcpu * vcpu)
{
    ...
    /*
     * Note that guest MSRs to be saved/restored can also be changed
     * when guest state is loaded. This happens when guest transitions
     * to/from long-mode by setting MSR_EFER.LMA.
     */
    if (!vmx->loaded_cpu_state || vmx->guest_msrs_dirty) {
        vmx->guest_msrs_dirty = false;
        for (i = 0; i < vmx->save_nmsrs; ++i)
            kvm_set_shared_msr(vmx->guest_msrs[i].index,
                       vmx->guest_msrs[i].data,
                       vmx->guest_msrs[i].mask);

    }
    ....
}
```
 在该函数中的for循环中，根据上一个函数setup_msrs()设置的vmx->guest_msrs_dirty以及vmx->save_nmsrs来依次调用kvm_set_shared_msr()函数。

### kvm_set_shared_msr()
```C
int kvm_set_shared_msr(unsigned slot, u64 value, u64 mask)                                                                           
{
        unsigned int cpu = smp_processor_id();
        struct kvm_shared_msrs *smsr = per_cpu_ptr(shared_msrs, cpu);
        int err; 

        if (((value ^ smsr->values[slot].curr) & mask) == 0)
                return 0;
        smsr->values[slot].curr = value;
        err = wrmsrl_safe(shared_msrs_global.msrs[slot], value);
        if (err)
                return 1;

        if (!smsr->registered) {
                smsr->urn.on_user_return = kvm_on_user_return;
                user_return_notifier_register(&smsr->urn);
                smsr->registered = true;
        }    
        return 0;
}
EXPORT_SYMBOL_GPL(kvm_set_shared_msr);
```
具体地kvm_set_shared_msr()函数取出当前cpu的percpu变量shared_msrs。
并将shared_msrs->values[slot].curr值更新为新的值，并且把该值写到物理的MSR寄存器中。
并检查该cpu的shared_msrs是否registered，如果没有，则注册。
注意，其中注册的kvm_on_uesr_return()函数:
 1. 如果该cpu的shared_msrs->registered，则设置其为false,并则user_return_notifier_unregister()该struct user_return_notifier.
 2. 对shared_msrs中所有values[]，判断其host是不是==curr,如果不是的，那么将host的值写入物理的MSR寄存器，并更新curr的值为host。


