---
title: InputFlinger(6)_SocketPair
date: 2018-02-11 12:00:00
categories: 
- Android
- InputFlinger
---

# 功能
每个app都会调用addWindow
创建一组socketpair，fd[0]注册到InputDispatcher,fd[1]注册到app
so InputDispatcher send，则app就可以receive
<!-- more -->
# 源码分析
```java
addWindow
	// 创建socketpair
	InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
		return nativeOpenInputChannelPair(name);	// android_view_InputChannel_nativeOpenInputChannelPair
			// 创建socketpair
			InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
				socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets);
				outServerChannel = new InputChannel(serverChannelName, sockets[0]);
				outClientChannel = new InputChannel(clientChannelName, sockets[1]);
			// 返回数组channelPair[2]，内容包含socket fd[]
			jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);	// 创建类型为gInputChannelClassInfo长度为2的一维数组
			jobject serverChannelObj = android_view_InputChannel_createInputChannel(env, new NativeInputChannel(serverChannel));
            jobject clientChannelObj = android_view_InputChannel_createInputChannel(env, new NativeInputChannel(clientChannel));
			env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
			env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
			return channelPair;
					
	// fd[0] registerInputChannel
	win.setInputChannel(inputChannels[0]);
		mInputChannel = inputChannel;

	// fd[1] -> app
    inputChannels[1].transferTo(outInputChannel);
    	nativeTransferTo(outParameter);	// android_view_InputChannel_nativeTransferTo
    		
	// 每个app都对应一个connection，connection和socket的fd进行绑定
    mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);	// fd[0]
    	nativeRegisterInputChannel(mPtr, inputChannel, inputWindowHandle, false);
    		inputChannel = android_view_InputChannel_getInputChannel(env, inputChannelObj);
    		im->registerInputChannel(env, inputChannel, inputWindowHandle, monitor);
    			mInputManager->getDispatcher()->registerInputChannel(inputChannel, inputWindowHandle, monitor);	// InputDispatcher::registerInputChannel
    				connection = new Connection(inputChannel, inputWindowHandle, monitor);
    				fd = inputChannel->getFd();
    				mConnectionsByFd.add(fd, connection);	// 绑定connection和fd
    				mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
```
