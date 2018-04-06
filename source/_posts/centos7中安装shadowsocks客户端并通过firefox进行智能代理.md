---
title: centos7中安装shadowsocks客户端并通过firefox进行智能代理
date: 2017-10-31 19:16:10
tags: 
- Centos7
- Shadowsocks
---

由于把台式机搬回寝室了，只能用实验室的服务器，
服务器是Centos7的系统，有很多东西没有windows习惯。
这里讲以下如何在Centos上翻墙。

<!-- more -->
shadowsocks（简称ss），在windows的客户端的配置很多，linux上却很少。

在centos7上的教程也有不少，但是他们都没讲清楚一个概念：ss-server 和 ss-local的区别。

这里推荐auooo大神的ss<a href="http://www.auooo.com/2015/06/26/shadowsocks%EF%BC%88%E5%BD%B1%E6%A2%AD%EF%BC%89%E4%B8%8D%E5%AE%8C%E5%85%A8%E6%8C%87%E5%8D%97/">不完全指南 </a>

里面对于ss的原理介绍的非常浅显易懂。

我在这里说下结论：ss-server是搭建服务器用到的组件（就是你买了一个vps，要用它来翻墙。那么你要在这个vps上面搭建ss服务器，搭建好了你才能用你的笔记本啊，手机啊链接这个vps上网。）

ss-local就是本地客户端。就是windows下面的那个ss。centos等linux版本上面，对应的就是ss-local。

---
# 安装ss服务
现在讲讲客户端构筑步骤（服务端教程其实基本一样，都可以用同一个配置文件。只不过一个是ss-local启动，一个ss-server启动）

1. git下载源码。这里采用源码安装。

`git clone https://github.com/shadowsocks/shadowsocks-libev.git`


2. yum安装必要rpm

`yum install -y gcc automake autoconf libtool make build-essential curl curl-devel zlib-devel openssl-devel perl perl-devel cpio expat-devel gettext-devel git  asciidoc xmlto` 

3. 进入shadowsocks-libev目录，编译源码并安装

 `./configure && make; make install`

4. 编写配置文件
```bash
mkdir ~/etc/shadowsocks/
vim ~/etc/shadowsocks/sslocal.json

#然后编辑如下内容，并保存
{
	"server":"ss帐号的地址",
	"server_port":"ss帐号的端口",
	"local_address":"127.0.0.1",
	"local_port":1080,
	"password":"帐号密码",
	"timeout":300,
	"method":"aes-256-cfb",
	"fast_open":false,
	"workers":1
}
```

5.添加开机自动启动的服务
```bash
vim /etc/systemd/system/sslocal.service

编辑为如下内容，并保存
[Unit]  
Description=Shadowsocks  

[Service]  
TimeoutStartSec=0  
#the_path_to为上一步创建的sslocal.json的绝对目录
ExecStart=/usr/local/bin/ss-local -c the_path_to/sslocal.json   

[Install]  
WantedBy=multi-user.target  
```

6. 运行`systemctl start sslocal`启动服务


---
# 配置firefox浏览器的代理
1. 安装foxyproxy插件，在adds-on搜索foxyproxy并安装，
2. 设置foxyproxy，
 - 在Proxies中点击Add New Proxy,
 - 在General中输入名字，sslocal;并选一个颜色；
 - 在Proxy Details中，选择 Manual Proxy Configuration,
  	+ Server or IP Address设置为127.0.0.1； Port为1080； 并勾选SOCKS proxy, SOCKS v5
 - 然后点击ok,创建该Proxy,
 - 此时需要先将代理模式更改为sslocal,因为后面添加模式订阅可能需要翻墙，
 - 在设置里面，配置Pattern Subscriptions，
   	+ 直接点go,创建新的模式订阅 ；
   	+ name 为gfwlist, description为gfwlist, URL为： https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt；
   	+ 在proxies中添加刚刚创建的sslocal；
   	+ Refresh Rate设置为600;格式设置为AutoProxy; Obfuscation设置为Base64;
 - 可以在QuickAdd中开启快速添加，
 - 最后将模式改为 “ues proxies based on pre-defined patterns and priorities”

