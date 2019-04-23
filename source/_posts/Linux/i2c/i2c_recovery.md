---
title: i2c总线死锁
date: 2019-4-23 10:23:00
categories:
- Linux
- i2c
---

# 现象
SCL一直拉高，SDA一直拉低，总线上的所有设备都无法访问

# 原因
当master正在和slave通信，如果master正好发生打算发第9个时钟，此时SCL为高，slave设备是先拉低SDA信号（作为ACK信号），等待master设备SCL由高变低，“取走”ACK信号后，slave再释放SDA为高。
但如果此时时序被打乱，例如master i2c通信时突然复位，master的SCL还没来得及变低，直接变成高电平，此时slave还在等待SCL变低，所以一直拉低SDA。
slave：等待SCL拉低才会释放SDA
master：SDA被slave拉低，故master认为i2c总线占用，一直等待SDA变高
这样master/slave进入一个相互等待的死锁过程。
<!--more-->
# 解决方案
在主机中增加I2C总线恢复程序。
每次主机复位后，如果检测到SDA一段时间内被拉低，则控制SCL产生<=9个时钟脉冲，每发送一个时钟脉冲就检测SDA是否被释放，如果SDA已经被释放就再模拟产生一个停止信号，这样从机就可以完成被挂起的读写操作，从死锁状态中恢复过来。

# 源码分析
## 注册
```c
static int i2c_imx_probe(struct platform_device *pdev)
{
	i2c_imx_init_recovery_info(i2c_imx, pdev);
		i2c_imx->pinctrl_pins_default = pinctrl_lookup_state(i2c_imx->pinctrl, PINCTRL_STATE_DEFAULT);
		i2c_imx->pinctrl_pins_gpio = pinctrl_lookup_state(i2c_imx->pinctrl, "gpio");
		rinfo->sda_gpio = of_get_named_gpio_flags(pdev->dev.of_node, "sda-gpios", 0, NULL);
		rinfo->scl_gpio = of_get_named_gpio_flags(pdev->dev.of_node, "scl-gpios", 0, NULL);
		rinfo->prepare_recovery = i2c_imx_prepare_recovery;
		rinfo->unprepare_recovery = i2c_imx_unprepare_recovery;
		rinfo->recover_bus = i2c_generic_gpio_recovery;
		i2c_imx->adapter.bus_recovery_info = rinfo;
}
```

## 调用
```c
static int i2c_imx_xfer(struct i2c_adapter *adapter,
						struct i2c_msg *msgs, int num)
{
	/* Start I2C transfer */
	result = i2c_imx_start(i2c_imx);
	if (result) {
		i2c_recover_bus(&i2c_imx->adapter);
				return adap->bus_recovery_info->recover_bus(adap);	// 	i2c_generic_gpio_recovery
					// 设置scl为输出，sda为输入
					ret = i2c_get_gpios_for_recovery(adap);
						ret = gpio_request_one(bri->scl_gpio, GPIOF_OPEN_DRAIN |
								GPIOF_OUT_INIT_HIGH, "i2c-scl");
						if (bri->get_sda) {
							if (gpio_request_one(bri->sda_gpio, GPIOF_IN, "i2c-sda")) {
								/* work without SDA polling */
								dev_warn(dev, "Can't get SDA gpio: %d. Not using SDA polling\n",
										bri->sda_gpio);
								bri->get_sda = NULL;
							}
						}
					// 核心代码
					ret = i2c_generic_recovery(adap);
					// free
					i2c_put_gpios_for_recovery(adap);
						gpio_free(bri->sda_gpio);
						gpio_free(bri->scl_gpio);

		result = i2c_imx_start(i2c_imx);
	}
}
```

```c
static int i2c_generic_recovery(struct i2c_adapter *adap)
{
	// pin设置为gpio模式
	bri->prepare_recovery(adap);
		pinctrl_select_state(i2c_imx->pinctrl, i2c_imx->pinctrl_pins_gpio);

	// 连发9个scl时钟，将slave从上一次的ack中释放出来
	while (i++ < RECOVERY_CLK_CNT * 2) {
		if (val) {
			/* Break if SDA is high */
			if (bri->get_sda && bri->get_sda(adap))
					break;
			/* SCL shouldn't be low here */
			if (!bri->get_scl(adap)) {
				dev_err(&adap->dev,
					"SCL is stuck low, exit recovery\n");
				ret = -EBUSY;
				break;
			}
		}

		// 发时钟
		val = !val;
		bri->set_scl(adap, val);
		ndelay(RECOVERY_NDELAY);
	}

	// pin设置为i2c模式
	bri->unprepare_recovery(adap);
		pinctrl_select_state(i2c_imx->pinctrl, i2c_imx->pinctrl_pins_default);
}
```
