---
title: Android Lights(2)_PowerManagerService
date: {{ date }}
categories: 
- Android
- Lights
---

## 源码

>   base\services\core\java\com\android\server\power\PowerManagerService.java

## 功能

>   注册了ContentObserver来监测背光值，数据变动将启动背光调节

## 源码分析
<!-- more -->
注册ContentObserver观察用户是否调节了背光

```java
public void systemReady(IAppOpsService appOps)
    mDisplayManagerInternal = getLocalService(DisplayManagerInternal.class);
    mSettingsObserver = new SettingsObserver(mHandler);    // 注册一个ContentObserver
    // 将Settings.System.SCREEN_BRIGHTNESS放入到ContentObserver进行监视
    resolver.registerContentObserver(Settings.System.getUriFor(
            Settings.System.SCREEN_BRIGHTNESS),
            false, mSettingsObserver, UserHandle.USER_ALL);
```

被观察的数据有变动时将被调用，最终调用DisplayPowerController.requestPowerState

```java
public void onChange(boolean selfChange, Uri uri)    // ContentObserver数据变动
    handleSettingsChangedLocked();
        updateSettingsLocked();
            // 得到背光值
            mScreenBrightnessSetting = Settings.System.getIntForUser(resolver,
                    Settings.System.SCREEN_BRIGHTNESS, mScreenBrightnessSettingDefault,
                    UserHandle.USER_CURRENT);
        updatePowerStateLocked();
            updateDisplayPowerStateLocked(dirtyPhase2);
                screenBrightness = mScreenBrightnessSetting;
                mDisplayPowerRequest.screenBrightness = screenBrightness;    // 存brightness值
                mDisplayManagerInternal.requestPowerState(mDisplayPowerRequest, mRequestWaitForNegativeProximity);    // DisplayManagerService.requestPowerState
```
