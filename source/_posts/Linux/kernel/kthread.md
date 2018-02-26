---
title: kthread
date: 2018-02-26 21:41:13
categories:
- Linux
- kernel
---

# kthread_run
启动内核线程
由kthread_create()和wake_up_process()两部分组成，这样的好处是kthread_run()创建的线程可以直接运行。
<!--more-->
> create and wake a thread.

```c
#define kthread_run(threadfn, data, namefmt, ...)			   \
({									   \
	struct task_struct *__k						   \
		= kthread_create(threadfn, data, namefmt, ## __VA_ARGS__); \
	if (!IS_ERR(__k))						   \
		wake_up_process(__k);					   \
	__k;								   \
})
```

# kthread_stop
结束指定线程，__设置should_stop标志__
```c
int kthread_stop(struct task_struct *k);
```

# kthread_should_stop
返回should_stop标志，用于检查
```c
int kthread_should_stop(void);
```

# 例子
1. 创建
	```c
	data->auto_update = kthread_run(adt7470_update_thread, client,
						dev_name(data->hwmon_dev));
	```
2. threadfn
	```c
	static int adt7470_update_thread(void *p)
	{
		while (!kthread_should_stop()) {
			...
		}
	}
	```
3. stop
	```c
	kthread_stop(data->auto_update);
	```
