---
title: widget & route
date: 2018-03-20 17:06:47
categories:
- Linux
- asoc
- dapm
---

> Wm8960.c (linux-3.0.86\sound\soc\codecs)
> soc-dapm.c (linux-3.0.86\sound\soc\codecs)

先注册再添加route

# snd_soc_dapm_new_controls
定义各种widget
```c
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
	SND_SOC_DAPM_INPUT("LINPUT1"),
	...
};
```
<!--more-->
注册widgets
```c
snd_soc_dapm_new_controls(dapm, wm8960_dapm_widgets, ARRAY_SIZE(wm8960_dapm_widgets));
	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_new_control(dapm, widget);
			w = dapm_cnew_widget(widget);
			list_add(&w->list, &dapm->card->widgets);	// 添加到链表中
		widget++;
	}
```

# snd_soc_dapm_add_routes

> sink <- control <- source

声明使用哪些widget组成route
```c
static const struct snd_soc_dapm_route audio_paths[] = {
	{ "Left Boost Mixer", "LINPUT1 Switch", "LINPUT1" }
	...
};
```

添加route
```c
snd_soc_dapm_add_routes(dapm, audio_paths, ARRAY_SIZE(audio_paths));
	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_add_route(dapm, route);
			sink = route->sink;
			source = route->source;
			// 通过name在dapm->card->widgets链表中搜索，将结果保存到wsink和wsource中
			list_for_each_entry(w, &dapm->card->widgets, list) {
				if (!wsink && !(strcmp(w->name, sink))) {
					wtsink = w;
					if (w->dapm == dapm)
						wsink = w;
					continue;
				}
				if (!wsource && !(strcmp(w->name, source))) {
					wtsource = w;
					if (w->dapm == dapm)
						wsource = w;
				}
			}
			
			// 创建并且init path
			path = kzalloc(sizeof(struct snd_soc_dapm_path), GFP_KERNEL);
			// 保存
			path->source = wsource;
			path->sink = wsink;
			path->connected = route->connected;
			
			// { "Left Input Mixer", NULL, "LINPUT1", }: 直连
			if (control == NULL) {
				list_add(&path->list, &dapm->card->paths);
				list_add(&path->list_sink, &wsink->sources);
				list_add(&path->list_source, &wsource->sinks);
				path->connect = 1;
				return 0;
			}
			// 不是直连的情况，看wsink类型
			switch (wsink->id) {
			case snd_soc_dapm_adc:
			case snd_soc_dapm_dac:
			case snd_soc_dapm_pga:
			case snd_soc_dapm_out_drv:
			case snd_soc_dapm_input:
			case snd_soc_dapm_output:
			case snd_soc_dapm_micbias:
			case snd_soc_dapm_vmid:
			case snd_soc_dapm_pre:
			case snd_soc_dapm_post:
			case snd_soc_dapm_supply:
			case snd_soc_dapm_aif_in:
			case snd_soc_dapm_aif_out:
				list_add(&path->list, &dapm->card->paths);
				list_add(&path->list_sink, &wsink->sources);
				list_add(&path->list_source, &wsource->sinks);
				path->connect = 1;
				return 0;
			// mux
			case snd_soc_dapm_mux:
			case snd_soc_dapm_virt_mux:
			case snd_soc_dapm_value_mux:
				ret = dapm_connect_mux(dapm, wsource, wsink, path, control,
					&wsink->kcontrol_news[0]);
				break;
			// mixer
			case snd_soc_dapm_switch:
			case snd_soc_dapm_mixer:
			case snd_soc_dapm_mixer_named_ctl:
				ret = dapm_connect_mixer(dapm, wsource, wsink, path, control);	// control == "LINPUT1 Switch"
				break;
			case snd_soc_dapm_hp:
			case snd_soc_dapm_mic:
			case snd_soc_dapm_line:
			case snd_soc_dapm_spk:
				list_add(&path->list, &dapm->card->paths);
				list_add(&path->list_sink, &wsink->sources);
				list_add(&path->list_source, &wsource->sinks);
				path->connect = 0;
				return 0;
			}
			
		route++;
	}
```

## snd_soc_dapm_mixer
mixer类型的widget必然有多个switch
```c
static const struct snd_kcontrol_new wm8960_lin_boost[] = {
	SOC_DAPM_SINGLE("LINPUT2 Switch", WM8960_LINPATH, 6, 1, 0),
	SOC_DAPM_SINGLE("LINPUT3 Switch", WM8960_LINPATH, 7, 1, 0),
	SOC_DAPM_SINGLE("LINPUT1 Switch", WM8960_LINPATH, 8, 1, 0),	// WM8960_LINPATH reg的第8位决定开关的状态
};
SND_SOC_DAPM_MIXER("Left Boost Mixer", WM8960_POWER1, 5, 0,
		   wm8960_lin_boost, ARRAY_SIZE(wm8960_lin_boost)),
```
```c
dapm_connect_mixer
	for (i = 0; i < dest->num_kcontrols; i++) {
		if (!strcmp(control_name, dest->kcontrol_news[i].name)) {
			list_add(&path->list, &dapm->card->paths);
			list_add(&path->list_sink, &dest->sources);
			list_add(&path->list_source, &src->sinks);
			path->name = dest->kcontrol_news[i].name;
			
			// 通过读取指定寄存器来判断是否connect
			dapm_set_path_status(dest, path, i);
				struct soc_mixer_control *mc = (struct soc_mixer_control *)
					w->kcontrol_news[i].private_value;
				unsigned int reg = mc->reg;	// 例如WM8960_LINPATH
				val = snd_soc_read(w->codec, reg);
				val = (val >> shift) & mask;
				if ((invert && !val) || (!invert && val))
					p->connect = 1;
				else
					p->connect = 0;
			return 0;
		}
	}
```
