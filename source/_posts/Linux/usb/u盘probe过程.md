---
title: u盘probe过程
date: 2018-11-15 10:49:00
categories:
- Linux
- usb
---

# 匹配方式
一般u盘都是使用usb_storage_usb_ids中的USUAL_DEV(USB_SC_SCSI, USB_PR_BULK)匹配成功的。

> lsusb -v

```c
      bInterfaceClass         8 Mass Storage
      bInterfaceSubClass      6 SCSI
      bInterfaceProtocol     80 Bulk-Only
```
<!--more-->

> USUAL_DEV(USB_SC_SCSI, USB_PR_BULK)

```c
#define USB_CLASS_MASS_STORAGE	8
#define USB_SC_SCSI				0x06	/* Transparent */
#define USB_PR_BULK				0x50	/* bulk only */

#define USB_DEVICE_ID_MATCH_INT_INFO \
		(USB_DEVICE_ID_MATCH_INT_CLASS | \
		USB_DEVICE_ID_MATCH_INT_SUBCLASS | \
		USB_DEVICE_ID_MATCH_INT_PROTOCOL)

#define USB_INTERFACE_INFO(cl, sc, pr) \
	.match_flags = USB_DEVICE_ID_MATCH_INT_INFO, \
	.bInterfaceClass = (cl), \
	.bInterfaceSubClass = (sc), \
	.bInterfaceProtocol = (pr)

#define USUAL_DEV(useProto, useTrans) \
{ USB_INTERFACE_INFO(USB_CLASS_MASS_STORAGE, useProto, useTrans) }

USUAL_DEV(USB_SC_SCSI, USB_PR_BULK),

struct usb_device_id usb_storage_usb_ids[] = {
#	include "unusual_devs.h"
	{ }		/* Terminating entry */
};

static struct usb_driver usb_storage_driver = {
	.id_table =	usb_storage_usb_ids,
};

module_usb_driver(usb_storage_driver);
```
以上，所以匹配。

# 代码分析
插入一个usb设备，有两次probe过程。
第一次是usb device的probe，第二次是usb interface的probe，usb_device_match会执行两次。
```c
static int usb_device_match(struct device *dev, struct device_driver *drv)
{
	/* devices and interfaces are handled separately */
	if (is_usb_device(dev)) {
		// 第一次
		/* interface drivers never match devices */
		if (!is_usb_device_driver(drv))
			return 0;

		/* TODO: Add real matching code */
		return 1;

	} else if (is_usb_interface(dev)) {
		// 第二次
		struct usb_interface *intf;
		struct usb_driver *usb_drv;
		const struct usb_device_id *id;

		/* device drivers never match interfaces */
		if (is_usb_device_driver(drv))
			return 0;

		intf = to_usb_interface(dev);
		usb_drv = to_usb_driver(drv);

		id = usb_match_id(intf, usb_drv->id_table);
		if (id)
			return 1;

		id = usb_match_dynamic_id(intf, usb_drv);
		if (id)
			return 1;
	}

	return 0;
}
```

## 注册了两种driver
都是挂载在usb_bus_type上
### usb_generic_driver
> struct usb_device_driver usb_generic_driver;

```c
struct usb_device_driver usb_generic_driver = {
	.probe = generic_probe,
};

static int __init usb_init(void)
{
	retval = usb_register_device_driver(&usb_generic_driver, THIS_MODULE);
		new_udriver->drvwrap.for_devices = 1;
		new_udriver->drvwrap.driver.bus = &usb_bus_type;
		new_udriver->drvwrap.driver.probe = usb_probe_device;

		retval = driver_register(&new_udriver->drvwrap.driver);
}
```

### usb_storage_driver
> static struct usb_driver usb_storage_driver;

```c
static struct usb_driver usb_storage_driver = {
	.name =		"usb-storage",
	.probe =	storage_probe,
	.id_table =	usb_storage_usb_ids,
};

module_usb_driver(usb_storage_driver);
```

## 第一次probe过程
1. 创建dev
```c
hub_port_connect(hub, port1, portstatus, portchange);
	// alloc
	udev = usb_alloc_dev(hdev, hdev->bus, port1);
		dev->dev.bus = &usb_bus_type;
		dev->dev.type = &usb_device_type;
		root_hub = 1;
		if (root_hub)	/* Root hub always ok [and always wired] */
			dev->authorized = 1;
	// new
	status = usb_new_device(udev);
		err = usb_enumerate_device(udev);	/* Read descriptors */
		err = device_add(&udev->dev);
```

2. match
调用usb_bus_type.match函数，因为usb_device_type，所以进**if分支**
```c
static int usb_device_match(struct device *dev, struct device_driver *drv)
{
	if (is_usb_device(dev)) {
		return 1;
	}
}
```

3. probe
match成功，调用usb_generic_driver.probe。
generic_probe中又创建了dev，所以引发第二次probe
```c
static int generic_probe(struct usb_device *udev)
{
	err = usb_set_configuration(udev, c);
		// 为device创建interface
		cp->interface[i] = intf = new_interfaces[i];
		intf->dev.bus = &usb_bus_type;
		intf->dev.type = &usb_if_device_type;

		ret = device_add(&intf->dev);	// interface
}
```

## 第二次probe过程
1. match
调用usb_bus_type.match函数，因为usb_if_device_type，所以进**else分支**
```c
static int usb_device_match(struct device *dev, struct device_driver *drv)
{
	else if (is_usb_interface(dev)) {
		id = usb_match_id(intf, usb_drv->id_table);
			for (; id->idVendor || id->idProduct || id->bDeviceClass ||
				   id->bInterfaceClass || id->driver_info; id++) {
				if (usb_match_one_id(interface, id))
					return usb_match_one_id_intf(dev, intf, id);
						// 因为match_flags = USB_DEVICE_ID_MATCH_INT_INFO，所以以下都要匹配才能return 1
						if ((id->match_flags & USB_DEVICE_ID_MATCH_INT_CLASS) &&
							(id->bInterfaceClass != intf->desc.bInterfaceClass))
							return 0;
						if ((id->match_flags & USB_DEVICE_ID_MATCH_INT_SUBCLASS) &&
							(id->bInterfaceSubClass != intf->desc.bInterfaceSubClass))
							return 0;
						if ((id->match_flags & USB_DEVICE_ID_MATCH_INT_PROTOCOL) &&
							(id->bInterfaceProtocol != intf->desc.bInterfaceProtocol))
							return 0;
						return 1;

					return id;
			}

		if (id)
			return 1;
	}

	return 0;
}
```

2. probe
调用usb_storage_driver.probe
```c
static int storage_probe(struct usb_interface *intf,
			 const struct usb_device_id *id)
{
	result = usb_stor_probe1(&us, intf, id, unusual_dev);

	result = usb_stor_probe2(us);
}
```
