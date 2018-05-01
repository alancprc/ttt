---
layout: post
title:  autofs failed to mount localhost when cable disconnected
categories: linux
---

## 问题表现
最近同事把公司的工作站(CentOS 6.2)搬到客户实验室做个实验，结果发现公司的软件(姑且称为`foobar`)在客户那没法启动，系统启动时间也很长。

## 问题重现
考虑到在客户实验室，和自己实验室的区别，我们把网线给拔了，然后重启，果然发现启动时间很长，并且foobar启动报错。

## 问题定位与分析

### 启动时间过长
启动的时候，把CentOS的图形启动界面切换到文字启动界面(左右箭头即可切换)，发现卡在`sendmail`服务。
这个之前有碰到，是因为本机hostname没法解析，于是新建了一个`/etc/hostname`并填入hostname，并在`/etc/hosts`中添加 hostname。

### foobar无法启动
查看同事在客户实验室保存的log，看到有一个报错消息提到`/net/<hostname>` 这个目录。正巧我前阵子看了些用`autofs`自动挂在nfs目录的文章，感觉和`autofs`相关。怀疑是系统启动时没有网络，导致autofs服务异常，没有把本机`/`挂载到`/etc/<hostname>`；而`foobar`又依赖于`/etc/<hostname>`，于是`foobar`无法启动。

查看`/etc/auto.master`文件，其中有一行`/net  -hosts`，查了一下，archlinux wiki如是说：
> Note: Each host name needs to be resolveable, e.g. the name an IP address in /etc/hosts or via DNS and please make sure you have nfs-utils installed and working. You also have to enable RPC systemctl start.

应该是说，`autofs`自动探测所有hostname的可挂载目录，将他们挂载到`/net`，前提是，这些hostname可以被解析，意味着他们要么被写在`/etc/hosts`文件中，要么可通过dns查询到。
鉴于拔掉网线了，本机hostname无法通过dns查询得来，也不在`/etc/hosts`中，`autofs`自然不会挂载本机`/`目录到`/net/<hostname>`。

## 解决方案
在`/etc/hosts`文件中，`127.0.0.1` 所在行的行末添加本机hostname。
并注意，在`/etc/sysconfig/network`中修改hostname之后，也需写入`/etc/hosts`文件。

## 其他
`/etc/hostname`并没有作用。
因为之前接触的是archlinux，arch中设置hostname是在`/etc/hostname`中写入。
而CentOS 6.2并不一样，是通过`/etc/sysconfig/network`文件设定hostname。
定位到和hostname相关时，我第一反应是在`/etc/hostname`和`/etc/hosts`中加入hostname信息，但是后来发现在CentOS 6.2下，`/etc/hostname`没有任何作用。




## references:
1. Archlinux wiki: autofs
[https://wiki.archlinux.org/index.php/autofs#NFS_network_mounts](https://wiki.archlinux.org/index.php/autofs#NFS_network_mounts)
