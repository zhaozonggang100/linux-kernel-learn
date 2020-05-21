### 1、进程优先级  

参考：

https://blog.csdn.net/u010479322/article/details/51347245

普通进程优先级：100~139，nice：-20~19（ps -axl：对应优先级0~39），数值越小优先级越高

rt进程优先级：0~100，数值越小优先级越高

deadline进程：没有优先级说法，从红黑树选择最后期限最小的进程运行

stop进程：优先于deadline调度，没有优先级说法（只能内核创建）

idle进程：最低优先调用，当cfs队列没有可运行进程时执行



ps时进程的优先级：

```
pri（动态优先级）
	内核调度使用的是该优先级，对于普通进程和实时进程都是数值越小优先级越高

rtprio（实时优先级）
	这是内核提供的修改实时进程优先级的函数使用的值，值越大优先级越高	
```

### 2、修改RT进程优先级（SHCED_FIFO|SCHED_RR）

chrt -p pid 优先级

```
SCHED_OTHER 最小/最大优先级	: 0/0
SCHED_FIFO 最小/最大优先级	: 1/99
SCHED_RR 最小/最大优先级	: 1/99
SCHED_BATCH 最小/最大优先级	: 0/0
SCHED_IDLE 最小/最大优先级	: 0/0
SCHED_DEADLINE 最小/最大优先级	: 0/0

【ubuntu18.04】不能直接设置普通进程的实时优先级

【板子】1对应ps -Al优先级41,99对应ps -Al优先级139，nice值一直显示19
```

### 3、设置普通进程

renice  -n nice值 -p pid

```
nice值：-20~19，对应优先级0~39，数值越低优先级越高，nice对应的优先级也在改				 【ubuntu】
nice值：-20~19，此处的nice值正数为+nice，负数为-nice，修改nice值并不改优先级			【板子】
```

