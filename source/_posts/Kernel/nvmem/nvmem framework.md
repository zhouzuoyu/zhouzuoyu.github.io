---
title: nvmem(1)_framework
date: 2018-02-4 12:00:00
categories:
- Kernel
- nvmem
---
# 源码
> Core.c (linux-4.13.1\drivers\nvmem)

# 功能
nvmem驱动框架的作用主要是：
生成节点文件，构建读写框架(最终回调nvmem device的读写函数)
<!--more-->
# 源码分析
生成/sys/bus/nvmem/devices/*/nvmem节点文件
```c
nvmem_register
	nvmem->dev.bus = &nvmem_bus_type;
	// 传递回调函数
	nvmem->reg_read = config->reg_read;
	nvmem->reg_write = config->reg_write;
	// 生成/sys/bus/nvmem/devices/*/nvmem节点
	// nvmem->dev->name: [config->name][config->id]，如/sys/bus/nvmem/devices/qfprom0/nvmem
	dev_set_name(&nvmem->dev, "%s%d", config->name ? : "nvmem", config->name ? config->id : nvmem->id);
	nvmem->dev.groups = nvmem->read_only ? nvmem_ro_root_dev_groups : nvmem_rw_root_dev_groups;	// nvmem_ro_root_dev_groups | nvmem_rw_root_dev_groups
	device_add(&nvmem->dev);
```

节点的读写函数，比如rw
```c
static struct bin_attribute bin_attr_rw_root_nvmem = {
	.attr	= {
		.name	= "nvmem",
		.mode	= S_IWUSR | S_IRUSR,
	},
	.read	= bin_attr_nvmem_read,
	.write	= bin_attr_nvmem_write,
};
bin_attr_nvmem_read
	nvmem_reg_read(nvmem, pos, buf, count);
		nvmem->reg_read(nvmem->priv, offset, val, bytes);
bin_attr_nvmem_write
	nvmem_reg_write(nvmem, pos, buf, count);
		nvmem->reg_write(nvmem->priv, offset, val, bytes);
```
