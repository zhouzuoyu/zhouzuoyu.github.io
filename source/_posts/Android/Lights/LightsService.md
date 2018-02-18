---
title: Android Lights(5)_LightsService
date: 2018-02-10 12:00:00
categories: 
- Android
- Lights
---
## 源码

>   base\services\core\java\com\android\server\lights\LightsService.java

## 功能

　　主要对JNI再封装了一层

　　对上：构造了setBrightness接口函数供调用

　　对下：调用JNI对应方法

<!-- more -->

## 源码分析

LightsService构造函数：

　　调用JNI的init方法

```java
public LightsService(Context context) {
    // 调用jni方法
    mNativePointer = init_native();
    
    for (int i = 0; i < LightsManager.LIGHT_ID_COUNT; i++) {
        mLights[i] = new LightImpl(i);
    }
}
```

setBrightness：

　　调用JNI的setLight方法进行背光调节

```java
public void setBrightness(int brightness)
    setBrightness(brightness, BRIGHTNESS_MODE_USER);
        setLightLocked(color, LIGHT_FLASH_NONE, 0, 0, brightnessMode);
            setLight_native(mNativePointer, mId, color, mode, onMS, offMS, brightnessMode); 

```
