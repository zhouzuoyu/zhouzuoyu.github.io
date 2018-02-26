---
title: Input子系统(3)_input_dev
date: 2018-01-30 13:48:21
categories:
- Linux
- Input
---

# 注册input device流程

一般驱动中只需要编写input device驱动

```c
s3c2410ts_probe
	// allocate
	input_dev = input_allocate_device();
	/*
	 * 设置input事件类型和范围来选择合适的input handler
	 * 注册为kbd device必须同时设置evbit和keybit
	 */
	ts.input->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_ABS);
	ts.input->keybit[BIT_WORD(BTN_TOUCH)] = BIT_MASK(BTN_TOUCH);
	input_set_abs_params(ts.input, ABS_X, 0, 0x3FF, 0, 0);
	input_set_abs_params(ts.input, ABS_Y, 0, 0x3FF, 0, 0);
	// register：register成功后会挂接input handler（例如evdev）
	ret = input_register_device(ts.input);
```
<!-- more -->
# 关键函数分析

## 设置属性

```c
input_set_abs_params(ts.input, ABS_X, 0, 0x3FF, 0, 0);
	input_alloc_absinfo(dev);
	absinfo = &dev->absinfo[axis];	// absinfo = input_dev->absinfo[ABS_X]
	absinfo->minimum = min;
	absinfo->maximum = max;
	__set_bit(EV_ABS, dev->evbit);
	__set_bit(axis, dev->absbit);
```

## 注册

>   挂接合适的input_handler：
>   　　和input handler的match函数是否能match上，能match上则挂接

```c
/*
 * register device，就是match input_handler_list，然后connect
 */
int input_register_device(struct input_dev *dev)
	list_for_each_entry(handler, &input_handler_list, node)	// FIXME: input_handler_list
		input_attach_handler(dev, handler);
			// 成功找到才进行connect
			input_match_device(handler, dev);
				if (!handler->match || handler->match(handler, dev))
					return id;
			handler->connect(handler, dev, id);
```

## 上报事件

>   调用所有能用的input_handler的event函数

```c
/*
 * 对input_dev上所有可用的handler都调用handler->events/handler->event进行数据上报
 */
input_report_abs(ts.input, ABS_X, ts.xp);	// code: ABS_X, value: ts.xp
	input_event(dev, EV_ABS, code, value);
		input_handle_event(dev, type, code, value);
			int disposition = input_get_disposition(dev, type, code, &value);	// type:EV_ABS, code:ABS_X
				disposition = input_handle_abs_event(dev, code, &value);	// INPUT_PASS_TO_HANDLERS
			// set
			v = &dev->vals[dev->num_vals++];
			v->type = type;
			v->code = code;
			v->value = value;
			dev->vals[dev->num_vals++] = input_value_sync;
			// report
			input_pass_values(dev, dev->vals, dev->num_vals);
				list_for_each_entry_rcu(handle, &dev->h_list, d_node)	// 遍历input_dev所有可用的handle
					if (handle->open)	// 只有上层open了，才能上传数据
						input_to_handler(handle, vals, count);
							struct input_handler *handler = handle->handler;	// 由handle找到handler
							/*
							 * handler有events则调用，没有则for调用event
							 */
							if (handler->events)
								handler->events(handle, vals, count);
							else if (handler->event)
								for (v = vals; v != vals + count; v++)
									handler->event(handle, v->type, v->code, v->value);
```
