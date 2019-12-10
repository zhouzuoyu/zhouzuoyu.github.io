---
title: audio_policy.conf解析
date: 2019-12-09 17:01:00
categories: 
- Android
- audio
---

这个文件中有很多hw_module，每个hw_module又可以有很多output和input.
**每个hw_module对应一个so，每个output对应一个playbackThread.**

# audio_policy.conf
hw_modules
&emsp;hw_module1
&emsp;&emsp;outputs
&emsp;&emsp;&emsp;output1
&emsp;&emsp;&emsp;output2
&emsp;&emsp;&emsp;output...
&emsp;&emsp;inputs
&emsp;&emsp;&emsp;input1
&emsp;&emsp;&emsp;input2
&emsp;&emsp;&emsp;input...
&emsp;hw_module2
&emsp;hw_module...
<!--more-->
```Makefile
audio_hw_modules {	// hw_modules，这个conf文件中可以有很多hw_module
  primary {		// 其中的一个hw_module，可以有很多hw_module
    outputs {		// outputs，每个hw_module可以有很多output
      primary {		// 其中的一个output
        sampling_rates 44100
        channel_masks AUDIO_CHANNEL_OUT_STEREO
        formats AUDIO_FORMAT_PCM_16_BIT
        devices AUDIO_DEVICE_OUT_SPEAKER|AUDIO_DEVICE_OUT_EARPIECE|AUDIO_DEVICE_OUT_WIRED_HEADSET|AUDIO_DEVICE_OUT_WIRED_HEADPHONE|AUDIO_DEVICE_OUT_ALL_SCO|AUDIO_DEVICE_OUT_AUX_DIGITAL	// 解析到mSupportedDevices
        flags AUDIO_OUTPUT_FLAG_PRIMARY
      }
    }
    inputs {		// inputs，每个hw_module可以有很多input
      primary {		// 其中的一个input
        sampling_rates 8000|11025|12000|16000|22050|24000|32000|44100|48000
        channel_masks AUDIO_CHANNEL_IN_MONO|AUDIO_CHANNEL_IN_STEREO
        formats AUDIO_FORMAT_PCM_16_BIT
        devices AUDIO_DEVICE_IN_BUILTIN_MIC|AUDIO_DEVICE_IN_WIRED_HEADSET|AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET|AUDIO_DEVICE_IN_AUX_DIGITAL|AUDIO_DEVICE_IN_VOICE_CALL
      }
    }
  }
}
```
# 代码分析
每个output都对应一个IOProfile
output.devices保存在IOProfile.mSupportedDevices
```c++
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
	loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE);	// audio_policy.conf
		loadHwModules(root);
			// 解析所有的audio_hw_modules
			cnode *node = config_find(root, AUDIO_HW_MODULE_TAG);	// "audio_hw_modules"
			while (node) {
				loadHwModule(node);
					// 解析所有的outputs
					node = config_find(root, OUTPUTS_TAG);			// "outputs"
					node = node->first_child;
					while (node) {
						status_t tmpStatus = module->loadOutput(node);
							sp<IOProfile> profile = new IOProfile(String8(root->name), AUDIO_PORT_ROLE_SOURCE, this);	// 每一个output都会对应一个IOProfile
							if (strcmp(node->name, DEVICES_TAG) == 0) {	// "devices"
								profile->mSupportedDevices.loadDevicesFromName((char *)node->value,
										                                       mDeclaredDevices);
							}

						node = node->next;
					}

				node = node->next;
			}
}
```
