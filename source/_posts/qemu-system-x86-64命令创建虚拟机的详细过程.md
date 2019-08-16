---
title: qemu-system-x86_64命令创建虚拟机的详细过程
date: 2019-06-22 19:14:06
tags:
- QEMU-KVM
categories:
- KVM
- QEMU
---

当使用`qemu-system-*arch*`启动虚拟机的时候，不用是使用的哪种`arch`的qemu程序，其函数入口都为vl.c中的main函数。
本文具体分析使用`qemu-system-x86_64`创建虚拟机的详细过程。

<!-- more -->

# main()函数
入口函数main()在vl.c中， 在vl.c中定义了一个全局变量`MachineState *current_machine`。
在main()函数中有：
```C
int main(int argc, char **argv, char **envp)
{
	...
	MachineClass *machine_class;
	...
	machine_class = select_machine();
	object_set_machine_compat_props(machine_class->compat_props);
	set_memory_options(&ram_slots, &maxram_size, machine_class);
	...
	current_machine = MACHINE(object_new(object_class_get_name(
						OBJECT_CLASS(machine_class))));
	...
}
```


# 1. 参数解析
在代码中判断完`userconfig`后，会进行`second pass of option parsing`，具体地，会在for循环中处理程序后面的参数。
```C
for(;;)
{
	...
	popt = lookup_opt(argc, argv, &optarg, &iptind);
	...
	switch(popt->index) {
	case QEMU_OPTION_cpu:
		/* hw initialization will check this */
		cpu_option = optarg;
		break;
	case ...
	...
	}
}
```
## -cpu参数
其中`QEMU_OPTION_cpu`负责解析`-cpu model,+feature,-feature`参数。会将参数值存入`cpu_option`，本文假设model为host,即`-cpu host`。
然后在后面的代码有：
```C
/* parse features once if machine provides default cpu_type */
current_machine->cpu_type = machine_class->default_cpu_type;
if (cpu_option) {
	current_machine->cpu_type = parse_cpu_option(cpu_option);
}
```
如果没有使用`-cpu`参数，那么设置cpu_type为default_cpu_type;否则从`cpu_option`中解析cpu_type。

### parse_cpu_option函数
```C
const char *parse_cpu_option(const char *cpu_option)
{
	ObjectClass *oc;
	CPUClass *cc;
	gchar **model_pieces;
	const char *cpu_type;

	model_pieces = g_strsplit(cpu_option, ",", 2);
	if (!model_pieces[0]) {
		error_report("-cpu option cannot be empty");
		exit(1);
	}

	oc = cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0]);
	if (oc == NULL) {
		error_report("unable to find CPU model '%s'", model_pieces[0]);
		g_strfreev(model_pieces);
		eixt(EXIT_FAILURE);
	}

	cpu_type = object_class_get_name(oc);
	cc = CPU_CLASS(oc);
	cc->parse_features(cpu_type, model_pieces[1], &error_fatal);
	g_strfreev(model_pieces);
	return cpu_type;
}
```
1. 将`cpu_option`参数用","进行切分存入model_pieces[]数组，model_pieces[0]应该指定cpu model;
2. 调用`cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0])`从model_pieces[0]解析指定cpu model的object:
  1. `cpu_class_by_name()`首先获取CPU_RESOLVING_TYPE的ObjectClass; //此时的CPU_RESOLVING_TYPE为TYPE_X86_CPU(x86_64-cpu)
	 请参考`cpu model`的定义和加载： [qemu中cpu model（cpu type）的设置][cpu_model]
  2. 然后将该ObjectClass转化为CPUClass *cc;
  3. 调用cc->class_by_name(model_pieces[0])获取该cpu model对应的ObjectClass;

  > 总结：
  > `cpu_class_by_name(CPU_RESOLVING_TYPE, model_pieces[0])`函数，最终会调用`object_class_by_name("cpu_model"-x86_64-cpu)`获取ObjectClass。
  > 其中“cpu model"为`-cpu`参数后指定的model,如果没有指定则为`qemu32`或者`qemu64`。

3. 获取到cpu model对应的`oc`后，会调用`object_calss_get_name(oc)`获取cput_type,即oc->type->name;
4. 最后调用`cc->parse_features()`函数，会根据`-cpu`参数后面的`-feature,+feature`来初始化`plus_features[]`和`minus_features[]`数组。

最后`parse_cpu_option()`函数将`-cpu host`解析的`cpu_type`为"host-x86_64-cpu"。

> 总结：
> `cpu_class_by_name()函数`
> 1. `object_class_by_name(CPU_RESOLVING_TYPE)`：根据CPU_RESOLVING_TYPE获取平台相关的基础typename的ObjectClass,(CPU_RESOLVING_TYPE在不同的平台定义不一样)
>    此处，如果是x86_64平台，那么CPU_RESOLVING_TYPE为TYPE_X86_CPU,即为x86_64-cpu。
>    获取其ObjectClass的过程为，先根据typename(也就是"x86_64-cpu")获取其TypeImpl *type;
>    然后调用`type_initialize(type)`,会调用class_init()， TYPE_X86_CPU的class_init()函数为`x86_cpu_common_class_init()`，其会把`class_by_name`赋值为`x86_cpu_class_by_name()`,这在后面会被用到；
>    最后返回type->class;
> 2. 将上一步获取的ObjectClass转化为CPUClass cc;
> 3. 调用cc->class_by_name(cpu_model)来获取具体的cpu model的ObjectClass; 
>    TYPE_X86_CPU对应的CPUClass的class_by_name()为`x86_cpu_class_by_name()`，其又调用`object_class_by_name()`获取model_pieces[0]-TYPE_X86_CPU（即host-x86_64-cpu）的ObjectClass.
>    该步骤同第一步调用的函数相同，只是其参数为cpu model对应的type name: host-x86_64-cpu；其根据该typename找到对应的TypeImpl,调用type_initialize, -> class_init()并返回ObjectClass;
> 
> `cc->parse_features()函数`
> 会调用`host-x86_64-cpu`的CPUClass的parse_features()函数，该函数由class_init()确定，因为host-x86_64-cpu的parent是max-x86_64-cpu, max的parent是x86_64-cpu，所以会依次从父类的class_init()初始化开始，可以发现max和host都没有对parse_features()进行赋值，即最终会使用并x86_64-cpu的class_init中cc->parse_features = x86_cpu_parse_featurestr的赋值，所以最终调用的是`x86_cpu_parse_featurestr(cpu_type, model_pieces[1], &error_fatal)`
>
> 最后可见，parse_cpu_option()函数，type_initialize(cpu-model-TypeImpl)（此时是host-x86_64）, 初始化`plus_features[]`和`minus_features[]`数组,最终返回cpu_type的名字，`-cpu host`为host-x86_64-cpu。




























[cpu_model]: ../2/qemu中cpu_model的设置.html