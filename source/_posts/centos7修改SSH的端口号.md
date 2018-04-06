---
title: centos7修改SSH的端口号
date: 2016-11-07 13:18:20
tags: 
- Centos7
- Ssh
categories: Linux
---

最近刚搬了实验室。
新的实验室的网络是一个H3C ER2100的路由器。
VPN只支持IPsec，没办法设置个人vpn。
想要外网连入实验室局域网的服务器，进行远程开发的话。
只能用H3C的虚拟服务器了。
<!-- more -->

比如想要访问局域网内的一台服务器（局域网的ip:192.168.1.8）,
H3C路由器的WAN ip: xxx.xxx.xxx.xxx
可以设置虚拟服务器为：
{% codeblock %}
外部端口：22
内部端口：22
内部服务器IP:192.168.1.8
{% endcodeblock %}

这样就可以通过`ssh user@xxx.xxx.xxx.xxx`来远程登录新实验室的服务器了。
结果通过腾讯云里的主机测试，发现连不上。
后来发觉可能是因为学校限制了 1-1024 号端口的外网访问。遂尝试更改服务器的ssh端口地址进行测试。

# 服务器的系统为centos7。更改ssh服务的端口地址的步骤为：
1. 修改/etc/ssh/sshd_config
{% codeblock lang:vim %}
vim /etc/ssh/sshd_config

#Port 22   //取消注释
Port 22222 //增加此行 
{% endcodeblock %}
即开放22和22222号端口

2. 修改SELinux
使用以下命令查看当前SELinux允许的ssh端口
`semanage port -l | grep ssh`
会发现只有22号端口
所以需要添加22222号端口，运行以下命令
`semanage port -a -t ssh_port_t -p tcp 22222`
最后确认是否添加成功，再次运行
`semanage port -l | grep ssh`
成功的话，会输出
`ssh_port	tcp	22,22222`

3. 重启ssh
systemctl restart sshd.service

4. 开放22222号端口  
centos7防火墙换成了firewalld  
`firewall-cmd --permanent --add-port=22222/tcp`

---
# 再次添加一个H3C路由器的虚拟服务器
{% codeblock %}
外部端口：22222
内部端口：22222
内部服务器IP:192.168.1.8
{% endcodeblock %}

在腾讯云主机测试`ssh -p 22222 user@xxx.xxx.xxx.xxx`
指定ssh的端口用`-p 22222`，
最后登录成功


---
**插曲**
实验室另一个学长的服务器装的是Ubuntu 16的系统。给他设置远程访问的时候烦死了。
真心不想用Ubuntu系统。
搞了半天才发现防火墙是ufw。


