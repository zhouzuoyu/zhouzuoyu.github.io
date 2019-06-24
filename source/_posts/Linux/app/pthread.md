---
title: pthread_cond_timedwait
date: 2019-06-24 10:16:00
categories:
- Linux
- app
---

# Caution
以下代码设置超时时间使用了绝对时间，如果中途修改系统时间，会出现问题不满足预期。
1. date -s "2000/01/1 00:25:00"
2. ./pthread &	// 运行以下程序，预期50s后(绝对时间)超时，即2000/01/1 00:25:50超时
3. date -s "2000/01/1 00:25:45"	// 修改系统时间(一下加速了45s)，发现5秒后超时

<!--more-->
# 示例代码
**创建线程，pthread_cond_timedwait设置等待时间50秒，但是100s才发送信号，所以必然超时**
```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>
#include <semaphore.h>
#include <sys/time.h>
#include <errno.h>

#define SENDSIGTIME 100	// 100s后发送信号

pthread_cond_t g_cond;
pthread_mutex_t g_mutex;

void thread1(void *arg)
{
    int inArg = (int)arg;
    int ret = 0;
    struct timeval tv;
    struct timespec abstime;

    pthread_mutex_lock(&g_mutex);

    gettimeofday(&tv, NULL);
    abstime.tv_sec = tv.tv_sec + 50;	// 等待超时时间50s
	abstime.tv_nsec = tv.tv_usec * 1000;

    ret = pthread_cond_timedwait(&g_cond, &g_mutex, &abstime);
	if (ETIMEDOUT == ret)
		printf("Time out happened.\n");

	pthread_mutex_unlock(&g_mutex);
}

int main(void)
{
    pthread_t id1;
    int ret;

	// create thread
    pthread_cond_init(&g_cond, NULL);
    pthread_mutex_init(&g_mutex, NULL);
    ret = pthread_create(&id1, NULL, (void *)thread1, (void *)1);
    if (0 != ret)    {
		printf("thread 1 create failed!\n");
		return 1;
    }

    printf("等待%ds发送信号!\n", SENDSIGTIME);
    sleep(SENDSIGTIME);				// 等待100s
    printf("正在发送信号....\n");

    pthread_mutex_lock(&g_mutex);
    pthread_cond_signal(&g_cond);	// 发送信号
    pthread_mutex_unlock(&g_mutex);

    pthread_join(id1, NULL);
    pthread_cond_destroy(&g_cond);
    pthread_mutex_destroy(&g_mutex);

    return 0;
}
```
