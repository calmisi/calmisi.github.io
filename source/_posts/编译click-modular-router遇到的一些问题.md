---
title: 编译click modular router遇到的一些问题
date: 2017-03-01 15:05:29
tags:
- Click
categories:
- Linux
---

系统为Ubuntu 16.04.1 LTS
#问题1：
{% codeblock %}
Cant't find /usr/src/linux, so I can't compile the linuxmodule driver
(You may need the --with-linux=DIR option.)
{% endcodeblock%}

<!-- more -->
因为Ubuntu的头文件在{% codeblock %}/usr/src/linux-`uname -r`-generic{% endcodeblock%}里面
所以要么建立一个文件夹软连接
{% codeblock %}
ln -s /usr/src/linux-headers-`uname -r` /usr/src/linux
{% endcodeblock%}
要么
{% codeblock %}
./configure --with-linux=/usr/src/linux-headers-`uname -r`
{% endcodeblock%}

#问题2：
{% codeblock %}
Can't find Linux System.map file in /usr/src/linux.
(You may need the --with-linux=DIR and/or --with-linux-map=MAP options.)
{% endcodeblock%}

同上，
{% codeblock %}
./configure --with-linux-map=/boot/System.map-`uname -r`
{% endcodeblock%}