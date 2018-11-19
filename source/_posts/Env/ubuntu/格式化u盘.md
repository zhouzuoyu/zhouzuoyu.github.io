---
title: ubuntu格式化u盘
date: 2018-05-31 15:06:00
categories:
- Env
- ubuntu
---

1. 查看U盘信息
> fdisk -l

	例如:
	```
	Device     Boot Start       End   Sectors   Size Id Type
	/dev/sdb4  *      256 248348100 248347845 118.4G  7 HPFS/NTFS/exFAT
	```

2. 卸载u盘
> umount /dev/sdb4

3. 格式化
> mkfs -t vfat /dev/sdb4

[ubuntu格式化优盘为fat32](https://blog.csdn.net/ztl0013/article/details/71440353)
