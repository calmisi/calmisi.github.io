---
title: 'CPL,RPL,DPL三者的关系，以及CPL的切换规则'
date: 2018-10-18 20:41:34
tags: Intel
categories: Intel-SDM
---

# 三个概念：

* CPL: Current privilege Level, 当前特权级，其为当前的CS段和SS段的段寄存器的第0-1位的值，如下图所示，段寄存器中的可见部分的内容，即为segment selector的值，所以当前代码的CPL即为载入段描述符时，段选择符的RPL的值。
 **注意：当前的CS段和SS段的段寄存器总的CPL值一定是一样的，当发生CPL切换时，对应的也会切换SS堆栈**
{% asset_img segment_register.JPG segment register %}
* DPL: Descriptor privilege level,为segment或者gate的特权级。其值在segment decriptor或gate descriptor的DPL区域。
{% asset_img segment_descriptor.JPG segment descriptor %}
* RPL: Requested Privilege level，如下图所示为segment selector的最低2位。
{% asset_img segment_selector.JPG segment selector %}

CPL,DPL,RPL的取值为0,1,2,3.数值越大，特权级越低。
当要访问数据段中的操作数的时候，数据段(data segment)对应的段选择符(segment selector)必须加载到对应的数据段寄存器(data segment registers, DS, ES, FS or GS)或者堆栈段(stack-segment register,SS)。
在加载的时候，需要进行特权级检查，需要对比当前正在运行的程序的CPL,目标操作数所在段的段选择符的RPL,以及目标操作数所在段的段描述符的DPL
只有当CPL <= DPL && RPL <= DPL，目标操作数所在的段才会加载成功，否则产生一个GP,general protection.

<!-- more -->

---

# 特权级的检查(Privilege level checking)
## 访问数据段(access data segment)
当访问数据段(data segment)的时候，段选择符(segment selector)必须加载到数据段寄存器（DS、ES、FS、GS）或者堆栈寄存器SS(Stack segment是一种特殊的数据段)。
> **条件：CPL <= DPL && RPL <= DPL**

**注意：segment selector的RPL是受软件控制的，也就是说一个CPL=3的程序可以将一个data-segment的RPL设为0,然后去访问一个data segment.那么RPL <= DPL必定成立，则只用检查DPL。**

### 访问代码段（code segment）中的数据
1. Load a data-segment register with a segment selector for a nonconforming, readable, code segment: CPL <= DPL && RPL <= DPL.
2. Load a data-segment register with a segment selector for a conforming, readable, code segment: ** always valid, 因为conforming的code segment的特权级总是和CPL一样的**
3. Use a code-segment override prefix(CS) to read a readable, code segment whose selector is already loaded in the CS register: ** always valid,因为CS寄存器选择的code segment的DPL和CPL是一样的** 

## 访问SS register
当想SS(stack segment) register中加载新的stack segment的segment selector时：
> **CPL == DPL == RPL**

---

## 访问代码段
程序控制权从一个code segment转移到另一个code segment时，目标段的segment selector必须加载到code-segment register。
此时会进行多种权限检查，如上面提到的CPL，DPL，RPL。以及目标代码段的type, limit等。
如果检查通过，则将目标code segment的segment selector加载到CS段寄存器，程序控制权则被转移到了新加载的代码段，并且程序从EIP寄存器所指向的地址开始执行。

### 程序控制权的转移
1. 通过以下指令：JMP，CALL，RET，SYSENTER，SYSEXIT，SYSCALL，SYSRET，INT n;
2. exception和interrupt机制，以及IERT指令；

#### JMP或CALL指令可以以下面4种方式来引用另一个code segment:
1. 目标操作数包含指向目标code segment 的segment selector；
2. 目标操作数指向一个call-gate descriptor,其包含一个指向目标code segment的segment selector.
3. 目标操作数指向一个TSS,其包含指向目标code segment的segment selector;
4. 目标操作数指向一个task gate,其指向一个TSS，TSS包含指向目标code segment的segment selector;


##### 1.直接跳转到code segment:目标操作数包含指向目标code segment 的segment selector；
* JMP, CALL 和RET指令的段内跳转，不用切换code segment。
* JMP, CALL 和RET指令的远程跳转，跳转到另一个code segment,需要进行特权级检查。
  转移程序控制权到另一个code segment,**且没有使用call gate的时候**：
  1. 当转移到Nonconforming code segment: CPL == DPL && RPL <= CPL(== DPL),**并且切换到新的code segment后，CPL不变（即使RPL < CPL）**
  2. 当转移到Conforming code segment: CPL >= DPL (RPL直接被忽略)
     此情形时，当程序控制权转移到conforming code segment,CPL不会改变，即使目标code segment的DPL比调用程序的CPL数值要小。因为CPL没有改变，所以也不会发生stack切换。
     一般Conforming segments为math libraries和exception handlers之类的代码模块。

##### 2.使用gate descriptor进行跳转
1. Call gates
2. Trap gates
3. Interrupt gates
4. Task gates
其中Task gates用来task swithcingl; Trap and interrupt gates是特殊的call gates用来调用exception and interrupt handlers.
下面主要讨论call gates.

### Call gates
Call gates用来在不同的特权级之间转移程序控制权，此外call gates也可以在16bits和32bits的代码段之间转移程序控制权。
Call gates descriptor存放于GDT或者LDT中，但不会出现在IDT中。
下图为call-gate descriptor的组成:

{% asset_img call_gate_desc.JPG call gate descriptor %}

1. 指明了需要被访问的code segment(segment selector)
2. 目标code segment的程序入口(offset in segment)
3. caller程序的特权级。（DPL域指明了call gate的特权级，which in turn is the privilege level required to access the selected procedure through the gate）
4. 如果需要进行stack切换，指明了需要在stack之间复制的可选参数的数量.(Parm Count specifies the number of words for 16-bit call gates and doublewords for 32-bit call gates)
5. 定义了要push到目标stack的size: 16-bit gates force 16-bit pushes and 32-bit gates forces 32-bit pushes.
6. 指明当前call-gate是否有效。(P flag)

### 通过call gate访问一个code segment。
为了访问call gate，CALL或者JMP指令的目标操作数需要为一个指向call gate的far pointer。
如下图所示，the segment selector指明了call gate, the offset是需要的，但是不被检查和使用。
会使用far pointer的segment selector在GDT或者LDT中选中一个call gate.然后使用该call gate中的segment selector和offset来组成目标线性地址。
如下图所示，需要检查4个特权级：

{% asset_img call_gate_checks_items.JPG Privilege Check for Control Transfer with Call Gate %}

+ CPL （当前caller的cs的特权级）
+ RPL （requestor's privilege level）：指向call gate descripter的selector的RPL
+ DPL （descriptor privilege level）：call gate descriptor的DPL
+ DPL： call gate descriptor中的segment selector所指向的code segment descriptor中的DPL
下图给出了使用JMP或者CALL指令是，上述4个特权级需要满足的关系：

{% asset_img call_gate_privilege_check.JPG Priviledge Check Rules for Call Gates %}

如果call了一个更高特权级的nonconforming的目标code segment.那么CPL会被减小数值之目标code segment的DPL。那么就会发生stack切换。
如果call或者jump了一个更高特权级的conforming的目标code segment.不会改变CPL，所以也不会发生stack切换。

> 总结1
> when access code segment
> 1. When transferring program control to another code segment without going through a call gate.
>    * nonconforming: CPL == DPL && RPL <= CPL (RPL <= CPL == DPL)。并且新的CPL不变，即使RPL比较小
>    * conforming: CPL >= DPL && RPL is ignored. 并且新的CPL不变。
> 
> 2. When transferring program control through a call gate.
>    **CPL <= DPL of call gate descriptor** && **RPL of call gate selector <= DPL of call gate descriptor**
>    * nonconforming code segment: DPL <= CPL (CALL)     DPL == CPL (JMP)
>    * conforming code segment: DPL <= CPL (CALL)     DPL <= CPL  (JMP)
>    That means: 
>    Only CALL instructions can use call gates to transfer program control to more privileged (数值更小的) **nonconforming** code segment.
>    A JMP instruction can use a call gate only to transfer program control to a **nonconforming** code segment with a DPL equal to the CPL.
>    CALL and JMP instruction can both transfer program control to a more privileged conforming code segment;

> 总结2
> the  change of CPL
> 1. without call gate
>    * near call: CPL不变；
>    * far call: CPL不变；
> 2. with call gate
>    * **more privileged nonconforming destination code segment: the CPL is lowered to the DPL of the destination code segment and a stack switch occurs.**
>    * more privileged conforming destination code segment: CPL不变，stack不切换。




