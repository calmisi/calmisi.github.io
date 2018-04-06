---
title: 'Linux网络编程之sockaddr，sockaddr_in,sockaddr_un结构体详解'
date: 2016-12-26 20:21:37
tags:
- Linux
- Socket
categories: Linux
---

# sockaddr
{% codeblock lang:c %}
struct sockaddr {
unsigned  short  sa_family;     /* address family, AF_xxx */
char  sa_data[14];                 /* 14 bytes of protocol address */
};
{% endcodeblock %}
<!-- more -->
sa_family是地址家族，一般都是“AF_xxx”的形式。好像通常大多用的是都是AF_INET。
sa_data是14字节协议地址。
此数据结构用做bind、connect、recvfrom、sendto等函数的参数，指明地址信息。

但一般编程中并不直接针对此数据结构操作，而是使用另一个与sockaddr等价的数据结构

# sockaddr_in
sockaddr_in（在netinet/in.h中定义）：
{% codeblock lang:c %}
struct  sockaddr_in {
short  int  sin_family;                      /* Address family */
unsigned  short  int  sin_port;       /* Port number */
struct  in_addr  sin_addr;              /* Internet address */
unsigned  char  sin_zero[8];         /* Same size as struct sockaddr */
};
struct  in_addr {
unsigned  long  s_addr;
};

typedef struct in_addr {
union {
            struct{
                        unsigned char s_b1,
                        s_b2,
                        s_b3,
                        s_b4;
                        } S_un_b;
           struct {
                        unsigned short s_w1,
                        s_w2;
                        } S_un_w;
            unsigned long S_addr;
          } S_un;
} IN_ADDR;
{% endcodeblock %}

sin_family指代协议族，在socket编程中只能是AF_INET
sin_port存储端口号（使用网络字节顺序）
sin_addr存储IP地址，使用in_addr这个数据结构
sin_zero是为了让sockaddr与sockaddr_in两个数据结构保持大小相同而保留的空字节。
s_addr按照网络字节顺序存储IP地址

sockaddr_in和sockaddr是并列的结构，指向sockaddr_in的结构体的指针也可以指向
sockadd的结构体，并代替它。也就是说，你可以使用sockaddr_in建立你所需要的信息,
在最后用进行类型转换就可以了bzero((char*)&mysock,sizeof(mysock));//初始化
mysock结构体名
mysock.sa_family=AF_INET;
mysock.sin_addr.s_addr=inet_addr("192.168.0.1");
……
等到要做转换的时候用：
（struct sockaddr*）mysock


# sockaddr_un
进程间通信的一种方式是使用UNIX套接字，人们在使用这种方式时往往用的不是网络套接字，而是一种称为本地套接字的方式。这样做可以避免为黑客留下后门。

1. 创建
使用套接字函数socket创建，不过传递的参数与网络套接字不同。域参数应该是AF_LOCAL或者AF_UNIX，而不能用PF_INET之类。本地套接字的通讯类型应该是SOCK_STREAM或SOCK_DGRAM，协议为默认协议。例如：
 int sockfd;
 sockfd = socket(PF_LOCAL, SOCK_STREAM, 0);

2. 绑定
创建了套接字后，还必须进行绑定才能使用。不同于网络套接字的绑定，本地套接字的绑定的是struct sockaddr_un结构。struct sockaddr_un结构有两个参数：sun_family、sun_path。sun_family只能是AF_LOCAL或AF_UNIX，而sun_path是本地文件的路径。通常将文件放在/tmp目录下。例如：
{% codeblock lang:c %}
 struct sockaddr_un sun;
 sun.sun_family = AF_LOCAL;
 strcpy(sun.sun_path, filepath);
 bind(sockfd, (struct sockaddr*)&sun, sizeof(sun));
{% endcodeblock %}

3. 监听
本地套接字的监听、接受连接操作与网络套接字类似。

4. 连接
连接到一个正在监听的套接字之前，同样需要填充struct sockaddr_un结构，然后调用connect函数。
连接建立成功后，我们就可以像使用网络套接字一样进行发送和接受操作了。甚至还可以将连接设置为非阻塞模式，这里就不赘述了。