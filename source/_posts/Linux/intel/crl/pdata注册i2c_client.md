---
title: pdata注册i2c_client
date: 2018-06-04 10:17:00
categories:
- Linux
- intel
- crl
---
## pdata
```c
static struct intel_ipu4_isys_subdev_info ds90ub940_crl_sd_1 = {
	.csi2 = &ds90ub940_csi2_cfg_1,
	.i2c = {
		.board_info = {
			I2C_BOARD_INFO(CRLMODULE_NAME, 0x2c),
			.platform_data = &ds90ub940_pdata_1,
		},
		.i2c_adapter_id = 5,	// FIXME
	},
};
```
<!--more-->

## register i2c_client
```c
static int isys_register_ext_subdev(struct intel_ipu4_isys *isys,
				struct intel_ipu4_isys_subdev_info *sd_info,
				bool acpi_only)
{
	struct i2c_adapter *adapter = i2c_get_adapter(sd_info->i2c.i2c_adapter_id);	// 5

	client = isys_find_i2c_subdev(adapter, sd_info);	// 默认client为NULL
	if (!client) {
		sd = v4l2_i2c_new_subdev_board(&isys->v4l2_dev, adapter,
					       &sd_info->i2c.board_info, 0);
			client = i2c_new_device(adapter, info);
	}
}
```
