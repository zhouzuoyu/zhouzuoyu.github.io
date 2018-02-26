---
title: snd_soc_write
date: 2018-02-8 12:00:00
categories:
- Linux
- asoc
---
# 源码
sound\soc\codecs\wm8960.c

# 功能
snd_soc_write调用snd_soc_7_9_write

# 源码分析
## 调用codec->write
```c
unsigned int snd_soc_write(struct snd_soc_codec *codec,
			   unsigned int reg, unsigned int val)
	return codec->write(codec, reg, val);
```
<!--more-->
## 设置codec->write
在probe中调用snd_soc_codec_set_cache_io(7, 9)，所以write为snd_soc_7_9_write
又control_type设置了SND_SOC_I2C，所以最终调用i2c_master_send
```c
struct io_types io_types[] = {
	{
		.addr_bits = 7, .data_bits = 9,
		.write = snd_soc_7_9_write, .read = snd_soc_7_9_read,
	},
};
static __devinit int wm8960_i2c_probe(struct i2c_client *i2c,
				      const struct i2c_device_id *id)
{
	wm8960->control_type = SND_SOC_I2C;
}
static int wm8960_probe(struct snd_soc_codec *codec)
	// 地址位7位，数据位9位
	snd_soc_codec_set_cache_io(codec, 7, 9, wm8960->control_type);
		// search
		for (i = 0; i < ARRAY_SIZE(io_types); i++)
			if (io_types[i].addr_bits == addr_bits &&
				io_types[i].data_bits == data_bits)
				break;
		// set
		codec->write = io_types[i].write;	// snd_soc_7_9_write
		codec->read = io_types[i].read;		// snd_soc_7_9_read
		
		case SND_SOC_I2C:
			codec->hw_write = (hw_write_t)i2c_master_send;
			if (io_types[i].i2c_read)
				codec->hw_read = io_types[i].i2c_read;

			codec->control_data = container_of(codec->dev,
							   struct i2c_client,
							   dev);
			break;
```

## snd_soc_7_9_write
```c
static int snd_soc_7_9_write(struct snd_soc_codec *codec, unsigned int reg,
			     unsigned int value)
{
	data[0] = (reg << 1) | ((value >> 8) & 0x0001);
	data[1] = value & 0x00ff;

	return do_hw_write(codec, reg, value, data, 2);
		codec->hw_write(codec->control_data, data, len);	// i2c_master_send
}
```
