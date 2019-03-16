---
title: kvm对MSR寄存器的处理（三）
date: 2019-01-23 20:53:24
tags:
- MSR
categories:
- KVM
---

# msrs_to_save[], emulated_msrs[], msr_based_features[]
在x86.c文件中定一个了msrs_to_save[], emulated_msrs[], msr_based_features[]三个保存MSR寄存器index的数组，
以及num_msrs_to_save, num_emulated_msrs, num_msr_based_features三个静态全局变量来表示以上三个数组的大小。
> **其中msrs_to_save[]和emulated_msrs[]数组合起来为暴露给userspace的MSRs: userspace(QEMU)通过KVM_GET_MSR_INDEX_LIST ioctl获取，并且之后就可以调用KVM_GET_MSRS, KVM_SET_MSRS对这些MSRs进行读写**
> **msr_based_features[]数组向userspace传递KVM支持的feature-based MSRs.**

> 注意
> **msr_based_features[]数组和前面2个数组不是互斥的。
> 即msr_based_features[]数组中的MSR可以同时属于前面两个数组，但msr_to_save[]和emulated_msrs[]是互斥的。**

<!-- more -->

根据KVM的注释，msr_to_save[]数组，会在kvm.ko模块加载的时候，根据实际物理cpu的capabilities来调整；
emulated_msrs[]数组的调整依赖于虚拟化的实现，而不是物理CPU feature,所以把它们放在两个数组里面，我们可以通过下一小节的描述看出差别；
msr_based_features[]数组存储msr-based features相关的MSR，hypervisor使用这些MSR来查询相关的CPU features。

> 注意：
> **CPU features enumeration一般是通过CPUID，即通过CPUID指令查询当前的物理CPU是否有某些feature。
> 而MSR则是用来控制某个feature的开关。
> 一般地，先通过CPUID指令具体的一个bit位来查询某个feature是否存在，如果存在，则set/clear MSR的bit来enable/disable这个feature。
> 但是有一部分feature是通过MSR来enumerated的，为MSR-specific features,比如VMX相关的feature, MSR, IA32_ARCH_CAPABILITIES等。
> 即通过MSR的bit来表示CPU是否有某些features。**

```c
/*
 * List of msr numbers which we expose to userspace through KVM_GET_MSRS
 * and KVM_SET_MSRS, and KVM_GET_MSR_INDEX_LIST.
 *
 * This list is modified at module load time to reflect the
 * capabilities of the host cpu. This capabilities test skips MSRs that are
 * kvm-specific. Those are put in emulated_msrs; filtering of emulated_msrs
 * may depend on host virtualization features rather than host cpu features.
 */

static u32 msrs_to_save[] = {
        MSR_IA32_SYSENTER_CS, MSR_IA32_SYSENTER_ESP, MSR_IA32_SYSENTER_EIP,
        MSR_STAR,
#ifdef CONFIG_X86_64
        MSR_CSTAR, MSR_KERNEL_GS_BASE, MSR_SYSCALL_MASK, MSR_LSTAR,
#endif
        MSR_IA32_TSC, MSR_IA32_CR_PAT, MSR_VM_HSAVE_PA,
        MSR_IA32_FEATURE_CONTROL, MSR_IA32_BNDCFGS, MSR_TSC_AUX,
        MSR_IA32_SPEC_CTRL, MSR_IA32_ARCH_CAPABILITIES,
        MSR_IA32_RTIT_CTL, MSR_IA32_RTIT_STATUS, MSR_IA32_RTIT_CR3_MATCH,
        MSR_IA32_RTIT_OUTPUT_BASE, MSR_IA32_RTIT_OUTPUT_MASK,
        MSR_IA32_RTIT_ADDR0_A, MSR_IA32_RTIT_ADDR0_B,
        MSR_IA32_RTIT_ADDR1_A, MSR_IA32_RTIT_ADDR1_B,
        MSR_IA32_RTIT_ADDR2_A, MSR_IA32_RTIT_ADDR2_B,
        MSR_IA32_RTIT_ADDR3_A, MSR_IA32_RTIT_ADDR3_B,
};

static unsigned num_msrs_to_save;

static u32 emulated_msrs[] = {
        MSR_KVM_SYSTEM_TIME, MSR_KVM_WALL_CLOCK,
        MSR_KVM_SYSTEM_TIME_NEW, MSR_KVM_WALL_CLOCK_NEW,
        HV_X64_MSR_GUEST_OS_ID, HV_X64_MSR_HYPERCALL,
        HV_X64_MSR_TIME_REF_COUNT, HV_X64_MSR_REFERENCE_TSC,
        HV_X64_MSR_TSC_FREQUENCY, HV_X64_MSR_APIC_FREQUENCY,
        HV_X64_MSR_CRASH_P0, HV_X64_MSR_CRASH_P1, HV_X64_MSR_CRASH_P2,
        HV_X64_MSR_CRASH_P3, HV_X64_MSR_CRASH_P4, HV_X64_MSR_CRASH_CTL,
        HV_X64_MSR_RESET,
        HV_X64_MSR_VP_INDEX,
        HV_X64_MSR_VP_RUNTIME,
        HV_X64_MSR_SCONTROL,
        HV_X64_MSR_STIMER0_CONFIG,
        HV_X64_MSR_VP_ASSIST_PAGE,
        HV_X64_MSR_REENLIGHTENMENT_CONTROL, HV_X64_MSR_TSC_EMULATION_CONTROL,
        HV_X64_MSR_TSC_EMULATION_STATUS,

        MSR_KVM_ASYNC_PF_EN, MSR_KVM_STEAL_TIME,
        MSR_KVM_PV_EOI_EN,

        MSR_IA32_TSC_ADJUST,
        MSR_IA32_TSCDEADLINE,
        MSR_IA32_MISC_ENABLE,
        MSR_IA32_MCG_STATUS,
        MSR_IA32_MCG_CTL,
        MSR_IA32_MCG_EXT_CTL,
        MSR_IA32_SMBASE,
        MSR_SMI_COUNT,
        MSR_PLATFORM_INFO,
        MSR_MISC_FEATURES_ENABLES,
        MSR_AMD64_VIRT_SPEC_CTRL,
};

static unsigned num_emulated_msrs;

/*
 * List of msr numbers which are used to expose MSR-based features that
 * can be used by a hypervisor to validate requested CPU features.
 */
static u32 msr_based_features[] = {
        MSR_IA32_VMX_BASIC,
        MSR_IA32_VMX_TRUE_PINBASED_CTLS,
        MSR_IA32_VMX_PINBASED_CTLS,
        MSR_IA32_VMX_TRUE_PROCBASED_CTLS,
        MSR_IA32_VMX_PROCBASED_CTLS,
        MSR_IA32_VMX_TRUE_EXIT_CTLS,
        MSR_IA32_VMX_EXIT_CTLS,
        MSR_IA32_VMX_TRUE_ENTRY_CTLS,
        MSR_IA32_VMX_ENTRY_CTLS,
        MSR_IA32_VMX_MISC,
        MSR_IA32_VMX_CR0_FIXED0,
        MSR_IA32_VMX_CR0_FIXED1,
        MSR_IA32_VMX_CR4_FIXED0,
        MSR_IA32_VMX_CR4_FIXED1,
        MSR_IA32_VMX_VMCS_ENUM,
        MSR_IA32_VMX_PROCBASED_CTLS2,
        MSR_IA32_VMX_EPT_VPID_CAP,
        MSR_IA32_VMX_VMFUNC,

        MSR_F10H_DECFG,
        MSR_IA32_UCODE_REV,
        MSR_IA32_ARCH_CAPABILITIES,
};

static unsigned int num_msr_based_features;
```
# 调整msrs_to_save[], emulated_msrs[], msr_based_features[]
在加载kvm.ko模块的时候，调用kvm_init() => kvm_arch_hardware_setup() => kvm_init_msr_list().
在kvm_init_msr_list()函数中,对上面提到的三个数组进行调整。
1. msr_to_save[]:首先调用rdmsr_safe()去验证MSR是否在物理CPU上支持，然后对部分MSR调用具体的虚拟化的kvm_x86_ops来查询对应的MSR是否应该被支持。通过以上两步筛选后，用num_msrs_to_save变量保存过滤后的数组大小。
2. emulated_msrs[]:统一调用kvm_x86_ops->has_emulated_msr()函数来过滤每一个MSR，用num_emulated_msrs来保存过滤后的大小;**由于是kvm-specific MSR，与hardware无关，基本上这些MSR都支持。**
3. msr_based_features[]:使用kvm_get_msr_feature()来过滤，用num_msr_based_features来保存过滤后的大小。
```c
static void kvm_init_msr_list(void)
{
        u32 dummy[2];
        unsigned i, j;

        for (i = j = 0; i < ARRAY_SIZE(msrs_to_save); i++) {
                if (rdmsr_safe(msrs_to_save[i], &dummy[0], &dummy[1]) < 0)
                        continue;

                /*
                 * Even MSRs that are valid in the host may not be exposed
                 * to the guests in some cases.
                 */
                switch (msrs_to_save[i]) {
                case MSR_IA32_BNDCFGS:
                        if (!kvm_mpx_supported())
                                continue;
                        break;
                case MSR_TSC_AUX:
                        if (!kvm_x86_ops->rdtscp_supported())
                                continue;
                        break;
                case MSR_IA32_RTIT_CTL:
                case MSR_IA32_RTIT_STATUS:
                        if (!kvm_x86_ops->pt_supported())
                                continue;
                        break;
                case MSR_IA32_RTIT_CR3_MATCH:
                        if (!kvm_x86_ops->pt_supported() ||
                            !intel_pt_validate_hw_cap(PT_CAP_cr3_filtering))
                                continue;
                        break;
                case MSR_IA32_RTIT_OUTPUT_BASE:
                case MSR_IA32_RTIT_OUTPUT_MASK:
                        if (!kvm_x86_ops->pt_supported() ||
                                (!intel_pt_validate_hw_cap(PT_CAP_topa_output) &&
                                 !intel_pt_validate_hw_cap(PT_CAP_single_range_output)))
                                continue;
                        break;
                case MSR_IA32_RTIT_ADDR0_A ... MSR_IA32_RTIT_ADDR3_B: {
                        if (!kvm_x86_ops->pt_supported() ||
                                msrs_to_save[i] - MSR_IA32_RTIT_ADDR0_A >=
                                intel_pt_validate_hw_cap(PT_CAP_num_address_ranges) * 2)
                                continue;
                        break;
                }
                default:
                        break;
                }

                if (j < i)
                        msrs_to_save[j] = msrs_to_save[i];
                j++;
        }
        num_msrs_to_save = j;

        for (i = j = 0; i < ARRAY_SIZE(emulated_msrs); i++) {
                if (!kvm_x86_ops->has_emulated_msr(emulated_msrs[i]))
                        continue;

                if (j < i)
                        emulated_msrs[j] = emulated_msrs[i];
                j++;
        }
        num_emulated_msrs = j;

        for (i = j = 0; i < ARRAY_SIZE(msr_based_features); i++) {
                struct kvm_msr_entry msr;

                msr.index = msr_based_features[i];
                if (kvm_get_msr_feature(&msr))
                        continue;

                if (j < i)
                        msr_based_features[j] = msr_based_features[i];
                j++;
        }
        num_msr_based_features = j;
}
```