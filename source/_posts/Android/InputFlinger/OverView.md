---
title: InputFlinger(1)_Overview
date: 2018-02-11 12:00:00
categories: 
- Android
- InputFlinger
---

以按键类事件阐述InputFlinger框架流程:

> {% post_link Android/InputFlinger/EventHub EventHub %}

提供读取/dev/input/eventx的接口函数

> {% post_link Android/InputFlinger/InputReader InputReader %}

<!-- more -->
启动InputReaderThread，getEvents，处理SystemKey，放入mInboundQueue中

> {% post_link Android/InputFlinger/InputDispatcher InputDispatcher %}

启动InputDispatcherThread，从mInboundQueue中取出事件，
处理SystemKey/GlobalKey，如果是UserKey则通过socketpair传递给app

interceptKeyBeforeQueueing/interceptKeyBeforeDispatching能够处理的，最终都会丢弃(DROP_REASON_POLICY)，不会传递给APP

综上，按键类事件数据流：
> NotifyKeyArgs -> interceptKeyBeforeQueueing -> mInboundQueue -> interceptKeyBeforeDispatching -> outboundQueue
