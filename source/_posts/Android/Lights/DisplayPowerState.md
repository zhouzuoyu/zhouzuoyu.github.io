---
title: Android Lights(4)_DisplayPowerState
date: 2018-02-10 12:00:00
categories: 
- Android
- Lights
---

## 源码

> base\services\core\java\com\android\server\display\DisplayPowerState.java

## 功能

　　对LightsService封装了一层

　　对上：提供mProperty.setValue方法

　　对下：调用LightsService.setBrightness方法，设置brightness
<!-- more -->
## 源码分析

　　构建了mProperty.setValue这个方法供调用，传入brightness值
```java
public static final IntProperty<DisplayPowerState> SCREEN_BRIGHTNESS =
        new IntProperty<DisplayPowerState>("screenBrightness") {
    public void setValue(DisplayPowerState object, int value)
        object.setScreenBrightness(value);    // DisplayPowerState.setScreenBrightness
            mScreenBrightness = brightness;
            scheduleScreenUpdate();
                postScreenUpdateThreadSafe();
                    mHandler.post(mScreenUpdateRunnable);    // 执行mScreenUpdateRunnable.run()
                        brightness = mScreenBrightness;
                        mPhotonicModulator.setState(mScreenState, brightness);
                            mPendingBacklight = backlight;    // brightness最终保存在PhotonicModulator.mPendingBacklight
                            mLock.notifyAll();    // 唤醒PhotonicModulator线程
```
notifyAll将唤醒PhotonicModulator线程，最终调用LightsService.setBrightness，请查看LightsService篇
```java
public void run() {
    for (;;) {
        backlight = mPendingBacklight;
        backlightChanged = (backlight != mActualBacklight);
        if (!stateChanged && !backlightChanged) {
               mLock.wait();    // 等待
            continue;
        }
        mActualBacklight = backlight;    // 更新数据

        if (backlightChanged)
            setBrightness(backlight);
                mBacklight.setBrightness(backlight);    // mLights[LIGHT_ID_BACKLIGHT].setBrightness(backlight)
    }
}
```
