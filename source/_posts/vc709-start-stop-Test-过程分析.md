---
title: vc709 start/stop Test 过程分析
date: 2017-03-24 11:11:39
tags:
- VC709
categories: vc709 DMA 驱动
---

本文分析VC709板卡自带的TRD程序中对应JAVA GUI界面的start和stop按钮对应的startTest和stopTest功能。
当选择performance mode的时候，GUI进入的界面可以设置四个DMA engine的包大小和选择对应的模式:
performance: loopback,pktchk,pktgen;
performance+raw_ethernet: loopback;
然后点击start,则App程序开始发包，点stop则程序App程序停止发包。

下面具体分析点击start和stop时，GUI程序给xdma驱动发送了什么命令，对FPGA进行了怎么样的配置。
<!-- more -->

**首先明确一点，startTest只用对TX engine进行配置，对应的RX Engine是不用配置的。**

---
# 软件准备testCmd
当软件设置好testmode后，调用`StartTest(int statsfd, int engine, int testmode, int maxSize)`;
第一个参数是xdma_stat的文件描述符，第二个参数为TX engine的编号(0,1,2,3),第三参数为上面设好的tmode,第四个参数为最大的包大小；

* 其中第三个参数testmode可取的值为下面宏的组合：
{% codeblock lang:c %}
#define ENABLE_PKTCHK       0x00000100  /**< Enable TX-side packet checker */
#define ENABLE_PKTGEN       0x00000400  /**< Enable RX-side packet generator */
#define ENABLE_LOOPBACK     0x00000200  /**< Enable loopback mode in test */
#define ENABLE_CRISCROSS    0x00002000  /**< Enable loopback mode in CRISCROSS test */

#define TEST_STOP           0x00000000  /**< Stop the test */
#define TEST_START          0x00008000  /**< Start the test */
#define TEST_IN_PROGRESS    0x00004000  /**< Test is in progress */
{% endcodeblock %}
`int testmode`可取下面的值：
1. ENABLE_LOOPBACK 	只有loopback
2. ENABLE_PKTCHK	只是硬件check
3. ENABLE_PKTGEN	只是硬件generate
4. ENABLE_PKTCHK|ENABLE_PKTGEN	硬件既check又generate

在StartTest函数中，根据第2,3,4 号参数初始化testCmd(TestCmd类型)，
testCmd.Engine = engine;
testCmd.TestMode = testmode;
testCmd.MaxPktSize = maxSize; 

**并且最后testCmd.TestMode |= TEST_START，表示是start test；**
如果是Raw_ETH的performance，还要testCmd.TestMode |= ENABLE_CRISCROSS;
最后将testCmd传给xdma的ioctl();

** 总结 **
{% codeblock lang:c %}
testCmd.TestMode
		loopback		check			generate		check|generate
init:		0x0000 0200		0x0000 0100		0x0000 0400		0x0000 0500	
Not Raw_ETH:	0x0000 8200		0x0000 8100		0x0000 8400		0x0000 8500
Raw_ETH:	0x0000 A200		0x0000 A100 		0x0000 A400		0x0000 A500

//即32位的寄存
0000 0000 0000 0000 0000 0000 0000 0000 (从右往左第0位开始)
                              
//第8位:check位
//第9位:loopback位
//第10位:gen位
//第14位：inprocess位
//第15位：startTest位：1表示start,0 表示stop
//第13位：criscross位，即raw_eth performance位
{% endcodeblock %}

---
# xdma驱动的ioctl()接收到命令
然后看到xdma驱动的`ioctl()`函数:
在`ISTART_TEST`和`ISTOP_TEST`的case分支时，
用传入的testCmd赋值ustate,然后调用`(uptr->UserSetState)(eptr, &ustate, uptr->privData)`;
这个函数才是真正的置寄存器函数。

---
# sgusr.c中的mySetState

我们看来sgusr.c中的mySetState
其中要判断engien是否为TX engine ==> if (privdata == 0x54545454);(说明了start test只用设置TX engine)
然后RawTestMode = ustate->TestMode；

然后根据RawTestMOde来初始化testmode = 0;
{% codeblock lang:c %}
if (RawTestMode & TEST_START)	//见下面的解释，给testmode置值
{
	testmode = 0;
	if (RawTestMode & ENABLE_LOOPBACK)
		testmode |= LOOPBACK;
	if (RawTestMode & ENABLE_PKTCHK)
		testmode |= PKTCHKR;
	if (RawTestMode & ENABLE_PKTGEN)
		testmode |= PKTGENR;
}
else //TEST_STOP时，将之前in process的testmode的对应位置0，如果是loopback模式，则第1位置0；如果是CHK或者GEN，则第0位置0
{
	if (RawTestMode & ENABLE_PKTCHK)
		testmode &= ~PKTCHKR;
	if (RawTestMode & ENABLE_PKTGEN)
		testmode &= ~PKTGENR;

	if (RawTestMode & ENABLE_LOOPBACK)
		testmode &= ~LOOPBACK;

}

/* Test start / stop conditions */
#define PKTCHKR             0x00000001	/* Enable TX packet checker */
#define PKTGENR             0x00000001	/* Enable RX packet generator */
#define CHKR_MISMATCH       0x00000001	/* TX checker reported data mismatch */
#define LOOPBACK            0x00000002	/* Enable TX data loopback onto RX */

所以
				loopback		check			generate		check|generate
TEST_START	testmode	0x0000 0002		0x0000 0001		0x0000 0001		0x0000 0001 	
{% endcodeblock %}

---
** Important **
然后开始写寄存器,即配置FPGA以改变功能
sgusr.c文件中，有一个条件编译`#ifdef RAW_ETH`,根据没有定义`RAW_ETH`会执行不同的代码，从而设置不同的寄存器值，
{% codeblock lang:c %}
//TEST_START时：
	//not Raw_ETH:	ndef RAW_ETH
		XIo_Out32 	(TXbarbase + DESIGN_MODE_ADDRESS,PERF_DESIGN_MODE);
				(TXbarbase + 0x9004, 0x0000 0003)
		XIo_Out32 	(TXbarbase + PKT_SIZE_ADDRESS, val);
				(TXbarbase + 0x9104/0x9204/0x9304/0x9404, maxpaketSize)
		XIo_Out32 	(TXbarbase + SEQNO_WRAP_REG , seqno);
				(TXbarbase + 0x9110/0x9210/0x9310/0x9410, 512)
						
		//然后重置CONFIG_ADDRESS
		XIo_Out32 	(TXbarbase + TX_CONFIG_ADDRESS, 0);
				(TXbarbase + 0x9108/0x9208/0x9308/0x9408, 0)
		//如果是PKTCHK|LOOPBACK:
		if (RawTestMode & (ENABLE_PKTCHK | ENABLE_LOOPBACK))
			XIo_Out32	(TXbarbase + TX_CONFIG_ADDRESS, testmode);
					(TXbarbase + 0x9108/0x9208/0x9308/0x9408, 0x00000001|0x00000002)
		if (RawTestMode & ENABLE_PKTGEN)
			XIo_Out32	(TXbarbase + RX_CONFIG_ADDRESS, testmode);
					(TXbarbase + 0x9100/0x9200/0x9300/0x9400, 0x00000001)	
			
	//Raw_ETH:	
		XIo_Out32 	(TXbarbase + PKT_SIZE_ADDRESS, val);
				(TXbarbase + 0x9104/0x9204/0x9304/0x9404, maxpaketSize)
						
		//然后重置CONFIG_ADDRESS
		XIo_Out32 	(TXbarbase + TX_CONFIG_ADDRESS, 0);
				(TXbarbase + 0x9108/0x9208/0x9308/0x9408, 0)
		//如果是PKTCHK|LOOPBACK:
		if (RawTestMode & (ENABLE_PKTCHK | ENABLE_LOOPBACK))
			XIo_Out32 	(TXbarbase + TX_CONFIG_ADDRESS, testmode);
					(TXbarbase + 0x9108/0x9208/0x9308/0x9408, 0x00000001|0x00000002)
			XIo_Out32 	(TXbarbase + DESIGN_MODE_ADDRESS,PERF_DESIGN_MODE);
					(TXbarbase + 0x9004, 0x0000 0000)注意这个非Raw_ETH的值不同
			if(RawTestMode & ENABLE_CRISCROSS):
				XIo_Out32(TXbarbase+NW_PATH_OFFSET_OTHER+XXGE_RCW1_OFFSET, 0x50000000);
			else:
				XIo_Out32(TXbarbase+NW_PATH_OFFSET+XXGE_RCW1_OFFSET, 0x50000000);
			XIo_Out32(TXbarbase+NW_PATH_OFFSET+XXGE_TC_OFFSET, 0x50000000);
		//如果是PKEGEN:
		if (RawTestMode & ENABLE_PKTGEN)
			XIo_Out32 	(TXbarbase + RX_CONFIG_ADDRESS, testmode);
					(TXbarbase + 0x9100/0x9200/0x9300/0x9400, 0x00000001)

//TEST_STOP时：
	XIo_Out32 (TXbarbase + TX_CONFIG_ADDRESS, testmode);
	XIo_Out32 (TXbarbase + RX_CONFIG_ADDRESS, testmode);
{% endcodeblock %}