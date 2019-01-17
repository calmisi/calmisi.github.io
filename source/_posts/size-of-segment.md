---
title: size of segment
date: 2018-10-28 15:23:53
tags: Intel
categories: Intel-SDM
---

# 在segment descriptor中，高4个bytes(4-7bytes)的bit 22为D/B字段。
（this flag should always be set to 1 for 32-bit code and data segments and 0 for 16-bit code and data segments）

{% asset_img segment_descriptor.JPG segment descriptor %}

<!-- more -->

1. Executable code segment:该字段为D flag，决定了该代码段中指令的有效地址的长度和操作数的长度。
   1表示使用32位的地址和32-bit(4字节)或者8-bit(1字节)的操作数；0表示使用16位的地址和16-bit(2字节)或者8-bit(1字节)的操作数.
   (当Lbit, bit 21 of the second doubleword of the segment descriptor indicates whether a code segment contains native 64-bit code.
    1表示该code segment中的指令在64-bit模式执行；0表示该code segment的指令在compatibility模式执行。
    如果L位置1了，那么D bit必须清除；当不在IA-32e模式，或者不是code segment时，L bit is reserved and should always be set to 0.``)
2. Stack segment (data segment pointed to by the SS register). 该字段为B（big）flag.该字段决定了堆栈指针的大小。
   1表示使用32-bit stack pointer,存放在32-bit的ESP寄存器中；0表示16-bit的stack pointer，存放在16-bit的SP寄存器中。
   如果stack segment是一个expand-down的data-segment，那么该B flag也制定了stack segment的upper bond.
3. Expand-down data segment:该字段为B flag,指定了segment的upper bound. 1表示upper bound为FFFFFFFF(4GBytes);0表示upper bound为FFFF(64KBytes).

 所以当segment为stack segment时，第2条和第3条同时生效。


# Stack Alignment
 The stack pointer for a stack segment should be aligned on 16-bit (word) or 32-bit (double-word) boundaries, depending on the width of the stack segment.
 The D flag in the segment secriptor for the current code segment sets the stack-segment width.
 (当Dbit为1时，表示使用32-bit或8bit的操作数；当D bit为0时，表示使用16-bit和8-bit的操作数)
 (即，当D bit为1时，stack的width为32bits；当D bit为0时，stack的width为16bits)

# Adress-Size Attributes for Stack Accesses: 
implicit adress of the top of the stack, and they may also have an explicit memory address (for example, PUSH Array1[EBX]).
  * The address-size attribute of the top of the stack determines whether SP or ESP is used for the stack access. 
    Stack operations with an address-size attribute of 16 use the 16-bit SP stack pointer can can use a maximum stack address of FFFFH;
    Stack operations with an address-size attribute of 32 bits ues the 32-bit ESP register and can usa a maximum address of FFFFFFFFH.
    The default addre-size attribute for data segments used as stack is controlled by the B flag of the segment's descriptor.
  * The attribute of the explicit address is determined by the D flag of the current code segment and the presence or absence of the 67H address-size prefix.

# Stack Behavior in 64-Bit Mode
  * 64位模式的时候，SS segment的基地址为0.Fields(base, limit, and attribute) in segment descriptor registers are ignored.
    SSregister的DPL is modified that it is always equal to CPL. This will be true even if it is the only field in the ss descriptor that is modified.
  * ESP, EIP, EBP寄存器扩展为64 bits，并重命名为RSP, RIP, RBP.
  * PUSH/POP指令以64-bit的宽度增加/减少堆栈。 When the contents of a segment register is pushed onto 64-bit stack, the pointer is automatically aligned to 64 bits (as with a stack that has a 32-bit width).