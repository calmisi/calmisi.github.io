---
title: QEMU对MSR的处理（二）
date: 2019-02-15 22:14:53
tags:
- MSR
categories:
- QEMU
---

在[QEMU对MSR的处理（一）][qemu_msr_1]中，我们介绍了qemu中有kvm_get_supported_msrs()函数。该函数会调用kvm_ioctl(KVM_GET_MSR_INDEX_LIST)，kvm会向qemu返回其定义的msr_to_save[]和emulated_msrs[]数组。
之后qemu就可以调用vcpu_ioctl(KVM_GET_MSRS)和vcpu_ioctl(KVM_SET_MSRS)来get/set之前获取到的suppoted_msrs[]。

<!-- more -->


# kvm对KVM_GET_MSRS, KVM_SET_MSRS的处理
```C
		case KVM_GET_MSRS: {
                int idx = srcu_read_lock(&vcpu->kvm->srcu);
                r = msr_io(vcpu, argp, do_get_msr, 1);
                srcu_read_unlock(&vcpu->kvm->srcu, idx);
                break;
        }    
        case KVM_SET_MSRS: {                                                                                                              
                int idx = srcu_read_lock(&vcpu->kvm->srcu);
                r = msr_io(vcpu, argp, do_set_msr, 0);
                srcu_read_unlock(&vcpu->kvm->srcu, idx);
                break;
        }
```
可以看到会调用do_get_msr()和do_set_msr()函数。
```C
static int do_get_msr(struct kvm_vcpu *vcpu, unsigned index, u64 *data)                                                                   
{
        struct msr_data msr;
        int r;

        msr.index = index;
        msr.host_initiated = true;
        r = kvm_get_msr(vcpu, &msr);
        if (r)
                return r;

        *data = msr.data;
        return 0;
}

static int do_set_msr(struct kvm_vcpu *vcpu, unsigned index, u64 *data)
{
        struct msr_data msr;

        msr.data = *data;
        msr.index = index;
        msr.host_initiated = true;
        return kvm_set_msr(vcpu, &msr);
}
```
对应的，又会调用kvm_get_msr()和kvm_set_msr()函数。
这两个函数干了什么，具体地可以参考[kvm中handle_rdmsr][kvm_msr_2]和[kvm中handle_wrmsr][kvm_msr_4]这两篇文章。

> **总结：QEMU和KVM调用get/set msr的区别
> QEMU和KVM都可以get/set guest的msr。
> QEMU调用get msr是主动查询guest中相应的msr的值，set_msr是修改guest中msr的值。
> KVM是对rdmsr和wrmsr指令的模拟（如果这两条指令会vm exit）。
> 具体地，QEMU和KVM最后调用的函数是一样的，都是vmx_get/set_msr()，唯一的区别就是：
> QEMU将msr.host_initiated = true;
> KVM将msr.host_initiated = false;**






[qemu_msr_1]: QEMU对MSR的处理（一）.html
[kvm_msr_2]: kvm对guest中MSR寄存器的处理（二）-rdmsr.html
[kvm_msr_4]: kvm对guest中MSR寄存器的处理（四）-wrmsr.html