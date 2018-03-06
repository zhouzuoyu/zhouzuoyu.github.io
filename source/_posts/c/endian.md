---
title: endian
date: 2018-03-06 11:40:28
categories:
- c
---

# 导言
在计算机系统中，我们是以字节为单位，每个地址单元都对应着一个字节。
在C语言中除了8bit的char之外，还有多字节的数据类型，比如int。另外，对于位数大于8位的处理器，例如16位或者32位的处理器，由于寄存器宽度大于一个字节，那么必然存在着如果将多个字节安排的问题。

因此就存在大端存储模式和小端存储模式。<!--more-->
大端模式：
> 数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址端

小端模式：
> 数据的高字节保存在内存的高地址中，低位字节保存在在内存的低地址端

在操作系统中，x86和一般的OS（如windows,FreeBSD,Linux）使用的是小端模式。但比如Mac OS是大端模式。

# 数据存放
```c
int main(void)
{
	int a = 0x12345678;
	char *ptr = &a;

	for (int i=0; i<sizeof(int)/sizeof(char); i++)
		printf("0x%x, ", *(ptr+i));
}
```

结果
> 0x78, 0x56, 0x34, 0x12

从该结果可以很明显看出**低地址存放数据的低位，高地址存放数据的高位，属于小端模式**

# 大小端判断
```c
int is_little_endian(void)
{
	int a = 0x12345678;
	char *ptr = &a;

	if (*ptr == 0x78) {
		printf("Is little endian\n");
		return 1;
	} else {
		printf("Is big endian\n");
		return 0;
	}
}
```
