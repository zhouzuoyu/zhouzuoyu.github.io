---
title: InputFlinger(3)_InputReader
date: {{ date }}
categories: 
- Android
- InputFlinger
---

# 源码
InputReader.cpp (native\services\inputflinger)

# 功能
以按键类事件阐述
1：读取
　　eventhub读取事件(mEventHub->getEvents)
2：处理
　　scancode->AKEYCODE_XXX(kl)
　　systemKey预处理(PhoneWindowManager.interceptKeyBeforeQueueing 音量键/电源键/耳机键...)，是否设置POLICY_FLAG_PASS_TO_USER
　　放入到mInboundQueue

<!-- more -->
# 源码分析
## InputReaderThread
```c++
InputReaderThread::threadLoop()
	mReader->loopOnce();
```

1: 调用eventhub接口读取/dev/input/事件
2: 交于不同的process分别处理
```c++
InputReader::loopOnce()
	// get events
	size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);	// mEventBuffer存放kernel上报的event数据
	// process
	processEventsLocked(mEventBuffer, count);
		for (const RawEvent* rawEvent = rawEvents; count;) {
            int32_t deviceId = rawEvent->deviceId;
            // 处理event数据
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
            	// 不同的device使用不同的process
            	ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
            	InputDevice* device = mDevices.valueAt(deviceIndex);
            	device->process(rawEvents, count);
        }
```

## process
按键类事件的process方法
1：scancode->AKEYCODE_XXX(kl文件)
2: 构建成NotifyKeyArgs传递给InputDispatcher
```c++
KeyboardInputMapper::process(const RawEvent* rawEvent)
	switch (rawEvent->type) {
	case EV_KEY: {
		int32_t scanCode = rawEvent->code;	// linux report key
		// 1：scancode -> AKEYCODE_XXX
		getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, &keyCode, &flags);	// key 158   BACK: 则scancode对应158，keyCode是AKEYCODE_BACK，flag为0
			Device* device = getDeviceLocked(deviceId);
		    // 一般都是从keyLayoutMap中获取对应关系
		    if (device->keyMap.haveKeyLayout()) {
		        if (!device->keyMap.keyLayoutMap->mapKey(scanCode, usageCode, outKeycode, outFlags)) {
		        	const Key* key = getKey(scanCode, usageCode);	// 使用scancode得到AKEYCODE_XXX
		        	*outKeyCode = key->keyCode;
    				*outFlags = key->flags;
		            return NO_ERROR;
		        }
		    }
		    
		// 2：构建NotifyKeyArgs给input_dispatch
		processKey(rawEvent->when, rawEvent->value != 0, keyCode, scanCode, flags);
			NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags, down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP, AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);	// policyFlags：0
			getListener()->notifyKey(&args);	// InputDispatcher::notifyKey
		break;
	}
}
```

### getListener
InputReader的Listener是InputDispatcher
> base\services\core\jni\com_android_server_input_InputManagerService.cpp

```c++
NativeInputManager::NativeInputManager
	sp<EventHub> eventHub = new EventHub();	// new
    mInputManager = new InputManager(eventHub, this, this);
		mDispatcher = new InputDispatcher(dispatcherPolicy);
	    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);	// mDispatcher为mReader的listener
```

### notifyKey
1: 调用PhoneWindowManager.interceptKeyBeforeQueueing进行处理(电话静音/截屏...)，不能处理则设置POLICY_FLAG_PASS_TO_USER
2: 把按键事件放入到mInboundQueue中
```c++
InputDispatcher::notifyKey(const NotifyKeyArgs* args)
	uint32_t policyFlags = args->policyFlags;
	policyFlags |= POLICY_FLAG_TRUSTED; // policyFlags: POLICY_FLAG_TRUSTED
    /*
     * NativeInputManager::interceptKeyBeforeQueueing
     * callback PhoneWindowManager.interceptKeyBeforeQueueing对特殊按键做预先处理，不是该种类则设置为POLICY_FLAG_PASS_TO_USER
     */
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);	// policyFlags是一个引用，所以可以改变
        jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);	// keyEvent转换下
        if (keyEventObj) {
        	/*
        	 * PhoneWindowManager.interceptKeyBeforeQueueing
        	 * 该函数可以处理的事件则处理，不能处理的则设置POLICY_FLAG_PASS_TO_USER
        	 */
            wmActions = env->CallIntMethod(mServiceObj,
                    gServiceClassInfo.interceptKeyBeforeQueueing,
                    keyEventObj, policyFlags);	

            android_view_KeyEvent_recycle(env, keyEventObj);
            env->DeleteLocalRef(keyEventObj);
        }

		// 如果该事件在PhoneWindowManager.interceptKeyBeforeQueueing中不能处理，则设置policyFlags为POLICY_FLAG_PASS_TO_USER
        handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
        	if (wmActions & WM_ACTION_PASS_TO_USER)
		        policyFlags |= POLICY_FLAG_PASS_TO_USER;
    
    // 放入到mInboundQueue中
    KeyEntry* newEntry = new KeyEntry(args->eventTime,
            args->deviceId, args->source, policyFlags,
            args->action, flags, keyCode, args->scanCode,
            metaState, repeatCount, args->downTime);	// policyFlags: 0
    needWake = enqueueInboundEventLocked(newEntry);
    	mInboundQueue.enqueueAtTail(entry);	// 将KeyEntry放到mInboundQueue
```

### interceptKeyBeforeQueueing
处理VOLUME_XXX/POWER/耳机键...
截屏/电话静音/电话挂断
```java
PhoneWindowManager.interceptKeyBeforeQueueing
	// check该键值是否是GlobalKey，如果是GlobalKey则直接return
	if (mGlobalKeyManager.shouldHandleGlobalKey(keyCode, event)) {
        if (isWakeKey) {
            mPowerManager.wakeUp(event.getEventTime());
        }
        return result;
    }
    // Handle special keys.
    switch (keyCode) {
   		case KeyEvent.KEYCODE_VOLUME_DOWN:
        case KeyEvent.KEYCODE_VOLUME_UP:
        case KeyEvent.KEYCODE_VOLUME_MUTE: {
        	// 按下音量下键，是触发截屏流程的必要条件
			if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
		        if (down) {
		            if (interactive && !mVolumeDownKeyTriggered
		                    && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
		                mVolumeDownKeyTriggered = true;
		                mVolumeDownKeyTime = event.getDownTime();
		                mVolumeDownKeyConsumedByScreenshotChord = false;
		                cancelPendingPowerKeyAction();
		                interceptScreenshotChord();
		                	mHandler.postDelayed(mScreenshotRunnable, getScreenshotChordLongPressDelay());
		                		takeScreenshot();	// 截屏
		            }
				}
		    }
		    // 有电话进来时按下VOLUME_DOWN/VOLUME_UP/VOLUME_MUTE则静音
		    if (down) {
		    	if (telecomManager.isRinging()) {
                    telecomManager.silenceRinger();

                    result &= ~ACTION_PASS_TO_USER;	// 不传递给APP
                    break;
                }
		    }
		}
		case KeyEvent.KEYCODE_POWER: {
            result &= ~ACTION_PASS_TO_USER;	// power键不会传递给APP
            if (down) {
            	// 按下电源键，是触发截屏流程的必要条件
                if (interactive && !mPowerKeyTriggered
                        && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                    mPowerKeyTriggered = true;
                    mPowerKeyTime = event.getDownTime();
                    interceptScreenshotChord();
                }

				// 静音/挂断电话
                TelecomManager telecomManager = getTelecommService();
                boolean hungUp = false;
                if (telecomManager != null) {
                    if (telecomManager.isRinging()) {
                        telecomManager.silenceRinger();
                    } else if ((mIncallPowerBehavior
                            & Settings.Secure.INCALL_POWER_BUTTON_BEHAVIOR_HANGUP) != 0
                            && telecomManager.isInCall() && interactive) {
                        hungUp = telecomManager.endCall();
                    }
                }
            }
            break;
        }
    }
```
