---
title: nand(3)_device
date: 2018-02-5 12:00:00
categories:
- Linux
- nand
---

# 源码
linux-4.13.1\drivers\mtd\nand\Nand_samsung.c

# 功能
此类文件描述各个manufacturer的detect方法(获得pagesize...参数)
同一个manufacturer，detect方法相同

# 参数介绍
256M nand: 
　　mtd->writesize:2\*1024
　　mtd->oobsize:64
　　mtd->erasesize:128\*1024
　　bbt_erase_shift:17(128KB)
　　page_shift:11(2KB)

可以按字节读写，但是只能按block擦除
<!-- more -->
# 源码分析
根据read id得回的第一个数据(制造商id)来确定板上nand的参数(writesize/oobsize/erasesize/buswidth)
```c
samsung_nand_decode_id(struct nand_chip *chip)
	if (chip->id.len == 6 && !nand_is_slc(chip) &&
	    chip->id.data[5] != 0x00) {
	} else {
		nand_decode_ext_id(chip);
			u8 *id_data = chip->id.data;
			chip->bits_per_cell = nand_get_bits_per_cell(id_data[2]);	// 0x10(0001 [00]00), return []+1
			
			extid = id_data[3];
			/*
			 * 0x95: 1001 0101
			 * mtd->writesize: 2KB
			 * mtd->oobsize: 64B
			 * mtd->erasesize: 128K
			 * chip->options = NAND_BUSWIDTH_8
			 */
			mtd->writesize = 1024 << (extid & 0x03);	// 2048 = 2048
			extid >>= 2;	// 10 0101
			mtd->oobsize = (8 << (extid & 0x01)) * (mtd->writesize >> 9);	// 16 × 4 = 64
			extid >>= 2;	// 1001
			mtd->erasesize = (64 * 1024) << (extid & 0x03);	// 64K << 1 = 128×1024 = 131072
			extid >>= 2;	// 10
			if (extid & 0x1)
				chip->options |= NAND_BUSWIDTH_16;
	}
const struct nand_manufacturer_ops samsung_nand_manuf_ops = {
	.detect = samsung_nand_decode_id,
	.init = samsung_nand_init,
};
static const struct nand_manufacturer nand_manufacturers[] = {
	{NAND_MFR_SAMSUNG, "Samsung", &samsung_nand_manuf_ops},
};
nand_get_manufacturer(u8 id)
	for (i = 0; i < ARRAY_SIZE(nand_manufacturers); i++)
		if (nand_manufacturers[i].id == id)
			return &nand_manufacturers[i];
```
