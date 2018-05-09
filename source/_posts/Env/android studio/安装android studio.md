---
title: 安装android studio过程
date: 2018-05-09 17:30:53
categories:
- Env
- android studio
---

## 32位机器无法安装android studio 3.0
编译出现这个报错信息
```
Error:java.util.concurrent.ExecutionException: java.lang.RuntimeException: No server to serve request. Check logs for details.
Error:Execution failed for task ':app:mergeDebugResources'.
> Error: java.util.concurrent.ExecutionException: java.lang.RuntimeException: No server to serve request. Check logs for details.
```
<!--more-->
先百度找了一些方案，没什么用，最后翻墙找到了这篇[配置 Android Studio (Ubuntu)](https://blog.csdn.net/caspiansea/article/details/78898272)。

主要原因是：
	我的机器是32位机器，最新的android studio 3.0已经不再支持32位系统了，所以最新的是没法用了。


## 卸载
android studio 3.0是通过执行android-studio/bin/studio.sh的方式来进行安装的，可以通过删除~/.android和~/.AndroidStudio3.0来进行卸载。
