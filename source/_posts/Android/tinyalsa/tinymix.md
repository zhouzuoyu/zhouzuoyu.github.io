---
title: tinymix
date: 2018-02-08 16:27:38
categories:
- Android
- tinyalsa
---
# 源码
> external/tinyalsa/tinymix.c
> external/tinyalsa/mixer.c

# 功能
获得kcontrol信息/设置kcontrol...
> Usage: tinymix \[-D card\] \[control id\] \[value to set\]

__其实质就是对/dev/snd/controlC0进行ioctl__
<!--more-->

# 源码分析
## 总体框架
tinymix有3种功能
* 罗列所有的kcontrol
* 获取指定kcontrol信息
* 对指定kcontrol进行设置

```c
int main(int argc, char **argv)
{
	mixer = mixer_open(card);

	if (argc == 1) {
		printf("Mixer name: '%s'\n", mixer_get_name(mixer));
		tinymix_list_controls(mixer);
	} else if (argc == 2) {
		tinymix_detail_control(mixer, argv[1], 1);
	} else if (argc >= 3) {
		tinymix_set_value(mixer, argv[1], &argv[2], argc - 2);
	} else {
		printf("Usage: tinymix [-D card] [control id] [value to set]\n");
	}

	mixer_close(mixer);
}
```

## mixer_open
ioctl获得info，保存在mixer->ctl[].info中

```c
struct mixer *mixer_open(unsigned int card)
{
	// 打开/dev/snd/controlC%u
	snprintf(fn, sizeof(fn), "/dev/snd/controlC%u", card);
	fd = open(fn, O_RDWR);
	
	// ioctl(SNDRV_CTL_IOCTL_ELEM_LIST)得到kcontrol count
	ioctl(fd, SNDRV_CTL_IOCTL_ELEM_LIST, &elist);
	mixer->count = elist.count;

	// 遍历所有的kcontrol
	// 使用ioctl(SNDRV_CTL_IOCTL_ELEM_INFO)获得info
	for (n = 0; n < mixer->count; n++) {
		struct snd_ctl_elem_info *ei = mixer->elem_info + n;
		ei->id.numid = eid[n].numid;
		if (ioctl(fd, SNDRV_CTL_IOCTL_ELEM_INFO, ei) < 0)
			goto fail;
		mixer->ctl[n].info = ei;
		mixer->ctl[n].mixer = mixer;
		// 进一步获得枚举类型kcontrol的信息
		if (ei->type == SNDRV_CTL_ELEM_TYPE_ENUMERATED) {
			char **enames = calloc(ei->value.enumerated.items, sizeof(char*));
			mixer->ctl[n].ename = enames;
			for (m = 0; m < ei->value.enumerated.items; m++) {
				tmp.id.numid = ei->id.numid;
				tmp.value.enumerated.item = m;
				if (ioctl(fd, SNDRV_CTL_IOCTL_ELEM_INFO, &tmp) < 0)
					goto fail;
				enames[m] = strdup(tmp.value.enumerated.name);
			}
		}
	}
}
```

## tinymix_list_controls
> 罗列所有的kcontrol

mixer_open已经通过ioctl获取到了所需info，这里只需做显示
![tinymix](tinymix/tinymix.png)

```c
static void tinymix_list_controls(struct mixer *mixer)
{
	// Number of controls: xx
	num_ctls = mixer_get_num_ctls(mixer);
		return mixer->count;
	printf("Number of controls: %d\n", num_ctls);

	// ctl	type	num	name                                     value
	// 0	ENUM	1	Audio_Amp_R_Switch                       Off
	printf("ctl\ttype\tnum\t%-40s value\n", "name");
	for (i = 0; i < num_ctls; i++) {
		ctl = mixer_get_ctl(mixer, i);	// mixer->ctl[i]

		name = mixer_ctl_get_name(ctl);
		return (const char *)ctl->info->id.name;
		type = mixer_ctl_get_type_string(ctl);
		num_values = mixer_ctl_get_num_values(ctl);
		printf("%d\t%s\t%d\t%-40s", i, type, num_values, name);
		tinymix_detail_control(mixer, name, 0);
	}
}
```

## tinymix_detail_control
> 显示指定项kcontrol信息

![2018-02-09 10-43-18屏幕截图](tinymix/2018-02-09 10-43-18屏幕截图.png)
*   MIXER_CTL_TYPE_INT
ioctl(SNDRV_CTL_IOCTL_ELEM_READ)获得数据
* MIXER_CTL_TYPE_ENUM
mixer_open时已经获得数据，直接输出

```c
static void tinymix_detail_control(struct mixer *mixer, const char *control,
                                   int print_all)
	// 用id和name都可以控制
	if (isdigit(control[0]))
		ctl = mixer_get_ctl(mixer, atoi(control));
	else
		ctl = mixer_get_ctl_by_name(mixer, control);

	printf("%s:", mixer_ctl_get_name(ctl));	// Audio_Amp_R_Switch:
	type = mixer_ctl_get_type(ctl);
	switch (type)
	{
	case MIXER_CTL_TYPE_ENUM:
		tinymix_print_enum(ctl, print_all);
			string = mixer_ctl_get_enum_string(ctl, i);
				return (const char *)ctl->ename[enum_id];
		printf("\t%s%s", mixer_ctl_get_value(ctl, 0) == (int)i ? ">" : "", string);
		break;
	}
}
```

## tinymix_set_value
> 对指定kcontrol进行设置

大致流程如下，通过ioctl写入数据
```c
static void tinymix_set_value(struct mixer *mixer, const char *control,
                              char **values, unsigned int num_values)
{
	// id/name都可以
	if (isdigit(control[0]))
		ctl = mixer_get_ctl(mixer, atoi(control));
	else
		ctl = mixer_get_ctl_by_name(mixer, control);

	int value = atoi(values[0]);
	mixer_ctl_set_value(ctl, i, value);
		// 先读再写
		ret = ioctl(ctl->mixer->fd, SNDRV_CTL_IOCTL_ELEM_READ, &ev);
		switch (ctl->info->type) {
		case SNDRV_CTL_ELEM_TYPE_INTEGER:
		    ev.value.integer.value[id] = value;
		    break;
		}
		return ioctl(ctl->mixer->fd, SNDRV_CTL_IOCTL_ELEM_WRITE, &ev);
}
```
