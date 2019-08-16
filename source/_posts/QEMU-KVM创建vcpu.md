---
title: QEMU-KVM创建vcpu
date: 2019-06-20 21:24:12
tags:
- QEMU-KVM
categories:
- KVM
- QEMU
---

本文介绍QEMU创建虚拟机后，如何为虚拟机创建vcpu.

<!-- more -->

# QEMU
在`qemu/target/i386/cpu.c`中，x86_cpu_common_class_init()函数中，
```C
...
device_class_set_parent_realize(dc, x86_cpu_realizefn, &xcc->parent_realize);
device_class_set_parent_unrealize(dc, x86_cpu_unrealizefn, &xcc->parent_unrealize);
...
```
其中`x86_cpu_realizefn()`函数主要做了以下几件事：
1. x86_cpu_expand_features(cpu, &local_err);
2. x86_cpu_filter_features(cpu);
3. cpu->check_cpuid || cpu->enforce_cpuid的检查
4. guest physical bits set
5. Cache information initialization
6. cpu_exec_realizefn(cs, &local_err);
7. mce_init(cpu);
8. qemu_init_vcpu(cs);
9. x86_cpu_apic_realize(cpu, &local_err);
10. cpu_reset(cs);
11. xcc->parent_realize(dev, &local_err);

其中第8项`qemu_init_vcpu()`函数如下：
```C
void qemu_init_vcpu(CPUState *cpu)
{
    cpu->nr_cores = smp_cores;
    cpu->nr_threads = smp_threads;
    cpu->stopped = true;

    if (!cpu->as) {
        /* If the target cpu hasn't set up any address spaces itself,
         * give it the default one.
         */
        cpu->num_ases = 1; 
        cpu_address_space_init(cpu, 0, "cpu-memory", cpu->memory);
    }    

    if (kvm_enabled()) {
        qemu_kvm_start_vcpu(cpu);
    } else if (hax_enabled()) {
        qemu_hax_start_vcpu(cpu);
    } else if (hvf_enabled()) {
        qemu_hvf_start_vcpu(cpu);
    } else if (tcg_enabled()) {
        qemu_tcg_init_vcpu(cpu);
    } else if (whpx_enabled()) {
        qemu_whpx_start_vcpu(cpu);
    } else {
        qemu_dummy_start_vcpu(cpu);
    }    

    while (!cpu->created) {
        qemu_cond_wait(&qemu_cpu_cond, &qemu_global_mutex);
    }    
}
```
1. 其根据`qemu-system-x86_64`的启动参数`-smp`指定的cores, threads初始化cpu相关的结构。
2. 然后初始化其address space；
3. 然后根据不同的平台支持调用start_vcpu()函数，在kvm平台中会调用`qemu_kvm_start_vcpu(cpu)`函数；

下面我们来看`qemu_kvm_start_vcpu()`函数的实现：
```C
static void qemu_kvm_start_vcpu(CPUState *cpu)
{
    char thread_name[VCPU_THREAD_NAME_SIZE];

    cpu->thread = g_malloc0(sizeof(QemuThread));
    cpu->halt_cond = g_malloc0(sizeof(QemuCond));
    qemu_cond_init(cpu->halt_cond);
    snprintf(thread_name, VCPU_THREAD_NAME_SIZE, "CPU %d/KVM",
             cpu->cpu_index);
    qemu_thread_create(cpu->thread, thread_name, qemu_kvm_cpu_thread_fn,
                       cpu, QEMU_THREAD_JOINABLE);
}
```
可见其调用了`qemu_thread_create()`创建了qemu thread，这些qemu threads会运行`qemu_kvm_cpu_thread_fn()`函数。
`qemu_thread_create()`函数使用的是pthread,感兴趣的可以去`util/qemu-thread-posix.c`文件看其实现；或者HANDLE hThread(util/qemu-thread-win32.c)。
```C
static void *qemu_kvm_cpu_thread_fn(void *arg)
{
    CPUState *cpu = arg;
    int r;

    rcu_register_thread();

    qemu_mutex_lock_iothread();
    qemu_thread_get_self(cpu->thread);
    cpu->thread_id = qemu_get_thread_id();
    cpu->can_do_io = 1;
    current_cpu = cpu;

    r = kvm_init_vcpu(cpu);
    if (r < 0) {
        error_report("kvm_init_vcpu failed: %s", strerror(-r));
        exit(1);
    }

    kvm_init_cpu_signals(cpu);

    /* signal CPU creation */
    cpu->created = true;
    qemu_cond_signal(&qemu_cpu_cond);

    do {
        if (cpu_can_run(cpu)) {
            r = kvm_cpu_exec(cpu);
            if (r == EXCP_DEBUG) {
                cpu_handle_guest_debug(cpu);
            }
        }
        qemu_wait_io_event(cpu);
    } while (!cpu->unplug || cpu_can_run(cpu));

    qemu_kvm_destroy_vcpu(cpu);
    cpu->created = false;
    qemu_cond_signal(&qemu_cpu_cond);
    qemu_mutex_unlock_iothread();
    rcu_unregister_thread();
    return NULL;
}
```
`qemu_kvm_cpu_thread_fn()`线程函数首先会调用`kvm_init_vcpu()`函数初始化/创建vcpu,然后发送信号(具体作用还不清除，to be figured out)，然后在while循环中不停的调用kvm_vcpu_exce(cpu)，
如果返回结果是EXCP_DEBUG,那么调用`cpu_handle_guest_debug()`进行userspace debug。
当从while循环中退出后，需要调用`qemu_kvm_destroy_vcpu()`函数来destroy之前创建的vcpu。同样也要发送信号，最后调用`rcu_unregister_thread()`函数unregister该线程。

下面，我们来分析`kvm_init_vcpu()`函数是如何初始化/创建vcpu的：	
```C
int kvm_init_vcpu(CPUState *cpu)
{
    KVMState *s = kvm_state;
    long mmap_size;
    int ret; 

    DPRINTF("kvm_init_vcpu\n");

    ret = kvm_get_vcpu(s, kvm_arch_vcpu_id(cpu));
    if (ret < 0) { 
        DPRINTF("kvm_create_vcpu failed\n");
        goto err; 
    }    

    cpu->kvm_fd = ret; 
    cpu->kvm_state = s; 
    cpu->vcpu_dirty = true;

    mmap_size = kvm_ioctl(s, KVM_GET_VCPU_MMAP_SIZE, 0);
    if (mmap_size < 0) { 
        ret = mmap_size;
        DPRINTF("KVM_GET_VCPU_MMAP_SIZE failed\n");
        goto err; 
    }    

    cpu->kvm_run = mmap(NULL, mmap_size, PROT_READ | PROT_WRITE, MAP_SHARED,
                        cpu->kvm_fd, 0);
    if (cpu->kvm_run == MAP_FAILED) {
        ret = -errno;
        DPRINTF("mmap'ing vcpu state failed\n");
        goto err; 
    }    

    if (s->coalesced_mmio && !s->coalesced_mmio_ring) {
        s->coalesced_mmio_ring =
            (void *)cpu->kvm_run + s->coalesced_mmio * PAGE_SIZE;
    }    

    ret = kvm_arch_init_vcpu(cpu);
err:
    return ret; 
}
```
`kvm_init_vcpu()`函数

最后调用`kvm_arch_init_vcpu()`函数初始化vcpu，其具体为：
1. 设置env->tsc_khz,使用kvm_arch_set_tsc_khz()函数或者kvm_vcpu_ioctl(cs, KVM_GET_TSC_KHZ);
2. 设置pv CPUIDs,(if(hyperv_enabled(cpu)));
3. if(cpu->expose_kvm)，则设置guest中KVM related CPUID的值，0x4000 0000，
4. 填充cpuid_data结构，最后调用KVM_SET_CPUID2向guest设置cpuid。


