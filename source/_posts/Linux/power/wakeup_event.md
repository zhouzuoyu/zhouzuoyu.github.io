---
title: PM(4)_wakeup_event
date: 2018-02-3 12:00:00
categories:
- Linux
- power
---

# 源码
Wakeup.c (linux-4.13.1\drivers\base\power)

# 功能
因为android的广泛使用，导致linux suspend的使用场景异常频繁，急需一种同步机制来保证suspend的正常执行，这就是wakeup_event
其心是一个计数器：__combined_event_count__
这个计数器存放了两个部分
part 1：__处理完__的wakeup event数量
part 2：__处理中__的wakeup event数量，如不为0则系统不能suspend
<!-- more -->
# 分析
## suspend流程与wakeup event联系
启动suspend，会检查wakeup event
```c
suspend_enter
	*wakeup = pm_wakeup_pending();	// 最后关头check有无wakeup event仍未处理，都处理完了则进入平台的enter函数，系统进入suspend
		split_counters(&cnt, &inpr);
		ret = (cnt != saved_count || inpr > 0);	// suspend时新增wakeup events，则停止suspend
		return ret || atomic_read(&pm_abort_suspend) > 0;
	if (!(*wakeup))
		suspend_ops->enter(state);
```

## wakeup event API
```c
static void split_counters(unsigned int *cnt, unsigned int *inpr)
{
	unsigned int comb = atomic_read(&combined_event_count);

	*cnt = (comb >> IN_PROGRESS_BITS);	// 处理完的wakeup event数量
	*inpr = comb & MAX_IN_PROGRESS;		// 待处理中的wakeup event数量，如不为0则系统不能suspend
}
```
```c
wakeup_source_activate
	atomic_inc_return(&combined_event_count);	// wakeup event增加，如数目不为零，则系统不能suspend
```
```c
wakeup_source_deactivate
	atomic_add_return(MAX_IN_PROGRESS, &combined_event_count);	// wakeup events in progress减1，registered wakeup events加1。
```

# 例子
> drivers\input\keyboard\gpio_keys.c

keypad这个wakeup source中断中调用wakeup_source_activate，则系统不能suspend
```c
gpio_keys_probe
	gpio_keys_setup_key(pdev, input, ddata, button, i, child);
		isr = gpio_keys_irq_isr;
			pm_wakeup_event(bdata->input->dev.parent, 0);
				return pm_wakeup_dev_event(dev, msec, false);
					pm_wakeup_ws_event(dev->power.wakeup, msec, hard);
						wakeup_source_report_event(ws, hard);
							// wakeup event cnt++ && 根据参数是否freeze_wake
							wakeup_source_activate(ws);
								atomic_inc_return(&combined_event_count);	// wakeup event cnt++
							if (hard)
								pm_system_wakeup();
									freeze_wake();
						wakeup_source_deactivate(ws);
							atomic_add_return(MAX_IN_PROGRESS, &combined_event_count);
		devm_request_any_context_irq(dev, bdata->irq, isr, irqflags, desc, bdata); // register irq
```
