---
title: InputFlinger(4)_InputDispatcher
date: {{ date }}
categories: 
- Android
- InputFlinger
---

# 源码
> InputDispatcher.cpp (native\services\inputflinger)

# 功能
* mInboundQueue中取出事件，没有设置POLICY_FLAG_PASS_TO_USER则不用传递给app，drop
* SystemKey/GlobalKey预处理(interceptKeyBeforeDispatching HOME/SEARCH/MENU/GlobalKey...)，设置INTERCEPT_KEY_RESULT_SKIP
* UserKey则发送事件给对应app
<!-- more -->

# 源码分析
## InputDispatcherThread
InputDispatcherThread一直循环执行InputDispatcher::dispatchOnce
```c++
InputDispatcherThread::threadLoop()
	mDispatcher->dispatchOnce();
		// 1：mInboundQueue -> mCommandQueue
		dispatchOnceInnerLocked(&nextWakeupTime);
			
		// 2：run mCommandQueue中所有的command
		runCommandsLockedInterruptible();
			// 对mCommandQueue中的每一个都执行command
			do {
				CommandEntry* commandEntry = mCommandQueue.dequeueAtHead();

				Command command = commandEntry->command;
				(this->*command)(commandEntry);

				commandEntry->connection.clear();
				delete commandEntry;
			} while (! mCommandQueue.isEmpty());


		// wait wake
		mLooper->pollOnce(timeoutMillis);
```

## dispatchOnceInnerLocked
处理按键
* 没有设置POLICY_FLAG_PASS_TO_USER(不需要传递给app)
　例如PhoneWindowManager.interceptKeyBeforeQueueing已经处理过的事件
　设置dropReason = DROP_REASON_POLICY，丢弃该事件
* 设置了POLICY_FLAG_PASS_TO_USER(传递给app)
　　1：pokeUserActivityLocked(mPendingEvent);
　　2：等待执行doInterceptKeyBeforeDispatchingLockedInterruptible，并改变entry->interceptKeyResult
　　3：dispatchEventLocked
```c++
dispatchOnceInnerLocked
	if (! mPendingEvent) {
        // Inbound queue has at least one entry.
        mPendingEvent = mInboundQueue.dequeueAtHead();	// 从mInboundQueue中取出一个事件

        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER)
            pokeUserActivityLocked(mPendingEvent);
    }
    
    DropReason dropReason = DROP_REASON_NOT_DROPPED;	// 初始化
    if (!(mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER))
        dropReason = DROP_REASON_POLICY;	// PhoneWindowManager.interceptKeyBeforeQueueing已经处理过了
    
	case EventEntry::TYPE_KEY: {
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);	// typedEntry包含按键事件
        	// 放入mCommandQueue
        	if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN) {	// 没有处理过
				if (entry->policyFlags & POLICY_FLAG_PASS_TO_USER) {
				    CommandEntry* commandEntry = postCommandLocked(& InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible);
						CommandEntry* commandEntry = new CommandEntry(command);
						mCommandQueue.enqueueAtTail(commandEntry);
				    return false; // __FIXME__：注意这个return，false代表该事件并没有处理完，还得再来一次
				}
			} else if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_SKIP) {	// HOME/MENU/SEARCH...，interceptKeyBeforeDispatching处理过的
				if (*dropReason == DROP_REASON_NOT_DROPPED) {
				    *dropReason = DROP_REASON_POLICY;
				}
			}
			
			// interceptKeyBeforeQueueing/interceptKeyBeforeDispatching 已经处理过了(dropReason==DROP_REASON_POLICY)则直接return不再分发给app
			if (*dropReason != DROP_REASON_NOT_DROPPED) {
				setInjectionResultLocked(entry, *dropReason == DROP_REASON_POLICY
				        ? INPUT_EVENT_INJECTION_SUCCEEDED : INPUT_EVENT_INJECTION_FAILED);
				return true;
			}
			
			// 分发按键事件给APP
			int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime, entry, inputTargets, nextWakeupTime);	// 找到最前面的窗口
        	dispatchEventLocked(currentTime, entry, inputTargets);
				for (size_t i = 0; i < inputTargets.size(); i++) {
					const InputTarget& inputTarget = inputTargets.itemAt(i);

					ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
					sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
					prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
						enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
							// 放入outboundQueue中
							enqueueDispatchEntryLocked(connection, eventEntry, inputTarget, InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
								DispatchEntry* dispatchEntry = new DispatchEntry(eventEntry, // increments ref
										inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
										inputTarget->scaleFactor);
									connection->outboundQueue.enqueueAtTail(dispatchEntry);
							// 分发event事件
							startDispatchCycleLocked(currentTime, connection);
								while (connection->status == Connection::STATUS_NORMAL && !connection->outboundQueue.isEmpty()) {
									status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
											keyEntry->deviceId, keyEntry->source,
											dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
											keyEntry->keyCode, keyEntry->scanCode,
											keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
											keyEntry->eventTime);
										return mChannel->sendMessage(&msg);
											::send(mFd, msg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);	// socket send，app receive
									connection->outboundQueue.dequeue(dispatchEntry);
								}
				}
        break;
    }
    
    // 事件处理结束
    if (done)
        releasePendingEventLocked();
        	mPendingEvent = NULL;
```

## interceptKeyBeforeDispatching
回调PhoneWindowManager.interceptKeyBeforeDispatching
处理SystemKey/GlobalKey
该函数返回值基本都是-1/0
　　-1：HOME/MENU/SEARCH/GlobalKey...
　　0：Let the application handle the key.
```c++
InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible
	delay = mPolicy->interceptKeyBeforeDispatching(commandEntry->inputWindowHandle, &event, entry->policyFlags);	// NativeInputManager::interceptKeyBeforeDispatching
		jlong delayMillis = env->CallLongMethod(mServiceObj, gServiceClassInfo.interceptKeyBeforeDispatching, inputWindowHandleObj, keyEventObj, policyFlags);
			return mWindowManagerCallbacks.interceptKeyBeforeDispatching(focus, event, policyFlags);	// PhoneWindowManager.interceptKeyBeforeDispatching
				// AKEYCODE_XXX
				final int keyCode = event.getKeyCode();
					return mKeyCode;
				if (mGlobalKeyManager.handleGlobalKey(mContext, keyCode, event))
					return -1;	// INTERCEPT_KEY_RESULT_SKIP
				
	if (delay < 0) {	// -1
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_SKIP;
    } else if (!delay) {// 0
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_CONTINUE;
    } else {			// >0
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_TRY_AGAIN_LATER;
        entry->interceptKeyWakeupTime = now() + delay;
    }
```
