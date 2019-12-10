---
title: AudioPolicyManager代码分析
date: 2019-12-10 11:16:00
categories: 
- Android
- audio
---

# new AudioPolicyManager
AudioPolicyService将会new AudioPolicyManager
```c++
void AudioPolicyService::onFirstRef()
{
	mAudioPolicyClient = new AudioPolicyClient(this);
		AudioPolicyClient(AudioPolicyService *service) : mAudioPolicyService(service) {}	// mAudioPolicyService = AudioPolicyService
	mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
		return new AudioPolicyManager(clientInterface);
}
```
<!--more-->
# AudioPolicyManager构造函数
1. 解析audio_policy.conf
2. 加载conf中指定的so，调用open返回hw_device_t给上层
3. 对conf中每一个output都创建playbackThread

```c++
#define AUDIO_POLICY_CONFIG_FILE "/system/etc/audio_policy.conf"
#define AUDIO_POLICY_VENDOR_CONFIG_FILE "/vendor/etc/audio_policy.conf"

AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
	mpClientInterface = clientInterface;

	// 解析audio_policy.conf，添加到mHwModules list中
	loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE || AUDIO_POLICY_CONFIG_FILE);
		data = (char *)load_file(path, NULL);
		loadHwModules(root);
			cnode *node = config_find(root, AUDIO_HW_MODULE_TAG);	// audio_hw_modules
			while (node) {
				loadHwModule(node);
				node = node->next;
			}

	// 对audio_policy.conf中的每个audio_hw_modules进行加载：audio.primary.xxx.so并且调用其中的open函数
	for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);	// AudioPolicyService::AudioPolicyClient::loadHwModule
			sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
				binder = sm->getService(String16("media.audio_flinger"));	// 获取media.audio_flinger service -> AudioFlinger
			return af->loadHwModule(name);	// AudioFlinger::loadHwModule
				return loadHwModule_l(name);
					int rc = load_audio_interface(name, &dev);
						// 1: 加载HAL
						rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
						// 2: 调用hal的open函数，hal的open会返回hw_device_t给上层
						rc = audio_hw_device_open(mod, dev);

		// 每一个output都会创建一个playbackThread
 		for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++) {
			const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];
			sp<AudioOutputDescriptor> outputDesc = new AudioOutputDescriptor(outProfile);
			status_t status = mpClientInterface->openOutput(outProfile->mModule->mHandle,	// AudioFlinger::openOutput
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            String8(""),
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);
		}
	}
}
```
