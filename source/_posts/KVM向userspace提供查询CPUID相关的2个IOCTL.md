---
title: KVM向userspace提供查询CPUID相关的2个IOCTL
date: 2019-03-16 17:17:32
tags:
- CPUID
categories:
- KVM
- QEMU
---

本文介绍KVM的两个IOCTL，其都为VM level的ioctl：`KVM_GET_SUPPORTED_CPUID`和`KVM_GET_EMULATED_CPUID`.
userspace的VMM程序，可以调用`KVM_GET_SUPPORTED_CPUD`来查询host kvm可以向guest支持哪些CPUID；可以调用`KVM_GET_SUPPORTED_CPUID`来查询kvm对guest支持的哪些CPUID是模拟的。

<!-- more -->

在x86.c中，kvm_arch_dev_ioctl中关于KVM_GET_SUPPORTED_CPUID和KVM_GET_EMULATED_CPUID的实现为：
```C
long kvm_arch_dev_ioctl(struct file *filp,
                        unsigned int ioctl, unsigned long arg) 
{
        void __user *argp = (void __user *)arg;
        long r;
	
	...
        case KVM_GET_SUPPORTED_CPUID:
        case KVM_GET_EMULATED_CPUID: {
                struct kvm_cpuid2 __user *cpuid_arg = argp;
                struct kvm_cpuid2 cpuid;

                r = -EFAULT;
                if (copy_from_user(&cpuid, cpuid_arg, sizeof(cpuid)))
                        goto out; 

                r = kvm_dev_ioctl_get_cpuid(&cpuid, cpuid_arg->entries,        
                                            ioctl);
                if (r)
                        goto out; 

                r = -EFAULT;
                if (copy_to_user(cpuid_arg, &cpuid, sizeof(cpuid)))
                        goto out; 
                r = 0; 
                break;
        }
	...
out:
	return r;
```
我们可以看到KVM中关于KMV_GET_SUPPORTED_CPUID和KVM_GET_EMULATED_CPUID这2个IOCTL的实现是公共的。
首先cory_from_user()获取userspace传进来的参数；
然后调用kem_dev_ioctl_get_get_cpuid()，如果返回值不为0，则说明发生了错误，那么直接return 返回值；
否则调用copy_to_user将获得的CPUID结构返回给userspace。
所以，如果结果不为0，那么该2个IOCTL调用都是失败的。

# kvm_dev_ioctl_get_cpuid()
在上一部函数调用为kvm_dev_ioctl_get_cpuid(&cpuid, cpuid_arg->entries, ioctl);
我们可以看到第一个参数&cpuid： 从userspace复制过来的参数struct kvm_cpuid2的机构体，传进来的该结构体只有argp->nent是有效的，其表示argp->entries[]数组的大小。
第二个参数cpuid_arg->entries为userspace传入的argp->entries[]数组的起始指针；
第三个参数为IOCTL的类型，为了区分是KMV_GET_SUPPORTED_CPUID和KVM_GET_EMULATED_CPUID；
```C
/* for KVM_SET_CPUID2 */
struct kvm_cpuid2 {                                                             
        __u32 nent;
        __u32 padding;
        struct kvm_cpuid_entry2 entries[0];
};
```
具体地，我们看kvm_dev_ioctl_get_cpuid()函数做了什么。
1. 如果userspace传进来cpuid->nent < 1，则return -E2BIG;
2. 如果userspace传进来的cpuid->entries[]的大小cpuid->nent大于KVM_MAX_CPUID_ENTRIES,那么更新cpuid->nent为KVM_MAX_CPUID_ENTRIES;
3. 会调用sanity_check_entries(entries, cpuid->nent, type)
   该函数检查userspace传递来的entries数组中的每一个entry->padding是否正确或者为0.
   如果padding有问题，返回true，则直接return -EINVAL;
   只检查KVM_GET_EMULATED_CPUID，对KVM_GET_SUPPORTED_CPUID不检查，直接返回false.
4. 然后动态vzalloc cpuid->nent数量的kvm_cpuid_entry2数组,如果分配不成功则return -ENOMEM;
5. 进入for循环，根据函数开头定义的kvm_cpuid_param param[]数组进行CPUID的读取。
   获取每一个param[i],
 -  判断ent->qualifier && !ent->qualifier(ent)，则跳过这个param[i];
    根据上面定义，只有在.func=0xC0000000的时候，有qualifier,该qualifie判断当前的CPU是不是centaur_cpu，
    由于我们考虑intel cpu,该ent->qualifier(ent)返回false,则continue跳过该param[i]，
    只有是centaur cpu的时候，才会执行该param[i]条目，读取相关的CPUID
 -  调用do_cpuid_ent()
/* is important!!!!
 * entry:需要填充的vzalloc的kvm_cpuid_entry2 cpuid_entries[nent]号成员，nent初始值为0
 * func:cpuid的主leaf
 * idx:cpudi的次leaf
 * nent:当前填充/操作的kvm_cpuid_entry2 cpuid_entries[nent]中nent的指针
 * maxnent:cpuid_entries的size
 * type:KVM_GET_EMULATED_CPUID或者KVM_GET_SUPPORTED_CPUID
	static int do_cpuid_ent(struct kvm_cpuid_entry2 *entry, u32 func,
		                u32 idx, int *nent, int maxnent, unsigned int type)
	
	{
		if (type == KVM_GET_EMULATED_CPUID)
			return __do_cpuid_ent_emulated(entry, func, idx, nent, maxnent);
	
		return __do_cpuid_ent(entry, func, idx, nent, maxnent);
	} 
 */

回去看笔记，A4纸
```C
int kvm_dev_ioctl_get_cpuid(struct kvm_cpuid2 *cpuid,
                            struct kvm_cpuid_entry2 __user *entries,
                            unsigned int type)
{
        struct kvm_cpuid_entry2 *cpuid_entries;
        int limit, nent = 0, r = -E2BIG, i;
        u32 func;
        static const struct kvm_cpuid_param param[] = {
                { .func = 0, .has_leaf_count = true },
                { .func = 0x80000000, .has_leaf_count = true },
                { .func = 0xC0000000, .qualifier = is_centaur_cpu, .has_leaf_count = true },
                { .func = KVM_CPUID_SIGNATURE },
                { .func = KVM_CPUID_FEATURES },
        };

        if (cpuid->nent < 1)
                goto out;
        if (cpuid->nent > KVM_MAX_CPUID_ENTRIES)
                cpuid->nent = KVM_MAX_CPUID_ENTRIES;

        if (sanity_check_entries(entries, cpuid->nent, type))
                return -EINVAL;
	
	r = -ENOMEM;
        cpuid_entries = vzalloc(array_size(sizeof(struct kvm_cpuid_entry2),
                                           cpuid->nent));
        if (!cpuid_entries)
                goto out;

	r = 0;
        for (i = 0; i < ARRAY_SIZE(param); i++) {
                const struct kvm_cpuid_param *ent = &param[i];

                if (ent->qualifier && !ent->qualifier(ent))
                        continue;

                r = do_cpuid_ent(&cpuid_entries[nent], ent->func, ent->idx,
                                &nent, cpuid->nent, type);
                if (r)
                        goto out_free;

                if (!ent->has_leaf_count)
                        continue;

                limit = cpuid_entries[nent - 1].eax;
                for (func = ent->func + 1; func <= limit && nent < cpuid->nent && r == 0; ++func)
                        r = do_cpuid_ent(&cpuid_entries[nent], func, ent->idx,
                                     &nent, cpuid->nent, type);

                if (r)
                        goto out_free;
        }

        r = -EFAULT;
        if (copy_to_user(entries, cpuid_entries,
                         nent * sizeof(struct kvm_cpuid_entry2)))
                goto out_free;
        cpuid->nent = nent;
        r = 0;

out_free:
        vfree(cpuid_entries);
out:
        return r;
}


