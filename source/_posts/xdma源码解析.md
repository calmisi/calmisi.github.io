---
title: xdma源码解读
date: 2017-03-15 20:34:49
tags:
- DMA
- VC709
categories: vc709 DMA 驱动
---


1. A buffer descriptor defines a DMA transaction.
2. the Dma_Bd结构体定义了BD;
3. 

# xdma.h
xdma驱动支持以下特性：
1.  Scatter-Gather DMA (SGDMA)
2.  Interrupts
3.  Interrupts are coalesced by the driver to improve performance
4.  32-bit addressing for Buffer Descriptors (BDs) and data buffers 
5.  APIs to manage BD movement to and from the SGDMA engine
6.  Virtual memory support
7.  PCI Express support
8.  Performance measurement statistics
8.  APIs to enable DMA driver to be used by other application drivers
<!-- more -->
关于一个DMA传输，需要source address,destination address, the number of bytes to transfer.
receive channel, sourec address在硬件中，不需要指定;destination address为系统buffer。
transmit channel，则destination address不要指定。

A packet is defined as a series of data bytes that represent a message. SGDMA allows a packet of data to be broken up into one or more `transactions`. For example, take an Ethernet IP packet which consists of a 14-byte header followed by a data payload of one or more bytes. With SGDMA, the application may point a BD to the header  and another BD to the payload, then transfer them as a single message. This strategy can make a TCP/IP stack more efficient by allowing it to  keep packet headers and data in different memory regions instead of assembling packets into contiguous blocks of memory.

---
<b> Software Initialization </b>
	* The driver does the following steps in order to prepare the DMA engine
	* to be ready to process DMA transactions:
	*
	* - DMA Initialization using Dma_Initialize() function. This step
	*   initializes a driver instance for the given DMA engine and resets the
	*   engine. One driver instance exists for a pair of (S2C and C2S) engines.
	* - BD Ring creation. A BD ring is needed per engine and can be built by
	*   calling Dma_BdRingCreate(). A parameter passed to this function is the
	*   number of BDs fit in a given memory range, and Dma_mBdRingCntCalc() helps
	*   calculate the value.
	* - (RX channel only) Prepare BDs with attached data buffers and give them to
	*   the RX channel. First, allocate BDs using Dma_BdRingAlloc(), then populate
	*   data buffer address, data buffer size and the control word fields of each
	*   allocated BD with valid values. Last call Dma_BdRingToHw() to give the
	*   BDs to the channel.
	* - Enable interrupts if interrupt mode is chosen. The application is
	*   responsible for setting up the interrupt system, which includes providing
	*   and connecting interrupt handlers and call back functions, before
	*   the interrupts are enabled.
	* - Start DMA channels: Call Dma_BdRingStart() to start a channel
<b> How to start DMA transactions </b>
	* RX channel is ready to start RX transactions once the initialization (see
	* Initialization section above) is finished. The DMA transactions are triggered
	* by the user IP (like Local Link TEMAC).
	*
	* Starting TX transactions needs some work. The application calls
	* Dma_BdRingAlloc() to allocate a BD list, then populates necessary
	* attributes of each allocated BD including data buffer address, data size,
	* and control word, and last passes those BDs to the TX channel
	* (see Dma_BdRingToHw()). The added BDs will be processed as soon as the
	* TX channel reaches them.
<b> Software Post-Processing on completed DMA transactions </b>
	* Some software post-processing is needed after DMA transactions are finished.
	*
	* If interrupts are set up and enabled, DMA channels notify the software
	* the finishing of DMA transactions using interrupts,  Otherwise the
	* application could poll the channels (see Dma_BdRingFromHw()).
	*
	* - Once BDs are finished by a channel, the application first needs to fetch
	*   them from the channel (see Dma_BdRingFromHw()).
	* - On TX side, the application now could free the data buffers attached to
	*   those BDs as the data in the buffers has been transmitted.
	* - On RX side, the application now could use the received data in the buffers
	*   attached to those BDs
	* - For both channels, those BDs need to be freed back to the Free group (see
	*   Dma_BdRingFree()) so they are allocatable for future transactions.
	* - On RX side, it is the application's responsibility for having BDs ready
	*   to receive data at any time. Otherwise the RX channel will refuse to
	*   accept any data once it runs out of RX BDs. As we just freed those hardware
	*   completed BDs in the previous step, it is good timing to allocate them
	*   back (see Dma_BdRingAlloc()), prepare them, and feed them to the RX
	*   channel again (see Dma_BdRingToHw())

<b>Address Translation</b>
	* When the BD list is setup with Dma_BdRingCreate(), a physical and
	* virtual address is supplied for the segment of memory containing the
	* descriptors. The driver will handle any translations internally. Subsequent
	* access of descriptors by the application is done in terms of their virtual
	* address.
	*
	* Any application data buffer address attached to a BD must be physical
	* address. The application is responsible for calculating the physical address
	* before assigns it to the buffer address field in the BD.

<b> Memory Barriers </b>
	* The DMA hardware expects the update to its Next BD pointer register to be
	* the event which initiates DMA processing. Hence, memory barrier wmb() calls
	* have been used to ensure this.

<b>Alignment</b>
	* <b> For BDs: </b>
	* The Northwest Logic DMA hardware requires BDs to be aligned at 32-byte
	* boundaries. In addition to the this, the driver has its own alignment 
	* requirements. It needs to store per-packet information in each BD, for 
	* example, the buffer virtual address. In order to do this, the software 
	* view of the BD may be larger than the hardware view of the BD. For example, 
	* DMA_BD_SW_NUM_WORDS can be set to 16 words (64 bytes), even though 
	* DMA_BD_HW_NUM_WORDS is 8 words (32 bytes). Due to this, the driver
	* gets additional space in which to store per-BD private information.
	*
	* Minimum alignment is defined by the constant DMA_BD_MINIMUM_ALIGNMENT.
	* This is the smallest alignment allowed by both hardware and software for them
	* to properly work. Other than DMA_BD_MINIMUM_ALIGNMENT, multiples of the
	* constant are the only valid alignments for BDs.
	*
	* If the descriptor ring is to be placed in cached memory, alignment also MUST
	* be at least the processor's cache-line size. If this requirement is not met
	* then system instability will result. This is also true if the length of a BD
	* is longer than one cache-line, in which case multiple cache-lines are needed
	* to accommodate each BD.
	*
	* Aside from the initial creation of the descriptor ring (see
	* Dma_BdRingCreate()), there are no other run-time checks for proper
	* alignment.

<b>For application data buffers:</b>
	* Application data buffer alignment is taken care of by the 
	* application-specific drivers.

<b>Reset After Stopping</b>
	* This driver is designed to allow for stop-reset-start cycles of the DMA
	* hardware while keeping the BD list intact. When restarted after a reset, this
	* driver will point the DMA engine to where it left off after stopping it.
	*
	* It is possible to load an application-specific driver, run it for some
	* time, and then unload it. Without unloading the DMA driver as well, it
	* should be possible to load another instance of the application-specific
	* driver and it should work fine.



# xdma_bd.h
描述Buffer Descriptor
