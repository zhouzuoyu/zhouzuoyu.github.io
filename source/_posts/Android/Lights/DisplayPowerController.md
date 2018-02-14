---
title: Android Lights(3)_DisplayPowerController
date: {{ date }}
categories: 
- Android
- Lights
---

## 源码

> base\services\core\java\com\android\server\display\DisplayPowerController.java

## 功能

　　对上：提供requestPowerState方法供调用

　　对下：调用mProperty.setValue进行调节（参见DisplayPowerState篇）
<!-- more -->
## 源码分析

　　取出brightness值，使用animateScreenBrightness渐变算法计算需要设置的背光值，进行背光设置
```java
public boolean requestPowerState(DisplayPowerRequest request, boolean waitForNegativeProximity)
    sendUpdatePowerStateLocked();
        Message msg = mHandler.obtainMessage(MSG_UPDATE_POWER_STATE);
        mHandler.sendMessage(msg);
            updatePowerState();
                brightness = clampScreenBrightness(mPowerRequest.screenBrightness);    // 取出brightness值
                animateScreenBrightness(brightness, slowChange ? BRIGHTNESS_RAMP_RATE_SLOW : BRIGHTNESS_RAMP_RATE_FAST);    // 渐变
                    mScreenBrightnessRampAnimator.animateTo(target, rate);
                        mProperty.setValue(mObject, target);    // SCREEN_BRIGHTNESS.setValue
``` 
