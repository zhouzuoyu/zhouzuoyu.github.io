---
title: AudioFlinge启动流程
date: 2019-12-09 13:46:00
categories: 
- Android
- audio
---

> frameworks\av\media\mediaserver\main_mediaserver.cpp

```c++
int main(int argc __unused, char** argv)
{
	AudioFlinger::instantiate();
}
```
<!--more-->
AudioFlinger继承BinderService
```c++
class AudioFlinger :
    public BinderService<AudioFlinger>,	// 继承BinderService
    public BnAudioFlinger
{
	static const char* getServiceName() ANDROID_API { return "media.audio_flinger"; }
}
```

因为继承BinderService，所以会导致BinderService::instantiate被调用
1. 添加media.audio_flinger服务
2. new AudioFlinger，AudioFlinger构造函数和onFirstRef被调用
```c++
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        return sm->addService(	// 添加服务
                String16(SERVICE::getServiceName()),	// AudioFlinger::getServiceName()
                new SERVICE(), allowIsolated);	// new AudioFlinger()
    }

    static void instantiate() { publish(); }
}
```
