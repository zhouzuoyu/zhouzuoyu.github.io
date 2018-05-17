---
title: tiny4412_tp
date: 2018-05-16 18:15:00
categories:
- Linux
- input
---

# 多点触摸协议
多点触摸报点，需要知道当前点属于之前的哪一条线
硬件支持识别，则设置**ABS_MT_TRACKING_ID**属性。
硬件不支持，则framework层会计算前一次5个点与本次5个点的距离来进行推算
参考：[A/B(Slot)协议](https://blog.csdn.net/fantasyhujian/article/details/12192761?utm_source=tuicool&utm_medium=referral)
<!--more-->
# 代码分析
## 注册
```c
static int ft5x0x_ts_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
	/*
	 * part 1：input device
	 */
	input_dev = input_allocate_device();
	
	set_bit(EV_SYN, input_dev->evbit);
	set_bit(EV_ABS, input_dev->evbit);
	set_bit(EV_KEY, input_dev->evbit);
	
	// 多点触摸：B协议
	set_bit(ABS_MT_TRACKING_ID, input_dev->absbit);
	set_bit(ABS_MT_TOUCH_MAJOR, input_dev->absbit);
	set_bit(ABS_MT_WIDTH_MAJOR, input_dev->absbit);
	set_bit(ABS_MT_POSITION_X, input_dev->absbit);
	set_bit(ABS_MT_POSITION_Y, input_dev->absbit);

	input_set_abs_params(input_dev, ABS_MT_POSITION_X, 0, ts->screen_max_x, 0, 0);
	input_set_abs_params(input_dev, ABS_MT_POSITION_Y, 0, ts->screen_max_y, 0, 0);
	input_set_abs_params(input_dev, ABS_MT_TOUCH_MAJOR, 0, ts->pressure_max, 0, 0);
	input_set_abs_params(input_dev, ABS_MT_WIDTH_MAJOR, 0, 200, 0, 0);
	input_set_abs_params(input_dev, ABS_MT_TRACKING_ID, 0, FT5X0X_PT_MAX, 0, 0);	// 最多支持5点触摸
	
	input_register_device(input_dev);
	
	/*
	 part 2：irq
	 */
	INIT_WORK(&ts->work, ft5x0x_ts_pen_irq_work);
	request_irq(client->irq, ft5x0x_ts_interrupt,
		IRQ_TYPE_EDGE_FALLING /*IRQF_TRIGGER_FALLING*/, "ft5x0x_ts", ts);
}
```

## ISR中启动queue_work
```c
static irqreturn_t ft5x0x_ts_interrupt(int irq, void *dev_id) {
	queue_work(ts->queue, &ts->work);	// 启动工作队列
}
```

```c
static void ft5x0x_ts_pen_irq_work(struct work_struct *work) {
	// 先读取再上报
	if (!ft5x0x_read_data(ts)) {
		ft5x0x_ts_report(ts);
	}
}
```

## 读取和上报坐标
读取数据，多点的话则一起收集
```c
static int ft5x0x_read_data(struct ft5x0x_ts_data *ts) {
	ft5x0x_i2c_rxdata(buf, 31);
	event->touch_point = buf[2] & 0x07;
	switch (event->touch_point) {
		case 5:
			event->x[4] = (s16)(buf[0x1b] & 0x0F)<<8 | (s16)buf[0x1c];
			event->y[4] = (s16)(buf[0x1d] & 0x0F)<<8 | (s16)buf[0x1e];
		case 4:
			event->x[3] = (s16)(buf[0x15] & 0x0F)<<8 | (s16)buf[0x16];
			event->y[3] = (s16)(buf[0x17] & 0x0F)<<8 | (s16)buf[0x18];
		case 3:
			event->x[2] = (s16)(buf[0x0f] & 0x0F)<<8 | (s16)buf[0x10];
			event->y[2] = (s16)(buf[0x11] & 0x0F)<<8 | (s16)buf[0x12];
		case 2:
			event->x[1] = (s16)(buf[0x09] & 0x0F)<<8 | (s16)buf[0x0a];
			event->y[1] = (s16)(buf[0x0b] & 0x0F)<<8 | (s16)buf[0x0c];
		case 1:
			event->x[0] = (s16)(buf[0x03] & 0x0F)<<8 | (s16)buf[0x04];
			event->y[0] = (s16)(buf[0x05] & 0x0F)<<8 | (s16)buf[0x06];
			break;
	}
}
```

上报数据
```c
static void ft5x0x_ts_report(struct ft5x0x_ts_data *ts) {
	for (i = 0; i < event->touch_point; i++) {
		x = event->x[i];
		y = event->y[i];

		input_report_abs(ts->input_dev, ABS_MT_POSITION_X, x);
		input_report_abs(ts->input_dev, ABS_MT_POSITION_Y, y);

		input_report_abs(ts->input_dev, ABS_MT_PRESSURE, event->pressure);
		input_report_abs(ts->input_dev, ABS_MT_TOUCH_MAJOR, event->pressure);
		input_report_abs(ts->input_dev, ABS_MT_TRACKING_ID, i);	// 追踪id

		input_mt_sync(ts->input_dev);
	}
	input_report_key(ts->input_dev, BTN_TOUCH, 1);
	input_sync(ts->input_dev);
}
```
