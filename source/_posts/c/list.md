---
title: 内核链表
date: 2018-02-8 12:00:00
categories:
- c
---
# 源码
> include\linux\list.h

# 功能
双向循环链表

# 源码分析
## 申明/定义
因为是指针，才能如此申明
```c
struct list_head {
	struct list_head *prev, *next;
};
```
<!--more-->
```c
#define LIST_HEAD_INIT(head) {&(head), &(head)}
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```

## add
```c
static void __list_add(struct list_head *new, struct list_head *prev, struct list_head *next)
{
	new->prev = prev;
	new->next = next;
	prev->next = new;
	next->prev = new;
}

static void list_add(struct list_head *new, struct list_head *head) 
{
	__list_add(new, head, head->next);
}

static void list_add_tail(struct list_head *new, struct list_head *head) 
{
	__list_add(new, head->prev, head);
}
```

## del
```c
static void list_del(struct list_head *node)
{
	node->next->prev = node->prev;
	node->prev->next = node->next;
}
```

## list_entry
结构体成员地址 -> 结构体基地址
有两种方式，但container_of定义了一个局部变量，会更为安全
```c
#if 1
#define offsetof(type, member)	(size_t) &(((type *)0)->member)
#define container_of(ptr, type, member) ({\
	const typeof(((type *)0)->member) *__mptr = (ptr);\
	(type *)((char *)__mptr - offsetof(type, member));\
})
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
#else
#define list_entry(ptr, type, member) ({\
	( (type *)((char *)ptr - (size_t)&(((type *)0)->member)) );\
})
#endif
```

## 遍历
```c
#define list_for_each(node, head) \
	for (node=(head)->next; node!=(head); node=node->next)
```

# 例子
```c
struct student {
	int age;
	char *name;
	struct list_head list;
};

int main(void)
{
	int i = 0;
	LIST_HEAD(head);
	struct student *stu = NULL;
	struct list_head *node = {0};

	for (i=0; i<5; ++i) {
		stu = calloc(1, sizeof(struct student));
		stu->age = i;
		list_add(&stu->list, &head);
	}

	list_for_each(node, &head) {
		stu = list_entry(node, struct student, list);
		printf("stu.age=%d\n", stu->age);
	}

	// del age = 3
	list_for_each(node, &head) {
		stu = list_entry(node, struct student, list);
		if (stu->age == 3)
			list_del(node, &head);
	}

	list_for_each(node, &head) {
		stu = list_entry(node, struct student, list);
		printf("stu.age=%d\n", stu->age);
	}

	// free
	list_for_each(node, &head)
		list_del(node);

	return 0;
}
```
