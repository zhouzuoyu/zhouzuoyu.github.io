---
title: soc-audio注册
date: {{date}}
categories:
- Kernel
- asoc
---
# 源码
> sound\soc\soc-core.c

# 功能
alsa在嵌入式系统中使用较为不便，so将alsa封装 -> asoc，框架上分为machine、platform、codec
该文件封装了alsa，引出了一些API
<!--more-->
# 源码分析
## 注册
1. soc_driver
	```c
	static struct platform_driver soc_driver = {
		.driver		= {
			.name		= "soc-audio",
			.owner		= THIS_MODULE,
			.pm		= &snd_soc_pm_ops,
		},
		.probe		= soc_probe,
		.remove		= soc_remove,
	};
	static int __init snd_soc_init(void)
		return platform_driver_register(&soc_driver);
	```
2. soc_device
	so需要在machine代码中注册匹配的soc_device, like this
	```c
	static int s3c24xx_uda134x_probe(struct platform_device *pdev)
		s3c24xx_uda134x_snd_device = platform_device_alloc("soc-audio", -1);
	```
3. soc_probe
	匹配成功则调用probe函数
	```c
	static int soc_probe(struct platform_device *pdev)
	{
		snd_soc_register_card(card);
			// 添加到card_list
			list_add(&card->list, &card_list);
			
			// 实例化所有card_list上的声卡
			snd_soc_instantiate_cards();
				list_for_each_entry(card, &card_list, list)
					snd_soc_instantiate_card(card);
	}
	```

4. 声卡注册
核心是snd_card_register，还是alsa那一套
```c
snd_soc_instantiate_cards
	// 同块声卡只能注册一次
	if (card->instantiated) {
		return;
	}
	
	/* bind DAIs */
	for (i = 0; i < card->num_links; i++)
		soc_bind_dai_link(card, i);
	
	// create && init
	snd_card_create(SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1,
			card->owner, 0, &card->snd_card);
	
	// 调用dai_link的probe
	for (i = 0; i < card->num_links; i++) {
		ret = soc_probe_dai_link(card, i);
	}
	
	ret = snd_card_register(card->snd_card);
	
	card->instantiated = 1;	// 注册成功
```
