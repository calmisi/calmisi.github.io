---
title: 设置iptables的NAT转发，让内网的服务器通过NAT服务器访问外网
date: 2017-09-07 22:31:49
tags: 
- Iptables
- NAT
categories: Networking
---


小组有了10台冷板式服务器(node1-node10)，和一台浸没式液冷服务器(node0)。
但是网络中心只分配了一个公网ip给我们。
我们把公网IP配置给了node0的ens5, node0的ens6为内网ip: 192.168.0.100
node1-node10的IP为eth0: 192.168.0.101-192.168.0.110


为了使node1-10能够上外网，并且外网能够访问node1-10的服务，比如ssh,vnc等。
我们使用iptables进行NAT转发。

<!-- more -->
---
# iptables的介绍
在Linux系统使用iptables实现防火墙、数据转发等功能。
iptables有不同的表(table)，每个table有不同的链(chain)，每条chain有一个或多个规则(rule)。
本文利用NAT(network address translation，网络地址转换)表来实现数据包的转发。
iptables命令要用-t来指定表，如果没有指明，则使用系统缺省的表“filter”。所以使用NAT的时候，就要用“-t nat”选项了。
NAT表有三条缺省的链，它们分别是`PREROUTING`、`POSTROUTING`和`OUTPUT`。
{% asset_img pic1.jpg NAT结构%}

1. `PREROUTING`：在数据包传入时，就进到`PREROUTIING`链。该链执行的是修改数据包内的目的IP地址，即`DNAT`（变更目的IP地址）。`PREROUTING`只能进行`DNAT`。因为进行了`DNAT`，才能在路由表中做判断，决定送到本地或其它网口。
2. `POSTROUTING`：相对的，在`POSTROUTING`链后，就传出数据包，该链是整个NAT结构的最末端。执行的是修改数据包的源IP地址，即SNAT。POSTROUTING只能进行`SNAT`。
3. `OUTPUT`：定义对本地产生的数据包的目的NAT规则。

每个数据包都会依次经过三个不同的机制，首先是`PREROUTING(DNAT)`，再到路由表，最后到`POSTROUTING(SNAT)`。下面给出数据包流方向：
{% asset_img pic2.jpg 数据包流向与SNAT及DNAT的关系%}
文中的网络拓扑图所示的数据包，是从eth0入，eth1出。但是，无论从eth0到eth1，还是从eth1到eth0，均遵守上述的原理。就是说，SNAT和DNAT并没有规定只能在某一个网口(某一侧)。

顺便给出netfilter的完整结构图：
{% asset_img pic3.jpg netfilter的完整结构%}

---
# 内网10个节点访问外网
首先在node0上配置外网网卡ens5, 编辑'/etc/sysconfig/network-scripts/ifcfg-ens5'文件，
```bash
TYPE=Ethernet
HWADDR=e8:61:1f:12:51:18
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
DEVICE=ens5
BOOTPROTO="static"
ONBOOT="yes"
IPADDR="A.B.C.D"
GATEWAY="A.B.C.D_"
NETMASK="255.255.255.224"
```

然后配置node0上的内网网卡ens6, 编辑'/etc/sysconfig/network-scripts/ifcfg-ens6'文件，
```bash
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
DEVICE=ens6
HWADDR=e8:61:1f:12:51:19
IPADDR=192.168.0.100
NETMASK=255.255.255.0
ONBOOT=yes
```

注意不要配置`GATEWAY`

然后配置`iptables`
```bash
iptables -A FORWARD -s 192.168.0.0/24 -j ACCEPT
iptables -A FORWARD -d 192.168.0.0/24 -j ACCEPT
iptables -t nat -A POSTROUTING -o ens5 -s 192.168.0.0/24 -j MASQUERADE
iptables-save
```

这样node0就配置完成，然后就是配置node1-node10
```bash
route add default gw 192.168.0.100
echo GATEWAY=192.168.0.100 >> /etc/sysconfig/network-scripts/ifcfg-eth0
echo DNS1=114.114.114.114 >> /etc/sysconfig/network-scripts/ifcfg-eth0
service network restart
```

需要配DNS服务器，不然只能通过IP地址访问外网，不能通过域名访问外网。

---
# 外网访问内网节点的服务

这里需要用到DNAT。

比如需要访问node1的vnc 5901号窗口。
我们将node0的59011映射到node1的5901
运行如下命令：
```bash
#设置DNAT让外网访问59011的端口的包，都转发到内网node1的5901
iptables -t nat -A PREROUTING -d A.B.C.D -p tcp --dport 59011 -j DNAT --to-destination 192.168.0.101:5901

#因为之前配置内网访问外网的时候，已经配置过SNAT和FORWARD了，下面的步骤不需要执行

#用SNAT作源地址转换（关键），以使回应包能正确返回
iptables -t nat -A POSTROUTING -d 192.168.0.101 -p tcp --dport 5901 -j SNAT --to 192.168.0.100
# 还要打开FORWARD链的相关端口，特此增加
iptables -A FORWARD -o end6 -d 10.1.1.27 -p tcp --dport 5901 -j ACCEPT  
iptables -A FORWARD -i ens6 -s 10.1.1.27 -p tcp --sport 5901 -j ACCEPT  
```

最后`iptables-save`

---

# 如何同时开启TCP和UDP转发
当只开启TCP转发时，用上一节的方法：
```bash
#设置DNAT让外网访问59011的端口的包，都转发到内网node1的5901
iptables -t nat -A PREROUTING -d A.B.C.D -p tcp --dport 59011 -j DNAT --to-destination 192.168.0.101:5901
#用SNAT作源地址转换（关键），以使回应包能正确返回
iptables -t nat -A POSTROUTING -d 192.168.0.101 -p tcp --dport 5901 -j SNAT --to 192.168.0.100
# 还要打开FORWARD链的相关端口，特此增加
iptables -A FORWARD -o end6 -d 10.1.1.27 -p tcp --dport 5901 -j ACCEPT  
iptables -A FORWARD -i ens6 -s 10.1.1.27 -p tcp --sport 5901 -j ACCEPT  
```

若要同时开启TCP和UDP，则
```bash
#UDP 
iptables -t nat -A PREROUTING -d A.B.C.D -p udp --dport 59011 -j DNAT --to-destination 192.168.0.101:5901
iptables -t nat -A POSTROUTING -s 192.168.0.101 -p udp --dport 5901 -j SNAT --to-source 192.168.0.100:59011
iptables -A FORWARD -o end6 -d 192.168.0.101 -p tcp --dport 5901 -j ACCEPT  
iptables -A FORWARD -i ens6 -s 192.168.0.101 -p tcp --sport 5901 -j ACCEPT 
#TCP
iptables -t nat -A PREROUTING -d A.B.C.D -p tcp --dport 59011 -j DNAT --to-destination 192.168.0.101:5901
iptables -t nat -A POSTROUTING -s 192.168.0.101 -p udp --dport 5901 -j SNAT --to-source 192.168.0.100:59011
iptables -A FORWARD -o end6 -d 192.168.0.101 -p tcp --dport 5901 -j ACCEPT  
iptables -A FORWARD -i ens6 -s 192.168.0.101 -p tcp --sport 5901 -j ACCEPT 
```
对比发现，只是改了`POSTROUTING`链，将`-d`改为`-s`,然后将`-to`改为`--to-source`，其他不变。

---
可参考<a href="http://blog.csdn.net/subfate/article/details/52645197"> iptables学习笔记：端口转发命令优化</a>。
```bash
# 查看配置结果
iptables -t nat -L  
```
