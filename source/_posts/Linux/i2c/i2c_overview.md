---
title: i2c(1)_overview
date: 2018-02-6 12:00:00
categories:
- Linux
- i2c
---

i2c驱动框架在kernel中大致分为3块

1.  i2c协议层

    协议相关的，脱离具体硬件平台，kernel中已经实现

2.  i2c总线驱动

    <!-- more -->
    i2c controller相关code，这部分每个厂商各不相同，驱动框架中将接口给出，填充即可
    这部分代码各个厂商已经提供，基本上也不需要修改

    1.  {% post_link Kernel/i2c/i2c_adapter i2c_adapter %}
    2.  {% post_link Kernel/i2c/i2c_algorithm i2c_algorithm %}

3.  i2c设备驱动

    一般的i2c驱动都是涉及该部分

    1.  {% post_link Kernel/i2c/i2c_client i2c_client %}
    2.  {% post_link Kernel/i2c/i2c_driver i2c_driver %}
