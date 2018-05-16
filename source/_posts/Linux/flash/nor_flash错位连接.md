---
title: nor_flash错位连接
date: 2018-05-16 14:43:00
categories:
- Linux
- flash
---

# 硬件错位连接
从这个图中可以看到nor flash的A0连接的是LADDR1，是一个错位的连接。
![nor flash硬件连接](nor_flash.png)
<!--more-->
# 分析
假如ARM处理器外部扩展的是16 bit的NOR FLASH, 地址线必须要错位连接，8 bit则不需要。
ARM处理器的数据信号D0-D15和FLASH的数据信号D0-D15是逐一对应的。而ARM处理器的地址信号和NOR FLASH的地址信号是错位连接的，ARM的A0悬空，ARM的A1连接FLASH的A0，ARM的A2连接FLASH的A1，依次类推。
需要错位连接的原因是:
> ARM处理器的每个地址对应的是一个BYTE的数据单元，而16-BIT的FLASH的每个地址对应的是一个HALF-WORD(16-BIT)的数据单元。为了保持匹配,所以必须错位连接。这样，从ARM处理器发送出来的地址信号的最低位A0对16-BIT FLASH来说就被屏蔽掉了。

**实际上原因是因为soc是以8 bit为单位，而nor flash是以16 bit为单位，所以需要硬件错位连接**

# 读取例子
1. ARM处理器需要从地址0x0读取一个BYTE
	ARM处理器在地址线An-A0上送出信号0x0;
	16-BIT FLASH在自己的地址信号An-A0上看到的地址是0x0,然后将地址0x0对应的16-BIT数据单元输出到D15-D0上;
	ARM处理器知道访问的是16-BIT的FLASH,从D7-D0上读取所需要的一个BYTE的数据.

2. ARM处理器需要从地址0x1读取一个BYTE
	ARM处理器在地址线An-A0上送出信号0x1;
	16-BIT FLASH在自己的地址信号An-A0上看到的地址__依然是0x0__, 然后将地址0x0对应的16-BIT数据单元输出到D15-D0上;
	ARM处理器知道访问的是16-BIT的FLASH,从D15-D8 上读取所需要的一个BYTE 的数据.
