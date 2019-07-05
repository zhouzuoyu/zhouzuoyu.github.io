---
title: input方式上报耳机拔插事件
date: 2019-07-04 10:12:00
categories: 
- Android
- audio
---

**InputReader.cpp->InputDispatcher.cpp->com_android_server_input_InputManagerService.cpp->InputManagerService.java->WiredAccessoryManager.java**

# 配置文件
> frameworks\base\core\res\res\values\config.xml

选择使用uevent还是input方式
```xml
	<!-- When true use the linux /dev/input/event subsystem to detect the switch changes
		 on the headphone/microphone jack. When false use the older uevent framework. -->
	<bool name="config_useDevInputEventForAudioJack">true</bool>
```
<!--more-->
# InputReader.cpp
> frameworks\native\services\inputflinger\InputReader.cpp

对switch事件做简单的处理，然后给InputDispatcher
```c++
void SwitchInputMapper::process(const RawEvent* rawEvent) {
	switch (rawEvent->type) {
	case EV_SW:
		processSwitch(rawEvent->code, rawEvent->value);
			// 解析code和value
			if (switchCode >= 0 && switchCode < 32) {
				if (switchValue) {
					mSwitchValues |= 1 << switchCode;
				} else {
					mSwitchValues &= ~(1 << switchCode);
				}
				mUpdatedSwitchMask |= 1 << switchCode;
			}
		break;

	case EV_SYN:
		if (rawEvent->code == SYN_REPORT) {
			sync(rawEvent->when);
				// 根据code/value构建args，然后给InputDispatcher
				uint32_t updatedSwitchValues = mSwitchValues & mUpdatedSwitchMask;
				NotifySwitchArgs args(when, 0, updatedSwitchValues, mUpdatedSwitchMask);
				getListener()->notifySwitch(&args);	// getListener为InputDispatcher，即调用InputDispatcher::notifySwitch
		}
	}
}
```

# InputDispatcher.cpp
> frameworks\native\services\inputflinger\InputDispatcher.cpp

将调用NativeInputManager::notifySwitch
```c++
// mPolicy就是NativeInputManager
NativeInputManager::NativeInputManager(jobject contextObj,
		jobject serviceObj, const sp<Looper>& looper) :
		mLooper(looper), mInteractive(true) {
	mInputManager = new InputManager(eventHub, this, this);
		mDispatcher = new InputDispatcher(dispatcherPolicy);	// dispatcherPolicy==this，即dispatcherPolicy就是NativeInputManager
			InputDispatcher::InputDispatcher(const sp<InputDispatcherPolicyInterface>& policy) :
				mPolicy(policy),	// mPolicy就是NativeInputManager
		mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
}
```

```c++
void InputDispatcher::notifySwitch(const NotifySwitchArgs* args) {
	mPolicy->notifySwitch(args->eventTime,
			args->switchValues, args->switchMask, policyFlags);	// mPolicy==NativeInputManager，所以就是调用NativeInputManager::notifySwitch
}
```

# com_android_server_input_InputManagerService.cpp
> frameworks\base\services\core\jni\com_android_server_input_InputManagerService.cpp

JNI，调用InputManagerService.notifySwitch
```c++
void NativeInputManager::notifySwitch(nsecs_t when,
		uint32_t switchValues, uint32_t switchMask, uint32_t policyFlags) {
	JNIEnv* env = jniEnv();

	env->CallVoidMethod(mServiceObj, gServiceClassInfo.notifySwitch,
			when, switchValues, switchMask);	// 调用InputManagerService.notifySwitch
	checkAndClearExceptionFromCallback(env, "notifySwitch");
}
```

# InputManagerService.java
将调用WiredAccessoryManager.notifyWiredAccessoryChanged，最终调用updateLocked
> frameworks\base\services\core\java\com\android\server\input\InputManagerService.java

```java
	private void startOtherServices() {
		inputManager = new InputManagerService(context);
			mUseDevInputEventForAudioJack =
					context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);	// frameworks\base\core\res\res\values\config.xml
			mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());	// 调用com_android_server_input_InputManagerService.cpp中的native函数

		inputManager.setWiredAccessoryCallbacks(
				new WiredAccessoryManager(context, inputManager));	// 建立WiredAccessoryManager和InputManagerService的联系，这样InputManagerService就能对应到WiredAccessoryManager
	}
```

```java
public class InputManagerService extends IInputManager.Stub
		implements Watchdog.Monitor {

	// 将调用WiredAccessoryManager.notifyWiredAccessoryChanged
	private void notifySwitch(long whenNanos, int switchValues, int switchMask) {
		if (mUseDevInputEventForAudioJack && (switchMask & SW_JACK_BITS) != 0) {
			mWiredAccessoryCallbacks.notifyWiredAccessoryChanged(whenNanos, switchValues,
					switchMask);	// WiredAccessoryManager.notifyWiredAccessoryChanged
				mSwitchValues = (mSwitchValues & ~switchMask) | switchValues;
				// 有SW_MICROPHONE_INSERT_BIT就是headset
				switch (mSwitchValues &
					(SW_HEADPHONE_INSERT_BIT | SW_MICROPHONE_INSERT_BIT | SW_LINEOUT_INSERT_BIT)) {
					case 0:
						headset = 0;
						break;

					case SW_HEADPHONE_INSERT_BIT:
						headset = BIT_HEADSET_NO_MIC;
						break;

					case SW_LINEOUT_INSERT_BIT:
						headset = BIT_LINEOUT;
						break;

					case SW_HEADPHONE_INSERT_BIT | SW_MICROPHONE_INSERT_BIT:
						headset = BIT_HEADSET;
						break;

					case SW_MICROPHONE_INSERT_BIT:
						headset = BIT_HEADSET;
						break;

					default:
						headset = 0;
						break;
				}
				updateLocked(NAME_H2W,
					(mHeadsetState & ~(BIT_HEADSET | BIT_HEADSET_NO_MIC | BIT_LINEOUT)) | headset);
		}
	}
}
```
