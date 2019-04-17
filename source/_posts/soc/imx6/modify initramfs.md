---
title: modify initramfs
date: 2019-04-17 16:30:00
categories:
- imx6
---

> 一定要在root用户下执行以下操作
# 解压
```
# dd if=uramdisk.img of=ramdisk.img.gz skip=64 bs=1
# gunzip ramdisk.img.gz
# mkdir ramdisk; cd ramdisk
# cpio -i < ../ramdisk.img
<make any changes needed>
```

# 重新打包
```
# find . | cpio --create --format='newc' | gzip > ../ramdisk.img
# mkimage -A arm -O linux -T ramdisk -C none -a LOADADDRESS -n "Label you want" -d ./ramdisk.img ./uramdisk.img
```
