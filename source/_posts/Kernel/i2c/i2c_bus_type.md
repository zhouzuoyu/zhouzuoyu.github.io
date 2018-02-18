---
title: i2c(4)_i2c_bus_type
date: 2018-02-6 12:00:00
categories:
- Kernel
- i2c
---

# 源码
> I2c-core-base.c (linux-4.13.1\drivers\i2c)

# 源码分析
通过name来进行match，match成功则调用driver->probe
```c
// match(strcmp(client->name, driver->id_table->name))成功，则调用driver->probe
struct bus_type i2c_bus_type = {
	.name		= "i2c",
	.match		= i2c_device_match,
	.probe		= i2c_device_probe,
	.remove		= i2c_device_remove,
	.shutdown	= i2c_device_shutdown,
};
```
<!-- more -->
## 匹配
```c
/*
 * 3种匹配方式
 * 主要集中在i2c_driver->driver->of_match_table/i2c_driver->id_table和dts进行match
 */
i2c_device_match(struct device *dev, struct device_driver *drv)
	// i2c_driver->driver->of_match_table和dts进行match
	if (i2c_of_match_device(drv->of_match_table, client))
		of_match_device(matches, &client->dev);
			of_match_node(matches, dev->of_node);
				__of_match_node(matches, node);
					for (; matches->name[0] || matches->type[0] || matches->compatible[0]; matches++)
						score = __of_device_is_compatible(node, matches->compatible, matches->type, matches->name);
	// i2c_driver->driver->of_match_table/i2c_driver->driver->acpi_match_table
	if (acpi_driver_match_device(dev, drv))
		return acpi_of_match_device(ACPI_COMPANION(dev), drv->of_match_table);
	// i2c_driver->id_table和dts进行对比
	if (i2c_match_id(driver->id_table, client))
		return 1;
```

## 调用driver->probe
```c
i2c_device_probe
	driver = to_i2c_driver(dev->driver);
		// call driver->probe_new | driver->probe
		if (driver->probe_new)
			status = driver->probe_new(client);
		else if (driver->probe)
			status = driver->probe(client, i2c_match_id(driver->id_table, client));
```
