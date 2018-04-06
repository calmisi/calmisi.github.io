---
title: Vivado remote Hardware Target
date: 2016-12-08 16:37:22
tags: Vivado
categories: Vivado development
---

由于FPGA开发板插在服务器上，往里面烧程序的话，就得去放服务器的房间，在服务器上的Vivado操作，那样就超级麻烦了。而且服务器配的显示器也很小，不便于开发。后来发现Vivado的Hardware manager支持remote Hardware Target.
<!-- more -->

这样就可以在自己的PC上开发，然后远程把程序烧到服务器上的FPGA板了。

1. 首先，登陆插有FPGA板的服务器，cd到Vivado/bin/目录。运行`hw_server`程序。
`hw_server`会在默认的3121号端口开启服务。

2. 在自己的PC上打开Vivado.在右侧的导航栏flow Navigator->Program and debug -> Hardware Manager ->Open Target ->Open new Targer.
点击Next，然后选择Remote server。输入服务器的ip地址，端口号为默认的3121.
{% asset_img 1.png 'Remote hardware target' %}

如果有防火墙，可以使用SSH tunnel.
可以通过自己的PC`ssh -L 3121:localhost:3121 server_ip`来建立与服务器的SSH隧道。
其中`3121:localhost:3121`,第一个3121表示远程server的端口。第二个和第三个表示本机的IP和端口。
**注意：如果提示SSH不能绑定本地的3121端口。则说明本地的vivado还运行local JTAG server。需要先关闭本地Vivado。然后建立隧道后，进行上述第1步，第2步的时候，直接选择local。**


---

# ssh tunnel的一些说明。

SSH的端口转发:本地转发Local Forward和远程转发Remote Forward

SSH 端口转发自然需要 SSH 连接，而 SSH 连接是有方向的，从 SSH Client 到 SSH Server 。
而我们所要访问的应用也是有方向的，应用连接的方向也是从应用的 Client 端连接到应用的 Server 端。比如需要我们要访问Internet上的Web站点时，Http应用的方向就是从我们自己这台主机(Client)到远处的Web Server。
如果SSH连接和应用的连接这两个连接的方向一致，那我们就说它是本地转发。
`ssh -L <local port>:<remote host>:<remote port> <SSH hostname>`
如果SSH连接和应用的连接这两个连接的方向不同，那我们就说它是远程转发。
`ssh -R <local port>:<remote host>:<remote port> <SSH hostname>`