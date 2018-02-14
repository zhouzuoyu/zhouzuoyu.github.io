---
title: InputFlinger(7)_InputDispatcher和PhoneWindowManager的联系
date: {{ date }}
categories: 
- Android
- InputFlinger
---

# 前言
该文档Trace了Dispatcher线程的mPolicy为什么是PhoneWindowManager

NativeInputManager::interceptKeyBeforeQueueing -> 
InputManagerService.interceptKeyBeforeQueueing -> 
InputMonitor.interceptKeyBeforeQueueing -> 
PhoneWindowManager.interceptKeyBeforeQueueing
<!-- more -->
# 分析
```c++
void InputDispatcher::notifyKey(const NotifyKeyArgs* args)
	policyFlags |= POLICY_FLAG_TRUSTED;
	// InputDispatcher通过JNI最终调用到PhoneWindowManager
	// 1:
	mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);
		if ((policyFlags & POLICY_FLAG_TRUSTED)) {
		    jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
		    if (keyEventObj) {
		    	// 2:
		        wmActions = env->CallIntMethod(mServiceObj, gServiceClassInfo.interceptKeyBeforeQueueing, keyEventObj, policyFlags);	// 调用的地方
		        	// 3:
		        	return mWindowManagerCallbacks.interceptKeyBeforeQueueing(event, policyFlags);	// mWindowManagerCallbacks指向mInputMonitor，所以调用InputMonitor::interceptKeyBeforeQueueing
		        		// 4:
		        		return mService.mPolicy.interceptKeyBeforeQueueing(event, policyFlags);		// WindowManagerService.PhoneWindowManager.interceptKeyBeforeQueueing
		        android_view_KeyEvent_recycle(env, keyEventObj);
		        env->DeleteLocalRef(keyEventObj);
		    }
			handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
    	}
```
__以下就对上述code中的1/2/3/4点进行阐述__

## InputDispatcher.mPolicy指向NativeInputManager
> base\services\core\jni\com_android_server_input_InputManagerService.cpp

```c++
NativeInputManager::NativeInputManager(jobject contextObj, jobject serviceObj, const sp<Looper>& looper)
    mInputManager = new InputManager(eventHub, this, this);	// this指向NativeInputManager对象
    	mDispatcher = new InputDispatcher(dispatcherPolicy);	// dispatcherPolicy和readerPolicy都指向NativeInputManager对象
    		InputDispatcher::InputDispatcher(const sp<InputDispatcherPolicyInterface>& policy) :
			    mPolicy(policy),	// InputDispatcher.mPolicy指向NativeInputManager对象
```
<!-- more -->
## JNI
```c++
register_android_server_InputManager(JNIEnv* env)
	// 1: java调用c++
	jniRegisterNativeMethods(env, "com/android/server/input/InputManagerService", gInputManagerMethods, NELEM(gInputManagerMethods));
	// 2: c++调用java
	FIND_CLASS(clazz, "com/android/server/input/InputManagerService");
	GET_METHOD_ID(gServiceClassInfo.interceptKeyBeforeQueueing, clazz, "interceptKeyBeforeQueueing", "(Landroid/view/KeyEvent;I)I");	// 设置的地方
```

## mWindowManagerCallbacks就是mInputMonitor
```java
private void startOtherServices()
	inputManager.setWindowManagerCallbacks(wm.getInputMonitor());	// return mInputMonitor;
		mWindowManagerCallbacks = callbacks;	// mWindowManagerCallbacks == mInputMonitor
```

## mService和mPolicy的出处
> base\services\core\java\com\android\server\wm\WindowManagerService.java

```java
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs
    /*
     * Part 1: mService指向WindowManagerService
     * 	  new InputMonitor将this指针传入，so mService指向了WindowManagerService实例本身
     */
	final InputMonitor mInputMonitor = new InputMonitor(this);
		public InputMonitor(WindowManagerService service)
		    mService = service;

	final WindowManagerPolicy mPolicy = PolicyManager.makeNewWindowManager();	// PhoneWindowManager
```

```java
/*
 * Part 2: mPolicy指向PhoneWindowManager
 */
// base\core\java\com\android\internal\policy\IPolicy.java: 申明接口
public class Policy implements IPolicy {
	public WindowManagerPolicy makeNewWindowManager()
}
// base\policy\src\com\android\internal\policy\impl\Policy.java: 实现接口
public class Policy implements IPolicy {
	public WindowManagerPolicy makeNewWindowManager() {
        return new PhoneWindowManager();
    }
}
// base\core\java\com\android\internal\policy\PolicyManager.java：调用
public final class PolicyManager {
	// 静态代码块中创建了class Policy的一个实例
	static {
		// 寻找com.android.internal.policy.impl.Policy类，即base\policy\src\com\android\internal\policy\impl\Policy.java
		Class policyClass = Class.forName("com.android.internal.policy.impl.Policy");
		sPolicy = (IPolicy)policyClass.newInstance();
	}
	
	public static WindowManagerPolicy makeNewWindowManager() {
        return sPolicy.makeNewWindowManager();
        	return new PhoneWindowManager();
    }
}
```
