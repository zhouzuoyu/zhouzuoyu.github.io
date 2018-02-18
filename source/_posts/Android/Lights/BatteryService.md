---
title: Android Lights(8)_BatteryService
date: 2018-02-10 12:00:00
categories: 
- Android
- Lights
---
# 源码
base\services\core\java\com\android\server\BatteryService.java

# 源码分析
1. 灯颜色
  ```java
  public BatteryService(Context context)
  	mLed = new Led(context, getLocalService(LightsManager.class));	// call public Led(Context context, LightsManager lights)
  		mBatteryLight = lights.getLight(LightsManager.LIGHT_ID_BATTERY);
  		mBatteryLowARGB = context.getResources().getInteger(
  	            com.android.internal.R.integer.config_notificationsBatteryLowARGB);		// 0xFFFF0000
  	    mBatteryMediumARGB = context.getResources().getInteger(
  	            com.android.internal.R.integer.config_notificationsBatteryMediumARGB);	// 0xFFFFFF00
  	    mBatteryFullARGB = context.getResources().getInteger(
  	            com.android.internal.R.integer.config_notificationsBatteryFullARGB);	// 0xFF00FF00
  ```
  <!--more-->

2. 电量显示
  充电中:
  　　低电量(<15%)：mBatteryLowARGB

  　　高电量(>= 90): mBatteryFullARGB

  　　other: mBatteryMediumARGB
  没有充电：
  　　低电量(<15%)：闪烁

  ```java
  processValuesLocked
  	mLed.updateLightsLocked();
  		if (level < mLowBatteryWarningLevel) {
  	        if (status == BatteryManager.BATTERY_STATUS_CHARGING) {
  	            // Solid red when battery is charging
  	            mBatteryLight.setColor(mBatteryLowARGB);
  	        } else {
  	            // Flash red when battery is low and not charging
  	            mBatteryLight.setFlashing(mBatteryLowARGB, Light.LIGHT_FLASH_TIMED,
  	                    mBatteryLedOn, mBatteryLedOff);
  	        }
  	    } else if (status == BatteryManager.BATTERY_STATUS_CHARGING
  	            || status == BatteryManager.BATTERY_STATUS_FULL) {
  	        if (status == BatteryManager.BATTERY_STATUS_FULL || level >= 90) {
  	            // Solid green when full or charging and nearly full
  	            mBatteryLight.setColor(mBatteryFullARGB);
  	        } else {
  	            // Solid orange when charging and halfway full
  	            mBatteryLight.setColor(mBatteryMediumARGB);
  	        }
  	    }
  ```
