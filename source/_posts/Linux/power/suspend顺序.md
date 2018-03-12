---
title: suspend顺序
date: 2018-03-07 10:43:15
categories:
- Linux
- power
---

# 导言
以i2c为例，有i2c_adapter和i2c_client两类设备，该文章描述这两类设备的suspend顺序，从逻辑来看必然是先关闭所有的i2c_client才能关闭i2c_adapter。

从{% post_link Linux/i2c/i2c_client i2c_client %}中可以得知，i2c_adapter必然是在i2c_client之前注册上的，且越早注册的device越晚调用其suspend func，所以所有的i2c_client都callback suspend func才会调用i2c_adapter的suspend func。
看的是device注册顺序而不是driver的顺序，所以和module_init, rootfs_init, subsys_initcall这些无关。

<!--more-->
# suspend调用顺序
从{% post_link Kernel/power/suspend流程 suspend流程 %}可以看到所有device都挂接在dpm_list这个链表上
最后通过to_device(dpm_prepared_list.prev)反向取出，**故越早注册的device越晚调用suspend**
```c
int suspend_devices_and_enter(suspend_state_t state)
	dpm_suspend_start(PMSG_SUSPEND);
		// part 1: 将dpm_list转到dpm_prepared_list
		dpm_prepare(state);
			while (!list_empty(&dpm_list)) {
				struct device *dev = to_device(dpm_list.next);
				dev->power.is_prepared = true;
				list_move_tail(&dev->power.entry, &dpm_prepared_list);
			}
		// part 2: 对每个device做suspend
		dpm_suspend(state);
			while (!list_empty(&dpm_prepared_list)) {
				struct device *dev = to_device(dpm_prepared_list.prev);	// FIXME: 注意这是prev
				device_suspend(dev);
				list_move(&dev->power.entry, &dpm_suspended_list);
			}
```
