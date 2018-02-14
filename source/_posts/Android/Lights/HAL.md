---
title: Android Lights(7)_Lights HAL
date: {{ date }}
categories: 
- Android
- Lights
---

# 功能
　　生成lights.xxx.so文件
　　构造所有硬件支持的light设备(主要在于set_light不同)供上层调用

# 源码分析
LIGHTS_HARDWARE_MODULE_ID是light HAL的唯一标示，JNI通过该ID load对应的so文件。
主要关注lights_module_methods
<!-- more -->
```c
struct hw_module_t HAL_MODULE_INFO_SYM = {
    .id = LIGHTS_HARDWARE_MODULE_ID,    // 唯一标示
    .methods = &lights_module_methods,
};
static struct hw_module_methods_t lights_module_methods = {
    .open =  open_lights,
};
  ```

## open_lights
  　　构造硬件支持light设备给予上层使用，不同device设置不同的set_light回调函数
```c

static int open_lights(const struct hw_module_t* module, char const* name, struct hw_device_t** device)
// 不同类型的light设置对应的set_light
if (0 == strcmp(LIGHT_ID_BACKLIGHT, name))
	set_light = set_light_backlight;
else if (0 == strcmp(LIGHT_ID_NOTIFICATIONS, name))
	set_light = set_light_notifications;
else if (0 == strcmp(LIGHT_ID_ATTENTION, name))
	set_light = set_light_attention;
dev->set_light = set_light; // 设置set_light

*device = (struct hw_device_t*)dev; // 输出型参数
```

## set_light
   　　以背光为例，Framework传递下来的参数经转化后写入“/sys/class/leds/lcd-backlight/brightness”这个节点(该节点由kernel创建)，从而达到改变背光的目的。
```c
static int set_light_backlight(struct light_device_t* dev, 
                             struct light_state_t const* state)
// 计算brightness，上层是0-255，底层为0-1023
int brightness = rgb_to_brightness(state);    // state->color中有brightness值

// 将brightness写入到brightness节点
write_int(LCD_FILE, brightness);    // "/sys/class/leds/lcd-backlight/brightness"
fd = open(path, O_RDWR);
int bytes = sprintf(buffer, "%d\n", value);    // int -> char
int amt = write(fd, buffer, bytes);
close(fd);
```
