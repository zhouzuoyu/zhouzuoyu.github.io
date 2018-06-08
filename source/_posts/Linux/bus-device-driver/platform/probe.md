---
title: probe
date: 2018-06-08 14:13:00
categories:
- Linux
- platform
---
# probe
可以看到调用的device_driver->probe而不是platform_driver->probe
```c
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
	ret = really_probe(dev, drv);
		dev->driver = drv;	// 这句话也是关键，因为drv->probe实际是platform_drv_probe，而在platform_drv_probe中要用到dev->driver
		
		if (dev->bus->probe) {
			ret = dev->bus->probe(dev);
		} else if (drv->probe) {	// 是device_driver->probe，而不是platform_driver->probe，所以这里涉及device_driver和platform_driver的联系
			ret = drv->probe(dev);
		}
}
```
<!--more-->
# platform_driver和device_driver的联系
device_driver->probe也是platform_driver->probe
```c
int platform_driver_register(struct platform_driver *drv)
{
	drv->driver.bus = &platform_bus_type;
	// 这里将link platform_driver和device_driver
	if (drv->probe)	// platform_driver
		drv->driver.probe = platform_drv_probe;	// device_driver
		
	return driver_register(&drv->driver);
}
```

```c
static int platform_drv_probe(struct device *_dev)
{
	struct platform_driver *drv = to_platform_driver(_dev->driver);	// really_probe中设置dev->driver = drv;
	struct platform_device *dev = to_platform_device(_dev);

	return drv->probe(dev);	// platform_driver->probe
}
```
