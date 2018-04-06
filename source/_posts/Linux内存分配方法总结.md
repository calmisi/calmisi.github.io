---
title: Linux内存分配方法总结
date: 2017-03-24 19:24:11
tags: 
- Linux
- DMA
categories: Linux
---

本文转自： http://www.cnblogs.com/wenhuisun/archive/2013/05/15/3079722.html
<!-- more -->

# 介绍
## 内存映射结构：
1. 32位地址线寻址4G的内存空间，其中0-3G为用户程序所独有，3G-4G为内核占有。
2. struct page：整个物理内存在初始化时，每个4kb页面生成一个对应的struct page结构，这个page结构就独一无二的代表这个物理内存页面，并存放在mem_map全局数组中。
3. 段式映射：首先根据代码段选择子cs为索引，以GDT值为起始地址的段描述表中选择出对应的段描述符，随后根据段描述符的基址，本段长度，权限信息等进行校验，校验成功后。cs:offset中的32位偏移量直接与本段基址相累加，得出最终访问地址。

## 0-3G与mem_map的映射方式：
因linux中采用的段式映射为flat模式，所以从逻辑地址到线性地址没有变化。从段式出来进入页式，每个用户进程都独自拥有一个页目录表（pdt），运行时存放于CR3。 
CR3（页目录） + 前10位 =>  页面表基址 + 中10位 => 页表项 + 后12位 => 物理页面地址

## 3G-4G与mem_map的映射方式：
分为三种类型：低端内存/普通内存/高端内存。
低端内存：3G-3G+16M 用于DMA        __pa线性映射
普通内存：3G+16M-3G+896M          __pa线性映射 （若物理内存<896M，则分界点就在3G+实际内存）
高端内存：3G+896-4G               采用动态的分配方式

## 高端内存(假设3G+896为高端内存起址)
作用：访问到1G以外的物理内存空间。
线性地址共分为三段：vmalloc段/kmap段/kmap_atomic段（针对与不同的内存分配方式）

---
# 从内存分配函数的结构来看主要分为下面几个部分:
a. 伙伴算法(最原始的面向页的分配方式)
alloc_pages 接口：
    struct page * alloc_page(unsigned int gfp_mask)——分配一页物理内存并返回该页物理内存的page结构指针。
    struct page * alloc_pages(unsigned int gfp_mask, unsigned int order)——分配 个连续的物理页并返回分配的第一个物理页的page结构指针。
    <释放函数：__free_page(s)>
    
    内核中定义：#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)   
    最终都是调用 __alloc_pages.
    其中MAX_ORDER 11，及最大分配到到页面个数为2^10（即4M）。
    分配页后还不能直接用，需要得到该页对应的虚拟地址：
    void *page_address(struct page *page);
    低端内存的映射方式：__va((unsigned long)(page  -  mem_map)  <<  12)
    高端内存到映射方式：struct page_address_map分配一个动态结构来管理高端内存。(内核是访问不到vma的3G以下的虚拟地址的) 具体映射由kmap / kmap_atomic执行。
    
get_free_page接口：(alloc_pages接口两步的替代函数)
    unsigned long get_free_page(unsigned int gfp_mask) 
    unsigned long __get_free_page(unsigned int gfp_mask) 
    Unsigned long __get_free_pages(unsigned int gfp_mask, unsigned int order)
    <释放函数：free_page>
    与alloc_page(s)系列最大的区别是无法申请高端内存，因为它返回到是一个线性地址，而高端内存是需要额外映射才可以访问的。

b. slab高速缓存（反复分配很多同一大小内存）   注：使用较少
    kmem_cache_t* xx_cache;
    创建： xx_cache = kmem_cache_create("name", sizeof(struct xx), SLAB_HWCACHE_ALIGN, NULL, NULL);
    分配： kmem_cache_alloc(xx_cache, GFP_KERNEL);
    释放： kmem_cache_free(xx_cache, addr);
  内存池
      mempool 不使用。
  
c. kmalloc（最常用的分配接口）         注：必须小于128KB
    GFP_ATOMIC 不休眠，用于中断处理等情况
    GFP_KERNEL 会休眠，一般状况使用此标记
    GFP_USER   会休眠
    __GFP_DMA  分配DMA内存
    kmalloc/kfree
    
d. vmalloc/vfree
    vmalloc采用高端内存预留的虚拟空间来收集内存碎片引起的不连续的物理内存页，是用于非连续物理内存分配。
当kmalloc分配不到内存且无物理内存连续的需求时，可以使用。（优先从高端内存中查找）
    
e. ioremap()/iounmap()
　　ioremap()的作用是把device寄存器和内存的物理地址区域映射到内核虚拟区域，返回值为内核的虚拟地址。使用的线性地址区间也在vmmlloc段
注：
vmalloc()与 alloc_pages(_GFP_HIGHMEM)+kmap()；前者不连续，后者只能映射一个高端内存页面
__get_free_pages与alloc_pages(NORMAL)+page_address()； 两者完全等同
内核地址通过 __va/__pa进行中低内存的直接映射
高端内存采用kmap/kmap_atomic的方式来映射

---    
# 个人总结如下：
a.在<128kB的一般内存分配时，使用kmalloc
    允许睡眠：GFP_KERNEL
    不允许睡眠：GFP_ATOMIC
b.在>128kB的内存分配时，使用get_free_pages，获取成片页面，直接返回虚拟地址（<4M）（或alloc_pages + page_address）
c.b失败，
    如果要求分配高端内存：alloc_pages(_GFP_HIGHMEM)+kmap（仅能映射一个页面）
    如果不要求内存连续： 则使用vmalloc进行分配逻辑连续的大块页面.(不建议)/分配速度较慢，访问速率较慢。
d.频繁创建和销毁很多较大数据结构,使用slab.
e.高端内存映射：
    允许睡眠：kmap              (永久映射)
    不允许睡眠：kmap_atomic      (临时映射)会覆盖以前到映射（不建议）