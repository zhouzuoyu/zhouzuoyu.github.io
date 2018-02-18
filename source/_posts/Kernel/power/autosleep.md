---
title: PM(8)_autosleep
date: 2018-02-3 12:00:00
categories:
- Kernel
- Power
---
# 源码
Autosleep.c (linux-4.13.1\kernel\power)

# 功能
创建了一个有序队列，一直尝试进入休眠(模拟echo mem > /sys/power/state的过程)
<!-- more -->
# 分析
创建一个有序队列
```c
pm_autosleep_init
	autosleep_wq = alloc_ordered_workqueue("autosleep", 0);
```
创造/sys/power/autosleep
```c
power_attr(autosleep);
```
```c
static ssize_t autosleep_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t n)
{
	if (state == PM_SUSPEND_MEM)
		state = mem_sleep_current;	// suspend_set_ops() mem_sleep_current = PM_SUSPEND_MEM;

	pm_autosleep_set_state(state);
		pm_wakep_autosleep_enabled(true);

again:
		queue_up_suspend_work();
			queue_work(autosleep_wq, &suspend_work);	// call try_to_suspend()
			/* 
			 * try_to_suspend()
			 * 模拟echo mem > /sys/power/state的过程
			 */
				// 记录处理完的wakeup events数量
				pm_get_wakeup_count(&initial_count, true);
				pm_save_wakeup_count(initial_count)
				// 启动suspend流程
				pm_suspend(autosleep_state);
				
				// next...
				goto again;
}
```
