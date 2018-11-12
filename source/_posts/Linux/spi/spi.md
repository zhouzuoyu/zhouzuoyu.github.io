---
title: spi驱动
date: 2018-11-12 13:22:00
categories:
- Linux
- spi
---

# 代码架构
分为spi设备驱动和spi总线驱动
> 设备驱动：drivers\spi\spidev.c
> 总线驱动：drivers\spi\spi-imx.c：只需要设置一些回调函数即可

<!--more-->
# spi设备驱动
## 创建/dev/spidev0.0节点
```c
static int __init spidev_init(void)
{
	status = register_chrdev(SPIDEV_MAJOR, "spi", &spidev_fops);
	spidev_class = class_create(THIS_MODULE, "spidev");
	status = spi_register_driver(&spidev_spi_driver);
}
```

```c
static int spidev_probe(struct spi_device *spi)
{
	spidev->devt = MKDEV(SPIDEV_MAJOR, minor);
	dev = device_create(spidev_class, &spi->dev, spidev->devt,
			    spidev, "spidev%d.%d",	// /dev/spidev0.0
			    spi->master->bus_num, spi->chip_select);
}
```

## spidev_write
```c
static const struct file_operations spidev_fops = {
	.owner =	THIS_MODULE,
	.write =	spidev_write,
};

static ssize_t
spidev_write(struct file *filp, const char __user *buf,
		size_t count, loff_t *f_pos)
{
	missing = copy_from_user(spidev->tx_buffer, buf, count);
	status = spidev_sync_write(spidev, count);
		spi_message_add_tail(&t, &m);
		return spidev_sync(spidev, &m);
			spi_async(spidev->spi, message);
				ret = __spi_async(spi, message);
					return master->transfer(spi, message);	// 即spi_queued_transfer
}
```

# spi总线驱动
## 总线驱动
只需要设置spi_imx->bitbang.txrx_bufs等回调函数，使用通用接口层提供的队列化框架
```c
static int spi_imx_probe(struct platform_device *pdev)
{
	// 设置回调函数
	spi_imx->bitbang.chipselect = spi_imx_chipselect;
	spi_imx->bitbang.setup_transfer = spi_imx_setupxfer;
	spi_imx->bitbang.txrx_bufs = spi_imx_transfer;
	spi_imx->bitbang.master->setup = spi_imx_setup;
	spi_imx->bitbang.master->cleanup = spi_imx_cleanup;
	spi_imx->bitbang.master->prepare_message = spi_imx_prepare_message;
	spi_imx->bitbang.master->unprepare_message = spi_imx_unprepare_message;
	spi_imx->bitbang.master->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;

	ret = spi_bitbang_start(&spi_imx->bitbang);
		master->prepare_transfer_hardware = spi_bitbang_prepare_hardware;
		master->unprepare_transfer_hardware = spi_bitbang_unprepare_hardware;
		master->transfer_one_message = spi_bitbang_transfer_one;

		ret = spi_register_master(spi_master_get(master));
			if (master->transfer)
				dev_info(dev, "master is unqueued, this is deprecated\n");
			else {	// master没有设置transfer，使用通用接口层提供的队列化框架
				status = spi_master_initialize_queue(master);
					master->transfer = spi_queued_transfer;	// 设置master->transfer

					ret = spi_init_queue(master);
						init_kthread_work(&master->pump_messages, spi_pump_messages);
					ret = spi_start_queue(master);					
						queue_kthread_work(&master->kworker, &master->pump_messages);
			}
}
```

## 框架
```c
static int spi_queued_transfer(struct spi_device *spi, struct spi_message *msg)
{
	return __spi_queued_transfer(spi, msg, true);
		list_add_tail(&msg->queue, &master->queue);
		if (!master->busy && need_pump)
			queue_kthread_work(&master->kworker, &master->pump_messages);
}
```

```c
static void spi_pump_messages(struct kthread_work *work)
	__spi_pump_messages(master, true);
		ret = master->transfer_one_message(master, master->cur_msg);	// spi_bitbang_transfer_one
			list_for_each_entry(t, &m->transfers, transfer_list) {
				status = bitbang->setup_transfer(spi, t);
				bitbang->chipselect(spi, BITBANG_CS_ACTIVE);
				status = bitbang->txrx_bufs(spi, t);	// 调用总线驱动的回调函数
				bitbang->chipselect(spi, BITBANG_CS_INACTIVE);
			}
			
```
