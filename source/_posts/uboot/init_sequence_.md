---
title: init_sequence_f/init_sequence_r
date: 2019-06-06 16:40:00
categories:
- uboot
---

> arch/arm/lib/crt0.S

```c
ENTRY(_main)
	bl	board_init_f
	ldr	pc, =board_init_r
```
__先调用各种初始化函数(init_sequence_f/init_sequence_r)，最后在main_loop中执行配置文件中的命令__
<!--more-->
# board_init_f
> board_f.c

```c
static init_fnc_t init_sequence_f[] = {
	...
};

void board_init_f(ulong boot_flags)
{
	initcall_run_list(init_sequence_f);
		// 调用init_sequence_f中的每个成员
		for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
			ret = (*init_fnc_ptr)();
		}
}
```

# board_init_r
> board_r.c

```c
// 函数指针数组
init_fnc_t init_sequence_r[] = {
	...
	board_init,		// 板级初始化，在各个板级文件中
	run_main_loop,	// 最后会调用
};

void board_init_r(gd_t *new_gd, ulong dest_addr)
{
	initcall_run_list(init_sequence_r);
		// 调用init_sequence_r中的每个成员
		for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
			ret = (*init_fnc_ptr)();
		}
}
```

```c
// 主要执行配置文件中的那些指令
static int run_main_loop(void)
{
	/* main_loop() can return to retry autoboot, if so just run it again */
	for (;;)
		/* We come here after U-Boot is initialised and ready to process commands */
		main_loop();
			autoboot_command(s);

	return 0;
}
```
