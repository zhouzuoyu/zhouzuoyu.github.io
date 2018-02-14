---
title: Android Sensors(3)_SensorService启动
date: 2018-01-29 17:00:15
categories: 
- Android
- Sensors
---


SystemServer进程启动后，会加载libandroid_servers.so，然后启动SensorService

```java
public final class SystemServer {
  	public static void main(String[] args) {
        new SystemServer().run();
    }
	private void run() {
		System.loadLibrary("android_servers");	// load JNI
        nativeInit();	// 调用JNI
    }
}
```
<!-- more -->
## libandroid_servers.so出处

frameworks/base/services/Android.mk

​	搜集frameworks/base/services/*/jni/下所有的Android.mk，最终编译生成libandroid_servers.so

```makefile
# include all the jni subdirs to collect their sources
include $(wildcard $(LOCAL_PATH)/*/jni/Android.mk)
LOCAL_MODULE:= libandroid_servers
```

所以必然包含：

​	base\services\core\jni\com_android_server_SystemServer.cpp

## nativeInit

启动SensorService

```c++
static void android_server_SystemServer_nativeInit(JNIEnv* env, jobject clazz) {
    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the sensor service
        SensorService::instantiate();
    }
}
static JNINativeMethod gMethods[] = {
    { "nativeInit", "()V", (void*) android_server_SystemServer_nativeInit },
};
int register_android_server_SystemServer(JNIEnv* env)
  	// 和SystemServer.java建立连接
	return jniRegisterNativeMethods(env, "com/android/server/SystemServer",
            gMethods, NELEM(gMethods));
```
