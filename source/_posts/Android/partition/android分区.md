---
title: android分区
date: 2018-05-17 18:28:00
categories: 
- Android
- partition
---

|partition	| name    									| size    	| Description
| --------  | :----- 									| :-----	| :---- |
| 0         | MBR										| 256K		| Master Boot Record and GUID Partition Table	|
| 1        	| preloader									| 256K		| First stage bootloader	|
| 2        	| bootloader(u-boot.bin)					| 384K		| Second stage bootloader 	|
| 3        	| misc										| 128K		| Second stage bootloader 	|
| 4        	| recovery (zImage + recovery-ramdisk.img) 	| 8M		| Second stage bootloader	|
| 5        	| boot (boot.img = zImage + ramdisk.img)	| 8M		| Second stage bootloader	|
| 6        	| system (system.img)						| 256M		| Second stage bootloader	|
| 7        	| cache (cache.img)							| 32M		| Second stage bootloader	|
| 8        	| userdata (userdata.img)					| Remaining	| Second stage bootloader	|
<!--more-->
1. misc
	这个分区包含各种复杂的类似于on/off的系统设置。这些设置可能是USB配置和某些硬件配置信息。这是一个重要的分区，如果该分区损坏或者丢失，设备的功能可能就工作不正常。
2. cache
	这个分区是Android系统存储频繁访问的数据和app的地方。擦除这个分区不影响你的个人数据，当你继续使用设备时，被擦除的数据就会自动被创建。

[Android eMMC 分区详解](https://blog.csdn.net/firefox_1980/article/details/38824143)
[android手机各大分区详解](https://www.cnblogs.com/yyh8/p/6724131.html)
[ramdisk.img](https://blog.csdn.net/allon19/article/details/37818905)
[MTK智能平台分区解析](http://blog.163.com/zz_forward/blog/static/212898222201301424257601/)
