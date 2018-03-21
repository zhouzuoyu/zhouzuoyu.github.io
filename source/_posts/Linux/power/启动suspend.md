---
title: PM(2)_启动suspend
date: 2018-02-3 12:00:00
categories:
- Linux
- power
---

# 功能
上层启动suspend流程，如果在suspend过程中没有wakeup event，则进入suspend
1. 将现在的wakeup_count记录下来
	用于判断是否新增wakeup events
2. 启动suspend流程
	如果suspend过程中，新增wakeup events，则停止suspend

<!-- more -->
# 源码分析
```c
// 得到当前的wakeup_count
do {
	ret = read(&cnt, "/sys/power/wakeup_count");
	if (ret) {
		ret = write(cnt, "/sys/power/wakeup_count");
			pm_save_wakeup_count(val)
				saved_count = count;	// 将处理完的wakeup events总数存放在saved_count里
	} else
		countine;
} while (!ret);
// 启动suspend
write("mem", "/sys/power/state");
```
