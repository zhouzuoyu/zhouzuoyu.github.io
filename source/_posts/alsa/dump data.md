---
title: dump alsa data
date: 2019-04-23 10:13:00
categories:
- alsa
---

> 源码路径：src/pcm/pcm.c


```c
static void dump_out_data(const void* buffer, size_t bytes, char * name, snd_pcm_stream_t dir)
{
	if (!strcmp("nfdspout", name)) {
		FILE *fd1 = fopen("/tmp/nfdspout.pcm", "a+");
		if (fd1) {
			int flen = fwrite((char *)buffer, bytes, 1, fd1);
			fclose(fd1);
		}
	} else if ...
}
```
<!--more-->

```c
snd_pcm_sframes_t snd_pcm_writei(snd_pcm_t *pcm, const void *buffer, snd_pcm_uframes_t size)
{
	assert(pcm);
	assert(size == 0 || buffer);
	if (CHECK_SANITY(! pcm->setup)) {
		SNDMSG("PCM not set up");
		return -EIO;
	}
	if (pcm->access != SND_PCM_ACCESS_RW_INTERLEAVED) {
		SNDMSG("invalid access type %s", snd_pcm_access_name(pcm->access));
		return -EINVAL;
	}
	// add this
	dump_out_data(buffer, snd_pcm_frames_to_bytes(pcm, size), pcm->name, pcm->stream);

	return _snd_pcm_writei(pcm, buffer, size);
}
```

