---
title: eclipse中相关issue的solutions
date: 2016-11-28 14:51:35
tags:
- Java
- Eclipse
categories: Java & eclipse
---

本文记录在使用eclipse进行java开发时运到的问题和相应的解决方法。

<!-- more -->

1. 错误`Access restriction:The type *** is not accessible due to restriction on... `解决方案
当报此类error的时候，需要重新添加libraries，因为在多个不同的jar里面有classes,remove了之后重新add JRE lib会让正确的classes成为first.
**Solution:  
Project->Properties->Java buildpath->libraries,
先remove掉JRE System Library,然后再Add Library重新加入系统默认的JRE.**  
Access restriction的原因是因为这些JAR默认包含了一系列的代码访问规则（Access Rules），如果代码中引用了这些访问规则所禁止引用类，那么就会提示这个错误信息。
一、既然存在访问规则，那么修改访问规则即可。打开项目的Build Path Configuration页面，打开报错的JAR包，选中Access rules条目，选择右侧的编辑按钮，添加一个访问规则即可。
二、网上的另外一种解决方案：Window - preference - Java - Compiler - Errors/Warnings界面的Deprecated and restricted API下。把Forbidden reference (access rules): 的规则由默认的Error改为Warning。
这种方案是修改整个Eclipse开发环境，将所有禁止访问的引用由原来的Error（默认）修改为Warning。这种规避方式比较粗暴，个人支持第一种方案