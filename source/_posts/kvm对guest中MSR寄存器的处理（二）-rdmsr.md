---
title: kvm对guest中MSR寄存器的处理（二）-- rdmsr
date: 2019-01-21 21:31:09
tags:
- MSR
categories: KVM
---

本文为kvm操作MSR寄存器的第二部分，介绍guest中rdmsr指令的处理过程。
之前在（一）中有介绍，如果对guest设置了`Use MSR bitmaps` VM-execution control bit,那么Read bitmap for low MSRs(0000 0000H - 0000 1FFFH)和Read bitmap for high MSRs(C000 0000H - C000 1FFFH)范围中的为0的bit对应的地址的MSR在执行rdmsr的时候不会产生VM EXIT。
其他情况，运行rdmsr均会触发VM exit,最后会进入vmx.c中的handle_rdmsr()函数。

<!-- more -->

# handle_rdmsr
如果guest vm exit时的exit reason为31号，即为rdmsr指令产生的vm exit。
具体到最后会调用handle_rdmsr()，其函数实现如下。
```C
static int handle_rdmsr(struct kvm_vcpu *vcpu)
{
        u32 ecx = vcpu->arch.regs[VCPU_REGS_RCX];                                                                                                           
        struct msr_data msr_info;

        msr_info.index = ecx; 
        msr_info.host_initiated = false;
        if (vmx_get_msr(vcpu, &msr_info)) {
                trace_kvm_msr_read_ex(ecx);
                kvm_inject_gp(vcpu, 0);
                return 1;
        }    

        trace_kvm_msr_read(ecx, msr_info.data);

        /* FIXME: handling of bits 32:63 of rax, rdx */
        vcpu->arch.regs[VCPU_REGS_RAX] = msr_info.data & -1u; 
        vcpu->arch.regs[VCPU_REGS_RDX] = (msr_info.data >> 32) & -1u; 
        return kvm_skip_emulated_instruction(vcpu);
}
```
1. 从vcpu->arch.regs[]数组里取出guest的RCX寄存器的时，其存放guest rdmsr指令需要读取的MSR的index.
2. 然后将需要读取的MSR的index存入msr_info，并设置msr_info.host_initiated = false;
3. 调用mvx_get_msr(vcpu, &msr_info)在kvm中对guest的rdmsr指令进行模拟；
4. 如果读取失败，则调用kvm_inject_gp()向guest插入一个#GP；
5. 如果读取成功，则将模拟rdmsr后的读到的数据data存入vcpu->arch.regs[RAX]和vpu->arch.regs[RDX]，然后更改guest_rip跳过这条指令
6. 当vm entry进入guest后，guest_rip已经指向rdmsr的下一条指令，并且rdmsr读取的值已经正常在GUEST_RAX和GUEST_RDX里面了；

> **注意其中trace_kvm_msr_read_ex(ecx)和trace_kvm_msr_read(ecx,msr_info.data)是给perf工具的trace用的。我们不讨论这个**。

# vmx_get_msr()
重点是vmx_get_msr()函数，其真正完成了KVM对guest里面的rdmsr指令的模拟。
如果其返回0,则表示对rdmsr的模拟是成功的，
其返回非0，则表示对rdmsr指令的模拟有问题，要向guest插入一个#GP。

```C
 * Reads an msr value (of 'msr_index') into 'pdata'.
 * Returns 0 on success, non-0 otherwise.
 * Assumes vcpu_load() was already called.
 */
static int vmx_get_msr(struct kvm_vcpu *vcpu, struct msr_data *msr_info)
{
        struct vcpu_vmx *vmx = to_vmx(vcpu);
        struct shared_msr_entry *msr;
        u32 index;

        switch (msr_info->index) {
#ifdef CONFIG_X86_64
        case MSR_FS_BASE:
                msr_info->data = vmcs_readl(GUEST_FS_BASE);
                break;
        case MSR_GS_BASE:
                msr_info->data = vmcs_readl(GUEST_GS_BASE);
                break;
        case MSR_KERNEL_GS_BASE:
                msr_info->data = vmx_read_guest_kernel_gs_base(vmx);
                break;
#endif
        case MSR_EFER:
                return kvm_get_msr_common(vcpu, msr_info);
        case MSR_IA32_SPEC_CTRL:
                if (!msr_info->host_initiated &&
                    !guest_cpuid_has(vcpu, X86_FEATURE_SPEC_CTRL))
                        return 1;

                msr_info->data = to_vmx(vcpu)->spec_ctrl;
                break;
        case MSR_IA32_ARCH_CAPABILITIES:
                if (!msr_info->host_initiated &&
                    !guest_cpuid_has(vcpu, X86_FEATURE_ARCH_CAPABILITIES))
                        return 1;
                msr_info->data = to_vmx(vcpu)->arch_capabilities;
                break;
        case MSR_IA32_SYSENTER_CS:
                msr_info->data = vmcs_read32(GUEST_SYSENTER_CS);
                break;
        case MSR_IA32_SYSENTER_EIP:
                msr_info->data = vmcs_readl(GUEST_SYSENTER_EIP);
                break;
        case MSR_IA32_SYSENTER_ESP:
                msr_info->data = vmcs_readl(GUEST_SYSENTER_ESP);
                break;
        case MSR_IA32_BNDCFGS:
                if (!kvm_mpx_supported() ||
                    (!msr_info->host_initiated &&
                     !guest_cpuid_has(vcpu, X86_FEATURE_MPX)))
                        return 1;
                msr_info->data = vmcs_read64(GUEST_BNDCFGS);
                break;
        case MSR_IA32_MCG_EXT_CTL:
                if (!msr_info->host_initiated &&
                    !(vmx->msr_ia32_feature_control &
                      FEATURE_CONTROL_LMCE))
                        return 1;
                msr_info->data = vcpu->arch.mcg_ext_ctl;
                break;
        case MSR_IA32_FEATURE_CONTROL:
                msr_info->data = vmx->msr_ia32_feature_control;
                break;
        case MSR_IA32_VMX_BASIC ... MSR_IA32_VMX_VMFUNC:
                if (!nested_vmx_allowed(vcpu))
                        return 1;
                return vmx_get_vmx_msr(&vmx->nested.msrs, msr_info->index,
                                       &msr_info->data);
        case MSR_IA32_XSS:
                if (!vmx_xsaves_supported())
                        return 1;
                msr_info->data = vcpu->arch.ia32_xss;
                break;
        case MSR_IA32_RTIT_CTL:
                if (pt_mode != PT_MODE_HOST_GUEST)
                        return 1;
                msr_info->data = vmx->pt_desc.guest.ctl;
                break;
        case MSR_IA32_RTIT_STATUS:
                if (pt_mode != PT_MODE_HOST_GUEST)
                        return 1;
                msr_info->data = vmx->pt_desc.guest.status;
                break;
        case MSR_IA32_RTIT_CR3_MATCH:
                if ((pt_mode != PT_MODE_HOST_GUEST) ||
                        !intel_pt_validate_cap(vmx->pt_desc.caps,
                                                PT_CAP_cr3_filtering))
                        return 1;
                msr_info->data = vmx->pt_desc.guest.cr3_match;
                break;
        case MSR_IA32_RTIT_OUTPUT_BASE:
                if ((pt_mode != PT_MODE_HOST_GUEST) ||
                        (!intel_pt_validate_cap(vmx->pt_desc.caps,
                                        PT_CAP_topa_output) &&
                         !intel_pt_validate_cap(vmx->pt_desc.caps,
                                        PT_CAP_single_range_output)))
                        return 1;
                msr_info->data = vmx->pt_desc.guest.output_base;
                break;
        case MSR_IA32_RTIT_OUTPUT_MASK:
                if ((pt_mode != PT_MODE_HOST_GUEST) ||
                        (!intel_pt_validate_cap(vmx->pt_desc.caps,
                                        PT_CAP_topa_output) &&
                         !intel_pt_validate_cap(vmx->pt_desc.caps,
                                        PT_CAP_single_range_output)))
                        return 1;
                msr_info->data = vmx->pt_desc.guest.output_mask;
                break;
        case MSR_IA32_RTIT_ADDR0_A ... MSR_IA32_RTIT_ADDR3_B:
                index = msr_info->index - MSR_IA32_RTIT_ADDR0_A;
                if ((pt_mode != PT_MODE_HOST_GUEST) ||
                        (index >= 2 * intel_pt_validate_cap(vmx->pt_desc.caps,
                                        PT_CAP_num_address_ranges)))
                        return 1;
                if (index % 2)
						 msr_info->data = vmx->pt_desc.guest.addr_b[index / 2];
                else
                        msr_info->data = vmx->pt_desc.guest.addr_a[index / 2];
                break;
        case MSR_TSC_AUX:
                if (!msr_info->host_initiated &&
                    !guest_cpuid_has(vcpu, X86_FEATURE_RDTSCP))
                        return 1;
                /* Otherwise falls through */
        default:
                msr = find_msr_entry(vmx, msr_info->index);
                if (msr) {
                        msr_info->data = msr->data;
                        break;
                }
                return kvm_get_msr_common(vcpu, msr_info);
        }

        return 0;
}
```
我们可以看见，其根据msr_info->index进行switch进行不同的处理，msr_info->index的值为guest中执行rdmsr指令是RCX寄存器的值，即为要读的MSR寄存的index。

在该switch()的实现中，我们可以分为以下几类：
1. 直接读取vmcs区域中guest MSR区域
	- MSR_FS_BASE (X86_64独有): vmcs_readl(GUEST_FS_BASE)
	- MSR_GS_BASE (X86_64独有): vmcs_readl(GUEST_GS_BASE)
	- MSR_KERNEL_GS_BASE (X86_64独有，其处理有点特殊，具体看代码): rdmsrl(MSR_KERNEL_GS_BASE, vmx->msr_guest_kernel_gs_base)
	- MSR_IA32_SYSENTER_CS: vmcs_read32(GUEST_SYSENTER_CS)
	- MSR_IA32_SYSENTER_RIP: vmcs_readl(GUEST_SYSENTER_EIP)
	- MSR_IA32_SYSENTER_ESP: vmcs_readl(GUEST_SYSENTER_ESP)
	- MSR_IA32_BNDCFGS: vmcs_read64(GUEST_BNDCFGS)
2. 读取strcut vcpu_vmx中保存的值
	- MSR_IA32_SPEC_CTRL: vmx->spec_ctrl;
	- MSR_IA32_ARCH_CAPABILITIES: vmx->arch_capabilities;
	- MSR_IA32_FEATURE_CONTROL: vmx->msr_ia32_feature_control;
	- MSR_IA32_RTIT_CTL: vmx->pt_desc.guest.ctl;
	- MSR_IA32_RTIT_STATUS: vmx->pt_desc.guest.status;
	- MSR_IA32_RTIT_CR3_MATCH: vmx->pt_desc.guest.cr3_match;
	- MSR_IA32_RTIT_OUTPUT_BASE: vmx->pt_desc.guest.output_base;
	- MSR_IA32_RTIT_OUTPUT_MASK: vmx->pt_desc.guest.output_mask;
	- MSR_IA32_RTIT_ADDR0_A ... MSR_IA32_RTIT_ADDR3_B: 
3. 读取strcut kvm_vcpu->arch的值
	- MSR_IA32_MCG_EXT_CTL: vcpu->arch.mcg_ext_ctrl;
	- MSR_IA32_XSS: vcpu->arch.ia32_xss;
4. 对nested模式中MSR的模拟，会调用vmx_get_vmx_msr
	- MSR_IA32_VMX_BASIC ... MSR_IA32_VMX_VMFUNC
5. 调用find_msr_entry(struct vcpu_vmx * vmx, u32 msr)函数，查找对应的MSR在vmx->guest_msrs[]数组中是否存在。
如果存在，则返回vmx->guest_msrs[]中对应MSR的值; 如果不存在，则到下一类，使用kvm_get_msr_common();
   保存在vmx->guest_msrs[]数组中的值，可以参考上一篇文章，kvm对guest中MSR寄存器的处理（一），其根据vmx_msr_index[]数组初始化，其中可能包括的MSR有：
	- MSR_SYSCALL_MASK (X86_64独有), 
	- MSR_LSTAR (X86_64独有), 
	- MSR_CSTAR (X86_64独有),
	- MSR_EFER (见下一类使用kvm_get_msr_common()), 
	- MSR_TSC_AUX, 
	- MSR_STAR
6. 调用`kvm_get_msr_common(struct kvm_vcpu * vcpu, struct msr_data * msr_data)`
	- MSR_EFER
	- 以及以上5类不包含的MSR

# kvm_get_msr_common()
根据上面对vmx_get_msr()函数的分析，可以发现大多数的MSR都落到了kvm_get_msr_common()这个函数，
下面我们来看这个函数,该函数的实现在x86.c中。
```c
int kvm_get_msr_common(struct kvm_vcpu *vcpu, struct msr_data *msr_info)
{
        switch (msr_info->index) {
        case MSR_IA32_PLATFORM_ID:
        case MSR_IA32_EBL_CR_POWERON:
        case MSR_IA32_DEBUGCTLMSR:
        case MSR_IA32_LASTBRANCHFROMIP:
        case MSR_IA32_LASTBRANCHTOIP:
        case MSR_IA32_LASTINTFROMIP:
        case MSR_IA32_LASTINTTOIP:
        case MSR_K8_SYSCFG:
        case MSR_K8_TSEG_ADDR:
        case MSR_K8_TSEG_MASK:
        case MSR_K7_HWCR:
        case MSR_VM_HSAVE_PA:
        case MSR_K8_INT_PENDING_MSG:
        case MSR_AMD64_NB_CFG:
        case MSR_FAM10H_MMIO_CONF_BASE:
        case MSR_AMD64_BU_CFG2:
        case MSR_IA32_PERF_CTL:
        case MSR_AMD64_DC_CFG:
        case MSR_F15H_EX_CFG:
                msr_info->data = 0; 
                break;
        case MSR_F15H_PERF_CTL0 ... MSR_F15H_PERF_CTR5:
        case MSR_K7_EVNTSEL0 ... MSR_K7_EVNTSEL3:
        case MSR_K7_PERFCTR0 ... MSR_K7_PERFCTR3:
        case MSR_P6_PERFCTR0 ... MSR_P6_PERFCTR1:
        case MSR_P6_EVNTSEL0 ... MSR_P6_EVNTSEL1:
                if (kvm_pmu_is_valid_msr(vcpu, msr_info->index))
                        return kvm_pmu_get_msr(vcpu, msr_info->index, &msr_info->data);
                msr_info->data = 0; 
                break;
        case MSR_IA32_UCODE_REV:
                msr_info->data = vcpu->arch.microcode_version;
                break;
        case MSR_IA32_TSC:
                msr_info->data = kvm_scale_tsc(vcpu, rdtsc()) + vcpu->arch.tsc_offset;
                break;
        case MSR_MTRRcap:
        case 0x200 ... 0x2ff:
                return kvm_mtrr_get_msr(vcpu, msr_info->index, &msr_info->data);
        case 0xcd: /* fsb frequency */
                msr_info->data = 3; 
                break;
                /*   
                 * MSR_EBC_FREQUENCY_ID
                 * Conservative value valid for even the basic CPU models.
                 * Models 0,1: 000 in bits 23:21 indicating a bus speed of
                 * 100MHz, model 2 000 in bits 18:16 indicating 100MHz,
                 * and 266MHz for model 3, or 4. Set Core Clock
                 * Frequency to System Bus Frequency Ratio to 1 (bits
                 * 31:24) even though these are only valid for CPU
                 * models > 2, however guests may end up dividing or
                 * multiplying by zero otherwise.
                 */
        case MSR_EBC_FREQUENCY_ID:
                msr_info->data = 1 << 24;
                break;
        case MSR_IA32_APICBASE:
                msr_info->data = kvm_get_apic_base(vcpu);
                break;
        case APIC_BASE_MSR ... APIC_BASE_MSR + 0x3ff:
                return kvm_x2apic_msr_read(vcpu, msr_info->index, &msr_info->data);
                break;
        case MSR_IA32_TSCDEADLINE:
                msr_info->data = kvm_get_lapic_tscdeadline_msr(vcpu);
                break;
        case MSR_IA32_TSC_ADJUST:
                msr_info->data = (u64)vcpu->arch.ia32_tsc_adjust_msr;
                break;
        case MSR_IA32_MISC_ENABLE:
                msr_info->data = vcpu->arch.ia32_misc_enable_msr;
                break;
        case MSR_IA32_SMBASE:
                if (!msr_info->host_initiated)
                        return 1;
                msr_info->data = vcpu->arch.smbase;
                break;
        case MSR_SMI_COUNT:
                msr_info->data = vcpu->arch.smi_count;
                break;
        case MSR_IA32_PERF_STATUS:
                /* TSC increment by tick */
                msr_info->data = 1000ULL;
                /* CPU multiplier */
                msr_info->data |= (((uint64_t)4ULL) << 40); 
                break;
        case MSR_EFER:
                msr_info->data = vcpu->arch.efer;
                break;
        case MSR_KVM_WALL_CLOCK:
        case MSR_KVM_WALL_CLOCK_NEW:
                msr_info->data = vcpu->kvm->arch.wall_clock;
                break;
        case MSR_KVM_SYSTEM_TIME:
        case MSR_KVM_SYSTEM_TIME_NEW:
                msr_info->data = vcpu->arch.time;
                break;
        case MSR_KVM_ASYNC_PF_EN:
                msr_info->data = vcpu->arch.apf.msr_val;
                break;
        case MSR_KVM_STEAL_TIME:
                msr_info->data = vcpu->arch.st.msr_val;
                break;
        case MSR_KVM_PV_EOI_EN:
                msr_info->data = vcpu->arch.pv_eoi.msr_val;
                break;
        case MSR_IA32_P5_MC_ADDR:
        case MSR_IA32_P5_MC_TYPE:
        case MSR_IA32_MCG_CAP:
        case MSR_IA32_MCG_CTL:
        case MSR_IA32_MCG_STATUS:
        case MSR_IA32_MC0_CTL ... MSR_IA32_MCx_CTL(KVM_MAX_MCE_BANKS) - 1: 
                return get_msr_mce(vcpu, msr_info->index, &msr_info->data,
                                   msr_info->host_initiated);
        case MSR_K7_CLK_CTL:
                /*
                 * Provide expected ramp-up count for K7. All other
                 * are set to zero, indicating minimum divisors for
                 * every field.
                 *
                 * This prevents guest kernels on AMD host with CPU
                 * type 6, model 8 and higher from exploding due to
                 * the rdmsr failing.
                 */
                msr_info->data = 0x20000000;
                break;
        case HV_X64_MSR_GUEST_OS_ID ... HV_X64_MSR_SINT15:
        case HV_X64_MSR_CRASH_P0 ... HV_X64_MSR_CRASH_P4:
        case HV_X64_MSR_CRASH_CTL:
        case HV_X64_MSR_STIMER0_CONFIG ... HV_X64_MSR_STIMER3_COUNT:
        case HV_X64_MSR_REENLIGHTENMENT_CONTROL:
        case HV_X64_MSR_TSC_EMULATION_CONTROL:
        case HV_X64_MSR_TSC_EMULATION_STATUS:
                return kvm_hv_get_msr_common(vcpu,
                                             msr_info->index, &msr_info->data,
                                             msr_info->host_initiated);
                break;
        case MSR_IA32_BBL_CR_CTL3:
                /* This legacy MSR exists but isn't fully documented in current
                 * silicon.  It is however accessed by winxp in very narrow
                 * scenarios where it sets bit #19, itself documented as
                 * a "reserved" bit.  Best effort attempt to source coherent
                 * read data here should the balance of the register be
                 * interpreted by the guest:
                 *
                 * L2 cache control register 3: 64GB range, 256KB size,
                 * enabled, latency 0x1, configured
                 */
                msr_info->data = 0xbe702111;
                break;
        case MSR_AMD64_OSVW_ID_LENGTH:
                if (!guest_cpuid_has(vcpu, X86_FEATURE_OSVW))
                        return 1;
                msr_info->data = vcpu->arch.osvw.length;
                break;
        case MSR_AMD64_OSVW_STATUS:
                if (!guest_cpuid_has(vcpu, X86_FEATURE_OSVW))
                        return 1;
                msr_info->data = vcpu->arch.osvw.status;
                break;
        case MSR_PLATFORM_INFO:
                if (!msr_info->host_initiated &&
                    !vcpu->kvm->arch.guest_can_read_msr_platform_info)
                        return 1;
                msr_info->data = vcpu->arch.msr_platform_info;
                break;
        case MSR_MISC_FEATURES_ENABLES:
                msr_info->data = vcpu->arch.msr_misc_features_enables;
                break;
        default:
                if (kvm_pmu_is_valid_msr(vcpu, msr_info->index))
                        return kvm_pmu_get_msr(vcpu, msr_info->index, &msr_info->data);
                if (!ignore_msrs) {
                        vcpu_debug_ratelimited(vcpu, "unhandled rdmsr: 0x%x\n",
                                               msr_info->index);
                        return 1;
                } else {
                        if (report_ignored_msrs)
                                vcpu_unimpl(vcpu, "ignored rdmsr: 0x%x\n",
                                        msr_info->index);
                        msr_info->data = 0;
                }
                break;
        }
        return 0;
}
EXPORT_SYMBOL_GPL(kvm_get_msr_common);
```
