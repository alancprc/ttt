---
layout:     post
title:      根据exif信息按月组织照片
categories: perl
---

上一篇文章讲到从iPhone导出了一大堆照片，现在接着说说整理的事情。

## 原始的目录结构

```
100APPLE
100CLOUD
101APPLE
101CLOUD
102APPLE
```

## 希望的目录结构
按照照片的拍照时间，每个月一个文件夹
```
2019-04
2019-05
2019-06
2019-07
2019-08
```
## Image::ExifTool
metacpan.org 上有个很好的perl模块，[Image::ExifTool](https://metacpan.org/pod/Image::ExifTool). 于是我写了个小脚本，利用Image::ExifTool，读取.jpg/.mov的拍摄时间，如果拍摄时间不可用，则读取文件修改时间，然后按照`yy-mm`方式分到文件夹。
脚本地址[arrange-photos-by-month](https://github.com/alancprc/arrange-pictures-by-month).

