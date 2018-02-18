---
title: strstr
date: 2018-02-8 12:00:00
categories:
- c
---

> 返回值：
> 非NULL：str2是str1的子串，返回str2在str1首次出现的地址
> NULL：str2不是str1的子串

```c
static char* strstr(const char *buf, const char *sub)
{
	if (buf==NULL || sub==NULL)
		return NULL;

	while (*buf) {
		for (int n=0; *(buf+n)==*(sub+n); n++) {
			// sub字串在buf都能对上
			if (*(sub+n+1) == '\0')
				return buf;     // 返回首次出现的位置
		}
		buf++;
	}
	return NULL;
}
```
