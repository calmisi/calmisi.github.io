---
title: vncserver的一些设置
date: 2016-11-08 10:18:11
tags:
- Vncserver
categories: Linux
---

实验室有台DELL的服务器，平常都开着。
服务器上部署着VNC-server,方便其他同学远程连接进行一些开发工作和实验。

刚好我的显示器有台是2k分辨率的，
用VNC-viewer登录服务器后，发现能设置的最大分辨率是1920*1200。
全屏的时候不能占满整个屏幕，这严重影响我操作时候的视觉美感。
<!-- more -->

---
# 设置分辨率
遂在网上搜了一下怎么设置分辨率。

1. 设置vnc server的分辨率。
在开启vncserver的服务时，加上`-geometry`参数。如：
{% codeblock %}vncserver -geometry 2560x1440 :2{% endcodeblock%}
即在5902号端口开启当前用户的vncserver服务，**注意2560x1440，其中是字母x,不是符号* **，

2. 查看系统中当前运行的vncserver进程有哪些。
{% codeblock %}ps -ef | grep -i vnc | grep -v grep{% endcodeblock%}
然后用`kill -9 pid`kill到自己想停掉的vnc服务。
因为有时候`vncserver -list`并不显示全自己已经开启的vncserver进程。

---
# 配置VNCserver开机自动开启一个进程。
由于机器要给组里其他同学做实验用，可能经常要开关机。为了每次开机都能自动开启VNCserver的一个端口服务，方便我自己远程连接。遂决定设置开机启动2个vncserver进程，5901给root,5902给自己的user.

vncserver的配置文件在 `/lib/systemd/system/vncserver@.service`  
将配置文件复制到`/etc/systemd/system/`目录。
{% codeblock %}
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:2.service
{% endcodeblock %}
vncserver@:2.service对应5902号端口(即启动vncserver时的命令：vncserver :2)对应的启动脚本。

修改刚刚复制的配置文件
`vim /etc/systemd/system/vncserver@:2.service`
原始文件如下：
{% asset_img 1.png %}
按照下图修改：
{% asset_img 2.png %}
** 说明：** User指明该脚本是以root运行的。不然的话，后面的步骤启动该服务会提示（code=exited,status=1/failed)的错误，就是因为权限不够。
将脚本中的两处<USER>修改为你需要运行这个端口的用户名。

然后运行`systemctl daemon-reload`
`systemctl start vncserver@:2.service`就可以启动对应的5902号端口的VNCserver服务
`systemctl stop vncserver@:2.service`是关闭5902号端口的VNCServer服务
`systemctl enable vncserver@:2.service`设置5902号端口的VNCserver服务开机自动启动

设置root的5901号端口的vncserver服务开机自动启动的话，直接把`/etc/systemd/system/vncserver@:2.servcie`复制一份命名为`vncserver@:1.servcie`,其中的1即代表5901号端口（同在命令行启动vncserver时的命令：vncserver :1）。然后将`vncserver@:1.servcie`里面2处对应<user>的地方改成`root`就可以了。

**PS:** 设置过程中，经过不断的重启测试自动启动服务是否正常。遇到如下问题。
1. `code=exited,status=1/failed`
权限问题，上面给出了解决方案。
2. `code=exited,status=2/failed`
是X11的临时文件的问题。去`/tmp/.X11-unix`删除对应的文件。几号端口有问题，就删除对应的`X几`文件。如果启动`vncserver@:2.servcie`出现`status=2`的问题，则删除`X2`文件。
3. 设置其他用户的vncserver某个端口服务自动启动后，非root用户用vncviewer连接时，会提示
{% codeblock %}
Authentication is required to set the network proxy used for downloading packages
{% endcodeblock %}
虽然直接点击`cancel`就可以了，但是有这个提示还是很烦的。所以我搜了一下解决办法：disable autostart of 'gnome-software=service'
{% codeblock %}
sed -e '$aX-GNOME-Autostart-enabled=false' -e '/X-GNOME-Autostart-enabled/d' -i.bak /etc/xdg/autostart/gnome-software-service.desktop
{% endcodeblock %}
for those of us who want to edit the file manually simply add `X-GNOME-Autostart-enabled=false` to the end of `/etc/xdg/autostart/gnome-software-service.desktop` and restart `vncserver`.
4. **Authentication of color** 
`Authentication is required to create a color managed device`
solution https://bugzilla.redhat.com/show_bug.cgi?id=1149893#c13
You can place a .rules file in /etc/polkit-1/rules.d
I'm doing in `02-allow-colord.rules`:
{% codeblock %}
polkit.addRule(function(action, subject) {
   if ((action.id == "org.freedesktop.color-manager.create-device" ||
        action.id == "org.freedesktop.color-manager.create-profile" ||
        action.id == "org.freedesktop.color-manager.delete-device" ||
        action.id == "org.freedesktop.color-manager.delete-profile" ||
        action.id == "org.freedesktop.color-manager.modify-device" ||
        action.id == "org.freedesktop.color-manager.modify-profile") &&
       subject.isInGroup("nwra")) {
      return polkit.Result.YES;
   }
});
{% endcodeblock %}
Which allows users in our group to access colord.

---
11.30.2016 add  
centos和fedora使用systemd管理配置信息。
1. 复制/lib/systemd/system/vncserver@.service到/etc/systemd/system/vncserver@.service来创建服务。  
2. 需要将/etc/systemd/system/vncserver@.service文件中的USER改成实际用户的用户名。-geometry参数可以指定桌面分辨率的大小，默认为1024x768。
3. 为不同的用户设置独立显示的VNC配置。
可以通过配置不同的显示设备号来实现，比如在配置文件名中加入设备号3和5：
`systemctl start vncserver-USER_1@:3.server`
`systemctl start vncserver-USER_2@:5.server`
其中`:3`和`:5`会被SYSTEMD自动替换成配置文件中的`%i`。
4. 限制VNC权限
如果你只用加密传输，你可以关闭非加密传输，在service配置文件里修改如下：
`ExecStart=/sbin/runuser -l user -c "/usr/bin/vncserver -localhost %i"`
这样VNC server对于非加密传输将只接受本机，不再允许远程方式。