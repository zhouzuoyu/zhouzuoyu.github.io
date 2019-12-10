---
title: AudioTrack选择output
date: 2019-12-04 14:10:00
categories: 
- Android
- audio
---

AudioTrack传入指定streamType，最终会选择哪个output。

# streamType -> strategy对应表
| audio_stream_type_t						| audio_attributes_t																			|routing_strategy|
| :-----------------------------			| :-----------------------------																| :-				|
|AUDIO_STREAM_DEFAULT<br>AUDIO_STREAM_MUSIC	|content_type = AUDIO_CONTENT_TYPE_MUSIC<br>usage = AUDIO_USAGE_MEDIA							|STRATEGY_MEDIA
|AUDIO_STREAM_VOICE_CALL					|content_type = AUDIO_CONTENT_TYPE_SPEECH<br>usage = AUDIO_USAGE_VOICE_COMMUNICATION			|STRATEGY_PHONE
|AUDIO_STREAM_ENFORCED_AUDIBLE				|flags &#124;= AUDIO_FLAG_AUDIBILITY_ENFORCED<br>content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_ASSISTANCE_SONIFICATION		|STRATEGY_ENFORCED_AUDIBLE
|AUDIO_STREAM_SYSTEM						|content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_ASSISTANCE_SONIFICATION 	|STRATEGY_MEDIA
|AUDIO_STREAM_RING							|content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_NOTIFICATION_TELEPHONY_RINGTONE |	STRATEGY_SONIFICATION
|AUDIO_STREAM_ALARM							|content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_ALARM 					|STRATEGY_SONIFICATION
|AUDIO_STREAM_NOTIFICATION					|content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_NOTIFICATION				|STRATEGY_SONIFICATION_RESPECTFUL
|AUDIO_STREAM_BLUETOOTH_SCO					|content_type = AUDIO_CONTENT_TYPE_SPEECH<br>usage = AUDIO_USAGE_VOICE_COMMUNICATION<br>flags &#124;= AUDIO_FLAG_SCO;		|STRATEGY_PHONE
|AUDIO_STREAM_DTMF							|content_type = AUDIO_CONTENT_TYPE_SONIFICATION<br>usage = AUDIO_USAGE_VOICE_COMMUNICATION_SIGNALLING	|STRATEGY_DTMF
|AUDIO_STREAM_TTS							|content_type = AUDIO_CONTENT_TYPE_SPEECH<br>usage = AUDIO_USAGE_ASSISTANCE_ACCESSIBILITY 				|STRATEGY_MEDIA
<!--more-->
# strategy对应output
**以STRATEGY_MEDIA为例**
因为availableOutputDeviceTypes为AUDIO_DEVICE_OUT_SPEAKER，所以最终的device为AUDIO_DEVICE_OUT_SPEAKER
再search所有的outputs，挑选最匹配的output
1: the output with the highest number of requested policy flags
2: the primary output
3: the first output in the list

# 源码分析
1: streamType对应Attribute(参照对应表)
```c++
sp<AudioTrack> track = new AudioTrack(AUDIO_STREAM_MUSIC,// stream type
       rate,
       AUDIO_FORMAT_PCM_16_BIT,// word length, PCM
       AUDIO_CHANNEL_OUT_MONO,
       iMem);
	mStatus = set(streamType, sampleRate, format, channelMask,
			0 /*frameCount*/, flags, cbf, user, notificationFrames,
			sharedBuffer, false /*threadCanCallJava*/, sessionId, transferType, offloadInfo,
			uid, pid, pAttributes);
		// 1. streamType对应Attribute
		setAttributesFromStreamType(streamType);
			mAttributes.content_type = AUDIO_CONTENT_TYPE_MUSIC;
			mAttributes.usage = AUDIO_USAGE_MEDIA;
```

2: Attribute对应strategy(参照对应表)
3: strategy对应device
	参照getDeviceForStrategy和audio_policy.conf
4: device对应output
	从audio_policy.conf中挑选出满足device条件最佳的output
```c++
audio_io_handle_t AudioPolicyManager::getOutputForAttr(const audio_attributes_t *attr,
                                    uint32_t samplingRate,
                                    audio_format_t format,
                                    audio_channel_mask_t channelMask,
                                    audio_output_flags_t flags,
                                    const audio_offload_info_t *offloadInfo)
{
	// 2: Attribute对应strategy
	routing_strategy strategy = (routing_strategy) getStrategyForAttr(attr);		// AUDIO_USAGE_MEDIA, 返回STRATEGY_MEDIA
		switch (attr->usage) {
		case AUDIO_USAGE_MEDIA:
			return (uint32_t) STRATEGY_MEDIA;
		}
	// 3: strategy对应device
	audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);	// STRATEGY_MEDIA，返回AUDIO_DEVICE_OUT_SPEAKER
		uint32_t device = AUDIO_DEVICE_NONE;
		audio_devices_t availableOutputDeviceTypes = mAvailableOutputDevices.types();	// AUDIO_DEVICE_OUT_SPEAKER
		device2 = availableOutputDeviceTypes & AUDIO_DEVICE_OUT_SPEAKER;	// 最终是AUDIO_DEVICE_OUT_SPEAKER
		int device3 = AUDIO_DEVICE_NONE;
		device2 |= device3;
		device |= device2;	// 最终是AUDIO_DEVICE_OUT_SPEAKER
	// 4: device对应output
	return getOutputForDevice(device, stream, samplingRate, format, channelMask, flags, offloadInfo);
		// 匹配conf中所有包含AUDIO_DEVICE_OUT_SPEAKER的outputs
		SortedVector<audio_io_handle_t> outputs = getOutputsForDevice(device, mOutputs);
			// 
			for (size_t i = 0; i < openOutputs.size(); i++) {
				if ((device & openOutputs.valueAt(i)->supportedDevices()) == device) {
					outputs.add(openOutputs.keyAt(i));
				}
			}
			return outputs;
		flags = (audio_output_flags_t)(flags & ~AUDIO_OUTPUT_FLAG_DIRECT);
		// 从outputs中选一个最合适的output
		output = selectOutput(outputs, flags, format);
			for (size_t i = 0; i < outputs.size(); i++) {
				// 1: 找flags匹配度最高的output
				sp<AudioOutputDescriptor> outputDesc = mOutputs.valueFor(outputs[i]);
				int commonFlags = popcount(outputDesc->mProfile->mFlags & flags);
				if (commonFlags > maxCommonFlags) {
					outputFlags = outputs[i];
					maxCommonFlags = commonFlags;
				}
				// 2: 找flags为AUDIO_OUTPUT_FLAG_PRIMARY的output
				if (outputDesc->mProfile->mFlags & AUDIO_OUTPUT_FLAG_PRIMARY) {
					outputPrimary = outputs[i];
				}
			}

			// 3: 不满足1/2的话则选择第一个output
			return outputs[0];
}
```
