---
title: timer
date: 2019-05-13 10:26:00
categories:
- Linux
- timer
---

# 初始化
```c
static int probe(struct platform_device *pdev)
{
	// 初始化
	init_timer(&engpio_delay_timer);
	engpio_delay_timer.function = engpio_delay_timer_func;
	engpio_delay_timer.data = NULL;
}

static int remove(struct platform_device *pdev)
{
	del_timer(&engpio_delay_timer);

	return 0;
}
```
<!--more-->
# 修改触发时间
__1HZ=1s，所以触发时间是1s/20=50ms__
```c
printk("Enter %s\n", __func__);
mod_timer(&engpio_delay_timer, jiffies + HZ/20);
```

# 时间到了触发指定function
```c
static void engpio_delay_timer_func(u_long a)
{
	printk("Enter %s\n", __func__);
}
```

# log
// 时间间隔50ms
>[    0.172768] Enter ldb_enable
>[    0.222668] Enter engpio_delay_timer_func
