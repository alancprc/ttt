---
layout: post
title:  Linux system backup and restore (BIOS+MBR)
categories: linux
---

Recently I need to backup and restore a CentOS 6(BIOS+MBR boot).
After searching and several experiment, I was successfully backup my own Archlinux and restore it on a brand new disk, in my virtualbox.
What I've done is as follows.

# 1. backup mbr on current disk
use dd to backup mbr, actually include partition table and grub bootcode

`dd if=/dev/sda of=/backupdir/mbr-backup.img bs=512 count=2048`

where `2048` should be the start sector number of the first partition.

you can get the start sector number by running:  
`fdisk /dev/sda`

# 2. backup system files
use rsync to backup to a filesystem supports ACLs...  
well, not sure if nfs supports. if not, external hdd will be needed.

```
rsync -aHAX --info=progress2 --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /path/to/backup/folder
```

# 3. restore mbr on new disk
use dd again to restore mbr besides partition table and grub bootcode.  
if the volumn of new disk varies the old one, re-config the partition using
tools like fdisk/cfdisk/parted.

`dd if=/backupdir/mbr-backup.img of=/dev/sdX bs=512 count=2048`

should double confirm with the /dev/sd**X** before running this command.

# 4. format each new partition
take ext3 for example:

`mkfs.ext3 /dev/sdXn`

also, double confirm with the /dev/sd**X***n* before running this command.

# 5. mount new partitions
e.g. mount /dev/sd**n***X* to /mnt /mnt/boot /mnt/var etc.

# 6. restore system files
```
rsync -aHAX --info=progress2 --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} /path/to/backup/folder/ /mnt
```

# 7. update /boot/grub/grub.cfg and /etc/fstab  
two method, I'll prefer method 1
1. get uuid of each new partition by running:  
`lsblk -a -f`  
or  
`ls -l /dev/disk/by-uuid`  
then update grub.cfg and fstab

2. backup uuid of partitions on old disk,  
then set the old uuid back to each corresponding new partition by:  
`tune2fs /dev/sdXn -U *uuid*`






# references:
1. Archlinux wiki: System backup  
[https://wiki.archlinux.org/index.php/System_backup](https://wiki.archlinux.org/index.php/System_backup)
1. Archlinux wiki: rsync as a backup utility  
[https://wiki.archlinux.org/index.php/Rsync#As_a_backup_utility](https://wiki.archlinux.org/index.php/Rsync#As_a_backup_utility)
1. Archlinux wiki: backup partition table using dd  
[https://wiki.archlinux.org/index.php/Fdisk#Using_dd](https://wiki.archlinux.org/index.php/Fdisk#Using_dd)
1. The difference between booting MBR and GPT with GRUB  
[https://www.anchor.com.au/blog/2012/10/the-difference-between-booting-mbr-and-gpt-with-grub](https://www.anchor.com.au/blog/2012/10/the-difference-between-booting-mbr-and-gpt-with-grub/)
1. 查看和修改分区uuid  
[https://blog.csdn.net/chrisniu1984/article/details/7245711](https://blog.csdn.net/chrisniu1984/article/details/7245711)
