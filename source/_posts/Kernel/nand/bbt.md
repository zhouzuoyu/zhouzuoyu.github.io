---
title: nand(5)_bbt
date: 2018-02-5 12:00:00
categories:
- Kernel
- nand
---

# 源码
> linux-4.13.1\drivers\mtd\nand\Nand_bbt.c

# 功能
bad block table，负责创建维护BBT，然后给予yaffs等文件系统使用

# 源码分析

## nand_chip结构体
> nand_chip->bbt_td->pages: nand中哪个page存bbt
> chip->bbt[]存储了bbt，最终将该bbt存储在nand中，每两位为一个块的信息，11好00坏

<!-- more -->

```c
struct nand_chip {
	struct mtd_info mtd;
	// nand controller填充函数
	void __iomem *IO_ADDR_R;
	void __iomem *IO_ADDR_W;
	uint8_t (*read_byte)(struct mtd_info *mtd);
	...
	// bbt相关
	uint8_t *bbt;	// bbt
	struct nand_bbt_descr *bbt_td;	// main bbt
		.options = NAND_BBT_LASTBLOCK;	// 最后块往前搜
		.page[]	// bbt存在哪个page中
		.maxblocks	// 只search maxblocks个块
		.pattern	// bbt的标志位: {'B', 'b', 't', '0' }
	struct nand_bbt_descr *bbt_md;	// mirror bbt
}
```

## nand_default_bbt
分为两部分
1: search，search最后4个块的1st page数据，如果匹配到bbt pattern({'B', 'b', 't', '0' })，说明找到了bbt。并且将bbt所在的page号保存在nand_chip->bbt_td->pages[]中
2: create，search不到bbt，则读取每块1st page oob的1st&2nd byte来构建bbt({ 0xff, 0xff })，2bits代表一块信息，00好块，11坏块，保存在nand_chip->bbt[]中，最后存储在nand中
```c
nand_default_bbt
	// setting
	this->bbt_td = &bbt_main_descr;
	this->bbt_md = &bbt_mirror_descr;
	nand_create_badblock_pattern(this);
		bd->pattern = scan_ff_pattern;	// { 0xff, 0xff }
		this->badblock_pattern = bd;
	nand_scan_bbt(mtd, this->badblock_pattern);
		len = (1 << this->bbt_erase_shift);	// 2^17 = 128KB(databuf)
		len += (len >> this->page_shift) * mtd->oobsize;	// (128*1024>>11)*64 = 4096(oobbuf)
		buf = vmalloc(len);	// 分配了一个block的databuf和oobbuf大小的空间
		// Part 1: search
		search_read_bbts(mtd, buf, td, md);
			// search primary table && mirror table
			search_bbt(mtd, buf, td);
			search_bbt(mtd, buf, md);
		// Part 2: 找不到的话则create & store
		check_create(mtd, buf, bd);
			// 遍历所有block构建了bbt，保存在nand_chip->bbt[]，ddr中
			create_bbt(mtd, buf, bd, chipsel);
			// 将nand_chip->bbt[]存储在nand中, main bbt & mirror bbt
			write_bbt(mtd, buf, td, md, chipsel);
			write_bbt(mtd, buf, md, td, chipsel);
```

### search_bbt
**search部分**
bbt的标示是({'B', 'b', 't', '0' })，使用这个来看是否有现存的bbt
```c
#define NAND_BBT_SCAN_MAXBLOCKS	4
static uint8_t bbt_pattern[] = {'B', 'b', 't', '0' };
static struct nand_bbt_descr bbt_main_descr = {
	.options = NAND_BBT_LASTBLOCK | NAND_BBT_CREATE | NAND_BBT_WRITE
		| NAND_BBT_2BIT | NAND_BBT_VERSION | NAND_BBT_PERCHIP,
	.offs =	8,
	.len = 4,
	.veroffs = 12,
	.maxblocks = NAND_BBT_SCAN_MAXBLOCKS,
	.pattern = bbt_pattern
};

search_bbt
	// NAND_BBT_LASTBLOCK: 则从最后块往前search，search 4个块
	startblock = (mtd->size >> this->bbt_erase_shift) - 1;	// 256*1024*1024 >> 17 -1 = 2047
	dir = -1;
	bbtblocks = this->chipsize >> this->bbt_erase_shift;	// 256*1024*1024 >> 17
	startblock &= bbtblocks - 1;	// 2047
	// search
	for (i = 0; i < chips; i++) {
		td->pages[i] = -1;	// -1为初始值
		for (block = 0; block < td->maxblocks; block++) {
			// 从最后往前search，search 4个块
			int actblock = startblock + dir * block;	// begin: 2047->2046->2045->2044
			loff_t offs = (loff_t)actblock << this->bbt_erase_shift;	// startblock << 17, byte为单位所以要进行转换
			// 1: 读取指定块1st page数据存放在buf中
			scan_read(mtd, buf, offs, mtd->writesize, td);	// offs: 字节为单位，读取一个page数据
				scan_read_oob(mtd, buf, offs, len);	// len: pagesize, 2KB
					ops.mode = MTD_OPS_PLACE_OOB;
					ops.datbuf = buf;	// data position
					ops.len = min(len, (size_t)mtd->writesize);	// mtd->writesize: 2KB
					ops.oobbuf = buf + ops.len;	// offset 2KB == oob position
					mtd_read_oob(mtd, offs, &ops);
						mtd->_read_oob(mtd, from, ops);	// mtd->_read_oob = nand_read_oob;
							nand_do_read_ops(mtd, from, ops);
								bufpoi = use_bufpoi ? chip->buffers->databuf : buf;	// bufpoi = buf;
								chip->cmdfunc(mtd, NAND_CMD_READ0, 0x00, page);		// 1: 发送地址
								chip->ecc.read_page(mtd, chip, bufpoi, oob_required, page);	// nand_read_page_swecc: 读取指定page内容到bufpoi
									chip->ecc.read_page_raw(mtd, chip, buf, 1, page);	// nand_read_page_raw
										// read data & oob
										chip->read_buf(mtd, buf, mtd->writesize);	// 2: read
										chip->read_buf(mtd, chip->oob_poi, mtd->oobsize);
										
			// 2: check
			check_pattern(buf, scanlen, mtd->writesize, td))
				/*
				 * offset pagesize+8
				 * 即bbt flag({'B', 'b', 't', '0' })存放在page[2048+8] = oob[8]的位置
				 * bbt保存在databuf中，bbt pattern保存在oob中
				 */
				memcmp(buf + paglen + td->offs, td->pattern, td->len);
			// found then store
			td->pages[i] = actblock << blocktopage;	// FIXME: 2047 << 6: 转换为pagenum
		}
	}
```

### check_create
**create部分**
遍历所有块，构建chip->bbt[]，最后写入到nand中
```c
static uint8_t scan_ff_pattern[] = { 0xff, 0xff };
check_create
	/*
	 * 1: check
	 * 检查bbt_main和bbt_mirror
	 */
	if (td->pages[i] == -1 && md->pages[i] == -1) {
		create = 1;
		writeops = 0x03;
	}
	/*
	 * 2: create: 没有main和mirror bbt，则create
	 * 遍历0->numblocks-1个块的坏块(1st 2nd page的oob area中的1st 2nd byte的数据不为0xff)，记录在chip->bbt[]中
	 */
	create_bbt(mtd, buf, bd, chipsel);	// chipsel = 0
		numpages = 2;
		numblocks = this->chipsize >> this->bbt_erase_shift;	// 2048
		startblock = chip * numblocks;	// 0
		numblocks += startblock;	// 2048
		from = (loff_t)startblock << this->bbt_erase_shift;	// 0
		// 0 -> 2047
		for (i = startblock; i < numblocks; i++) {
			ret = scan_block_fast(mtd, bd, from, buf, numpages);	// numpages==2
				/*
				 * 读取指定块1st和2nd page的oob数据，如果1st和2nd字节数据都是0xff，则是好块，否则为坏块
				 */
				ret = mtd_read_oob(mtd, offs, &ops);
				if (check_short_pattern(buf, bd))
					if (memcmp(buf + td->offs, td->pattern, td->len))	//  { 0xff, 0xff }
						return -1;
					return 1;
			if (ret) {
				// 有异常则标记坏块
				bbt_mark_entry(this, i, BBT_BLOCK_FACTORY_BAD);	// 标记了坏块
					/*
					 * 两个bits代表一个块, 一个字节能指示4个块的信息
					 * 如chip->bbt[0]: 11 00 11 00
					 * 			   块: 3  2  1  0
					 *			 状态: 坏 好 坏 好
					 */
					uint8_t msk = (mark & BBT_ENTRY_MASK) << ((block & BBT_ENTRY_MASK) * 2);
					chip->bbt[block >> BBT_ENTRY_SHIFT] |= msk;	// FIXME: chip->bbt[]坏块表
				mtd->ecc_stats.badblocks++;
			}

			from += (1 << this->bbt_erase_shift);	// += 2^17
		}

	/*
	 * 将chip->bbt[]写入到合适的block中(nand最后4个块中的没有用过的好块)存储
	 * buf为输出型参数
	 */
	write_bbt(mtd, buf, td, md, chipsel);
		/* 1st: 计算bbt可以存储的地址, 最后转换为byte */
		numblocks = (int)(this->chipsize >> this->bbt_erase_shift);	// 256*1024*1024 >> 17 = 2048
		/* block: bbt可以存储在哪个block中 [2047/2046/2045/2044] */
		block = get_bbt_block(this, td, md, chip);
			// NAND_BBT_LASTBLOCK, down -> top
			startblock = numblocks * (chip + 1) - 1;
			dir = -1;
			for (i = 0; i < td->maxblocks; i++) {
				block = startblock + dir * i;
				// 坏块 过
				switch (bbt_get_entry(this, block)) {
				case BBT_BLOCK_WORN:
				case BBT_BLOCK_FACTORY_BAD:
					continue;
				}
				// 好块 over
				return block;
			}
		page = block << (this->bbt_erase_shift - this->page_shift);	// 块号转换为页号: 2047 << (17-11) = 2047 * 2^6 = 131008
		to = ((loff_t)page) << this->page_shift;	// 页号转换为byte: 131008 << 11 = 131008*2^11 = 268304384

		/* 2nd: 将chip->bbt[]拿出来放到buf中 */
		sft = 2; sftmsk = 0x06;
		msk[0] = 0x00; msk[1] = 0x01; msk[3] = 0x03;
		memset(buf, 0xff, len + (len >> this->page_shift)* mtd->oobsize);	// 清buf为0xff size = 2048+2048>>11*mtd->oobsize = 2048+64
		memcpy(&buf[ooboffs + td->offs], td->pattern, td->len);	// bbt pattern
		for (i = 0; i < numblocks; i++) {
			int sftcnt = (i << (3 - sft)) & sftmsk;	// sftcnt = block*2 & 0110 (0/2/4/6)
			// 得到指定block的位图，2 bits
			dat = bbt_get_entry(this, chip * numblocks + i);	// 0->2047
				entry = chip->bbt[block >> BBT_ENTRY_SHIFT];	// block 0/1/2/3都在bbt[0]中，entry = bbt[0]
				entry >>= (block & BBT_ENTRY_MASK) * 2;	// entry = bbt[0] = 11 00 11 00, block 3: bbt[0] >>= 3*2 = [11] 00 11 00: 11是坏块
				return entry & BBT_ENTRY_MASK;
			/*
			 * FIXME: chip->bbt[0] = 11001100 -> buf[0] = 00110011, 相反
			 * ddr中: 坏[11], 好[00]
			 * nand中:坏[00], 好[11]
			 * erase后都是0xff，而坏块毕竟是少数，把good置为11，会减少写flash的bit
			 */
			buf[offs + (i >> sft)] &= ~(msk[dat] << sftcnt);
		}
		
		/* 3rd: 将buf中的内容写入到nand中 */
		// 先擦再写
		einfo.addr = to;	// unit: byte
		einfo.len = 1 << this->bbt_erase_shift;	// 1<<17 = 128KB
		nand_erase_nand(mtd, &einfo, 1);	// erase
		scan_write_bbt(mtd, to, len, buf, td->options & NAND_BBT_NO_OOB ? NULL : &buf[len]);	// write
	
		td->pages[chip++] = page;
```
