---
title: qualcomm dapm分析
date: 2020-07-12 14:46:47
categories:
- Linux
- asoc
- dapm
---

# 定义kcontrol/widget/path
## snd_kcontrol_new
```c
static const struct snd_kcontrol_new sec_tdm_rx_0_mixer_controls[] = {
	SOC_DOUBLE_EXT("MultiMedia1", SND_SOC_NOPM,
	MSM_BACKEND_DAI_SEC_TDM_RX_0,
	MSM_FRONTEND_DAI_MULTIMEDIA1, 1, 0, msm_routing_get_audio_mixer,
	msm_routing_put_audio_mixer),

	展开后：
{
	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = ("MultiMedia1"),\
	.info = snd_soc_info_volsw, \
	.get = msm_routing_get_audio_mixer, .put = msm_routing_put_audio_mixer, \
	.private_value = \
		SOC_DOUBLE_VALUE(SND_SOC_NOPM, MSM_BACKEND_DAI_SEC_TDM_RX_0, MSM_FRONTEND_DAI_MULTIMEDIA1, 1, 0, 0) }
};
```
<!--more-->
## widget
```c
static const struct snd_soc_dapm_widget msm_qdsp6_widgets[] = {
	SND_SOC_DAPM_AIF_IN("MM_DL1", "MultiMedia1 Playback", 0, 0, 0, 0),

	SND_SOC_DAPM_AIF_OUT("SEC_TDM_RX_0", "Secondary TDM0 Playback",
				0, 0, 0, 0),

	// 一个snd_soc_dapm_widget可能有很多snd_kcontrol_new
	SND_SOC_DAPM_MIXER("SEC_TDM_RX_0 Audio Mixer", SND_SOC_NOPM, 0, 0,
				sec_tdm_rx_0_mixer_controls,
				ARRAY_SIZE(sec_tdm_rx_0_mixer_controls)),
	展开后：
	{	.id = snd_soc_dapm_mixer, .name = "SEC_TDM_RX_0 Audio Mixer", \
		SND_SOC_DAPM_INIT_REG_VAL(SND_SOC_NOPM, 0, 0), \
		.kcontrol_news = sec_tdm_rx_0_mixer_controls, .num_kcontrols = ARRAY_SIZE(sec_tdm_rx_0_mixer_controls)}
};
```

## path
```c
static const struct snd_soc_dapm_route intercon[] = {	
	{"SEC_TDM_RX_0 Audio Mixer", "MultiMedia1", "MM_DL1"},
	{"SEC_TDM_RX_0", NULL, "SEC_TDM_RX_0 Audio Mixer"},
};
```

# 注册kcontrol/widget/path
```c
msm_routing_probe
	// 注册widget，把msm_qdsp6_widgets挂载到list上
	snd_soc_dapm_new_controls(&platform->component.dapm, msm_qdsp6_widgets, ARRAY_SIZE(msm_qdsp6_widgets));
		for (i = 0; i < num; i++) {
			snd_soc_dapm_new_control_unlocked(dapm, widget);
				w->name = kstrdup_const(widget->name, GFP_KERNEL);
				w->power_check = dapm_generic_check_power;
				list_add_tail(&w->list, &dapm->card->widgets);	// FIXME，注册，把widgets挂载，snd_soc_dapm_new_widgets中使用
		}

	// 1. 创建path
	// 2. 根据reg设置path->connect
	snd_soc_dapm_add_routes(&platform->component.dapm, intercon, ARRAY_SIZE(intercon));
		for (i = 0; i < num; i++) {
			r = snd_soc_dapm_add_route(dapm, route);	// {"SEC_TDM_RX_0 Audio Mixer", "MultiMedia1", "MM_DL1"},
				sink = route->sink;		// 这个例子中"SEC_TDM_RX_0 Audio Mixer"
				source = route->source;	// "MM_DL1"
				// 遍历msm_qdsp6_widgets，找到之前注册的widget
				list_for_each_entry(w, &dapm->card->widgets, list) {
					if (!wsink && !(strcmp(w->name, sink))) {
						wtsink = w;
						if (w->dapm == dapm) {
							wsink = w;
							if (wsource)
								break;
						}
						continue;
					}
					if (!wsource && !(strcmp(w->name, source))) {
						wtsource = w;
						if (w->dapm == dapm) {
							wsource = w;
							if (wsink)
								break;
						}
					}
				}

				snd_soc_dapm_add_path(dapm, wsource, wsink, route->control, route->connected);
					path = kzalloc(sizeof(struct snd_soc_dapm_path), GFP_KERNEL);	// create path
					ret = dapm_connect_mixer(dapm, path, control);
						for (i = 0; i < path->sink->num_kcontrols; i++) {
							if (!strcmp(control_name, path->sink->kcontrol_news[i].name)) {	// "MultiMedia1", 
								path->name = path->sink->kcontrol_news[i].name;	// "MultiMedia1"
								dapm_set_mixer_path_status(path, i, nth_path++);
									p->connect = 0;
								return 0;
							}
						}

					list_add(&path->list, &dapm->card->paths);
					snd_soc_dapm_for_each_direction(dir)
						list_add(&path->list_node[dir], &widgets[dir]->edges[dir]);

			route++;
		}

	// 将card->widgets->kcontrol_news都注册kcontrol
	snd_soc_dapm_new_widgets(platform->component.dapm.card);
		list_for_each_entry(w, &card->widgets, list) {
			dapm_new_mixer(w);
				for (i = 0; i < w->num_kcontrols; i++) {
					ret = dapm_create_or_share_kcontrol(w, i);
						case snd_soc_dapm_mixer:
							wname_in_long_name = true;
							kcname_in_long_name = true;
							break;
						/* long_name = "SEC_TDM_RX_0 Audio Mixer MultiMedia1" */
						long_name = kasprintf(GFP_KERNEL, "%s %s",
									 w->name + prefix_len,				// SEC_TDM_RX_0 Audio Mixer
									 w->kcontrol_news[kci].name);		// MultiMedia1
						name = long_name;
						kcontrol = snd_soc_cnew(&w->kcontrol_news[kci], NULL, name, prefix);
						ret = snd_ctl_add(card, kcontrol);

						ret = dapm_kcontrol_add_widget(kcontrol, w);
						w->kcontrols[kci] = kcontrol;
				}
		}
```
