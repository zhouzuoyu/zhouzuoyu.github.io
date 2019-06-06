---
title: bootdelay
date: 2019-06-06 15:44:00
categories:
- uboot
---

__有两种配置方法__
> bootdelay=3
> #define CONFIG_BOOTDELAY	1

<!--more-->
```c
void main_loop(void)
{
	// 获取delay时间
	s = bootdelay_process();
		s = getenv("bootdelay");
		bootdelay = s ? (int)simple_strtol(s, NULL, 10) : CONFIG_BOOTDELAY;
		stored_bootdelay = bootdelay;

	autoboot_command(s);
		// 延迟指定时间
		abortboot(stored_bootdelay);
			while ((bootdelay > 0) && (!abort)) {
				--bootdelay;
				/* delay 1000 ms */
				ts = get_timer(0);
				do {
					udelay(10000);	// 从这个代码来看，这个延时时间是1s
				} while (!abort && get_timer(ts) < 1000);

				printf("\b\b\b%2d ", bootdelay);
			}

		// 启动
		s = getenv("bootcmd");
		run_command_list(s, -1, 0);
}
```
