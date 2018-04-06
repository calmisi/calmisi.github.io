---
title: vc709 DMA 驱动程序源码理解
date: 2017-03-19 21:31:54
tags: 
- DMA
- VC709
categories: vc709 DMA 驱动
---

看了一周的vc709这块板子自带的驱动程序的源码，现作点总结，以便往后回顾。
<!-- more -->
该dma驱动分为2个部分，一部分为base,即xdma源代码文件夹；另一部分为user application specific部分。
这样把驱动分为基础部分，和用户自定义的部分，方便了不同的用户根据自己应用的需求来定制dma功能。

---

# user-application-specific
我们先来看user application specific的部分，我选取的是xrawdata0这个驱动程序源码，该驱动很简单就是测试vc709 dma performance模式对应的应用驱动。
源码文件为`sguser.c`，该驱动对应的一个字符设备为`/dev/xraw_data0`

关于字符设备的系统调用，有以下这些函数
{% codeblock lang:c %}
static int xraw_dev_open(struct inode *in, struct file * filp);
static int xraw_dev_release(struct inod *in, struct file*filp);
static long xraw_dev_ioctl (struct file *filp, unsigned int cmd, unsigned long arg);
static ssize_t xraw_dev_read (struct file *file, char __user * buffer, size_t length, loff_t * offset);
static ssize_t xraw_dev_write (struct file *file, const char __user * buffer, size_t length, loff_t * offset);
static int __init rawdata_init (void);
static void __exit rawdata_cleanup (void);
{% endcodeblock %}

并且还有对应的userFunctions调用，usr-application-specific的dma驱动实现这些函数调用，因为xdma没有实现这些，在具体的xdma是需要这些函数的
{% codeblock lang:c %}
int myInit (u64 barbase, unsigned int );
int myGetRxPkt (void *, PktBuf *, unsigned int, int, unsigned int);
int myPutTxPkt (void *, PktBuf *, int, unsigned int);
int myPutRxPkt (void *, PktBuf *, int, unsigned int);
int mySetState (void *hndl, UserState * ustate, unsigned int privdata);
int myGetState (void *hndl, UserState * ustate, unsigned int privdata);
{% endcodeblock %}

---
## user application specific驱动注册
下面我们来看具体的user-application-specific的驱动是如何和xmda驱动关联起来工作的。
首先我们看`xrawdata_init()`函数，
该函数前面都是注册其对应的字符设备的工作，重点看if (xraw_DriverState < POLLING)里面的内容，
该段代码向xmda驱动注册2个dma engine,
首先指定ufuncs的函数句柄; privData(辨别是TX还是RX dma engine); mode(是raw ethernet mode还是performance mode);
最后调用`handle[0] = DmaRegister (ENGINE_TX, MYBAR, &ufuncs, BUFSIZE)`向xdma对应的engine注册，
第一个参数ENGINE_TX:要注册的DMA ENGINE号（xdma 总共有可以有64个DMA engine,但是硬件只实现了8个为0,1,2,3 for TX; 32,33,34,35 for Rx）;
第二个参数MYBAR:指定是BAR0-BAR5中哪个一个，这是为0，即BAR0；
第三个参数&funcs：上面所说的函数调用的句柄；
第四个参数BUFSIZE：该engine所处理的数据包的通常大小，这里为一个内存页的大小，即4096 B;
该函数返回的handle为对应的engine的结构体的指针：Dma_engine * eptr;该结构体为xdma驱动定义，

而DmaRegister()函数在`xdma_user.c`中实现，即为xmda驱动的一部分，
下面分析该函数都作了些什么：
1. 首先，将传进来的第三个参数和四个参数赋给对应的Engine: eptr->user = * uptr, eptr->pktSize = pktsize;
2. 然后，设置uptr->versionReg为BAR0的虚拟地址+0x9000; //0x9000之后的地址为user space registers
3. 该函数调用了userFunctions->UserInit(即sgusr.c中的myInit);
4. 然后调用了descriptor_init(eptr->dev, eptr)，初始化该DMA Engine的BDring；
该函数调用了`Dma_BdRingCreate()`创建该Engine对应的BDring,
如果是RX Engine，还需要调用DmaSetupRecvBuffers(pdev, eptr)；为BDring中的BD关联对应的pkt buffer，此时需要调用`Dma_BdRingAlloc()`和`Dma_BdRingToHw()`,具体说明可以看`xdma.h`中<b> Software Initialization </b>这一节，
需要注意的是，该段代码只有在ETHERNET_APPMODE才会初始化RX engine.
5. 当BDring初始化成功之后，最后调用`Dma_BdRingStart()`启动该dma engine

可见，大多数工作都是xdma的驱动完成的，user application驱动只用到了myInit();

## user application-specific DMA write(TX)逻辑
具体为`sguser.c`文件中的`xraw_dev_write()`函数，
用户程序要使用DMA发送数据的时候，使用系统调用write(fd,char * buffer,size),驱动对应的调用`xraw_dev_write`，
函数内部判断如果testmode是start并且是checker或者loopback，则可以TX，对应调用`DmaSetupTransmit(handle[0],1,buffer,length)`，其中handle[0]，即为该application-specific驱动注册的TX Engine,第二个参数看了函数具体代码后，发觉是没用的。
DmaSetupTransmit()函数，首先为用户空间的buffer创建对应的cache pages，然后动态创建发送用的pktBuf，并用cache pages初始化，最后调用`DmaSendPages_TX(hndl, pkts, allocPages)`启动TX；
`DmaSendPages_TX()`函数为`xdma`提供，该函数的第一参数为TX engine的handle,第二个参数为用户要发送数据的pktbuf的数组，第三个参数为该pktbuf数组的大小，也是要TX的数组占用了多少个内存page,因为一个pktbuf最多只能表示一个page的数据；

* `PktBuf`结构体对应software BD,`Dma_Bd`对应硬件BD,
DmaSendPages_TX()函数用hndl对应的engine TX allocPages个PktBuf(存在pkts数组中),其中以PktBuf为软件BD,指定的buffer最大为一个内存Page,
pkts数组可以包含多个packet,一个packet可以由n(n>=1)个PktBuf表示，根据PktBuf->flags字段可以有`PKT_SOP`,`PKT_EOP`，用此间隔pkts数组中不同的packet由哪些pktbuf组成，
另外还有`PKT_ALL`，当指向一个packet的多个pktbuf的flag设置了此字段，则`DmaSendPages_TX()`会依次检查组成这个packet的所有pktbuf,如果没有含`PKT_EOP`的pktbuf,则不发送这个packet,也就是说设置了这个字段后，TX的时候会检查包的完整性。



---
# How to start MDA transactions
1. RX channel
RX比较简单，在初始化阶段，也就是DMA register阶段，已经为RX engine调用了`DmaSetupRecvBuffers(pdev, eptr)`,只用等待硬件触发DMA transactions就行了。
2. TX channel
在开始TX时，需要一些准备工作。
首先应用调用`Dma_BdRingAlloc()`分配一个BD表，然后初始化这些BDs的data buffer address, data size, and control word, etc...；
然后调用`Dma_BdRingToHw()`将这些BDs传递给硬件操作，
具体过程参见`xdma_user.c`中的`DmaSendPkt(void * handle, PktBuf * pkts, int numpkts)`和`DmaSendPages_Tx(void * handle, PktBuf ** pkts, int numpkts)`

其中DmaSendPkt中用的是pci_map_single()，将要发送的包的起始地址和包长度进行映射得到总线地址（x86平台即物理地址）;
DmaSendPages_Tx中用的是pci_map_page()，将要发送的包的内存页的起始地址，页内偏移和，包长度进行page映射，得到总线地址;
**note: pci_map_page()用在高端内存中**;






---
# xdma驱动
该部分驱动为dma 的base 驱动，在xdma文件中，
首先我们看`xdma_base.c`文件，在文件最后有：
{% codeblock lang:c %}
module_init(xdma_init);
module_exit(xdma_cleanup);
{% endcodeblock %}
可见模块insmod的时候，调用`xdma_init()`函数，在`xdma_init()`函数中，
只初始化4个自旋锁，并调用`pci_register_driver(&xdma_driver)`；
在xdma_driver结构体中，指定了`probe()`函数,调用pci_register_driver()会触发`xdma_probe()`;
所以载入xdma驱动的时候，主要工作都在`probe()`中，下面我们来看`probe()`函数：

---
{% codeblock lang:c %}
static int  xdma_probe(struct pci_dev *pdev, const struct pci_device_id *ent);
{% endcodeblock %}
首先调用pci_enable_device(pdev)使能xdma的pci设备;
然后在for循环中初始化10个pktPool，使他们连成链表，并且每个都指向pktArrray[i],(pktArray[10][1999]);
然后动态分配一个`priData`结构体类型的变量dmaData,该结构体表示了VC709开发板的pci device的私有数据:
{% codeblock lang:c %}
struct privData {
	struct pci_dev * pdev;          /**< PCI device entry */
	/** BAR information discovered on probe. BAR0 is understood by this driver.
	 * Other BARs will be used as app. drivers register with this driver.
	 */
	u32 barMask;                    /**< Bitmask for BAR information */
	struct {
		unsigned long basePAddr;    /**< Base address of device memory */
		unsigned long baseLen;      /**< Length of device memory */
		void __iomem * baseVAddr;   /**< VA - mapped address */
	} barInfo[MAX_BARS];			//MAX_BARS=6

	u32 index;                    /**< Which interface is this */

	/**
	 * The user driver instance data. An instance must be allocated
	 * for each user request. The user driver request will request separately
	 * for one C2S and one S2C DMA engine instances, if required. The DMA
	 * driver will not allocate a pair of engine instances on its own.
	 */
	long long engineMask;           /**< For storing a 64-bit mask */ //因为有64个engine,每位一个mask.
	Dma_Engine Dma[MAX_DMA_ENGINES];/**< Per-engine information */ //MAX_DMA_ENGINES=64 最多64个DMA ENGINE 

	int userCount;                  /**< Number of registered users */
};
{% endcodeblock %}
然后初始化dmaData的barMask,engineMask和userCount,
然后调用`pci_set_master(pdev)`将该pci设备设置设为 bus_mastering模式，
后面调用了pci_request_regions，我也不知道是干什么的。
然后根据系统位数，调用pci_set_dma_mask()检查总线是否可以接收给定大小的总线地址(mask)，如果可以，则通知总线层给定的外围设备将使用该大小的总线地址。
后面的for循环，读取bar配置空间的数据，
{% codeblock lang:c %}
//从配置区相应寄存器得到I/O区域的内存区域长度：
pci_resource_length(struct pci_dev *dev,  int bar)    Bar值的范围为0-5。
//从配置区相应寄存器得到I/O区域的内存的相关标志：
pci_resource_flags(struct pci_dev *dev,  int bar)    Bar值的范围为0-5。
//从配置区相应寄存器得到I/O区域的基址：
pci_resource_start(struct pci_dev *dev,  int bar)    Bar值的范围为0-5。
{% endcodeblock %}
先检查配置了哪些BAR地址空间，bar0是必须的，
然后判断存在的BAR空间是不是memory-mapped，该驱动只支持memory-mapped.
然后依次设置damDate结构体中各个BAR空间的basePAddr（bar基地址,是物理地址），size(bar空间长度),baseVAddr(基 虚拟地址)

然后disable中断，
设置dmaData的pdev，index,

然后调用`ReadDMAEngineConfiguration(pdev,dmaData)`函数配置Dma Engine;
* 具体来查看`ReadDMAEngineConfiguration(struct pci_dev * pdev, struct privData * dmaInfo)`
{% codeblock lang:c %}
dma Engine的配置信息都在BAR0空间的寄存器中，
根据UG962可知，该trd的程序用到了8个DMA Engine，对应的offset为：
DMA Channel 	Offset from BAR0
Channel 0 		S2C 0x0
Channel 1 		S2C 0x100
Channel 2 		S2C 0x200
Channel 3 		S2C 0x300
Channel 0 		C2S 0x2000
Channel 1 		C2S 0x2100
Channel 2 		C2S 0x2200
Channel 3 		C2S 0x2300
{% endcodeblock %}
for循环的`offset=0,i=0; offset < 64 * 0x100; offset += 0x100, i++`
i=0,1,2,3和32,33,34,35时，if(val & DMA_ENG_PRESENT_MASK)有效，调用`DMA_Initialize()`初始化了对应dmaData.DMA[0,1,2,3,32,33,34,35]，并reset.
初始化内容有：
1.首先将对应的Dma_Engine对应的指针区域用memset置零;
2.设置该Engine的RegBase(寄存器基地址)为BaseAddress(即上面对应的channel的地址)，Type(是c2s还是s2c);
3.设置对应的BDring的状态为`XST_DMA_SG_IS_STOPPED`(没有在运行),设置Engine的状态为INITIALIZED;
4.设置对应的BDring的chanBase地址为BaseAddress,和BDring的IsRxChannel字段;
5.最后Reset对应的Engine

调用pci_set_drvdata(pdev, dmaData)，将dmaData设置为该pci设备的私有数据，
然后初始化xdma_driver对应的字符设备，
然后将xdma_driver的DriverState设为INITIALIZED;
然后启动轮询的计时器：
{% codeblock lang:c %}
init_timer(timer);
timer->expires=jiffies+(HZ/500);   	//2ms之后启动
timer->data=(unsigned long) pdev; 	//传递给function的数据，
timer->function = poll_routine;
add_timer(timer);
{% endcodeblock %}

* 调用的函数为`poll_routine()`,我们来看该函数
{% codeblock lang:c %}
static void poll_routine(unsigned long __opaque)
{
	struct pci_dev *pdev = (struct pci_dev *)__opaque;
	Dma_Engine * eptr;
	struct privData *lp;
	int i, offset;

	if(DriverState == UNREGISTERING)
		return;

	lp = pci_get_drvdata(pdev);

	for(i=0; i<MAX_DMA_ENGINES; i++) 
	{
#ifdef TH_BH_ISR
		/* Do housekeeping only if adequate time has elapsed since
		 * last ISR.
		 */
		if(jiffies < (LastIntr[i] + (HZ/50))) continue;
#endif

		if(!((lp->engineMask) & (1LL << i)))
			continue;

		eptr = &(lp->Dma[i]);
		if(eptr->EngineState != USER_ASSIGNED)
			continue;

		/* The spinlocks need to be handled within this function, so
		 * don't do them here.
		 */
		PktHandler(i, eptr);
	}

	/* Reschedule poll routine. Incase interrupts are enabled, the
	 * bulk of processing should happen in the ISR. 
	 */
#ifdef TH_BH_ISR
	offset = HZ / 50;
#else
	offset = 0;
#endif
	poll_timer.expires = jiffies + offset;
	add_timer(&poll_timer);
}
{% endcodeblock %}
如果DriverState为正在unregistering，则什么都不干；
重点注意for循环中，为每个启用的DMA Engine调用了PktHandler(i, eptr);
最后在重新调用该轮询函数(注意在probe函数中，struct timer_list * timer = &poll_timer; 当时已经指定了function)
所以重点是`PktHandler()`
{% codeblock lang:c %}PktHandler(int eng, Dma_Engine * eptr){% endcodeblock %}
首先调用`if ((bd_processed = Dma_BdRingFromHw(rptr, DMA_BD_CNT, &BdPtr)) > 0)`
如果硬件产生了DMA操作，返回的BDs不为零，则进行操作，具体的操作是：
* `BdPtr`指向post-processing的Bd，即硬件操作完的BD;
取消之前硬件DMA操作占用的内存page的dma映射(`pci_unmap_page()`);
调用`Dma_BdRingFree()`将post-processing的Bd置为free;
然后调用`(uptr->UserPutPkt)(eptr, ppool->pbuf, bd_processed_save, uptr->privData)`将pktbuf还给application;
最后检查是否为ETHERNET_APPMODE,如果是，并且对应的Engine是RX engine，则调用`DmaSetupRecvBuffers(pdev, eptr)`，为RX的BDring关联对应的buffer;





