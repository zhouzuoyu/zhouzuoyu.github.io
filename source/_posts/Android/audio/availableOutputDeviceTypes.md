---
title: availableOutputDeviceTypes
date: 2019-12-09 15:46:00
categories: 
- Android
- audio
---

mAvailableOutputDevices.type来自audio_policy.conf的global_configuration.attached_output_devices

audio_policy.conf
```
global_configuration {
  attached_output_devices AUDIO_DEVICE_OUT_SPEAKER
  default_output_device AUDIO_DEVICE_OUT_SPEAKER
  attached_input_devices AUDIO_DEVICE_IN_BUILTIN_MIC
}
```
<!--more-->

解析ATTACHED_OUTPUT_DEVICES_TAG
```c++
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
	loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE);
		loadGlobalConfig(root, module);
			if (strcmp(ATTACHED_OUTPUT_DEVICES_TAG, node->name) == 0) {	// attached_output_devices
		        mAvailableOutputDevices.loadDevicesFromName((char *)node->value,
		                                                    declaredDevices);
					add(deviceDesc);
		    }
}
```
