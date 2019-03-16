---
title: kernel对CPUID所能枚举的features的处理
date: 2019-02-18 22:20:53
tags:
- CPUID
categories:
- Kernel
---

本文介绍Linux kernel是如何读取CPU支持哪些cpu features,并是如何将`CPUID`指令查询的cpu features保存到`boot_cpu_data.x86_capabilities`结构中的。

<!-- more -->

#CPUID在kernel中的组织
在kernel启动的时候，会读取平台支持的CPUID features，并存储在boot_cpu_data.x86_capability[]数组中。
在kernel实际运行的时候，直接查询boot_cpu_data.x86_capability[]数组，而不是直接运行CPUID 指令。

```C
#define NCAPINTS                        19         /* N 32-bit words worth of info */
#define NBUGINTS                        1          /* N 32-bit bug flags */

struct cpuinfo_x86 {
	...
	__u32	x86_capability[NCAPINTS + NBUGINTS]
		__aligned(sizeof(unsigned long));
	...
}
```

in init/main.c
```C
asmlinkage __visible void __init start_kernel(void)
{
	...
	boot_cpu_init();
	...
	setup_arch(&command_line);
	...
}

void __init setup_arch(char **cmdline_p)
{
	...
#ifdef CONFIG_X86_32
	memcpy(&boot_cpu_data, &new_cpu_data, sizeof(new_cpu_data));
	...
#else
	printk(KERN_INFO "Command line: %s\n", boot_command_line);
	boot_cpu_data.x86_phys_bits = MAX_PHYSMEM_BITS;
#endif
	...
	early_cpu_init();
	...
}

void __init early_cpu_init(void)
{
	...
	early_identify_cpu(&boot_cpu_data);
}
```
可以看到从`start_kernel()`开始，会调用`setup_arch()`，然后调用`early_cpu_init()`函数，在`early_cpu_init()`函数中调用`early_identify_cpu(&boot_cpu_data)`函数来初始化`boot_cpu_data`数据结构。下面我们来看`early_identify_cpu()`函数。
```C
static void __init early_identify_cpu(struct cpuinfo_x86 *c)
{
#ifdef CONFIG_X86_64
        c->x86_clflush_size = 64;
        c->x86_phys_bits = 36;
        c->x86_virt_bits = 48;
#else
        c->x86_clflush_size = 32;
        c->x86_phys_bits = 32;
        c->x86_virt_bits = 32;
#endif
        c->x86_cache_alignment = c->x86_clflush_size;

        memset(&c->x86_capability, 0, sizeof(c->x86_capability));
        c->extended_cpuid_level = 0;

        if (!have_cpuid_p())
                identify_cpu_without_cpuid(c);

        /* cyrix could have cpuid enabled via c_identify()*/
        if (have_cpuid_p()) {
                cpu_detect(c);
                get_cpu_vendor(c);
                get_cpu_cap(c);
                get_cpu_address_sizes(c);
                setup_force_cpu_cap(X86_FEATURE_CPUID);

                if (this_cpu->c_early_init)
                        this_cpu->c_early_init(c);

                c->cpu_index = 0;
                filter_cpuid_features(c, false);

                if (this_cpu->c_bsp_init)
                        this_cpu->c_bsp_init(c);
        } else {
                setup_clear_cpu_cap(X86_FEATURE_CPUID);
        }

        setup_force_cpu_cap(X86_FEATURE_ALWAYS);

        cpu_set_bug_bits(c);

        fpu__init_system(c);

#ifdef CONFIG_X86_32
        /*
         * Regardless of whether PCID is enumerated, the SDM says
         * that it can't be enabled in 32-bit mode.
         */
        setup_clear_cpu_cap(X86_FEATURE_PCID);
#endif

        /*
         * Later in the boot process pgtable_l5_enabled() relies on
         * cpu_feature_enabled(X86_FEATURE_LA57). If 5-level paging is not
         * enabled by this point we need to clear the feature bit to avoid
         * false-positives at the later stage.
         *
         * pgtable_l5_enabled() can be false here for several reasons:
         *  - 5-level paging is disabled compile-time;
         *  - it's 32-bit kernel;
         *  - machine doesn't support 5-level paging;
         *  - user specified 'no5lvl' in kernel command line.
         */
        if (!pgtable_l5_enabled())
                setup_clear_cpu_cap(X86_FEATURE_LA57);

        detect_nopl();
}
```
`early_identify_cpu()`函数会先将boot_cpu_data.x86_capability清零；设置c->extended_cpuid_level = 0;
然后判断CPU是否支持CPUID指令（if(have_cpuid_p()），支持则运行if语句：
* cpu_detect()函数，设置boot_cpu_data的cpuid_level, x86_vendor_id[0], x86_vendor_id[8], x86_vendor_id[4], x86, x86_model, x86_stepping, x86_clflush_size, x86_cache_alignment;
* get_cpu_vendor()
* get_cpu_cap(c)
```C
void get_cpu_cap(struct cpuinfo_x86 *c)
{       
        u32 eax, ebx, ecx, edx;
                
        /* Intel-defined flags: level 0x00000001 */
        if (c->cpuid_level >= 0x00000001) {
                cpuid(0x00000001, &eax, &ebx, &ecx, &edx);
                
                c->x86_capability[CPUID_1_ECX] = ecx;
                c->x86_capability[CPUID_1_EDX] = edx;
        }       
        
        /* Thermal and Power Management Leaf: level 0x00000006 (eax) */
        if (c->cpuid_level >= 0x00000006)
                c->x86_capability[CPUID_6_EAX] = cpuid_eax(0x00000006);
                
        /* Additional Intel-defined flags: level 0x00000007 */
        if (c->cpuid_level >= 0x00000007) {
                cpuid_count(0x00000007, 0, &eax, &ebx, &ecx, &edx);
                c->x86_capability[CPUID_7_0_EBX] = ebx;
                c->x86_capability[CPUID_7_ECX] = ecx;
                c->x86_capability[CPUID_7_EDX] = edx;
        }
                
        /* Extended state features: level 0x0000000d */
        if (c->cpuid_level >= 0x0000000d) {
                cpuid_count(0x0000000d, 1, &eax, &ebx, &ecx, &edx);
                
                c->x86_capability[CPUID_D_1_EAX] = eax;
        }               

        /* Additional Intel-defined flags: level 0x0000000F */
        if (c->cpuid_level >= 0x0000000F) {
                        
                /* QoS sub-leaf, EAX=0Fh, ECX=0 */
                cpuid_count(0x0000000F, 0, &eax, &ebx, &ecx, &edx); 
                c->x86_capability[CPUID_F_0_EDX] = edx;
                               
                if (cpu_has(c, X86_FEATURE_CQM_LLC)) {
                        /* will be overridden if occupancy monitoring exists */
                        c->x86_cache_max_rmid = ebx;
                
                        /* QoS sub-leaf, EAX=0Fh, ECX=1 */
                        cpuid_count(0x0000000F, 1, &eax, &ebx, &ecx, &edx);
                        c->x86_capability[CPUID_F_1_EDX] = edx;
        
                        if ((cpu_has(c, X86_FEATURE_CQM_OCCUP_LLC)) ||
                              ((cpu_has(c, X86_FEATURE_CQM_MBM_TOTAL)) ||
                               (cpu_has(c, X86_FEATURE_CQM_MBM_LOCAL)))) {
                                c->x86_cache_max_rmid = ecx;
                                c->x86_cache_occ_scale = ebx;
                        }
                } else {
                        c->x86_cache_max_rmid = -1;
                        c->x86_cache_occ_scale = -1;
                }       
        }               
                
        /* AMD-defined flags: level 0x80000001 */
        eax = cpuid_eax(0x80000000);
        c->extended_cpuid_level = eax; 
                
        if ((eax & 0xffff0000) == 0x80000000) {
                if (eax >= 0x80000001) {
                        cpuid(0x80000001, &eax, &ebx, &ecx, &edx);
        
                        c->x86_capability[CPUID_8000_0001_ECX] = ecx;
                        c->x86_capability[CPUID_8000_0001_EDX] = edx;
                }
        }       
        
        if (c->extended_cpuid_level >= 0x80000007) {
                cpuid(0x80000007, &eax, &ebx, &ecx, &edx);
                
                c->x86_capability[CPUID_8000_0007_EBX] = ebx;
                c->x86_power = edx;
        }

        if (c->extended_cpuid_level >= 0x80000008) {
                cpuid(0x80000008, &eax, &ebx, &ecx, &edx);
                c->x86_capability[CPUID_8000_0008_EBX] = ebx;
        }

        if (c->extended_cpuid_level >= 0x8000000a)
                c->x86_capability[CPUID_8000_000A_EDX] = cpuid_edx(0x8000000a);

        init_scattered_cpuid_features(c);
        init_speculation_control(c);

        /*
         * Clear/Set all flags overridden by options, after probe.
         * This needs to happen each time we re-probe, which may happen  
         * several times during CPU initialization.
         */
        apply_forced_caps(c);
}
```
`get_cpu_cap()`函数会
