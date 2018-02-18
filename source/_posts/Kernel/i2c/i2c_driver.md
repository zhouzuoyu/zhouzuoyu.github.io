---
title: i2c(6)_i2c_driver
date: 2018-02-6 12:00:00
categories:
- Kernel
- i2c
---

一般编写的i2c驱动就是此类驱动
* i2c_client和i2c_driver的name能匹配上，则调用i2c_driver->probe函数(符合设备总线驱动模型)
<!-- more -->

```c
// i2c_driver
i2c_register_driver
	// driver register
	driver->driver.owner = owner;
	driver->driver.bus = &i2c_bus_type;
	driver_register(&driver->driver);
	/*
	 * 对i2c_bus上所有的device都执行__process_new_driver(call driver->detect && driver->attach_adapter)
	 */
	i2c_for_each_dev(driver, __process_new_driver);	// (data, fn)
		bus_for_each_dev(&i2c_bus_type, NULL, data, fn);
			// 从i2c_bus取出device
			while ((dev = next_device(&i)) && !error)
				error = fn(dev, data);
		// call driver->detect && driver->attach_adapter
		__process_new_driver
			i2c_do_add_adapter(data, to_i2c_adapter(dev));
				// 1st: driver->detect
				i2c_detect(adap, driver);
					if (!driver->detect || !address_list)
						return 0;
					/* 对address_list[]中的所有address都进行i2c_detect_address */
					for (i = 0; address_list[i] != I2C_CLIENT_END; i += 1) {
						temp_client->addr = address_list[i];
						// 先call driver->detect，成功则i2c_new_device
						err = i2c_detect_address(temp_client, driver);
							// 1st: call driver->detect
							info.addr = addr;
							err = driver->detect(temp_client, &info);
							// 2nd: i2c_new_device
							client = i2c_new_device(adapter, &info);
							list_add_tail(&client->detected, &driver->clients);
						if (unlikely(err))
							break;
					}
				// 2nd: driver->attach_adapter
				driver->attach_adapter(adap)
```
