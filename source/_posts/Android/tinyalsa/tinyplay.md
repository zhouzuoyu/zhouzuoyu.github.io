---
title: tinyplay
date: 2018-02-8 12:00:00
categories: 
- Android
- tinyalsa
---
# 源码
>   external/tinyalsa/tinyplay.c
>   external/tinyalsa/pcm.c

# 功能
使用tinyplay播放riff/wave格式的音频文件
> Usage: tinyplay file.wav \[-D card\] \[-d device\] \[-p period_size\] \[-n n_periods\]

<!--more-->
# 源码分析
## 总体框架
>   tinyplay.c

*   读取音频文件数据
*   播放音频文件数据

```c
int main(int argc, char **argv)
	// 读取音频文件数据
	filename = argv[1];
	file = fopen(filename, "rb");
	fread(&riff_wave_header, sizeof(riff_wave_header), 1, file);
	// 播放
	play_sample(file, card, device, chunk_fmt.num_channels, chunk_fmt.sample_rate,
			chunk_fmt.bits_per_sample, period_size, period_count);
```

打开音频playback节点(/dev/snd/pcmCxDxp)
循环帧数据写入

```c
void play_sample(FILE *file, unsigned int card, unsigned int device, unsigned int channels,
                 unsigned int rate, unsigned int bits, unsigned int period_size,
                 unsigned int period_count)
	// 参数打包到config
	config.channels = channels;
	config.rate = rate;
	config.period_size = period_size;
	config.period_count = period_count;
	if (bits == 32)
		config.format = PCM_FORMAT_S32_LE;
	else if (bits == 16)
		config.format = PCM_FORMAT_S16_LE;
	config.start_threshold = 0;
	config.stop_threshold = 0;
	config.silence_threshold = 0;

	/*
	 * open /dev/snd/pcmCxDxp
	 * 设置参数
	 */
	pcm = pcm_open(card, device, PCM_OUT, &config);

	// 播放：循环写音频帧数据
	do {
		num_read = fread(buffer, 1, size, file);	// 把音频文件中的数据读取到buffer中
		if (num_read > 0) {
			if (pcm_write(pcm, buffer, num_read)) {	// pcm播放
				fprintf(stderr, "Error playing sample\n");
				break;
			}
		}
	} while (!close && num_read > 0);
```

## pcm API

>   pcm.c

*   open /dev/snd/pcmCxDxp
*   设置参数：SNDRV_PCM_IOCTL_HW_PARAMS
　　　　　SNDRV_PCM_IOCTL_SW_PARAMS

```c
struct pcm *pcm_open(unsigned int card, unsigned int device,
                     unsigned int flags, struct pcm_config *config)
	// open /dev/snd/pcmCxDxp(c: capture，p: playback)
	snprintf(fn, sizeof(fn), "/dev/snd/pcmC%uD%u%c", card, device,
			flags & PCM_IN ? 'c' : 'p');
	pcm->fd = open(fn, O_RDWR);

	// 设置参数
	// 将config中的参数转化到params
	param_init(&params);
	param_set_mask(&params, SNDRV_PCM_HW_PARAM_FORMAT,
					pcm_format_to_alsa(config->format));
	param_set_mask(&params, SNDRV_PCM_HW_PARAM_SUBFORMAT,
					SNDRV_PCM_SUBFORMAT_STD);
	param_set_min(&params, SNDRV_PCM_HW_PARAM_PERIOD_SIZE, config->period_size);
	param_set_int(&params, SNDRV_PCM_HW_PARAM_SAMPLE_BITS,
					pcm_format_to_bits(config->format));
	param_set_int(&params, SNDRV_PCM_HW_PARAM_FRAME_BITS,
					pcm_format_to_bits(config->format) * config->channels);
	param_set_int(&params, SNDRV_PCM_HW_PARAM_CHANNELS,
					config->channels);
	param_set_int(&params, SNDRV_PCM_HW_PARAM_PERIODS, config->period_count);
	param_set_int(&params, SNDRV_PCM_HW_PARAM_RATE, config->rate);
	// SNDRV_PCM_IOCTL_HW_PARAMS
	ioctl(pcm->fd, SNDRV_PCM_IOCTL_HW_PARAMS, &params);
	// SNDRV_PCM_IOCTL_SW_PARAMS
	ioctl(pcm->fd, SNDRV_PCM_IOCTL_SW_PARAMS, &sparams)
```

写一帧数据
```c
int pcm_write(struct pcm *pcm, const void *data, unsigned int count)
	x.buf = (void*)data;
	x.frames = count / (pcm->config.channels *
						pcm_format_to_bits(pcm->config.format) / 8);
	for (;;) {
		if (!pcm->running) {
			// 第一次进入
			int prepare_error = pcm_prepare(pcm);
			if (ioctl(pcm->fd, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &x))
				return oops(pcm, errno, "cannot write initial data");
			pcm->running = 1;
			return 0;
		}
		// 后续
		ioctl(pcm->fd, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &x);
		return 0;
	}
```
