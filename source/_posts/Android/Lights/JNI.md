---
title: Android Lights(6)_Lights JNI
date: 2018-02-10 12:00:00
categories: 
- Android
- Lights
---

## 源码地址

>   base\services\core\jni\com_android_server_lights_LightsService.cpp

## 功能

定义JNI接口函数供Framework调用，主要设置init_native和setLight_native

>   **init_native：获得HAL层支持的light devices，保存在devices中**
>   **setLight_native：配置不同light设备的set_light方法**

<!-- more -->

## 源码分析

申明JNI接口函数，关注init_native和setLight_native

```c
static JNINativeMethod method_table[] = {
	{ "init_native", "()J", (void*)init_native },
	{ "finalize_native", "(J)V", (void*)finalize_native },
	{ "setLight_native", "(JIIIIII)V", (void*)setLight_native },
};
```

### init_native
1：load HAL的so文件
2：依次查找支持的light设备，并且保存在devices->lights[]中（其中包含了set_light方法）
3：返回devices
> devices中包含了各类lights设备

```c
static jlong init_native(JNIEnv *env, jobject clazz)
{
	// load HAL so
	hw_get_module(LIGHTS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
	// devices->lights[]中存储了HAL不同lights设备，用BACKLIGHT举例
	devices->lights[LIGHT_INDEX_BACKLIGHT] = get_device(module, LIGHT_ID_BACKLIGHT);
		module->methods->open(module, name, &device);　　// 将会call HAL的open_lights，返回HAL的light_device_t，其中包含set_light方法
		return (light_device_t*)device;
	devices->lights[LIGHT_INDEX_NOTIFICATIONS] = get_device(module, LIGHT_ID_NOTIFICATIONS);
	...

	return (jlong)devices;
}
```

### setLight_native
配置各类light设备的set_light方法
```c
static void setLight_native(JNIEnv *env, jobject clazz, jlong ptr, jint light, 
                            jint colorARGB, jint flashMode, jint onMS, jint offMS, jint brightnessMode)
{
	Devices* devices = (Devices*)ptr;
	light_state_t state;

	// 设置
	memset(&state, 0, sizeof(light_state_t));
	state.color = colorARGB;
	state.flashMode = flashMode;
	state.flashOnMS = onMS;
	state.flashOffMS = offMS;
	state.brightnessMode = brightnessMode;

	// 调用HAL set_light
	devices->lights[light]->set_light(devices->lights[light], &state);
}
```
