---
layout:     post
title:      将iPhone的照片导入PC
categories: iPhone
---

前几天忙着把iPhone上的照片导入到PC上，花费了不少功夫，记录一下。
# 导入照片

## Apple/Microsoft推荐的方法
首先，自然时到苹果官方网站上，了解如何将照片导入到Windows。要点如下：

- 安装最新itunes
- 打开Microsoft Photos，右上角Import，From a USB device

还是挺容易的嘛，接上iPhone，找到了3000+照片和视频，选择目标文件夹，开始导入！
1,2,3... 100
咦，导入停止了，something went wrong.
断开手机连接，再接上，再次尝试通过Microsoft Photos导入，导入了一部分照片之后再次停止。

## OneDrive
OneDrive是个挺实惠的网盘，买Office365，赠送1T空间。用来存照片是挺足够了。
在iPhone上开启上传相册，可以自动按照年月组织照片，上传速度也还可以。
美中不足的是，iPhone版只能上传相机拍摄的照片，其他的比如截屏/下载的照片，都没法上传，不知道是不是iPhone的限制。

## FonePaw DoTrans
常规的方法不行，看看有没有什么好用的第三方软件，找到一个DoTrans，是个收费软件，单用户的费用大概rmb 220，但是提供试用，试用期间能够导入20张照片，还是挺良心的。
试用效果很不错，它不需要借助itunes，使用很稳定，用它先导出了两三个比较大的视频，先缓缓空间。不过为了不太常用的需求，花费两百多，还是有点贵。

## ifuse on ArchLinux
刚好电脑上同时也装了Archlinux，Archlinux素来以文档齐全，aur资源超级丰富著称。说不定有惊喜。
[https://wiki.archlinux.org/index.php/IOS](https://wiki.archlinux.org/index.php/IOS) 上介绍了两种方式
- `gvfs`方式。安装 `gvfs-afc` 和 `gvfs-gphoto2`，然后通过文件管理器，如GNOME Files、Thunar(xfce4)
- `ifuse`方式。安装`ifuse`，直接挂载iPhone的内部存储导入照片。
一开始没看到`ifuse`方式，前一天晚上通过Thunar打开`gvfs`挂载点，通过`rsync`导入了一半，第二天再尝试的时候居然连不上iPhone，可能是因为这期间升级了系统。不过正好，让我再次看看Arch的文档，发现了`ifuse`方式，可以直接挂载iPhone的内部存储，然后通过`rsync`进行同步，无缝接上前一天的内容，而且同步速度还翻倍了。用`gvfs`方式，同步速度大概5MB/s，`ifuse`方式则约10MB/s，完美！

# 查看照片
jpeg/png之类的常见照片格式自然不用说，几乎每个图片查看软件都支持，要说一说的是苹果现在用的`.heic`格式(介绍请参见维基百科 [https://zh.wikipedia.org/zh-cn/高效率图像文件格式](https://zh.wikipedia.org/zh-cn/高效率图像文件格式)。在Windows 10中，只要在Microsoft Store中给Microsoft Photos安装一个HEVC扩展就可以了。而对于这个HEVC扩展，有意思的是，通过Microsoft Photos的连接打开的是标价rmb 7元的收费版，但对于品牌机，还有另一个叫`来自设备制造商的 HEVC 视频扩展`，并且直接搜索无法找到，需要通过链接[自设备制造商的 HEVC 视频扩展 https://www.microsoft.com/zh-cn/p/hevc-video-extensions-from-device-manufacturer/9n4wgh0z6vhq?activetab=pivot%3Aoverviewtab](https://www.microsoft.com/zh-cn/p/hevc-video-extensions-from-device-manufacturer/9n4wgh0z6vhq?activetab=pivot%3Aoverviewtab)打开。