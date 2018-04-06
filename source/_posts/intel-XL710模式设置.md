---
title: intel XL710模式设置
date: 2017-02-16 19:12:12
tags: Intel XL710
categories: Networking
---

买了块Intel 40G的网卡XL710QDA2.
由于实验室的交换机是pica8的，只有4个10G的光口，为了让XL710网卡能和交换机互连，就需要设置其工作模式。

设置其工作模式需要用到Intel的QCU工具（QSFP+ Configuration Utility），
官方有个说明文档：http://www.intel.com/content/dam/www/public/us/en/documents/guides/qsfp-configuration-utility-quick-usage-guide.pdf

<!-- more -->
对应的QCU软件的下载地址为：https://downloadcenter.intel.com/zh-cn/download/25851/-QSFP-Linux-

在应用QCU软件的之前，需要先更新网卡的NVM，因为NVM版本在4.42之前的不支持。这个可以用`ethtool -i ethX`命令查看，最好去官网下载最新的来更新网卡。我现在用的是5.05版本。

下载好QCU工具后，如linux版本解压后为`qcu64e`,`chmod a+x ./qcu64e`,给执行权限。
{% codeblock lang:shell%}
./qcu64e /DEVICES 列出当前系统识别的NIC的号码，第一列NIC下面的数字后面要用到
./qcu64e /NIC=1 /INFO 显示NIC号为1的这块网卡的信息，
./qcu64e /NIC=1 /SET 2x2x10A 设置网卡的2个40G的QSPF+的网卡分别工作在2个10GSPF+模式
{% endcodeblock%}