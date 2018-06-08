---
title: register platform device
date: 2018-06-07 15:22:00
categories:
- Linux
- platform
---
# add device
```c
static void __init mini2440_init(void)
{
	platform_add_devices(mini2440_devices, ARRAY_SIZE(mini2440_devices));
}
```
<!--more-->
```c
int platform_add_devices(struct platform_device **devs, int num)
{
	for (i = 0; i < num; i++)
		ret = platform_device_register(devs[i]);
			return platform_device_add(pdev);
				pdev->dev.bus = &platform_bus_type;	// FIXME
				// set name
				if (pdev->id != -1)	// 
					dev_set_name(&pdev->dev, "%s.%d", pdev->name,  pdev->id);
				else
					dev_set_name(&pdev->dev, "%s", pdev->name);

				ret = device_add(&pdev->dev);
					bus_probe_device(dev);
}
```

## bus_probe_device
增加一个device时会对platform bus上的所有driver，都执行__device_attach
```c
void bus_probe_device(struct device *dev)
{
	struct bus_type *bus = dev->bus;

	if (bus && bus->p->drivers_autoprobe)
		ret = device_attach(dev);
			ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
				// 遍历platform bus上的所有driver，都执行fn()
				klist_iter_init_node(&bus->p->klist_drivers, &i,
							start ? &start->p->knode_bus : NULL);
				while ((drv = next_driver(&i)) && !error)
					error = fn(drv, data);
}
```

### __device_attach
先match，match成功则probe
```c
static int __device_attach(struct device_driver *drv, void *data)
{
	driver_match_device(drv, dev);
		return drv->bus->match ? drv->bus->match(dev, drv) : 1;	// platform_match
			
	return driver_probe_device(drv, dev);
}
```
