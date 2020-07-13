---
title: dai widget/stream widget
date: 2020-07-13 15:05:47
categories:
- Linux
- asoc
- dapm
---

widget中可以定义一些stream widget，
在snd_soc_dai_driver中会自动生成dai widget，
如果两者的name一致的话，则会生成一条path。
例如：
```c
	{ "Left Input Mixer", NULL, "LINPUT1", },
	{ "Left ADC", NULL, "Left Input Mixer" },
```
"Left ADC".id = snd_soc_dapm_adc，它并不是一个output endpoint，不能形成complete path，所以必须要dai widget配合才能形成complete path。
<!--more-->
# stream widget
|  stream widget |
|  ----  |
| snd_soc_dapm_aif_in |
| snd_soc_dapm_aif_out |
| snd_soc_dapm_dac |
| snd_soc_dapm_adc |
```c
// 用这些宏定义的是stream_widget
/* stream domain */
#define SND_SOC_DAPM_AIF_IN(wname, stname, wslot, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_aif_in, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_AIF_IN_E(wname, stname, wslot, wreg, wshift, winvert, \
			      wevent, wflags)				\
{	.id = snd_soc_dapm_aif_in, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags }
#define SND_SOC_DAPM_AIF_OUT(wname, stname, wslot, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_aif_out, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_AIF_OUT_E(wname, stname, wslot, wreg, wshift, winvert, \
			     wevent, wflags)				\
{	.id = snd_soc_dapm_aif_out, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags }
#define SND_SOC_DAPM_DAC(wname, stname, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_dac, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert) }
#define SND_SOC_DAPM_DAC_E(wname, stname, wreg, wshift, winvert, \
			   wevent, wflags)				\
{	.id = snd_soc_dapm_dac, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags}

#define SND_SOC_DAPM_ADC(wname, stname, wreg, wshift, winvert) \
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
#define SND_SOC_DAPM_ADC_E(wname, stname, wreg, wshift, winvert, \
			   wevent, wflags)				\
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, \
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), \
	.event = wevent, .event_flags = wflags}
#define SND_SOC_DAPM_CLOCK_SUPPLY(wname) \
{	.id = snd_soc_dapm_clock_supply, .name = wname, \
	.reg = SND_SOC_NOPM, .event = dapm_clock_event, \
	.event_flags = SND_SOC_DAPM_PRE_PMU | SND_SOC_DAPM_POST_PMD }
```

例如：
```c
SND_SOC_DAPM_ADC("Left ADC", "Capture", WM8960_POWER1, 3, 0),
{	.id = snd_soc_dapm_adc, .name = wname, .sname = stname, \	// .sname = Capture
	SND_SOC_DAPM_INIT_REG_VAL(wreg, wshift, winvert), }
```

# dai widget
每一个snd_soc_dai_driver会自动创建dai_widget，如果它stream_name和stream_widget中的sname一致，则会自动创建path: {dai_widget, NULL, stream_widget}
```c
static struct snd_soc_dai_driver wm8960_dai = {
	.name = "wm8960-hifi",				// used for match
	.playback = {
		.stream_name = "Playback",	// dai_widget.stream_name
		.channels_min = 1,
		.channels_max = 2,
		.rates = WM8960_RATES,
		.formats = WM8960_FORMATS,},
	.capture = {
		.stream_name = "Capture",	// dai_widget.stream_name
		.channels_min = 1,
		.channels_max = 2,
		.rates = WM8960_RATES,
		.formats = WM8960_FORMATS,},
	.ops = &wm8960_dai_ops,
	.symmetric_rates = 1,
};
```
```c
// 挂载到component->dai_list
	ret = snd_soc_register_codec(&i2c->dev, &soc_codec_dev_wm8960, &wm8960_dai, 1);
		codec->component.probe = snd_soc_codec_drv_probe;	// return codec->driver->probe(codec);
		ret = snd_soc_register_dais(&codec->component, dai_drv, num_dai, false);
			dai = soc_add_dai(component, dai_drv + i, count == 1 && legacy_dai_naming);
				list_add(&dai->list, &component->dai_list);
		snd_soc_component_add_unlocked(&codec->component);
			list_add(&component->list, &component_list);
```
```c
// 注册声卡时会使用component->dai_list
	ret = devm_snd_soc_register_card(&pdev->dev, card);
		ret = snd_soc_register_card(card);
			ret = snd_soc_instantiate_card(card);
				ret = soc_probe_link_components(card, rtd, order);
					ret = soc_probe_component(card, component);
						// 把所有的dai都new_dai_widgets
						list_for_each_entry(dai, &component->dai_list, list) {
							ret = snd_soc_dapm_new_dai_widgets(dapm, dai);
						}
```
```c
// 创建dai widget并且挂载到dapm->card->widgets
snd_soc_dapm_new_dai_widgets
	template.reg = SND_SOC_NOPM;	// dai_widget是软件概念，不需要reg设置上下电
	if (dai->driver->playback.stream_name) {
		template.id = snd_soc_dapm_dai_in;	// is_ep
		template.name = dai->driver->playback.stream_name;	// Playback
		template.sname = dai->driver->playback.stream_name;	// Playback
		w = snd_soc_dapm_new_control_unlocked(dapm, &template);
		dai->playback_widget = w;
	}
	if (dai->driver->capture.stream_name) {
		// snd_soc_dai_driver创建dai_widget
		template.id = snd_soc_dapm_dai_out;	// is_ep
		template.name = dai->driver->capture.stream_name;	// Capture
		template.sname = dai->driver->capture.stream_name;	// Capture
		w = snd_soc_dapm_new_control_unlocked(dapm, &template);
			w->is_ep = SND_SOC_DAPM_EP_SINK;
			list_add_tail(&w->list, &dapm->card->widgets);

		dai->capture_widget = w;
	}
```
```c
// 通过sname进行match && create new path: {dai_widget, NULL, stream_widget}
snd_soc_dapm_link_dai_widgets
	list_for_each_entry(dai_w, &card->widgets, list) {
		// 筛选出dai_widget
		switch (dai_w->id) {
		case snd_soc_dapm_dai_in:
		case snd_soc_dapm_dai_out:
			break;
		default:
			continue;
		}
		// 
		list_for_each_entry(w, &card->widgets, list) {
			// 筛选出非dai_widget进行match，即match dai_widget和stream_widget
			switch (w->id) {
			case snd_soc_dapm_dai_in:
			case snd_soc_dapm_dai_out:
				continue;
			default:
				break;
			}

			if (!w->sname || !strstr(w->sname, dai_w->sname))	// Capture，通过sname来进行match
				continue;
			// 找到了
			if (dai_w->id == snd_soc_dapm_dai_in) {
				// playback
				src = dai_w;
				sink = w;
			} else {
				// capture
				src = w;		// Left ADC
				sink = dai_w;
			}
			// 相当于多了一个path{capture_dai_widget, NULL, "Left ADC"}, 这才是完整的一个complete_path
			snd_soc_dapm_add_path(w->dapm, src, sink, NULL, NULL);	// Left ADC, dai_w, NULL, NULL: control是null，所以是直连
		}
	}
```
