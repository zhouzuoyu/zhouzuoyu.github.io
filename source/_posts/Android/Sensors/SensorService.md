---
title: Android Sensors(4)_SensorService
date: 2018-01-29 17:55:15
categories: 
- Android
- Sensors
---

共同构建了SensorService，在Android sensor框架中起承上启下作用，上挂接JNI，下对接sensor HAL。

## SensorDevice.cpp

>native\services\sensorservice\SensorDevice.cpp

功能：

>封装sensor HAL层接口

<!-- more -->

分析：

使用单例模式，其他code只能通过SensorDevice::getInstance()来进行引用

```c++
ANDROID_SINGLETON_STATIC_INSTANCE(SensorDevice)
```

构造函数：

　　load HAL so库，做一些基础init

```c++
SensorDevice::SensorDevice()
  	// load HAL so
	hw_get_module(SENSORS_HARDWARE_MODULE_ID, (hw_module_t const**)&mSensorModule);
	// open: 获得device
	sensors_open_1(&mSensorModule->common, &mSensorDevice);
		return module->methods->open(module, SENSORS_HARDWARE_POLL, (struct hw_device_t**)device);
	// get list
	ssize_t count = mSensorModule->get_sensors_list(mSensorModule, &list);
	// deactive
	for (size_t i=0 ; i<size_t(count) ; i++) {
        mSensorDevice->activate(
				reinterpret_cast<struct sensors_poll_device_t *>(mSensorDevice), 
          		list[i].handle, 0);
    }
```

poll：

　　通过调用sensor HAL的poll方法来获得数据

```c++
ssize_t SensorDevice::poll(sensors_event_t* buffer, size_t count) {
    do {
      	// 调用HAL的poll方法得到数据，放到buffer中
        c = mSensorDevice->poll(reinterpret_cast<struct sensors_poll_device_t *> (mSensorDevice), buffer, count);
    } while (c == -EINTR);
    return c;
}
```



## SensorService.cpp

>   native\services\sensorservice\SensorService.cpp

功能：

>   **作为SensorService，启动threadLoop，读取sensor数据并且分发给各个client**

分析：

启动SensorService后将会调用SensorService::onFirstRef

```c++
void SensorService::onFirstRef()
	SensorDevice& dev(SensorDevice::getInstance());	// 得到单例实例
	ssize_t count = dev.getSensorList(&list);	// 获取sensor list
	// 注册HAL所有配置的sensors
	for (ssize_t i=0 ; i<count ; i++)
    	registerSensor( new HardwareSensor(list[i]) );
```

threadLoop：

　　线程中一直获取以及分发sensor数据

```c++
bool SensorService::threadLoop()
  	SensorDevice& device(SensorDevice::getInstance());
	do {
      	// 调用SensorDevice的poll方法来获得数据
      	ssize_t count = device.poll(mSensorEventBuffer, numEventMax);
      	// 分发数据
      	for (size_t i=0 ; i < numConnections; ++i) {
            if (activeConnections[i] != 0)
                activeConnections[i]->sendEvents(mSensorEventBuffer, count, mSensorEventScratch, mMapFlushEventsToConnections);
        }
    } while (!Thread::exitPending());
```
