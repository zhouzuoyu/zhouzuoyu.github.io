---
title: dev name
date: 2018-06-07 15:22:00
categories:
- Linux
- platform
---

```c
int platform_add_devices(struct platform_device **devs, int num)
{
	for (i = 0; i < num; i++)
		ret = platform_device_register(devs[i]);
			return platform_device_add(pdev);
				// set name
				if (pdev->id != -1)	// 
					dev_set_name(&pdev->dev, "%s.%d", pdev->name,  pdev->id);
				else
					dev_set_name(&pdev->dev, "%s", pdev->name);
}
```
<!--more-->
## case: id = -1
```c
struct platform_device s3c_device_ohci = {
	.name		  = "s3c2410-ohci",
	.id		  = -1,
};
```
则dev name为"s3c2410-ohci".

## case: id != -1
```c
static struct platform_device mini2440_led1 = {
	.name		= "s3c24xx_led",
	.id		= 1,
};
```
则dev name为"s3c24xx_led.1".
