---
title: centos安装与配置原版texlive
date: 2018-04-25 17:22:09
tags: Texlive
categories: Linux
---

以前一直是在windows系统上编译论文，
现在想在服务器的centos系统上修改论文终稿，于是就探索了一下。

# 使用`yum`安装
尝试的第一种方法是`yum`安装
```
yum install texlive-latex
#安装texlive-latex
yum install texmaker
#安装tex文件编辑器
```
使用yum很方便，当使用texmaker编译之前论文的源文件的时候，提示好多包无法识别，
没有办法，还是觉得像windows那样，安装原版TEXLIVE
<!-- more -->

# 安装原版texlive
+ 下载原版texlive iso文件
下载最新的texlive-2017.iso

+ 挂载ISO镜像
```
mkdir /mnt/texlive
mount -t iso9660 -o ro,loop,noauto /home/xxx/Downloads/texlive.iso /mnt/texlive
```

+ 安装
首先需要安装依赖包，否则不能正确运行install-tl脚本，
```
yum install perl-Digest-MD5 perl-Tk
```
 然后运行install-tl脚本进行安装
```
./install-tl #以命令行的方式安装
./install-tl -gui  #以图像化界面安装
```

+ 安装完成后卸载镜像文件以及配置PATH
```
umount /mnt/texlive
#编辑/etc/profile,添加

export MANPATH=${MANPATH}:/usr/local/texlive/texmf-dist/doc/man
export INFOPATH=${INFOPATH}:/usr/local/texlive/texmf-dist/doc/info
export PATH=${PATH}:/usr/local/texlive/bin/x86_64-linux
```

# 安装后配置
+ 更新源配置
配置合适的CTAN源可以加快宏包更新的网速，以中科大的源为例:
```
tlmgr option repository http://mirrors.ustc.edu.cn/CTAN/systems/texlive/tlnet
#之后就可以用tlmgt命令进行网络更新，
#CTAN 上的包更新很频繁，所以即便是最新版的texlive2016，其中也有大量的宏包需要更新（可能包括tlmgr程序本身）
tlmgr update --self --all
```
 
+ tlmgr 也可以GUI运行
 ```
 tlmgr --gui --gui-lang zh_CN
 ```

+ 字体配置
XeTeX 和 LuaTeX 可以直接使用系统字体。
然而 texlive 自带的字体并不在系统的字体目录里面。
为了让系统可以使用texlive所带的字体，需要进行如下配置。

 将texlive的字体配置文件复制到系统内
 ```
 sudo cp /usr/local/texlive/texmf-var/fonts/conf/texlive-fontconfig.conf /etc/fonts/conf.d/09-texlive.conf
 #建议将 /etc/fonts/conf.d/09-texlive.conf包含type1字体的那行删掉，以避免在其它软件中显示成百上千的type1字体，即删掉
 <dir>/usr/local/texlive/2016/texmf-dist/fonts/type1</dir>
 ```

+ 刷新系统字体缓存

 ```
 sudo fc-cache -fsv
 ```

+ dummy package 安装
 texlive2016安装之后需要“告诉”系统texlive相关软件包都安装好了。
 这样在系统安装依赖于tex的软件（比如R）时就不必重新下载软件仓库中的旧版 texlive 相关软件。
 也不会造成不同版本 tex 命令的冲突。dummy package 就是解决这样的软件依赖问题的“虚包”。

 dummy package 的制作可以参考[TUG](https://www.tug.org/texlive/debian.html#vanilla)上的官方说明. 
 [这里](https://ctan.org/pkg/texlive-dummy-enterprise-linux-7)是RHEL 7系列的dummy package, 
 下载后直接安装即可：
 ```
 rpm -ivh texlive-dummy-2012a-1.el7.*
 ```

