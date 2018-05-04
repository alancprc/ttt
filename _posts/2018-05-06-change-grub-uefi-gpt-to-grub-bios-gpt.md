---
layout: post
title:  change grub-uefi-gpt to grub-bios-gpt
categories: linux
---

## 问题来源
上次提到在单位的工作站上的空闲硬盘上，安装了一个CentOS 7系统。但不完美的地方在于，安装的时候没有考虑到双系统启动，导致安装完成之后，在第一块硬盘上的CentOS 6和第二块硬盘的CentOS 7之前切换的时候，必须进入BIOS设置启动顺序，颇为麻烦。
于是想着如何用一个grub引导两个系统。

## 系统信息
- hdd1:
    - CentOS 6
    - grub legacy 0.97
    - MRR
- hdd2:
    - CentOS 7
    - grub2
    - GPT，有单独EFI分区

## 走过的弯路
### 首先想到的是os-prober
自然，利用os-prober直接生成CentOS 6的菜单。
但是grub2-mkconfig生成的grub.cfg中，CentOS 6的菜单，是用linuxefi/initrdefi 加载kernel和initramfs，如下：

```
menuentry 'CentOS release 6.2 test' --class gnu-linux --class gnu --class os $menuentry_id_option 'osprober-gnulinux-simple-0ffb7b13-c89f-4759-8af0-083a35c4a879' {
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1  0d239944-4594-4ac7-ba60-1b3026cb6a47
        else
          search --no-floppy --fs-uuid --set=root 0d239944-4594-4ac7-ba60-1b3026cb6a47
        fi
        kernelefi /vmlinuz-2.6.32-220.el6.x86_64 ro root=UUID=0ffb7b13-c89f-4759-8af0-083a35c4a879 nomodeset rd_NO_LUKS KEYBOARDTYPE=pc KEYTABLE=us LANG=en_US.UTF-8 rd_NO_MD quiet SYSFONT=latarcyrheb-sun16 rhgb rd_NO_LVM rd_NO_DM
        initrdefi /initramfs-2.6.32-220.el6.x86_64.img
}
```

而`linuxefi`会提示`kernel too old`，不能引导。  
将`linuxefi`改成`linux`、`linux16`呢？`command not found`
所以，os-prober看来行不通

### 备份-重新安装成BIOS+MBR/GPT模式-还原
肯定能解决，但是动作有点大，时间也比较长，先不考虑。

## 解决方案
冷静下来，细细想一下我们的现状和目标。  
两块硬盘上，分别是GRUB+BIOS+MBR/GRUB+UEFI+GPT。要实现一个GRUB引导两个系统。  
如果有办法能把GRUB+UEFI+GPT变成GRUB+BIOS+GPT，似乎就能达到我们的目标了。

先看一下GRUB+BIOS+GPT和GRUB+UEFI+GPT区别：
- BIOS+GRUB+GPT:
    - 需要一个[BIOS 启动分区](https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html#BIOS-installation)
    - 建立BIOS启动分区的方式参见 [arch wiki](https://wiki.archlinux.org/index.php/GRUB#GUID_Partition_Table_.28GPT.29_specific_instructions)
- UEFI+GRUB+GPT:
    - 需要一个`ESP`(EFI System Partition)分区。[EFI System Partition](https://wiki.archlinux.org/index.php/GRUB#Check_for_an_EFI_System_Partition)

所以把`ESP`分区变成`BIOS 启动分区`，然后重新以BIOS方式安装GRUB，应该就可以。

1. 改ESP分区为BIOS 启动分区，可用fdisk/parted等
- 修改之前:

```
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sda: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  525MB   524MB   fat16        EFI System Partition  boot
 2      525MB   1050MB  524MB   ext4
 3      1050MB  66.6GB  65.5GB                                     lvm
```

- 修改之后:
```
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sda: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      17.4kB  1049kB  1031kB                     bios_grub
 2      525MB   1050MB  524MB   ext4
 3      1050MB  66.6GB  65.5GB                     lvm
```

2. 更新/etf/fstab文件，注释掉包含/boot/efi的一行
```
# /etc/fstab
# Created by anaconda on Thu May  3 15:53:56 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos7-root /                       ext4    defaults        1 1
UUID=ae36de86-3ee1-4e0f-9d1f-11d0e190c832 /boot                   ext4    defaults        1 2
# UUID=00C9-A677          /boot/efi               vfat    umask=0077,shortname=winnt 0 0
/dev/mapper/centos7-var /var                    ext4    defaults        1 2
/dev/mapper/centos7-swap swap                    swap    defaults        0 0
```

3. 重新安装grub
```
yum install grub2-pc
grub2-install  --target=i386-pc /dev/sda
```

4. 重新生成/boot/grub2/grub.cfg
```
grub2-mkconfig  -o /boot/grub2/grub.cfg
```
**注意**：
- 安装grub时可能有warning，需要删除原先的symlink `rm /boot/grub2/grubenv`
- 重新生成的grub.cfg可能仍然使用linuxefi/initrdefi命令，重启时会提示linuxefi command not found。需要修改为linux16/initrd16。

## 等等，还有最后一步
上面做下来，重启后发现进入emergency mode，看log发现和boot-efi相关，显示还有些内容需要改动。
但是 rescue mode 正常启动，于是查看grub.cfg中两者的差别：
```
### BEGIN /etc/grub.d/10_linux ###
menuentry 'CentOS Linux (3.10.0-327.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-327.el7.x86_64-advanced-65a46f15-8b6f-430d-9a5b-6803e18b5acb' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_gpt
	insmod ext2
	set root='hd0,gpt2'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2  ae36de86-3ee1-4e0f-9d1f-11d0e190c832
	else
	  search --no-floppy --fs-uuid --set=root ae36de86-3ee1-4e0f-9d1f-11d0e190c832
	fi
	linux16 /vmlinuz-3.10.0-327.el7.x86_64 root=/dev/mapper/centos7-root ro rd.lvm.lv=centos7/root rd.lvm.lv=centos7/swap rhgb quiet 
	initrd16 /initramfs-3.10.0-327.el7.x86_64.img
}
menuentry 'CentOS Linux (0-rescue-cb7527f57967497db1ab0053cd791a95) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-cb7527f57967497db1ab0053cd791a95-advanced-65a46f15-8b6f-430d-9a5b-6803e18b5acb' {
	load_video
	insmod gzio
	insmod part_gpt
	insmod ext2
	set root='hd0,gpt2'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2  ae36de86-3ee1-4e0f-9d1f-11d0e190c832
	else
	  search --no-floppy --fs-uuid --set=root ae36de86-3ee1-4e0f-9d1f-11d0e190c832
	fi
	linux16 /vmlinuz-0-rescue-cb7527f57967497db1ab0053cd791a95 root=/dev/mapper/centos7-root ro rd.lvm.lv=centos7/root rd.lvm.lv=centos7/swap rhgb quiet 
	initrd16 /initramfs-0-rescue-cb7527f57967497db1ab0053cd791a95.img
}

### END /etc/grub.d/10_linux ###
```

一是kernel，二是initramfs，试下来是initramfs的问题，于是重新生成initramfs。
如果以normal版本内核搭配rescue initramfs启动：
```
dracut -f
```
如果以 rescue mode 启动，需要指定kernel版本
```
dracut -f /boot/initramfs-3.10.0-327.el7.x86_64.img 3.10.0-327.el7.x86_64
```
重启后正常启动。


## references:
1. Archlinux wiki: GRUB
[https://wiki.archlinux.org/index.php/GRUB](https://wiki.archlinux.org/index.php/GRUB)
2. sys cook book: RHEL: Rebuilding the initial ramdisk image
[https://sites.google.com/site/syscookbook/rhel/rhel-kernel-rebuild](https://sites.google.com/site/syscookbook/rhel/rhel-kernel-rebuild)
3. grub online manual
[https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html#BIOS-installation](https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html#BIOS-installation)
