---
title: match
date: 2018-06-07 15:22:00
categories:
- Linux
- platform
---
# match
可以看到match有三种方式
```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```
<!--more-->
## 方式1：of_driver_match_device
dts中的node.xxx字段和driver.of_match_table.xxx字段进行match
```c
static inline int of_driver_match_device(struct device *dev,
					 const struct device_driver *drv)
{
	return of_match_device(drv->of_match_table, dev) != NULL;
		return of_match_node(matches, dev->of_node);
			while (matches->name[0] || matches->type[0] || matches->compatible[0]) {
				int match = 1;
				if (matches->name[0])
					match &= node->name
						&& !strcmp(matches->name, node->name);
				if (matches->type[0])
					match &= node->type
						&& !strcmp(matches->type, node->type);
				if (matches->compatible[0])
					match &= of_device_is_compatible(node,
								matches->compatible);
				if (match)
					return matches;
				matches++;
			}
}
```

xxx可以是name/type/compatible，但一般compatible居多，分析compatible
```c
#define of_compat_cmp(s1, s2, l)	strcasecmp((s1), (s2))	// 最终还是strcmp

int of_device_is_compatible(const struct device_node *device,
		const char *compat)
{
	const char* cp;
	int cplen, l;

	cp = of_get_property(device, "compatible", &cplen);
	while (cplen > 0) {
		if (of_compat_cmp(cp, compat, strlen(compat)) == 0)
			return 1;
		l = strlen(cp) + 1;
		cp += l;
		cplen -= l;
	}

	return 0;
}
```

### 例子
device：
dts中设置
```c
DMA0: dma0@400100100 {
	compatible = "ibm,dma-440spe";
}; 
```

driver：
```c
static const struct of_device_id ppc440spe_adma_of_match[] __devinitconst = {
	{ .compatible	= "ibm,dma-440spe", },
	{},
};

static struct platform_driver ppc440spe_adma_driver = {
	.driver = {
		.of_match_table = ppc440spe_adma_of_match,
	},
};
```

## 方式2：platform_match_id
platform_device.name和id_table.name进行match
```c
struct platform_driver {
	const struct platform_device_id *id_table;
};
```

```c
static const struct platform_device_id *platform_match_id(
			const struct platform_device_id *id,
			struct platform_device *pdev)
{
	while (id->name[0]) {
		if (strcmp(pdev->name, id->name) == 0) {
			pdev->id_entry = id;
			return id;
		}
		id++;
	}
	return NULL;
}
```

## 方式3
以前最常用的方式，即对比platform_device.name和platform_driver.driver.name
```c
static struct platform_driver acc_con_driver = {
	.driver		= {
		.name		= "acc_con",
		.owner		= THIS_MODULE,
	},
};
```

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	return (strcmp(pdev->name, drv->name) == 0);
}
```

