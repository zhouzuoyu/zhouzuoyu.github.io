---
title: AudioFlinger加载hal
date: 2019-12-06 17:51:00
categories: 
- Android
- audio
---

# 概述
a. 加载对应audio.[primary/a2dp...].xxx.so
&emsp;例如primary，在/vendor/lib64/hw/和/system/lib64/hw/依次检查
&emsp;**audio.primary.<ro.hardware.audio.primary>.so**
&emsp;**audio.primary.<ro.hardware>.so**
&emsp;**audio.primary.<ro.product.board>.so**
&emsp;**audio.primary.<ro.board.platform>.so**
&emsp;**audio.primary.<ro.arch>.so**
&emsp;**audio.primary.default.so**
b. 调用audio.[primary/a2dp...].xxx.so的open函数，返回hal的audio_hw_device
<!--more-->
# 源码分析
```cpp
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
	return loadHwModule_l(name);
		load_audio_interface(name, &dev);
			// Part 1: 加载对应audio.[primary/a2dp...].xxx.so
			hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);	// if_name = primary
				snprintf(name, PATH_MAX, "%s.%s", class_id, inst);	// audio.primary
				snprintf(prop_name, sizeof(prop_name), "ro.hardware.%s", name);	// ro.hardware.audio.primary
				// 先检查audio.primary.tiny4412.so
				if (property_get(prop_name, prop, NULL) > 0) {	// 看这个属性值ro.hardware.audio.primary，例如这个值是tiny4412
					if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
						// 先在/vendor/lib64/hw/下看有没有audio.primary.tiny4412.so
						snprintf(path, path_len, "%s/%s.%s.so", HAL_LIBRARY_PATH2, name, subname);	// /vendor/lib64/hw/audio.primary.tiny4412.so
						if (access(path, R_OK) == 0)
								return 0;
						// 再在/system/lib64/hw/下看有没有audio.primary.tiny4412.so
						snprintf(path, path_len, "%s/%s.%s.so",
								 HAL_LIBRARY_PATH1, name, subname);
						if (access(path, R_OK) == 0)
							return 0;

						goto found;
					}
				}
				// 再检查audio.primary.<ro.hardware>.so/audio.primary.<ro.product.board>.so/audio.primary.<ro.board.platform>.so/audio.primary.<ro.arch>.so
				for (i=0 ; i<HAL_VARIANT_KEYS_COUNT; i++) {
					if (property_get(variant_keys[i], prop, NULL) == 0) {
						continue;
					}
					if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
						goto found;
					}
				}
				// 都没有，则检查audio.primary.default.so
				/* Nothing found, try the default */
				if (hw_module_exists(path, sizeof(path), name, "default") == 0) {
					goto found;
				}

				// load指定so
				return load(class_id, path, module);

			// Part 2: 调用audio.[primary/a2dp...].xxx.so的open函数
			// 返回hal的audio_hw_device
			audio_hw_device_open(mod, dev);
				return module->methods->open(module, AUDIO_HARDWARE_INTERFACE,
											(struct hw_device_t**)device);

		
		audio_module_handle_t handle = nextUniqueId();
		mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));	// 添加到这个list

		return handle;
}
```
