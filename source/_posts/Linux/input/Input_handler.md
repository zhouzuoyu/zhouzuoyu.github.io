---
title: Input子系统(4)_input_handler
date: 2018-01-30 15:34:06
categories:
- Linux
- input
---

# evdev.c分析

## 功能描述

*   input_dev必然会使用到的input_handler
*   生成节点**/dev/input/eventx**
*   汇集input_dev的上报数据，供上层read使用
<!-- more -->
## 注册input_handler

```c
static struct input_handler evdev_handler = {
	// 没有实现match则match every input_device
	.event		= evdev_event,	// input_device调用input_report_xxx将会callback此类函数
	.events		= evdev_events,
	.connect	= evdev_connect,// 主要是生成/dev/input/eventx节点
	.id_table	= evdev_ids,	// match everything
};
static const struct file_operations evdev_fops = {
	.open		= evdev_open,
	.read		= evdev_read,
	.write		= evdev_write,
};
static int __init evdev_init(void)
	return input_register_handler(&evdev_handler);
```

## 关键函数分析

### input_handler->connect

>   **生成/dev/input/eventx节点**

```c
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
			 const struct input_device_id *id)
  	// 找到一个合适的次设备号
  	for (minor = 0; minor < EVDEV_MINORS; minor++)
		if (!evdev_table[minor])
			break;
	// 注册生成/dev/input/eventx节点
	dev_set_name(&evdev->dev, "event%d", minor);
	evdev->dev.devt = MKDEV(INPUT_MAJOR, EVDEV_MINOR_BASE + minor);
	evdev->dev.class = &input_class;
	evdev->dev.parent = &dev->dev;
	evdev->dev.release = evdev_free;
	error = device_add(&evdev->dev);
```

### input_handler->fops

>   eventx节点的fops，负责input_dev上报数据的处理

handle->open只在上层open /dev/eventx时才置位
既 **只有上层open后才能上报事件**

```c
static int evdev_open(struct inode *inode, struct file *file)
	evdev_attach_client(evdev, client);
		list_add_tail_rcu(&client->node, &evdev->client_list);
	evdev_open_device(evdev);
		input_open_device(&evdev->handle);
			handle->open++;	// FIXME
```



input_report_xxx将会callback
将event数据保存在client->buffer[]中

```c
static void evdev_events(struct input_handle *handle, const struct input_value *vals, unsigned int count)
	list_for_each_entry_rcu(client, &evdev->client_list, node)
		evdev_pass_values(client, vals, count, ev_time);
		for (v = vals; v != vals + count; v++) {
			event.type = v->type;
			event.code = v->code;
			event.value = v->value;
			__pass_event(client, &event);
				client->buffer[client->head++] = *event;
		}
```



上层调用read()将会callback
将client->buffer[]中的数据通过copy_to_user传递给用户空间

```c
static ssize_t evdev_read(struct file *file, char __user *buffer,
			  size_t count, loff_t *ppos)
	for (;;) {
		while (read + input_event_size() <= count && evdev_fetch_next_event(client, &event)) {
			// evdev_fetch_next_event
			*event = client->buffer[client->tail++];	// 得到client->buffer[]
					
			input_event_to_user(buffer + read, &event);
				copy_to_user(buffer, event, sizeof(struct input_event));
				
			read += input_event_size();
		}
	}
```

