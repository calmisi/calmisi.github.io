---
title: QEMU对MSR的处理（一）
date: 2019-02-14 20:36:22
tags:
- MSR
categories: QEMU
---

在kvm-all.c里面的kvm_init()函数，
会调用kvm_arch_init()函数，
1. kvm_get_supported_msrs();
2. kvm_get_supported_feature_msrs();

<!-- more -->

# 1. kvm_get_supported_msrs()
1. QEMU会使用kvm_ioctl(KVM_GET_MSR_INDEX_LIST)，然后KVM会返回当前kvm模块支持的msr_to_save[]数组和emulated_msr[]数组（[参见KVM对MSR寄存器的处理（三）][kvm_msr_1]）给QEMU。
2. 但QEMU并不保存这些MSR的内容，而是根据返回的支持的MSR来设置一些全局的静态变量来表示是否有对应的MSR。
3. 如has_msr_star, has_msr_hsave_pa, ... , has_msr_arch_capabs等，这里QEMU是记录其关心的MSR，应该有部分KVM返回的MSR是没有用的（本人没有统计）。

# 2. kvm_get_supported_feature_msrs()
1. QEMU首先kvm_check_extension(KVM_CAP_GET_MSR_FEATURES)来查询KVM是否支持查询KVM_GET_MSR_FEATURE_INDEX_LIST;
2. 然后调用kvm_ioctl(KVM_GET_MSR_FEATURE_INDEX_LIST)获取kvm的msr_based_features[]数组的大小；
3. 然后为全局变量kvm_feature_msrs申请对需要的空间来存储从KVM获得msr_based_features[]数组；
4. 并再一次调用kvm_ioctl(KVM_GET_MSR_FEATURE_INDEX_LIST)，最后将获取的msr_based_features[]数组存到全局变量kvm_feature_msrs变量中。

> 注意：
> **我们可以看到，kvm_get_supported_msrs()获取到相应的msr数组后，没有保存对应的index,而是取其需要的msr来设置对应的全局标志变量,不一定会用到所有的KVM返回的MSR；
> 而kvm_get_supported_feature_msrs()会获取KVM的msr_based_features[]数组，并且不做过滤，直接保存。**

# 3. kvm_arch_get_supported_msr_feature(KVMState *s, uint32_t index)
该函数负责向KVM获取指定的feature MSR的值。

1. 检查全局变量kvm_feature_msrs是否为NULL，即为上一小节中kvm_get_supported_feature_msrs()初始化的；如果为NULL，则说明host不支持feature MSRs;那么直接return 0。
2. 依次遍历kvm_feature_msrs中每一个MSR，找到目标index所对应的MSR，如果在kvm_feature_msrs中没有，那么直接return 0。
3. 到了这一步，说明需要查询的feature MSR存在，即KVM支持；那么调用kvm_ioctl(KVM_GET_MSRS)获取该MSR的值。


# 4. x86_cpu_get_supported_feature_word(FeatureWord w, bool migratable_only)
第一个参数FeatureWord w为枚举变量，该函数会根据w，取出静态全局数组Static FeatureWordInfo feature_word_info[FEATURE_WORDS]中对应的FeatureWordInfo条目。
然后判断该FeatureWordInfo->type是CPUID_FEATURE_WORD还是MSR_FEATURE_WORD,然后调用不同的函数来更新对应的FeatureWordInfo.
 
* CPUID_FEATURE_WORD
会调用kvm_arch_get_supported_cpuid(kvm_state, wi->cpuid.eax, wi->cpuid.ecx, wi->cpuid.reg);
1. 在该函数首先调用get_supported_cpuid()函数，查询KVM支持哪些CPUID。


* MSR_FEATURE_WORD
会调用kvm_arch_get_supported_msr_feature(kvm_state, wi->msr.index);







[kvm_msr_1]: kvm对MSR寄存器的处理（三）.html