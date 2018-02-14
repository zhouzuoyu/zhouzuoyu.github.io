---
title: i2c(3)_algorithm
date: {{ date }}
categories:
- Kernel
- i2c
---

## 源码
> linux-3.0.86\drivers\i2c\busses\i2c-s3c2410.c

## 功能
i2c协议是通用的，kernel中已经实现，但是各个厂商的i2c controller各不相同，所以需要填充差异部分
主要是要填充i2c_algorithm->master_xfer
<!-- more -->
## 源码分析
注册i2c_algorithm
```c
static u32 s3c24xx_i2c_func(struct i2c_adapter *adap)
{
	return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL | I2C_FUNC_NOSTART |
		I2C_FUNC_PROTOCOL_MANGLING;
}
// algorithm
static const struct i2c_algorithm s3c24xx_i2c_algorithm = {
	.master_xfer		= s3c24xx_i2c_xfer,
	.functionality		= s3c24xx_i2c_func,
};
s3c24xx_i2c_probe
	i2c->adap.algo = &s3c24xx_i2c_algorithm;	// i2c_algorithm挂接在i2c_adapter上
	devm_request_irq(&pdev->dev, i2c->irq, s3c24xx_i2c_irq, 0, dev_name(&pdev->dev), i2c);	// register interrupt
```

i2c传输有两种方式：
	1：polling
 	2：interrupt，IICDS数据传输完毕则产生一次中断
	以下分析interrupt方式：s3c24xx_i2c_xfer作为启动(S3C2410_IICSTAT_START)，后续的传输通过interrupt来完成

```c
/*
 * 1. Write own slave address on IICADD register, if needed.
   2. Set IICCON register.
	a) Enable interrupt
	b) Define SCL period
   3. Set IICSTAT to enable Serial Output
 */
{ .compatible = "samsung,s3c2440-i2c", .data = (void *)QUIRK_S3C2440 },	// Not polling
s3c24xx_i2c_xfer
	for (retry = 0; retry < adap->retries; retry++)	// 可以做指定次数retry
		s3c24xx_i2c_doxfer(i2c, msgs, num);
			i2c->msg     = msgs;	// i2c device设置
			i2c->msg_num = num;
			i2c->msg_ptr = 0;
			i2c->msg_idx = 0;
			i2c->state   = STATE_START;	// 1st: STATE_START
			// 使能中断
			s3c24xx_i2c_enable_irq(i2c);
				// set IICCON[5]: Tx/Rx Interrupt
				tmp = readl(i2c->regs + S3C2410_IICCON);
				writel(tmp | S3C2410_IICCON_IRQEN, i2c->regs + S3C2410_IICCON);
			//
			s3c24xx_i2c_message_start(i2c, msgs);
				// Refer to Operations for Master/Transmitter Mode
				stat |=  S3C2410_IICSTAT_TXRXEN;		// 1<<4: Enable Rx/Tx
				unsigned int addr = (msg->addr & 0x7f) << 1;	// 左移
				// 最低位为读写位 写为0读为1
				if (msg->flags & I2C_M_RD) {
					// RX则|=1
					stat |= S3C2410_IICSTAT_MASTER_RX;
					addr |= 1;
				} else
					stat |= S3C2410_IICSTAT_MASTER_TX;	// 3<<6: Master transmit mode
			
				// Enable i2c controller
				writel(stat, i2c->regs + S3C2410_IICSTAT);	// IICSTAT: 1<<4 | 1<<5 | 3<<6
				// Write addr
				writeb(addr, i2c->regs + S3C2410_IICDS);	// IICDS: i2c_addr
				// START
				stat |= S3C2410_IICSTAT_START;	// 1<<5: START signal generation.
				writel(stat, i2c->regs + S3C2410_IICSTAT);	// IICSTAT: 1<<5
```

中断处理函数
```
// ISR
s3c24xx_i2c_irq
	i2c_s3c_irq_nextbyte(i2c, status);
		i2c->state = STATE_WRITE;	// 2nd: STATE_WRITE
		/*
		 * 可能需要传输多个msg(i2c->msg_num)，每个msg有多个byte(i2c->msg->len)
		 * 先检查该msg是否传输完毕
		 * 再检查全部msg是否传输完毕
		 * 两者都传输完毕则stop
		 */
		if (!is_msgend(i2c)) {	// return i2c->msg_ptr >= i2c->msg->len;	len: 该msg有几个byte
			// 传输完该msg
			byte = i2c->msg->buf[i2c->msg_ptr++];
			writeb(byte, i2c->regs + S3C2410_IICDS);
		} else if (!is_lastmsg(i2c)) {	// i2c->msg_idx >= (i2c->msg_num - 1);	几个msg
			// 传输下一个msg
			i2c->msg_ptr = 0;
			i2c->msg_idx++;
			i2c->msg++;

			/* send the new start */
			s3c24xx_i2c_message_start(i2c, i2c->msg);
			i2c->state = STATE_START;
		} else {
			/* send stop */
			s3c24xx_i2c_stop(i2c, 0);
		}
```
