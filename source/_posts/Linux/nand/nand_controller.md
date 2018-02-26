---
title: nand(2)_controller
date: 2018-02-5 12:00:00
categories:
- Linux
- nand
---

# 源码
S3c2410.c (android-5.0.2\linux-3.0.86\drivers\mtd\nand)

# 功能
两部分
1：设置nand_controller
　　填充nand_chip结构体(nand的读写函数...)
2：scan板载的nand ic(read id)
<!-- more -->
# 源码分析
## 配置options
machine code: Mach-mini2440.c (linux-4.13.1\arch\arm\mach-s3c24xx)
配置nand controller的一些选项
```c
/*
 * chip->options两部分决定：
	在machine端设置是0, 在s3c2410.c中设置为NAND_SKIP_BBTSCAN
 */
static struct s3c2410_nand_set mini2440_nand_sets[] __initdata = {
	[0] = {
		.options = 0,	// 因为options在mini2440_nand_sets没有设置，所以是0
	},
};
static struct s3c2410_platform_nand mini2440_nand_info __initdata = {
	.sets = mini2440_nand_sets,
};
s3c_nand_set_platdata(&mini2440_nand_info);
	s3c_set_platdata(nand, sizeof(struct s3c2410_platform_nand), &s3c_device_nand);
		pdev->dev.platform_data = npd;	// s3c_device_nand.dev.platform_data = &mini2440_nand_info;
// nand controller code: S3c2410.c (linux-4.13.1\drivers\mtd\nand)
s3c24xx_nand_probe(struct platform_device *pdev)
	plat = to_nand_plat(pdev);
		return dev->platform_data;
	sets = (plat != NULL) ? plat->sets : NULL;	// sets = dev->platform_data->sets
	s3c2410_nand_init_chip(info, nmtd, sets);
		chip->options = set->options;		// chip->options = 0
	s3c2410_nand_update_chip(info, nmtd);
		chip->options |= NAND_SKIP_BBTSCAN;	// chip->options = NAND_SKIP_BBTSCAN;
```

## 注册nand controller
主要是填充nand_chip的成员
将nand_scan拆分为两部分
1: nand_scan_ident: nand_set_defaults/nand_detect
2: nand_scan_tail: ecc setting/bbt
```c
s3c24xx_nand_probe
	/* 
	 * 填充chip结构体，设置各个相关成员函数
	 * 根据各个平台来设置
	 */
	s3c2410_nand_init_chip(info, nmtd, sets);
		struct nand_chip *chip = &nmtd->chip;
			// 填充chip结构体的成员函数
			chip->options	= set->options;	// mini2440_nand_sets.options = 0;
			chip->read_buf  = s3c2440_nand_read_buf;
			chip->write_buf	= s3c2440_nand_write_buf;	// writesl(info->regs + S3C2440_NFDATA, buf, len >> 2);
			chip->select_chip  = s3c2410_nand_select_chip;
			chip->IO_ADDR_W = regs + S3C2440_NFDATA;
			chip->cmd_ctrl  = s3c2440_nand_hwcontrol;
			chip->IO_ADDR_R = chip->IO_ADDR_W;
			/*
			 * ecc
			 * plat = to_nand_plat(pdev);
			 * info->platform	= plat;
			 * mini2440_nand_info.ecc_mode = NAND_ECC_SOFT
			 */
			chip->ecc.mode = info->platform->ecc_mode;	// NAND_ECC_SOFT
	/*
	 * nand_scan拆分为两个part
	 * part 1: nand_scan_ident
	 * part 2: nand_scan_tail
	 */
	// part 1: 发0x90读id命令，和Nand_ids.c进行match，获得pagesize/blocksize/oob size...
	nand_scan_ident(mtd, (sets) ? sets->nr_chips : 1, NULL);	// 传入mtd和table，如传入的时NULL，则使用默认表 nand_flash_ids
		// nand controller没有填充的配置上default
		nand_set_defaults(chip);
			busw = chip->options & NAND_BUSWIDTH_16;
			chip->cmdfunc = nand_command;
			chip->scan_bbt = nand_default_bbt;
			chip->block_bad = nand_block_bad;
			chip->read_byte = busw ? nand_read_byte16 : nand_read_byte;	// nand_read_byte
		/*
		 * 读取nand id，进行match，进而获得nand的各项参数，回填到chip中(chip->mtd->writesize)
		 * 发0x90命令，得到id[]，主要是4th Cycle，该字节有nand的各项参数
		 */
		nand_detect(chip, table);
			// read id
			nand_reset(chip, 0);
			chip->select_chip(mtd, 0);
			chip->cmdfunc(mtd, NAND_CMD_READID, 0x00, -1);	// nand_command
			maf_id = chip->read_byte(mtd);	// EC: Maker Code(samsung)
			dev_id = chip->read_byte(mtd);	// 0xDA: device code
			// 为了稳定，又读了一次，返回值保存在chip->id.data中
			u8 *id_data = chip->id.data;
			chip->cmdfunc(mtd, NAND_CMD_READID, 0x00, -1);
			for (i = 0; i < 8; i++)
				id_data[i] = chip->read_byte(mtd);
			// match Maker Code
			manufacturer = nand_get_manufacturer(maf_id);	// 1
			chip->manufacturer.desc = manufacturer;
			/*
			 * nand_flash_ids中有两种类型的表，match到详细表，则找到了，否则还要nand_manufacturer_detect配合再找一次
			 *	1: 详细表: 	{"TC58NVG0S3E 1G 3.3V 8-bit",
								{ .id = {0x98, 0xd1, 0x90, 0x15, 0x76, 0x14, 0x01, 0x00} },
									SZ_2K, SZ_128, SZ_128K, 0, 8, 64, NAND_ECC_INFO(1, SZ_512),
									2 }
				2: other: EXTENDED_ID_NAND("NAND 256MiB 3,3V 8-bit",  0xDA, 256, LP_OPTIONS),
			 */
			type = nand_flash_ids;
			for (; type->name != NULL; type++) {
				if (is_full_id_nand(type)) {	// type->id_len
					// 读id的方式
					if (find_full_id_nand(chip, type))
						goto ident_done;
				} else if (dev_id == type->dev_id) {
					break;
				}
			}
			
			chip->chipsize = (uint64_t)type->chipsize << 20;	// 256 << 20
			
			/*
			 * 参见上面分析，根据id获得板上nand各项指标
			 * writesize/oobsize/erasesize/buswidth
			 */
			nand_manufacturer_detect(chip);					// 2
				chip->manufacturer.desc->ops->detect(chip);
			
			/*
			 * The initial invalid block(s) status is defined by the 1st byte in the spare area
			 * Check "FFh" at the column address 2048 of the 1st and 2nd page in the block
			 * 大页: oob的第1个字节
			 * 小页: oob的第6个字节
			 */
			nand_decode_bbm_options(chip);
				chip->badblockpos = NAND_LARGE_BADBLOCK_POS;	// 0
			
			chip->page_shift = ffs(mtd->writesize) - 1;	// page_shift = 11: 2^11 = 2048 = 2KB
			chip->bbt_erase_shift = chip->phys_erase_shift = ffs(mtd->erasesize) - 1;	// bbt_erase_shift = 17: 2^17 = 128K
			
			chip->cmdfunc = nand_command_lp;
			
			nand_manufacturer_init(chip);
				chip->options |= NAND_SAMSUNG_LP_OPTIONS;

		mtd->size = i * chip->chipsize;	// 268435456 == 256M
			
	// 更新ecc相关信息
	s3c2410_nand_update_chip(info, nmtd);
		case NAND_ECC_SOFT:	// mach代码中确定
			chip->ecc.algo = NAND_ECC_HAMMING;
			break;
	
	// part 2: 主要是为了建立bbt
	nand_scan_tail(mtd);
		// 设置ecc相关
		case NAND_ECC_SOFT:
			case NAND_ECC_HAMMING:
				ecc->read_oob = nand_read_oob_std;
					chip->cmdfunc(mtd, NAND_CMD_READOOB, 0, page);
					chip->read_buf(mtd, chip->oob_poi, mtd->oobsize);	// 保存在chip->oob_poi中
				break;
			break;
		// FIXME: 建立bbt
		ret = chip->scan_bbt(mtd);	// nand_default_bbt
	
	// 建立分区表
	s3c2410_nand_add_partition(info, nmtd, sets);
		mtd_device_parse_register(mtdinfo, NULL, NULL, set->partitions, set->nr_partitions);
```
