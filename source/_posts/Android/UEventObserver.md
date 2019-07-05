---
title: UEventObserver
date: 2019-06-12 10:12:00
categories: 
- Android
---

# HAL
>  hardware\libhardware_legacy\uevent\uevent.c

提供netlink相关接口，给JNI使用
```c
// bind NETLINK_KOBJECT_UEVENT
int uevent_init()
{
	memset(&addr, 0, sizeof(addr));
	addr.nl_family = AF_NETLINK;
	addr.nl_pid = getpid();
	addr.nl_groups = 0xffffffff;

	s = socket(PF_NETLINK, SOCK_DGRAM, NETLINK_KOBJECT_UEVENT);

	setsockopt(s, SOL_SOCKET, SO_RCVBUFFORCE, &sz, sizeof(sz));

	if(bind(s, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
		close(s);
		return 0;
	}

	fd = s;
	return (fd > 0);
}
```
<!--more-->
```c
// 监听NETLINK_KOBJECT_UEVENT
int uevent_next_event(char* buffer, int buffer_length)
{
	while (1) {
		fds.fd = fd;
		fds.events = POLLIN;
		fds.revents = 0;
		nr = poll(&fds, 1, -1);
	 
		if(nr > 0 && (fds.revents & POLLIN)) {
			int count = recv(fd, buffer, buffer_length, 0);
			if (count > 0) {
				struct uevent_handler *h;
				pthread_mutex_lock(&uevent_handler_list_lock);
				LIST_FOREACH(h, &uevent_handler_list, list)
					h->handler(h->handler_data, buffer, buffer_length);
				pthread_mutex_unlock(&uevent_handler_list_lock);

				return count;
			} 
		}
	}
}
```

# JNI
>  frameworks\base\core\jni\android_os_UEventObserver.cpp

// 注册方法
```cpp
static JNINativeMethod gMethods[] = {
	{ "nativeSetup", "()V",
			(void *)nativeSetup },
	{ "nativeWaitForNextEvent", "()Ljava/lang/String;",
			(void *)nativeWaitForNextEvent },
	{ "nativeAddMatch", "(Ljava/lang/String;)V",
			(void *)nativeAddMatch },
	{ "nativeRemoveMatch", "(Ljava/lang/String;)V",
			(void *)nativeRemoveMatch },
};
```

```cpp
static void nativeSetup(JNIEnv *env, jclass clazz) {
	if (!uevent_init()) {
		jniThrowException(env, "java/lang/RuntimeException",
				"Unable to open socket for UEventObserver");
	}
}

static jstring nativeWaitForNextEvent(JNIEnv *env, jclass clazz) {
	char buffer[1024];

	/*
	 * 1: 监听所有的uevent事件
	 * 2: 过滤指定事件(例如信息中带DEVPATH=/devices/virtual/switch/h2w)然后return该msg
	 */
	for (;;) {
		int length = uevent_next_event(buffer, sizeof(buffer) - 1);
		buffer[length] = '\0';

		if (isMatch(buffer, length)) {
			// match过程
			for (size_t i = 0; i < gMatches.size(); i++) {
				const String8& match = gMatches.itemAt(i);	// 取出match key

				const char* field = buffer;
				const char* end = buffer + length + 1;
				do {
					if (strstr(field, match.string())) {
						ALOGV("Matched uevent message with pattern: %s", match.string());
						return true;
					}
					field += strlen(field) + 1;
				} while (field != end);
			}

			// Assume the message is ASCII.
			jchar message[length];
			for (int i = 0; i < length; i++) {
				message[i] = buffer[i];
			}
			return env->NewString(message, length);
		}
	}
}

// 将match字段放入到gMatches中，例如 DEVPATH=/devices/virtual/switch/h2w
static void nativeAddMatch(JNIEnv* env, jclass clazz, jstring matchStr) {
	ScopedUtfChars match(env, matchStr);

	AutoMutex _l(gMatchesMutex);
	gMatches.add(String8(match.c_str()));
}
```

# Framework
> frameworks\base\core\java\android\os\uEventObserver.java

```java
public abstract class UEventObserver {
	// native方法
	private static native String nativeWaitForNextEvent();
	private static native void nativeSetup();

	// 创建UEventThread，开始监测
	public final void startObserving(String match) {
		final UEventThread t = getThread();
			// 第一次将new UEventThread，之后则直接返回sThread不需要重复new
			if (sThread == null) {
				sThread = new UEventThread();
				sThread.start();
					// UEventThread开始run
					public void run() {
						nativeSetup();	// userspace注册netlink

						// 等待事件，有事件则回调各自的onUEvent
						while (true) {
							String message = nativeWaitForNextEvent();
							if (message != null) {
								sendEvent(message);
									// 
									if (!mTempObserversToSignal.isEmpty()) {
										final UEvent event = new UEvent(message);
										final int N = mTempObserversToSignal.size();
										for (int i = 0; i < N; i++) {
											final UEventObserver observer = mTempObserversToSignal.get(i);
											observer.onUEvent(event);	// FIXME
										}
									}
							}
						}
					}
			}
			return sThread;

		t.addObserver(match, this);
	}
}
```
