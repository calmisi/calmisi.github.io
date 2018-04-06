---
title: NVIDIA dirver is incompatible with GNOME desktop in vncserver
date: 2017-11-21 12:19:28
tags: 
- VNC
- Nvidia
categories: 
- Linux
---

之前一直是用的Centos自带的nouveau,
但是自从接了个2k屏幕后，经常待机锁屏后无法恢复，屏幕黑屏。
遂装了显卡NVIDIA的驱动。  

在 Linux 操作系统下，NVIDIA 显卡的驱动需要手动安装，而且需要手动设置，其中有几步需要注意的地方，
详见 <a href="http://blog.csdn.net/xueshengke/article/details/51965504">NVIDIA 显卡驱动安装教程</a> 。  

<!-- more -->

安装的过程需要屏蔽nouveau模块，还要重新生成initramfs (dracut --force)。  
完全安装好后，显示效果好多了，待机也不黑屏了，但是出现了新问题，远程VNC连接无法现实桌面。
{% asset_img vnc-breakdown.png  Something has gone wrong %}

---

经过网上各种查找解决方案（like this <a href="http://grokbase.com/t/centos/centos/14bjgtzygc/centos-7-nvidia-opengl-breaks-vncserver"> Centos 7 Nvidia openGL breaks vncserver </a>），仍然没有找到有效的解决方案。

网上给出的一些建议是：

- NVIDIA 显卡驱动会自带 OpenGL，与 GNOME 桌面环境会冲突，造成桌面崩溃。
- Linux 的桌面系统不止一种，除了 GNOME，还有很常用的 KDE 。可以使用它代替 GNOME（GNOME 桌面既美观又简洁，KDE 实在不能比）。

所以只能二选一了，
 
 1 要么卸载 NVIDIA 驱动，

```
$ nvidia-uninstall
```

 2 要么更换桌面系统。
```
$ yum groupinstall "KDE Plasma Workspaces"

```
 如果 Linux 系统有多个桌面环境，需要指定开机时选择哪一个
```
$ vim /etc/sysconfig/desktop
DESKTOP="KDE" 或 DESKTOP="GNOME"

$ reboot
```
