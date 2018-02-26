---
title: PM(7)_wakelock
date: 2018-02-3 12:00:00
categories:
- Linux
- Power
---

# 源码
* kernel\power\main.c
创建/sys/power/wake_lock和/sys/power/wake_unlock节点

* kernel\power\wakelock.c
wakelock功能的具体实现

* kernel\power\wakeup.c
wakup event的API接口函数

<!-- more -->
# 功能
在/sys/power/下创建wake_lock和wake_unlock
往wake_lock中写一个字串(其本质是添加了wakeup event)，则不允许系统进入suspend
往wake_unlock写相同字串，则取消往wake_lock

# 源码分析
## 创建节点
```c
power_attr(wake_lock);
power_attr(wake_unlock);
```

## wake_lock
```c
static ssize_t wake_lock_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t n)
{
	int error = pm_wake_lock(buf);
		// 1: check这个wakelock是否已经创建过，没有则new一个
		wakelock_lookup_add(buf, len, true);	// true: 如果搜索不到则new一个
			wakelocks_lru_add(wl);
				list_add(&wl->lru, &wakelocks_lru_list);
		// 2: active一个wakeup event事件
		__pm_stay_awake(&wl->ws);
			wakeup_source_report_event(ws, false);
				wakeup_source_activate(ws);	// up
					ws->active = true;
					atomic_inc_return(&combined_event_count);
		// 3:
		wakelocks_lru_most_recent(wl);
			list_move(&wl->lru, &wakelocks_lru_list);
	return error ? error : n;
}
```

## wake_unlock
```c
static ssize_t wake_unlock_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t n)
{
	int error = pm_wake_unlock(buf);
		// 1: search wakelock
		wl = wakelock_lookup_add(buf, len, false);	// false: 只在原来的树上查找
		// 2: deactive一个wakeup event事件
		__pm_relax(&wl->ws);
			wakeup_source_deactivate(ws);	// down
				atomic_add_return(MAX_IN_PROGRESS, &combined_event_count);
		// 3: free
		wakelocks_lru_most_recent(wl);
			list_move(&wl->lru, &wakelocks_lru_list);
		wakelocks_gc();
			schedule_work(&wakelock_work);
				list_del(&wl->lru);
				
	return error ? error : n;
}
```
