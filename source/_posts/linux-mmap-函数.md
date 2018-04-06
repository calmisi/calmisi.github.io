---
title: linux mmap()函数
date: 2017-04-20 14:47:35
tags: Linux
categories: 系统函数
---

头文件：#include <unistd.h>    #include <sys/mman.h>

定义函数：`void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offsize);`

参数说明：<br />
<table>
<tbody>
<tr>
<th>
参数</th>
<th>
说明</th>
</tr>
<tr>
<td>
start</td>
<td>
指向欲对应的内存起始地址，通常设为NULL，代表让系统自动选定地址，对应成功后该地址会返回。</td>
</tr>
<tr>
<td>
length</td>
<td>
代表将文件中多大的部分对应到内存。</td>
</tr>
<tr>
<td>
prot</td>
<td>
&nbsp;代表映射区域的保护方式，有下列组合：
<ul>
<li>
PROT_EXEC &nbsp;映射区域可被执行；</li>
<li>
PROT_READ &nbsp;映射区域可被读取；</li>
<li>
PROT_WRITE &nbsp;映射区域可被写入；</li>
<li>
PROT_NONE &nbsp;映射区域不能存取。</li>
</ul>
</td>
</tr>
<tr>
<td>
flags</td>
<td>
会影响映射区域的各种特性：
<ul>
<li>
MAP_FIXED &nbsp;如果参数 start 所指的地址无法成功建立映射时，则放弃映射，不对地址做修正。通常不鼓励用此旗标。</li>
<li>
MAP_SHARED &nbsp;对应射区域的写入数据会复制回文件内，而且允许其他映射该文件的进程共享。</li>
<li>
MAP_PRIVATE &nbsp;对应射区域的写入操作会产生一个映射文件的复制，即私人的&quot;写入时复制&quot; (copy&nbsp;on write)对此区域作的任何修改都不会写回原来的文件内容。</li>
<li>
MAP_ANONYMOUS &nbsp;建立匿名映射，此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享。</li>
<li>
MAP_DENYWRITE &nbsp;只允许对应射区域的写入操作，其他对文件直接写入的操作将会被拒绝。</li>
<li>
MAP_LOCKED &nbsp;将映射区域锁定住，这表示该区域不会被置换(swap)。</li>
</ul>
<br />
<span style="color:#b22222;">在调用mmap()时必须要指定MAP_SHARED 或MAP_PRIVATE。</span></td>
</tr>
<tr>
<td>
fd</td>
<td>
open()返回的文件描述词，代表欲映射到内存的文件。</td>
</tr>
<tr>
<td>
offset</td>
<td>
文件映射的偏移量，通常设置为0，代表从文件最前方开始对应，offset必须是分页大小的整数倍。</td>
</tr>
</tbody>
</table>

返回值：若映射成功则返回映射区的内存起始地址，否则返回MAP_FAILED(-1)，错误原因存于errno 中。

错误代码：
- EBADF  参数fd 不是有效的文件描述词。
- EACCES  存取权限有误。如果是MAP_PRIVATE 情况下文件必须可读，使用MAP_SHARED 则要有PROT_WRITE 以及该文件要能写入。
- EINVAL  参数start、length 或offset 有一个不合法。
- EAGAIN  文件被锁住，或是有太多内存被锁住。
- ENOMEM  内存不足。

范例：利用mmap()来读取/etc/passwd 文件内容。
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
main(){
    int fd;
    void *start;
    struct stat sb;
    fd = open("/etc/passwd", O_RDONLY); /*打开/etc/passwd */
    fstat(fd, &sb); /* 取得文件大小 */
    start = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if(start == MAP_FAILED) /* 判断是否映射成功 */
        return;
    printf("%s", start); munma(start, sb.st_size); /* 解除映射 */
    closed(fd);
}
```

执行结果：
```
root : x : 0 : root : /root : /bin/bash
bin : x : 1 : 1 : bin : /bin :
daemon : x : 2 : 2 :daemon : /sbin
adm : x : 3 : 4 : adm : /var/adm :
lp : x :4 :7 : lp : /var/spool/lpd :
sync : x : 5 : 0 : sync : /sbin : bin/sync :
shutdown : x : 6 : 0 : shutdown : /sbin : /sbin/shutdown
halt : x : 7 : 0 : halt : /sbin : /sbin/halt
mail : x : 8 : 12 : mail : /var/spool/mail :
news : x :9 :13 : news : /var/spool/news :
uucp : x :10 :14 : uucp : /var/spool/uucp :
operator : x : 11 : 0 :operator : /root:
games : x : 12 :100 : games :/usr/games:
gopher : x : 13 : 30 : gopher : /usr/lib/gopher-data:
ftp : x : 14 : 50 : FTP User : /home/ftp:
nobody : x :99: 99: Nobody : /:
xfs :x :100 :101 : X Font Server : /etc/xll/fs : /bin/false
gdm : x : 42 :42 : : /home/gdm: /bin/bash
kids : x : 500 :500 :/home/kids : /bin/bash
```
