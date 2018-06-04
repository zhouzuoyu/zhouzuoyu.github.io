---
title: power
date: 2018-06-04 19:39:00
categories:
- Linux
- intel
- crl
---

## register
```c
struct crl_power_seq_entity ds90ub940_power_items[] = {
	{
		.type = CRL_POWER_ETY_GPIO_FROM_PDATA,
		.val = 1,
		.undo_val = 1, // Keep xshutdown HIGH all the time. (normally is 0)
	},
};

struct crl_sensor_configuration ds90ub940_crl_configuration = {
	.power_items = ARRAY_SIZE(ds90ub940_power_items),
	.power_entities = ds90ub940_power_items,
};

static const struct crlmodule_sensors supported_sensors[] = {
	{ "DS90UB940-1", "ds90ub940", &ds90ub940_crl_configuration },
};
```
<!--more-->
## runtime_resume
```c
static const struct dev_pm_ops crlmodule_pm_ops = {
	.runtime_suspend = crlmodule_runtime_suspend,
	.runtime_resume = crlmodule_runtime_resume,
	.suspend	= crlmodule_suspend,
	.resume		= crlmodule_resume,
};

static struct i2c_driver crlmodule_i2c_driver = {
	.driver	= {
		.name = CRLMODULE_NAME,
		.pm = &crlmodule_pm_ops,
	},
	.probe	= crlmodule_probe,
	.remove	= crlmodule_remove,
	.id_table = crlmodule_id_table,
```

```c
static int crlmodule_runtime_resume(struct device *dev)
{
	struct i2c_client *client = to_i2c_client(dev);
	struct v4l2_subdev *sd = i2c_get_clientdata(client);
	struct crl_sensor *sensor = to_crlmodule_sensor(sd);

	return __crlmodule_powerup_sequence(sensor);
		for (idx = 0; idx < sensor->sensor_ds->power_items; idx++) {
			entity = &sensor->pwr_entity[idx];
			switch (entity->type) {
			case CRL_POWER_ETY_GPIO_FROM_PDATA:
				gpio_set_value(sensor->platform_data->xshutdown,
						entity->val);
				break;
			}
		}
}
```
