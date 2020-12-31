参考：

```
http://abcdxyzk.github.io/blog/2017/07/23/kernel-clock-6/
https://www.cnblogs.com/arnoldlu/p/7025751.html
```



### 1、概述

Linux高精度定时器，可达到ns级

### 2、hrtimer的应用

- 1、msleep

```
1、实现基于低精度定时器，精度是1/HZ（默认是10ms）
2、会将当前任务设置为TASK_UNINTERRUPTIBLE状态，然后调用schedule_timeout，不能用于中断上下文
3、无返回值
```

- 2、msleep_interrupt

```
1、类似msleep，只是将进程状态设置为TASK_INTERRUPT，然后调用schedule_timeout，也不能用于中断上下文
2、返回值是未完成的睡眠时长
```

- 3、udelay

- 4、mdelay

```
内部实现基于udelay
```

- 5、hrtimer_nanosleep

```
1、基于hrtimer实现，精度是ns
2、hrtimer到期函数是hrtimer_wakeup
3、用户态调用是nanosleep
```

- 6、schedule_hrtimeout   

```
使得当前进程休眠指定的时间，使用CLOCK_MONOTONIC计时系统
```

- 7、schedule_hrtimeout_range  

```
使得当前进程休眠指定的时间范围，使用CLOCK_MONOTONIC计时系统
```

- 8、schedule_hrtimeout_range_clock  

```
使得当前进程休眠指定的时间范围，可以自行指定计时系统
```

- 9、usleep_range 

```
使得当前进程休眠指定的微妙数，使用CLOCK_MONOTONIC计时系统
```

- 10、用户态select、poll

```
超时实现内核调用schedule_hrtimeout_range
```

- 11、__queue_delayed_work

```
基于普通定时器delay
```

