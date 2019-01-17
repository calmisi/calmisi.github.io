---
title: kvm源码解析之kvm_main.c
date: 2018-08-30 19:23:11
tags:
- KVM
categories:
- KVM
---

kvm_main.c中的init函数首先会调用，kvm_arch_init(opaque);

1. kvm_arch_init
这时会调用与architecture相关的对应的初始化函数，
我们只关心x86平台，所以看对应的arch/x86/kvm/x86.c中的实现：

<!-- more -->

```c
int kvm_arch_init(void *opaque)
{
	int r;
	//该结构体为与x86平台相关的kvm需要使用的函数的指针集合
	struct kvm_x86_ops *ops = opaque;

	//首先判断该指针是否已经被赋值，即是否已经初始化x86平台相关
	if (kvm_x86_ops) {
		printk(KERN_ERR "kvm: already loaded the other module\n");
		r = -EEXIST;
		goto out;
	}

	//调用x86平台对应的函数实现，检测cpu是否支持KVM
	if (!ops->cpu_has_kvm_support()) {
		printk(KERN_ERR "kvm: no hardware support\n");
		r = -EOPNOTSUPP;
		goto out;
	}
	if (ops->disabled_by_bios()) {
		printk(KERN_ERR "kvm: disabled by bios\n");
		r = -EOPNOTSUPP;
		goto out;
	}

	r = -ENOMEM;
	//为每个CPU建立shared msrs
	shared_msrs = alloc_percpu(struct kvm_shared_msrs);
	if (!shared_msrs) {
		printk(KERN_ERR "kvm: failed to allocate percpu kvm_shared_msrs\n");
		goto out;
	}

	//初始化mmu
	r = kvm_mmu_module_init();
	if (r)
		goto out_free_percpu;

	kvm_set_mmio_spte_mask();

	//注意，这里给kvm_x86_ops赋值，即下次在调用该函数进行x86平台的初始化，则会提示已经加载了与x86平台的其他实现。
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

2. kvm_irqfd_init()

3. zalloc_cpu_mask_var()

4. kvm_arch_hardware_setup()
会调用与architecture相关的hardware setup函数，
这里我们看arch/x86/kvm/x86.c

```c
int kvm_arch_hardware_setup(void)
{
	int r;

	//在第1步中，kvm_arch_init()函数已经初始化好了kvm_x86_ops，
	//这里只用调用其hardware_setup()函数，
	//具体的该函数在vmx.c中，其为intel x86平台对vmm的硬件支持的实现。
	//并且kvm模块的入口函数也是在vmx.c中，在vmx.c中有 module_init(vmx_init)，
	//vmx_init()中，调用kvm_main.c中的kvm_arch_init(),即该文档的起始原因。
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
	//根据 msrs_to_save, emulated_msrs, msr_based_features三个数组中存的值，并查询具体的CPU的支持，并进行更新。
	return 0;
}
```
