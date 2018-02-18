---
title: kcontrol流程
date: 2018-02-8 12:00:00
categories:
- Kernel
- asoc
---

# 源码
> external\tinyalsa\tinymix.c
> sound\soc\codecs\wm8960.c

# 功能
描述tinymix写入的数据如何传递给kernel设置
只要上层控制kcontrol，其流程都如此
<!--more-->
# 源码分析
## tinymix
> 下ioctl(SNDRV_CTL_IOCTL_ELEM_WRITE)

```c
int main(int argc, char **argv)
	tinymix_set_value(mixer, argv[1], &argv[2], argc - 2);
		mixer_ctl_set_value(ctl, i, value);
			case SNDRV_CTL_ELEM_TYPE_INTEGER:
				ev.value.integer.value[id] = value;	// 传入数据
				break;
			return ioctl(ctl->mixer->fd, SNDRV_CTL_IOCTL_ELEM_WRITE, &ev);
```

## kernel
在{% post_link Kernel/asoc/kcontrol_fops kcontrol_fops %}已经阐述，最终调用的fops是snd_ctl_f_ops
最后执行kctl->put
```c
static long snd_ctl_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
	case SNDRV_CTL_IOCTL_ELEM_WRITE:
		return snd_ctl_elem_write_user(ctl, argp);
			control = memdup_user(_control, sizeof(*control));	// 得到数据
			snd_ctl_elem_write(card, file, control);
				// name/id来找到对应kcontrol
				kctl = snd_ctl_find_id(card, &control->id);
					if (id->numid != 0)
						return snd_ctl_find_numid(card, id->numid);
						
					list_for_each_entry(kctl, &card->controls, list) {
						if (kctl->id.iface != id->iface)
							continue;
						if (kctl->id.device != id->device)
							continue;
						if (kctl->id.subdevice != id->subdevice)
							continue;
						if (strncmp(kctl->id.name, id->name, sizeof(kctl->id.name)))
							continue;
						if (kctl->id.index > id->index)
							continue;
						if (kctl->id.index + kctl->count <= id->index)
							continue;
						return kctl;
					}
				// exec
				kctl->put(kctl, control);
```

### kctl->put
> 最终使用 {% post_link Kernel/asoc/snd_soc_write snd_soc_write %}写寄存器

1. description
	```c
	static const struct snd_kcontrol_new wm8960_snd_controls[] = {
	SOC_DOUBLE_R_TLV("Capture Volume", WM8960_LINVOL, WM8960_RINVOL,
			 0, 63, 0, adc_tlv),
	};

	```
	```c
	#define SOC_DOUBLE_R_TLV(xname, reg_left, reg_right, xshift, xmax, xinvert, tlv_array) \
	{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = (xname),\
		.access = SNDRV_CTL_ELEM_ACCESS_TLV_READ |\
			 SNDRV_CTL_ELEM_ACCESS_READWRITE,\
		.tlv.p = (tlv_array), \
		.info = snd_soc_info_volsw_2r, \
		.get = snd_soc_get_volsw_2r, .put = snd_soc_put_volsw_2r, \
		.private_value = (unsigned long)&(struct soc_mixer_control) \
			{.reg = reg_left, .rreg = reg_right, .shift = xshift, \
			.max = xmax, .platform_max = xmax, .invert = xinvert} }

	```
2. put回调函数
	```c
	int snd_soc_put_volsw_2r(struct snd_kcontrol *kcontrol,
		struct snd_ctl_elem_value *ucontrol)
	{
		// 解析数据
		val = (ucontrol->value.integer.value[0] & mask);
		val2 = (ucontrol->value.integer.value[1] & mask);
		val = val << shift;
		val2 = val2 << shift;
		// 设置数据
		snd_soc_update_bits_locked(codec, reg, val_mask, val);
		snd_soc_update_bits_locked(codec, reg2, val_mask, val2);
			snd_soc_update_bits(codec, reg, mask, value);
				snd_soc_read(codec, reg);
					codec->read(codec, reg);
				new = (old & ~mask) | (value & mask);
				snd_soc_write(codec, reg, new);
					codec->write(codec, reg, val);
	}
	```
