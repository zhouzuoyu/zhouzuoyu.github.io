---
title: i2c(2)_adapter
date: {{ date }}
categories:
- Kernel
- i2c
---

## 代码
> linux-4.13.1\drivers\i2c\busses\i2c-s3c2410.c

## 功能
> 总线驱动，注册i2c_adapter

## 注册总线驱动
* 挂接i2c_algorithm
* 注册i2c_adapter

<!-- more -->
```c
s3c24xx_i2c_probe
	i2c->adap.algo = &s3c24xx_i2c_algorithm;	// 挂载i2c_algorithm
	i2c_add_numbered_adapter(&i2c->adap);	// register i2c_adapter
```

### i2c_add_numbered_adapter

* 注册/dev/i2c-x设备节点
* 对挂载在该总线上的所有i2c设备都i2c_new_device

```c
i2c_add_numbered_adapter(&i2c->adap);
	i2c_add_adapter(adap);
		id = of_alias_get_id(dev->of_node, "i2c");	// if i2c0 then id = 0
		adapter->nr = id;
		__i2c_add_numbered_adapter(adapter);
			i2c_register_adapter(adap);
				// 1：注册/dev/i2c-x
				dev_set_name(&adap->dev, "i2c-%d", adap->nr);
				adap->dev.bus = &i2c_bus_type;	// FIXME
				adap->dev.type = &i2c_adapter_type;
				res = device_register(&adap->dev);
				// 2：遍历__i2c_board_list，将挂载在该busnum上的i2c_device通过i2c_new_device创建i2c_client
				i2c_scan_static_board_info(adap);
					list_for_each_entry(devinfo, &__i2c_board_list, list)
						if (devinfo->busnum == adapter->nr && !i2c_new_device(adapter, &devinfo->board_info))
				// 3：对每个i2c_driver都callback __process_new_adapter
				bus_for_each_drv(&i2c_bus_type, NULL, adap, __process_new_adapter);
				// __process_new_adapter
					i2c_do_add_adapter(to_i2c_driver(d), data);
```

### i2c_do_add_adapter

```c
// process_new_adapter || process_new_driver will call i2c_do_add_adapter
i2c_do_add_adapter
	/*
	 * i2c_driver必须同时指定address_list和detect
	 * 对所有的address都执行detect：尝试所有的addr直到找到device
	 * address_list: 指定ic所有的addr列表
	 * detect: 是否是指定设备(读id...)
	 */
	i2c_detect(adap, driver);
		address_list = driver->address_list;
		if (!driver->detect || !address_list)
			return 0;
		for (i = 0; address_list[i] != I2C_CLIENT_END; i += 1) {
			temp_client->addr = address_list[i];
			err = i2c_detect_address(temp_client, driver);
				/*
				 * part 1: identification
				 * part 2: 设置info.type
				 */
				err = driver->detect(temp_client, &info);
				// i2c_driver->detect有设置name，则new i2c_client
				if (info.type[0] != '\0') {
					client = i2c_new_device(adapter, &info);
					list_add_tail(&client->detected, &driver->clients);
				}
			if (unlikely(err))
				break;
		}
	// callback i2c_driver->attach_adapter
	driver->attach_adapter(adap);
```
