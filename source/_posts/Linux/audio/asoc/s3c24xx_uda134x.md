---
title: s3c24xx_uda134x驱动分析
date: 2020-07-17 12:00:00
categories:
- Linux
- asoc
---

platform驱动会分配一块连续的内存用于dma传输
这块内存上层和底层共用，用ringbuffer进行管理
上层会把数据cp到这块内存中
底层会把这块内存暴露给dma使用

dma控制器：
src是上面的这块内存
dst是IISFIFO
count = period_size

上层一次写入流程：
1. 写入buffer_size(例如4*period_size)数据到内存中
2. dma每次从内存中搬period_size数据到IISFIFO，搬完产生dma中断，分4次搬完
<!--more-->

# machine
```c
static struct snd_soc_dai_link s3c24xx_uda134x_dai_link = {
	.name = "UDA134X",
	.stream_name = "UDA134X",
	.codec_name = "uda134x-codec",		// uda134x.c
	.codec_dai_name = "uda134x-hifi",
	.cpu_dai_name = "s3c24xx-iis",		// S3c24xx-i2s.c
	.ops = &s3c24xx_uda134x_ops,
	.platform_name	= "samsung-audio",	// dma.c
};
```

# platform
## cpu_dai
> sound/soc/samsung/s3c24xx-i2s.c

需要把iis信息传递给dma：要将S3C2410_IISFIFO设置为dma的dst
```c
// 设置
static struct s3c_dma_params s3c24xx_i2s_pcm_stereo_out = {
	.client		= &s3c24xx_dma_client_out,
	.channel	= DMACH_I2S_OUT,
	.dma_addr	= S3C2410_PA_IIS + S3C2410_IISFIFO,	// 0x55000010
	.dma_size	= 2,	// S3C2410_DCON_HALFWORD
};

s3c24xx_i2s_hw_params
	dma_data = &s3c24xx_i2s_pcm_stereo_out;
	snd_soc_dai_set_dma_data(rtd->cpu_dai, substream, dma_data);	// 这里set，下面get
```

## dma
> sound/soc/samsung/dma.c

### 分配内存
分配一块连续的内存用于dma传输
```c
struct snd_soc_platform_driver samsung_asoc_platform = {
	.ops		= &dma_ops,
	.pcm_new	= dma_new,
	.pcm_free	= dma_free_dma_buffers,
};

static int dma_new(struct snd_card *card,
	struct snd_soc_dai *dai, struct snd_pcm *pcm)
{
	struct snd_pcm_substream *substream = pcm->streams[stream].substream;
	struct snd_dma_buffer *buf = &substream->dma_buffer;	// pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream->dma_buffer
	size_t size = dma_hardware.buffer_bytes_max;	// 128*1024
	preallocate_dma_buffer(pcm, SNDRV_PCM_STREAM_PLAYBACK);
		// 底层分配128k字节的内存空间用于dma传输
		buf->area = dma_alloc_writecombine(pcm->card->dev, size, &buf->addr, GFP_KERNEL);	// buf->addr保存dma内存的物理地址，buf->area保存虚拟地址
		buf->bytes = size;
}
```

### 获得上层传递下来的和iis驱动的参数信息
```c
dma_hw_params
	struct snd_pcm_runtime *runtime = substream->runtime;
	struct runtime_data *prtd = runtime->private_data;	// prtd = substream->runtime->private_data
	/*
	 * 上层设置 = period_size*period_count*一帧的bytes数
	 * tinyplay中默认是1024*4*sample(32bits)*ch(2) = 1024*4*8 = 32k
	 */ 
	unsigned long totbytes = params_buffer_bytes(params);

	// 获得iis的驱动信息，主要要得到IISFIFO，作为dma的dst
	struct s3c_dma_params *dma = snd_soc_dai_get_dma_data(rtd->cpu_dai, substream);	// get
	prtd->params = dma;	// 放入到runtime中

	s3c2410_dma_set_buffdone_fn(prtd->params->channel, audio_buffdone);	// 设置dma传输中断的回调函数

	snd_pcm_set_runtime_buffer(substream, &substream->dma_buffer);
		struct snd_pcm_runtime *runtime = substream->runtime;
		runtime->dma_buffer_p = bufp;
		runtime->dma_area = bufp->area;	// dma buffer virtual address
		runtime->dma_addr = bufp->addr;	// physical address
		runtime->dma_bytes = bufp->bytes;

	prtd->dma_period = params_period_bytes(params);	// = period_size*一帧的bytes数 = 1024*8 = 8k
	prtd->dma_start = runtime->dma_addr;	// 分配dma内存的物理地址
	prtd->dma_pos = prtd->dma_start;
	prtd->dma_end = prtd->dma_start + totbytes;
```

### dma_prepare
设置dma的dst(IISFIFO)/传输unit(2 bytes)，准备开启dma传输
```c
dma_prepare
	// dst为S3C2410_IISFIFO且地址固定，src地址变动
	s3c2410_dma_devconfig(prtd->params->channel,
			      S3C2410_DMASRC_MEM,
			      prtd->params->dma_addr);	// 目的地址：S3C2410_IISFIFO
		hwcfg |= S3C2410_DISRCC_INC;	// 1: fixed
		dma_wrreg(chan, S3C2410_DMA_DISRCC, (0<<1) | (0<<0));
		dma_wrreg(chan, S3C2410_DMA_DIDST,  devaddr);	// S3C2410_IISFIFO
		dma_wrreg(chan, S3C2410_DMA_DIDSTC, hwcfg & 3);	// DIDST is fixed
		chan->addr_reg = dma_regaddr(chan, S3C2410_DMA_DISRC);	// src
	// 设置dma最小的传输单元是2个字节，则TC需要设置为period_bytes/2
	s3c2410_dma_config(prtd->params->channel,
			   prtd->params->dma_size);	// 2
		chan->xfer_unit = xferunit;	// 2

	// 调用s3c2410_dma_enqueue
	dma_enqueue(substream);	// 没明白为什么要在dma_prepare调这个函数，调这个函数是要开始dma传输了，但是数据呢？
		dma_addr_t pos = prtd->dma_pos;	// 最开始=dma_alloc_writecombine的起始地址
		// len一般为dma_period，除非到ringbuffer末尾
		unsigned long len = prtd->dma_period;
		if ((pos + len) > prtd->dma_end)
			len  = prtd->dma_end - pos;
		// 传输完一次period_size数据，时间为period_size/sample rate
		ret = s3c2410_dma_enqueue(prtd->params->channel,
				substream, pos, len);
		// 更新pos，传输完一次+period_size
		pos += prtd->dma_period;
		if (pos >= prtd->dma_end)
			pos = prtd->dma_start;
```

设置dma控制器的src/count(period_size)
```c
s3c2410_dma_enqueue(prtd->params->channel, substream, pos, len);	// len一般为period_bytes
	buf = kmem_cache_alloc(dma_kmem, GFP_ATOMIC);
	buf->data  = buf->ptr = data;	// pos
	buf->size  = size;	// period_size

	// 把buff放在适合的位置(3个里面选一个)等待传输
	if (chan->curr == NULL) {
		chan->curr = buf;
		chan->end  = buf;
		chan->next = NULL;
	} else {
		chan->end->next = buf;
		chan->end = buf;
	}
	if (chan->next == NULL)
		chan->next = buf;
	
	while (s3c2410_dma_canload(chan) && chan->next != NULL) {
		// 把内存地址给dma的src，count设置为period_size
		s3c2410_dma_loadbuffer(chan, chan->next);
			// 把dma内存中目前使用到的物理地址写到dma控制器的src
			writel(buf->data, chan->addr_reg);	// chan->addr_reg = dma_regaddr(chan, S3C2410_DMA_DISRC);
			/*
			 * 设置transfer count: buf->size/2(因为S3C2410_DCON_HALFWORD，unit为2 bytes，所以要/2)，
			 * Interrupt will occur when TC reaches 0
			 */
			dma_wrreg(chan, S3C2410_DMA_DCON, chan->dcon | reload | (buf->size/chan->xfer_unit));	// 每次传输period_size
			chan->next = buf->next;

	}
```

### start dma
dma.c这个驱动写的挺复杂的，流程上来说只要先走s3c2410_dma_start，后面调s3c2410_dma_loadbuffer更新地址就行了
```c
// 开启dma传输
dma_trigger
	switch (cmd) {
	case SNDRV_PCM_TRIGGER_START:
		prtd->state |= ST_RUNNING;
		s3c2410_dma_ctrl(prtd->params->channel, S3C2410_DMAOP_START);
			return s3c2410_dma_start(chan);
				s3c2410_dma_loadbuffer(chan, chan->next);		// 更新内存地址给dma的src，count设置为period_size
				// enable irq和设置S3C2410_DMA_DMASKTRIG是dma_start特有的，即其他地方调用s3c2410_dma_loadbuffer也没用，trigger是开始的地方
				enable_irq(chan->irq);							// enable dma interrupt
				tmp = dma_rdreg(chan, S3C2410_DMA_DMASKTRIG);	// 启动dma传输
				tmp &= ~S3C2410_DMASKTRIG_STOP;
				tmp |= S3C2410_DMASKTRIG_ON;
				dma_wrreg(chan, S3C2410_DMA_DMASKTRIG, tmp);

		break;
```

### dma中断
传输完毕，dma产生中断，更新hw_ptr指针，然后下一次dma传输
```c
s3c2410_dma_request
	err = request_irq(chan->irq, s3c2410_dma_irq, IRQF_DISABLED,
			  client->name, (void *)chan);

// Interrupt will occur when TC reaches 0
// TC: period_size
s3c2410_dma_irq
	s3c2410_dma_buffdone(chan, buf, S3C2410_RES_OK);
		(chan->callback_fn)(chan, buf->id, buf->size, result);	// audio_buffdone
			snd_pcm_period_elapsed(substream);	// 常规操作：update hw_ptr
			dma_enqueue(substream);				// 下一次dma传输
```

# 寄存器说明
```
DMA INITIAL SOURCE (DISRC) REGISTER
	 Base address (start address) of source data to transfer

DMA INITIAL SOURCE CONTROL (DISRCC) REGISTER
LOC	[1]	0: the source is in the system bus (AHB).
		1: the source is in the peripheral bus (APB).
INC	[0]	0 = Increment 1= Fixed

DMA INITIAL DESTINATION (DIDST) REGISTER
	Base address (start address) of destination for the transfer.

DMA INITIAL DESTINATION CONTROL (DIDSTC) REGISTER
CHK_INT	[2]	Select interrupt occurrence time when auto reload is setting.
			0 : Interrupt will occur when TC reaches 0.
			1 : Interrupt will occur after autoreload is performed.
```
