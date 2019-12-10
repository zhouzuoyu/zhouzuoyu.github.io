---
title: Audio HAL
date: 2019-12-09 13:50:00
categories: 
- Android
- audio
---

# HAL代码结构
1. hardware\libhardware_legacy\audio\Audio_hw_hal.cpp
	框架代码，给上层提供统一的接口
2. device\friendly-arm\common\libaudio\AudioHardware.cpp
	具体的hal实现代码

# HAL代码分析
1. load so文件
2. 通过步骤1得到outputStream
3. 通过步骤2操作pcm流<!--more-->

## APM loadHwModule
APM的构造函数会调用HAL的open函数：APM mpClientInterface->loadHwModule(mHwModules[i]->mName)
**HAL会把构建的legacy_audio_device.audio_hw_device返回给上层**
上层可以利用这个audio_hw_device得到legacy_audio_device，可以得到AudioHardwareInterface，从而可以操控硬件
```c++
struct legacy_audio_device {
    struct audio_hw_device device;			// 返回给上层

    struct AudioHardwareInterface *hwif;	// 厂家提供的实现函数
};
```
**这个结构体很关键，因为其中的成员audio_hw_device会返回给上层，从而可以操作硬件**

```c++
static struct hw_module_methods_t legacy_audio_module_methods = {
        open: legacy_adev_open
};

struct legacy_audio_module HAL_MODULE_INFO_SYM = {
    module: {
        common: {
            tag: HARDWARE_MODULE_TAG,
            module_api_version: AUDIO_MODULE_API_VERSION_0_1,
            hal_api_version: HARDWARE_HAL_API_VERSION,
            id: AUDIO_HARDWARE_MODULE_ID,
            name: "LEGACY Audio HW HAL",
            methods: &legacy_audio_module_methods,
        },
    },
};

static int legacy_adev_open(const hw_module_t* module, const char* name,
                            hw_device_t** device)	// 输出型参数
{
	struct legacy_audio_device *ladev;
	ladev = (struct legacy_audio_device *)calloc(1, sizeof(*ladev));	// 构建legacy_audio_device
    ladev->device.open_output_stream = adev_open_output_stream;
	*device = &ladev->device.common;	// 把构建legacy_audio_device.audio_hw_device返回给上层

    ladev->hwif = createAudioHardware();	// 厂家的实现函数
		return new AudioHardware();
}
```

## APM openOutput
legacy_stream_out中包含两类结构体audio_stream_out和AudioStreamOut
```c++
struct legacy_stream_out {
    struct audio_stream_out stream;	// 返回给上层

    AudioStreamOut *legacy_out;		// 厂家提供：audioHardware.cpp
};
```

**APM的openOutput将会调用HAL的open_output_stream，返回audio_stream_out，上层拿到这个结构体可以操作pcm流**
```c++
static int adev_open_output_stream(struct audio_hw_device *dev,
                                   audio_io_handle_t handle,
                                   audio_devices_t devices,
                                   audio_output_flags_t flags,
                                   struct audio_config *config,
                                   struct audio_stream_out **stream_out,	// 输出型参数
                                   const char *address __unused)
{
	out = (struct legacy_stream_out *)calloc(1, sizeof(*out));

	out->legacy_out = ladev->hwif->openOutputStreamWithFlags(devices, flags,
                                                    (int *) &config->format,
                                                    &config->channel_mask,
                                                    &config->sample_rate, &status);
		return openOutputStream(devices, format, channels, sampleRate, status);	// AudioHardware::openOutputStream
			out = new AudioStreamOutALSA();
			rc = out->set(this, devices, format, channels, sampleRate);
			return out.get();

	out->stream.write = out_write;

	*stream_out = &out->stream;
}

static ssize_t out_write(struct audio_stream_out *stream, const void* buffer,
                         size_t bytes)
{
    struct legacy_stream_out *out =
        reinterpret_cast<struct legacy_stream_out *>(stream);	// 根据小结构体得到大结构体，从而可以获取到audioHardware
    return out->legacy_out->write(buffer, bytes);	// AudioHardware::AudioStreamOutALSA::write
}
```

## PlaybackThread write
将调用AudioHardware::AudioStreamOutALSA::write

第一次write时将调用open函数，打开对应驱动设备
```c++
status_t AudioHardware::AudioStreamOutALSA::open_l()
{
	mPcm = mHardware->openPcmOut_l();
        struct pcm_config config = {
            channels : 2,
            rate : AUDIO_HW_OUT_SAMPLERATE,
            period_size : AUDIO_HW_OUT_PERIOD_SZ,
            period_count : AUDIO_HW_OUT_PERIOD_CNT,
            format : PCM_FORMAT_S16_LE,
            start_threshold : 0,
            stop_threshold : 0,
            silence_threshold : 0,
        };
		mPcm = pcm_open(0, 0, flags, &config);	// tinyalsa的接口，Card 0, Device 0
		return mPcm;

	mMixer = mHardware->openMixer_l();

	// 设置通路：根据mDevices设置不同的kctl
	const AudioMixer *mixer = mHardware->getOutputRouteFromDevice(mDevices);
    for(int cnt = 0 ; NULL != mixer[cnt].ctl ; ++cnt){
        mRouteCtl = mixer_get_ctl_by_name(mMixer, mixer[cnt].ctl);
        if (NULL != mRouteCtl) {
            for (unsigned int idx = 0 ; mixer_ctl_get_num_values(mRouteCtl) > idx ; ++idx)
                mixer_ctl_set_value(mRouteCtl, idx, mixer[cnt].val);
        }
    }
}
```

调用pcm_write
```c++
ssize_t AudioHardware::AudioStreamOutALSA::write(const void* buffer, size_t bytes)
{
	const uint8_t* p = static_cast<const uint8_t*>(buffer);
	// 第一次open
	if (mStandby) {
		open_l();
		mStandby = false;
	}

	ret = pcm_write(mPcm,(void*) p, bytes);		// 调用tinyalsa的接口
}
```
