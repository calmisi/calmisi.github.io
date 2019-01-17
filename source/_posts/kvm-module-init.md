---
title: kvm源码解析之kvm-intel.ko模块初始化
date: 2018-09-18 21:51:00
tag:
- KVM
categories:
- KVM
---

# 1. x86 arch with Intel cpu needs two modules kvm.ko and kvm-intel.ko
KVM是kernel-based virtual machine,但平台使用的是Intel CPU的时，需要用到内核中的2个模块：kvm.ko和kvm-intel.ko。
其中kvm-intel.ko模块由vmx.c和pmu_intel.c两个文件编译。
本文分析KVM源码中，系统加载这两个模块的时候，内核做了哪些工作。

<!-- more -->

# kvm-intel.ko模块初始化
vmx.c包含了Intel CPU对x86平台的虚拟化支持。
在加载kvm-intel.ko模块的时候，根据`module_init(vmx_init)`首先会调用vmx_init函数
```c
static int __init vmx_init(void)
{
    int r;
    ...
    
    r = kvm_init(&vmx_x86_ops, sizeof(struct vcpu_vmx),
                __alignof__(struct vcpu_vmx), THIS_MODULE);

    if (r)
        return r;

    if (boot_cpu_has(X86_BUG_L1TF)) {
        r = vmx_setup_l1d_flush(vmenrty_l1d_flush_param);
        if (r) {
            vmx_exit();
            return r;
        }
    }

    vmx_check_vmcs12_offsets();

    return 0;
}
```
其中会调用`kvm_init`,并传入vmx_x86_ops,和对应vcpu的strcut vcpu_vmx,以及其对齐性。

```c
//kvm_main.c 为kvm.ko模块提供的一个函数
int kvm_init(void * opaque, unsigned vcpu_size, unsigned vcpu_align,
             strcut module * module)
{
    int r;
    int cpu;

    r = kvm_arch_init(opaque);
    ...
}
```
---
在kvm_init()中，分别做了以下几件事情：
- 调用了kvm_arch_init(),其会根据具体的architecture(mips,powerpc,s390,x86,arm)选择不同的实现.
 我们这里为Intel的架构，为x86架构，其对应的kvm_arch_init()函数实现在x86.c文件中。
```c
int kvm_arch_init(void * opaque)
{
    int r;
    struct kvm_x86_ops * ops = oqaque;//
    
    ...//do some checks to check whether the machine support KVM, and whether if enable it
    
    shared_msrs = alloc_percpu(struct kvm_shared_msrs);
    if (!shared_msrs) {
        printk();
        goto out;
    }
    
    r = kvm_mmu_module_init();
    if (r)
		goto out_free_percpu;

	kvm_set_mmio_spte_mask();

	kvm_x86_ops = ops;

	kvm_mmu_set_mask_ptes(PT_USER_MASK, PT_ACCESSED_MASK,
			PT_DIRTY_MASK, PT64_NX_MASK, 0,
			PT_PRESENT_MASK, 0, sme_me_mask);
	kvm_timer_init();

	perf_register_guest_info_callbacks(&kvm_guest_cbs);

	if (boot_cpu_has(X86_FEATURE_XSAVE))
		host_xcr0 = xgetbv(XCR_XFEATURE_ENABLED_MASK);

	kvm_lapic_init();
#ifdef CONFIG_X86_64
	pvclock_gtod_register_notifier(&pvclock_gtod_notifier);

	if (hypervisor_is_type(X86_HYPER_MS_HYPERV))
		set_hv_tscchange_cb(kvm_hyperv_tsc_notifier);
#endif

	return 0;

out_free_percpu:
	free_percpu(shared_msrs);
out:
	return r;
}
```
该函数主要做了以下几件事：
1. alloc_percpu(struct kvm_shared_msrs)为每个CPU分配kvm_shared_msrs结构体
2. kvm_mmu_module_init()
3. kvm_set_mmio_spte_mask() 
4. kvm_mmu_set_mask_ptes() 初始化MMU
5. kvm_time_init()  初始化kvm的timer
6. perf_register_guest_info_callbacks(&kvm_guest_cbs);
7. kvm_lapic_init();
{
    /* do not patch jump label more than once per second */
    jump_label_rate_limit(&apic_hw_disabled, HZ);
    jump_label_rate_limit(&apic_sw_disabled, HZ);
}
8. (optional)host_xcr0 = xgetbv(XCR_XFEATURE_ENABLED_MASK);
(optional)pvclock_gtod_register_notifier(&pvclock_gtod_notifier);
(optional)set_hv_tscchange_cb(kvm_hyperv_tsc_notifier);

- 接着在kvm_init()函数中，调用了`kvm_irqfd_init()` 如果没有配置CONFIG_HAVE_KVM_IRQFD，则该函数return 0.

- 接着调用kvm_arch_hardware_setup()函数，进行体系结构相关的硬件配置
```c
// in x86.c
kvm_arch_hardware_setup()
{
    int r;

    r = kvm_x86_ops->hardware_setup();
    if (r != 0)
		return r;

	if (kvm_has_tsc_control) {
		/*
		 * Make sure the user can only configure tsc_khz values that
		 * fit into a signed integer.
		 * A min value is not calculated because it will always
		 * be 1 on all machines.
		 */
		u64 max = min(0x7fffffffULL,
			      __scale_tsc(kvm_max_tsc_scaling_ratio, tsc_khz));
		kvm_max_guest_tsc_khz = max;

		kvm_default_tsc_scaling_ratio = 1ULL << kvm_tsc_scaling_ratio_frac_bits;
	}

	kvm_init_msr_list();
	return 0;

}
```
可以看见在kvm_arch_hardware_setup()函数中，调用了kvm_x86_ops的hardware_setup(),`kvm_x86_ops`为我们最开始在vmx.c文件中定义的，并通过模块的初始化函数vmx_init()->kvm_init()->kvm_arch_hardware_setup()
```c
static __init int hardware_setup(void)
{
	unsigned long host_bndcfgs;
	int r = -ENOMEM, i;

	rdmsrl_safe(MSR_EFER, &host_efer);

	for (i = 0; i < ARRAY_SIZE(vmx_msr_index); ++i)
		kvm_define_shared_msr(i, vmx_msr_index[i]);

	for (i = 0; i < VMX_BITMAP_NR; i++) {
		vmx_bitmap[i] = (unsigned long *)__get_free_page(GFP_KERNEL);
		if (!vmx_bitmap[i])
			goto out;
	}

	memset(vmx_vmread_bitmap, 0xff, PAGE_SIZE);
	memset(vmx_vmwrite_bitmap, 0xff, PAGE_SIZE);

	if (setup_vmcs_config(&vmcs_config) < 0) {
		r = -EIO;
		goto out;
	}

	if (boot_cpu_has(X86_FEATURE_NX))
		kvm_enable_efer_bits(EFER_NX);

	if (boot_cpu_has(X86_FEATURE_MPX)) {
		rdmsrl(MSR_IA32_BNDCFGS, host_bndcfgs);
		WARN_ONCE(host_bndcfgs, "KVM: BNDCFGS in host will be lost");
	}

	if (!cpu_has_vmx_vpid() || !cpu_has_vmx_invvpid() ||
		!(cpu_has_vmx_invvpid_single() || cpu_has_vmx_invvpid_global()))
		enable_vpid = 0;

	if (!cpu_has_vmx_ept() ||
	    !cpu_has_vmx_ept_4levels() ||
	    !cpu_has_vmx_ept_mt_wb() ||
	    !cpu_has_vmx_invept_global())
		enable_ept = 0;

	if (!cpu_has_vmx_ept_ad_bits() || !enable_ept)
		enable_ept_ad_bits = 0;

	if (!cpu_has_vmx_unrestricted_guest() || !enable_ept)
		enable_unrestricted_guest = 0;

	if (!cpu_has_vmx_flexpriority())
		flexpriority_enabled = 0;

	if (!cpu_has_virtual_nmis())
		enable_vnmi = 0;

	/*
	 * set_apic_access_page_addr() is used to reload apic access
	 * page upon invalidation.  No need to do anything if not
	 * using the APIC_ACCESS_ADDR VMCS field.
	 */
	if (!flexpriority_enabled)
		kvm_x86_ops->set_apic_access_page_addr = NULL;

	if (!cpu_has_vmx_tpr_shadow())
		kvm_x86_ops->update_cr8_intercept = NULL;

	if (enable_ept && !cpu_has_vmx_ept_2m_page())
		kvm_disable_largepages();

#if IS_ENABLED(CONFIG_HYPERV)
	if (ms_hyperv.nested_features & HV_X64_NESTED_GUEST_MAPPING_FLUSH
	    && enable_ept)
		kvm_x86_ops->tlb_remote_flush = vmx_hv_remote_flush_tlb;
#endif

	if (!cpu_has_vmx_ple()) {
		ple_gap = 0;
		ple_window = 0;
		ple_window_grow = 0;
		ple_window_max = 0;
		ple_window_shrink = 0;
	}

	if (!cpu_has_vmx_apicv()) {
		enable_apicv = 0;
		kvm_x86_ops->sync_pir_to_irr = NULL;
	}

	if (cpu_has_vmx_tsc_scaling()) {
		kvm_has_tsc_control = true;
		kvm_max_tsc_scaling_ratio = KVM_VMX_TSC_MULTIPLIER_MAX;
		kvm_tsc_scaling_ratio_frac_bits = 48;
	}

	set_bit(0, vmx_vpid_bitmap); /* 0 is reserved for host */

	if (enable_ept)
		vmx_enable_tdp();
	else
		kvm_disable_tdp();

	if (!nested) {
		kvm_x86_ops->get_nested_state = NULL;
		kvm_x86_ops->set_nested_state = NULL;
	}

	/*
	 * Only enable PML when hardware supports PML feature, and both EPT
	 * and EPT A/D bit features are enabled -- PML depends on them to work.
	 */
	if (!enable_ept || !enable_ept_ad_bits || !cpu_has_vmx_pml())
		enable_pml = 0;

	if (!enable_pml) {
		kvm_x86_ops->slot_enable_log_dirty = NULL;
		kvm_x86_ops->slot_disable_log_dirty = NULL;
		kvm_x86_ops->flush_log_dirty = NULL;
		kvm_x86_ops->enable_log_dirty_pt_masked = NULL;
	}

	if (cpu_has_vmx_preemption_timer() && enable_preemption_timer) {
		u64 vmx_msr;

		rdmsrl(MSR_IA32_VMX_MISC, vmx_msr);
		cpu_preemption_timer_multi =
			 vmx_msr & VMX_MISC_PREEMPTION_TIMER_RATE_MASK;
	} else {
		kvm_x86_ops->set_hv_timer = NULL;
		kvm_x86_ops->cancel_hv_timer = NULL;
	}

	if (!cpu_has_vmx_shadow_vmcs())
		enable_shadow_vmcs = 0;
	if (enable_shadow_vmcs)
		init_vmcs_shadow_fields();

	kvm_set_posted_intr_wakeup_handler(wakeup_handler);
	nested_vmx_setup_ctls_msrs(&vmcs_config.nested, enable_apicv);

	kvm_mce_cap_supported |= MCG_LMCE_P;

	return alloc_kvm_area();

out:
	for (i = 0; i < VMX_BITMAP_NR; i++)
		free_page((unsigned long)vmx_bitmap[i]);

    return r;
}
```
总结一下，vmx的hardware_setup做了以下的工作：
+ host_efer;
+ 使用vmx体系中的vmx_msr_index[]数组，去初始化x86.c文件中kvm_shared_msrs_global[]数组;
+ 为vmx_vmread_bitmap和vmx_vmwrite_bitmap申请内存空间，每个为4kB，占一个page页;
+ setup_vmcs_config(&vmcs_config)根据host cpu的cpuid和msrs初始化该平台的vmcs结构体vmcs_config和vmx_capability;
+ 设置x86.c文件中的efer_reserved_bits；
+ 根据vmcs_config和vmx_capability在setup_vmcs_config()设置的值修改vmx.c文件中enable_vpid, enable_ept, enable_ept_ad_bits, enable_unrestricted_guest, flexpriority_enabled, enable_vnmi, largepages_enabled的值。
+ 如果当前vmcs_config不支持PLE，则将ple_gap, ple_window, ple_window_grow, ple_window_max, ple_window_shrink设为0;
+ 如果vmcs_config不支持apicv,则设置enable_apicv和kvm_x86_ops->sync_pir_to_irr函数句柄;
+ 如果vmcs_config支持tsc scaling,则设置kvm_has_tsc_control, kvm_max_tsc_scaling_ratio, kvm_tsc_scaling_ratio_frac_bits；
+ 设置vmx_vpid_bitmap为0
+ 如果enable_ept，则vmx_enable_tdp(),否则kvm_disable_tdp();