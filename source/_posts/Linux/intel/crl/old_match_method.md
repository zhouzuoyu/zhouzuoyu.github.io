---
title: CRL老的匹配方式
date: 2018-05-21 11:11:00
categories:
- Linux
- intel
- crl
---
这种方式可能是CRL老的匹配方式，通过读取id寄存器的方式来进行匹配，先记录下
# 配置
通过读取id寄存器对比进行match，这个case中是读取0xF0-0xF5，看读回来的值是不是和默认值一致
![ID](match.png)
<!--more-->
## 设置ID寄存器
```c
struct crl_sensor_detect_config ds90ub940_sensor_detect_regset[] = {
	/* UB940 */
	{
		.reg = { 0xF1, CRL_REG_LEN_08BIT, 0xFF },
		.width = 5,
	},
	{
		.reg = { 0xF2, CRL_REG_LEN_08BIT, 0xFF },
		.width = 5,
	},
	{
		.reg = { 0xF3, CRL_REG_LEN_08BIT, 0xFF },
		.width = 5,
	},
	{
		.reg = { 0xF4, CRL_REG_LEN_08BIT, 0xFF },
		.width = 5,
	},
	{
		.reg = { 0xF5, CRL_REG_LEN_08BIT, 0xFF },
		.width = 5,
	},
};

struct crl_sensor_configuration ds90ub940_crl_configuration = {
	.id_reg_items = ARRAY_SIZE(ds90ub940_sensor_detect_regset),
	.id_regs = ds90ub940_sensor_detect_regset,
};
```

## 设置ID寄存器默认值
```c
static struct crlmodule_platform_data ds90ub940_pdata = {
	.xshutdown = 1,
	.lanes = DS90UB940_LANES,
	.ext_clk = 24000000,
	.op_sys_clock = (uint64_t []){600000000},
	.module_name = "DS90UB940",
	.id_string = "0x55 0x42 0x39 0x34 0x30",
};
```

# match代码分析
```c
static int crlmodule_registered(struct v4l2_subdev *subdev)
{
	rval = crlmodule_identify_module(subdev);
		for (i = 0; i < sensor->sensor_ds->id_reg_items; i++)
			size += sensor->sensor_ds->id_regs[i].width + 1;	// 30
		
		// ID寄存器默认值
		expect_id = sensor->platform_data->id_string;	// "0x55 0x42 0x39 0x34 0x30"
		
		// 读取ds90ub940_sensor_detect_regset配置的id寄存器
		for (i = 0; i < sensor->sensor_ds->id_reg_items; i++) {
			// 通过i2c来进行读取
			ret = crlmodule_read_reg(sensor,
						 sensor->sensor_ds->id_regs[i].reg,
						 &val);
					crlmodule_i2c_read(sensor, reg.dev_i2c_addr, reg.address,
						  reg.len, val);

			if (i)
				pos += snprintf(id_string + pos, size - pos, " 0x%x", val);	// 0x55 0x42 ...
			else
				pos = snprintf(id_string, size, "0x%x", val);	// 第一次是0x55
		}
		
		// match
		if (expect_id &&
		   (strnlen(id_string, size) != strnlen(expect_id, size + 1) ||
			strncmp(id_string, expect_id, size))) {
			dev_err(&client->dev,
				"Sensor detection failed: expect \"%s\" actual \"%s\"",
				expect_id, id_string);
			ret = -ENODEV;
		}
}
```
