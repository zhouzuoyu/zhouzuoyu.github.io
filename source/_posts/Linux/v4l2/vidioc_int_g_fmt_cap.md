---
title: vidioc_int_g_fmt_cap和slave中函数的对应关系
date: 2019-4-23 14:39:00
categories:
- Linux
- v4l2
---

# 结论
如果调用vidioc_int_g_fmt_cap(cam->sensor, &tv_fmt);则在slave code中搜vidioc_int_g_fmt_cap_num，就能找到对应的函数。
__{vidioc_int_g_fmt_cap_num, (v4l2_int_ioctl_func*)ioctl_g_fmt_cap}__
<!--more-->
# 定义vidioc_int_g_fmt_cap
```c
V4L2_INT_WRAPPER_1(g_fmt_cap, struct v4l2_format, *);
// 定义了vidioc_int_g_fmt_cap函数
static inline int vidioc_int_##name(struct v4l2_int_device *d,	\
					    arg_type asterisk arg)	\
	{								\
		return v4l2_int_ioctl_1(d, vidioc_int_##name##_num,	\	// vidioc_int_g_fmt_cap_num
					(void *)(unsigned long)arg);
			/*
			 * 二分查找法执行对应的func，然后执行
			 * 使用vidioc_int_g_fmt_cap_num找到slave中对应的函数
			 */
			return ((v4l2_int_ioctl_func_1 *)
				find_ioctl(d->u.slave, cmd,
					   (v4l2_int_ioctl_func *)no_such_ioctl_1))(d, arg);	// ioctl_g_fmt_cap(d, arg);
	}
```

# 调用vidioc_int_g_fmt_cap
> ioctl(fd_capture_v4l, VIDIOC_G_STD, &id)

```c
static long mxc_v4l_do_ioctl(struct file *file,
			    unsigned int ioctlnr, void *arg)
{
	case VIDIOC_G_STD: {
		v4l2_std_id *e = arg;
		retval = mxc_v4l2_g_std(cam, e);
			tv_fmt.type = V4L2_BUF_TYPE_PRIVATE;
			vidioc_int_g_fmt_cap(cam->sensor, &tv_fmt);	// 调用slave的回调函数
			*e = tv_fmt.fmt.pix.pixelformat;
		break;
	}
}
```

# 注册slave
```c
static struct v4l2_int_ioctl_desc adw10023_ioctl_desc[] = {
	{vidioc_int_g_fmt_cap_num, (v4l2_int_ioctl_func*)ioctl_g_fmt_cap},	// 定义对应关系
	...
};

static struct v4l2_int_slave adw10023_slave = {
	.ioctls = adw10023_ioctl_desc,
	.num_ioctls = ARRAY_SIZE(adw10023_ioctl_desc),
};

static struct v4l2_int_device adw10023_int_device = {
	.module = THIS_MODULE,
	.name = "adw10023",
	.type = v4l2_int_type_slave,
	.u = {
		.slave = &adw10023_slave,
	},
};

static int adw10023_probe(struct i2c_client *client,
			 const struct i2c_device_id *id)
{
	ret = v4l2_int_device_register(&adw10023_int_device);
		// 进行排序
		if (d->type == v4l2_int_type_slave)
			sort(d->u.slave->ioctls, d->u.slave->num_ioctls,
				 sizeof(struct v4l2_int_ioctl_desc),
				 &ioctl_sort_cmp, NULL);
}
```
