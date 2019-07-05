---
title: uevent helper
date: 2019-07-01 16:43:00
categories:
- Linux
---

mdev使用uevent_helper机制，即__调用kobject_uevent_env就会触发uevent_helper所指定的用户程序(mdev)__

# 设置uevent_helper
> echo "/sbin/mdev" > /proc/sys/kernel/hotplug

```c
static ssize_t uevent_helper_show(struct kobject *kobj,
				  struct kobj_attribute *attr, char *buf)
{
	return sprintf(buf, "%s\n", uevent_helper);
}

static ssize_t uevent_helper_store(struct kobject *kobj,
				   struct kobj_attribute *attr,
				   const char *buf, size_t count)
{
	if (count+1 > UEVENT_HELPER_PATH_LEN)
		return -ENOENT;
	memcpy(uevent_helper, buf, count);	// 存到uevent_helper中
	uevent_helper[count] = '\0';
	if (count && uevent_helper[count-1] == '\n')
		uevent_helper[count-1] = '\0';
	return count;
}

KERNEL_ATTR_RW(uevent_helper);	// uevent_helper_attr
```
<!--more-->
```c
static struct attribute * kernel_attrs[] = {
	&uevent_helper_attr.attr,
	NULL
};

static struct attribute_group kernel_attr_group = {
	.attrs = kernel_attrs,
};

static int __init ksysfs_init(void)
{
	kernel_kobj = kobject_create_and_add("kernel", NULL);
	error = sysfs_create_group(kernel_kobj, &kernel_attr_group);	// /proc/sys/kernel/hotplug
}

core_initcall(ksysfs_init);
```

# 使用uevent_helper
```c
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
			   char *envp_ext[])
{
	retval = add_uevent_var(env, "HOME=/");

	retval = add_uevent_var(env,
				"PATH=/sbin:/bin:/usr/sbin:/usr/bin");

	retval = init_uevent_argv(env, subsystem);
		env->argv[0] = uevent_helper;

	info = call_usermodehelper_setup(env->argv[0], env->argv,	// param 1: path to usermode executable == uevent_helper
					 env->envp, GFP_KERNEL,
					 NULL, cleanup_uevent_env, env);
	if (info) {
		retval = call_usermodehelper_exec(info, UMH_NO_WAIT);	// 调用应用程序
		env = NULL;	/* freed by cleanup_uevent_env */
	}
}
```
