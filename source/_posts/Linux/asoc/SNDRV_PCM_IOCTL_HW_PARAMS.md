---
title: ioctl(SNDRV_PCM_IOCTL_HW_PARAMS)调用过程
date: 2018-11-19 15:53:00
categories:
- Linux
- asoc
---

# APP
aplay会设置参数，最终调用**ioctl(pcm_hw->fd, SNDRV_PCM_IOCTL_HW_PARAMS, params)**
## 设置
```c
static int snd_pcm_hw_hw_params(snd_pcm_t *pcm, snd_pcm_hw_params_t * params)
{
	hw_params_call(hw, params);
		return ioctl(pcm_hw->fd, SNDRV_PCM_IOCTL_HW_PARAMS, params);
}

static const snd_pcm_ops_t snd_pcm_hw_ops = {
	.hw_params = snd_pcm_hw_hw_params,
};

snd_pcm_hw_open_fd
	pcm->ops = &snd_pcm_hw_ops;
```
<!--more-->
## 调用
```c
int main(int argc, char *argv[])
	playback(argv[optind++]);
		switch(file_type) {
		case FORMAT_WAVE:
			playback_wave(name, &loaded);
				playback_go(fd, dtawave, pbrec_count, FORMAT_WAVE, name);
						set_params();
							err = snd_pcm_hw_params(handle, params);
								err = _snd_pcm_hw_params_internal(pcm, params);
									err = pcm->ops->hw_params(pcm->op_arg, params);	// 即snd_pcm_hw_hw_params

			break;
		}
```

# kernel
调用**ioctl(pcm_hw->fd, SNDRV_PCM_IOCTL_HW_PARAMS, params)**，将会导致以下的hw_params回调函数被调用：
1. dai_link->ops->hw_params
2. codec_dai->driver->ops->hw_params
3. cpu_dai->driver->ops->hw_params
4. platform->driver->ops->hw_params

## 设置
```c
soc_pcm_hw_params
{
	if (rtd->dai_link->ops && rtd->dai_link->ops->hw_params) {
		ret = rtd->dai_link->ops->hw_params(substream, params);
	}

	if (codec_dai->driver->ops->hw_params) {
		ret = codec_dai->driver->ops->hw_params(substream, params, codec_dai);
	}

	if (cpu_dai->driver->ops->hw_params) {
		ret = cpu_dai->driver->ops->hw_params(substream, params, cpu_dai);
	}

	if (platform->driver->ops && platform->driver->ops->hw_params) {
		ret = platform->driver->ops->hw_params(substream, params);
	}
}

static int soc_new_pcm(struct snd_soc_pcm_runtime *rtd, int num)
{
	soc_pcm_ops->hw_params	= soc_pcm_hw_params;
	if (playback)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, soc_pcm_ops);
			for (substream = stream->substream; substream != NULL; substream = substream->next)
				substream->ops = ops;
}
```

## 使用
```c
const struct file_operations snd_pcm_f_ops[2] = {
	{
		.unlocked_ioctl =	snd_pcm_playback_ioctl,
	},
	{
		.unlocked_ioctl =	snd_pcm_capture_ioctl,
	}
};

static long snd_pcm_playback_ioctl(struct file *file, unsigned int cmd,
				   unsigned long arg)
{
	return snd_pcm_playback_ioctl1(file, pcm_file->substream, cmd, (void __user *)arg);
		return snd_pcm_common_ioctl1(file, substream, cmd, arg);
			switch (cmd) {
			case SNDRV_PCM_IOCTL_HW_PARAMS:
				return snd_pcm_hw_params_user(substream, arg);
					params = memdup_user(_params, sizeof(*params));
						err = snd_pcm_hw_params(substream, params);
							if (substream->ops->hw_params != NULL) {
								err = substream->ops->hw_params(substream, params);	// 即soc_pcm_hw_params
							}
						copy_to_user(_params, params, sizeof(*params));
			}
}
```
