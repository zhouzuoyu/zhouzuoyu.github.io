---
title: Input子系统(1)_Overview
date: 2018-01-30 14:11:06
categories:
- Linux
- Input
---

input子系统下的设备分为两类(input_handler/input_dev)
input_handler：

>   可以用于处理各种device的事件，如evdev.c

input_dev：

>   一般驱动写的就是该类设备，选择合适的handler来处理设备的event，如s3c2410_ts.c
<!-- more -->
input子系统有两条链表(**input_handler_list/input_dev_list**)，如注册input_dev时需遍历匹配input_dev_list上所有的input_handler，反之亦然。

```c
input_register_handler
	list_for_each_entry(dev, &input_dev_list, node)
		input_attach_handler(dev, handler);
```

```c
input_register_device
	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
```

input_dev connect handler时将会new handle。

handle中有input_dev和input_handler的信息(handle->dev = input_dev，handle->handler = handler)，并且一个input_dev对应多个handle，handle对应一种handler，即构建了一个input_dev可以有多种handler对它进行处理的框架。
