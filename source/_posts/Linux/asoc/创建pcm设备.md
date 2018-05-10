---
title: 注册pcm设备
date: 2018-05-10 16:03:00
categories:
- Linux
- asoc
---

# 注册pcm设备
先添加到链表card->devices，后续再统一callback ops.dev_register
```c
static void snd_soc_instantiate_card(struct snd_soc_card *card)
	ret = soc_probe_dai_link(card, i);
		ret = soc_new_pcm(rtd, num);
			ret = snd_pcm_new(rtd->card->snd_card, new_name,
					num, playback, capture, &pcm);
				static struct snd_device_ops ops = {
					.dev_free = snd_pcm_dev_free,
					.dev_register =	snd_pcm_dev_register,
					.dev_disconnect = snd_pcm_dev_disconnect,
				};
				snd_device_new(card, SNDRV_DEV_PCM, pcm, &ops);
					list_add(&dev->list, &card->devices);
```
<!--more-->
统一callback
```c
int snd_card_register(struct snd_card *card)
	snd_device_register_all(card);
		list_for_each_entry(dev, &card->devices, list) {
			if (dev->state == SNDRV_DEV_BUILD && dev->ops->dev_register) {
				if ((err = dev->ops->dev_register(dev)) < 0)
					return err;
				dev->state = SNDRV_DEV_REGISTERED;
			}
		}
```
注册函数：注册pcm设备，需要关注snd_minors\[minor\](用于route fops用)
```c
static int snd_pcm_dev_register(struct snd_device *device)
{
	for (cidx = 0; cidx < 2; cidx++) {
		switch (cidx) {
		case SNDRV_PCM_STREAM_PLAYBACK:
			sprintf(str, "pcmC%iD%ip", pcm->card->number, pcm->device);
			devtype = SNDRV_DEVICE_TYPE_PCM_PLAYBACK;
			break;
		case SNDRV_PCM_STREAM_CAPTURE:
			sprintf(str, "pcmC%iD%ic", pcm->card->number, pcm->device);
			devtype = SNDRV_DEVICE_TYPE_PCM_CAPTURE;
			break;
		}
		snd_register_device_for_dev(devtype, pcm->card,
						  pcm->device,
						  &snd_pcm_f_ops[cidx],
						  pcm, str, dev);
			preg->f_ops = f_ops;
			minor = snd_kernel_minor(type, card, dev);
			snd_minors[minor] = preg;	// FIXME：route fops用
			preg->dev = device_create(sound_class, device, MKDEV(major, minor),
							private_data, "%s", name);
	}
}
```

# route fops
通过minor次设备号来找到对应的fops，机理和misc_dev类似
1. 
	所有的audio设备主设备号都是116
	> crw-rw---T+  1 root audio 116,  6  5月  9 19:07 controlC0
	> crw-rw---T+  1 root audio 116,  4  5月  9 19:08 pcmC0D0c
	> crw-rw---T+  1 root audio 116,  3  5月 10 15:26 pcmC0D0p
	
	使用register_chrdev注册设备，so访问任意的音频节点都是走同一个fops
	```c
	#define CONFIG_SND_MAJOR	116
	static int major = CONFIG_SND_MAJOR;
	static const struct file_operations snd_fops =
	{
		.owner =	THIS_MODULE,
		.open =		snd_open,
		.llseek =	noop_llseek,
	};
	static int __init alsa_sound_init(void)
		register_chrdev(major, "alsa", &snd_fops);
	```

2. 
	通过次设备号进行route
	```c
	static int snd_open(struct inode *inode, struct file *file)
		unsigned int minor = iminor(inode);
		mptr = snd_minors[minor];	// snd_pcm_dev_register中设置了，这里使用
	
		file->f_op = fops_get(mptr->f_ops);	// 变更了fops
		file->f_op->open(inode, file);	// 调用次设备号对应fops的open
	```
