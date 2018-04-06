---
title: 搭建shadowsocks server
date: 2017-04-27 21:28:23
tags: shadowsocks
categories: Linux
---

自从搬了新实验室，虽然都是校内网，无奈网络差太多，LOL都打不了。  

今天发现了有Proxifier这个东西，它可以将基于SOCK5的代理变为操作系统全局的。
这样就可以先ssh连上服务器221（221服务器所在的机房的网络好嘛，打游戏方便）。  
在221的ssh上创建一个基于SOCK5的动态隧道。
然后用Proxifier将这个SOCK5的代理设为全局的，就可以用221服务器那边的网络玩LOL啦。  

可是实验发现基于SSH隧道的SOCK5代理，用了proxifier还是只能代理浏览器的流量，真烦心。  
没办法，只能尝试用SS的SOCK5代理了，首先用我的境外SS账号测试，发现基于SS的proxifier全局代理可以代理其他应用（非浏览器，如chrome）的流量。  
那么要用221的网打LOL,只能在221上装一个SS的server了。  
<!-- more -->
# 安装shadowsocks server
shadowsocks server为`shadowsocks-libev`[1]https://github.com/shadowsocks/shadowsocks-libev  
具体参见其`README`文档的`Installation`部分，  
221服务器为centos 7系统，yum安装的话，需要先下载对应的`yum repo`，  
``` shell
 wget https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo
```
然后复制到`/etc/yum.repos.d`,然后激活，最后 install `shadowsocks-libev` via yum
```
cp ./librehat-shadowsocks-epel-7.repo /etc/yum.repos.d/
su -c 'yum update'
su -c 'yum install shadowsocks-libev'
```

安装完成之后呢，进行配置
```
vim /etc/shadowsocks-libev/config.json

{
    "server":["[::0]","0.0.0.0"],
    "server_port":your_server_port,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"your_password",
    "timeout":600,
    "method":"aes-256-cfb"
}
```

最后启动服务
```
systemctl start shadowsocks-libev.service
```
# 用脚本安装
```
wget --no-check-certificate -O shadowsocks-libev.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-libev.sh
chmod +x shadowsocks-libev.sh
./shadowsocks-libev.sh 2>&1 | tee shadowsocks-libev.log
```
安装完成后，按脚本提示配置,提示如下
```
Congratulations, Shadowsocks-libev install completed!
Your Server IP:your_server_ip
Your Server Port:your_server_port
Your Password:your_password
Your Local IP:127.0.0.1
Your Local Port:1080
Your Encryption Method:aes-256-cfb

Welcome to visit:https://teddysun.com/357.html
Enjoy it!
```

本脚本安装完成后，会将 Shadowsocks-libev 加入开机自启动。

卸载的方法  
`./shadowsocks-libev.sh uninstall`

---
以下转自网络：
http://www.360doc.com/content/17/0427/23/42433521_649228978.shtml

1. Shadowsocks 有几种版本？区别是什么？
首先要明确一点，不管 Shadowsocks 有几种版本，都分为服务端和客户端，服务端是部署在服务器（VPS）上的，客户端是在你的电脑上使用的。
Shadowsocks 服务端大体上有 4 种版本，按照程序语言划分，分别为 Python ，libev ，Go ， Nodejs ，目前主流使用前 3 种。
Shadowsocks 客户端几乎包括了所有的终端设备，PC ，Mac ，Android ，iOS ，Linux 等。

2. Shadowsocks 的最低安装需求是多少？
个人建议最少 128MB 内存，因为在连接数比较多的情况下，还是占用不少内存的，如果内存不足，进程就会被系统 kill 掉，这时候就需要手动重启进程。当然，低于 128MB 也是可以安装的，Go 版是二进制安装，无需编译，非常简单快捷，libev 版运行过程中，占用内存较少，可以搭建在 Openwrt 的路由器上。
自己个人使用，且连接数不是特别大的情况下，64MB 内存也基本够用了。如果你要分享给朋友们一起使用，最好还是选用大内存的。

3. 为什么我安装（启动） Shadowsocks 失败？
我只能说脚本并没有在所有的 VPS 上都测试过，所以遇到问题是在所难免的。大部分情况下，请参考《Troubleshooting》一文，自行解决。据我所知，很多人都是配置文件出了问题导致的启动失败。还有部分是改错了 iptables 导致的。
在 Amazon EC2 ，百度云，青云上启动失败，连接不上怎么办？
在这类云 VPS 上搭建，需要注意，配置服务器端时，应使用内网IP；Amazon EC2 缺省不允许 inbound traffic，需要在security group里配置允许连接的端口，和开通SSH client连接类似，这个在 Amazon EC2 使用指南里有说明。同样的，青云，百度云也差不多，默认不允许入网流量，网卡绑定的是内网IP，因此需要将配置文件里的 server 值改为对应的内网 IP 后再重新启动。然后在云管理界面，允许入网端口。
我帮人设置了过之后，才发觉这些云和普通的 VPS 不一样，所以需要注意以上事项。

4. Shadowsocks 有没有控制面板？
答案是有的，有人基于 PHP + MySQL 写出来一个前端控制面板，被很多人用来发布收费或免费的 Shadowsocks 服务。Github 地址如下：
https://github.com/orvice/ss-panel
具体怎么安装和使用，别来问我，自己研究去。

5. 多用户怎么开启？
Shadowsocks 有多种服务端程序，目前据我所知只有 Python 和 Go 版是支持在配置文件里直接设置多端口的，至于 libev 版则需要使用多个配置文件并开启多个实例才行。
所谓的多用户，其实就是把不同的端口给不同的人使用，每个端口则对应不同的密码。Python 和 Go 版通过简单的修改单一配置文件，然后重启程序即可。

# Shadowsocks 一键安装脚本（四合一）：
系统支持：CentOS 6+，Debian 7+，Ubuntu 12+
内存要求：≥128M
日期　　：2017 年 01 月 21 日

## 关于本脚本：
1. 一键安装 Shadowsocks-Python， ShadowsocksR， Shadowsocks-Go， Shadowsocks-libev 版（四选一）服务端；
2. 各版本的启动脚本及配置文件名不再重合；
3. 每次运行可安装一种版本；
4. 支持以多次运行来安装多个版本，且各个版本可以共存（注意端口号需设成不同）；
5. 若已安装多个版本，则卸载时也需多次运行（每次卸载一种）；
6. Shadowsocks-Python 和 ShadowsocksR 安装后不可同时启动（因为本质上都属 Python 版）。

## 默认配置
服务器端口：自己设定（如不设定，默认为 8989）
密码：自己设定（如不设定，默认为 teddysun.com）
备注：脚本默认创建单用户配置文件，如需配置多用户，请手动修改相应的配置文件后重启即可。

## 使用方法
使用root用户登录，运行以下命令：
```bash
第一行
wget –no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
第二行
chmod +x shadowsocks-all.sh
第三行
./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log

安装完成后，脚本提示如下
Congratulations, your_shadowsocks_version install completed!
Your Server IP :your_server_ip
Your Server Port :your_server_port
Your Password :your_password
Your Encryption Method:aes-256-cfb

Welcome to visit:https://teddysun.com/486.html
Enjoy it!
```

## 卸载方法：
若已安装多个版本，则卸载时也需多次运行（每次卸载一种）

使用root用户登录，运行以下命令：
./shadowsocks-all.sh uninstall

## 启动脚本：
启动脚本后面的参数含义，从左至右依次为：启动，停止，重启，查看状态。
```Bash
Shadowsocks-Python 版：
/etc/init.d/shadowsocks-python start | stop | restart | status

ShadowsocksR 版：
/etc/init.d/shadowsocks-r start | stop | restart | status

Shadowsocks-Go 版：
/etc/init.d/shadowsocks-go start | stop | restart | status

Shadowsocks-libev 版：
/etc/init.d/shadowsocks-libev start | stop | restart | status
```
---
# Kcptun是一款加速软件
VPS云端Kcptun配置
软件:Xshell

## 安装代码:
```Bash
第一行：
wget https://raw.githubusercontent.com/kuoruan/kcptun_installer/master/kcptun.sh
第二行：
chmod +x ./kcptun.sh
第三行:
./kcptun.sh
```

## 设置流程(注意点)：
1. Kcptun 的服务端端口：请输入一个未被占用的端口，Kcptun 运行时将使用此端口。
2. 设置加速的 IP：如果你想加速的Shadowsocks 就在运行在当前服务器上，直接回车即可。
3. 设置需要加速的端口：即ss使用的服务器端口
4. 当前没有软件使用此端口, 确定加速此端口?(y/n)y
5. 设置 Kcptun 密码：如果不设置客户端就不用加 –key 这一参数(默认是 it’s a secrect )
6. 其余默认即可，可适当更改。

## 注意信息(客户端配置用)：
服务器IP:45.32.60.33
端口: 1139
加速地址: 127.0.0.1
加密方式 Crypt: none
加速模式 Mode: fast

## 后期维护：
```Bash
1. 更新：
./kcptun.sh update
2. 重新配置：
./kcptun.sh reconfig
3. 卸载：
./kcptun.sh uninstall
```

## 本地Kcptun客户端：
本地 Windows 64位系统为例，首先下载 Kcptun 的 Windows 版本。

1. 新建文件夹Kcptun
下载
https://github.com/xtaci/kcptun/releases/download/v20160906/kcptun-windows-amd64-20160906.tar.gz
并解压到文件夹下。
(已搬运)

2. 下载Kcptun客户端配置管理工具，按照配置图完成设置(然后导入刚才解压的客户端)。

**注意：**
Shadowsocks Android IOS等其他客户端

1、可以在关闭 KCP 协议的情况下，测试一下配置是不是正常，如果能正常联网，可以继续下一步，配置 KCP 协议。
2、KCP 端口，即为加速端口。
3、KCP 参数 -autoexpire 60 -key “你设的密码”
