---
title: numactl of centos
date: 2017-09-06 16:30:34
tags: 编译
categories: Linux
---

今天编译最新的DPDK，出现了没有<numa.h> <numaif.h>等错误，
最后上网查，和查看最新的DPDK文档，发现是需要libnuma-devel的库，可是用
`yum install libnuma-devel`发现没有这个库。

<!-- more -->
google了一下，发现centos上是`numactl-devel`这个库，

下面有个一个<a href="http://fibrevillage.com/sysadmin/534-numactl-installation-and-examples"> 文章</a>是介绍如何使用`numactl`这个工具的。