---
title: linux设备驱动，内核相关知识点
date: 2016-12-27 14:29:43
tags:
- Linux
- Device driver
categories: Linux
---

本文记录一些在开发Linux 设备驱动过程中，遇到的函数和知识点。
仅供以后回顾。
<!-- more -->


# 函数
1. `void *kmalloc(size_t size, int flags)`
kmalloc计算机语言的一种函数名，分配内存。
语法，void *kmalloc(size_t size, int flags);
size要分配内存的大小. 以字节为单位;flags要分配内存的类型。
在设备驱动程序或者内核模块中动态开辟内存，不是用malloc，而是kmalloc,vmalloc,或者用get_free_pages直接申请页。释放内存用的是kfree,vfree，或free_pages。
kmalloc函数返回的是虚拟地址(线性地址)。kmalloc特殊之处在于它分配的内存是物理上连续的,这对于要进行DMA的设备十分重要. 而用vmalloc分配的内存只是线性地址连续,物理地址不一定连续,不能直接用于DMA。
kmalloc最大只能开辟128k-16，16个字节是被页描述符结构占用了。
内存映射的I/O口，寄存器或者是硬件设备的RAM(如显存)一般占用F0000000以上的地址空间。
在驱动程序中不能直接访问，要通过kernel函数vremap获得重新映射以后的地址。
另外，很多硬件需要一块比较大的连续内存用作DMA传送。这块内存需要一直驻留在内存，不能被交换到文件中去。
但是kmalloc最多只能开辟大小为32 X PAGE_SIZE的内存,一般的PAGE_SIZE=4kB,也就是128kB的大小的内存。

2. 
---
# 结构
1. pthread_mutex_t
linux下为了多线程同步，通常用到锁的概念。
posix下抽象了一个锁类型的结构：ptread_mutex_t。
通过对该结构的操作，来判断资源是否可以访问。顾名思义，加锁(lock)后，别人就无法打开，只有当锁没有关闭(unlock)的时候才能访问资源。
它主要用如下5个函数进行操作。
1：pthread_mutex_init(pthread_mutex_t * mutex,const pthread_mutexattr_t *attr);
初始化锁变量mutex。attr为锁属性，NULL值为默认属性。
2：pthread_mutex_lock(pthread_mutex_t *mutex);加锁
3：pthread_mutex_tylock(pthread_mutex_t *mutex);加锁，但是与2不一样的是当锁已经在使用的时候，返回为EBUSY，而不是挂起等待。
4：pthread_mutex_unlock(pthread_mutex_t *mutex);释放锁
5：pthread_mutex_destroy(pthread_mutex_t *mutex);使用完后释放

---
# 命令
1. lsof
lsof（list open files）是一个列出当前系统打开文件的工具。
在linux环境下，任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。
所以如传输控制协议 (TCP) 和用户数据报协议 (UDP) 套接字等，系统在后台都为该应用程序分配了一个文件描述符，无论这个文件的本质如何，该文件描述符为应用程序与基础操作系统之间的交互提供了通用接口。
因为应用程序打开文件的描述符列表提供了大量关于这个应用程序本身的信息，因此通过lsof工具能够查看这个列表对系统监测以及排错将是很有帮助的。
{% codeblock %}
lsof [参数][文件]
-a 列出打开文件存在的进程
-c<进程名> 列出指定进程所打开的文件
-g  列出GID号进程详情
-d<文件号> 列出占用该文件号的进程
+d<目录>  列出目录下被打开的文件
+D<目录>  递归列出目录下被打开的文件
-n<目录>  列出使用NFS的文件
-i<条件>  列出符合条件的进程。（4、6、协议、:端口、 @ip ）
-p<进程号> 列出指定进程号所打开的文件
-u  列出UID号进程详情
-h 显示帮助信息
-v 显示版本信息
{% endcodeblock %}

2. 内存页，PAGE_MASK判定addr是否是4096(4K)倍数
{% codeblock lang:c %}
#define PAGE_SHIFT 12
#define PAGE_SIZE   (1UL << PAGE_SHIFT)
#define PAGE_MASK   (~(PAGE_SIZE-1))
{% endcodeblock%}
PAGE_SIZE = 1 0000 0000 0000        = 2^12              = 4K
PAGE_MASK = ~(1 0000 0000 0000 - 1) = ~(1111 1111 1111) = 0000 0000 0000
PAGE_ALIGN(addr)函数将地址addr调整为页对齐： 
{% codeblock lang:c %}
#define PAGE_ALIGN(addr)    (((addr)+PAGE_SIZE-1)&PAGE_MASK)
{% endcodeblock%}
如addr为0x22000001 
PAGE_ALIGN(addr)=(0x22000001+4096-1)&0xfffff000
=(0x22000001+0xfff)&0xfffff000=0x22001000&0xfffff000
=0x22001000;
同样，比如addr为0x22000003，PAGE_ALIGN(addr)后仍然是0x22001000；
{% asset_img page_align.png 内存页4k对齐 %}

---
# _IO, _IOR, _IOW, _IOWR 宏的解析与用法 
写设备驱动的时候，需要用ioctl()函数向设备传递控制信息。
{% codeblock lang:c %}
/* Perform the I/O control operation specified by REQUEST on FD.
   One argument may follow; its presence and type depend on REQUEST.
   Return value depends on REQUEST.  Usually -1 indicates error.  */
extern int ioctl (int __fd, unsigned long int __request, ...) __THROW;
{% endcodeblock %}
其中`long int __request`是应用程序用于区别设备驱动程序请求处理内容的值。其大小4 Byte,32位。共分为4个域：
bit 31~30(2位): 	dir
bit 29~16(14位):	size
bit 15~08(8位):	type
bit 07~00(8位):	nr
内核定义了 _IO() , _IOR() , _IOW() 和 _IOWR() 这 4 个宏来辅助生成上面的`__request`。
下面分析 _IO() 的实现，其它的类似。
在`asm-generic/ioctl.h`里可以看到这四个宏的定义：
{% codeblock lang:c %}
#define _IO(type,nr)		_IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)	_IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)	_IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size)	_IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
{% endcodeblock %}
再看 _IOC() 的定义：
{% codeblock lang:c %}
#define _IOC(dir,type,nr,size) \
	(((dir)  << _IOC_DIRSHIFT) | \
	 ((type) << _IOC_TYPESHIFT) | \
	 ((nr)   << _IOC_NRSHIFT) | \
	 ((size) << _IOC_SIZESHIFT))
#define _IOC_TYPECHECK(t) (sizeof(t))
{% endcodeblock %}
可见， _IO()最后的结果由 _IOC()中的4个参数移位组合而成。
在具体的看移位参数值，可以发现_IOC_TYPESHIFT = 8 ; _IOC_SIZESHIFT = 16 ; _IOC_DIRSHIFT = 30
所以，
	(dir)  << 30) | \	左移30位，得到方向（读写）属性
	(type) << 8) | \	左移8位，得到魔区（magic）
	(nr)   << 0) | \	左移0位，基（序列号）数
	(size) << 16		左移16位，得到“数据大小”区

这几个宏的使用格式为：
_IO (魔数， 基数);
_IOR (魔数， 基数， 变量型)
_IOW  (魔数， 基数， 变量型)
_IOWR (魔数， 基数，变量型 )

1. 魔数(type) 
魔数范围为 0~255 。
通常，用英文字符 "A" ~ "Z" 或者 "a" ~ "z" 来表示。
设备驱动程序从传递进来的命令获取魔数，然后与自身处理的魔数相比较，如果相同则处理，不同则不处理。
魔数是拒绝误使用的初步辅助状态。设备驱动 程序可以通过 _IOC_TYPE (cmd) 来获取魔数。
不同的设备驱动程序最好设置不同的魔数，但并不是要求绝对，也是可以使用其他设备驱动程序已用过的魔数。
设备驱动程序通过`_IOC_TYPE(__request)`获取魔数

2. 基数，序列号(nr) 
用于区别发向同一个设备的各种不同的命令。
通常，从 0开始递增，相同设备驱动程序上可以重复使用该值。
例如，读取和写入命令中使用了相同的基数，设备驱动程序也能分辨出来，原因在于设备驱动程序区分命令时，使用 switch ，且直接使用命令变量__request值,即dir+nr来区别一个命令。
设备驱动程序通过`_IOC_NR(__request)`获取nr

3. 变量型(size) 设备驱动程序通过`_IOC_SIZE(__request)`获取
指定ioctl()函数的第三个参数的数据大小。
{% codeblock lang:c %}
/* Perform the I/O control operation specified by REQUEST on FD.
   One argument may follow; its presence and type depend on REQUEST.
   Return value depends on REQUEST.  Usually -1 indicates error.  */
extern int ioctl (int __fd, unsigned long int __request, ...) __THROW;
{% endcodeblock %}
比如第三个参数为args,即函数调用为`ioctl(fd,request,args)`
args为ARG类型,则size对应的为：ARG。
这里直接输入的是变量或者变量的类型，是因为使用了`_IOC_TYPECHECK`宏，
{% codeblock lang:c %}#define _IOC_TYPECHECK(t) (sizeof(t)){% endcodeblock %}
设备驱动程序通过`_IOC_SIZE(__request)`获取size

## _IO 宏 
该宏函数没有可传送的变量，只是用于传送命令。例如如下约定：
`#define TEST_DRV_RESET _IO ('Q', 0)`
此时，省略由应用程序传送的 arg 变量或者代入 0 。在应用程序中使用该宏时，比如：
ioctl (dev, TEST_DEV_RESET, 0)   或者  ioctl (dev, TEST_DRV_RESET) 。 
这是因为变量的有效因素是可变因素。只作为命令使用时，没有必要判 断出设备上数据的输出或输入。因此，设备驱动程序没有必要执行设备文件大开选项的相关处理。 
## _IOR 宏 
该函数用 于创建从设备读取数据的命令，例如可如下约定：
`#define TEST_DEV_READ  _IRQ('Q', 1, int)`
这说明应用程序从设备读取数据的大小为 int 。下面宏用于判断传送到设备驱动程序的`__request`命令的读写状态：
`_IOC_DIR (cmd) `
运行该宏时，返回值的类型 如下： 
_IOC_NONE                             	:  无属性
_IOC_READ                             	:  可读属性
_IOC_WRITE                           	: 可写属性
_IOC_READ | _IOC_WRITE 					: 可读，可写属性
使用该命令时，应用程序的 ioctl() 的 arg 变量值指定设备驱动程序上读取数据时的缓存(结构体)地址。
## _IOW 宏 
用于创建设 备上写入数据的命令，其余内容与 _IOR 相同。通常，使用该命令时，ioctl() 的 arg 变量值指定设备驱动程序上写入数据时的缓存(结构体)地址。
## _IOWR 宏 
用于创建设备上读写数据的命令。其余内 容与 _IOR 相同。通常，使用该命令时，ioctl() 的 arg 变量值指定设备驱动程序上写入或读取数据时的缓存 (结构体) 地址。

---