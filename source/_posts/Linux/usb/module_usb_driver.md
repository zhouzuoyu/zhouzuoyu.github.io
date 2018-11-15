---
title: module_usb_driver
date: 2018-11-14 15:58:00
categories:
- Linux
- usb
---

# module_usb_driver
> module_usb_driver(usb_storage_driver);

宏展开如下：
```c
#define module_usb_driver(__usb_driver) \
	module_driver(__usb_driver, usb_register, \
		       usb_deregister)

#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ \
	return __register(&(__driver) , ##__VA_ARGS__); \
} \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \
	__unregister(&(__driver) , ##__VA_ARGS__); \
} \
module_exit(__driver##_exit);
```
<!--more-->
所以等价于:
```c
static int __init usb_storage_driver_init(void)
{
	return usb_register(&(usb_storage_driver));
}
module_init(usb_storage_driver_init);
static int __init usb_storage_driver_exit(void)
{
	return usb_deregister(&(usb_storage_driver));
}
module_exit(usb_storage_driver_exit);
```

# usb_register
挂载在usb_bus_type上
```c
#define usb_register(driver) \
	usb_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)

int usb_register_driver(struct usb_driver *new_driver, struct module *owner,
			const char *mod_name)
{
	new_driver->drvwrap.for_devices = 0;
	new_driver->drvwrap.driver.bus = &usb_bus_type;

	retval = driver_register(&new_driver->drvwrap.driver);
}
```
