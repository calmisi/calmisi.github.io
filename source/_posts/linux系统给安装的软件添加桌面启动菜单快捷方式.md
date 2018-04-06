---
title: linux系统给安装的软件添加桌面启动菜单快捷方式
date: 2016-11-22 15:09:09
tags: 
- Linux
- Desktop
categories: Linux
---

FPGA板开发的原因，Xilinx官方给的test example是用java awt写的GUI，但是在服务器上运行的时候有错误，而且出现错误后关掉程序重新运行就会提示`Another instance of GUI already running`。  
没办法咯，只能自己看下JAVA程序的源码咯。所以就下载了个Eclipse,解压后直接用的，我想要加入centos右上角那个Application索引里面，所以就上网搜了一下。

<!-- more -->
1. 解压下载的Eclipse package
{% codeblock %}
解压到/opt目录下
tar -zxvf eclipse-java-neon-1a-linux-gtk-x86_64.tar.gz -C /opt
{% endcodeblock %}
`-C /opt`表示解压到/opt目录

2. 使用符号链接，将eclipse执行文件链接到/usr/bin
{% codeblock %}
ln -s /opt/eclipse/eclipse /usr/bin/eclipse
{% endcodeblock %}

3. 创面一个共享的桌面启动器
{% codeblock %}
vim /usr/share/application/eclipse.desktop
{% endcodeblock %}
编辑如下内容
{% codeblock %}
[Desktop Entry]
Encoding=UTF-8
Name=Eclipse Neon
Comment=Eclipse Neon
Exec=/usr/bin/eclipse
Icon=/opt/eclipse/icon.xpm
Categories=Application;Development;Java;IDE
Version=1.0
Type=Application
Terminal=0
{% endcodeblock %}
