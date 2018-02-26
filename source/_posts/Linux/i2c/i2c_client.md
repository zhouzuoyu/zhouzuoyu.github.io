---
title: i2c(5)_i2c_client
date: 2018-02-6 12:00:00
categories:
- Linux
- i2c
---

注册i2c_client，看起来有两种注册方式，但实质上只有一种

## i2c_register_board_info

这类方法有一个弊端，从代码中可以看到它只是挂接到\__i2c_board_list上，
只有注册i2c_adapter驱动时才会将\__i2c_board_list的i2c_client进行i2c_new_device
**所以有可能注册不成功**
<!-- more -->
```c
/*
 * i2c_client，大致有两种注册方式，但是本质是i2c_new_device
 */
/*
 * 静态注册，将i2c_board_info挂接到__i2c_board_list上
 * i2c_adapter注册时将遍历该链表上的设备，并且注册i2c_client
 */
i2c_register_board_info(int busnum, struct i2c_board_info const *info, unsigned len)
	for (status = 0; len; len--, info++) {
		devinfo->busnum = busnum;
		devinfo->board_info = *info;
		list_add_tail(&devinfo->list, &__i2c_board_list);
	}
```

## i2c_new_device

```c
// 动态注册
i2c_new_device(struct i2c_adapter *adap, struct i2c_board_info const *info)
	client = kzalloc(sizeof *client, GFP_KERNEL);
	client->adapter = adap;
	// i2c_board_info传递给i2c_client
	client->flags = info->flags;
	client->addr = info->addr;
	client->irq = info->irq;
	// check
	i2c_check_addr_validity(client->addr, client->flags);
		if (addr == 0x00 || addr > 0x7f)
			return -EINVAL;
	// device register
	client->dev.bus = &i2c_bus_type;
	client->dev.type = &i2c_client_type;
	device_register(&client->dev);
```
