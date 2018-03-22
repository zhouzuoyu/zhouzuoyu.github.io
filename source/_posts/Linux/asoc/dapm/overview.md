---
title: dapm overview
date: 2018-03-22 13:46:56
categories:
- Linux
- asoc
- dapm
---

# 导言
各类widget和snd_kcontrol_new一同组成route，route构成path，path组成一条complete path，只有能connect到input_ep和ouput_ep上的所有widget才会被上电。
<!--more-->

# 各类节点
## kcontrol
有两种注册方式
1. snd_soc_add_controls注册
	```c
	static const struct snd_kcontrol_new wm8960_snd_controls[] = {
	SOC_DOUBLE_R_TLV("Capture Volume", WM8960_LINVOL, WM8960_RINVOL,
			 0, 63, 0, adc_tlv),
	};
	```
	```c
	snd_soc_add_controls(codec, wm8960_snd_controls,
						 ARRAY_SIZE(wm8960_snd_controls));
	```

2. 注册到widget中
	```c
	static const struct snd_kcontrol_new wm8960_lin_boost[] = {
		SOC_DAPM_SINGLE("LINPUT2 Switch", WM8960_LINPATH, 6, 1, 0),
		SOC_DAPM_SINGLE("LINPUT3 Switch", WM8960_LINPATH, 7, 1, 0),
		SOC_DAPM_SINGLE("LINPUT1 Switch", WM8960_LINPATH, 8, 1, 0),
	};
	```
	```c
	static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
	SND_SOC_DAPM_MIXER("Left Boost Mixer", WM8960_POWER1, 5, 0,
			   wm8960_lin_boost, ARRAY_SIZE(wm8960_lin_boost)),
	};
	```

## widget
dapm中的最小unit
```c
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
	SND_SOC_DAPM_INPUT("LINPUT1"),
	...
};
```

## route
描述各种widget，kcontrol之间的关系
```c
static const struct snd_soc_dapm_route audio_paths[] = {
	{ "Left Boost Mixer", "LINPUT1 Switch", "LINPUT1" },
	...
};
```
