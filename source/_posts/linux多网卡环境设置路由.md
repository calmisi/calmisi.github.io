---
title: linux多网卡环境设置路由
date: 2016-11-30 15:50:25
tags:
- Route setting
categories: Linux
---

在linux系统中，如果系统配置多块网卡，则需要正确的设置路由以引导不同的流量走不同的网卡。

# 一.查看路由
`route`

# 二.默认路由设置
1. 删除默认路由
`route del default`  
此条命令会删除`route`显示的路由表的最上面的一条默认路由。
2. 增加默认路由
`route add default gw IP(如：192.168.1.1)`
<!-- more -->

# 三.网段路由设置
1. 增加网段路由
`route add –net IP netmask MASK eth0`
`route add –net IP netmask MASK gw IP`
`route add –net IP/24 eth1`
2. 删除路由
删除一条路由
`route del -net 192.168.122.0 netmask 255.255.255.0`
删除的时候不用写网关,会删除用`route`命令显示的路由表中匹配的第一条
# 四.主机路由设置 
1. 增加一条路由
`route add –host 192.168.168.110 dev eth0`
`route add –host 192.168.168.119 gw 192.168.168.1`

# 五.永久路由设置
在/etc/rc.local里添加路由设置命令：
`route add -net 192.168.3.0/24 dev eth0`
`route add -net 192.168.2.0/24 gw 192.168.3.254`