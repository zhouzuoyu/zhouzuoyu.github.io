---
title: mOutputProfiles
date: 2019-12-09 15:47:00
categories: 
- Android
- audio
---

AudioPolicyManager.mHwModules.mOutputProfiles
```c++
class AudioPolicyManager: public AudioPolicyInterface
{
public:
	Vector < sp<HwModule> > mHwModules;
}

class HwModule : public RefBase
{
public:
    Vector < sp<IOProfile> > mOutputProfiles; // output profiles exposed by this module
    Vector < sp<IOProfile> > mInputProfiles;  // input profiles exposed by this module
};
audio_policy.conf中的每一个output都会创建一个OutputProfiles与之对应。
```
<!--more-->
# mOutputProfiles来源
```c++
void AudioPolicyManager::loadHwModule(cnode *root)
{
	sp<HwModule> module = new HwModule(root->name);
    node = config_find(root, OUTPUTS_TAG);
    if (node != NULL) {
        node = node->first_child;
        while (node) {
            status_t tmpStatus = module->loadOutput(node);
				// 对audio_policy.conf中的每一个output构建IOProfile
				sp<IOProfile> profile = new IOProfile(String8(root->name), AUDIO_PORT_ROLE_SOURCE, this);
				...
				mOutputProfiles.add(profile);

            node = node->next;
        }
    }

	mHwModules.add(module);
}
```

# mOutputProfiles使用
对mHwModules中的每一个mOutputProfiles都调用openOutput
```c++
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);

        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++) {
            const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];
			audio_devices_t profileType = outProfile->mSupportedDevices.types();

            status_t status = mpClientInterface->openOutput(outProfile->mModule->mHandle,
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            String8(""),
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

		}
}
```
