---
title: complete_path例子
date: 2018-06-05 11:36:48
categories:
- Linux
- asoc
- dapm
---

# 导言
{% post_link Linux/asoc/dapm/complete-path complete-path %}可以知道：
1. snd_soc_dapm_dac/ snd_soc_dapm_aif_in/ snd_soc_dapm_input/
snd_soc_dapm_vmid/ snd_soc_dapm_mic/ snd_soc_dapm_line都属于input_ep
2. snd_soc_dapm_adc/ snd_soc_dapm_aif_out/ snd_soc_dapm_output/
snd_soc_dapm_hp/ snd_soc_dapm_spk/ snd_soc_dapm_line都属于output_ep
<!--more-->

# 例子
## input_ep
```c
#define SND_SOC_DAPM_INPUT(wname) \
{	.id = snd_soc_dapm_input, .name = wname, .kcontrol_news = NULL, \
	.num_kcontrols = 0, .reg = SND_SOC_NOPM }
```

## output_ep
```c
#define SND_SOC_DAPM_ADC(wname, stname, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, .reg = wreg, \
	.shift = wshift, .invert = winvert}
```

## widget
```c
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
	SND_SOC_DAPM_INPUT("LINPUT1"),
	SND_SOC_DAPM_ADC("Left ADC", "Capture", WM8960_POWER1, 3, 0),
};
```

## path
> "sink", "control", "source"

```c
static const struct snd_soc_dapm_route audio_paths[] = {
	{ "Left Input Mixer", NULL, "LINPUT1", },
	{ "Left ADC", NULL, "Left Input Mixer" },
};
```

所以：
LINPUT1 -> Left Input Mixer -> Left ADC就组成了一条complate path。
