---
title: nand(1)_overview
date: 2018-02-5 12:00:00
categories:
- Linux
- flash
---

nand驱动大概分为3部分

>   {% post_link Kernel/nand/nand_controller nand_controller %}

linux-4.13.1\drivers\mtd\nand\S3c2410.c
nand controller driver, 控制器的驱动, 填充nand_chip结构体，调用nand_scan


>   {% post_link Kernel/nand/nand_device nand_device %}

linux-4.13.1\drivers\mtd\nand\Nand_samsung.c
nand driver, nand这颗料本身，获取pagesize/blocksize/oob size...
__通过nand_ids.c这个表将nand_controller和nand_device链接起来__


>   nand协议层

