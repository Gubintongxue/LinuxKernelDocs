# 基于crash+vmlinux+kcore调试内核

**1\. 引言**

　　crash具有强大的调试功能，不仅仅可以调试内核也可以调式用户进程，下面通过一个简单的例子简要介绍一下如何查看一个文件被打开的次数．

　　这个例子主要是为了方便大家对crash有一个非常基本的认识，至少可以上手用它了．

  

**２．例子**

　　先写如下代码：

　　#include <stdio.h>

　　#include <stdlib.h>

　　#include <string.h>

　　#include <fcntl.h>

　　#include <unistd.h>

  

int main(int argc, char\*\* argv) {

int flag = O\_WRONLY;

if (argc > 1) {

flag = atoi(argv\[1\]);

}

printf("open file with flag=%d\\n", flag);

int fd = open("crashlookupfile",flag);

getchar();

}

  

**3\. 启动该进程,　默认以写的方式打开**

./testfile

  

**4\. 启动crash调试环境**

　假设内核对应的可调试镜像文件是vmlinux-86.

/proc/kcore是内核提供的将内存信息以elf格式提供出来，理论上可以查看内核的所有数据也可以查看用户态的所有数据．

　启动命令如下：

　crash /home/zp/vmlinux-86 /proc/kcore

如果启动成功最后看到的打印信息如下：

KERNEL: xxxx/vmlinux-86

DUMPFILE: /proc/kcore

CPUS: 64

DATE: Sat May 30 15:58:27 2020

UPTIME: 20:42:55

LOAD AVERAGE: 0.53, 0.32, 0.25

TASKS: 1355

NODENAME: xxx-PC

RELEASE: 4.19.0-arm64-server

VERSION: #86 SMP Fri May 29 10:25:07 CST 2020

MACHINE: aarch64 (unknown Mhz)

MEMORY: 62.4 GB

PID: 49092

COMMAND: "crash"

TASK: ffff8483d5dc9a80 \[THREAD\_INFO: ffff8483d5dc9a80\]

CPU: 38

STATE: TASK\_RUNNING (ACTIVE)

  

crash>

  

**4.1 查找这个进程的pid**

crash> ps |grep testfile

47764 47762 52 ffff848399efc240 IN 0.0 1912 972 testfile

这个47764就是该testfile的pid, 47762是它的父进程pid, ffff848399efc240是该进程的主线程的task\_struct这个数据结构的首地址，基它线程的task\_struct可以通过next, prev指针遍历到

task\_struct信息非常有用，但是对于这个简单问题，还用不上，直接用files命令即可．

  

**４.2 找到testfile这个进程打开的文件的情况**

crash> files 47764

PID: 47764 TASK: ffff848399efc240 CPU: 52 COMMAND: "modifymem"

ROOT: / CWD: /home/zp

FD FILE DENTRY INODE TYPE PATH

0 ffff8483b831c700 ffff84839eb16240 ffff848389273230 CHR /dev/pts/2

1 ffff8483b831c700 ffff84839eb16240 ffff848389273230 CHR /dev/pts/2

2 ffff8483b831c700 ffff84839eb16240 ffff848389273230 CHR /dev/pts/2

3 ffff84839ff17200 ffff8483aaef18c0 ffff8483892b4bb8 REG /home/zp/crashlookupfile

该进程打开了4个文件，其中最后那么文件/home/zp/crashlookupfile是我们要关心的

  

这儿FILE对应内核的struct file数据结构

DENTRY对应内核的struct dentry数据结构

INODE对应内核的struct inode数据结构

查看这些数据结构的原型用ptype struct file就可以了．

  

crash提供查看这些地址的内容的命令和数据结构的命名一致，去掉struct直接用后面的名字就可以了．

  

**4.3 查看crashlookupfile被引用的情况**

首先查看名字确认一下是不是我们想要看的文件，可以灵活地使用这几种方式：

第一种方式：

dentry ffff8483892b4bb8

这些会看到整个数据结构的信息，显示出来后通过查看的方式会看到如下信息：

d\_name = {

{

{

hash = 1942099336,

len = 15

},

hash\_len = 66366608776

},

name = 0xffff8483aaef18f8 "crashlookupfile"

},

  

  

第二种方式，基于grep搜索

crash> dentry ffff8483aaef18c0|grep name

d\_name = {

name = 0xffff8483aaef18f8 "crashlookupfile"

d\_iname = "crashlookupfile\\000\\060\\060\\060\\060\\060.peu0-c0\\000\\000",

  

第三种方式，使用数据结构成员的完整名字

crash> dentry.d\_name.name ffff8483aaef18c0

d\_name.name = 0xffff8483aaef18f8 "crashlookupfile"

  

继续查看文件引用情况：

dentry ffff8483aaef18c0

d\_lockref = {

{

lock\_count = 8589934592,

{

lock = {

{

rlock = {

raw\_lock = {

{

val = {

counter = 0

},

{

locked = 0 '\\000',

pending = 0 '\\000'

},

{

locked\_pending = 0,

tail = 0

}

}

}

}

}

},

**count = 1**

}

}

这儿的count就是该文件被引用的次数

  

当数据结构了解后，直接使用下面的命令：

crash> dentry.d\_lockref.count ffff8483aaef18c0

d\_lockref.count = 1

  

**4.4 再启动一个进程打开这个文件确认引用计数是否会增加**

这次传入的是O\_RDONLY标志，即以只读的方式来打开文件

./testfile 0

  

再次使用

crash> dentry.d\_lockref.count ffff8483aaef18c0

d\_lockref.count = 2

可以看到引用次数已经增加为2了．

  

**4.5 为什么引用计数会立即反映到第一个进程来呢？**

先确认这两个进程的pid

crash> ps |grep testfile

47764 47762 52 ffff848399efc240 IN 0.0 1912 972 testfile

48299 47328 55 ffff848388134240 IN 0.0 1912 980 testfile

  

直接4289查看该进程打开的文件

crash> files 48299

PID: 48299 TASK: ffff848388134240 CPU: 55 COMMAND: "modifymem"

ROOT: / CWD: /home/home/uos/cyc

FD FILE DENTRY INODE TYPE PATH

0 ffff8483cee7f600 ffff848389350b40 ffff84838934c470 CHR /dev/pts/3

1 ffff8483cee7f600 ffff848389350b40 ffff84838934c470 CHR /dev/pts/3

2 ffff8483cee7f600 ffff848389350b40 ffff84838934c470 CHR /dev/pts/3

3 ffff8483d7911500 **ffff8483aaef18c0 ffff8483892b4bb8** REG home/zp/crashlookupfile

  

这个是47764进程的：

3 ffff84839ff17200 **ffff8483aaef18c0 ffff8483892b4bb8** REG /home/zp/crashlookupfile

  

原因已经很清楚，因为两个进程打开的是同一个文件，所以在内核态它们共享同样的struct dentry和struct inode, 但struct file这个数据结构是对应自己进程打开的fd的，和打开的哪个文件没有直接关系，

所以这个是以进程为单位，每一个fd一份struct file.

  

**5.　使用lsof查看同一个文件被打开的情况**

　　lsof命令可以做到，如下：

root@uos-PC:/home/zp# lsof |grep crashlookupfile

testfile 47764 root 3w REG 8,4 0 10879403 /home/zp/crashlookupfile

testfile 48299 root 3r REG 8,4 0 10879403 /home/zp/crashlookupfile

lsof的原理是遍历每个进程打开的文件，这儿只需要再加一个grep命令搜索我们关心的文件就可以了．

　　 lsof只是列出了这个文件被哪些时程占用，而crash则可以做到获取这个被打的文件的详细信息，因此将两者结合起来就可以获得这个文件的所有信息了，上面讲的例子只是去看了引用计数这一项．

  

**６．总结**

　　crash基于vmlinux，kcore及gdb提供了强大的调试功能，理论上可以查看任何一个进程的所有数据也包括内核的所有数据．上面的例子中我们直接从file相关的内核数据结构出来来查看信息，更强大的是通过task\_struct来查看．

　　基于crash提供的强大的读写功能及对内核数据结构及原理的了解，可以方便我们调试获取各种内存信息来解决疑难问题

　　对于内核本身的一些模块比较简单的方式是基于内核提供的全局主量来查看．

　　对于上面提到的文件引用情况及lsof和crash命令结合起来更好，因为通过我们不知道这个文件是被哪个进程打开的．而lsof的遍历加上grep的搜索功能能让我们快速知道



## 参考

https://zhuanlan.zhihu.com/p/146458802