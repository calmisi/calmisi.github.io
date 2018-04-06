---
title: 使用git时的一些技巧、问题和解决方案
categories: Git
date: 2016-10-29 20:34:03
tags: Git
---

本文描述一些使用git时，需要用到的一些技巧。以及可能遇到的问题和相应的解决方案。
<!-- more -->
# 可能遇到的问题
## 1.warning: LF will be replaced by CRLF 
 
* CRLF: Carriage-Return Line-Feed回车换行  
就是回车(CR, ASCII 13, \r) 换行(LF, ASCII 10, \n)。  
这两个ACSII字符不会在屏幕有任何输出，但在Windows中广泛使用来标识一行的结束。而在Linux/UNIX系统中只有换行符。
也就是说在windows中的换行符为 CRLF，而在linux下的换行符为：LF  
当原始文件为Linux系统中生成的文件时，其中为LF，当在windows系统执行`git`指令时，系统提示：LF 将被转换成 CRLF。  

**解决方法：**  
配置git参数`core.autocrlf`为false
```
$ git config --global core.autocrlf false
```

# git中的一些技巧
## 1.使用分支  
在Git中，分支是你日常开发流程中的一部分。当你想要添加一个新的功能或是修复一个bug时——不管bug是大是小——你都应该新建一个分支来封装你的修改。这确保了不稳定的代码永远不会被提交到主代码库中，它同时给了你机会，在并入主分支前清理你feature分支的历史。
