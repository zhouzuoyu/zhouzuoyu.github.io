---
title: PM(6)_runtime
date: {{ date }}
categories:
- Kernel
- Power
---

# 源码
drivers\base\power\runtime.c

# 功能
device进入suspend有两种方式，主动和被动
被动：系统启动suspend流程
主动：pm_runtime
<!-- more -->
1. 注册
各个driver: 实现driver->pm->runtime_xxx接口函数

2. 使用
在driver的适当地方call pm_runtime_get()/pm_runtime_put()
其实质是：get时usage_count++，put时usage_count--，当usage_count减到0时，代表device没有使用者，框架调用runtime_suspend关闭device

# 源码分析
## pm_runtime_get
如果不是RPM_ACTIVE，则调用driver->pm->runtime_resume
```c
pm_runtime_get
	return __pm_runtime_resume(dev, RPM_GET_PUT | RPM_ASYNC);
		// 计数器++
		atomic_inc(&dev->power.usage_count);
		rpm_resume(dev, rpmflags);
			// 已经RPM_ACTIVE则不必active again
			if (dev->power.disable_depth == 1 && dev->power.is_suspended
				&& dev->power.runtime_status == RPM_ACTIVE)
				goto out;
			// 等待 RPM_RESUMING || RPM_SUSPENDING 结束
			if (dev->power.runtime_status == RPM_RESUMING || dev->power.runtime_status == RPM_SUSPENDING) {
				for (;;) {
					if (dev->power.runtime_status != RPM_RESUMING
						&& dev->power.runtime_status != RPM_SUSPENDING)
						break;
				}
			}
			/*
			 * 同步和异步两种方式，两者选其一
			 */
			// 1: 异步方式: 启动工作队列，over
			if (rpmflags & RPM_ASYNC) {
				queue_work(pm_wq, &dev->power.work);
				goto out;
			}
			// 2: 同步方式
			// 更新状态为: RPM_RESUMING
			__update_runtime_status(dev, RPM_RESUMING);
			// 执行driver->pm->runtime_resume()
			callback = RPM_GET_CALLBACK(dev, runtime_resume);
				__rpm_get_callback(dev, offsetof(struct dev_pm_ops, callback))
					cb = *(pm_callback_t *)((void *)dev->driver->pm + cb_offset);
			rpm_callback(callback, dev);
				__rpm_callback(cb, dev);
					retval = cb(dev);
			// 更新状态为: RPM_ACTIVE
			__update_runtime_status(dev, RPM_ACTIVE);
```

## pm_runtime_put
如果usage_count为0，则调用driver->pm->runtime_idle || driver->pm->runtime_suspend
```c
pm_runtime_put
	__pm_runtime_idle(dev, RPM_GET_PUT | RPM_ASYNC);
		// 仍在使用，则不suspend(usage_count不为0，则返回)
		if (!atomic_dec_and_test(&dev->power.usage_count))
			return 0;
		rpm_idle(dev, rpmflags);
			// 异步方式: 启动工作队列
			if (rpmflags & RPM_ASYNC) {
				if (!dev->power.request_pending) {
					dev->power.request_pending = true;
					queue_work(pm_wq, &dev->power.work);
				}
				return 0;
			}
			// 同步方式
			// 有提供driver->pm->runtime_idle，则执行driver->pm->runtime_idle
			callback = RPM_GET_CALLBACK(dev, runtime_idle);
			retval = __rpm_callback(callback, dev);
			// 没有提供，则执行driver->pm->runtime_suspend
			rpm_suspend(dev, rpmflags | RPM_AUTO);
				// 等待 RPM_SUSPENDING 结束
				if (dev->power.runtime_status == RPM_SUSPENDING) {
					for (;;) {
						if (dev->power.runtime_status != RPM_SUSPENDING)
							break;
					}
				}
				// 异步
				if (rpmflags & RPM_ASYNC) {
					queue_work(pm_wq, &dev->power.work);
					goto out;
				}
				// 同步
				// 更新状态为: RPM_SUSPENDING
				__update_runtime_status(dev, RPM_SUSPENDING);
				// 执行driver->pm->runtime_suspend()
				callback = RPM_GET_CALLBACK(dev, runtime_suspend);
				retval = rpm_callback(callback, dev);
				// 更新状态为: RPM_SUSPENDED
				__update_runtime_status(dev, RPM_SUSPENDED);
```

# 例子
i2c_new_device将会pm_runtime_init
```c
i2c_new_device
	device_register(&client->dev);
		device_initialize(dev);
			device_pm_init(dev);
				pm_runtime_init(dev);
					dev->power.runtime_status = RPM_SUSPENDED;
					dev->power.disable_depth = 1;
					atomic_set(&dev->power.usage_count, 0);
```
```c
const struct dev_pm_ops cyttsp4_pm_ops = {
	SET_SYSTEM_SLEEP_PM_OPS(cyttsp4_core_suspend, cyttsp4_core_resume)
	SET_RUNTIME_PM_OPS(cyttsp4_core_suspend, cyttsp4_core_resume, NULL)
};

SET_RUNTIME_PM_OPS(cyttsp4_core_suspend, cyttsp4_core_resume, NULL)
```
