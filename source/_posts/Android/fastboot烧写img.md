---
title: fastboot烧写img
date: 2018-05-17 17:14:00
categories: 
- Android
---

分区中会存放分区表(MBR/GPT)，fastboot mode后会读取分区表信息，从而得到各个img对应的起始地址。
所以可以使用**fastboot flash boot boot.img**来进行烧写而不用指定emmc具体地址。

[fastboot烧写img](https://blog.csdn.net/firefox_1980/article/details/38824143)
