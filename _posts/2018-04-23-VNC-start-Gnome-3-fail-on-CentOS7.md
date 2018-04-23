---
layout: post
title: "VNC start Gnome 3 failed on CentOS 7"
categories: linux
---

first level header
=

second level header
==

system info:
OS:     CentOS 7.2  
VNC:    tiger vnc server  
DE:     Gnome 3  

when I tried to connect to the vnc server, I got an error message saying something went wrong, and I need to logout. then I was left with a black screen cannot do anything.
![pic-vnc-gnome3-went-wrong1]({{ site.baseurl }}/pic/vnc-gnome3-went-wrong1.png)
![pic-vnc-gnome3-went-wrong2]({{ site.baseurl }}/pic/vnc-gnome3-went-wrong2.png)

later I searched the web and found many post there saying vnc with gnome-session have similar issue.
with original ~/.vnc/xstartup, it actually starts a gnome-session, which is the default DE on my CentOS 7 system.
So i modified xstartup to start kde with `startkde &` and a kde session was started successfully on vnc.

![pic-xstartup-old]({{ site.baseurl }}/pic/xstartup-old.png)
![pic-xstartup-new]({{ site.baseurl }}/pic/xstartup-new.png)
