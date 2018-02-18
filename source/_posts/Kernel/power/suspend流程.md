---
title: PM(3)_suspend流程
date: 2018-02-3 12:00:00
categories:
- Kernel
- Power
---

# 源码
1. 总框架
  在/sys/power/下创建各类接口
> kernel\power\main.c

2. /sys/power/state模块代码
  suspend_set_ops: link平台代码
> kernel\power\suspend.c

<!-- more -->
3. 平台相关的suspend & resume函数
> arch\arm\plat-samsung\pm.c
> arch\arm\mach-s3c24xx\pm.c

4. Device suspend & resume的驱动框架和API
> drivers\base\power\main.c

# 功能
__echo mem > /sys/power/state__将会导致系统进入suspend流程
1：各个devices的suspend函数被调用
2：平台端suspend被调用

# 源码分析
## /sys/power/state
1. 注册/sys/power/state
	```c
	power_attr(state);
	static struct attribute * g[] = {
		&state_attr.attr
	};
	pm_init
		// 创建/sys/power/xxx属性节点
		power_kobj = kobject_create_and_add("power", NULL);
		error = sysfs_create_group(power_kobj, &attr_group);
	```

2. suspend流程
	写/sys/power/state将会调用该函数
	```c
	state_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t n)
		error = pm_suspend(state);
			error = enter_state(state);
				// 判断valid
				valid_state(state)
					suspend_ops->valid(state);	// suspend_valid_only_mem: 只支持STR模式
				// prepare <--> finish
				suspend_prepare(state);
					pm_prepare_console();
					suspend_freeze_processes();
			
				// suspend & resume
				suspend_devices_and_enter(state);
					// Device端的suspend
					dpm_suspend_start(PMSG_SUSPEND);
					// 平台端的suspend和resume
					do {
						error = suspend_enter(state, &wakeup);
							// suspend
							platform_suspend_prepare(state);// suspend_ops->prepare(): s3c_pm_prepare
							disable_nonboot_cpus();
							arch_suspend_disable_irqs();
							syscore_suspend();
							suspend_ops->enter(state);		// FIXME: s3c_pm_enter
							// resume
							syscore_resume();
							arch_suspend_enable_irqs();
							enable_nonboot_cpus();
							platform_resume_finish(state);	// suspend_ops->finish(): s3c_pm_finish
					} while (!error && !wakeup && platform_suspend_again(state));
					// Device端的resume
					dpm_resume_end(PMSG_RESUME);
				// finish <--> prepare
				suspend_finish();
					suspend_thaw_processes();
					pm_restore_console();
	```

## 平台suspend_ops
填充platform_suspend_ops
```c
/* register platform pm ops */
static const struct platform_suspend_ops s3c_pm_ops = {
	.enter		= s3c_pm_enter,	// 平台端的suspend函数
	.prepare	= s3c_pm_prepare,
	.finish		= s3c_pm_finish,
	.valid		= suspend_valid_only_mem,
};
s3c_pm_enter
	// check extint resource
	s3c_pm_configure_extint();
	// exec platform suspend func: assemble code
	ret = cpu_suspend(0, pm_cpu_sleep);
		pm_cpu_sleep = s3c2410_cpu_suspend;	// pm_cpu_sleep form
		ENTRY(s3c2410_cpu_suspend)	// linux-4.13.1\arch\arm\mach-s3c24xx\sleep-s3c2410.S

s3c_pm_init
	suspend_set_ops(&s3c_pm_ops);
		suspend_ops = ops;	// FIXME: suspend.c
```

## Device suspend/resume
1. register
	```
	int device_add(struct device *dev)
		device_pm_add(dev);
			list_add_tail(&dev->power.entry, &dpm_list);	// dpm_list
	```

2. device suspend
	callback func:
	　　dev->pm_domain->ops
	　　dev->type->pm
	　　dev->class->pm
	　　dev->bus->pm
	　　dev->driver->pm
	涉及dpm_list/dpm_prepared_list/dpm_suspended_list等链表
	```c
	dpm_suspend_start(PMSG_SUSPEND);
		dpm_prepare(state);
			while (!list_empty(&dpm_list)) {
				device_prepare(dev, state);	// callback prepare func.
					callback = dev->pm_domain->ops.prepare; || dev->type->pm->prepare; || callback = dev->class->pm->prepare; || callback = dev->bus->pm->prepare; || callback = dev->driver->pm->prepare;
					callback(dev);
				list_move_tail(&dev->power.entry, &dpm_prepared_list);	// dpm_list -> dpm_prepared_list
			}
		dpm_suspend(state);
			while (!list_empty(&dpm_prepared_list)) {
				device_suspend(dev);
					__device_suspend(dev, pm_transition, false);
						callback = pm_op(&dev->pm_domain->ops, state); || pm_op(dev->type->pm, state); || pm_op(dev->class->pm, state); || pm_op(dev->bus->pm, state); || pm_op(dev->driver->pm, state);
						dpm_run_callback(callback, dev, state, info);
				list_move(&dev->power.entry, &dpm_suspended_list);		// dpm_prepared_list -> dpm_suspended_list
			}
	```

3. device resume
	```c
	dpm_resume_end(PMSG_RESUME);
		dpm_resume(state);
			while (!list_empty(&dpm_suspended_list)) {
				device_resume(dev, state, false);
					callback = pm_op(&dev->pm_domain->ops, state); || pm_op(dev->type->pm, state); || pm_op(dev->class->pm, state); || pm_op(dev->bus->pm, state); || pm_op(dev->driver->pm, state);
					dpm_run_callback(callback, dev, state, info);
				list_move_tail(&dev->power.entry, &dpm_prepared_list);	// dpm_suspended_list -> dpm_prepared_list
			}
		dpm_complete(state);
			while (!list_empty(&dpm_prepared_list)) {
				list_move(&dev->power.entry, &list);
				device_complete(dev, state);
					callback = dev->pm_domain->ops.complete; || dev->type->pm->complete; || dev->class->pm->complete; || dev->bus->pm->complete; || dev->driver->pm->complete;
					callback(dev);
			}
			list_splice(&list, &dpm_list);								// dpm_prepared_list -> dpm_list
	```
