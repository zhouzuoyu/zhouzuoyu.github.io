---
title: ldb
date: 2019-06-06 16:44:00
categories:
- imx6
---

# 注册
> drivers\video\fbdev\mxc\ldb.c

注册handle
```c
static struct mxc_dispdrv_driver ldb_drv = {
	.name		= DRIVER_NAME,
	.init		= ldb_init,
	.setup		= ldb_setup,
	.enable		= ldb_enable,
	.disable	= ldb_disable
};

static int ldb_probe(struct platform_device *pdev)
{
	ldb->mddh = mxc_dispdrv_register(&ldb_drv);
		new = kzalloc(sizeof(struct mxc_dispdrv_entry), GFP_KERNEL);
		new->drv = drv;
		list_add_tail(&new->list, &dispdrv_list);	// 挂载到dispdrv_list
}
```
<!--more-->
# 使用
> drivers\video\fbdev\mxc\mxc_ipuv3_fb.c

获取handle
```c
static int mxcfb_probe(struct platform_device *pdev)
{
	ret = mxcfb_dispdrv_init(pdev, fbi);
		mxcfbi->dispdrv = mxc_dispdrv_gethandle(disp_dev, &setting);	// 得到ldb.c中的mxc_dispdrv_driver
			list_for_each_entry(entry, &dispdrv_list, list) {
				if (!strcmp(entry->drv->name, name) && (entry->drv->init)) {	// 通过name来进行匹配
					ret = entry->drv->init((struct mxc_dispdrv_handle *)
						entry, setting);
					if (ret >= 0) {
						entry->active = true;
						found = 1;
						break;
					}
				}
			}
			return found ? (struct mxc_dispdrv_handle *)entry:ERR_PTR(-ENODEV);
}
```

使用handle
```c
static int mxcfb_set_par(struct fb_info *fbi)
{
	retval = mxc_fbi->dispdrv->drv->enable(mxc_fbi->dispdrv, fbi);
}
```
