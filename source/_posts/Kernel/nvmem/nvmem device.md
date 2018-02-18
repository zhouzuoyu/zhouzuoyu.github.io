---
title: nvmem(2)_device
date: 2018-02-4 12:00:00
categories:
- Kernel
- nvmem
---
# 功能
注册为一个nvmem device
主要是构建读写的回调函数

# 源码分析
## device
> Mach-mini2440.c (linux-4.13.1\arch\arm\mach-s3c24xx)

<!--more-->
静态注册i2c_client
```c
static struct i2c_board_info mini2440_i2c_devs[] __initdata = {
	{
		I2C_BOARD_INFO("24c08", 0x50),
		.platform_data = &at24c08,
	},
};
```

## driver
1. 注册
	```c
	static struct i2c_driver at24_driver = {
		.driver = {
			.name = "at24",
			.acpi_match_table = ACPI_PTR(at24_acpi_ids),
		},
		.probe = at24_probe,
		.remove = at24_remove,
		.id_table = at24_ids,
	};
	at24_init
		i2c_add_driver(&at24_driver);
	```

2. probe
	i2c_bus_type match成功后，will call probe
	```c
	at24_probe(struct i2c_client *client, const struct i2c_device_id *id)
		// 因为i2c_algorithm里做了定义I2C_FUNC_I2C，so不使用I2C_SMBUS
		use_smbus = 0;
		i2c_check_functionality(client->adapter, I2C_FUNC_I2C)
			return (func & i2c_get_functionality(adap)) == func;	// return 1
		at24->use_smbus = use_smbus;
	
		at24->read_func = at24_eeprom_read_i2c;
		at24->write_func = at24_eeprom_write_i2c;	
		// set nvmem
		at24->nvmem_config.name = dev_name(&client->dev);
		at24->nvmem_config.dev = &client->dev;
		at24->nvmem_config.read_only = !writable;
		at24->nvmem_config.root_only = true;
		// set callback func
		at24->nvmem_config.reg_read = at24_read;
		at24->nvmem_config.reg_write = at24_write;
		// 注册成一个nvmem设备
		at24->nvmem = nvmem_register(&at24->nvmem_config);
	```

3. callback func
	```c
	at24_read
		while (count) {
			at24->read_func(at24, buf, off, count);	// at24_eeprom_read_i2c
				i2c_transfer(client->adapter, msg, 2);
			buf += status;
			off += status;
			count -= status;
		}

	at24_write
		while (count) {
			at24->write_func(at24, buf, off, count);	// at24_eeprom_write_i2c
				i2c_transfer(client->adapter, &msg, 1);
			buf += status;
			off += status;
			count -= status;
		}

	i2c_transfer
		__i2c_transfer(adap, msgs, num);
			adap->algo->master_xfer(adap, msgs, num);	// callback i2c_adapter->algo: s3c24xx_i2c_xfer
	```
