---
layout: post
title: "VNC start Gnome 3 failed on CentOS 7"
categories: linux
---

单位的一台机器，原先运行CentOS 6，最近新装了一个CentOS 7系统，在设置VNC的时候遇到了点问题，没有找到彻底的解决办法，但找到了一个work around，记录如下：

# system info:
- OS:     CentOS 7.2  
- VNC:    tiger vnc server  
- DE:     Gnome 3  

用 `vncserver :1` 启动vnc端口后，客户端连接成功，但是看到的是这个界面  
![pic-vnc-gnome3-went-wrong1]({{ site.baseurl }}/pic/vnc-gnome3-went-wrong1.png)

点击 `Log Out`之后，只有一个vnc的设置窗口，其余黑乎乎一片，无法操作  
![pic-vnc-gnome3-went-wrong2]({{ site.baseurl }}/pic/vnc-gnome3-went-wrong2.png)  

# work around:
搜索一番之后，看到很多类似问题，但是没有看到有效的解决方法，有些文章提到是gnome的问题。
于是尝试将VNC启动的DE设置为KDE，重新启动vnc端口后，客户端成功连接并打开KDE界面。

打开 `~/.vnc/xstartup`，默认内容如下：  
```
#!/bin/sh

unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec /etc/X11/xinit/xinitrc
```

按照如下内容修改：  
```
#!/bin/sh

unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
#exec /etc/X11/xinit/xinitrc
startkde &
```
