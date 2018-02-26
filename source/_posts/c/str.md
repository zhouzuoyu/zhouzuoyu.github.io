---
title: 字符串处理
date: 2018-02-8 12:00:00
categories:
- c
---

## strstr
> 返回值：
> 非NULL：str2是str1的子串，返回str2在str1首次出现的地址
> NULL：str2不是str1的子串

```c
static char* strstr(const char *buf, const char *sub)
{
	if (buf==NULL || sub==NULL)
		return NULL;

	while (*buf) {
		// 进行一次比较
		for (int n=0; *(buf+n)==*(sub+n); n++) {
			// sub字串在buf中都能对上
			if (*(sub+n+1) == '\0')
				return buf;     // 返回首次出现的位置
		}
		// 下一次
		buf++;
	}
	return NULL;
}
```

## strcmp
> 返回值：
> 0：s1和s2完全一致
> -1：s1和s2不一致

两个字符串逐个比较直到末尾('\0')
```c
static int strcmp(const char *s1, const char *s2)
{
	if (s1==NULL || s2==NULL)
		return -1;

	// 一直比较，直到尽头
	while ((*s1!='\0') && (*s2!='\0')) {
		if (*s1 != *s2)
			return -1;
		else {
			s1++;
			s2++;
		}
	}

	if ((*s1=='\0') && (*s2=='\0')) // 比较到最后，都一直相等
		return 0;
	else
		return -1;
}
```

## 回文
```c
static int fun(const char *str)
{
	if (str == NULL)
		return -1;

	int length = strlen(str);
	int n = length>>1;
	const char *front=str, *back=str+length-1;

	// 一个指向前，一个指向后，逐个对比
	for (int i=0; i<n; i++) {
		if (*(front++) != *(back--))
			return -1;
	}

	return 0;
}
```
