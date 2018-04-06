---
title: Xlinx vc709 trd DMA驱动源码解读
date: 2016-12-29 21:20:42
tags:
- Driver source code
categories: vc709 DMA 驱动
---

# raw ethernet performance测试

在运行`Raw Ethernet Perfomance Mode`的时候,
安装驱动运行的脚本为`/v7_xt_conn_trd/software/linux/v7_xt_conn_trd/linux_driver_app/run_raw_ethermode.sh`
其内容为：
{% codeblock lang:bash %}
#!/bin/sh
compilation_error=1
module_insertion_error=2
compilation_clean_error=3

pgrep App|xargs kill -SIGINT 1>/dev/null 2>&1
sleep 5;
cd App
make clean 1>/dev/null 2>&1
make APP_MODE=RAWETHERNET 1>/dev/null 2>&1
./App 1>Applog  2>&1 &
cd ../

/bin/sh remove_modules.sh
cd driver
make DRIVER_MODE=RAWETHERNET clean
if [ "$?" != "0" ]; then
	echo "Error in cleaning RAW_ETHERNET performance driver"
	exit $compilation_clean_error;
fi
make DRIVER_MODE=RAWETHERNET
if [ "$?" != "0" ]; then
	echo "Error in compiling RAW_ETHERNET performance driver"
	exit $compilation_error;
fi
sudo make DRIVER_MODE=RAWETHERNET insert 
if [ "$?" != "0" ]; then
	echo "Error in inserting RAW_ETHERNET performance driver"
	exit $module_insertion_error;
fi
{% endcodeblock %}
可以看到6-12行代码，先判断系统中是否有App程序在运行，如果有就Kill掉。
然后等待5秒后，进入App目录，重新编译App程序，编译完成后以RAWETHERNET模式运行APP。

<!-- more -->

可见App程序一直在运行，然后我们看App程序的源码，在`/v7_xt_conn_trd/software/linux/v7_xt_conn_trd/linux_driver_app/App/threads.c`
可以发现Main()函数中的无限for循环中有这么一段代码：
{% codeblock lang:c %}
if (FD_ISSET(s, &read_set))
{
	if ((n = recvfrom(s, &testCmd, sizeof(testCmd), 0,(struct sockaddr *)&remote, &t)) == -1) 
	{
		perror("recvfrom");
			continue ;
	}
	printf("Received command for engine %d and max pkt size as %d\n",testCmd.Engine,testCmd.MaxPktSize);
	if(testCmd.TestMode & TEST_START)
		StartTest(testCmd.Engine,testCmd.TestMode , testCmd.MaxPktSize);
	else 
		StopTest(testCmd.Engine,testCmd.TestMode , testCmd.MaxPktSize);
}
{% endcodeblock %}
App程序的socket一直在监听其他进程的消息，当有revefrom()接收到消息之后，根据testCmd中的TestMode来选择开始测试或者停止测试。
{% codeblock lang:c %}int StartTest(int engine, int testmode, int maxSize){% endcodeblock %}
在StartTest内部会根据engine号，testmode(start or stop)，和包的大小来调用`xlnx_thread_create(&txthread_write, &testcmd0);`,
`xlnx_thread_create()`会在一个新的线程里调用`txthread_write`函数，参数为`testcmd0`,

下面我们来看`txthread_write(TestCmd *test)`函数，
其主要函数体如下：
{% codeblock lang:c %}
void *  txthread_write(TestCmd *test)
{
	if(PacketSize % 4){
		chunksize = PacketSize + (4 - (PacketSize % 4));
	}
	else
	{
		chunksize = PacketSize;
	}
	bufferLen = BUFFER_SIZE - (BUFFER_SIZE % (chunksize * 512)); 
	//log_verbose("thread TxWrite %d started engine is %d size %d \n", id ,test->Engine,test->MaxPktSize);
	buffer =(char *) valloc(bufferLen);
	if(!buffer)
	{
		printf("Unable to allocate memory \n"); 
		exit(-1);
	}
	...
	//initialize the available memory with the total memory.
	if (0 != initMemorySync(TxDoneSync, bufferLen))
	{
		//error
		perror("Bad Pointer TxDoneSync: MemorySync");
	}
	FormatBuffer(buffer,bufferLen,chunksize,PacketSize);
	
while(1)
	{
		if(0 == ReserveAvailable(TxDoneSync, chunksize))
		{
			if(PacketSent + chunksize <= bufferLen )
			{
				ret=write(file_desc,buffer+PacketSent,PacketSize);
				if(ret < PacketSize)
				{
					FreeAvailable(TxDoneSync, chunksize); 
					//TODO  
				}
				else
				{
					PacketSent = PacketSent + chunksize;
				}
			}
			else
			{
				FreeAvailable(TxDoneSync,chunksize);
				PacketSent = 0;
			}
		}
	...
	}
}
{% endcodeblock %}
注意25行调用了`FormatBuffer(buffer,bufferLen,chunksize,PacketSize);`函数，
看之前的代码可以发现chunksize是PacketSize取顶刚好是4的倍数,bufferlen刚好可以512*chunksize的倍数。
这个函数初始化从指针buffer开始的bufferlen个字节的内存空间，并以chunksize为单位初始化一个packet，其头2个Byte为PacketSize，往后每2B存放Packet包编号，
下一个Packet，头2个字节依旧是PacketSize，后面的编号++，一直到512后就重新置0。

注意33行调用了`write(file_desc,buffer+PacketSent,PacketSize)`，
即，向对应的`/dev/xraw_data0`设备文件写入生成缓存包，从buffer地址开始，一次写入PacketSize大小的包。
`write()`对应`xrawdata0/sguser.c`里面的
{% codeblock lang:c %}
static ssize_t xraw_dev_write (struct file *file, const char __user * buffer, size_t length, loff_t * offset)
{
	int ret_pack=0;


	if ((RawTestMode & TEST_START) &&
			(RawTestMode & (ENABLE_PKTCHK | ENABLE_LOOPBACK)))
		ret_pack = DmaSetupTransmit(handle[0], 1, buffer, length);

	/* 
	 *  return the number of bytes sent , currently one or none
	 */
	return ret_pack;
}
{% endcodeblock %}
其调用了`DmaSetupTransmit()`函数：
{% codeblock lang:c %}
static int DmaSetupTransmit(void * hndl, int num ,const char __user * buffer, size_t length)   
{
	offset = offset_in_page(buffer);
	first = ((unsigned long)buffer & PAGE_MASK) >> PAGE_SHIFT;
	last  = (((unsigned long)buffer + length-1) & PAGE_MASK) >> PAGE_SHIFT;
	allocPages = (last-first)+1;

	pkts = kmalloc( allocPages * (sizeof(PktBuf*)), GFP_KERNEL);
	if(pkts == NULL)
	{
		printk(KERN_ERR "Error: unable to allocate memory for packets\n");
		return -1;
	}

	cachePages = kmalloc( (allocPages * (sizeof(struct page*))), GFP_KERNEL );
	if( cachePages == NULL )
	{
		printk(KERN_ERR "Error: unable to allocate memory for cachePages\n");
		kfree(pkts);
		return -1;
	}

	memset(cachePages, 0, sizeof(allocPages * sizeof(struct page*)) );
	down_read(&(current->mm->mmap_sem));
	status = get_user_pages(current,        // current process id
			current->mm,                // mm of current process
			(unsigned long)buffer,      // user buffer
			allocPages,
			WRITE_TO_CARD,
			0,                          /* don't force */
			cachePages,
			NULL);
	up_read(&current->mm->mmap_sem);
	if( status < allocPages) {
		printk(KERN_ERR ".... Error: requested pages=%d, granted pages=%d ....\n", allocPages, status);

		for(j=0; j<status; j++)
			page_cache_release(cachePages[j]);
		kfree(pkts);
		kfree(cachePages);
		return -1;
	}
	allocPages = status;	// actual number of pages system gave

	for(j=0; j< allocPages; j++)		/* Packet fragments loop */
	{
		pbuf = kmalloc( (sizeof(PktBuf)), GFP_KERNEL);

		if(pbuf == NULL) {
			printk(KERN_ERR "Insufficient Memory !!\n");
			for(j--; j>=0; j--)
				kfree(pkts[j]);
			for(j=0; j<allocPages; j++)
				page_cache_release(cachePages[j]);
			kfree(pkts);
			kfree(cachePages);
			return -1;
		}

		//spin_lock_bh(&RawLock);
		pkts[j] = pbuf;

		// first buffer would start at some offset, need not be on page boundary
		if(j==0) {
			if(j == (allocPages-1)) { 
				pbuf->size = length;
			}
			else
				pbuf->size = ((PAGE_SIZE)-offset);
		} 
		else {
			if(j == (allocPages-1)) { 
				pbuf->size = length-total;
			}
			else pbuf->size = (PAGE_SIZE);
		}
		pbuf->pktBuf = (unsigned char*)cachePages[j];		

		pbuf->pageOffset = (j == 0) ? offset : 0;	// try pci_page_map

		pbuf->bufInfo = (unsigned char *) buffer + total;
		pbuf->pageAddr= (unsigned char*)cachePages[j];
		pbuf->userInfo = length;

		pbuf->flags = PKT_ALL;
		if(j == 0)
		{
			pbuf->flags |= PKT_SOP;
		}
		if(j == (allocPages - 1) )
		{
			pbuf->flags |= PKT_EOP;
		}
		total += pbuf->size;
		//spin_unlock_bh(&RawLock);
	}
	/****************************************************************/

	allocPages = j;           // actually used pages

	result = DmaSendPages_Tx (hndl, pkts,allocPages);
	if(result == -1)
	{
		for(j=0; j<allocPages; j++) {
			page_cache_release(cachePages[j]);
		}
		total = 0;
	}
	kfree(cachePages);

	for(j=0; j<allocPages; j++) {
		kfree(pkts[j]);
	}
	kfree(pkts);

	return total;
}
{% endcodeblock %}
其中hndl即为handle[0]，即`/dev/xraw_data0`,从用户空间buffer地址开始，传length长度的数据。
注意SECTION1，
offset为buffer在内存页中的偏移，
first为要传的数据占用的内存第一个页的编号
last为要传的数据占用的内存的最后一个页的编号
allocPages为要传的数据所需要分配的内存页的个数

(PktBuf \*)指向一个描述packet buffer的数据结构，
其中packet buffer最大的大小为一个内存页（4K），而PktBuf指针只能指向一个内存页
所以如果一个Packet占有allocPages个内存页的话，需要allocPages个指针。
`pkts = kmalloc( allocPages * (sizeof(PktBuf*)), GFP_KERNEL);`
pkts指针指向(PktBuf \*)数组头，即分配了allocPages个 (PktBuf \*)的指针

`cachePages = kmalloc( (allocPages * (sizeof(struct page*))), GFP_KERNEL );`
在内核空间开辟一块连续的内存空间来存储要传的数据，大小为allocPages个内存页。
{% codeblock lang:c %}
down_read(&(current->mm->mmap_sem));
status = get_user_pages(current,        // current process id
	current->mm,                // mm of current process
	(unsigned long)buffer,      // user buffer
	allocPages,
	WRITE_TO_CARD,
	0,                          /* don't force */
	cachePages,
	NULL);
up_read(&current->mm->mmap_sem);
{% endcodeblock %}
将buffer指向的要传的数据复制进cachePage中，这里因为buffer是用户空间的地址，而该部分功能属于驱动，是内核态的。
所以需要用get_user_pages，把用户态的数据映射到内核态。

1. get_user_pages()接口真是个好东东，它能获取用户区进程使用内存的某个页(struct page)，然后可以在内核区通过kmap_atomic(), kmap()等函数映射到内核区线性地址，从而可以在内核区向其写入数据。
get_user_pages()的函数声明如下：
{% codeblock lang:c %}
int get_user_pages(struct task_struct *tsk, struct mm_struct *mm, unsigned long start, int len, int write, int force, struct page **pages, struct vm_area_struct **vmas); 
{% endcodeblock %}
其中
tsk ：指定进程，如current表示当前进程
mm ： 进程的内存占用结构，如current->mm，
start ：要获取其页面的起始逻辑地址（也叫线性地址？），它是用户空间使用的一个地址
len ：要获取的页数
write ：是否要对该页进行写入 /* 我不知道如果是写会做什么特别的处理 */
force ：/* 不知道有什么特殊的动作 */
pages ：存放获取的struct page的指针数组
vms ： 返回各个页对应的struct vm_area_struct，可以传入NULL表示不获取，struct vm_area_struct应该是用于组成用户区进程内存的堆的基本元素？没仔细研究
返回值：数返回实际获取的页数，貌似对每个实际获取的页都是给页计数值增1，如果实际获取的页不等于请求的页，要放弃操作则必须对已获取的页计数值减1，即
page_cache_release()，相当于put_page()。
其中
{% codeblock lang:c %}
down_read(&current->mm->mmap_sem);
result = get_user_pages(current, current->mm, user_addr, data->npages, 1, 0, data->pagevec, NULL);
up_read(&current->mm->mmap_sem);
{% endcodeblock %}
mm->mmap_sem应该是个信号锁吧，在处理 current->mm时要先获得锁

后面的`for(j=0; j< allocPages; j++)`循环中，每次都动态分配一个PktBuf，对其进行初始化
初始化完成后，然后调用`DmaSendPages_Tx (hndl, pkts,allocPages);`将数据pkts指向的PktBuf *的packet buffer发送出去。

`DmaSendPages_Tx(void * handle, PktBuf ** pkts, int numpkts)`在xdma_user.c文件中，


---
# xdma源码解读
dma驱动的源码在
`/v7_xt_conn_trd/software/linux/v7_xt_conn_trd/linux_driver_app/driver/xdma`文件夹内,其中：
1. `xdma.h`定义struct Dma_Engine(DMA Engine实例的数据结构)和struct privaData（device的私有数据的数据结构）
2. `xdma.c`定义了Dma_Initialize(Dma_Engine * InstancePtr, u64 BaseAddress, u32 Type)函数和Dma_Reset(Dma_Engine * InstancePtr)函数，分别作为DMA Engine的初始化和重置。

打开`xdma_base.c`文件，
xdma驱动在载入系统内核的时候，会调用`xdma_probe(struct pci_dev *pdev, const struct pci_device_id *ent)`函数，进行驱动的初始化。
在第2890行，`dmaData = kmalloc(sizeof(struct privData), GFP_KERNEL);`为dmaData分配空间。
dmaData的定义在214行:`struct privData * dmaData = NULL;`,是一个privData的指针，用来存放和DMA设备相关的信息。
在`xdma.h`的517行，`extern struct privData * dmaData;`，将dmaData申明为全局变量。

然后对dmaData进行配置。
第2957行开始的for循环`for(i=0; i<MAX_BARS; i++)`，MAX_BARS=6
对pcie设备的BAR空间进行配置(BAR0 - BAR5)
其后，第3023行，`ReadDMAEngineConfiguration(pdev, dmaData);`配置设备DMA engine的信息。
{% codeblock lang:c %}
static void ReadDMAEngineConfiguration(struct pci_dev * pdev, struct privData * dmaInfo)
{
#ifdef X86_64
	u64 base, offset;
#else
	u32 base, offset;
#endif	
	u32 val, type, dirn, num, bc;
	int i;
    int scalval=0,reg=0; 
	Dma_Engine * eptr;

	/* DMA registers are in BAR0 */
#ifdef X86_64
	base = (dmaInfo->barInfo[0].baseVAddr);
#else	
	base = (u32)(dmaInfo->barInfo[0].baseVAddr);
#endif
	/* Walk through the capability register of all DMA engines */
	for(offset = DMA_OFFSET, i=0; offset < DMA_SIZE; offset += DMA_ENGINE_PER_SIZE, i++)
	{
		log_verbose(KERN_INFO "Reading engine capability from %x\n", 
				(base+offset+REG_DMA_ENG_CAP));
		val = Dma_mReadReg((base+offset), REG_DMA_ENG_CAP);
		log_verbose(KERN_INFO "REG_DMA_ENG_CAP returned %x\n", val);
                

		if(val & DMA_ENG_PRESENT_MASK)
		{
			printk(KERN_INFO "##Engine capability is %x##\n", val);
            scalval = (val & DMA_ENG_SCAL_FACT) >> 30;  
            printk(KERN_INFO ">> DMA engine scaling factor = 0x%x \n", scalval);
            reg = XIo_In32(base+PCIE_CAP_REG);
            XIo_Out32(base+PCIE_CAP_REG,(reg | scalval ));
 
			eptr = &(dmaInfo->Dma[i]);

			log_verbose(KERN_INFO "DMA Engine present at offset %x: ", offset);

			dirn = (val & DMA_ENG_DIRECTION_MASK);
			if(dirn == DMA_ENG_C2S)
				printk("C2S, ");
			else
				printk("S2C, ");

			type = (val & DMA_ENG_TYPE_MASK); 
			if(type == DMA_ENG_BLOCK)
				printk("Block DMA, ");
			else if(type == DMA_ENG_PACKET)
				printk("Packet DMA, ");
			else
				printk("Unknown DMA %x, ", type);

			num = (val & DMA_ENG_NUMBER) >> DMA_ENG_NUMBER_SHIFT;
			printk("Eng. Number %d, ", num);

			bc = (val & DMA_ENG_BD_MAX_BC) >> DMA_ENG_BD_MAX_BC_SHIFT;
			printk("Max Byte Count 2^%d\n", bc);

			if(type != DMA_ENG_PACKET) {
				log_normal(KERN_ERR "This driver is capable of only Packet DMA\n");
				continue;
			}

			/* Initialise this engine's data structure. This will also
			 * reset the DMA engine. 
			 */
			Dma_Initialize(eptr, (base + offset), dirn);
			eptr->pdev = pdev;

			dmaInfo->engineMask |= (1LL << i);
		}
	}
	log_verbose(KERN_INFO "Engine mask is 0x%llx\n", dmaInfo->engineMask);
}
{% endcodeblock %}
`ReadDMAEngineConfiguration()`函数会读取DMA配置寄存器的信息，
{% asset_img dma_channel.png %}





