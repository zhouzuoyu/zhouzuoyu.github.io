---
title: uart示例
date: 2019-06-24 10:16:00
categories:
- Linux
- app
---

```c
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <termios.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

static int open_uart(void)
{
	int fd = 0;

	// open uart
    fd = open("/dev/ttymxc2", O_RDWR | O_NONBLOCK);
    if (-1 == fd) {
        printf("Can't Open /dev/ttymxc1\n");
        return -1;
    }

	// some check
    if (fcntl(fd, F_SETFL, 0) < 0)
        printf("fcntl failed!\n");

    if (isatty(STDIN_FILENO) == 0)
        printf("standard input is not a terminal device\n");

    return fd;
}
```
<!--more-->
```c
static int config_uart(int fd, int nSpeed, int nBits, char nEvent, int nStop)
{
    struct termios newtio,oldtio;

	// clear
    if (tcgetattr(fd, &oldtio)  !=  0) { 
        perror("SetupSerial 1");
        return -1;
    }
    bzero(&newtio, sizeof(newtio));
    newtio.c_cflag  |=  CLOCAL | CREAD; 
    newtio.c_cflag &= ~CSIZE; 

	// bit
    switch (nBits) {
    case 7:
        newtio.c_cflag |= CS7;
        break;
    case 8:
        newtio.c_cflag |= CS8;
        break;
    }

	// check
    switch(nEvent) {
    case 'O':	// 奇校验
        newtio.c_cflag |= PARENB;
        newtio.c_cflag |= PARODD;
        newtio.c_iflag |= INPCK;
        break;
    case 'E':	// 偶校验
        newtio.c_iflag |= INPCK;
        newtio.c_cflag |= PARENB;
        newtio.c_cflag &= ~PARODD;
        break;
    case 'N':	// 无校验
        newtio.c_cflag &= ~PARENB;
        break;
    }

	// baud rate
	switch (nSpeed) {
    case 2400:
        cfsetispeed(&newtio, B2400);
        cfsetospeed(&newtio, B2400);
        break;
    case 4800:
        cfsetispeed(&newtio, B4800);
        cfsetospeed(&newtio, B4800);
        break;
    case 9600:
        cfsetispeed(&newtio, B9600);
        cfsetospeed(&newtio, B9600);
        break;
    case 115200:
        cfsetispeed(&newtio, B115200);
        cfsetospeed(&newtio, B115200);
        break;
	case 230400:
        cfsetispeed(&newtio, B230400);
        cfsetospeed(&newtio, B230400);
        break;
    default:
        cfsetispeed(&newtio, B9600);
        cfsetospeed(&newtio, B9600);
        break;
    }

	// stop bit
    if (nStop == 1)
        newtio.c_cflag &=  ~CSTOPB;
    else if (nStop == 2)
        newtio.c_cflag |=  CSTOPB;

	// set
    newtio.c_cc[VTIME]  = 0;
    newtio.c_cc[VMIN] = 0;
    tcflush(fd,TCIFLUSH);
    if ((tcsetattr(fd,TCSANOW,&newtio)) != 0) {
        perror("com set error");
        return -1;
    }

    return 0;
}

void main(void)
{
	int g_uart_fd = 0;
	
	g_uart_fd = open_uart();
	config_uart(g_uart_fd, 115200, 8, 'E', 1);

	uint8_t send_buf[10] = {0xff};
	while (1) {
		write(g_uart_fd, send_buf, 10);
		usleep(5*1000);
	}

	close(g_uart_fd);
}
```
