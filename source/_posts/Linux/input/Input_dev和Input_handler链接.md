---
title: Input子系统(2)_input_dev和input_handler链接
date: 2018-01-30 15:41:36
categories:
- Linux
- input
---

注册input_dev时将会遍历所有的input_handler_list，然后调用input_attach_handler进行attach，注册input_handler亦然。

>   input_register_device，先调用所有input_handler的match函数（没有则必定成功），成功再调用connect函数

<!-- more -->
```c
int input_register_device(struct input_dev *dev)
	// 添加到input_dev_list链表
	list_add_tail(&dev->node, &input_dev_list);
	// 遍历attach
	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
```
```c
int input_register_handler(struct input_handler *handler)
	// 添加到input_handler_list
	list_add_tail(&handler->node, &input_handler_list);
	// 遍历attach
	list_for_each_entry(dev, &input_dev_list, node)
		input_attach_handler(dev, handler);
```

从以下函数可以看到，**如果一个handler中没有match函数，一定能match上**

```c
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
  	// 1: 先调用handler->match
	id = input_match_device(handler, dev);
		if (!handler->match || handler->match(handler, dev))
			return id;
	if (!id)
		return -ENODEV;

	// 2: match成功则调用handler->connect
	error = handler->connect(handler, dev, id);
```

