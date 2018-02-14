---
title: kcontrol fops
date: {{date}}
categories:
- Kernel
- asoc
---

# 源码
kcontrol框架
> sound\core\control.c

# 功能
__/dev/snd/controlCx__: kcontrol接口节点，供上层调用
![controlCx节点](kcontrol_fops/2018-02-12 14-58-26屏幕截图.png)

可以看到controlCx节点的主设备号是116，即116只对应一个fops
主设备号可以有256个次设备号，256个次设备号都对应一个fops实在是浪费
所以使用snd_minors[]起中转作用，达到路由的效果
__操作controlCx，看起来是调用snd_fops，实际上是调用snd_ctl_f_ops__
<!--more-->
# 分析
## snd_fops
主设备号116 -> snd_fops
```c
static int major = CONFIG_SND_MAJOR;	// 116
static const struct file_operations snd_fops =
{
	.owner =	THIS_MODULE,
	.open =		snd_open,
	.llseek =	noop_llseek,
};
static int __init alsa_sound_init(void)
	register_chrdev(major, "alsa", &snd_fops);
```
替换fops
```c
static int snd_open(struct inode *inode, struct file *file)
	// 转换fops
	mptr = snd_minors[minor];
	old_fops = file->f_op;
	file->f_op = fops_get(mptr->f_ops);
	// call open
	if (file->f_op->open) {
		err = file->f_op->open(inode, file);
	}
```

## snd_ctl_f_ops
实质就是注册一个snd_device
1. 挂接到card->devices链表
	```c
	int snd_card_create(int idx, const char *xid,
				struct module *module, int extra_size,
				struct snd_card **card_ret)
		snd_ctl_create(card);
			static struct snd_device_ops ops = {
				.dev_free = snd_ctl_dev_free,
				.dev_register =	snd_ctl_dev_register,
				.dev_disconnect = snd_ctl_dev_disconnect,
			};
			/*
			 * 增加control节点
			 * 挂接到card->devices链表上
			 */
			return snd_device_new(card, SNDRV_DEV_CONTROL, card, &ops);
				list_add(&dev->list, &card->devices);
		
	```
2. card->devices链表上所有的snd_device都执行dev->ops->dev_register
	```c
	int snd_card_register(struct snd_card *card)
	{
		snd_device_register_all(card);
			// 挂接到card->devices上的所有snd_device都执行dev->ops->dev_register
			list_for_each_entry(dev, &card->devices, list) {
				if (dev->state == SNDRV_DEV_BUILD && dev->ops->dev_register) {
					if ((err = dev->ops->dev_register(dev)) < 0)
						return err;
					dev->state = SNDRV_DEV_REGISTERED;
				}
			}
	}
	```
3. dev->ops->dev_register
	callback snd_ctl_dev_register
	```c
	static const struct file_operations snd_ctl_f_ops =
	{
		.owner =	THIS_MODULE,
		.read =		snd_ctl_read,
		.open =		snd_ctl_open,
		.release =	snd_ctl_release,
		.llseek =	no_llseek,
		.poll =		snd_ctl_poll,
		.unlocked_ioctl =	snd_ctl_ioctl,
		.compat_ioctl =	snd_ctl_ioctl_compat,
		.fasync =	snd_ctl_fasync,
	};
	static int snd_ctl_dev_register(struct snd_device *device)
		cardnum = card->number;
		sprintf(name, "controlC%i", cardnum);	// controlCx
		// search可用的次设备号，注册节点
		snd_register_device(SNDRV_DEVICE_TYPE_CONTROL, card, -1, &snd_ctl_f_ops, card, name)
			// 将snd_ctl_f_ops放入到snd_minors[minor]->fops中
			preg->f_ops = f_ops;
			minor = snd_kernel_minor(type, card, dev);
				case SNDRV_DEVICE_TYPE_CONTROL:
					minor = SNDRV_MINOR(card->number, type);
					break;
			snd_minors[minor] = preg;
			// 生成节点
			preg->dev = device_create(sound_class, device, MKDEV(major, minor),
							private_data, "%s", name);
	```
