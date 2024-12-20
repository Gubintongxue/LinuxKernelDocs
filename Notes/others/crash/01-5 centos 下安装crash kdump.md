# centos 下安装crash kdump

2015文章，可能不再具有参考意义

linux内核发送崩溃时，kdump会生成一个内核转储文件vmcore。 可以通过分析vmcore分析出内核崩溃的原因。  
crash是一个被广泛应用的内核奔溃转储文件分析工具。  
使用crash调试内核转储文件，需要安装crash工具和内核调试工具kernel\-debuginfo。

安装、配置、启动kdump

安装kdump：  
yum search kexec-tools  
yum install kexec-tools.x86\_64

配置kdump：  
vim /boot/grub/menu.lst： 设置crashkernel=auto 自动方式

启动kdump：  
service kdump start

安装crash

yum install crash\*

查看内核信息注意一致性

\[root@cmb-ha-111-007-163-053 ~\]# uname -r  
2.6.32-573.7.1.el6.x86\_64

\[root@cmb-ha-111-007-163-053 ~\]# cat /etc/issue  
CentOS release 6.3 (Final)  
Kernel \\r on an \\m

wget [http://debuginfo.centos.org/6/x86\_64/kernel-debuginfo-2.6.32-573.7.1.el6.x86\_64.rpm](http://debuginfo.centos.org/6/x86_64/kernel-debuginfo-2.6.32-573.7.1.el6.x86_64.rpm)

wget [http://debuginfo.centos.org/6/x86\_64/kernel-debuginfo-common-x86\_64-2.6.32-573.7.1.el6.x86\_64.rpm](http://debuginfo.centos.org/6/x86_64/kernel-debuginfo-common-x86_64-2.6.32-573.7.1.el6.x86_64.rpm)

rpm -ivh kernel-debuginfo-common-XXX.rpm

rpm -ivh kernel-debuginfo-XXX.rpm

加载到开机启动  
在 /etc/rc.local 加入service kdump start

运行

执行crash

如果死机的crash文件执行

/usr/bin/crash /usr/lib/debug/lib/modules/XXX/vmlinux vmcore

常用命令bt 看死机信息

加载模块  
mod -s \[mould\_name\] \[position\]

查看死机时的代码位置  
sym \[addr\]

查看内存信息

具体分析方法见前面博客  
[http://blog.csdn.net/divlee130/article/details/47806551](http://blog.csdn.net/divlee130/article/details/47806551)