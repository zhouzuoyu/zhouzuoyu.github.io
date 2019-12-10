---
title: mOutputs
date: 2019-12-09 18:00:00
categories: 
- Android
- audio
---

AudioPolicyManager.mOutputs
mOutputs存放了audio_policy.conf中所有的output
```c++
class AudioPolicyManager: public AudioPolicyInterface
{
	DefaultKeyedVector<audio_io_handle_t, sp<AudioOutputDescriptor> > mOutputs;
}
```
<!--more-->
每个output系统都会对应一个唯一id: nextUniqueId
每个output都会对应一个PlaybackThread
将所有的output都add到mOutputs
```c++
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
	mDefaultOutputDevice = new DeviceDescriptor(String8(""), AUDIO_DEVICE_OUT_SPEAKER);	// 默认输出设备是AUDIO_DEVICE_OUT_SPEAKER

    for (size_t i = 0; i < mHwModules.size(); i++) {
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++) {
			const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];				// 对audio_policy.conf中的每一个output
			sp<AudioOutputDescriptor> outputDesc = new AudioOutputDescriptor(outProfile);	// 每一个output构建AudioOutputDescriptor

			// 支持的设备中包含AUDIO_DEVICE_OUT_SPEAKER，则选用AUDIO_DEVICE_OUT_SPEAKER
			audio_devices_t profileType = outProfile->mSupportedDevices.types();
			if ((profileType & mDefaultOutputDevice->mDeviceType) != AUDIO_DEVICE_NONE) {
                profileType = mDefaultOutputDevice->mDeviceType;
            }			
			outputDesc->mDevice = profileType;	// AUDIO_DEVICE_OUT_SPEAKER
			audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;	// conf中的每一个output都对应唯一的id值
            status_t status = mpClientInterface->openOutput(outProfile->mModule->mHandle,	// AudioFlinger::openOutput
                                                            &output,				// 输出型参数，会进行修改，每一个output都会获得唯一的id
                                                            &config,
                                                            &outputDesc->mDevice,	// AUDIO_DEVICE_OUT_SPEAKER
                                                            String8(""),
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);
				sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
					// 每一个output都会对应一个playbackThreads
					if (*output == AUDIO_IO_HANDLE_NONE) {
						*output = nextUniqueId();	// 每一个output都会获得一个唯一的id
					}
					thread = new MixerThread(this, outputStream, *output, devices);
					mPlaybackThreads.add(*output, thread);	// output对应到PlaybackThread

			addOutput(output, outputDesc);
				mOutputs.add(output, outputDesc);	// 添加到mOutputs中
		}
	}
}
```
