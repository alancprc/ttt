---
layout:     post
title:      VirtualBox NAT 模式下 mount NFS
categories: virtualbox
---

在VirtualBox 5.2.18之前，如果NFS server没有设置`insecure`的话，虚拟机内要访问NFS需要设置Bridge桥接模式。
NFS server没有设置insecure时，要求client连接的端口号小于1024。
而默认NAT下，尽管Guest可以使用1024以下的端口，但经过NAT后，Host其实是1024以上的端口号在跟NFS server连接，然后被无情地拒绝。

作为对比，VMWare Player的NAT模式就可以直接挂载NFS。。。

其实，早在许多年前，VirtualBox的文档上有就一个`sameports`选项，可以让NAT模式下Host的端口号和Guest
OS的端口号一样，只是很可惜，实际上并没有起作用，所以数年前就有人在VirtualBox上提交了相应的bug，直到2018年发布的5.2.18版本中才被修复。
所以，在5.2.18上，就可以方便地用NAT挂载NFS，不需要bridge了。

这个设置需要通过命令行开启，GUI上并没有相关的设置。

```
VBoxManage.exe modifyvm <vm name> --nataliasmode<N> sameports
```
其中，<vm name>是VM的名字，<N>是NAT adapter的编号，比如我只加了第一个NAT adapter的时候，N=1。即
```
VBoxManage.exe modifyvm CentOS7 --nataliasmode1 sameports
```

sameports bug 链接：
[the option --nataliasmode1 sameports is not honoured -> fixed in 5.2.18](https://www.virtualbox.org/ticket/13000)

VirtualBox文档：
![NAT Networking settings]({{ site.baseurl }}/pic/virtualbox-nat-network-settings.png)
