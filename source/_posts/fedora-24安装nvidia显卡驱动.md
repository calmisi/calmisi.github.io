---
title: fedora 24安装nvidia显卡驱动
date: 2016-11-16 19:39:34
tags:
- Fedora
- Nvidia
categories: Linux
---

生命在于折腾。真想狗带。

前言：
昨天买的FPGA板到了，run了一下BIST(built-in-self-test),功能都是正常的。然后就是测试PCI-E和10G光口。
噩梦就这么开始了。

<!-- more -->
最开始是服务器的电源没有D形4pin 的母口，FPGA板自带的4Pin转6pin的转接线没法接在服务器的电源上，准备直接用服务器电源的6pin口接FPGA，结果服务器插上电就闪红灯，提示电源有问题。没办法，只好用外接的电源适配器。
然后把板子插上后，就准备进系统了。
结果，进入centos后，在等待的时候，屏幕上方闪了一排企鹅，屏幕就黑了，没有视频输入了。懵逼。  
以为是系统的原因，切到WIN10系统，然后也是进不去系统。
后来进bios,可以识别FPGA板。
结果尝试了把uefi换成legacy重装系统，还是不行。
一直折腾了到第二天，连DELL的工程师都没办法。
最后，我觉得不是没进去系统，是显卡没输出信号了。然后整了个小显示器插在了DVI口上，奇迹就这么发生了。有显示了。
原来是插上FPGA板后,显卡不支持2K DP口输出了。
没办法，只能先用小显示器，在系统里装好Nvidia的驱动在用2K显示器咯。

# 安装Nvidia驱动
那么接下来就是正文咯。fedora 24中安装Nvidia驱动，这个也是历经千辛万苦。

## 去官网下载对应的驱动
一般驱动为`NVIDIA-Linux-x86_64-xxx.xx.run`
安装驱动，只需要以root运行 `./NVIDIA-Linux-x86_64-xxx.xx.run`

下面列一下遇到的问题。
## 1.不能在图形界面下安装Nvidia的显卡驱动
{% asset_img 1.png %}
提示不能在图形界面下安装Nvidia的显卡驱动，必须退出X server才行。
解决方法1：`init 3` 进入level 3 字符界面操作。  
解决方法2：Fedora 使用的init程序是systemd,所以其进入字符界面的方法是以root运行`systemctl set-default multi-user.target`

## 2.Gcc
{% asset_img 2.png %}
提示我们安装驱动需要先安装gcc。
解决方法：`dnf install gcc`

## 3.需要内核源码
{% asset_img 3.png %}
提示没有内核源码，驱动需要和内核一起编译。
解决方法：`dnf install kernel-devel`，需要注意的是`kernel-devel`需要和系统本来的`kernel`版本是一致的。

## 4.提示`nvidia.io`模块无法加载
{% asset_img 4.png %}
该错误提示的意思是说 nvidia.ko 模块无法成功加载，是因为 nouveau 模块还在。
要禁掉 nouveau 模块，只需要在 /etc/modprobe.d/ 目录下建立一个 .conf 文件，在里面写上 blacklist nouveau 即可，这件事 Nvidia 驱动的安装程序已经帮我们做了，但是依然无法阻止 nouveau 模块的加载。
为什么呢？那是因为 Linux 启动时会先加载 initramfs 中的模块，如果不更新 initramfs 的话，单纯写 /etc/modprobe.d/ 目录下的配置文件也没有什么用。
在 Fedora 中更新 initramfs 使用这个命令`dracut --force`。

---
最后使用`init 5`或者`systemctl set-default graphical.target`命令设置让系统开机时进入图形界面，然后reboot命令重启。

# 卸载Nvidia驱动
装好驱动后本来以为2k屏可以愉快地工作了，结果发现我太天真了。2k屏能显示的最大的分辨率是1080p,而且刷新率只有23Hz。
简直不能忍。只好卸载驱动咯。

使用`./NVIDIA-Linux-x86_64-xxx.xx.run -h`运行，可以看到安装程序的帮助信息。可以发现-x选项，对文件进行压缩。
运行`./NVIDIA-Linux-x86_64-xxx.xx.run -x`后，可以发现有好多文件。
然后运行`nvidia-installer --uninstall`命令，就可以将Nvidia


# 安装过程中的一些问题总结
1. {% codeblock%}
  ERROR: You appear to be running an X server; please exit X before            
         installing.  For further details, please see the section INSTALLING   
         THE NVIDIA DRIVER in the README available on the Linux driver         
         download page at www.nvidia.com.
{% endcodeblock %}
如果init 3了，还是不行。可以用`ps ax | grep X`查看具体哪些进程在用Xserver,然后`kill -9 ID`杀掉进程。

2. yum install akmod-nvidia 