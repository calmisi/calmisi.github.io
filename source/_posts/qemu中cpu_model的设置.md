---
title: qemu中cpu model（cpu type）的设置
date: 2019-02-17 14:36:38
tags:
categories:
- QEMU
---

KVM需要配合QEMU使用，当我们使用QEMU起虚拟机的时候，会使用qemu-system-x86_64程序。具体地我们会使用参数`-cpu model`来指定guest使用的cpu model。本文分析`-cpu`参数是如何工作的。
<!-- more -->

# 一、 QEMU cpu model
## cpu model 定义
QEMU中的cpu model定义在target/i386/cpu.c中builtin_x86_defs[]中。
```C
static X86CPUDefinition builtin_x86_defs[] = {
	{
        .name = "qemu64",
        .level = 0xd,
        .vendor = CPUID_VENDOR_AMD,
        .family = 6,
        .model = 6,
        .stepping = 3,
        .features[FEAT_1_EDX] =
            PPRO_FEATURES |
            CPUID_MTRR | CPUID_CLFLUSH | CPUID_MCA |
            CPUID_PSE36,
        .features[FEAT_1_ECX] =
            CPUID_EXT_SSE3 | CPUID_EXT_CX16,
        .features[FEAT_8000_0001_EDX] =
            CPUID_EXT2_LM | CPUID_EXT2_SYSCALL | CPUID_EXT2_NX,
        .features[FEAT_8000_0001_ECX] =
            CPUID_EXT3_LAHF_LM | CPUID_EXT3_SVM,
        .xlevel = 0x8000000A,
        .model_id = "QEMU Virtual CPU version " QEMU_HW_VERSION,
    },
	...
	{   
        .name = "Cascadelake-Server",
        .level = 0xd,
        .vendor = CPUID_VENDOR_INTEL,
        .family = 6,
        .model = 85,
        .stepping = 6,
        .features[FEAT_1_EDX] =
            CPUID_VME | CPUID_SSE2 | CPUID_SSE | CPUID_FXSR | CPUID_MMX |
            CPUID_CLFLUSH | CPUID_PSE36 | CPUID_PAT | CPUID_CMOV | CPUID_MCA |
            CPUID_PGE | CPUID_MTRR | CPUID_SEP | CPUID_APIC | CPUID_CX8 |
            CPUID_MCE | CPUID_PAE | CPUID_MSR | CPUID_TSC | CPUID_PSE |
            CPUID_DE | CPUID_FP87,
        .features[FEAT_1_ECX] =
            CPUID_EXT_AVX | CPUID_EXT_XSAVE | CPUID_EXT_AES |
            CPUID_EXT_POPCNT | CPUID_EXT_X2APIC | CPUID_EXT_SSE42 |
            CPUID_EXT_SSE41 | CPUID_EXT_CX16 | CPUID_EXT_SSSE3 |
            CPUID_EXT_PCLMULQDQ | CPUID_EXT_SSE3 |
            CPUID_EXT_TSC_DEADLINE_TIMER | CPUID_EXT_FMA | CPUID_EXT_MOVBE |
            CPUID_EXT_PCID | CPUID_EXT_F16C | CPUID_EXT_RDRAND,
        .features[FEAT_8000_0001_EDX] =
            CPUID_EXT2_LM | CPUID_EXT2_PDPE1GB | CPUID_EXT2_RDTSCP |
            CPUID_EXT2_NX | CPUID_EXT2_SYSCALL,
        .features[FEAT_8000_0001_ECX] =
            CPUID_EXT3_ABM | CPUID_EXT3_LAHF_LM | CPUID_EXT3_3DNOWPREFETCH,
        .features[FEAT_7_0_EBX] =
            CPUID_7_0_EBX_FSGSBASE | CPUID_7_0_EBX_BMI1 |
            CPUID_7_0_EBX_HLE | CPUID_7_0_EBX_AVX2 | CPUID_7_0_EBX_SMEP |
            CPUID_7_0_EBX_BMI2 | CPUID_7_0_EBX_ERMS | CPUID_7_0_EBX_INVPCID |
            CPUID_7_0_EBX_RTM | CPUID_7_0_EBX_RDSEED | CPUID_7_0_EBX_ADX |
            CPUID_7_0_EBX_SMAP | CPUID_7_0_EBX_CLWB |
            CPUID_7_0_EBX_AVX512F | CPUID_7_0_EBX_AVX512DQ |
            CPUID_7_0_EBX_AVX512BW | CPUID_7_0_EBX_AVX512CD |
            CPUID_7_0_EBX_AVX512VL | CPUID_7_0_EBX_CLFLUSHOPT,
        .features[FEAT_7_0_ECX] =
            CPUID_7_0_ECX_PKU | CPUID_7_0_ECX_OSPKE | 
            CPUID_7_0_ECX_AVX512VNNI,
        .features[FEAT_7_0_EDX] =
            CPUID_7_0_EDX_SPEC_CTRL | CPUID_7_0_EDX_SPEC_CTRL_SSBD,
        /* Missing: XSAVES (not supported by some Linux versions,
                * including v4.1 to v4.12).
                * KVM doesn't yet expose any XSAVES state save component,
                * and the only one defined in Skylake (processor tracing)
                * probably will block migration anyway. 
                */
        .features[FEAT_XSAVE] =
            CPUID_XSAVE_XSAVEOPT | CPUID_XSAVE_XSAVEC |
            CPUID_XSAVE_XGETBV1,
        .features[FEAT_6_EAX] =
            CPUID_6_EAX_ARAT,
        .xlevel = 0x80000008,
        .model_id = "Intel Xeon Processor (Cascadelake)",
    },
	...
}
```
可以看到这里定义了，x86平台中支持的那些cpu model。具体的可以通过`qemu-system-x86_64 -cpu help`指令查看。

## cpu model(cpu type)的初始化
```C
in target/i386/cpu.c
static void x86_cpu_register_types(void)
{
    int i;

    type_register_static(&x86_cpu_type_info);
    for (i = 0; i < ARRAY_SIZE(builtin_x86_defs); i++) {
        x86_register_cpudef_type(&builtin_x86_defs[i]);
    }
    type_register_static(&max_x86_cpu_type_info);
    type_register_static(&x86_base_cpu_type_info);
#if defined(CONFIG_KVM) || defined(CONFIG_HVF)
    type_register_static(&host_x86_cpu_type_info);
#endif  
}

type_init(x86_cpu_register_types)
```
在target/i386/cpu.c文件中的最后一行有`tpye_init(x86_cpu_register_types)`，`type_init()`为宏定义。其宏定义展开如下：
```C
in include/qemu/module.h
#define type_init(function) module_init(function, MODULE_INIT_QOM)

#ifdef BUILD_DSO
void DSO_STAMP_FUN(void);
/* This is a dummy symbol to identify a loaded DSO as a QEMU module, so we can
 * distinguish "version mismatch" from "not a QEMU module", when the stamp
 * check fails during module loading */
void qemu_module_dummy(void);

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_dso_module_init(function, type);                               \
}
#else
/* This should not be used directly.  Use block_init etc. instead.  */
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
#endif
```
具体地，会实现为一个`__attribute__((constructor))`函数，会在main()函数之前运行。
即会在main()函数之前运行`x86_cpu_register_types()`函数，该函数注册x86平台支持的cpu_type。

### type_register_static&x86_register_cpudef_type
在`x86_cpu_register_types()`函数中主要调用了`type_register_static()`函数和`x86_register_cpudef_type()`函数。
其中`type_register_static()`函数直接调用`type_register()`函数注册QEMU定义好的TypeInfo（e.g., x86_cpu_type_info, max_x86_cpu_type_info, x86_base_cpu_type_info, host_x86_cpu_type_info）；
而`x86_register_cpudef_type()`函数会根据builtin_x86_defs[]数组定义的cpu model生成对应的TypeInfo，然后再调用`type_register()`函数，所以注册cpu type的过程就是：
1. 准备好TypeInfo的结构体；
2. 调用`type_register()`函数注册对应的TypeInfo；

下面我们来看`type_register()`函数：
```C
TypeImpl *type_register_static(const TypeInfo *info)
{
    return type_register(info);
}

TypeImpl *type_register(const TypeInfo *info)
{
    assert(info->parent);
    return type_register_internal(info);
}

static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti; 
    ti = type_new(info);

    type_table_add(ti);
    return ti;
}
```
可以看到`type_register()`函数首先判断传入的TypeInfo指针是否存在`parent`，如果不存在，则报错；存在则调用`type_register_internal()`函数,该函数会调用`type_new(TypeInfo *)`函数根据TypeInfo动态生成一个TypeImpl的实例，并使用`type_table_add()`函数将`(TypeInfo->name,TypeImpl)`的(key,value)对加入hash_table。以供下一节中`type_get_by_name()`函数根据`TypeInfo->name`来从hash_table中查询指定的TypeImpl。

下面我们重新回到`x86_cpu_register_types()`函数，看看其注册哪些TypeInfo类型的cpu type:（注意，此处为了方便说明和分类，与`x86_cpu_register_types()`中调用的顺序不一致）

1. type_register_static(&x86_cpu_type_info)：
```C
static const TypeInfo x86_cpu_type_info = {
	.name = TYPE_X86_CPU,	//"x86_64-cpu" or "i386-cpu" 
	.parent = TYPE_CPU,		//"cpu"
	.instance_size = sizeof(X86CPU),
	.instance_init = x86_cpu_initfn,
	.abstract = true,
	.class_size = sizeof(X86CPUClass),
	.class_init = x86_cpu_common_class_init,
};
```
2. type_register_static(&max_x86_cpu_type_info)：
```C
#define X86_CPU_TYPE_SUFFIX "-" TYPE_X86_CPU
#define X86_CPU_TYPE_NAME(name) (name X86_CPU_TYPE_SUFFIX)
#define CPU_RESOLVING_TYPE TYPE_X86_CPU

static const TypeInfo max_x86_cpu_type_info = {
    .name = X86_CPU_TYPE_NAME("max"),	//"max""-"TYPE_X86_CPU
    .parent = TYPE_X86_CPU,
    .instance_init = max_x86_cpu_initfn,
    .class_init = max_x86_cpu_class_init,
};
```
3. type_register_static(&x86_base_cpu_type_info)：
```C
static const TypeInfo x86_base_cpu_type_info = {
	.name = X86_CPU_TYPE_NAME("base"),	//"base""-"TYPE_X86_CPU
	.parent = TYPE_X86_CPU,
	.class_init = x86_cpu_base_class_init,
};
```
4. type_register_static(&host_x86_cpu_type_info)：
```C
static const TypeInfo host_x86_cpu_type_info = {
	.name = X86_CPU_TYPE_NAME("host"),	//"host""-"TYPE_X86_CPU
	.parent = X86_CPU_TYPE_NAME("max"),
	.class_init = host_x86_cpu_class_init,
};
```

5. x86_register_cpudef_type(&builtin_x86_defs[i])：
```C
static void x86_register_cpudef_type(X86CPUDefinition *def)
{
    char *typename = x86_cpu_type_name(def->name);
    TypeInfo ti = {
        .name = typename,	//"def->name""-"TYPE_X86_CPU; 如Cascadelake-Server-x86_64-cpu
        .parent = TYPE_X86_CPU,	
        .class_init = x86_cpu_cpudef_class_init,
        .class_data = def,
    };
    ...
}
```
<a id="cpu_type_name"></a>我们可以看到`x86_cpu_register_types()`函数中注册了
* x86_64-cpu;
* max-x86_64-cpu;
* base-x86_64-cpu;
* host-x86_64-cpu;(如果支持KVM或者HVF)
* (cpu model)-x86_64-cpu;(builtin_x86_defs[]中定义的cpu model)
这5类名字的cpu type;这5类都为`TypeInfo->name`。我们此时用TypeInfo->name来表示不同的TypeInfo变量。
`x86_64-cpu`为基础类型,其parent为TYPE_CPU,并且其为abstract,即为抽象类。其定义了，instance_size、instance_init、class_size、class_init。
以`x86_64-cpu`为基础类型，定义了子类型`max-x86_64-cpu`、 `base-x86_64-cpu`、 `host-x86_64-cpu`、 `cpu model-x86_64-cpu`。这些子类型可以定义自己的instance_init和class_init来覆盖父类型`x86_64-cpu`定义的。

在`type_register_internal()`时，会调用`type_new()`根据TypeInfo生成TypeImpl实例：
```C
static TypeImpl *type_new(const TypeInfo *info)
{
    TypeImpl *ti = g_malloc0(sizeof(*ti));
    int i;

    g_assert(info->name != NULL);

    if (type_table_lookup(info->name) != NULL) {
        fprintf(stderr, "Registering `%s' which already exists\n", info->name);
        abort();
    }    

    ti->name = g_strdup(info->name);
    ti->parent = g_strdup(info->parent);

    ti->class_size = info->class_size;
    ti->instance_size = info->instance_size;

    ti->class_init = info->class_init;
    ti->class_base_init = info->class_base_init;
    ti->class_data = info->class_data;

    ti->instance_init = info->instance_init;
    ti->instance_post_init = info->instance_post_init;
    ti->instance_finalize = info->instance_finalize;

    ti->abstract = info->abstract;

    for (i = 0; info->interfaces && info->interfaces[i].type; i++) {
        ti->interfaces[i].typename = g_strdup(info->interfaces[i].type);
    }    
    ti->num_interfaces = i; 

    return ti;
}
```
可以，`type_new()`函数将TypeInfo的信息都复制给TypeImpl。到目前为止hash table中存的是(TypeImpl->name, TypeImpl *)的键值对，此时的TypeImpl还没有初始化完成，后续的初始化在`object_class_by_name()`函数中调用`type_initialize(TypeImpl)`完成。
具体地，在`type_initialize()`函数会根据`type->class_size`动态分配`type->class`,并将自己存到`type->class->type = ti`中，完成双向引用(`object_class_by_name()`会返回type->class,通过class->type可以重新获取type的指针）。

---

# 二、 main()函数初始化
在QEMU的源代码的linux-user/main.c函数中，是linux平台的main()函数入口。
```C
static const char *cpu_model;
static const char *cpu_type;

int main(int argc, char **argv, char **envp)
{
	...
	cpu_model = NULL;
	...
	optind = parse_args(argc, argv);
	...
	if (cpu_model == NULL) {
		cpu_model = cpu_get_model(get_elf_eflags(execfd));
	}
	cpu_type = parse_cpu_model(cpu_model);
	...
}

static void handle_arg_cpu(const char *arg)                                                                                                                 
{
    cpu_model = strdup(arg);
    if (cpu_model == NULL || is_help_option(cpu_model)) {
        /* XXX: implement xxx_cpu_list for targets that still miss it */
#if defined(cpu_list)
        cpu_list(stdout, &fprintf);
#endif
        exit(EXIT_FAILURE);
    }   
}
```
`parse_args(argc, argv)`函数会解析`qemu-system-x86_64`命令后面的参数。
其中对`-cpu`选项的处理为：handle_arg_cpu()函数，其会将`-cpu`后的参数存入cpu_model.
经过parse_args()函数处理后，如果cpu_model还为NULL，则说明命令行参数没有指定`-cpu`参数,那么调用cpu_get_model(get_elf_eflags(execfd));该函数会如果是i386则设置`cpu_model = "qemu32"`,如果是x86_64则设置`cpu_model = "qemu64"`。
然后调用parse_cpu_model()函数使用`cpu_model`设置`cpu_type`,

## parse_cpu_model()
下面我们来看parse_cpu_model()函数
```C
const char *parse_cpu_model(const char *cpu_model)
{
    ObjectClass *oc; 
    CPUClass *cc; 
    gchar **model_pieces;
    const char *cpu_type;

    //将-cpu选项后的参数以“,”字符分割存入model_pieces[]
    //比如-cpu host,-nx,+cpuid 那么model_pieces[0]=host, model_pieces[1]=-nx,+cpuid
    model_pieces = g_strsplit(cpu_model, ",", 2);

    oc = cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0]);
    if (oc == NULL) {
        error_report("unable to find CPU model '%s'", model_pieces[0]);
        g_strfreev(model_pieces);
        exit(EXIT_FAILURE);
    }    

    //objetc_class_get_name()函数返回 oc->type->name;
    cpu_type = object_class_get_name(oc);
    cc = CPU_CLASS(oc);
    //调用用model_pieces[0]-TYPE_X86_CPU获得的CPUClass->parse_features()
    //具体地x86平台会调用x86_cpu_parse_featurestr()
    cc->parse_features(cpu_type, model_pieces[1], &error_fatal);
    g_strfreev(model_pieces);
    return cpu_type;
}
```

在parse_cpu_model()函数中，首先会根据cpu_class_by_name(CPU_RESOLVING, model_pieces[0])函数获取cpu_model的ObjectClass。
```C
ObjectClass *cpu_class_by_name(const char *typename, const char *cpu_model)
{
    CPUClass *cc = CPU_CLASS(object_class_by_name(typename));

    assert(cpu_model && cc->class_by_name);
    return cc->class_by_name(cpu_model);
}
```
* 传入的参数1：typename: CPU_RESOLVING_TYPE,即为TYPE_X86_CPU,(x86_64-cpu)；
* 传入的参数2：cpu_model: model_pieces[0],即为qemu-system-x86_64传入的`-cpu参数`；
cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0])函数会调用object_class_by_name(CPU_RESOLVING_TYPE)获取"x86_64-cpu"对应的ObjectClass；
```C
ObjectClass *object_class_by_name(const char *typename)
{
    TypeImpl *type = type_get_by_name(typename);

    if (!type) {
        return NULL;
    }    

    type_initialize(type);

    return type->class;
}

static TypeImpl *type_get_by_name(const char *name)
{
    if (name == NULL) {
        return NULL;
    }

    return type_table_lookup(name);
}

static TypeImpl *type_table_lookup(const char *name)
{       
    return g_hash_table_lookup(type_table_get(), name);
}
```
 `object_class_by_name(typename)`会调用`type_get_by_name(typename)`，其会调用`type_table_lookup(name)`在[hash_table](#type-register-static-amp-x86-register-cpudef-type)中查找对应的name对应的TypeImpl。
该hash_table中的元素是在上一节[cpu model(cpu type)的初始化](#cpu-model-cpu-type-的初始化)中调用`x86_cpu_register_types()`==>`type_register_static()`/`x86_register_cpudef_type()`==>`type_register()`==>`tyep_register_internal()`函数向hash_table中加入的。
获取到typename对应的TypeImpl实例后，会调用`type_initialize()`函数，该函数会动态分配type-class，并且还会调用type->class_init()函数，初始化type->class。
调用`object_class_by_name(CPU_RESOLVING_TYPE)`获取到ObjectClass后，将其转化为CPUClass,然后调用cc->class_by_name(cpu_model)。
具体地，CPU_RESOLVING_TYPE(TYPE_X86_CPU)的`.class_init = x86_cpu_common_class_init`，在x86_cpu_common_class_init()函数中将该CPUClass的`class_by_name`句柄设置为`x86_cpu_class_by_name()`，所以会调用`x86_cpu_class_by_name()`函数，其调用object_class_by_name(model_pieces[0]"-"TYPE_X86_CPU)获取oc;
>总结：
>cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0])函数，最终会调用object_class_by_name(cpu_model-x86_64-cpu)获取ObjectClass
>其中cpu_model如果`-cpu`参数后面有指定，则用指定的；没有指定，则为qemu32或qemu64。


获取到cpu_model对应的oc后，调用`object_class_get_name()`获取cpu_type即oc->type->name，具体name的值可以看上一节[cpu type name](#cpu_type_name)；
然后调用cc->parse_features()函数会根据`-cpu`函数后面的-feature,+feature等初始化plus_features[]和minus_features[]数组。

> 总结
> `parse_cpu_model()`函数，不仅获得了cpu_type，还初始化了plus_features[]和minus_features[]数组。
