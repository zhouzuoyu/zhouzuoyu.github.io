---
title: sort
date: 2018-02-8 12:00:00
categories:
- c
---
# 功能
一些排列算法

排序方法		|	平均时间	|	最坏时间	|	辅助空间	|	稳定性
---			|	---		|	---		|	---		|	---
冒泡排序		|	O(n2)	|	O(n2)	|	O(1)	|	稳定
简单选择排序	|	O(n2)	|	O(n2)	|	O(1)	|	稳定
直接插入排序	|	O(n2)	|	O(n2)	|	O(1)	|	稳定
希尔排序		|	O(nlogn)|	O(n2)	|	O(1)	|	不稳定
堆排序		|	O(nlogn)|	O(nlogn)|	O(1)	|	不稳定
并归排序		|	O(nlogn)|	O(nlogn)|	O(n)	|	稳定
快速排序		|	O(nlogn)|	O(n2)	|	O(nlogn)|	不稳定
基数排序		|	O(d(n+r))|	O(d(n+r))|	O(n)	|	稳定
<!--more-->
# 源码
## bubble_sort

```c
static void bubble_sort(int a[], int n)
{
	int i=0, j=0, tmp=0;

	for (i=0; i<n-1; i++) {
		for (j=0; j<n-1-i; j++) {
			if (a[j] > a[j+1]) {
				tmp = a[j+1];
				a[j+1] = a[j];
				a[j] = tmp;
			}
		}
	}
}
```

## select_sort
```c
// find max
void select_sort(int a[], int n)
{
	int i=0, j=0, max=0;

    for (i=0; i<n-1; i++) { // 已排序序列的末尾
        max = 0;
 
        for (j=0; j<n-1-i; j++) { // 未排序序列
            if (a[j+1] > a[max]) // 依次找出未排序序列中的最小值,存放到已排序序列的末尾
                max = j+1;
        }
		if (max != n-1-i) {
			int tmp = a[max];
		    a[max] = a[n-1-i];
		    a[n-1-i] = tmp;
		}
    }
}
```
```c
//find min
void select_sort(int a[], int n)
{
	int i=0, j=0, min=0;

    for (i=0; i<n-1; i++) { // 已排序序列的末尾
        min = i;
 
        for (j=i; j<n-1; j++) { // 未排序序列 最小的放在最开始
            if (a[min] > a[j+1]) // 依次找出未排序序列中的最小值,存放到已排序序列的末尾
                min = j+1;
        }
		if (min != i) {
			int tmp = a[min];
		    a[min] = a[i];
		    a[i] = tmp;
		}
    }
}
```

## quick_sort
递归
```c
void quick_sort(int *a, int n)
{
	int i=0, j=0, p=0, tmp=0;

	if (n < 2)
		return;

	p = a[n/2];	// middle
	for (i=0,j=n-1; ;i++,j--) {
		while (a[i] < p)
			i++;
		while (a[j] > p)
			j--;
		if (i >= j)
			break;
		// exchange
		tmp = a[i]; a[i] = a[j]; a[j] = tmp;
	}
	quick_sort(a, i);
	quick_sort(a+i, n-i);
}
```
