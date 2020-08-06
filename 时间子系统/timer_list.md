### 1、概述

timer_list是Linux内核的普通精度定时器，和jiffes精度一致

- 常用概念

  1、HZ

  ​		每秒的节拍数

  2、jiffies

  ​		记录系统从开机到现在运行的时长，jiffies/HZ就是从开机到现在运行的秒数

### 2、使用

数据结构：

```c
 // file:linux/timer.h
 12 struct timer_list {
 13     /*
 14      * All fields that change during normal runtime grouped to the
 15      * same cacheline
 16      */
 17     struct hlist_node   entry;
 18     unsigned long       expires;		// 到期时间，以linux的jiffies来衡量
 19     void            (*function)(unsigned long);	// 到期执行函数
 20     unsigned long       data;	// 传递给到期执行函数的参数，主要在多个定时器同时使用一个函数时区分是哪个定时器
 21     u32         flags;
 22     int         slack;		// 定时器延时
 23 
 24 #ifdef CONFIG_LOCKDEP
 25     struct lockdep_map  lockdep_map;
 26 #endif
 27 };
```

- 1、初始化timer_list

```c
#define init_timer(struct timer_list*)	__init_timer((timer), 0)
#define init_timer_deferrable(struct timer_list*)	_init_timer((timer), TIMER_DEFERRABLE)
```

- 2、往系统中添加timer_list

```c
add_timer(struct timer_list*)
```

- 3、改定时器的超时时间为jiffies_timerou

```c
mod_timer(struct timer_list *, unsigned long jiffier_timerout)
```

- 4、删除定时器

```c
del_timer(struct timer_list*)

/*
	SMP架构中timer可能运行在别的处理器上，此函数用来等待别的处理器运行完timer，不能用于中断上下文
*/
int del_timer_sync(struct timer_list *timer)
```



>  其他api

- 1、查询定时器状态，如果在系统的定时器列表中则返回1，否则返回0；

```c
timer_pending(struct timer_list *)
```

- 2、时间与jiffies转换函数

```c
unsigned int jiffies_to_msecs(unsigned long);
unsigned int jiffies_to_usecs(unsigned long);
unsigned long msecs_to_jiffies(unsigned int);
unsigned long usecs_to_jiffies(unsigned int);
```

