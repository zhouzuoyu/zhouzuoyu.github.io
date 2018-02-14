---
title: Android Lights(1)_Overview
date: {{ date }}
categories: 
- Android
- Lights
---

以背光调节阐述：

> {% post_link Android/Lights/PowerManagerService PowerManagerService%}

　　使用ContentObserver对Settings.System.SCREEN_BRIGHTNESS进行观测，数据变动则启动一次背光调节

> {% post_link Android/Lights/DisplayPowerController DisplayPowerController%}

　　使用animateScreenBrightness算法（渐变）计算出背光值，调用DisplayPowerState提供的接口进行调节
<!-- more -->
> {% post_link Android/Lights/DisplayPowerState DisplayPowerState%}

　　接收到背光值，调用LightService提供的接口进行调节

> {% post_link Android/Lights/LightsService LightsService%}

　　调用JNI提供的setLight_native进行调节

> {% post_link Android/Lights/JNI JNI%}

　　load HAL层so库，调用backlight类light device的set_light进行背光调节

> {% post_link Android/Lights/HAL HAL%}

　　配置硬件上支持的light device的set_light函数，例如backlight，写入背光值到"/sys/class/leds/lcd-backlight/brightness"
