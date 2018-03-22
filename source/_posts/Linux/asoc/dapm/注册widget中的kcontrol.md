---
title: 注册widget中的kcontrol
date: 2018-03-22 13:34:53
categories:
- Linux
- asoc
- dapm
---

# 导言
从overview中可以看到kcontrol的注册方式有两种，在此描述第二种方式。

# 分析
## 申明
```c
static const struct snd_kcontrol_new wm8960_lin_boost[] = {
	SOC_DAPM_SINGLE("LINPUT2 Switch", WM8960_LINPATH, 6, 1, 0),
	SOC_DAPM_SINGLE("LINPUT3 Switch", WM8960_LINPATH, 7, 1, 0),
	SOC_DAPM_SINGLE("LINPUT1 Switch", WM8960_LINPATH, 8, 1, 0),
};
```
<!--more-->
```c
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
SND_SOC_DAPM_MIXER("Left Boost Mixer", WM8960_POWER1, 5, 0,
		   wm8960_lin_boost, ARRAY_SIZE(wm8960_lin_boost)),
};
```

## 注册kcontrol
注册声卡时，会对所有注册上的widget都执行：
1. 设置power_check函数
2. 注册kcontrol
```c
static int soc_probe_dai_link(struct snd_soc_card *card, int num)
	ret = soc_post_component_init(card, codec, num, 0);
		snd_soc_dapm_new_widgets(&codec->dapm);
			list_for_each_entry(w, &dapm->card->widgets, list)
			{
				// 设置power_check函数
				w->power_check = dapm_generic_check_power;
				dapm_new_mixer(w);
					// 注册kcontrols
					for (i = 0; i < w->num_kcontrols; i++) {	// 注册多个kcontrl，例如本case，有3个kcontrol
						list_for_each_entry(path, &w->sources, list_sink) {	// 找到path
							/*
							 * set kcontrol name: widget name + kcontrol name
							 * 例如：Left Boost Mixer LINPUT1 Switch
							 */
							snprintf(path->long_name, name_len, "%s %s",
							 	w->name + prefix_len,
							 	w->kcontrol_news[i].name);
							path->kcontrol = snd_soc_cnew(&w->kcontrol_news[i],
											  wlist, path->long_name,
											  prefix);
							ret = snd_ctl_add(card, path->kcontrol);
								list_add_tail(&kcontrol->list, &card->controls);
						}
					}
			}
```

## power_check
```c
static int dapm_generic_check_power(struct snd_soc_dapm_widget *w)
	in = is_connected_input_ep(w);
	out = is_connected_output_ep(w);
	return out != 0 && in != 0;
```
