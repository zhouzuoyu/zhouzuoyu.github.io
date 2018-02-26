---
title: PM(5)_wakeup_count
date: 2018-02-3 12:00:00
categories:
- Linux
- Power
---
# 源码
> kernel\power\main.c

# 功能
创建/sys/power/wakeup_count节点，用于回存wakeup events数量
启动suspend前：__先读取再写入wakeup_count__，得到当前的wakeup events数量
启动suspend：suspend流程中会比较wakeup events数量，如果不一致，说明在suspend流程中有wakeup event发生，则中断suspend
<!-- more -->
# 源码分析
## wakeup count节点
```c
power_attr(wakeup_count);
```

## show
读取处理完的wakeup counter
```c
wakeup_count_show
	return pm_get_wakeup_count(&val, true) ? sprintf(buf, "%u\n", val) : -EINTR;
		// 等待wakeup events全部处理完
		for (;;) {
			split_counters(&cnt, &inpr);
			if (inpr == 0 || signal_pending(current))
				break;
		}
		// 返回处理完的wakeup counter
		split_counters(&cnt, &inpr);
		*count = cnt;
```

## store
保存处理完的wakeup counter
```c
wakeup_count_store
	pm_save_wakeup_count(val)
		saved_count = count;	// 将处理完的wakeup events总数存放在saved_count里
```
