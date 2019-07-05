---
title: uevent方式上报耳机拔插事件
date: 2019-07-04 10:12:00
categories: 
- Android
- audio
---

> frameworks\base\services\core\java\com\android\server\WiredAccessoryManager.java

1. 创建uevent的监听list
check /sys/class/switch/h2w/state节点在不在
如果该节点都不在，则没必要监听h2w事件

```java
// 创建监测list
mObserver = new WiredAccessoryObserver();
	mUEventInfo = makeObservedUEventList();
		// 添加uevent检测list
		// /sys/class/switch/xxx/state	(h2w/usb_audio/hdmi_audio/hdmi)
		List<UEventInfo> retVal = new ArrayList<UEventInfo>();
		// Monitor h2w
		if (!mUseDevInputEventForAudioJack) {
			uei = new UEventInfo(NAME_H2W, BIT_HEADSET, BIT_HEADSET_NO_MIC, BIT_LINEOUT);	// "h2w", mState1Bits=BIT_HEADSET, mState2Bits=BIT_HEADSET_NO_MIC, mStateNbits=BIT_LINEOUT
			if (uei.checkSwitchExists()) {
				File f = new File(getSwitchStatePath());
					return String.format(Locale.US, "/sys/class/switch/%s/state", mDevName);	// /sys/class/switch/h2w/state
				return f.exists();

				retVal.add(uei);	// 添加到list中
			}
		}
```
<!--more-->
2. startObserving
创建线程，监听uevent
有指定的uevent事件则回调onUEvent

```java
/*
 * 开始监测
 * 1: 监听过滤uevent
 * 2: 有指定事件则回调onUEvent
 */
private void onSystemReady() {
	mObserver.init();
		for (int i = 0; i < mUEventInfo.size(); ++i) {
			UEventInfo uei = mUEventInfo.get(i);
			startObserving("DEVPATH="+uei.getDevPath());	// match字段，例如 DEVPATH=/devices/virtual/switch/h2w，监听带上报带此类信息的uevent
		}
}
```

> frameworks\base\core\java\android\os\UEventObserver.java

```java
public abstract class UEventObserver {
	// 启动线程，监听过滤uevent事件，然后sendEvent
	public final void startObserving(String match) {
		// 创建并启动thread，监听所有的uevent事件
		final UEventThread t = getThread();
			sThread = new UEventThread();
			sThread.start();	// 调用UEventThread.run
				public void run() {
					nativeSetup();	// socket(PF_NETLINK, SOCK_DGRAM, NETLINK_KOBJECT_UEVENT);

					while (true) {
						String message = nativeWaitForNextEvent();	// poll & recv & match
						if (message != null) {
							sendEvent(message); // UEventThread.sendEvent
						}
					}
				}
		// 设置指定事件，nativeWaitForNextEvent中会进行过滤
		t.addObserver(match, this);
			mKeysAndObservers.add(match);
			mKeysAndObservers.add(observer);
			nativeAddMatch(match);
	}

	private static final class UEventThread extends Thread {
		// 回调WiredAccessoryObserver.onUEvent
		private void sendEvent(String message) {
			// 再次确认message有没有match字段
			synchronized (mKeysAndObservers) {
				final int N = mKeysAndObservers.size();
				for (int i = 0; i < N; i += 2) {
					final String key = (String)mKeysAndObservers.get(i);
					if (message.contains(key)) {
						final UEventObserver observer =
								(UEventObserver)mKeysAndObservers.get(i + 1);
						mTempObserversToSignal.add(observer);
					}
				}
			}

			if (!mTempObserversToSignal.isEmpty()) {
				final UEvent event = new UEvent(message);	// 根据message重新构建UEvent
				final int N = mTempObserversToSignal.size();
				for (int i = 0; i < N; i++) {
					final UEventObserver observer = mTempObserversToSignal.get(i);
					observer.onUEvent(event);	// 回调WiredAccessoryObserver.onUEvent
				}
			}
		}
	}
}
```
3. 调用updateLocked，uevent和input方式都会调用这个函数

```java
// 如果获取到相应事件则交给此函数处理
class WiredAccessoryObserver extends UEventObserver {
	public void onUEvent(UEventObserver.UEvent event) {
		String devPath = event.get("DEVPATH");
		String name = event.get("SWITCH_NAME");
		int state = Integer.parseInt(event.get("SWITCH_STATE"));
		synchronized (mLock) {
			updateStateLocked(devPath, name, state);
				for (int i = 0; i < mUEventInfo.size(); ++i) {
					UEventInfo uei = mUEventInfo.get(i);
					if (devPath.equals(uei.getDevPath())) {
						updateLocked(name, uei.computeNewHeadsetState(mHeadsetState, state));	// 也是调用updateLocked，后续流程和input方式上报耳机拔插事件一致
						return;
					}
				}
		}
	}
}
```

4. updateLocked 
主要是将事件给AudioService广播出去

```java
final class WiredAccessoryManager implements WiredAccessoryCallbacks {

	public WiredAccessoryManager(Context context, InputManagerService inputManager) {
		mAudioManager = (AudioManager)context.getSystemService(Context.AUDIO_SERVICE);	// WiredAccessoryManager和AUDIO_SERVICE建立关系，从而状态可以传递给audio
		mUseDevInputEventForAudioJack =
				context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);
	}

	// uevent和input方式都会调用此函数，主要是将事件给AudioService广播出去
	private void updateLocked(String newName, int newState) {
		int headsetState = newState & SUPPORTED_HEADSETS;
		Message msg = mHandler.obtainMessage(MSG_NEW_DEVICE_STATE, headsetState,
				mHeadsetState, newName);
		mHandler.sendMessage(msg);	// handleMessage
			setDeviceStateLocked(curHeadset, headsetState, prevHeadsetState, headsetName);
				if (headset == BIT_HEADSET) {
					outDevice = AudioManager.DEVICE_OUT_WIRED_HEADSET;
					inDevice = AudioManager.DEVICE_IN_WIRED_HEADSET;
				} else if (headset == BIT_HEADSET_NO_MIC){
					outDevice = AudioManager.DEVICE_OUT_WIRED_HEADPHONE;
				} else if (headset == BIT_LINEOUT){
					outDevice = AudioManager.DEVICE_OUT_LINE;
				}
				if (outDevice != 0) {
					mAudioManager.setWiredDeviceConnectionState(outDevice, state, headsetName); // AudioManager
						service.setWiredDeviceConnectionState(device, state, name);				// AudioService
							queueMsgUnderWakeLock(mAudioHandler,
									MSG_SET_WIRED_DEVICE_CONNECTION_STATE,
									device,
									state,
									name,
									delay);
								sendMsg(handler, msg, SENDMSG_QUEUE, arg1, arg2, obj, delay);	// mAudioHandler.handleMessage
									case MSG_SET_WIRED_DEVICE_CONNECTION_STATE:
										onSetWiredDeviceConnectionState(msg.arg1, msg.arg2, (String)msg.obj);
											handleDeviceConnection((state == 1), device, (isUsb ? name : ""));	// FIXME
											sendDeviceConnectionIntent(device, state, name);	// 发送广播
												if (device == AudioSystem.DEVICE_OUT_WIRED_HEADSET) {
													connType = AudioRoutesInfo.MAIN_HEADSET;
													intent.setAction(Intent.ACTION_HEADSET_PLUG);
													intent.putExtra("microphone", 1);
												} else if (device == AudioSystem.DEVICE_OUT_WIRED_HEADPHONE ||
														   device == AudioSystem.DEVICE_OUT_LINE) {
													/*do apps care about line-out vs headphones?*/
													connType = AudioRoutesInfo.MAIN_HEADPHONES;
													intent.setAction(Intent.ACTION_HEADSET_PLUG);
													intent.putExtra("microphone", 0);
												}
												ActivityManagerNative.broadcastStickyIntent(intent, null, UserHandle.USER_ALL);
										break;
				}
				if (inDevice != 0) {
					mAudioManager.setWiredDeviceConnectionState(inDevice, state, headsetName);
				}

		mHeadsetState = headsetState;	// 最后更新
	}
}
```
