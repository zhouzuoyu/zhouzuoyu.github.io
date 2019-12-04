---
title: 匹配codec_name
date: 2018-05-4 12:00:00
categories:
- Linux
- asoc
---

该例子是用pdev->dev->name和snd_soc_dai_link.codec_name进行match

# 匹配方式
通过machine代码中的snd_soc_dai_link来进行match
```c
static struct snd_soc_dai_link s3c24xx_uda134x_dai_link = {
	.name = "UDA134X",
	.stream_name = "UDA134X",
	.codec_name = "uda134x-codec",	// 指定codec
	.codec_dai_name = "uda134x-hifi",
	.cpu_dai_name = "s3c24xx-iis",
	.ops = &s3c24xx_uda134x_ops,
	.platform_name	= "samsung-audio",
};
```
<!--more-->
# 设置codec->name
## 设置pdev->dev的name
注册platform_device时会设置其名字
```c
static struct platform_device uda1340_codec = {
	.name = "uda134x-codec",	// 这个名字才是用于设置codec_name
	.id = -1,
};
static struct platform_device *mini2440_devices[] __initdata = {
	&uda1340_codec,
};
static void __init mini2440_init(void)
{
	platform_add_devices(mini2440_devices, ARRAY_SIZE(mini2440_devices));
		ret = platform_device_register(devs[i]);
			return platform_device_add(pdev);
				if (pdev->id != -1)
					dev_set_name(&pdev->dev, "%s.%d", pdev->name,  pdev->id);	// platform_device的id != -1，则设置platform_device->dev的name为[pdev->name.pdev->id]
				else
					dev_set_name(&pdev->dev, "%s", pdev->name);					// platform_device的id == -1，则设置platform_device->dev的name为[pdev->name]
}
```

## probe时会传入pdev->dev（其中包含name）
```c
static struct platform_driver uda134x_codec_driver = {
	.driver = {
		.name = "uda134x-codec",	// 这个名字主要是为了进入probe
		.owner = THIS_MODULE,
	},
	.probe = uda134x_codec_probe,
	.remove = __devexit_p(uda134x_codec_remove),
};
static int __devinit uda134x_codec_probe(struct platform_device *pdev)
{
	return snd_soc_register_codec(&pdev->dev,
			&soc_codec_dev_uda134x, &uda134x_dai, 1);
}
```

## pdev->dev->name设置到codec->name
```c
int snd_soc_register_codec(struct device *dev,
			   const struct snd_soc_codec_driver *codec_drv,
			   struct snd_soc_dai_driver *dai_drv,
			   int num_dai)
{
	codec->name = fmt_single_name(dev, &codec->id);
		strlcpy(name, dev_name(dev), NAME_SIZE);	// 参考dev_set_name
		// 该例子中name为uda134x-codec，dev->driver->name也为uda134x-codec，所以found为true，但是没有id，所以并没有什么用
		found = strstr(name, dev->driver->name);
		if (found) {
			/* get ID */
			if (sscanf(&found[strlen(dev->driver->name)], ".%d", id) == 1) {

				/* discard ID from name if ID == -1 */
				if (*id == -1)
					found[strlen(dev->driver->name)] = '\0';
			}
		}
		return kstrdup(name, GFP_KERNEL);	// pdev->dev->name
		
	list_add(&codec->list, &codec_list);			// 添加到codec_list链表
}
```

# codec->name匹配过程
```c
static int soc_bind_dai_link(struct snd_soc_card *card, int num)
{
	list_for_each_entry(codec, &codec_list, list) {	// 从codec_list中取出node
		if (!strcmp(codec->name, dai_link->codec_name)) {	// 对比name
			rtd->codec = codec;

			/* CODEC found, so find CODEC DAI from registered DAIs from this CODEC*/
			list_for_each_entry(codec_dai, &dai_list, list) {
				if (codec->dev == codec_dai->dev &&
						!strcmp(codec_dai->name, dai_link->codec_dai_name)) {
					rtd->codec_dai = codec_dai;
					goto find_platform;
				}
			}

			goto find_platform;
		}
	}
}
```

# snd_soc_register_codec
以上就是codec->name的匹配过程，现在再看一下snd_soc_register_codec
```c
int snd_soc_register_codec(struct device *dev,
			   const struct snd_soc_codec_driver *codec_drv,
			   struct snd_soc_dai_driver *dai_drv,
			   int num_dai)
```
第一个参数会用来设置codec->name，第二参数传入snd_soc_codec_driver(dapm等信息)，第三个参数传入snd_soc_dai_driver
