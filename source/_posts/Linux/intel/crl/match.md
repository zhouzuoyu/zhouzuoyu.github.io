---
title: crl match
date: 2018-05-21 11:15:00
categories:
- Linux
- intel
- crl
---
# 匹配成功log
> $ dmesg
> [x.xxxxx] crlmodule 2-000E: crlmodule_populate_ds xxx selected
> 2-000E: [adapter bus id]-[i2c addr]

# 代码分析
1. 需要注册一个名字为CRLMODULE_NAME的i2c_client，将会callback crlmodule_probe
2. 添加到supported_sensors，和关键的CRL configuration file联系起来
<!--more-->
## 注册device
分两步
### Adding a sensor device to a Linux Device Tree
其实只要配置好相应的宏，例如打开CONFIG_INTEL_IPU4_DS90UB940基本就能注册上该设备。
```c
static struct crlmodule_platform_data ds90ub940_pdata = {
	.xshutdown = 1,
	.lanes = DS90UB940_LANES,
	.ext_clk = 24000000,
	.op_sys_clock = (uint64_t []){600000000},
	.module_name = "DS90UB940",	// FIXME
	.id_string = "0x55 0x42 0x39 0x34 0x30",
};

static struct intel_ipu4_isys_subdev_info ds90ub940_crl_sd = {
	.csi2 = &ds90ub940_csi2_cfg,
	.i2c = {
		.board_info = {
			I2C_BOARD_INFO(CRLMODULE_NAME, DS90UB940_I2C_ADDRESS),	// FIXME，将会调用crlmodule_probe
			.platform_data = &ds90ub940_pdata,
		},
		.i2c_adapter_id = 9,
	},
};

static struct intel_ipu4_isys_subdev_pdata pdata = {
	.subdevs = (struct intel_ipu4_isys_subdev_info *[]) {
#ifdef CONFIG_INTEL_IPU4_DS90UB940
		&ds90ub940_crl_sd,
#endif
	},
};
```

### Adding the new sensor configuration file to the CRL framework
```c
static const struct crlmodule_sensors supported_sensors[] = {
	{ "DS90UB940", "ds90ub940", &ds90ub940_crl_configuration },	// FIXME
};
```

## driver
```c
static struct i2c_driver crlmodule_i2c_driver = {
	.driver	= {
		.name = CRLMODULE_NAME,
		.pm = &crlmodule_pm_ops,
	},
	.probe	= crlmodule_probe,
	.remove	= crlmodule_remove,
	.id_table = crlmodule_id_table,
};

static int crlmodule_probe(struct i2c_client *client,
			   const struct i2c_device_id *devid)
{
	sensor->platform_data = client->dev.platform_data;
	ret = crlmodule_populate_ds(sensor, &client->dev);
		for (i = 0; i < ARRAY_SIZE(supported_sensors); i++) {
			/* Check the ACPI supported modules */
			if (!strcmp(dev_name(dev), supported_sensors[i].pname)) {
				sensor->sensor_ds = supported_sensors[i].ds;
				dev_info(dev, "%s %s selected\n",
					 __func__, supported_sensors[i].name);
				return 0;
			};

			/* Check the non ACPI modules */
			if (!strcmp(sensor->platform_data->module_name,
					supported_sensors[i].pname)) {
				sensor->sensor_ds = supported_sensors[i].ds;
				dev_info(dev, "%s %s selected\n",
					 __func__, supported_sensors[i].name);
				return 0;
			};
		}
}
```

## match有两种方式
There are two ways for APL Yocto Linux to add your sensor into a Linux Device Tree:
• Define the ACPI configuration
• Define the sensor in the IPU4 Platform data module.
# ACPI supported modules
> strcmp(dev_name(dev), supported_sensors[i].pname)

即client->dev和supported_sensors[i].pname的对比

# non ACPI modules
> strcmp(sensor->platform_data->module_name, supported_sensors[i].pname)

这个case使用这种方式，
即client->dev.platform_data->module_name(.module_name = "DS90UB940",)和
supported_sensors[i].pname({ "DS90UB940", "ds90ub940", &ds90ub940_crl_configuration },)的对比
