---
title: nand(4)_nand_base API
date: {{ date }}
categories:
- Kernel
- nand
---

# 源码
> Nand_base.c (linux-4.13.1\drivers\mtd\nand)

# 源码分析
分析nand_erase_nand/mark_bbt_block_bad/mtd_block_markbad

## 坏块标志位
```c
#define BBT_BLOCK_WORN		0x01	// 用坏的
#define BBT_BLOCK_RESERVED	0x02
#define BBT_BLOCK_FACTORY_BAD	0x03// 出厂坏块 0b11
```
<!-- more -->
## bbt_mark_entry
从code来看bbt_mark_entry最终由以下几个函数调用
bbt_mark_entry
	nand_scan_tail	// register nand controller，最开始注册nand controller的时候使用，所以不care
	mtd_block_markbad

## nand_erase_nand
擦除erase_info.addr所在的块，返回擦除状态
successful: 0, failed: -EIO
```c
einfo.addr = to;	// unit: byte
einfo.len = 1 << this->bbt_erase_shift;	// 1<<17 = 128KB
nand_erase_nand(mtd, &einfo, 1);
	page = (int)(instr->addr >> chip->page_shift);
	len = instr->len;	// 128KB
	while (len) {
		/*
		 * 先检查该块是否是坏块，如果是坏块，则直接goto erase_exit;
		 */
		nand_block_checkbad(mtd, ((loff_t) page) << chip->page_shift, allowbbt)	// byte为单位		
			return nand_isbad_bbt(mtd, ofs, allowbbt);
				block = (int)(offs >> this->bbt_erase_shift);	// 换算为block
				res = bbt_get_entry(this, block);	// 对照bbt
				switch (res) {
				case BBT_BLOCK_GOOD:
					return 0;
				case BBT_BLOCK_WORN:
					return 1;
				case BBT_BLOCK_RESERVED:
					return allowbbt ? 0 : 1;
				}
				return 1;

		// exec erase
		status = chip->erase(mtd, page & chip->pagemask);	// single_erase
			chip->cmdfunc(mtd, NAND_CMD_ERASE1, -1, page);	// 0x60
			chip->cmdfunc(mtd, NAND_CMD_ERASE2, -1, -1);	// 0xd0
			return chip->waitfunc(mtd, chip);	// nand_wait
				// 等待erase结束
				timeo = jiffies + msecs_to_jiffies(timeo);
				do {
					if (chip->dev_ready(mtd))	// s3c2440_nand_devready
						break;
				} while (time_before(jiffies, timeo));
				// 读取Status Register
				status = (int)chip->read_byte(mtd);
				return status;
		
		// erase failed
		if (status & NAND_STATUS_FAIL) {
			instr->state = MTD_ERASE_FAILED;
			instr->fail_addr = ((loff_t)page << chip->page_shift);
			goto erase_exit;
		}
		
erase_exit:
		ret = instr->state == MTD_ERASE_DONE ? 0 : -EIO;	// failed: ret = -EIO
	}
```

## mark_bbt_block_bad
指定块标记为坏块
update bbt(ddr) & markbad(在指定块的1st和2nd page的oob size的1st和2nd字节写入0x0)
```c
mark_bbt_block_bad(struct nand_chip *this,
			       struct nand_bbt_descr *td,
			       int chip, int block)
	// refresh bbt
	bbt_mark_entry(this, block, BBT_BLOCK_WORN);
	to = (loff_t)block << this->bbt_erase_shift;
	// write bbt to nand
	res = this->block_markbad(mtd, to);
		/*
		 * nand_default_block_markbad:
		 * 	在指定块的1st和2nd page的oob size的1st和2nd字节写入0x0
		 */
		uint8_t buf[2] = { 0, 0 };
		ops.oobbuf = buf;
		ops.ooboffs = chip->badblockpos;	// 0
		ops.len = ops.ooblen = 1;
		ops.mode = MTD_OPS_PLACE_OOB;
		do {
			res = nand_do_write_oob(mtd, ofs, &ops);	// unit: byte
				page = (int)(to >> chip->page_shift);
				nand_fill_oob(mtd, ops->oobbuf, ops->ooblen, ops);	// {0, 0}
					// 将ops->oobbuf cp到chip->oob_poi中
					memset(chip->oob_poi, 0xff, mtd->oobsize);
					memcpy(chip->oob_poi + ops->ooboffs, oob, len);
				status = chip->ecc.write_oob(mtd, chip, page & chip->pagemask);	// nand_write_oob_std
					/*
					 * column = mtd->writesize = 2048, page_addr = page
					 * 在指定page的指定位置(0->2048+64-1)中写入数据
					 */
					chip->cmdfunc(mtd, NAND_CMD_SEQIN, mtd->writesize, page);	// nand_command_lp
						// send 0x80 cmd，column address, row address
						chip->cmd_ctrl(mtd, command, NAND_NCE | NAND_CLE | NAND_CTRL_CHANGE); // s3c2440_nand_hwcontrol
						// 2 column address
						int ctrl = NAND_CTRL_CHANGE | NAND_NCE | NAND_ALE;
						chip->cmd_ctrl(mtd, column, ctrl);
						ctrl &= ~NAND_CTRL_CHANGE;
						chip->cmd_ctrl(mtd, column >> 8, ctrl);
						// 3 row address
						chip->cmd_ctrl(mtd, page_addr, ctrl);
						chip->cmd_ctrl(mtd, page_addr >> 8, NAND_NCE | NAND_ALE);
						chip->cmd_ctrl(mtd, page_addr >> 16, NAND_NCE | NAND_ALE);
					// send data
					const uint8_t *buf = chip->oob_poi;
					chip->write_buf(mtd, buf, length);	// {0, 0}
					// 0x10 cmd
					chip->cmdfunc(mtd, NAND_CMD_PAGEPROG, -1, -1);
					// wait
					status = chip->waitfunc(mtd, chip);
			if (!ret)
				ret = res;

			i++;
			ofs += mtd->writesize;
		} while ((chip->bbt_options & NAND_BBT_SCAN2NDPAGE) && i < 2);	// NAND_BBT_SCAN2NDPAGE: 检查1st和2nd page
```

## mtd_block_markbad
指定块标记为坏块
erase & markbad & update bbt(ddr&nand)
和mark_bbt_block_bad最大的不同是，__该函数会更新nand中的bbt__
```c
mtd_block_markbad(struct mtd_info *mtd, loff_t ofs)
	mtd->_block_markbad(mtd, ofs);
		/*
		 * nand_block_markbad
		 */
		// part 1: 到bbt中看这块是否是坏块, 如果已经标记坏块，则直接return
		nand_block_isbad(mtd, ofs);
			ret = nand_block_checkbad(mtd, offs, 0);
				return nand_isbad_bbt(mtd, ofs, allowbbt);
					res = bbt_get_entry(this, block);
					switch (res) {
					case BBT_BLOCK_GOOD:
						return 0;
					}
					return 1;
		// part 2: erase & markbad & update bbt
		return nand_block_markbad_lowlevel(mtd, ofs);	// ofs单位是byte
			// 指定块是好块则擦除
			einfo.mtd = mtd;
			einfo.addr = ofs;
			einfo.len = 1ULL << chip->phys_erase_shift;	// 1<<17 = 128KB
			nand_erase_nand(mtd, &einfo, 0);
				// 到nand中看这块是否是坏块
				nand_block_checkbad(mtd, ((loff_t) page) << chip->page_shift, allowbbt);
				// 好块则擦除
				page = (int)(instr->addr >> chip->page_shift);	// ofs>>11
				status = chip->erase(mtd, page & chip->pagemask);
			// 指定块写入坏块标记(在该块的1st和2nd page的oob的1st和2nd byte写入{ 0, 0 })
			ret = chip->block_markbad(mtd, ofs);	// nand_default_block_markbad
			// 更新bbt，包括ddr&nand
			nand_markbad_bbt(mtd, ofs);
				block = (int)(offs >> this->bbt_erase_shift);
				// chip->bbt[]中标记上
				bbt_mark_entry(this, block, BBT_BLOCK_WORN);
				// 将chip->bbt[]的内容重新写入到nand中
				nand_update_bbt(mtd, offs);
					write_bbt(mtd, buf, td, md, chipsel);
					write_bbt(mtd, buf, md, td, chipsel);
```
