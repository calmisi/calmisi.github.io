---
title: 'qemu对CPUID的处理'
date: 2019-02-17 23:16:29
tags:
categories:
- QEMU
---

本文介绍当使用`-cpu`参数指令cpu model后(不论是`-cpu host`或者`-cpu IceLake`等)，QEMU是如何根据不同的cpu model来设置CPUID features的。
关于`-cpu`参数的工作原理，可以参考[qemu中cpu model（cpu type）的设置][cpu_model]
<!-- more -->

# QEMU Feature
关于x86的CPU，有CPUID指令可以查询CPU支持哪些feature。QEMU作为模拟器，在模拟CPU的时候，就涉及到如何表示x86 CPU的这些feature。
> 在linux系统中，可以通过`lscpu`命令查看当前CPU支持哪些features(即`lscpu`命令输出的Flags，或者直接查看`/proc/cpuinfo`中的flags)，这些flags即表示当前CPU支持哪些features。
> 通常这些feature，可以通过`CPUID`指令查询指定leaf_subleaf_E(A/B/C/D)X中某一个bit来表示当前CPU是否支持某个feature。
> 后来出现了feature-based MSR,即MSR中不同bit也可以表示是否支持某个feature。
> 在QEMU中对这些feature的不同索引定义为`FeatureWord`，每个`FeatureWord`可以是CPUID(leaf_subleaf_E(A/B/C/D)X)或者MSR。并且用`enum FeatureWord`表示x86平台总共支持的FeatureWord数。
在QEMU中，target/i386/cpu.c文件中
```C
/* CPUID feature words */
typedef enum FeatureWord {
    FEAT_1_EDX,         /* CPUID[1].EDX */
    FEAT_1_ECX,         /* CPUID[1].ECX */
    FEAT_7_0_EBX,       /* CPUID[EAX=7,ECX=0].EBX */
    FEAT_7_0_ECX,       /* CPUID[EAX=7,ECX=0].ECX */
    FEAT_7_0_EDX,       /* CPUID[EAX=7,ECX=0].EDX */
    FEAT_8000_0001_EDX, /* CPUID[8000_0001].EDX */
    FEAT_8000_0001_ECX, /* CPUID[8000_0001].ECX */
    FEAT_8000_0007_EDX, /* CPUID[8000_0007].EDX */
    FEAT_8000_0008_EBX, /* CPUID[8000_0008].EBX */
    FEAT_C000_0001_EDX, /* CPUID[C000_0001].EDX */
    FEAT_KVM,           /* CPUID[4000_0001].EAX (KVM_CPUID_FEATURES) */
    FEAT_KVM_HINTS,     /* CPUID[4000_0001].EDX */
    FEAT_HYPERV_EAX,    /* CPUID[4000_0003].EAX */
    FEAT_HYPERV_EBX,    /* CPUID[4000_0003].EBX */
    FEAT_HYPERV_EDX,    /* CPUID[4000_0003].EDX */
    FEAT_HV_RECOMM_EAX, /* CPUID[4000_0004].EAX */
    FEAT_HV_NESTED_EAX, /* CPUID[4000_000A].EAX */
    FEAT_SVM,           /* CPUID[8000_000A].EDX */
    FEAT_XSAVE,         /* CPUID[EAX=0xd,ECX=1].EAX */
    FEAT_6_EAX,         /* CPUID[6].EAX */
    FEAT_XSAVE_COMP_LO, /* CPUID[EAX=0xd,ECX=0].EAX */
    FEAT_XSAVE_COMP_HI, /* CPUID[EAX=0xd,ECX=0].EDX */
    FEAT_ARCH_CAPABILITIES,
    FEATURE_WORDS,
} FeatureWord;
```
然后在QEMU中定义了`feature_word_info[]`数组，其表示了每个FeatureWord中有效的bit位有哪些，分别表示哪个feature等信息。
```C
static FeatureWordInfo feature_word_info[FEATURE_WORDS] = {
	[FEAT_1_EDX] = {
		...
	},
	[FEAT_1_ECX] = {
		...
	},
	...
	[FEAT_ARCH_CAPABILITIES] = {
		...
	},
};
```


# How to load and filter CPUID data
```C
/***** Steps involved on loading and filtering CPUID data
 *
 * When initializing and realizing a CPU object, the steps
 * involved in setting up CPUID data are:
 *
 * 1) Loading CPU model definition (X86CPUDefinition). This is
 *    implemented by x86_cpu_load_def() and should be completely
 *    transparent, as it is done automatically by instance_init.
 *    No code should need to look at X86CPUDefinition structs
 *    outside instance_init.
 *
 * 2) CPU expansion. This is done by realize before CPUID
 *    filtering, and will make sure host/accelerator data is
 *    loaded for CPU models that depend on host capabilities
 *    (e.g. "host"). Done by x86_cpu_expand_features().
 *
 * 3) CPUID filtering. This initializes extra data related to
 *    CPUID, and checks if the host supports all capabilities
 *    required by the CPU. Runnability of a CPU model is
 *    determined at this step. Done by x86_cpu_filter_features().
 *
 * Some operations don't require all steps to be performed.
 * More precisely:
 *
 * - CPU instance creation (instance_init) will run only CPU
 *   model loading. CPU expansion can't run at instance_init-time
 *   because host/accelerator data may be not available yet.
 * - CPU realization will perform both CPU model expansion and CPUID
 *   filtering, and return an error in case one of them fails.
 * - query-cpu-definitions needs to run all 3 steps. It needs
 *   to run CPUID filtering, as the 'unavailable-features'
 *   field is set based on the filtering results.
 * - The query-cpu-model-expansion QMP command only needs to run
 *   CPU model loading and CPU expansion. It should not filter
 *   any CPUID data based on host capabilities.
 */
```
# 1) Load CPU model definition
仅存在于使用`-cpu`指定了特定了特定的CPU model的情况。
## instance_init
在上一篇文章[qemu中cpu model（cpu type）的设置][cpu_model]中提到了5类CPU TYPE:
1. `x86_64-cpu`:
 parent = cpu;
 instance_init = `x86_cpu_initfn`;
 class_init = `x86_cpu_common_class_init`;
2. `max-x86_64-cpu`:
 parent = x86_64-cpu;
 instance_init = `max_x86_cpu_initfn`;
 class_init = `max_x86_cpu_class_init`;
3. `base-x86_64-cpu`:
 parent = x86_64-cpu 
 instance_init = parent.instance_init(`x86_cpu_initfn`);
 class_init = `x86_cpu_base_class_init`;
4. `host--x86_64-cpu`:
 parent = max-x86_64-cpu;
 instance_init = parent.instance_init(`max_x86_cpu_initfn`);
 class_init = `host_x86_cpu_class_init`;
5. `(cpu model)-x86_64-cpu`:
 parent = x86_64-cpu;
 instance_init = parent.instance_init(`x86_cpu_initfn`);
 class_init = `x86_cpu_cpudef_class_init`;

## x86_cpu_initfn
`x86_64-cpu`、`base-x86_64-cpu`、`(cpu model)-x86_64-cpu`会调用`x86_cpu_initfn()`作为instance_init。
```C
static void x86_cpu_initfn(Object *obj)
{
    CPUState *cs = CPU(obj);
    X86CPU *cpu = X86_CPU(obj);
    X86CPUClass *xcc = X86_CPU_GET_CLASS(obj);
    CPUX86State *env = &cpu->env;
    FeatureWord w;

    cs->env_ptr = env; 

    object_property_add(obj, "family", "int",
                        x86_cpuid_version_get_family,
                        x86_cpuid_version_set_family, NULL, NULL, NULL);
    object_property_add(obj, "model", "int",
                        x86_cpuid_version_get_model,
                        x86_cpuid_version_set_model, NULL, NULL, NULL);
    object_property_add(obj, "stepping", "int",
                        x86_cpuid_version_get_stepping,
                        x86_cpuid_version_set_stepping, NULL, NULL, NULL);
    object_property_add_str(obj, "vendor",
                            x86_cpuid_get_vendor,
                            x86_cpuid_set_vendor, NULL);
    object_property_add_str(obj, "model-id",
                            x86_cpuid_get_model_id,
                            x86_cpuid_set_model_id, NULL);
    object_property_add(obj, "tsc-frequency", "int",
                        x86_cpuid_get_tsc_freq,
                        x86_cpuid_set_tsc_freq, NULL, NULL, NULL);
    object_property_add(obj, "feature-words", "X86CPUFeatureWordInfo",
                        x86_cpu_get_feature_words,
                        NULL, NULL, (void *)env->features, NULL);
    object_property_add(obj, "filtered-features", "X86CPUFeatureWordInfo",
                        x86_cpu_get_feature_words,
                        NULL, NULL, (void *)cpu->filtered_features, NULL);

    object_property_add(obj, "crash-information", "GuestPanicInformation",
                        x86_cpu_get_crash_info_qom, NULL, NULL, NULL, NULL);

    cpu->hyperv_spinlock_attempts = HYPERV_SPINLOCK_NEVER_RETRY;

    for (w = 0; w < FEATURE_WORDS; w++) {
        int bitnr;

        for (bitnr = 0; bitnr < 32; bitnr++) {
            x86_cpu_register_feature_bit_props(cpu, w, bitnr);
        }    
    }    

    object_property_add_alias(obj, "sse3", obj, "pni", &error_abort);
    object_property_add_alias(obj, "pclmuldq", obj, "pclmulqdq", &error_abort);
    object_property_add_alias(obj, "sse4-1", obj, "sse4.1", &error_abort);
    object_property_add_alias(obj, "sse4-2", obj, "sse4.2", &error_abort);
    object_property_add_alias(obj, "xd", obj, "nx", &error_abort);
    object_property_add_alias(obj, "ffxsr", obj, "fxsr-opt", &error_abort);
    object_property_add_alias(obj, "i64", obj, "lm", &error_abort);

    object_property_add_alias(obj, "ds_cpl", obj, "ds-cpl", &error_abort);
    object_property_add_alias(obj, "tsc_adjust", obj, "tsc-adjust", &error_abort);
    object_property_add_alias(obj, "fxsr_opt", obj, "fxsr-opt", &error_abort);
    object_property_add_alias(obj, "lahf_lm", obj, "lahf-lm", &error_abort);
    object_property_add_alias(obj, "cmp_legacy", obj, "cmp-legacy", &error_abort);
    object_property_add_alias(obj, "nodeid_msr", obj, "nodeid-msr", &error_abort);
    object_property_add_alias(obj, "perfctr_core", obj, "perfctr-core", &error_abort);
    object_property_add_alias(obj, "perfctr_nb", obj, "perfctr-nb", &error_abort);
    object_property_add_alias(obj, "kvm_nopiodelay", obj, "kvm-nopiodelay", &error_abort);
    object_property_add_alias(obj, "kvm_mmu", obj, "kvm-mmu", &error_abort);
    object_property_add_alias(obj, "kvm_asyncpf", obj, "kvm-asyncpf", &error_abort);
    object_property_add_alias(obj, "kvm_steal_time", obj, "kvm-steal-time", &error_abort);
    object_property_add_alias(obj, "kvm_pv_eoi", obj, "kvm-pv-eoi", &error_abort);
    object_property_add_alias(obj, "kvm_pv_unhalt", obj, "kvm-pv-unhalt", &error_abort);
    object_property_add_alias(obj, "svm_lock", obj, "svm-lock", &error_abort);
    object_property_add_alias(obj, "nrip_save", obj, "nrip-save", &error_abort);
    object_property_add_alias(obj, "tsc_scale", obj, "tsc-scale", &error_abort);
    object_property_add_alias(obj, "vmcb_clean", obj, "vmcb-clean", &error_abort);
    object_property_add_alias(obj, "pause_filter", obj, "pause-filter", &error_abort);
    object_property_add_alias(obj, "sse4_1", obj, "sse4.1", &error_abort);
    object_property_add_alias(obj, "sse4_2", obj, "sse4.2", &error_abort);

    if (xcc->cpu_def) {
        x86_cpu_load_def(cpu, xcc->cpu_def, &error_abort);
    }    
}
```

可以看到在函数的最后调用了x86_cpu_load_def(cpu, xcc->cpu_def, &error_abort)函数，在x86_cpu_load_def()函数中有for循环：
```C
static void x86_cpu_load_def(X86CPU *cpu, X86CPUDefinition *def, Error **errp)
{
	...
	for (w = 0; w < FEATURE_WORDS; w++) {
		env->features[w] = def->features[w];
	}
	...
}
```
会使用def->features[]数组来初始化env->features[]数组。


# 2) CPU expansion
```C
/* Expand CPU configuration data, based on configured features
 * and host/accelerator capabilities when appropriate.
 */
static void x86_cpu_expand_features(X86CPU *cpu, Error **errp)
{
    CPUX86State *env = &cpu->env;
    FeatureWord w;
    GList *l;
    Error *local_err = NULL;

    /*TODO: Now cpu->max_features doesn't overwrite features
     * set using QOM properties, and we can convert
     * plus_features & minus_features to global properties
     * inside x86_cpu_parse_featurestr() too.
     */
    if (cpu->max_features) {
        for (w = 0; w < FEATURE_WORDS; w++) {
            /* Override only features that weren't set explicitly
             * by the user.
             */
            env->features[w] |=
                x86_cpu_get_supported_feature_word(w, cpu->migratable) &
                ~env->user_features[w] & \
                ~feature_word_info[w].no_autoenable_flags;
        }
    }

    for (l = plus_features; l; l = l->next) {
        const char *prop = l->data;
        object_property_set_bool(OBJECT(cpu), true, prop, &local_err);
        if (local_err) {
            goto out;
        }
    }

    for (l = minus_features; l; l = l->next) {
        const char *prop = l->data;
        object_property_set_bool(OBJECT(cpu), false, prop, &local_err);
        if (local_err) {
            goto out;
        }
    }

    if (!kvm_enabled() || !cpu->expose_kvm) {
        env->features[FEAT_KVM] = 0;
    }

    x86_cpu_enable_xsave_components(cpu);

    /* CPUID[EAX=7,ECX=0].EBX always increased level automatically: */
    x86_cpu_adjust_feat_level(cpu, FEAT_7_0_EBX);
    if (cpu->full_cpuid_auto_level) {
        x86_cpu_adjust_feat_level(cpu, FEAT_1_EDX);
        x86_cpu_adjust_feat_level(cpu, FEAT_1_ECX);
        x86_cpu_adjust_feat_level(cpu, FEAT_6_EAX);
        x86_cpu_adjust_feat_level(cpu, FEAT_7_0_ECX);
        x86_cpu_adjust_feat_level(cpu, FEAT_8000_0001_EDX);
        x86_cpu_adjust_feat_level(cpu, FEAT_8000_0001_ECX);
        x86_cpu_adjust_feat_level(cpu, FEAT_8000_0007_EDX);
        x86_cpu_adjust_feat_level(cpu, FEAT_8000_0008_EBX);
        x86_cpu_adjust_feat_level(cpu, FEAT_C000_0001_EDX);
        x86_cpu_adjust_feat_level(cpu, FEAT_SVM);
        x86_cpu_adjust_feat_level(cpu, FEAT_XSAVE);
        /* SVM requires CPUID[0x8000000A] */
        if (env->features[FEAT_8000_0001_ECX] & CPUID_EXT3_SVM) {
            x86_cpu_adjust_level(cpu, &env->cpuid_min_xlevel, 0x8000000A);
        }

        /* SEV requires CPUID[0x8000001F] */
        if (sev_enabled()) {
            x86_cpu_adjust_level(cpu, &env->cpuid_min_xlevel, 0x8000001F);
        }
    }

    /* Set cpuid_*level* based on cpuid_min_*level, if not explicitly set */
    if (env->cpuid_level == UINT32_MAX) {
        env->cpuid_level = env->cpuid_min_level;
    }
    if (env->cpuid_xlevel == UINT32_MAX) {
        env->cpuid_xlevel = env->cpuid_min_xlevel;
    }
    if (env->cpuid_xlevel2 == UINT32_MAX) {
        env->cpuid_xlevel2 = env->cpuid_min_xlevel2;
    }

out:
    if (local_err != NULL) {
        error_propagate(errp, local_err);
    }
}
```
在代码的第17行开始的for循环中，针对每一个FeatureWord，设置env->feature[w]为`x86_cpu_get_supported_feature_word(w, cpu->migratable) & ~env->user_features[w] & ~feature_word_info[w].no_autoenable_flags`:
* `x86_cpu_get_supported_feature_word(w, cpu->migratable)`:QEMU会调用`kvm_arch_get_supported_cpuid()`或者`kvm_arch_get_supported_msr_features()`向KVM查询支持的feature_word。
 然后根据cpu->migrable进行进一步处理，如果cpu->migrable为true,则将刚才获取的feature_word & feature_word_migratable;其feature_word_migratable为对应的FeatureWord中migratable_flags表示的bit位和FeatureWord.feat_names[i]有效并且不为unmigratable_flags的bit位（**即为对应的FeatureWord.feat_names[]中有效bits去掉unmigratable的bits**）。
* ~env->user_features[w]: 去掉env->user_features[w]bit
* ~feature_word_info[w].no_autoenable_flags; 取消对应的FeatureWord中不自动enable的bit.
由上面3个组成的FeatureWord于CPU Model中的env->features[w]相于，进行expand。

然后根据`plus_features[]`数组和`minus_features[]`数组进行进一步的调整。
最后更新env->cpuid_level。

# 3) CPUID filtering
```C
/*
 * Finishes initialization of CPUID data, filters CPU feature
 * words based on host availability of each feature.
 *
 * Returns: 0 if all flags are supported by the host, non-zero otherwise.
 */
static int x86_cpu_filter_features(X86CPU *cpu)
{
    CPUX86State *env = &cpu->env;
    FeatureWord w;
    int rv = 0;

    for (w = 0; w < FEATURE_WORDS; w++) {
        uint32_t host_feat =
            x86_cpu_get_supported_feature_word(w, false);
        uint32_t requested_features = env->features[w];
        env->features[w] &= host_feat;
        cpu->filtered_features[w] = requested_features & ~env->features[w];
        if (cpu->filtered_features[w]) {
            rv = 1;
        }
    }

    if ((env->features[FEAT_7_0_EBX] & CPUID_7_0_EBX_INTEL_PT) &&
        kvm_enabled()) {
        KVMState *s = CPU(cpu)->kvm_state;
        uint32_t eax_0 = kvm_arch_get_supported_cpuid(s, 0x14, 0, R_EAX);
        uint32_t ebx_0 = kvm_arch_get_supported_cpuid(s, 0x14, 0, R_EBX);
        uint32_t ecx_0 = kvm_arch_get_supported_cpuid(s, 0x14, 0, R_ECX);
        uint32_t eax_1 = kvm_arch_get_supported_cpuid(s, 0x14, 1, R_EAX);
        uint32_t ebx_1 = kvm_arch_get_supported_cpuid(s, 0x14, 1, R_EBX);

        if (!eax_0 ||
           ((ebx_0 & INTEL_PT_MINIMAL_EBX) != INTEL_PT_MINIMAL_EBX) ||
           ((ecx_0 & INTEL_PT_MINIMAL_ECX) != INTEL_PT_MINIMAL_ECX) ||
           ((eax_1 & INTEL_PT_MTC_BITMAP) != INTEL_PT_MTC_BITMAP) ||
           ((eax_1 & INTEL_PT_ADDR_RANGES_NUM_MASK) <
                                           INTEL_PT_ADDR_RANGES_NUM) ||
           ((ebx_1 & (INTEL_PT_PSB_BITMAP | INTEL_PT_CYCLE_BITMAP)) !=
                (INTEL_PT_PSB_BITMAP | INTEL_PT_CYCLE_BITMAP)) ||
           (ecx_0 & INTEL_PT_IP_LIP)) {
            /*
             * Processor Trace capabilities aren't configurable, so if the
             * host can't emulate the capabilities we report on
             * cpu_x86_cpuid(), intel-pt can't be enabled on the current host.
             */
            env->features[FEAT_7_0_EBX] &= ~CPUID_7_0_EBX_INTEL_PT;
            cpu->filtered_features[FEAT_7_0_EBX] |= CPUID_7_0_EBX_INTEL_PT;
            rv = 1;
        }
    }

    return rv;
}
```
第15行查询KVM物理host支持的该FeatureWord的值（并且不去掉unmigratable的bit）;
第17行根据物理host支持的FeatureWord更新env->features[w]；
第18行设置cpu->filtered_features[w]为被物理Host过滤掉的feature,即物理平台不支持的feature.
第24行开始为对Intel PT feature的特殊处理。






[cpu_model]: qemu中cpu_model的设置.html