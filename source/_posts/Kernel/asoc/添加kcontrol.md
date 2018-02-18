---
title: 添加kcontrol
date: 2018-02-8 12:00:00
categories:
- Kernel
- asoc
---
# 源码
codec driver code
> sound\soc\codecs\wm8960.c

# 源码分析
## register
```c
static const struct snd_kcontrol_new wm8960_snd_controls[] = {
	SOC_DOUBLE_R_TLV("Capture Volume", WM8960_LINVOL, WM8960_RINVOL,
		 0, 63, 0, adc_tlv),
  	...
};
static int wm8960_probe(struct snd_soc_codec *codec)
	snd_soc_add_controls(codec, wm8960_snd_controls, ARRAY_SIZE(wm8960_snd_controls));
```
<!--more-->
## snd_soc_add_controls
将kcontrol添加到snd_card->controls上
```c
int snd_soc_add_controls(struct snd_soc_codec *codec,
	const struct snd_kcontrol_new *controls, int num_controls)
{
	// 添加所有kcontrol
	for (i = 0; i < num_controls; i++) {
		const struct snd_kcontrol_new *control = &controls[i];
		err = snd_ctl_add(card, snd_soc_cnew(control, codec,
						     control->name,
						     codec->name_prefix));
      		// 添加到card->controls链表中
      		list_add_tail(&kcontrol->list, &card->controls);
      		// 计算器
      		card->controls_count += kcontrol->count;
			kcontrol->id.numid = card->last_numid + 1;
			card->last_numid += kcontrol->count;
	}
}
```
