---
title: 单文件夹makefile
date: 2018-03-23 17:03:55
categories:
- makefile
---

> target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在command中.

```makefile
target ... : prerequisites ...
	command
	...
	...
```
<!--more-->
目录下有main.c和func.c，编译以下makefile来进行编译
```makefile
# 编译选项
CC = gcc
CFLAGS = -g -Wall

# 链接
main: main.o func.o # main这个目标依赖main.o和func.o
	$(CC) main.o func.o -o main

# 编译
main.o: main.c		# main.o需要main.c
	$(CC) $(CFLAGS) -c main.c -o main.o
func.o: func.c
	$(CC) $(CFLAGS) -c func.c -o func.o

clean:
	rm -rf *.o
```

因为make的"隐晦规则"，xxx.o会自动找xxx.c并且设置cmd：cc -c xxx.c，所以上述makefile简化后如下：
```makefile
# 编译选项
CC = gcc
CFLAGS = -g -Wall

# 链接
main: main.o func.o # main这个目标需要main.o和func.o
	$(CC)  main.o func.o -o main

clean:
	rm -rf *.o
```
