---
title: 'xraw_data driver解析 '
date: 2017-03-09 22:27:30
tags: 
- DMA
- VC709
categories: vc709 DMA 驱动
---

Xilinx VC709开发板，对应的程序中有个performance mode，是查看DMA性能的。
该板子的DMA驱动为xdma_v7.ko，在路径`linux_driver_app/driver/xdma`中。
选择performance mode后，该模式只使用一个DMA engine(即engine 0)，其有2个通道分别为TX和RX，
对应的设备驱动为xrawdata0.ko，在路径`linux_driver_app/driver/xrawdata0`中,对应的驱动源程序为`sgusr.c`。

xraw_data0的驱动需要向XDMA的驱动注册，以完成相应的接口，其在系统中的接口为一个字符文件`/dev/xraw_data0`。

下面我们具体来看`sgusr.c`文件的源码。
<!-- more -->
---
# 1.文件前面是各种定义，寄存器地址等。
第277-278行，定义RawTestMode = TEST_STOP,即初始化时，没有开始测试。
RawMinPktSize为64 Byte,RawMaxPktSize为4个内存页(4page=4*4096=32768 Byte)。


---
# 2.初始化函数 
static int __int rawdata_int(void)

第1457行，初始化xraw_data0的状态为 INITIALIZED,后面依次初始化3个spin_lock.
第1471行，调用`alloc_chrdev_region（）`，请求系统给xrawDev设备分配设备号。
如果设备号分配成功，申请代表字符设备的`struct cdev`结构来初始化`xrawCdev`指针,成功后，初始化对应的文件操作函数到这个设备上，即初始化`file_operations`类型的xrawDevFileOps结构体（1489-1494行），最后第1498行调用`cdev_add（）`将设备加入内核。


xraw_data0作为使用DMA的应用，需要实现xdma驱动的对应功能的callback函数，想DMA注册。
从第1516行开始，对应的先指定ufuncs中对应的callback，然后使用`DmaRegister()`将自己注册到base DMA驱动中，比如第1542行
`DmaRegister (ENGINE_TX, MYBAR, &ufuncs, BUFSIZE)) `，将xraw_data0应用注册XDMA的ENGINE_TX号引擎上，对应的BAR寄存器地址为MYBAR,对应的函数调用为ufuncs,正常的包大小为BUFFSIZE.
DmaRegister()的实现在xdma驱动源码的`xdma_user.c`中
{% codeblock lang:c %}
/*****************************************************************************/
/**
 * This function must be called by the user driver to register itself with
 * the base DMA driver. After doing required checks to verify the choice
 * of engine and BAR register, it initializes the engine and the BD ring
 * associated with the engine, and enables interrupts if required. 
 *
 * Only one user is supported per engine at any given time. Incase the
 * engine has already been registered with another user driver, an error
 * will be returned.
 *
 * @param engine is the DMA engine the user driver wants to use.
 * @param bar is the BAR register the user driver wants to use.
 * @param uptr is a pointer to the function callbacks in the user driver.
 * @param pktsize is the size of packets that the user driver will normally
 *        use.
 *
 * @return NULL incase of any error in completing the registration.
 * @return Handle with which the user driver is registered.
 *
 * @note This function should not be called in an interrupt context 
 *
 *****************************************************************************/
void * DmaRegister(int engine, int bar, UserPtrs * uptr, int pktsize)
{
	Dma_Engine * eptr;
#ifdef X86_64
	u64 barbase;
#else	
	u32 barbase;
#endif	
	int result;

	log_verbose(KERN_INFO "User register for engine %d, BAR %d, pktsize %d\n",
			engine, bar, pktsize);
#ifdef PM_SUPPORT
	if(DriverState == PM_PREPARE)     
	{
		log_verbose(KERN_ERR "DMA driver state %d - entering into Power Down states\n", DriverState);
		return NULL;
	}
#endif

	if(DriverState != INITIALIZED)
	{
		log_verbose(KERN_ERR "DMA driver state %d - not ready\n", DriverState);
		return NULL;
	}

	if((bar < 0) || (bar > 5)) {
		log_verbose(KERN_ERR "Requested BAR %d is not valid\n", bar);
		return NULL;
	}

	if(!((dmaData->engineMask) & (1LL << engine))) {
		log_verbose(KERN_ERR "Requested engine %d does not exist\n", engine);
		return NULL;
	}
	eptr = &(dmaData->Dma[engine]);
#ifdef X86_64
	barbase = (dmaData->barInfo[bar].baseVAddr);
#else	
	barbase = (u32)(dmaData->barInfo[bar].baseVAddr);
#endif
	if(eptr->EngineState != INITIALIZED) {
		log_verbose(KERN_ERR "Requested engine %d is not free\n", engine);
		return NULL;
	}

	/* Later, add check for reasonable packet size !!!! */

	/* Later, add check for mandatory user function pointers. For optional
	 * ones, assign a stub function pointer. This is better than doing
	 * a NULL value check in the performance path. !!!!
	 */

	/* Copy user-supplied parameters */
	eptr->user = *uptr;
	eptr->pktSize = pktsize;

	/* Specify user-specific version information register. Note that this is
	 * in the BAR0 space.
	 */
#ifdef X86_64
	uptr->versionReg = (dmaData->barInfo[0].baseVAddr) + 0x9000;
#else	 
	uptr->versionReg = (u32)(dmaData->barInfo[0].baseVAddr) + 0x9000;
#endif
	if ((uptr->UserInit)(barbase, uptr->privData) < 0)
	{
		log_verbose(KERN_ERR "Initialization unsuccessful\n");
		return NULL;
	}

	spin_lock_bh(&DmaLock);

	/* Should inform the user of the errors !!!! */
	result = descriptor_init(eptr->pdev, eptr);
	if (result) {
		/* At this point, handle has not been returned to the user.
		 * So, user refuses to prepare buffers. Will be trying again in
		 * the poll_routine. So, do not abort here.
		 */
		printk(KERN_ERR "Cannot create BD ring, will try again later.\n");
		//return NULL;
	}

	/* Change the state of the engine, and increment the user count */
	eptr->EngineState = USER_ASSIGNED;
	dmaData->userCount ++ ;

	/* Start the DMA engine */
	if (Dma_BdRingStart(&(eptr->BdRing)) == XST_FAILURE) {
		log_normal(KERN_ERR "DmaRegister: Could not start Dma channel\n");
		return NULL;
	}

#ifdef TH_BH_ISR
	printk("Now enabling interrupts\n");
	Dma_mEngIntEnable(eptr);
#endif

	spin_unlock_bh(&DmaLock);

	log_verbose(KERN_INFO "Returning user handle %p\n", eptr);

	return eptr;
}
{% endcodeblock%}

---
# 3.DMA写函数
{% codeblock lang:c %}
xraw_dev_write (struct file *file, const char __user * buffer, size_t length, loff_t * offset)
{% endcodeblock %}
其中调用了
{% codeblock lang:c %}
DmaSetupTransmit(handle[0], 1, buffer, length)
{% endcodeblock %}
ENGINE_TX:只实现了ufuncs.UserPutPkt = myPutTxPkt;(用来将packet buffer还给application-specific dirver,以便reuse,对应代码里面更新TxSync)
ENGINE_RX:实现了ufuncs.UserPutPkt = myPutRxPkt; 和ufuncs.UserGetPkt = myGetRxPkt;

即write对应的是handle[0],即ENGINE_TX发送。