---
title: Android Sensors(2)_HAL测试程序
date: {{ date }}
categories: 
- Android
- Sensors
---

# 源码
> hardware\libhardware\tests\nusensors\Nusensors.cpp

# 源码分析
* load HAL so
* poll所有可用的sensors数据
<!-- more -->

```c++
// 测试HAL层代码使用
int main(int argc, char** argv)
	// load对应so
    hw_get_module(SENSORS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
    
    sensors_open(&module->common, &device);
    	return module->methods->open(module, SENSORS_HARDWARE_POLL, (struct hw_device_t**)device);
    count = module->get_sensors_list(module, &list);    // get_list，放入到list中
    // 0 -> 1
    for (int i=0 ; i<count ; i++)
        device->activate(device, list[i].handle, 0);
    for (int i=0 ; i<count ; i++) {
        device->activate(device, list[i].handle, 1);
        device->setDelay(device, list[i].handle, ms2ns(10));
    }
    
    /*
     * 1：poll获得数据
     * 2：数据逐个显示
     */
    do {
        int n = device->poll(device, buffer, numEvents);	// 数据放入buffer
        
        for (int i=0 ; i<n ; i++) {
            const sensors_event_t& data = buffer[i];

            switch(data.type) {
                case SENSOR_TYPE_ACCELEROMETER:
                case SENSOR_TYPE_MAGNETIC_FIELD:
                case SENSOR_TYPE_ORIENTATION:
                case SENSOR_TYPE_GYROSCOPE:
                case SENSOR_TYPE_GRAVITY:
                case SENSOR_TYPE_LINEAR_ACCELERATION:
                case SENSOR_TYPE_ROTATION_VECTOR:
                    printf("sensor=%s, time=%lld, value=<%5.1f,%5.1f,%5.1f>\n",
                            getSensorName(data.type),
                            data.timestamp,
                            data.data[0],
                            data.data[1],
                            data.data[2]);
                    break;

                case SENSOR_TYPE_LIGHT:
                case SENSOR_TYPE_PRESSURE:
                case SENSOR_TYPE_TEMPERATURE:
                case SENSOR_TYPE_PROXIMITY:
                case SENSOR_TYPE_RELATIVE_HUMIDITY:
                case SENSOR_TYPE_AMBIENT_TEMPERATURE:
                    printf("sensor=%s, time=%lld, value=%f\n",
                            getSensorName(data.type),
                            data.timestamp,
                            data.data[0]);
                    break;

                default:
                    printf("sensor=%d, time=%lld, value=<%f,%f,%f, ...>\n",
                            data.type,
                            data.timestamp,
                            data.data[0],
                            data.data[1],
                            data.data[2]);
                    break;
            }
        }
    } while (1); // fix that
    
    // close
    for (int i=0 ; i<count ; i++)
		device->activate(device, list[i].handle, 0);
	sensors_close(device);
		return device->common.close(&device->common);
```
