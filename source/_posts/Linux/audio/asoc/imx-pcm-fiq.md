---
title: DMA使用定时器告知传输完period size
date: 2019-07-16 11:02:00
categories:
- Linux
- audio
---

# 设计思路
每次dma传输完成一个周期的数据传输后，都要调用snd_pcm_period_elapsed()告知pcm native一个周期的数据已经传送到FIFO上了，然后再次调用dma传输音频数据...如此循环。
但有些Platform可能由于设计如此或设计缺陷，**dma传输完一个周期的数据不会产生硬件中断**。这样系统如何知道什么时候传输完一个周期的数据了呢？
I2S总线传输一个周期的数据所需的时间其实就是dma搬运一个周期的数据所需的时间。这很容易理解：I2S FIFO消耗完一个周期的数据，dma才能接着搬运一个周期的数据到I2S FIFO。

因此可以用定时器来模拟这种硬件中断：
1. 触发dma搬运数据时，启动定时器开始计时；
2. 当定时到1*period_size/sample_rate，这时I2S已传输完一个周期的音频数据了，进入定时器中断处理：调用 snd_pcm_period_elapsed() 告知pcm native一个周期的数据已经处理完毕了，同时准备下一次的数据搬运；
3. 继续执行步骤1；

<!--more-->
# 源码分析
> sound\soc\imx\imx-pcm-fiq.c

## 注册snd_soc_platform_driver
```c
static struct snd_pcm_ops imx_pcm_ops = {
	.open		= snd_imx_open,
	.close		= snd_imx_close,
	.ioctl		= snd_pcm_lib_ioctl,
	.hw_params	= snd_imx_pcm_hw_params,
	.prepare	= snd_imx_pcm_prepare,
	.trigger	= snd_imx_pcm_trigger,
	.pointer	= snd_imx_pcm_pointer,
	.mmap		= snd_imx_pcm_mmap,
};

static struct snd_soc_platform_driver imx_soc_platform_fiq = {
	.ops		= &imx_pcm_ops,
	.pcm_new	= imx_pcm_fiq_new,
	.pcm_free	= imx_pcm_fiq_free,
};

static int __devinit imx_soc_platform_probe(struct platform_device *pdev)
{
	ret = snd_soc_register_platform(&pdev->dev, &imx_soc_platform_fiq);
}
```

## 配置定时器
配置定时时间为period_size/rate，这段时间必然会传输完毕period_size数据
```c
// 设置回调函数
static int snd_imx_open(struct snd_pcm_substream *substream)
{
	// 初始化定时器
	hrtimer_init(&iprtd->hrt, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
	iprtd->hrt.function = snd_hrtimer_callback;	// 回调函数
}

// 设置定时时间
static int snd_imx_pcm_hw_params(struct snd_pcm_substream *substream,
				struct snd_pcm_hw_params *params)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	struct imx_pcm_runtime_data *iprtd = runtime->private_data;

	iprtd->size = params_buffer_bytes(params);
	iprtd->periods = params_periods(params);
	iprtd->period = params_period_bytes(params) ;

	// 传输完period所需要的时间
	// 例如rate是48kHz，period_size是4800，则传输时间=4800/48000=100ms
	iprtd->poll_time_ns = 1000000000 / params_rate(params) *
				params_period_size(params);	// params_period_size: Approx frames between interrupts.

	return 0;
}
```
## 开启定时器
```c
static int snd_imx_pcm_trigger(struct snd_pcm_substream *substream, int cmd)
{
	switch (cmd) {
	case SNDRV_PCM_TRIGGER_START:
	case SNDRV_PCM_TRIGGER_RESUME:
	case SNDRV_PCM_TRIGGER_PAUSE_RELEASE:
		hrtimer_start(&iprtd->hrt, ns_to_ktime(iprtd->poll_time_ns),	// 开启定时器
		      HRTIMER_MODE_REL);
		break;
	}

	return 0;
}
```

## 回调函数
传输完毕后调用snd_pcm_period_elapsed，告知pcm native一个周期的数据已经传送到FIFO上了
```c
static enum hrtimer_restart snd_hrtimer_callback(struct hrtimer *hrt)
{
	/* How much data have we transferred since the last period report? */
	if (iprtd->offset >= iprtd->last_offset)
		delta = iprtd->offset - iprtd->last_offset;
	else
		delta = runtime->buffer_size + iprtd->offset
			- iprtd->last_offset;

	/* If we've transferred at least a period then report it and
	 * reset our poll time */
	if (delta >= iprtd->period) {
		snd_pcm_period_elapsed(substream);	// 告知pcm native一个周期的数据已经传送到FIFO上了
		iprtd->last_offset = iprtd->offset;
	}

	hrtimer_forward_now(hrt, ns_to_ktime(iprtd->poll_time_ns));	// 重新触发定时器

	return HRTIMER_RESTART;
}
```
