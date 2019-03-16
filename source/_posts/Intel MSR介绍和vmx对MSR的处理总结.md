---
title: Intel MSR介绍和vmx对MSR的处理总结
date: 2019-02-01 18:59:29
tags:
- MSR
categories:
- KVM
---

# MSR寄存器的介绍
The scope of an MSR defines the set of processors that access the same MSR with RDMSR and WRMSR.
Thread-scope MSRs are unique to every logical processor; Core-scope MSRs are shared by the threads in the same core; similarly for module-scope, die-scope, and package-scope.
When a processor package contains a single die, die-scope and package-scope are synonymous. When a package contains multiple die, they are distinct。
MSR寄存器的作用域可以是thread，即单个超线程，可以是每个core有单独的，也可以是一个die共用，还是是整个package。
<!-- more -->

## Architectural MSRs
A subset of MSRs and associated bit fields, which do not change on future processor generations, are now considered architectural MSRs. 这些architectural MSRs都有`IA32_`前缀。

## wrmsr指令：
wrmsr指令将EDX:EAX的值写入由ECX指定的64-bit的MSR寄存器中。
其中EDX写入MSR的高32位，EAX写入MSR的低32位。RAX,RCX,RDX的高32位都被忽略。
<!-- more -->

1. 该指令只能在CPL = 0 执行，否则会产生#GP;
2. 如果往Reserved or unimplemented MSR地址写，也会产生一个#GP
3. undefined or reserved bits in an MSR should be set to values previously read,否则会产生#GP；
4. The CPUID instruction should be used to determine whether MSRs are suppoted (CPUID.01H:EDX[5]=1) before using this instruction.
   CPUID.01H:EDX[5]=1表示支持rdmsr/wrmsr指令。

## rdmsr指令：
rdmsr指令将由ECX指定的64-bit的MSR寄存器的值读取到EDX:EAX。
其中高32位写入EDX,低32位写入EAX。RAX,RCX,RDX的高32位都被忽略。
If fewer than 64 bits are implemented in the MSR being read, the values returned to EDX:EAX in unimplemented bit locations are undefined。

1. 该指令只能在CPL = 0 或 real-address mode执行，否则会产生#GP;
2. 如果ECX为一个reserved or unimplemented MSR的index,也会产生一个#GP;
3. The CPUID instruction should be used to determine whether MSRs are suppoted (CPUID.01H:EDX[5]=1) before using this instruction.

## VMM对MSR寄存器的处理
理论上，guest中MSR寄存器的值应该完全独立host,但是物理硬件只有一份，即guest和host分时复用这些MSR寄存器硬件。
所以正确的做法就是，当vm entry的时候，把host中MSR寄存器的值存起来，加载guest MSR寄存器的值。当vm exit的时候，把guest中MSR寄存器的值都存起来，加载host MSR寄存器的值。

然后就涉及到guest中执行rdmsr和wrmsr指令的问题。
如果在VM entry/exit时将guest和host的MSR全部都切换，那么就可以在guest执行这两个指令的时候不用vm exit，直接访问硬件。
可是这里涉及到VMM对guest的模拟，不能直接让guest访问硬件MSR。比如：
1. 某个MSR的bit位置1表示启用某个功能，现在不想让guest使用这个功能，那么就必须在guest写这个MSR的时候，trap出来，VMM做相应的处理；
2. 某个MSR是read only的，kernel会读取其中某个bit来决定是否有某个功能，现在VMM不想让guest有这个功能，那么在rdmsr该MSR时如果直接读物理硬件，就可以读到有这个feature，而是应该trap出来，VMM返回给guest一个假的值，告诉它没有这个功能。

所以，下一节[Intel VMCS中对MSR的支持](#Intel-VMCS中对MSR的支持)中提到的第3点：**MSR bitmap**。
其可以控制rdmsr和wrmsr指令是否会触发vm exit。如果不触发vm exit，那么可以直接访问hardware；如果触发vm exit，那么会trap出来到vmm，进行指令的模拟。
关于MSR bitmap的具体介绍可以参考[kvm对guest中MSR寄存器的处理（一）][msr_bitmap]。本文后面的章节[Use MSR bitmaps和MSR-bitmap address](#2-Use-MSR-bitmaps和MSR-bitmap-address)具体分析了KVM中如何使用MSR bitmap，在KVM中默认是启用MSR bitmap的（如果host cpu支持），并且除了`MSR_IA32_TSC`、`MSR_FS_BASE`、 `MSR_GS_BASE`、 `MSR_KERNEL_GS_BASE`、 `MSR_IA32_SYSENTER_CS`、 `MSR_IA32_SYSENTER_ESP`、 `MSR_IA32_SYSENTER_EIP`之外，其他的MSR在rdmsr/wrmsr时都会vm exit到KVM。
那么在KVM中大多数的MSR都会从guest trap出去，KVM需要对rdmsr/wrmsr进行模拟。
现在假设情况是这样的，在创建VM的时候，由于没有VM entry进入到guest过。KVM需要给guset MSRs设置默认值，然后第一次VM entry到guest，此时MSR中会load KVM给guest设置的默认值，并将host的值保存起来（这个过程可以使用CPU提供的硬件load/store,也可以KVM软件实现）；然后CPU给guest运行，当guest发生vm exit时，又会把guest的MSRs都保存起来，并load host MSRs的值。
> 注意：
> 在KVM的代码中，其vcpu_vmx, vcpu_vmx->arch等cpu的结构体中都有域直接记录guest的相关MSR的值，这个方便KVM直接获取MSR的值。
> 所以在vm entry/vm exit的时候，要将这些域里面的值load到MSR,或者将MSR的值store到这些域。如果采用硬件辅助VMCS guest state或者vm-entry msr-store），这将这些值写到对应的VMCS区域中；其他没有使用硬件辅助的，则需要KVM自己在vm entry/vm exit的时候把guest/host MSR的值写到对应的MSR中。

所以下面就得分情况讨论：
- MSR是只读的：在VM entry/ VM exit的时候，不用切换host/guest的值，因为其不能写。所以只用在KVM中相关的结构（e.g., vcpu_vmx->, vcpu_vmx->arch->）中软件维持其值，当guest rdmsr时，trap出来返回kvm 模拟的值。这里需要注意：当host有某个feature的是，可以返回给guest没有这个feature；反过来的话，就需要KVM模拟这个feature。
- MSR是读写的：在VM entry/ VM exit的时候，就需要切换host/guest的值，即需要将guest/host的MSRs的值写到物理的MSR中去。


---

# Intel VMCS中对MSR的支持
VMCS的data域可以分为Guest-state area, Host-state area, VM-execution control fields, VM-exit control fields, VM-entry control fields, VM-exit information fields.
其中与MSR相关的有以下这些：
1. Guest-state area -> Guest Register State -> following MSRs
2. Host-state are -> following MSRs
3. VM-execution control fields -> Primary processor-based vm-exectuion controls.use MSR bitmaps(bit 28)
   VM-execution control fields -> MSR-bitmap address
4. VM-exit Control fields -> VM-exit controls.Save debug controls(bit 2)
   VM-exit Control fields -> VM-exit controls.Load IA32_PERF_GLOBAL_CTRL(bit 12)
   VM-exit Control fields -> VM-exit controls.Save IA32_PAT(bit 18)
   VM-exit Control fields -> VM-exit controls.Load IA32_PAT(bit 19)
   VM-exit Control fields -> VM-exit controls.Save IA32_EFER(bit20)
   VM-exit Control fields -> VM-exit controls.Load IA32_EFER(bit21)
   VM-exit Control fields -> VM-exit controls.Clear IA32_BNDCFGS(bit23)
5. VM-entry Control fileds -> VM-entry controls.Load debug controls(bit 2)
   VM-entry Control fileds -> VM-entry controls.Load IA32_PERF_GLOBAL_CTRL(bit 13)
   VM-entry Control fileds -> VM-entry controls.Load IA32_PAT(bit 14)
   VM-entry Control fileds -> VM-entry controls.Load IA32_EFER(bit 15)
   VM-entry Control fileds -> VM-entry controls.Load IA32_BNDCFGS(bit 16)
6. VM-exit Control fields -> VM-exit MSR-store count
   VM-exit Control fields -> VM-exit MSR-store address
   VM-exit Control fields -> VM-exit MSR-load count
   VM-exit Control fields -> VM-exit MSR-load address
7. VM-entry Control fileds -> VM-entry MSR-load count
   VM-entry Control fileds -> VM-entry MSR-load address



以上列了7点。其中1，2，4，5是相关的。
1，2表示VMCS中有特定的几个MSR寄存器的存储区域，4,5表示可以控制硬件自动将相关的MSR存在对应的MSR区域，不需要软件VMM来保存。
3为msr bitmap,用来控制rdmsr/wrmsr指令在操作具体的MSR时是否会产生VM exit。
6,7 提供了硬件保存host/guest的MSR的能力。

---

## 1. Guest-state area和Host-state area和VM-exit controls和VM-entry controls
在Guest-state和Host-state区域分别会保存一些特殊的MSR寄存器，具体如下：
   
Guest Register State		Host Register State
IA32_DEBUGCTL			
IA32_SYSENTER_CS		IA32_SYSENTER
IA32_SYSENTER_ESP		IA32_SYSENTER_ESP
IA32_SYSENTER_EIP		IA32_SYSENTER_EIP
IA32_PERF_GLOBAL_CTRL(*)	IA32_PERF_GLOBAL_CTRL(*)
IA32_PAT(*)			IA32_PAT(*)
IA32_EFER(*)			IA32_EFER(*)
IA32_BNDCFGS(*)

> 注意：
> Guest-state中，
> IA32_PERF_GLOBAL_CTRL只有在 VM-entry control.load IA32_PERF_GLOBAL_CTRL 可以设置为1时，才存在这个域。
> IA32_PAT只有在 VM-entry control.load IA32_PAT || VM-exit control.save IA32_PAT 可以设置为1时，才存在这个域。
> IA32_EFER只有在 VM-entry control.load IA32_EFER || VM-exit control.save IA32_EFER 可以设置为1时，才存在这个域。
> IA32_BNDCFGS只有 VM-entry control.load IA32_BNDCFGS || VM-exit control.clear IA32_BNDCFGS 可以设置为1时，才存在这个域。
>
> Host-state中，
> IA32_PERF_GLOBAL_CTRL 只有在VM-exit control.load IA32_PERF_GLOBAL_CRTL 可以设置为1时，才存在这个域。
> IA32_PAT 只有在VM-exit control.load IA32_PAT 可以设置为1时，才存在这个域。
> IA32_EFER 只有在VM-exit control.load IA32_EFER 可以设置1时，才存在这个域。

---

## 2. Use MSR bitmaps和MSR-bitmap address
### 定义
请参考[MSR bitmap][msr_bitmap]的第一节。
其控制rdmsr/wrmsr指令是否会触发VM exit,即控制guest是直接读写hardware上的MSR，还是VM exit到KVM，让KVM来模拟。

### KVM中的实现<span id="kvm-msr-bitmap"> </span>
KVM会调用vmx_create_vcpu()函数来创建vcpu，在该函数中会调用`alloc_loaded_vmcs(&vmx->vmcs01)`为该vcpu创建loaded_vmcs结构。
```C
int alloc_loaded_vmcs(struct loaded_vmcs *loaded_vmcs)
{
    loaded_vmcs->vmcs = alloc_vmcs(false);
    if (!loaded_vmcs->vmcs)
        return -ENOMEM;

    loaded_vmcs->shadow_vmcs = NULL;
    loaded_vmcs_init(loaded_vmcs);

    if (cpu_has_vmx_msr_bitmap()) {
        loaded_vmcs->msr_bitmap = (unsigned long *)__get_free_page(GFP_KERNEL);
        if (!loaded_vmcs->msr_bitmap)
            goto out_vmcs;
        memset(loaded_vmcs->msr_bitmap, 0xff, PAGE_SIZE);

        if (IS_ENABLED(CONFIG_HYPERV) &&
            static_branch_unlikely(&enable_evmcs) &&
            (ms_hyperv.nested_features & HV_X64_NESTED_MSR_BITMAP)) {
            struct hv_enlightened_vmcs *evmcs =
                (struct hv_enlightened_vmcs *)loaded_vmcs->vmcs;

            evmcs->hv_enlightenments_control.msr_bitmap = 1;
        }
    }

    memset(&loaded_vmcs->host_state, 0, sizeof(struct vmcs_host_state));

    return 0;

out_vmcs:
    free_loaded_vmcs(loaded_vmcs);
    return -ENOMEM;
}
```
1. 在该函数中首先调用alloc_vmcs()为该vcpu创建vmcs结构；
2. 设置loaded_vmcs->shadow_vmcs为NULL;
3. 调用loaded_vmcs_init(loaded_vmcs);
4. **如果当前的cpu支持msr bitmap特性，那么为该loaded_vmcs结构申请一个4k的内存页，作为该vcpu的msr bitmap, 并设置该msr bitmap为全1**;
5. 将loaded_vmcs->host_state置0；

可以看到alloc_loaded_vmcs函数中分配的msr bitmap是全1的。也就是说，所有MSR的read和write都会触发VM EXIT。
> **注意
> 想要rdmsr/wrmsr不触发VM exit,那么需要以下3个条件同时满足：
> 1.use bitmap;
> 2.该MSR在0000 0000H - 0001 FFFFH或者C000 0000H - C000 1FFFFH这个范围
> 3.该MSR对应的bitmap为0**

然后在vmx_create_vcpu()函数中执行了
```C
	msr_bitmap = vmx->vmcs01.msr_bitmap;
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_IA32_TSC, MSR_TYPE_R);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_FS_BASE, MSR_TYPE_RW);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_GS_BASE, MSR_TYPE_RW);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_KERNEL_GS_BASE, MSR_TYPE_RW);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_IA32_SYSENTER_CS, MSR_TYPE_RW);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_IA32_SYSENTER_ESP, MSR_TYPE_RW);
    vmx_disable_intercept_for_msr(msr_bitmap, MSR_IA32_SYSENTER_EIP, MSR_TYPE_RW);
    vmx->msr_bitmap_mode = 0;
```
即取消了`MSR_IA32_TSC` read时候的trap（即不产生VM EXIT）；
取消了`MSR_FS_BASE`、 `MSR_GS_BASE`、 `MSR_KERNEL_GS_BASE`、 `MSR_IA32_SYSENTER_CS`、 `MSR_IA32_SYSENTER_ESP`、 `MSR_IA32_SYSENTER_EIP`, read和write时候的trap；
并设置vmx->msr_bitmap_mode = 0;

然后调用了vmx_vcpu_setup()函数，在该函数中，执行了以下：
```C
static void vmx_vcpu_setup(struct vcpu_vmx *vmx)
{
    ...
    if (cpu_has_vmx_msr_bitmap())
        vmcs_write64(MSR_BITMAP, __pa(vmx->vmcs01.msr_bitmap));
	...

```
将申请的bitmap的物理写入对应的VMCS域，并且在vmx_setup_vmcs_config()函数中默认是把primary vm-execution control.usr msr bitmap(bit 28)置上的。
所以只要CPU支持`Use MSR bitmap`特性，KVM就会使用该特性，并且除了上述的7个MSR外，其他MSR的rdmsr/wrmsr指令都会产生VM EXIT。

--- 

## 3. MSR store/load
### 定义
在VM-exit control fields域中，有VM-exit MSR-store count(32 bits)和VM-exit MSR-store address(64 bits)。
VM-exit MSR-store count 指定在VM exit需要保存的MSR的数量。（intel推荐该数量不要超过512）
VM-exit MSR-store address 指定VM-exit MSR-store area的物理地址。该area一个entries表，其中每个表项16 Bytes,表项的数目由VM-exit MSR-store count指定。注意，当VM-exit MSR-store count不为0时，该address必须为 16-byte对齐。
表项的结构如下图：

同理，VM-exit MSR-load count/VM-exit MSR-load address以及VM-entry MSR-load count/VM-entry MSR-load address。

> **注意
> 可以发现VM exit的时候，有MSR load和store；而在VM entry的时候，只有MSR load。
> 即VM exit的时候，可以设置硬件load host的MSR，以及store guest的MSR。
> 当VM entry的时候，只能设置硬件 load guest的MSR，却不会自动store host的MSR(即需要软件KVM来保存)。**
> 
> **其实，我们只需要两个区域，即两个address。一个保存host MSR, 一个保存guest MSR。
> 当VM exit时，将guest MSRs store到guest区域，从host区域load host MSRs。
> 当VM entry时，将host MSRs store到host区域，从guet区域load guest MSRs。
> 将VM-exit MSR-store address和VM-entry MSR-load address指向 guest MSR区域,并设置好相应的MSR count那么硬件会自动的保存和加载guest的MSR;
> 将VM-exit MSR-load address指向host MSR区域，并设置好相应的MSR count,硬件会自动加载host MSRs，只是需要软件在VM entry的时候手动保存host的MSR值到 host MSR区域。
> KVM是怎么处理的，请看下一小节。
> **

### KVM中的实现
在vmx_vcpu_setup()函数中，
```C
    vmcs_write32(VM_EXIT_MSR_STORE_COUNT, 0);
    vmcs_write32(VM_EXIT_MSR_LOAD_COUNT, 0);
    vmcs_write64(VM_EXIT_MSR_LOAD_ADDR, __pa(vmx->msr_autoload.host.val));
    vmcs_write32(VM_ENTRY_MSR_LOAD_COUNT, 0);
    vmcs_write64(VM_ENTRY_MSR_LOAD_ADDR, __pa(vmx->msr_autoload.guest.val));
```
可以看到，只初始化了VM-exit MSR-load address为vmx->msr_autoload.host.val 和VM-entry MSR-load address为msr_autoload.guest.val。表明KVM只要硬件在VM exit/VM entry的时候自动load host/guest的MSR值，对MSR的保存，KVM自己以软件的方式实现。
其中vmx->msr_autoload结构体中包含两个vmx_msrs的结构体分别表示vm entry时需要autoload的guest MSRs区域；以及vm exit时需要autoload的host MSRs区域。
注意，KVM中定义NR_AUTOLOAD_MSRS为8，即只能用这种方法处理8个MSRs。
```C
struct vcpu_vmx {
	...
	struct msr_autolaod {
		struct vmx_msrs guest;
		struct vmx_msrs host;
	} msr_autoload;
	...
}

#define NR_AUTOLOAD_MSRS 8

struct vmx_msrs {
	unsigned int nr;
	struct vmx_msr_entry val[NR_AUTOLOAD_MSRS];
};

struct vmx_msr_entry {
	u32 index;
	u32 reserved;
	u64 value;
} __aligned(16);
```
可以看到在vmx_vcpu_setup()函数，已经设置好了VM entry和VM exit时，需要自动load的guest/host MSR的address,后面需要做的，就是：
往address中添加需要自动load的msr的entry条目，以及更新MSR-load count VMCS域。
具体地，该功能通过add_atomic_switch_msr()函数实现，具体如下：

```C
static void add_atomic_switch_msr(struct vcpu_vmx *vmx, unsigned msr,
                  u64 guest_val, u64 host_val, bool entry_only)
{
    int i, j = 0;
    struct msr_autoload *m = &vmx->msr_autoload;

    switch (msr) {
    case MSR_EFER:
        if (cpu_has_load_ia32_efer()) {
            add_atomic_switch_msr_special(vmx,
                    VM_ENTRY_LOAD_IA32_EFER,
                    VM_EXIT_LOAD_IA32_EFER,
                    GUEST_IA32_EFER,
                    HOST_IA32_EFER,
                    guest_val, host_val);
            return;
        }
        break;
    case MSR_CORE_PERF_GLOBAL_CTRL:
        if (cpu_has_load_perf_global_ctrl()) {
            add_atomic_switch_msr_special(vmx,
                    VM_ENTRY_LOAD_IA32_PERF_GLOBAL_CTRL,
                    VM_EXIT_LOAD_IA32_PERF_GLOBAL_CTRL,
                    GUEST_IA32_PERF_GLOBAL_CTRL,
                    HOST_IA32_PERF_GLOBAL_CTRL,
                    guest_val, host_val);
            return;
        }
        break;
    case MSR_IA32_PEBS_ENABLE:
        /* PEBS needs a quiescent period after being disabled (to write
         * a record).  Disabling PEBS through VMX MSR swapping doesn't
         * provide that period, so a CPU could write host's record into
         * guest's memory.
         */
        wrmsrl(MSR_IA32_PEBS_ENABLE, 0);
    }

    i = find_msr(&m->guest, msr);
    if (!entry_only)
        j = find_msr(&m->host, msr);

    if (i == NR_AUTOLOAD_MSRS || j == NR_AUTOLOAD_MSRS) {
        printk_once(KERN_WARNING "Not enough msr switch entries. "
                "Can't add msr %x\n", msr);
        return;
    }
    if (i < 0) {
        i = m->guest.nr++;
        vmcs_write32(VM_ENTRY_MSR_LOAD_COUNT, m->guest.nr);
    }
    m->guest.val[i].index = msr;
    m->guest.val[i].value = guest_val;

    if (entry_only)
        return;

    if (j < 0) {
        j = m->host.nr++;
        vmcs_write32(VM_EXIT_MSR_LOAD_COUNT, m->host.nr);
    }
    m->host.val[j].index = msr;
    m->host.val[j].value = host_val;
}
```
该函数传入的参数：
1. vmx: 当前vcpu的vcpu_vmx指针
2. msr: 需要设置的MSR的index
3. guest_val: 需要设置的MSR在guest中的值
4. host_val: 需要设置的MSR在host中的值
5. entry_only: 是不是只需要设置VM entry的自动load guest的值到hardware MSR中。

具体地，我们先忽略switch对特殊的MSR的处理，看下面的部分。
1. 首先获取该MSR在guest区域的entry号i，以及如果！entry_only时，该MSR在host区域的entry号j；
2. 判断i和j是否超出NR_AUTOLOAD_MSRS的限制，如果超过了，则内核输出warning,并直接返回；（此处是一个bug，i不会等于NR_AUTOLOAD_MSRS,需要提交一个patch）
3. 如果i < 0，则说明该MSR不在auto load的guest区域，则可以将该MSR加入autoload的guest区域，具体地，
   将vmx->msr_autoload.guest.nr++,并更新vmcs中 VM-entry MSR-load count；
4. 如果i >0 && i < NR_AUTOLOAD_MSRS,则说明该MSR已经在msr_autoload.guest.val中，则只用更新其值，
   将该MSR 保存到vmx->msr_autoload.guest.val[i]中.
5. 如果entry_only则直接返回，如果不是，则同3.4步更新autoload的host区域。

下面我们来看switch中特殊的处理。
1. 如果是MSR_EFER：如果启用了vm-entry和vm exit control中的load_efe功能，那么步使用msr-load功能，在guest state和host state有专门的区域存这个值。
   会调用add_atomic_switch_msr_special()来处理。具体地，会设置VMCS中guest state和host state中相关的MSR区域，并且置上VMCS中相关的控制位。
  这里对MSR_EFER也有特殊的处理，即如果是更新HOST MSR_EFER,则不执行更新。
2. 如果是MSR_CORE_PERF_GLOBAL_CRTL.同上。
3. 如果是MSR_IA32_PEBS_ENABLE: 需要先将host上的该功能disable一段时间，具体为什么我没研究。
4. 根据前面在host state, guest state的分析，MSR_IA32_PAT应该也在case里面，但是不在，可见KVM有特殊处理，目前我还没研究。





[msr_bitmap]: kvm对guest中MSR寄存器的处理（一）.html
