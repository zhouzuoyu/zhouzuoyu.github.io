---
title: mHwModules
date: 2019-12-09 17:57:00
categories: 
- Android
- audio
---

AudioPolicyManager.mHwModules
存放了audio_policy.conf中所有的HwModule
```c++
class AudioPolicyManager: public AudioPolicyInterface
{
	Vector < sp<HwModule> > mHwModules;
}
```
<!--more-->
# mHwModules来源
audio_policy.conf会放在/vendor/etc/或者/system/etc/下。
将audio_policy.conf的audio_hw_modules都添加到mHwModules中。
```c
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
	loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE || AUDIO_POLICY_CONFIG_FILE);
		loadHwModules(root);
			cnode *node = config_find(root, AUDIO_HW_MODULE_TAG);	// audio_hw_modules
			node = node->first_child;
			while (node) {
				loadHwModule(node);
					// 构建一个module，添加到mHwModules中
					sp<HwModule> module = new HwModule(root->name);	// primary
					mHwModules.add(module);

				node = node->next;
			}
}
```

# mHwModules使用
对每一个mHwModules都加载对应的so文件
```c
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);
	}
}
```
