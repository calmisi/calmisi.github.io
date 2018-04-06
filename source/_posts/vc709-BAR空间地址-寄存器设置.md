---
title: vc709 BAR空间地址&寄存器设置
date: 2017-03-30 14:53:42
tags:
- DMA
- VC709
categories: vc709 DMA 驱动
---

本文记录Xilinx VC709这块开发板的PCI BAR空间的设置相关信息。  
<!-- more -->
- 所有的Registers都是32 bits宽度的，从左只有依次是31位到0位；  
- 所有没有被定义的bits都是保留的，当被读的时候默认返回0；
- reset之后，所有的registers都返回默认值；
- 所有的registers都被mapped to `BAR0`


# DMA
> **The Scatter Gather Packet DMA IP is provided by Northwest Logic.**
> **The DMA controller requires a `64KB` register space mapped to BAR0.**
> **All DMA registers are mapped to BAR0 from `0x0000` to `0x7FFF`. The address range from `0x8000` to `0xFFFF` is availabel to you by way of this interface.**
> **`The front-end` of the DMA interfaces to the AXI4-stream interface on PCIe Endpoint IP core.**
> **`The backend` of the DMA provides an AXI4-stream interface as well, which connects to the user appplication side.**


- Register address offsets from 0x0000 to 0x7FFF on `BAR0` 被DMA engine 自身使用
- Address offset space on `BAR0` from 0x800 to 0xFFFF is provided to user
    - 0x9000 to 0x9FFF 分配给了user space registers
    - 0xB000 to 0xEFFF 分配给了4个MACs （使用了1x5 AXI4LITE Interconnect 来route 不同的request 到对应的slave）
``` c
      DMA Channel | Offset from BAR0
    Channel 0 S2C |   0x0
    Channel 1 S2C |   0x100
    Channel 2 S2C |   0x200
    Channel 3 S2C |   0x300
    Channel 0 C2S |   0x2000
    Channel 1 C2S |   0x2100
    Channel 2 C2S |   0x2200
    Channel 3 C2S |   0x2300
```
