---
title: 字符设备(2)_Misc设备
date: {{ date }}
categories: 
- Kernel
- Chrdev
---

1. 源码：

   linux-3.0.86\drivers\char\misc.c

2. 功能：

   调用misc_register注册misc设备
<!-- more -->
3. 源码分析：

    先看注册字符设备部分，init中字符设备只注册了一部分，另一部分需要misc_register实现
    所有主设备号为MISC_MAJOR的字符设备都使用misc_fops

    ```c
    misc_init
            misc_class = class_create(THIS_MODULE, "misc");
            register_chrdev(MISC_MAJOR,"misc",&misc_fops);
    int misc_register(struct miscdevice * misc)
            // minor有两种方式，可以在次设备号驱动中指定，也可以指定为MISC_DYNAMIC_MINOR，misc驱动框架中找到空位自动分配
            if (misc->minor == MISC_DYNAMIC_MINOR) {
                int i = find_first_zero_bit(misc_minors, DYNAMIC_MINORS);
                misc->minor = DYNAMIC_MINORS - i - 1;
                set_bit(i, misc_minors);
            }
            dev = MKDEV(MISC_MAJOR, misc->minor);

            device_create(misc_class, misc->parent, dev, misc, "%s", misc->name);
    ```

    misc_fops只有misc_open，起中转作用，会替换到次设备号驱动的fops，所以read，write会调用到次设备号驱动fops的对应函数

    ```c
    static const struct file_operations misc_fops = {
            .owner		= THIS_MODULE,
            .open		= misc_open,
            .llseek		= noop_llseek,
    };
    static int misc_open(struct inode * inode, struct file * file)
            // search真正的fops
            list_for_each_entry(c, &misc_list, list) {
	            if (c->minor == minor) {
		            new_fops = fops_get(c->fops);    // 通过次设备号得到真正的fops
		            break;
	            }
            }
            // 转化fops
            old_fops = file->f_op;
            file->f_op = new_fops;
            // 调用真正的fops->open
            if (file->f_op->open) {
	            file->private_data = c;
                err=file->f_op->open(inode,file);
            }
        }
    ```

4. 实例
    最后看一个使用例子，只需要指定minor，fops即可
    ```c
    static struct miscdevice acq_miscdev = {
        .minor	= WATCHDOG_MINOR,    // 也可以指定为MISC_DYNAMIC_MINOR，自动分配
        .name	= "watchdog",
        .fops	= &acq_fops,
    };
    acq_probe
        misc_register(&acq_miscdev);
    ```
