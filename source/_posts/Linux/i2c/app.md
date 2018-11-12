---
title: i2c app
date: 2018-11-12 10:48:00
categories:
- Linux
- i2c
---

# 原理
需要构建i2c_rdwr_ioctl_data，通过ioctl(fd,I2C_RDWR,&packets)进行下发。
ps: 此时ioctl会返回-1，但是属于正常情况(参考i2c-tools源码)。
```c
__s32 i2c_smbus_access(int file, char read_write, __u8 command,
		       int size, union i2c_smbus_data *data)
{
	struct i2c_smbus_ioctl_data args;
	__s32 err;

	args.read_write = read_write;
	args.command = command;
	args.size = size;
	args.data = data;

	err = ioctl(file, I2C_SMBUS, &args);
	if (err == -1)
		err = -errno;
	return err;
}
```
<!--more-->

# 源码
## app
```c
#include <stdio.h>
#include <linux/types.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <errno.h>
#include <linux/i2c.h>
#include <linux/i2c-dev.h>

typedef unsigned char u8;
typedef unsigned short u16;
typedef unsigned int u32;

int main()
{
	int ret, fd;
	struct i2c_msg msg[2];
	u8 val = 0;

	fd = open("/dev/i2c-3",O_RDWR);
	if (fd < 0){
		printf("open i2c-3 controller failed\n");
		return fd;
	}

	// 先设置要读取的寄存器地址
	val = 0x01;				// reg addr
	msg[0].addr = 0x2c;		// i2c device addr
	msg[0].flags = 0;		// write
	msg[0].len = 1;
	msg[0].buf[0] = &val;
	// 再读取len个数据到buf中
	msg[1].addr = 0x2c;		// i2c device addr
	msg[1].flags = 1;		// read
	msg[1].len = 1;
	msg[1].buf[0] = &val;

	// 下发数据
	struct i2c_rdwr_ioctl_data packets;
	packets.msgs	= msg;
	packets.nmsgs	= 2;

	ret = ioctl(fd,I2C_RDWR,&packets);
	if ((ret<0) && (ret!=-1)) {	// -1是正常情况
		printf("[I2C] ioctl failed ret:%d\n",ret);
		return ret;
	}

	printf("[I2C] read result is %x\n",msg[1].buf[0]);

	close(fd);

	return 0;
}
```

## 底层对应code分析
```c
static const struct file_operations i2cdev_fops = {
	.owner			= THIS_MODULE,
	.unlocked_ioctl	= i2cdev_ioctl,
};

static long i2cdev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
	switch (cmd) {
	case I2C_RDWR:
		return i2cdev_ioctl_rdrw(client, arg);
			// 先要知道有几个msg
			if (copy_from_user(&rdwr_arg,
					   (struct i2c_rdwr_ioctl_data __user *)arg,
					   sizeof(rdwr_arg)))
				return -EFAULT;
			// 再将这几个msg dump到rdwr_pa
			rdwr_pa = memdup_user(rdwr_arg.msgs,
						  rdwr_arg.nmsgs * sizeof(struct i2c_msg));

			// 构建msg
			for (i = 0; i < rdwr_arg.nmsgs; i++) {
				data_ptrs[i] = (u8 __user *)rdwr_pa[i].buf;
				rdwr_pa[i].buf = memdup_user(data_ptrs[i], rdwr_pa[i].len);
			}
			// 发送msg
			res = i2c_transfer(client->adapter, rdwr_pa, rdwr_arg.nmsgs);
			// cp数据到userspace
			while (i-- > 0) {
				if (res >= 0 && (rdwr_pa[i].flags & I2C_M_RD)) {
					if (copy_to_user(data_ptrs[i], rdwr_pa[i].buf,
							 rdwr_pa[i].len))
						res = -EFAULT;
				}
			}
	}
}
```
