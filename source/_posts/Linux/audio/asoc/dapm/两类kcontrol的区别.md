---
title: 两类kcontrol的区别
date: 2018-03-23 14:32:09
categories:
- Linux
- asoc
- dapm
---

两种kcontrol
```c
static const struct snd_kcontrol_new wm8960_snd_controls[] = {
	SOC_SINGLE("Speaker DC Volume", WM8960_CLASSD3, 3, 5, 0),
};
```

```c
static const struct snd_kcontrol_new wm8960_lin_boost[] = {
	SOC_DAPM_SINGLE("LINPUT2 Switch", WM8960_LINPATH, 6, 1, 0),
};
```
<!--more-->
观察两个宏
```c
#define SOC_SINGLE(xname, reg, shift, max, invert) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
	.put = snd_soc_put_volsw, \
	.private_value =  SOC_SINGLE_VALUE(reg, shift, max, invert) }

```

```c
#define SOC_DAPM_SINGLE(xname, reg, shift, max, invert) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.info = snd_soc_info_volsw, \
	.get = snd_soc_dapm_get_volsw, .put = snd_soc_dapm_put_volsw, \
	.private_value =  SOC_SINGLE_VALUE(reg, shift, max, invert) }

```

区别只在于get和put方法上
## snd_soc_put_volsw
直接写
```c
int snd_soc_put_volsw(struct snd_kcontrol *kcontrol,
	struct snd_ctl_elem_value *ucontrol)
	// 直接写
	val = (ucontrol->value.integer.value[0] & mask);
	return snd_soc_update_bits_locked(codec, reg, val_mask, val);
		snd_soc_update_bits(codec, reg, mask, value);
			snd_soc_read(codec, reg);
			snd_soc_write(codec, reg, new);
```

## snd_soc_dapm_put_volsw
调用dapm_power_widgets，触发一系列dapm方面流程
```c
int snd_soc_dapm_put_volsw(struct snd_kcontrol *kcontrol,
	struct snd_ctl_elem_value *ucontrol)
	
	// 读取reg，看需要设置的值和读取到的值是否一致，不一致才需要update
	change = snd_soc_test_bits(widget->codec, reg, mask, val);
		old = snd_soc_read(codec, reg);
		new = (old & ~mask) | value;
		change = old != new;
		return change;
	if (change) {
		for (wi = 0; wi < wlist->num_widgets; wi++) {
			widget = wlist->widgets[wi];

			widget->value = val;

			// 将reg封装到update中
			update.kcontrol = kcontrol;
			update.widget = widget;
			update.reg = reg;
			update.mask = mask;
			update.val = val;
			widget->dapm->update = &update;

			dapm_mixer_update_power(widget, kcontrol, connect);
				dapm_power_widgets(widget->dapm, SND_SOC_DAPM_STREAM_NOP);
					dapm_widget_update(dapm);
						snd_soc_update_bits(w->codec, update->reg, update->mask,
					  			update->val);	// update
		}
	}
```
