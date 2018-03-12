---
title: i2c(5)_i2c_client
date: 2018-02-6 12:00:00
categories:
- Linux
- i2c
---

注册i2c_client，看起来有两种注册方式，但实质上只有一种。

## 静态注册
这类方法有一个弊端，使用i2c_register_board_info只是将info挂接到\__i2c_board_list上，只有注册i2c_adapter驱动时才会new \__i2c_board_list上的i2c_client(i2c_new_device)，即i2c_adapter设备是在i2c_client设备之前注册。
**所以当i2c_adapter注册完毕后再使用i2c_register_board_info，必然无法注册i2c_client**<!-- more -->
```c
i2c_register_board_info(int busnum, struct i2c_board_info const *info, unsigned len)
	for (status = 0; len; len--, info++) {
		devinfo->busnum = busnum;
		devinfo->board_info = *info;
		list_add_tail(&devinfo->list, &__i2c_board_list);
	}
```

当i2c_adapter probe时会使用__i2c_board_list进行i2c_new_device
```c
static int s3c24xx_i2c_probe(struct platform_device *pdev)
	i2c_add_numbered_adapter(&i2c->adap);
		i2c_register_adapter(adap);
			i2c_scan_static_board_info(adap);
				list_for_each_entry(devinfo, &__i2c_board_list, list) {
					if (devinfo->busnum == adapter->nr
							&& !i2c_new_device(adapter,
									&devinfo->board_info))
						dev_err(&adapter->dev,
							"Can't create device at 0x%02x\n",
							devinfo->board_info.addr);
				}	
```

## 动态注册
```c
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
